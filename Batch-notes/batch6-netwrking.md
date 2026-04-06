# Batch 6 — Networking Fundamentals
### Staff+ Level | Self-Interrogation + Spot Grading on Operational Questions
---

## Q41. What is the TCP 3-way handshake?

### 🏭 Production Context
The handshake is not just a textbook concept — it's a failure surface. In production, handshake failures manifest as: connection timeouts, half-open connection accumulation, SYN flood attacks, latency spikes under load. Every `connection refused`, `connection timed out`, and `connection reset` maps to a specific point in or around the handshake.

### ⚙️ Inner Workings

```
Client                              Server
  │                                   │
  │──── SYN (seq=X) ─────────────────>│  Client: SYNC_SENT
  │                                   │  Server: SYN_RCVD, allocates half-open slot
  │<─── SYN-ACK (seq=Y, ack=X+1) ────│
  │                                   │
  │──── ACK (ack=Y+1) ───────────────>│  Client: ESTABLISHED
  │                                   │  Server: ESTABLISHED
  │                                   │
  │<────── Data exchange ─────────────│
```

**Kernel state machine:**
- Server calls `listen(fd, backlog)` → socket enters LISTEN state
- SYN arrives → enters **SYN queue** (syn backlog / incomplete connections queue)
- SYN-ACK sent → client responds with ACK → moves to **Accept queue** (complete connections, waiting for `accept()` call)
- Application calls `accept()` → dequeues from accept queue → returns socket FD

**Two queues to understand:**
```
SYN queue:     net.ipv4.tcp_max_syn_backlog (default: 256)
               Half-open connections (SYN received, ACK not yet received)
               
Accept queue:  listen() backlog parameter (often 128 or 1024)
               Fully established connections waiting for accept()
```

### 🔍 Follow-up (Level 1)
**Q: SYN queue overflows during a traffic spike. What happens to new connections? What does the application see?**

**A:**
When SYN queue is full:
- Default behavior (without SYN cookies): kernel silently drops incoming SYNs
- Client doesn't receive any response → times out based on TCP retransmit timer (SYN retransmit: 1s, 3s, 7s, 15s... up to ~75 seconds total)
- Application sees: connection hanging then "Connection timed out"

**SYN cookies** (enabled by default on modern Linux): When SYN queue fills, kernel encodes state into the SYN-ACK's sequence number (cryptographic cookie) instead of keeping state in the queue. If client responds with valid ACK (proving it's real): connection established without queue entry.

```bash
# Check SYN cookie activation
cat /proc/sys/net/ipv4/tcp_syncookies   # 1 = enabled, 2 = always on

# See SYN queue overflow
netstat -s | grep -i "syn"
# "SYNs to LISTEN sockets dropped" → SYN queue overflowed

ss -lnt   # Listen sockets: Recv-Q = accept queue depth, Send-Q = max backlog

# Tune:
sysctl -w net.ipv4.tcp_max_syn_backlog=65536    # SYN queue
# In application: listen(fd, 65536)              # Accept queue
sysctl -w net.core.somaxconn=65536              # System-wide accept queue limit
```

**SYN cookie limitation:** stateful TCP options (window scaling, SACK) can't be stored in the cookie → some TCP performance features degrade for cookied connections.

### 🔥 Follow-up (Level 2)
**Q: Accept queue is full (backlog exhausted). Client sends ACK completing the handshake. Server kernel has fully established state. What does the server send back?**

**A:**
When accept queue is full and a new connection completes (ACK received):
- Kernel silently drops the ACK
- From client's perspective: connection is ESTABLISHED (it sent ACK and considers connection open)
- Server never calls `accept()` for it
- Client sends data → server responds with RST (connection not accepted)
- Client sees: connection established, first send succeeds, then receives RST

OR:
- If `tcp_abort_on_overflow = 1` (disabled by default): kernel sends RST when accept queue full → client gets immediate "Connection reset"
- Default (`tcp_abort_on_overflow = 0`): kernel drops → client retries → if server accept queue drains → connection eventually accepted

```bash
# See accept queue overflow
netstat -s | grep "times the listen queue of a socket overflowed"
ss -lnt | awk '{print $2, $3, $4}'   # Recv-Q > Send-Q → accept queue full

# Tuning:
# Increase accept queue: raise listen() backlog in application
# Increase system max: sysctl -w net.core.somaxconn=65536
```

### 💡 Real-World Insight
An Nginx upstream had `listen 80 backlog=128` (default). During a traffic spike (50k req/s): accept queue filled in milliseconds. New connections hung for seconds then timed out. `netstat -s | grep overflow` showed 100k drops/minute. Fix: `listen 80 backlog=65535` in Nginx + `net.core.somaxconn=65535`. Zero code change, 10-second deploy. P99 latency went from 5s to 50ms. Lesson: listen backlog is tuned almost nowhere by default.

---

## Q42. What is the difference between TCP and UDP?

### ⚙️ Inner Workings

