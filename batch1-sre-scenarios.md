# Batch 1 — SRE Scenario Questions
### Staff+ Level | Self-Interrogation + Spot Grading
---

## Q1. How do you differentiate between an L2 or L3 issue when two services on different subnets can't communicate?

---

### 🏭 Production Context

In a microservices fleet, "two services can't talk" is one of the most deceptively complex symptoms.
The blast radius difference between an L2 and L3 failure is enormous:
- L2 failure → entire VLAN segment dark, multiple services impacted, noisy
- L3 failure → surgical, route-specific, often one subnet pair, can be invisible in dashboards

At scale (500+ nodes, multiple VLANs, BGP routing), misidentifying the layer wastes 30–60 minutes during an active incident.

---

### ⚙️ Inner Workings

**Layer 2 (Data Link):**
- Operates on MAC addresses within a broadcast domain (same VLAN)
- ARP resolves IP → MAC; failure here means no ARP reply = no MAC mapping = no frame delivery
- Governed by: switch CAM tables, VLAN tags (802.1Q), STP state
- Kernel path: `arp_cache`, `neighbour subsystem`, `net/core/neighbour.c`

**Layer 3 (Network):**
- Operates on IP routing — packets routed hop-by-hop based on routing tables
- Failure here: missing route, wrong next-hop, firewall drop, TTL expiry
- Kernel path: `ip_route_output_flow()`, routing table lookup via FIB (Forwarding Information Base)
- Between subnets, L2 is irrelevant — packets MUST traverse a router/gateway

**Key Differentiator:**
- Same subnet → L2 problem if unreachable (ARP failure, VLAN mismatch)
- Different subnets → L3 problem by definition (routing, firewall, gateway)
- But: a misconfigured host may *think* it's on the same subnet when it isn't → L2 attempt for an L3 destination → silent failure

---

### 🔧 Tooling & Debug Sequence

```bash
# Step 1: Verify addressing — are both hosts configured correctly?
ip addr show          # Check IP, prefix length, VLAN tag
ip route show         # Check routing table — is there a route to destination subnet?

# Step 2: L3 reachability test
ping <dest-ip>        # ICMP — basic L3 test
traceroute <dest-ip>  # Shows hop-by-hop; where does it die?
traceroute -I <dest-ip>  # ICMP-based (bypasses firewall UDP rules)

# Step 3: L2 inspection (same subnet only)
arp -n                # Check ARP cache — is destination MAC resolved?
arping -I eth0 <dest-ip>  # Force ARP request — reply means L2 is OK

# Step 4: Packet capture — confirm packets leave and/or arrive
tcpdump -i eth0 host <dest-ip> -nn
# On sender: do packets go out?
# On receiver: do packets arrive?
# If packets leave sender but never arrive → L3/firewall issue in transit
# If packets don't leave sender → routing table issue on sender

# Step 5: Firewall / iptables inspection
iptables -L -n -v --line-numbers   # Check DROP/REJECT rules
conntrack -L | grep <dest-ip>      # Check connection tracking state

# Step 6: Route validation
ip route get <dest-ip>             # Shows exact route kernel would use
# Check: is next-hop correct? Is interface correct?

# Step 7: MTU mismatch check (silent killer)
ping -M do -s 1472 <dest-ip>       # Force DF bit, detect MTU issues
```

---

### 🔍 Follow-up Question (Level 1)

**Q: ping works between the two services but curl hangs. How does this change your L2/L3 assessment?**

**A:**
This is a classic trap. ping (ICMP) works → L3 routing and basic connectivity are fine.
curl hanging → the issue is above L3, or specifically in TCP establishment or application layer.

- ping uses ICMP — no ports, no TCP state, stateless
- curl opens a TCP connection: SYN → SYN-ACK → ACK

Possible causes when ping works but curl hangs:
1. **Firewall blocks TCP but allows ICMP** — stateless ACL that permits ICMP explicitly but has no TCP rule; check `iptables` or cloud security group rules
2. **TCP SYN sent, SYN-ACK never received** — asymmetric routing; SYN goes one path, SYN-ACK tries to return via different path that has a firewall
3. **MTU issue** — ICMP ping with small packets succeeds; TCP with larger MSS causes fragmentation that's dropped (ICMP unreachable messages also blocked)
4. **SYN backlog full on destination** — server saturated; `ss -s` shows `SYN-RECV` queue overflow
5. **Application not listening** — port closed; curl would get RST immediately though, not hang

Debug:
```bash
# Confirm TCP SYN is being sent and if SYN-ACK returns
tcpdump -i eth0 'tcp and host <dest-ip>'
# On sender: see SYN go out, watch for SYN-ACK
# If SYN goes, no SYN-ACK → firewall or routing asymmetry

ss -s          # On dest: check SYN-RECV queue depth
ss -tnp        # Active TCP sessions on dest

# MTU test
ping -M do -s 1472 <dest-ip>
# If this fails but small pings succeed → MTU/fragmentation issue
```

---

### 🔥 Follow-up Question (Level 2 — Edge Case)

**Q: Both services are in a Kubernetes cluster. Pod A can't reach Pod B on a different node. ARP looks fine from the node perspective. Where do you look?**

**A:**
This is where the L2/L3 abstraction breaks down in Kubernetes — because the pod network is an overlay that has its own routing layer independent of the node network.

Kubernetes pod-to-pod across nodes:
- Pod IPs are from the pod CIDR, not the node subnet
- Packets are encapsulated (VXLAN/Geneve with Flannel/Calico in VXLAN mode, or routed with BGP in Calico native mode)
- Node-level ARP is irrelevant to pod IP resolution

Failure points specific to K8s:
1. **CNI misconfiguration** — pod CIDR routes not programmed on node; `ip route show` on node won't have a route to the remote pod CIDR
2. **VXLAN encapsulation failure** — `tcpdump -i flannel.1` or `cali+` interface shows no traffic → CNI not forwarding
3. **iptables FORWARD chain DROP** — kernel's default forward policy drops inter-pod traffic; CNI must add ACCEPT rules
4. **kube-proxy iptables corruption** — if using ClusterIP, iptables DNAT rules may be incomplete after a node restart
5. **MTU mismatch between overlay and underlay** — overlay adds 50 bytes of header; if host MTU is 1500, effective pod MTU should be 1450; if misconfigured at 1500, fragmentation causes silent drops