**TCP (Transmission Control Protocol):**
- Connection-oriented: 3-way handshake required before data exchange
- Reliable: every segment acknowledged; lost segments retransmitted
- Ordered: sequence numbers ensure in-order delivery
- Flow control: receiver's window size limits sender rate (prevents overwhelming receiver)
- Congestion control: slow start, AIMD (Additive Increase Multiplicative Decrease) — adapts to network capacity
- Error detection: checksum on header + data
- Cost: connection state (~70+ bytes kernel struct per socket), handshake RTT before first data

**UDP (User Datagram Protocol):**
- Connectionless: send and forget, no handshake
- Unreliable: no acknowledgment, no retransmission
- Unordered: packets may arrive out of order
- No flow/congestion control: sender can blast at any rate (good for real-time, bad for network health)
- Minimal header: 8 bytes vs TCP's 20+
- Cost: essentially zero state per "connection" (just socket FD)

**Kernel-level:**
- TCP socket: `SOCK_STREAM`, kernel maintains `tcp_sock` with full state machine
- UDP socket: `SOCK_DGRAM`, kernel maintains `udp_sock` — much simpler, no state machine

```bash
# TCP: stateful
ss -tn   # Shows: ESTABLISHED, SYN_SENT, TIME_WAIT, CLOSE_WAIT, etc.

# UDP: stateless
ss -un   # Shows sockets but no state (UDP has no connection state)
```

### 🔍 Follow-up (Level 1)
**Q: TCP has congestion control. UDP doesn't. Why does this matter for your infrastructure?**

**A:**
TCP's congestion control is altruistic — it backs off when it detects loss (network congestion signal). Multiple TCP flows automatically share bandwidth without explicit coordination.

UDP has no such mechanism. A single UDP sender can saturate 100% of network bandwidth indefinitely, starving all other traffic.

Production implications:
1. **UDP-based services (DNS, syslog, metrics) under load can starve other traffic:** a metrics agent sending UDP at 1Gbps fills the interface buffer → TCP ACKs drop → TCP retransmits → all TCP services degrade
   ```bash
   # See UDP send errors (dropped because buffer full):
   netstat -su | grep -i "send buffer"
   cat /proc/net/snmp | grep Udp   # UdpSndBufferErrors
   ```

2. **QUIC (HTTP/3) implements congestion control in userspace:** QUIC runs over UDP but implements its own congestion control (similar to CUBIC). Gets reliability without kernel TCP overhead.

3. **DNS amplification DDoS:** attacker sends small UDP queries spoofing victim IP; DNS resolver sends large UDP responses to victim → amplification attack. Not possible with TCP (handshake required).

---

## Q43. When should you use UDP over TCP?

### 🏭 Production Context
The right answer isn't "UDP for speed" — it's "UDP when latency matters more than reliability AND when you can tolerate or handle loss in your application layer."

### ⚙️ Decision Framework

**Use UDP when:**

1. **Real-time, latency-sensitive, loss-tolerant:**
   - Video/audio streaming: a dropped frame is better than a delayed one (TCP retransmit = freeze)
   - Online games: stale position data is worthless; latest position is what matters
   - VoIP: packet loss = brief audio glitch; TCP retransmit = hundreds of ms of silence

2. **Single request-response, small payload (DNS model):**
   - DNS: query fits in 1 UDP datagram; if lost, client retries after timeout
   - Lower latency than TCP (no handshake for 1-shot operations)
   - Note: DNS also uses TCP for large responses (>512 bytes without EDNS0)

3. **Broadcast/multicast:**
   - Service discovery: one UDP broadcast reaches all listeners
   - TCP requires individual connection to each recipient

4. **High-throughput telemetry where loss is acceptable:**
   - Metrics (statsd over UDP): losing 0.1% of counter samples doesn't matter
   - Syslog (RFC 5424 UDP): losing an occasional log line acceptable vs TCP overhead

5. **When implementing custom reliability (QUIC, SCTP):**
   - You need reliability but also multiplexing, 0-RTT, or other features TCP doesn't have
   - Build reliability at application layer over UDP

**Use TCP when:**
- Data must be complete (file transfers, API calls, database queries)
- Order matters (any transactional operation)
- You can't afford packet loss (billing, medical, financial data)

---

## Q44. What happens when you run `curl google.com`?

### 📊 Grading Applied (Operational — Classic Interview Question)

### 🏭 Production Context
This question covers the entire network stack. In production, a failure in any step causes a different symptom. Knowing the exact sequence lets you pinpoint: "DNS lookup failed" vs "TCP reset" vs "TLS handshake failed" vs "HTTP 301 redirect" vs "response routing issue."

### ⚙️ The Complete Journey

**Step 1: DNS Resolution**
```
curl parses URL: "google.com" → hostname to resolve
→ Check /etc/hosts (no entry typically)
→ Check nscd/systemd-resolved cache (if configured)
→ Query /etc/resolv.conf nameserver (e.g., 8.8.8.8) via UDP port 53
→ DNS hierarchy:
    Root nameserver → .com TLD nameserver → Google's authoritative nameserver
→ Response: google.com → 142.250.80.46 (A record)
→ Also queries for AAAA (IPv6) typically
```

**Step 2: TCP Connection**
```
curl calls connect(fd, sockaddr{142.250.80.46, port 80}, ...)
→ 3-way handshake: SYN → SYN-ACK → ACK
→ RTT to Google: ~10ms (depends on location)
→ Socket now ESTABLISHED
```

**Step 3: HTTP Request**
```
curl sends:
GET / HTTP/1.1
Host: google.com
User-Agent: curl/7.81.0
Accept: */*

→ write() syscall → kernel TCP buffer → NIC → wire → Google server
```

**Step 4: HTTP Response**
```
Google sends: HTTP/1.1 301 Moved Permanently
Location: https://www.google.com/

→ curl follows redirect (with -L flag, or by default in modern curl)
→ New request to www.google.com
→ Port 443 (HTTPS) → TLS handshake added
→ TLS: ClientHello → ServerHello + Certificate → verify → session keys
→ HTTP GET over TLS → HTML response
```

**Step 5: Kernel network path (for every packet)**
```
send():
  Application buffer → socket buffer (tcp_sndbuf)
  → TCP segmentation → IP header added → routing lookup
  → ARP for gateway MAC (if not cached)
  → DMA to NIC buffer → wire

recv():
  Wire → NIC DMA → kernel ring buffer
  → Softirq: parse headers → deliver to socket buffer (tcp_rcvbuf)
  → Application read() → copy to userspace
```

```bash
# Trace the actual journey:
strace curl google.com 2>&1 | grep -E "socket|connect|sendto|recvfrom|write|read"

# DNS query:
tcpdump -i eth0 -nn 'udp port 53 and host 8.8.8.8'

# TCP handshake:
tcpdump -i eth0 -nn 'tcp and host google.com' | head -10

# Timing breakdown:
curl -w "\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
     -o /dev/null -s https://google.com
```

### 🔍 Follow-up (Level 1)
**Q: `curl google.com` works from your laptop but fails from a production pod. Same DNS config. What do you check?**

**A:**
Same DNS config ≠ same network path. Pod is in a different network namespace, potentially behind different firewall rules, different routing.

```bash
# Inside the pod:
kubectl exec <pod> -- nslookup google.com   # DNS resolution works?
kubectl exec <pod> -- curl -v --connect-timeout 5 google.com 2>&1
# Parse: resolving... → connecting... → sending... → receiving...

# Is it DNS?
kubectl exec <pod> -- dig google.com @8.8.8.8   # Test specific resolver
kubectl exec <pod> -- cat /etc/resolv.conf        # What DNS server is configured?
# ClusterDNS in K8s: 10.96.0.10 (kube-dns service)

# Is it routing?
kubectl exec <pod> -- ip route show   # Default gateway correct?
kubectl exec <pod> -- traceroute google.com   # Where does it die?

# Is it firewall/NetworkPolicy?
kubectl describe networkpolicy -n <namespace>   # Any egress restrictions?
# K8s default: no NetworkPolicy = all egress allowed
# With NetworkPolicy: must have explicit egress allow

# Is it egress NAT?
# Pods need SNAT/masquerade to reach external IPs
iptables -t nat -L POSTROUTING -n -v | grep MASQUERADE
# If missing → pod traffic exits with pod IP, internet drops it (private IP)
```

### 🔥 Follow-up (Level 2)
**Q: DNS resolves. TCP connect succeeds (SYN-ACK received). But `curl` hangs after sending GET. What's happening?**

**A:**
Connect succeeded → handshake complete → connection ESTABLISHED. Data sent. Hang = one of:

1. **Firewall allows SYN-ACK (stateless) but blocks data packets:** stateless firewall that explicitly allows TCP port 80 SYN packets but has a FORWARD DROP rule for established traffic. Rare but real.
   ```bash
   tcpdump -i eth0 'tcp and host <dest>' -nn   # Do ACKs come back? Do PSH+ACK data frames get ACKed?
   ```

2. **HTTP proxy required but not configured:** corporate network requires CONNECT proxy for external traffic. SYN-ACK succeeds (routed), but GET /index.html is sent to a proxy that drops it.
   ```bash
   env | grep -i proxy
   curl --proxy http://proxy:8080 google.com
   ```

3. **TCP ACK received but data ACKs not arriving (asymmetric routing):** sender sent GET, server received it, server responds, response takes different route that's blocked.
   ```bash
   tcpdump on both sides:
   # Sender: SYN out, SYN-ACK in, ACK out, GET out (PSH+ACK) → no response → retransmits after timeout
   # Server: SYN in, SYN-ACK out, ACK in, GET in (data arrives) → response out → vanishes
   # → Response routing asymmetry; response blocked on return path
   ```