```bash
# On the node hosting Pod A
ip route show | grep <pod-B-node-ip>   # Route to remote node?
ip route show | grep <pod-B-cidr>      # Route to pod CIDR?

# Check CNI-specific interfaces
ip link show    # flannel.1, cali*, tunl0 etc — are they UP?

# Check VXLAN forwarding (Flannel)
bridge fdb show dev flannel.1          # MAC → VTEP mapping

# Check iptables FORWARD chain
iptables -L FORWARD -n -v | head -20

# Packet trace: does packet leave pod A's node?
tcpdump -i <node-eth0> udp port 8472   # VXLAN traffic on node interface
tcpdump -i flannel.1                    # Decapsulated traffic

# MTU check
ip link show flannel.1 | grep mtu
kubectl exec pod-a -- ping -M do -s 1422 <pod-b-ip>
```

---

### 💡 Real-World Insight

**The Asymmetric Routing Trap:**
In multi-homed hosts (dual NIC or bonded interfaces with ECMP), a SYN can exit via eth0, but the SYN-ACK returns via eth1 — which has a different firewall context. The connection appears to hang. `ping` passes because ICMP is stateless. This has caused ~45-minute P1 incidents at companies running BGP-on-the-host routing (e.g., Calico BGP mode + node with two uplinks), where traceroute showed a clean path but TCP never completed. The fix: `rp_filter` tuning and confirming symmetric route advertisement from BGP peers.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 9 |
| Failure Awareness | 8 |
| Clarity & Signal | 8 |

**Total: 43/50**
Strong layer-by-layer separation with real K8s overlay nuance; loses a point for not covering cloud-provider-specific quirks (security groups, VPC route tables) that are often the real culprit.

**🔍 Critical Feedback:**
- Missing cloud-layer (AWS security groups, GCP firewall rules) — in most production K8s environments this is the #1 actual cause
- Could go deeper on Calico BGP mode vs VXLAN mode behavioral difference during failover
- The MTU section is correct but doesn't mention PMTUD (Path MTU Discovery) being blocked by firewalls — another silent killer

---
---

## Q2. How would you troubleshoot cross-node pod-to-pod communication issues in Kubernetes?

---

### 🏭 Production Context

Cross-node pod communication failures are one of the most common silent killers in K8s clusters at scale.
They surface as:
- Intermittent 5xx from services that resolve correctly
- Health checks passing but actual traffic failing
- Latency spikes only between specific nodes (noisy neighbor or CNI bug)
- Complete blackhole: pod A always fails to reach pod B on node X, succeeds on node Y

The severity: in a 200-node cluster with Flannel VXLAN, a single node's VTEP table corruption causes all pods on that node to lose cross-node connectivity — but intra-node traffic works fine, making it look like an application issue.

---

### ⚙️ Inner Workings — The Full Packet Path

```
Pod A (Node 1) → veth pair → cbr0/cni0 bridge → iptables FORWARD → 
overlay encapsulation (flannel.1/tunl0) → eth0 (Node 1) → 
physical network → eth0 (Node 2) → overlay decapsulation → 
iptables FORWARD → cbr0/cni0 bridge → veth pair → Pod B (Node 2)
```

Every arrow is a potential failure point.

**Key kernel interactions:**
- `veth` pairs connect pod netns to host netns
- `iptables` FORWARD chain must ACCEPT inter-pod traffic — CNI programs these rules
- Overlay: VXLAN uses UDP/8472; BGP routing adds routes directly to FIB
- `conntrack` tracks connection state — table overflow causes silent drops

---

### 🔧 Debug Sequence

```bash
# ── Phase 1: Confirm the symptom ──
kubectl exec pod-a -- curl -v http://<pod-b-ip>:<port>
kubectl exec pod-a -- ping <pod-b-ip>
# Is it all traffic or just TCP? ICMP vs TCP tells you a lot.

# ── Phase 2: Node-level routing ──
# On Node 1 (hosting Pod A):
ip route show | grep <pod-b-ip>         # Route to pod B's IP or its /24?
ip route show | grep <node-2-ip>        # Route to Node 2?

# ── Phase 3: CNI interface health ──
ip link show                             # flannel.1, cali*, tunl0 — UP?
ip -d link show flannel.1               # VXLAN details: VNI, local IP

# ── Phase 4: Packet tracing (the real work) ──
# Terminal 1: capture on pod A's node underlay
tcpdump -i eth0 -nn 'udp port 8472 and host <node-2-ip>'
# Terminal 2: capture on node 2's underlay
tcpdump -i eth0 -nn 'udp port 8472 and host <node-1-ip>'
# Terminal 3: capture on flannel.1 (decapsulated)
tcpdump -i flannel.1 -nn host <pod-b-ip>

# ── Phase 5: iptables chain inspection ──
iptables -L FORWARD -n -v              # ACCEPT rules for CNI?
iptables -L -t nat -n -v              # DNAT/MASQUERADE rules

# ── Phase 6: conntrack table ──
conntrack -L | grep <pod-b-ip>         # Is connection tracked?
conntrack -S                           # Check for insert_failed — table overflow?

# ── Phase 7: ARP/FDB for VXLAN ──
bridge fdb show dev flannel.1          # Does Node 1 know Node 2's MAC for VTEP?
arp -n | grep <node-2-ip>             # ARP entry for Node 2?

# ── Phase 8: K8s control plane checks ──
kubectl get pods -o wide               # Confirm pod IPs and node placement
kubectl describe node <node-name>      # Events: any CNI errors? disk/memory pressure?
kubectl logs -n kube-system <cni-pod> # CNI pod logs — route programming errors?

# ── Phase 9: MTU ──
kubectl exec pod-a -- ping -M do -s 1422 <pod-b-ip>
# 1500 (node MTU) - 50 (VXLAN overhead) - 28 (IP+ICMP) = 1422
```

---

### 🔍 Follow-up Question (Level 1)

**Q: You run tcpdump on Node 1 and see VXLAN packets going to Node 2. On Node 2, tcpdump shows packets arriving on eth0. But Pod B never receives them. Where exactly is the drop?**

**A:**
Packets arrive at Node 2's eth0 but never reach Pod B. The drop is happening *inside* Node 2's network stack, after decapsulation.

Possible locations:
1. **flannel.1/tunl0 decapsulation fails** — VXLAN VNI mismatch or wrong destination MAC in inner packet
   ```bash
   tcpdump -i flannel.1 -nn   # If this shows nothing, decap is failing
   ```
2. **iptables FORWARD DROP** — The inner packet (pod IP) hits FORWARD chain and gets dropped
   ```bash
   iptables -L FORWARD -n -v
   # Look for policy DROP and no matching ACCEPT for pod CIDR
   # Symptom: CNI pod crashed and didn't restore its iptables rules
   ```