4. **MTU black hole:** GET request fits in first packet. Response is large (>MTU), fragmented. ICMP "fragmentation needed" messages blocked → client doesn't know to reduce MSS → large response packets dropped → hang.
   ```bash
   ping -M do -s 1472 <dest>   # Test DF bit with large packet
   ```

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

**🔍 Critical Feedback:**
- TLS handshake steps could go deeper (certificate chain validation, OCSP stapling, session resumption — all affect latency)
- Should mention Happy Eyeballs (RFC 8305): curl races IPv4 and IPv6 simultaneously, takes faster one
- Missing: `curl --resolve google.com:80:1.2.3.4` for bypassing DNS in debugging

---

## Q45. How does DNS resolution work?

### ⚙️ Inner Workings

**Resolution hierarchy (recursive resolution):**
```
Application: gethostbyname("api.service.com")
     ↓
nsswitch.conf: check /etc/hosts first, then DNS
     ↓
Stub resolver (libc): query /etc/resolv.conf nameserver (e.g., 192.168.1.1)
     ↓
Recursive resolver (e.g., your router or 8.8.8.8):
  Cache check → miss
  Query root nameserver (a-m.root-servers.net): 
    "Who handles .com?" → response: .com TLD nameservers
  Query .com TLD nameserver:
    "Who handles service.com?" → response: authoritative nameservers for service.com
  Query service.com authoritative nameserver:
    "What is api.service.com?" → response: 10.0.1.50 (A record), TTL=300
  Cache result for TTL seconds
     ↓
Return to stub resolver → return to application
```

**DNS record types:**
```
A:     hostname → IPv4 address
AAAA:  hostname → IPv6 address
CNAME: hostname → another hostname (alias)
MX:    domain → mail server hostname
NS:    domain → authoritative nameserver hostname
PTR:   IP → hostname (reverse DNS)
SRV:   service → host:port (used by K8s, etcd)
TXT:   arbitrary text (SPF, DKIM, verification)
SOA:   Start of Authority (zone metadata)
```

**In Kubernetes:**
- CoreDNS runs as a service (ClusterIP: 10.96.0.10 typically)
- Pod's `/etc/resolv.conf` points to CoreDNS
- Query `service-name.namespace.svc.cluster.local` → CoreDNS → ClusterIP
- CoreDNS also forwards external queries to upstream resolver

---

## Q46. How do you debug DNS issues?

### 📊 Grading Applied (Operational)

### 🔧 Debug Sequence

```bash
# ── Step 1: Confirm it's a DNS issue ──
# Test by IP (bypass DNS):
curl -v --resolve api.service.com:443:10.0.1.50 https://api.service.com/health
# If this works → DNS is the problem; if fails → something else

# ── Step 2: Basic resolution test ──
nslookup api.service.com              # Using system resolver
dig api.service.com                   # More detail (query time, TTL, server)
dig api.service.com @8.8.8.8         # Using specific resolver (bypass local)
dig api.service.com @192.168.1.1     # Using specific internal resolver

# Compare: system resolver vs 8.8.8.8
# Different answers → split-horizon DNS or caching issue

# ── Step 3: Check what resolver is being used ──
cat /etc/resolv.conf
cat /etc/nsswitch.conf   # Order: files dns → checks /etc/hosts first
# In containers:
cat /etc/resolv.conf | head -5   # nameserver and search domains

# ── Step 4: DNS server reachability ──
dig +time=2 api.service.com @<resolver-ip>
# If timeout → resolver unreachable
nc -zu <resolver-ip> 53   # UDP port 53 open?
tcpdump -i eth0 'udp port 53' -nn   # Are queries going out? Are responses coming back?

# ── Step 5: NXDOMAIN vs timeout vs SERVFAIL ──
dig api.service.com; echo "Status: $?"
# NXDOMAIN: name doesn't exist (check record, check zone)
# SERVFAIL: resolver couldn't get answer (auth server issue, DNSSEC validation failure)
# Timeout: resolver unreachable or dropped queries

# ── Step 6: Cache state ──
# Flush DNS cache:
systemd-resolve --flush-caches     # systemd-resolved
# or
/etc/init.d/nscd restart           # nscd
# or simply: wait for TTL

# Check current cache (systemd-resolved):
systemd-resolve --statistics       # Cache hits/misses

# ── Step 7: Kubernetes-specific ──
# CoreDNS health:
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod> | grep -E "error|SERVFAIL"

# Test resolution from inside a pod:
kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local
# Should resolve to ClusterIP of kubernetes service

# Check search domains (common K8s issue):
kubectl exec <pod> -- cat /etc/resolv.conf
# search default.svc.cluster.local svc.cluster.local cluster.local
# 5 dots rule: FQDN queries skip search domains
# "api.internal" → tries api.internal.default.svc.cluster.local first (5 queries before success)

# ── Step 8: ndots setting (K8s performance issue) ──
# Default ndots:5 in K8s pods means:
# Every short hostname triggers 5 DNS lookups before hitting the right one
# Fix for external domains:
# Use FQDNs with trailing dot: curl https://api.external.com./endpoint
# Or set ndots:1 in pod's /etc/resolv.conf (if your app doesn't need cluster DNS)
```

### 🔍 Follow-up (Level 1)
**Q: DNS resolution works from your debug pod but fails intermittently from the production pod. Same namespace. What do you check?**

**A:**
Same namespace, different behavior → pod-specific or timing-specific issue:

1. **ndots + search domain cascade:** production pod sending many DNS queries (high traffic) → CoreDNS connection limit hit → some queries dropped
   ```bash
   kubectl get pods -n kube-system | grep coredns
   kubectl top pods -n kube-system | grep coredns   # CoreDNS CPU/memory
   # CoreDNS at CPU limit → query drops → intermittent failures
   ```

2. **DNS query coalescing issue:** multiple pods querying same name simultaneously; CoreDNS deduplicates but a bug causes some drops

3. **UDP vs TCP threshold:** large DNS responses (>512 bytes without EDNS0, or >4096 bytes) require TCP. If pod's network policy blocks TCP port 53 → large responses drop → intermittent for names with many A records or long TXT records

4. **CoreDNS conntrack issue:** CoreDNS uses UDP; Linux conntrack tracks UDP "connections." Under high query rate, conntrack table fills → new queries dropped
   ```bash
   conntrack -S | grep insert_failed   # Conntrack drops?
   ```

5. **Node-level DNS caching:** some clusters run node-local DNS cache (NodeLocal DNSCache) — check if it's misconfigured on specific nodes

### 💡 Real-World Insight
A microservice made 50 external API calls per request. Each call triggered a DNS lookup (no client-side caching). At 1000 req/s: 50,000 DNS queries/s to CoreDNS. CoreDNS had 2 replicas with CPU limit 100m each. Queries backed up → timeouts → intermittent 503s. Fix: client-side DNS caching (TTL-respecting), CoreDNS horizontal scaling (4 replicas), and `ndots:1` to reduce search domain cascade queries from 5 to 1 per external hostname.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 8 |
| Debugging Practicality | 10 |
| Failure Awareness | 9 |
| Clarity & Signal | 9 |

**Total: 45/50**

**🔍 Critical Feedback:**
- DNSSEC validation failures (SERVFAIL) should be explored more — common in enterprise environments with misconfigured DNSSEC
- Missing `dig +trace` which shows the actual recursive resolution path — excellent for diagnosing authoritative server issues
- The ndots section is excellent; should also mention `dnsmasq` and `unbound` as common alternatives to systemd-resolved

---

## Q47. What is ARP and how does it work?

### ⚙️ Inner Workings

ARP (Address Resolution Protocol) maps IP addresses to MAC addresses within a Layer 2 network (same broadcast domain / VLAN).

**Before sending a packet to 192.168.1.10 on the local subnet:**
```
1. Kernel checks ARP cache: is 192.168.1.10 in arp table?
   → ip neigh show (or arp -n)
   
2. If not cached: broadcast ARP request:
   "Who has 192.168.1.10? Tell 192.168.1.1"
   → Ethernet frame: src=our_MAC, dst=FF:FF:FF:FF:FF:FF (broadcast)
   
3. Host with 192.168.1.10 responds: ARP reply (unicast)
   "192.168.1.10 is at AA:BB:CC:DD:EE:FF"
   
4. Kernel caches the mapping (typically 20 minutes, gc_stale_time)

5. Packet is now sent with correct dst MAC
```

**Gratuitous ARP:** a host sends an ARP reply without a request — announces its own IP/MAC mapping to update caches of others. Used by:
- Keepalived/VRRP during VIP failover (announcing new MAC for the VIP)
- VM migration (announcing new NIC MAC)

```bash
# View ARP cache
ip neigh show
arp -n

# ARP cache stats
ip -s neigh show   # Shows state: REACHABLE, STALE, FAILED

# Send gratuitous ARP manually (failover testing)
arping -U -I eth0 10.0.0.100   # -U = gratuitous ARP

# ARP debug: watch ARP traffic
tcpdump -i eth0 arp -nn
```

### 🔍 Follow-up (Level 1)
**Q: Two hosts are on the same subnet and can ping each other. But after 20 minutes, they can't. ARP cache shows FAILED. What happened?**

**A:**
ARP cache entry went STALE → kernel tried to refresh → no response → FAILED state.

Possible causes:
1. **Host went down:** the target host crashed or lost network. ARP probe sent but no reply.
2. **ARP rate limiting:** target host's kernel has ARP rate limiting configured:
   ```bash
   cat /proc/sys/net/ipv4/neigh/default/gc_stale_time   # 60s default before re-probe
   ```