3. **Wrong route to pod — sent to wrong interface** — After decap, kernel routes inner packet but wrong interface is selected
   ```bash
   ip route get <pod-b-ip>   # What interface does kernel pick?
   ```
4. **veth pair down or missing** — Pod B's veth might have been partially cleaned up
   ```bash
   ip link show | grep <pod-b-veth>
   kubectl exec pod-b -- ip link show eth0   # Is pod's interface up?
   ```
5. **conntrack DROP** — If pod B's IP was recently re-used (pod restart), stale conntrack entries may DROP new connections thinking they're invalid mid-session packets
   ```bash
   conntrack -L | grep <pod-b-ip>
   conntrack -D -d <pod-b-ip>   # Clear stale entries
   ```

Use eBPF to trace the exact drop point:
```bash
# bpftrace: trace where the packet is dropped
bpftrace -e 'kprobe:kfree_skb { printf("drop at %s\n", ksym(*(reg("ip")-8))); }'
# Or use dropwatch:
dropwatch -l kas
```

---

### 🔥 Follow-up Question (Level 2 — Edge Case)

**Q: The issue is intermittent — 1 in 50 requests fails, only between two specific nodes, and only under load. How does your approach change?**

**A:**
Intermittent + load-triggered + node-pair specific → this pattern points to:

1. **conntrack table overflow** — Under load, `conntrack` table fills up; new connections can't be tracked and get dropped
   ```bash
   cat /proc/sys/net/netfilter/nf_conntrack_count    # current
   cat /proc/sys/net/netfilter/nf_conntrack_max      # limit
   conntrack -S | grep insert_failed                 # Failed inserts = drops
   ```
   Fix: tune `nf_conntrack_max` or switch kube-proxy to IPVS (uses less conntrack)

2. **ECMP hashing asymmetry** — With multiple uplinks, VXLAN UDP packets are load-balanced by 5-tuple. If inner packet 5-tuple always hashes to the same outer 5-tuple, one path gets saturated, the other idle — causing intermittent drops from queue overflow on that interface
   ```bash
   ethtool -S eth0 | grep -i drop   # NIC queue drops?
   ```

3. **Flannel VTEP ARP storm** — Under churn (pod reschedules), flannel sends many ARP/FDB updates; if the flannel daemon is slow, VTEP table is stale for brief windows → dropped packets

4. **CPU-pinned IRQs** — All NIC interrupts pinned to CPU 0; under load, CPU 0 saturates, packets drop in softirq queue before reaching flannel
   ```bash
   cat /proc/interrupts | grep eth0
   # Check if all interrupts on one CPU
   ```

For intermittent issues, use packet counters + eBPF sampling:
```bash
# Watch interface drops in real time
watch -n1 'ip -s link show eth0 | grep -A2 RX'

# bpftrace: count drops per source/dest
bpftrace -e 'kprobe:kfree_skb /args->reason != 0/ { @drops[args->reason]++; }'
```

---

### 💡 Real-World Insight

**The conntrack insert_failed silent killer:**
At a company running 800-pod clusters, 1 in 20 inter-pod connections would fail randomly during peak hours. `ping` always worked. TCP connections occasionally failed. Root cause: `nf_conntrack_max` was set to default (65536) — under load, the table filled, `insert_failed` counter incremented silently, and new connections were dropped with no log. The fix: increase `nf_conntrack_max` to 524288 + switch to IPVS kube-proxy which batches conntrack entries more efficiently. Monitoring: never alert on `nf_conntrack_count` alone — always alert on `insert_failed` rate.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 10 |
| Failure Awareness | 9 |
| Clarity & Signal | 8 |

**Total: 45/50**
Excellent packet-path decomposition with real conntrack insight; docked for not explicitly covering Calico/eBPF mode which is increasingly common, and for not mentioning NetworkPolicy as a drop source.

**🔍 Critical Feedback:**
- NetworkPolicy is a very common cause of cross-node drops — should be in Phase 1 check
- Calico in eBPF mode bypasses iptables entirely — the iptables debug steps would be misleading in that environment
- Should mention `kubectl debug node` (ephemeral container on node) for live packet capture without SSH

---
---

## Q3. How would you design a highly available HAProxy and Keepalived setup for zero downtime, and what considerations would you factor in for health checks and failover?

---

### 🏭 Production Context

HAProxy + Keepalived is the classic L4/L7 HA pattern outside Kubernetes, still heavily used for:
- Database load balancing (where L7 ingress is too slow)
- Edge load balancing before traffic hits K8s ingress
- Internal service mesh bypass for high-throughput services

Failure modes at scale:
- VIP flapping (Keepalived VRRP split-brain)
- HAProxy accepting connections on dead backends
- Health check timing causing false negatives during deployments
- Failover taking 10–30 seconds instead of sub-second

---

### ⚙️ Architecture Design

```
                     VIP: 10.0.0.100 (VRRP)
                          │
              ┌───────────┴──────────┐
              │                      │
         HAProxy-1              HAProxy-2
         (MASTER)               (BACKUP)
              │                      │
    Keepalived VRRP         Keepalived VRRP
    priority: 100           priority: 90
              │                      │
              └──────────┬───────────┘
                         │
              ┌──────────┼──────────┐
           Backend1   Backend2   Backend3
```

**Keepalived VRRP Config (Master):**
```
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1                    # VRRP advertisement every 1s
    authentication {
        auth_type PASS
        auth_pass <secret>
    }
    virtual_ipaddress {
        10.0.0.100/24
    }
    track_script {
        chk_haproxy                 # Monitor HAProxy process
    }
    notify_master /etc/keepalived/notify_master.sh
    notify_fault  /etc/keepalived/notify_fault.sh
}

vrrp_script chk_haproxy {
    script "kill -0 $(cat /var/run/haproxy.pid)"
    interval 2
    weight -30                      # Drop priority by 30 if HAProxy dies
    fall 2                          # Fail after 2 consecutive failures
    rise 2                          # Recover after 2 consecutive successes
}
```

**HAProxy Health Check Design:**
```
backend app_servers
    balance leastconn
    option httpchk GET /health HTTP/1.1\r\nHost:\ internal
    http-check expect status 200
    default-server inter 2s fastinter 500ms downinter 5s rise 2 fall 3
    server app1 10.0.1.10:8080 check
    server app2 10.0.1.11:8080 check
    server app3 10.0.1.12:8080 check maxconn 200
```