3. **NIC hardware issue:** NIC not processing ARP requests at hardware/driver level
4. **Firewall blocking ARP:** iptables rule dropping ARP packets (ARP runs below IP, but `ebtables` can filter it)
   ```bash
   ebtables -L   # Layer 2 firewall rules (can drop ARP)
   ```
5. **Gratuitous ARP from migration:** VM migrated, new host responds on different NIC → ARP cache points to old MAC → FAILED when old NIC gone

```bash
# Test ARP specifically:
arping -I eth0 192.168.1.10   # Sends ARP request, shows MAC if response
# No response → ARP-level failure, not IP-level
```

---

## Q48. What is subnetting and CIDR?

### ⚙️ Inner Workings

**CIDR (Classless Inter-Domain Routing):** notation for expressing IP address ranges with a variable-length prefix mask.

`192.168.1.0/24`:
- Network address: 192.168.1.0
- Prefix length: 24 bits → subnet mask 255.255.255.0
- Usable hosts: 2^(32-24) - 2 = 254 (subtract network and broadcast addresses)
- Broadcast: 192.168.1.255
- Range: 192.168.1.1 – 192.168.1.254

**Subnet calculation:**
```
10.0.0.0/8   → 16,777,214 hosts (Class A scale)
10.0.0.0/16  → 65,534 hosts
10.0.0.0/24  → 254 hosts
10.0.0.0/28  → 14 hosts
10.0.0.0/30  → 2 hosts (point-to-point links)
10.0.0.0/32  → 1 host (loopback, specific host route)
```

**Production use:**
- VPC design: `/16` for VPC, `/24` for subnets per AZ
- Kubernetes pod CIDR: `10.244.0.0/16` (Flannel default) → each node gets `/24` = 254 pods
- Service CIDR: `10.96.0.0/12` → ClusterIPs allocated here

```bash
# Subnet calculation tool
ipcalc 192.168.1.0/24
# Shows: network, broadcast, host range, hosts count

# Determine if two IPs are on same subnet:
ip route get 192.168.1.50   # If via local → same subnet; via gateway → different subnet

# CIDR supernet: does IP fall in range?
python3 -c "import ipaddress; print('10.0.1.5' in ipaddress.ip_network('10.0.0.0/16'))"
```

### 🔍 Follow-up (Level 1)
**Q: Kubernetes node has pod CIDR `10.244.1.0/24`. Maximum pods per node = 254. But `kubectl describe node` shows `Allocatable pods: 110`. Why the discrepancy?**

**A:**
IP capacity ≠ Kubernetes pod limit. Multiple constraints:
1. **Kubernetes node pod limit:** `kubelet --max-pods` flag (default: 110). A deliberate soft limit — Kubernetes found 110 to be a stable operational limit (resource overhead, watch connections to API server, etc.)
2. **CNI allocation limit:** some CNI plugins (AWS VPC CNI) have additional limits based on instance type and ENI slots
3. **Resource limits:** pods that fit in IP space may not fit in CPU/memory/disk

The 254 IPs in `/24` is the theoretical max. The real limit is `min(ip_capacity, kubelet_max_pods, resource_capacity)`.

To change: `kubelet --max-pods=250` (if your node can handle it) + ensure pod CIDR is large enough.

---

## Q49. How does routing work in Linux/network?

### ⚙️ Inner Workings

When a packet is sent, the kernel must decide: which interface does it exit from, and what is the next hop (gateway)?

**Linux routing table (FIB — Forwarding Information Base):**
```bash
ip route show
# Output:
default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.100 metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100
10.244.0.0/16 via 10.0.0.1 dev flannel.1 # CNI-added route
```

**Route lookup (longest prefix match):**
- For destination IP, find route with longest matching prefix
- `10.0.1.5`: matches `10.244.0.0/16` (/16)? No. Matches `default` (/0)? Yes → via gateway
- `192.168.1.50`: matches `192.168.1.0/24` (/24)? Yes → direct, via eth0

**Route types:**
- `via X.X.X.X` → next hop gateway (L3 routing required)
- `dev eth0 scope link` → directly connected (same subnet, L2 forwarding)
- `default` → 0.0.0.0/0 → catch-all for unknown destinations

```bash
# Lookup: how would kernel route to a destination?
ip route get 8.8.8.8
# Shows: exact route, interface, source IP used

# Add/delete routes:
ip route add 10.0.2.0/24 via 192.168.1.1   # Add route
ip route del 10.0.2.0/24                    # Remove route

# Policy routing (multiple routing tables):
ip rule show                   # Routing policy rules
ip route show table 100        # Secondary routing table
```

### 🔍 Follow-up (Level 1)
**Q: Two routes to the same destination with different metrics. Which wins?**

**A:**
Lower metric = higher priority = preferred route. If both routes have same prefix length, the one with lower metric is selected.