Key parameters:
- `inter 2s` — health check every 2s in normal state
- `fastinter 500ms` — check every 500ms when transitioning (after first failure)
- `downinter 5s` — check every 5s when confirmed down (reduce noise)
- `rise 2 fall 3` — require 3 failures to mark down, 2 successes to bring back up
- `maxconn 200` — per-server connection cap; prevents single backend overload

---

### 🔧 Tooling & Observability

```bash
# Monitor Keepalived VRRP state
ip addr show eth0 | grep 10.0.0.100   # Does this node hold the VIP?
journalctl -u keepalived -f            # Real-time VRRP events

# HAProxy runtime API (socket)
echo "show info" | socat stdio /var/run/haproxy/admin.sock
echo "show stat" | socat stdio /var/run/haproxy/admin.sock | cut -d, -f1,2,17,18
# Shows: backend, server, status, active sessions

# Drain a backend (zero-downtime removal)
echo "set server app_servers/app1 state drain" | socat stdio /var/run/haproxy/admin.sock
# State: drain = no new sessions, existing sessions finish

# Health check status
echo "show servers state" | socat stdio /var/run/haproxy/admin.sock

# Verify VIP failover time
hping3 -S -p 80 10.0.0.100 -i u100 | ts   # 100µs interval, timestamped
# Count gaps during failover to measure actual downtime
```

---

### 🔍 Follow-up Question (Level 1)

**Q: What happens during a Keepalived failover — specifically what causes the ~2–3 second downtime window even in a "zero-downtime" setup?**

**A:**
True zero-downtime with Keepalived is often a lie. The actual failover path:

1. **Detection delay:** MASTER stops sending VRRP advertisements (every `advert_int` = 1s)
   - BACKUP waits `(3 × advert_int) + skew_time` before assuming MASTER is dead = ~3 seconds default
   - With `advert_int 1`, BACKUP waits 3 seconds before taking VIP

2. **ARP gratuitous broadcast:** BACKUP sends gratuitous ARP to announce new MAC → VIP mapping
   - Switches update CAM tables immediately (usually <100ms)
   - But: clients with cached ARP entries for the VIP still point to MASTER's MAC
   - ARP cache timeout on clients = 60 seconds by default → clients drop packets for up to 60s

3. **HAProxy connection state:** Existing TCP connections to VIP are lost on failover — BACKUP's HAProxy has no state about MASTER's in-flight connections

**Reducing actual downtime:**
- `advert_int 0.1` (100ms advertisements, requires Keepalived 2.x)
- `garp_master_repeat 5` + `garp_master_delay 1` — send multiple gratuitous ARPs on takeover
- Set low ARP cache timeouts on clients: `net.ipv4.neigh.default.gc_stale_time = 60` (already 60s, reduce with caution)
- For truly zero-downtime: use ECMP with BFD (Bidirectional Forwarding Detection) — millisecond failover, but requires BGP infrastructure

---

### 🔥 Follow-up Question (Level 2 — Edge Case)

**Q: Both HAProxy nodes are healthy, both believe they're MASTER (split-brain). How does this happen and how do you detect/prevent it?**

**A:**
VRRP split-brain: both nodes send VRRP advertisements thinking they're MASTER. Both claim the VIP. Both send gratuitous ARPs. Network switches get confused — clients connect to whichever node answered last ARP. This causes:
- Non-deterministic load balancing
- Duplicate connection handling
- Session state corruption

**Causes:**
1. **Network partition** — HAProxy-1 and HAProxy-2 can't reach each other but both can reach the VIP subnet (different failure paths)
2. **VRRP packets dropped by firewall** — VRRP uses protocol 112 (not TCP/UDP); `iptables` rules that only allow TCP/UDP silently drop VRRP → each node thinks the other is dead
3. **Multicast blocked** — VRRP uses 224.0.0.18 multicast; some switches or VLANs block multicast → VRRP communication fails

**Detection:**
```bash
# Check if both nodes think they're MASTER
# On both nodes simultaneously:
ip addr show eth0 | grep <VIP>
# If BOTH show the VIP → split-brain

# Check VRRP traffic
tcpdump -i eth0 -nn proto 112   # Protocol 112 = VRRP
# If Node 2 sees no VRRP from Node 1 → communication failure

# Check multicast
ip maddr show eth0 | grep 224.0.0
```

**Prevention:**
1. **VRRP authentication** — `auth_type AH` (stronger than PASS) — prevents rogue MASTER
2. **Dedicated VRRP heartbeat interface** — separate NIC for VRRP traffic, not shared with data plane
3. **Firewall rule for VRRP:**
   ```bash
   iptables -A INPUT -p 112 -j ACCEPT
   ```
4. **notify_fault script** — if HAProxy dies and node transitions to FAULT, script runs; use it to actively remove VIP if split-brain is detected
5. **External arbitrator (quorum)** — use a third node/service as tie-breaker (similar to etcd quorum in K8s)

---

### 💡 Real-World Insight

**The Health Check False Positive During Deployment:**
During rolling restarts, backends have a brief window where the process is up but not ready (e.g., warming caches, loading config). HAProxy marks them UP immediately after `/health` returns 200 — but the app is still initializing. Requests hit this backend and fail. Fix: implement `/health/ready` vs `/health/live` separation — HAProxy checks `/health/ready` which only returns 200 after full initialization. Combine with `rise 3` to require 3 consecutive successes before marking backend UP — adds 6 seconds delay but eliminates false-positive readiness.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 8 |
| Debugging Practicality | 8 |
| Failure Awareness | 9 |
| Clarity & Signal | 9 |

**Total: 43/50**
Strong on VRRP mechanics and split-brain; loses points for not covering HAProxy hot reload mechanics and for missing the BGP alternative discussion depth.

**🔍 Critical Feedback:**
- HAProxy hot reload (`haproxy -sf`) is critical for zero-downtime config changes — completely absent
- Should discuss hitless reload vs restart and how HAProxy uses `SO_REUSEPORT` to hand off listeners
- The BFD/BGP alternative for millisecond failover was mentioned but not elaborated — Staff engineers should know this deeply
- Missing: `nbthread` and `cpu-map` tuning for HAProxy under high connection rates

---
---

## Q4. How would you identify the cause of CPU saturation on a production node, and what steps would you take to mitigate it?

---

### 🏭 Production Context

CPU saturation at scale doesn't present as "one process is using 100% CPU" — it manifests as:
- Increased request latency across all services on the node (noisy neighbor)
- OOM killer activating because reclaim routines can't run fast enough
- Kernel scheduler starvation: low-priority processes (monitoring agents) starve, causing gaps in metrics precisely when you need them most
- Thundering herd: N pods on a node all wake simultaneously, saturating CPU in burst, latency spike, then idle

The signal is often: P99 latency spikes on a node that looks "fine" at P50 CPU util.

---

### ⚙️ Inner Workings

**Linux CPU Scheduler (CFS — Completely Fair Scheduler):**
- Maintains a red-black tree of runnable tasks sorted by virtual runtime (vruntime)
- Each task accumulates CPU time; the task with lowest vruntime runs next
- Saturation = run queue depth > 1 per CPU core for sustained periods
- `load average` = exponentially weighted moving average of run queue depth (R state) + uninterruptible sleep (D state)
- Key distinction: high load average from I/O wait ≠ CPU saturation

**User vs Kernel CPU time:**
- `%us` — userspace computation
- `%sy` — kernel syscalls, context switches
- `%si` — software interrupt (softirq): network packet processing, timer interrupts
- `%hi` — hardware interrupt
- `%st` — steal time (hypervisor took CPU from VM) — critical in cloud environments

---

### 🔧 Debug Sequence

```bash
# ── Phase 1: Quick signal ──
top -b -n1                        # Overall CPU breakdown
vmstat 1 10                       # r column = run queue depth; b = blocked
# r > num_cpus = saturation

uptime                            # Load average — compare to CPU count
# Load avg > 2× CPU count = severe saturation

# ── Phase 2: Identify the consumer ──
# Top processes by CPU
ps aux --sort=-%cpu | head -20

# Per-CPU breakdown
mpstat -P ALL 1 5                 # Is saturation on all CPUs or just some?
# If one CPU saturated → likely single-threaded app, IRQ pinning issue, or scheduling problem

# ── Phase 3: Thread-level resolution ──
# Find which threads of a process consume CPU
pidstat -t -p <pid> 1 5
top -H -p <pid>                   # -H shows threads

# ── Phase 4: What is the process doing? ──
perf top                          # Live: which kernel/userspace symbols consuming cycles?
perf record -g -p <pid> sleep 10  # Sample call graph
perf report --sort=sym            # Flame graph data

# Generate actual flame graph:
perf record -F 99 -p <pid> -g -- sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg

# ── Phase 5: Syscall analysis ──
strace -c -p <pid>                # Count syscalls — excessive futex/poll = contention
perf stat -p <pid>                # IPC, cache misses, branch mispredictions

# ── Phase 6: Steal time (cloud) ──
vmstat 1 | awk '{print $16}'      # st column — steal time
# > 5% steal = hypervisor noisy neighbor, not your app's fault

# ── Phase 7: Softirq / Network ──
watch -n1 'cat /proc/softirqs'    # Is NET_RX/NET_TX saturated?
# High softirq CPU → network processing overwhelming one CPU
cat /proc/interrupts | grep <nic> # Are IRQs pinned to one CPU?
ethtool -L eth0 combined 8        # Set RX queues = CPU count for RSS
```

---

### 🔍 Follow-up Question (Level 1)

**Q: `perf top` shows `__futex_abstimed_wait_restart` dominating. What does this tell you and what do you do?**

**A:**
`futex` is the kernel primitive for userspace mutex/semaphore/condvar — used by pthreads, Java synchronized, Go runtime, etc.

Heavy futex wait in `perf top` = lock contention, not CPU computation. The CPU is being consumed by:
1. Threads spinning briefly before futex wait (spin-then-sleep pattern in glibc pthreads)
2. Kernel overhead of `futex_wait`/`futex_wake` syscalls at high concurrency

What this means:
- Application is NOT CPU-bound in computation — it's CONCURRENCY-bound
- More CPUs won't help — you need to reduce lock contention
- Common causes: single global mutex in thread pool, database connection pool contention, Java GC STW pause causing all threads to block

Diagnosis:
```bash
# Count futex syscalls per second
strace -c -e trace=futex -p <pid>   # futex call count and time

# Java-specific: thread dump shows who holds the lock
jstack <pid> | grep -A5 "BLOCKED"

# Go-specific: goroutine dump
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Linux: show which threads are in futex wait
cat /proc/<pid>/task/*/wchan | sort | uniq -c | sort -rn
# wchan = "wait channel" — what kernel function is the thread sleeping in
```

Mitigation:
- Shard the lock (multiple smaller locks instead of one global)
- Use lock-free data structures (atomic operations)
- Reduce critical section size
- For connection pools: ensure pool size matches workload concurrency

---

### 🔥 Follow-up Question (Level 2 — Edge Case)

**Q: CPU is saturated on 1 out of 8 CPUs. All others are idle. Network traffic is balanced. Why?**

**A:**
Single-CPU saturation with others idle → one of these:

1. **IRQ affinity — all NIC interrupts pinned to CPU 0:**
   ```bash
   cat /proc/interrupts | grep eth0
   # If all counts accumulate on CPU 0
   # Fix:
   service irqbalance stop    # Disable irqbalance if manually tuning
   set_irq_affinity.sh eth0   # Pin different IRQ queues to different CPUs
   ```

2. **Single-threaded application — can't use more than 1 CPU:**
   - Node.js, Redis, single-thread Python — by design, only 1 CPU is ever used
   - Fix: run multiple instances pinned to different CPUs, or use clustering

3. **NUMA topology — process pinned to wrong NUMA node:**
   ```bash
   numactl --hardware              # Show NUMA topology
   numastat -p <pid>               # NUMA memory allocation per node
   # If process is on NUMA node 1 but memory is on node 0 → remote NUMA access → CPU spins waiting for memory
   ```

4. **Kernel timer wheel or RCU callbacks concentrated on one CPU:**
   ```bash
   mpstat -P ALL 1 | grep -v "^$" | sort -k4 -rn
   # perf top per CPU:
   perf top --cpu=0
   # Look for: rcu_process_callbacks, run_timer_softirq
   ```

5. **cgroup CPU quota exhausted — throttling manifests as single-CPU saturation:**
   ```bash
   cat /sys/fs/cgroup/cpu/<container>/cpu.stat | grep throttled
   # nr_throttled > 0 → container hitting CPU quota
   # The saturated CPU is actually doing work for quota accounting overhead
   ```

---

### 💡 Real-World Insight

**The Steal Time Trap:**
Team got paged at 3am: CPU saturation, all services slow. `top` showed `%us` at 80%. `perf` showed legitimate app code. No obvious resource leak. Response: vertical scale the instance. Problem persisted. Root cause: `%st` (steal time) was 40% — the hypervisor was time-slicing the physical CPU away from this VM due to a noisy neighbor VM on the same host. The real CPU available was 60% of nominal. Fix: migrate to a dedicated host or instance type with CPU isolation. Lesson: always check `%st` in cloud environments before assuming the app is the problem.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 10 |
| Failure Awareness | 9 |
| Clarity & Signal | 8 |