```bash
# Example:
ip route add 10.0.0.0/8 via 192.168.1.1 metric 100  # Primary
ip route add 10.0.0.0/8 via 192.168.2.1 metric 200  # Backup
# → Traffic uses 192.168.1.1; if that route removed, fails over to 192.168.2.1

# ECMP (Equal Cost Multi-Path): same prefix, same metric → load balance:
ip route add 10.0.0.0/8 nexthop via 192.168.1.1 nexthop via 192.168.2.1
# → Kernel hashes flows (5-tuple) across both next-hops
```

ECMP is how Kubernetes multi-path routing works with some CNIs (Calico, Cilium in BGP mode).

---

## Q50. What is a default gateway?

### ⚙️ Inner Workings

The default gateway is the router that handles all traffic not matched by more specific routes. It's the "exit point" for your subnet to reach the rest of the network.

```bash
ip route show default
# default via 192.168.1.1 dev eth0
# → All unknown destinations: send to 192.168.1.1 via eth0

# The gateway must be:
# 1. On the same subnet (L2 reachable — or it would need its own route)
# 2. A router that knows how to forward beyond your subnet
```

**When there's no default gateway:**
- Host can only reach its directly connected subnets
- Any packet to an external IP → `ip route get 8.8.8.8` returns `RTNETLINK answers: Network is unreachable`
- Common in misconfigured containers or VMs

**Multiple default gateways (policy routing):**
```bash
# Source-based routing: traffic from eth0 uses gateway1, traffic from eth1 uses gateway2
ip rule add from 192.168.1.100 table 1   # Traffic from this IP → use table 1
ip route add default via 10.0.0.1 table 1 # Table 1's default gateway
```

---

## Q51–Q52. What is a firewall and stateful vs stateless?

### ⚙️ Inner Workings

**Firewall:** filters packets based on rules. In Linux: `iptables`/`nftables` (in kernel netfilter framework).

**Stateless firewall:**
- Evaluates each packet independently
- Rules based on: src/dst IP, port, protocol, TCP flags
- Cannot distinguish between valid TCP responses and spoofed packets
- Example: allow TCP dst port 80 → allows ANY packet to port 80, even SYN without prior handshake
- Fast: no state table maintenance

**Stateful firewall (conntrack):**
- Tracks TCP connection state (SYN_SENT, ESTABLISHED, CLOSE_WAIT)
- NEW: first packet of a connection
- ESTABLISHED: packets matching an existing tracked connection
- RELATED: related connections (FTP data channel)
- INVALID: doesn't match any tracked connection

```bash
# iptables stateful rules (conntrack):
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate NEW -p tcp --dport 80 -j ACCEPT
# → Only new TCP connections to port 80 and established connections pass

# Without stateful: allow established traffic explicitly:
iptables -A INPUT -p tcp --sport 80 -j ACCEPT  # Allows any packet from port 80, even spoofed

# View connection tracking table:
conntrack -L
conntrack -S   # Statistics: inserts, found, invalid, ignore

# Common production issue: conntrack table full:
sysctl net.netfilter.nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count
conntrack -S | grep insert_failed   # Drops due to table full
```

### 🔍 Follow-up (Level 1)
**Q: A firewall allows only ESTABLISHED connections inbound. A new service is deployed and can't receive any connections. What's wrong?**

**A:**
NEW connections are being blocked. Stateful firewall requires an explicit rule for NEW connections (the initial SYN). The `ESTABLISHED` rule only allows reply traffic.

```bash
iptables -L INPUT -n -v | grep ACCEPT
# If only: ACCEPT ... ctstate ESTABLISHED,RELATED → NEW is DROP (default policy DROP)
# Fix:
iptables -I INPUT 1 -p tcp --dport <new_service_port> -m conntrack --ctstate NEW -j ACCEPT
```

This is why adding a new service requires a firewall rule change — it's a new connection being initiated inbound.

---

## Q53. How does a packet travel from client to server?

### ⚙️ Inner Workings

Complete cross-internet journey:
```
Client (192.168.1.100, port 54321):

1. Application: connect(fd, "93.184.216.34:80")
2. Kernel builds TCP SYN segment
3. IP layer: src=192.168.1.100, dst=93.184.216.34
4. Routing: no specific route → default gateway 192.168.1.1
5. ARP: resolve 192.168.1.1 → MAC aa:bb:cc:dd:ee:ff
6. Ethernet frame: src=client_MAC, dst=gateway_MAC, ethertype=IPv4
7. NIC DMA → physical wire

Client's Router (192.168.1.1):
8. Decapsulate Ethernet frame
9. IP destination: 93.184.216.34 → route lookup → ISP uplink
10. NAT (SNAT/Masquerade): rewrite src IP from 192.168.1.100 → 203.0.113.5 (public IP)
    src port: 54321 → 54321 (or new port if collision)
    Track in NAT table: (203.0.113.5:54321 ↔ 192.168.1.100:54321)
11. Forward to ISP router

Internet transit (multiple router hops, BGP routing):
12. Each router: decrement TTL, route lookup, forward to next hop

Server (93.184.216.34):
13. Packet arrives: src=203.0.113.5:54321, dst=93.184.216.34:80
14. NIC receives → DMA to ring buffer → softirq → netif_receive_skb
15. iptables INPUT → ACCEPT
16. TCP layer: SYN → new socket in SYN queue
17. Send SYN-ACK

Return path:
18. src=93.184.216.34:80, dst=203.0.113.5:54321
19. Client's router receives → NAT lookup → rewrite dst to 192.168.1.100:54321
20. Forward to client
```

---

## Q54. What is NAT?

### ⚙️ Inner Workings

NAT (Network Address Translation) modifies IP/port headers as packets traverse a router, enabling:
- Multiple private hosts to share one public IP (SNAT/Masquerade)
- Traffic redirection to internal hosts (DNAT — port forwarding)

**SNAT (Source NAT / Masquerade):**
```
Private: 192.168.1.100:12345 → Public: 203.0.113.5:12345
Router tracks: (private:port ↔ public:port) in conntrack + NAT table
Return traffic: rewrites dst from public back to private
```

**DNAT (Destination NAT):**
```
External client → 203.0.113.5:80 → Internal server 192.168.1.10:8080
Router rewrites dst; server sees client IP, not router IP (unless double-NAT)
```

**In Kubernetes:**
```bash
# kube-proxy iptables mode: ClusterIP is DNAT
iptables -t nat -L KUBE-SERVICES -n | grep <service-ip>
# KUBE-SVC-XXX: DNAT rules for load balancing to pod IPs

# View NAT rules for a specific service:
iptables -t nat -L -n | grep <clusterIP>
```

### 🔍 Follow-up (Level 1)
**Q: A service behind NAT needs to know the real client IP. How?**

**A:**
1. **HTTP `X-Forwarded-For` header:** load balancer/proxy adds client IP in header. Application reads it. Requires trusted proxy.

2. **PROXY protocol:** HAProxy/Nginx can prepend TCP-level header with client IP before TLS → backend reads it. Works for any TCP protocol, not just HTTP.
   ```
   PROXY TCP4 203.0.113.5 192.168.1.10 54321 80
   ```

3. **tproxy (transparent proxy):** kernel feature where original client IP is preserved through NAT by policy routing. Complex to configure.

4. **AWS/GCP NLB:** Network Load Balancers can preserve source IP without X-Forwarded-For (L4 passthrough).

---

## Q55. Where can a packet drop in the network path?

### 🏭 Production Context
Packet loss manifests as TCP retransmits, increased latency, timeouts. Locating the drop point is the primary debugging task.

### ⚙️ Drop Points

```
Client NIC:
  → Send buffer overflow (socket buffer full: ENOBUFS/EAGAIN)
  → NIC TX queue overflow (ethtool -S eth0 | grep tx_dropped)

Client OS:
  → iptables DROP/REJECT
  → Routing: no route (ICMP net unreachable)
  → conntrack: table full (insert_failed)

Network path:
  → Switch: CAM table miss → flood/drop; STP blocking port
  → Router: TTL expired (ICMP TTL exceeded); routing loop
  → Firewall: ACL deny
  → MTU: packet too large, DF bit set → ICMP fragmentation needed (if blocked = silent drop)
  → QoS: traffic shaping queue drop (policing, RED)
  → Congestion: buffer bloat → tail drop or AQM (CoDel) drop

Server OS:
  → iptables DROP/REJECT
  → SYN queue full (silent drop or RST)
  → Accept queue full
  → Socket receive buffer full (ss -nt Recv-Q column)
  → conntrack: full

Server application:
  → Application not reading socket fast enough → buffer fill → eventual drop
```

```bash
# Client-side drops:
netstat -s | grep -E "failed|error|drop"
ip -s link show eth0   # NIC-level drops
ethtool -S eth0 | grep drop

# Server-side drops:
netstat -s | grep -E "SYN|accept|drop|overflow"
ss -nt | awk '{if ($2 > 0) print}'   # Recv-Q: app not reading fast enough

# Network path:
traceroute -n <dst>   # Find where TTL expires (packet hop limit)
mtr --report <dst>    # Sustained packet loss per hop
tcpdump on both ends: if TX shows packets but RX doesn't → drop between them
```

---

## Q56. Why does ping work but curl hangs?

*(Covered extensively in Q1's follow-up and Q44's follow-up. Summary:)*

**Root causes:**
1. Firewall: ICMP allowed, TCP blocked (different rules for different protocols)
2. MTU black hole: small ICMP ping fits; TCP data with large MSS triggers fragmentation that's blocked by firewall (ICMP Fragmentation Needed dropped)
3. Routing asymmetry: SYN goes one path, SYN-ACK returns different path that's firewalled
4. SYN backlog full: server accepts ICMP but TCP listen queue full → SYN dropped
5. Application not listening: `curl` to closed port → RST immediately (actually this doesn't hang, it fails fast)
6. HTTP proxy required: ICMP