**Total: 45/50**
Excellent steal time inclusion and IRQ affinity coverage; loses a point for not covering CPU frequency throttling (thermal/power capping) and not mentioning `turbostat` for cloud metal instances.

**🔍 Critical Feedback:**
- Missing `turbostat` and CPU frequency scaling — p-state throttling looks identical to steal time but different fix
- `perf stat` interpretation (IPC < 1 = memory-bound, not CPU-bound) should be highlighted as a key differentiator
- cgroup CPU throttling section is good but should mention `cpu.cfs_quota_us` tuning explicitly

---
---

## Q5. If a microservice runs out of file descriptors, how would you diagnose the cause and prevent it from recurring in production?

---

### 🏭 Production Context

FD exhaustion is insidious because:
- The service keeps running (no crash, no OOM kill)
- Error messages are generic: `Too many open files`, `EMFILE`, `ENFILE`
- Often surfaces as inability to accept new connections, not inability to process existing ones
- At scale: a service with 200 threads, each holding 50 FDs for DB connections, HTTP keep-alives, log files = 10,000 FDs per process; hit the `ulimit` and you see only errors — no crash to trigger an alert

Two distinct limits:
- `EMFILE` — per-process limit (`ulimit -n`), default 1024
- `ENFILE` — system-wide limit (`/proc/sys/fs/file-max`)

---

### ⚙️ Inner Workings

Every `open()`, `socket()`, `pipe()`, `epoll_create()`, `eventfd()`, `timerfd_create()` consumes an FD.
FD table: each process has a table in kernel (`struct fdtable`), pointed to by `task_struct->files`.
FDs are integers — indices into this table.
File descriptions (the underlying object) are shared: `dup()`, `fork()` → same file description, different FD.

Hidden FD consumers:
- Every accepted TCP socket
- Every open log file (if not using centralized logging via stdout)
- epoll instance itself (1 FD) + each monitored socket registered with epoll
- Unix domain sockets (health check endpoints, metrics)
- inotify watches
- Every thread that uses `pthread_create` internally in some runtimes

---

### 🔧 Debug Sequence

```bash
# ── Phase 1: Confirm FD exhaustion ──
# Look for EMFILE in application logs
grep -i "too many open files\|EMFILE\|ENFILE" /var/log/app/*.log

# ── Phase 2: Current FD usage ──
# Per-process FD count
ls /proc/<pid>/fd | wc -l              # Count open FDs for process
ls -la /proc/<pid>/fd/                 # What are they? (files, sockets, pipes)

# Per-process limit
cat /proc/<pid>/limits | grep "open files"
# Shows: soft limit, hard limit, max

# System-wide
cat /proc/sys/fs/file-nr               # allocated / free / max
# If allocated ≈ max → ENFILE (system-wide exhaustion)

# ── Phase 3: What types of FDs? ──
ls -la /proc/<pid>/fd | awk '{print $NF}' | sed 's/\[.*\]/[socket]/g' | sort | uniq -c | sort -rn
# Breakdown: sockets vs files vs pipes vs eventfd

# Inspect specific socket FDs
ss -tulnp | grep <pid>                 # What sockets does process hold?
lsof -p <pid> | grep ESTABLISHED | wc -l   # How many established connections?
lsof -p <pid> | grep CLOSE_WAIT | wc -l    # CLOSE_WAIT = FD leak pattern!

# ── Phase 4: FD leak detection ──
# CLOSE_WAIT connections = server hasn't called close() on socket FD
# Application bug: received FIN from client, failed to close socket
ss -s | grep CLOSE_WAIT                # System-wide CLOSE_WAIT count
lsof -p <pid> -i | grep CLOSE_WAIT    # Process-specific

# Time-series: watch FD count grow
watch -n5 'ls /proc/<pid>/fd | wc -l'
# If growing monotonically → leak

# ── Phase 5: strace to catch unclosed FDs ──
strace -e trace=open,openat,close,socket,accept -p <pid> 2>&1 | grep -v close
# If open/accept calls vastly outnumber close calls → leak

# eBPF: trace FD open without close
bpftrace -e '
tracepoint:syscalls:sys_exit_openat { @fds[pid] = count(); }
tracepoint:syscalls:sys_exit_close  { @fds[pid] = count(); }
'

# ── Phase 6: Confirm limits and fix immediately ──
ulimit -n 65536                        # Bump soft limit for current shell/process
# Permanent fix via systemd:
# LimitNOFILE=65536 in service unit file
# /etc/security/limits.conf for non-systemd
```

---

### 🔍 Follow-up Question (Level 1)

**Q: You increase `ulimit -n` to 65536 but within hours the service exhausts FDs again. What's really happening?**

**A:**
If increasing ulimit doesn't solve it — it's not a limit configuration problem, it's a **leak**.

The service is opening FDs and not closing them. Common leak patterns:

1. **HTTP keep-alive connections not cleaned up:**
   - Client connects, does request, goes idle, eventually disconnects
   - Server receives FIN (TCP), but application never reads EOF and never calls `close()`
   - Socket stays in CLOSE_WAIT indefinitely → FD held forever
   ```bash
   ss -tnp state close-wait | grep <pid>
   # Growing CLOSE_WAIT count = definitive leak signal
   ```

2. **Database connection pool leak:**
   - Exception path in code skips `connection.close()`
   - Pool keeps allocating new connections; old ones never returned
   - Each DB connection = 1 socket FD
   ```bash
   lsof -p <pid> -i | grep :5432   # Count Postgres connections
   # Should match connection pool size; if 10× pool size → leak
   ```

3. **File handle leak in log rotation:**
   - App opens log file, log rotator moves file, app still holds FD to deleted file
   - `/proc/<pid>/fd` shows symlink to `(deleted)` path
   ```bash
   ls -la /proc/<pid>/fd | grep deleted   # FDs to deleted files
   ```

4. **Thread leak holding FDs:**
   - Threads inherit parent's FDs; if threads are created and never joined, their FDs accumulate
   ```bash
   ls /proc/<pid>/task | wc -l   # Thread count — growing?
   ```

Prevention:
- Use connection pool libraries with proper lifecycle management (HikariCP, pgbouncer)
- Set `SO_LINGER` and explicit timeout on idle connections
- Implement FD usage as a Prometheus metric: `process_open_fds` — alert at 80% of limit
- Use `-D FIN_WAIT_TIMEOUT` tuning for TCP socket cleanup
- Code review: ensure all resource-opening code is in try-with-resources / defer blocks

---

### 🔥 Follow-up Question (Level 2 — Edge Case)

**Q: FD usage looks stable in the process, but new connections are still rejected with EMFILE. `ls /proc/<pid>/fd | wc -l` shows 1024. ulimit is set to 65536. How?**

**A:**
This is a classic gotcha: `ulimit` was set **after** the process was already running. Changes to `ulimit -n` in a shell don't affect already-running processes.

More specifically:
- The process was started by systemd with default `LimitNOFILE=1024`
- Operator ran `ulimit -n 65536` in their SSH session
- The running process still has soft limit of 1024

Verify:
```bash
cat /proc/<pid>/limits | grep "open files"
# If soft limit = 1024 → process has old limit regardless of current shell ulimit
```

Fix without restart:
```bash
# Use prlimit to change limits of running process
prlimit --pid <pid> --nofile=65536:65536
cat /proc/<pid>/limits | grep "open files"   # Confirm it took effect
```

Permanent fix:
```bash
# systemd service file:
[Service]
LimitNOFILE=131072

systemctl daemon-reload
systemctl restart <service>

# Verify after restart:
cat /proc/<pid>/limits | grep "open files"
```

Also: the kernel's system-wide limit:
```bash
cat /proc/sys/fs/file-max             # System total
sysctl -w fs.file-max=2097152         # Increase if hitting system-wide limit
echo "fs.file-max = 2097152" >> /etc/sysctl.conf
```

---

### 💡 Real-World Insight

**The CLOSE_WAIT Accumulation at 2am:**
A Node.js API gateway would exhaust FDs every night around 2am — coinciding with a batch job that made thousands of HTTP requests to the gateway and then terminated. Root cause: the gateway used HTTP keep-alive connections; when batch job processes died, they sent FIN. Gateway never read EOF on idle connections (no timeout configured), so connections stayed in CLOSE_WAIT. Over 8 hours of batch jobs, CLOSE_WAIT accumulated to 65,536 (the ulimit). Fix: set `server.keepAliveTimeout = 5000` in Node.js + added `SO_KEEPALIVE` with aggressive TCP keepalive tuning to reap dead connections. Monitoring: alert on `CLOSE_WAIT count > 1000` for this service.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 9 |
| Failure Awareness | 9 |
| Clarity & Signal | 9 |

**Total: 45/50**
Excellent CLOSE_WAIT and prlimit coverage; loses a point for not mentioning `/proc/sys/fs/nr_open` (the hard ceiling that even root can't exceed without kernel recompile) and for not covering the Java-specific FD issue with `finalize()` delay.

**🔍 Critical Feedback:**
- `/proc/sys/fs/nr_open` is the absolute ceiling — even with `LimitNOFILE=unlimited`, this is the real limit; missing
- Java FD leak via uncleaned finalizers is a very common production issue worth a mention
- Should mention `lsof +L1` to find FDs to unlinked (deleted) files specifically

---
---

## Q6. How would you handle a Kubernetes rolling deployment where some pods fail to reach ready state?

---

### 🏭 Production Context

A rolling deployment stalling mid-way is one of the most dangerous states in K8s:
- Old pods being terminated, new pods not ready → serving capacity drops
- If `maxUnavailable=1` and `maxSurge=1`, you can have (n-2) pods serving during stall
- The Deployment controller halts — it won't proceed OR rollback automatically
- This window can last indefinitely if readiness probe keeps failing
- In production: alert on `deployment_replicas_unavailable > 0 for 10m`

---

### ⚙️ Inner Workings — Rolling Update State Machine

```
Desired: 5 replicas, maxSurge=1, maxUnavailable=1

State during stall:
  Old pods:  3 Running+Ready, 1 Terminating
  New pods:  2 Running, NOT Ready (stuck in 0/1)
  
Deployment controller: HALTED. Waiting for new pod to become Ready.
```

Readiness probe failure → pod stays in NotReady → Deployment stalls → endpoints controller removes pod from Service endpoints → no traffic to new pod → but old pods being terminated → capacity gap

---

### 🔧 Debug Sequence

```bash
# ── Phase 1: Understand current state ──
kubectl rollout status deployment/<name>
# "Waiting for deployment to finish: 2 out of 5 new replicas have been updated..."

kubectl get pods -l app=<label> -o wide
# STATUS column: Running vs CrashLoopBackOff vs Pending vs Init:Error

kubectl describe deployment/<name>
# Events section: shows what controller is doing/why it's stalled

# ── Phase 2: Diagnose failing pod ──
kubectl describe pod <failing-pod>
# CRITICAL SECTIONS:
# - Events: pull errors, OOMKilled, Liveness probe failed, readiness probe failed
# - Conditions: Ready=False + reason
# - Last State: exit code (for crash loops)

# Common exit codes:
# 0   = success
# 1   = application error
# 137 = OOMKilled (128 + SIGKILL)
# 143 = SIGTERM received (graceful shutdown, likely readiness race)

kubectl logs <failing-pod> --previous   # Logs from crashed container
kubectl logs <failing-pod> -c <init-container>  # Init container logs

# ── Phase 3: Readiness probe investigation ──
# What is the probe?
kubectl get deployment/<name> -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}'

# Manually test probe from inside cluster
kubectl exec <healthy-pod> -- curl -v http://<failing-pod-ip>:<port>/health/ready
# Does it return 200? If not → app not ready, probe is correct
# If 200 → probe misconfigured (wrong port, wrong path, wrong threshold)

# ── Phase 4: Resource constraints ──
kubectl describe pod <failing-pod> | grep -A5 "Limits\|Requests"
kubectl top pod <failing-pod>          # Actual CPU/memory usage
# Pod hitting CPU limits → throttled → slow startup → probe times out before app ready
# Pod hitting memory limits → OOMKilled → CrashLoop

# ── Phase 5: Dependency issues ──
# Is the app failing because a dependency (DB, config, secrets) isn't available?
kubectl exec <failing-pod> -- nslookup <db-service>    # DNS resolution
kubectl exec <failing-pod> -- nc -zv <db-host> 5432    # TCP connectivity
kubectl exec <failing-pod> -- env | grep DB_           # Env vars populated?

kubectl get secret <required-secret>   # Does secret exist?
kubectl get configmap <required-cm>    # Does configmap exist?

# ── Phase 6: Decide — rollback or proceed ──
# Rollback immediately if:
# - CrashLoopBackOff (app broken)
# - OOMKilled (memory regression)
# - Configuration error

kubectl rollout undo deployment/<name>
kubectl rollout status deployment/<name>   # Monitor rollback

# Pause if need more time to diagnose:
kubectl rollout pause deployment/<name>    # Halts controller — keeps current state
kubectl rollout resume deployment/<name>  # Resume when ready

# ── Phase 7: Post-fix — update and retry ──
kubectl set image deployment/<name> <container>=<image>:<fixed-tag>
# Or apply updated manifest:
kubectl apply -f deployment.yaml
```

---

### 🔍 Follow-up Question (Level 1)

**Q: The app logs look healthy — no errors, no exceptions. The readiness probe returns 200 when curled manually. But Kubernetes still marks the pod NotReady. Why?**

**A:**
This is a probe configuration mismatch, not an application issue. The probe is configured correctly in intent but incorrectly in parameters.

Possibilities:

1. **Wrong probe port or path in spec:**
   ```bash
   kubectl get pod <pod> -o yaml | grep -A10 readinessProbe
   # Compare: containerPort vs probe port. Common mistake: app on 8080, probe on 80
   ```

2. **Probe timeoutSeconds too short:**
   - App is slow to respond to `/health` (e.g., 2s) but `timeoutSeconds=1`
   - Probe times out → fails → pod marked NotReady
   - Manually testing with curl doesn't have this timeout
   ```yaml
   readinessProbe:
     timeoutSeconds: 5    # Increase from default 1
     periodSeconds: 10
     failureThreshold: 3
   ```

3. **initialDelaySeconds too short:**
   - App starts in 10s but `initialDelaySeconds=5`
   - Probe fires at 5s, app isn't ready, marks NotReady, then never recovers because `failureThreshold` already tripped

4. **Probe hits a different container port than what's being tested:**
   - Multi-container pod; probe hitting sidecar's health endpoint which always returns 200
   - But the main container on a different port is actually broken

5. **Network policy blocking probe traffic:**
   - Kubelet sends probe from node IP (not pod IP)
   - NetworkPolicy that restricts ingress to pod may block kubelet's source IP
   ```bash
   kubectl describe networkpolicy -n <namespace>
   # Check: does any policy deny traffic from node CIDR?
   ```

---

### 🔥 Follow-up Question (Level 2 — Edge Case)

**Q: The deployment stalls with 50% of pods on new version, 50% on old. Old pods refuse to terminate — they're stuck in Terminating state indefinitely. What's happening?**

**A:**
`Terminating` indefinitely = pod received SIGTERM but didn't exit within `terminationGracePeriodSeconds`, and then received SIGKILL but the container STILL didn't exit. OR the pod has a finalizer blocking deletion.

**Cause 1: Finalizer blocking deletion**
```bash
kubectl get pod <pod-name> -o yaml | grep finalizers
# If finalizers list is non-empty → pod won't be garbage collected until finalizer is cleared
# Usually: service mesh sidecar (Istio, Linkerd) that registers finalizers
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
# USE WITH CAUTION — removes finalizers, forcing deletion
```

**Cause 2: App ignoring SIGTERM (long graceful shutdown)**
- App caught SIGTERM and is draining: finishing in-flight requests
- Default `terminationGracePeriodSeconds=30` — if drain takes >30s, SIGKILL is sent
- Container receives SIGKILL but is in uninterruptible sleep (D state) due to stuck I/O
```bash
kubectl describe pod <pod> | grep "Termination Grace"
kubectl exec <pod> -- kill -0 1   # Is PID 1 alive? (before SIGKILL)
```
Fix: increase `terminationGracePeriodSeconds` OR implement faster drain logic

**Cause 3: Stuck in D state (uninterruptible sleep)**
- After SIGKILL, process is in D state waiting for kernel I/O (e.g., NFS mount hanging)
- Kernel cannot kill D-state processes
```bash
# On the node:
ps aux | grep <pod-process>   # D state?
# D state + NFS: check NFS mount responsiveness
mount | grep nfs
# Force unmount:
umount -l <nfs-mount>   # Lazy unmount
```

**Cause 4: Volume unmount failure**
- Pod deletion process tries to unmount PV before container terminates
- If PV unmount hangs (cloud disk detach API timeout) → pod stuck Terminating
```bash
kubectl describe pod <pod> | grep -A5 "Volumes"
kubectl get pv <pv-name>   # Check PV status
# On node: check kubelet logs for unmount errors
journalctl -u kubelet | grep <pod-name>
```

---

### 💡 Real-World Insight

**The Readiness Probe Thundering Herd:**
During a cluster upgrade, 50 pods of a cache-warming service were restarted simultaneously. Each pod's readiness probe hit the database to verify schema version. 50 simultaneous DB connections during startup saturated the connection pool. Probes timed out. All 50 pods stayed NotReady. The Deployment controller went into a loop, Kubernetes kept trying to reschedule, making it worse. Root cause: readiness probe was too aggressive (no backoff) + no connection pooling. Fix: add jitter to startup (random `initialDelaySeconds` between 5–30), reduce probe frequency, and use `pg_bouncer` in front of DB. Monitoring: alert on `kube_deployment_status_replicas_unavailable` metric in Prometheus.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 8 |
| Debugging Practicality | 9 |
| Failure Awareness | 9 |
| Clarity & Signal | 9 |

**Total: 44/50**
Strong finalizer and D-state coverage; loses points for not covering `preStop` hook interaction with termination and for not discussing Pod Disruption Budgets (PDB) which can block rolling deployments.

**🔍 Critical Feedback:**
- Pod Disruption Budgets (PDB) can block a rolling deployment completely if `minAvailable` is set too high — very common production footgun
- `preStop` hook + `terminationGracePeriodSeconds` interaction is subtle and often causes the "stuck terminating" mystery
- Should mention `kubectl rollout history` for understanding what changed between versions
- Istio sidecar injection during rolling deploy can cause readiness race — very common with service mesh

---

# 📋 Batch 1 Summary

| Q | Topic | Score |
|---|---|---|
| Q1 | L2 vs L3 Differentiation | 43/50 |
| Q2 | K8s Cross-Node Pod Comms | 45/50 |
| Q3 | HAProxy + Keepalived HA | 43/50 |
| Q4 | CPU Saturation | 45/50 |
| Q5 | File Descriptor Exhaustion | 45/50 |
| Q6 | K8s Rolling Deployment | 44/50 |

**Next Batch: Linux Internals — Process & Memory Management**
Topics: fork/exec, COW, zombie/orphan, syscalls, virtual memory, page faults, swap thrashing
