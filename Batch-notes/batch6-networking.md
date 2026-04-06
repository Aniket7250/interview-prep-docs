# Batch 6 — Networking Fundamentals
### Staff+ Level | Self-Interrogation + Spot Grading on Operational Questions
---

## Q41. What is the TCP 3-way handshake?

### 🏭 Production Context
The handshake is not just a textbook concept — it's a live failure surface. SYN queue overflows, accept queue saturation, half-open connection accumulation, SYN flood attacks — all happen here. Every `connection timed out` and `connection reset` maps to a specific point in or around the handshake sequence.

### ⚙️ Inner Workings

```
Client                              Server
  │                                   │
  │──── SYN (seq=X) ─────────────────>│  Client: SYN_SENT
  │                                   │  Server: SYN_RCVD → enters SYN queue
  │<─── SYN-ACK (seq=Y, ack=X+1) ────│
  │                                   │
  │──── ACK (ack=Y+1) ───────────────>│  Both: ESTABLISHED
  │                                   │  Server: moves to accept queue
  │<──────── Data flows ──────────────│
```

**Two kernel queues — both are failure surfaces:**

| Queue | Controlled by | Overflow behavior |
|---|---|---|
| SYN queue (half-open) | `net.ipv4.tcp_max_syn_backlog` | Silent drop or SYN cookie |
| Accept queue (full connections) | `listen(fd, backlog)` + `net.core.somaxconn` | Drop ACK or send RST |

```bash
# View listen socket queue depths
ss -lnt
# Recv-Q = current accept queue depth
# Send-Q = max accept queue size (backlog)
# Recv-Q > 0 in steady state → application not calling accept() fast enough

# SYN queue overflow events
netstat -s | grep "SYNs to LISTEN"
# "X SYNs to LISTEN sockets dropped" → SYN queue overflow

# Accept queue overflow
netstat -s | grep "times the listen queue"

# Tune:
sysctl -w net.ipv4.tcp_max_syn_backlog=65536
sysctl -w net.core.somaxconn=65536
# In application: listen(fd, 65536)
```

### 🔍 Follow-up (Level 1)
**Q: SYN queue overflows. What is the difference in client behavior when SYN cookies are on vs off?**

**A:**
**Without SYN cookies (tcp_syncookies=0):**
- Kernel silently drops SYN when queue full
- Client receives nothing → TCP retransmit timer fires: 1s, 3s, 7s, 15s...
- Client waits up to ~75 seconds → `Connection timed out`
- Meanwhile new legitimate SYNs also dropped → service appears down

**With SYN cookies (tcp_syncookies=1, default):**
- Kernel encodes connection state into the SYN-ACK's ISN (initial sequence number) using a cryptographic hash
- No entry in SYN queue needed → queue "never fills" from cookie perspective
- Client responds with ACK → server decodes cookie, verifies → ESTABLISHED
- Downside: TCP options (SACK, window scaling, timestamps) can't be stored in the cookie → negotiated conservatively → reduced throughput for those connections

```bash
cat /proc/sys/net/ipv4/tcp_syncookies
# 0 = disabled, 1 = enable when queue full, 2 = always on

# Monitor cookie usage
netstat -s | grep cookies
# "X SYN cookies sent" → syn queue was/is full
# "X SYN cookies received" → cookies are being validated
```

### 🔥 Follow-up (Level 2)
**Q: Accept queue is full. Client's ACK arrives completing the handshake. Server has fully established TCP state. What does the server do with it?**

**A:**
When accept queue is full on ACK arrival:
- With `tcp_abort_on_overflow=0` (default): kernel **silently drops the ACK**
- Connection is ESTABLISHED on client side (client sent ACK, considers it open)
- Server has no record of accepting it
- Client sends data → server has no socket for it → sends RST
- Client sees: connect succeeds, first write appears fine, then RST

```bash
cat /proc/sys/net/ipv4/tcp_abort_on_overflow
# 0 = drop ACK (default — gives server time to drain queue)
# 1 = send RST immediately — faster failure for client, but more aggressive

# Monitor:
netstat -s | grep "overflowed"
ss -lnt   # Watch Recv-Q vs Send-Q on listen socket
```

With `abort_on_overflow=0`, there's actually a recovery mechanism: the server will periodically retransmit SYN-ACK to the client (thinking ACK was lost). If the accept queue drains in that window, the client resends ACK → connection established. This is why timeouts are preferable to RST for temporary burst overload.

### 💡 Real-World Insight
Nginx default `listen 80 backlog=511`. Under a flash traffic event at 80k req/s, accept queue saturated in ~6ms. Connections appeared to establish (client got ESTABLISHED) then immediately RST'd. Application logs showed clean responses. Only `netstat -s | grep overflow` revealed the drop. Fix: `listen 80 backlog=65535` + `net.core.somaxconn=65535`. Response: sub-10ms P99 restored.

---

## Q42. What is the difference between TCP and UDP?

### ⚙️ Inner Workings

**TCP — connection-oriented, reliable stream:**
- 3-way handshake before data exchange
- Every segment acknowledged; lost → retransmitted
- Sequence numbers enforce ordering
- Flow control (receiver window) + congestion control (CUBIC/BBR) adapt to network
- Header: 20–60 bytes; stateful per connection (`tcp_sock` in kernel)

**UDP — connectionless, unreliable datagram:**
- Send immediately, no handshake, no state
- No retransmission, no ordering, no congestion control
- Header: 8 bytes; minimal kernel state
- App layer must handle reliability if needed (QUIC does this)

**Kernel path comparison:**
```
TCP send:
  write() → tcp_sendmsg() → segmentation → congestion window check
  → retransmit queue → NIC

UDP send:
  sendto() → udp_sendmsg() → IP layer → NIC
  (no queue, no window check, no retransmit logic)
```

```bash
# TCP: stateful, shows connection state
ss -tn   # ESTABLISHED, TIME_WAIT, CLOSE_WAIT...

# UDP: stateless, just open sockets
ss -un   # No state — UDP sockets don't track connection state
```

### 🔍 Follow-up (Level 1)
**Q: UDP has no congestion control. What happens to TCP flows sharing a bottleneck link with a misbehaving UDP sender?**

**A:**
UDP saturates the link → router queue fills → router starts dropping packets. TCP's congestion control detects drops (or ECN marks) → halves its congestion window (AIMD) → backs off → sends less. UDP doesn't back off at all.

Result: TCP is starved. UDP gets majority of bandwidth. TCP throughput collapses. The UDP sender is the "greedy" flow — it benefits at the expense of all TCP flows.

Real-world: a metrics agent sending UDP at wire rate during a network incident caused all TCP-based services (API calls, database queries) to experience 60%+ packet loss. The UDP agent was trying to flush its buffer urgently — but made the incident far worse.

```bash
# Detect UDP-induced TCP starvation
netstat -s | grep "segments retransmitted"   # TCP retransmit count
netstat -s | grep "packets received" | grep udp  # UDP receive rate
# Rising TCP retransmits + high UDP traffic on same interface = starvation
```

---

## Q43. When should you use UDP over TCP?

### ⚙️ Decision Framework

**Use UDP when ALL of these are true:**
1. Latency matters more than reliability
2. Application can tolerate or handle packet loss
3. You control both ends (or use a standard protocol like DNS)

**Production UDP use cases:**

| Use Case | Why UDP |
|---|---|
| DNS queries | Single packet; client retries on loss; handshake RTT unacceptable |
| Video/audio streaming | Stale frame = useless; retransmit arrives too late anyway |
| Online games (position updates) | Latest state wins; old state is worthless |
| VoIP | Brief glitch < 200ms pause from retransmit |
| Metrics (statsd) | Losing 0.01% of counter increments is acceptable |
| QUIC / HTTP/3 | UDP + application-layer reliability + multiplexing without HOL blocking |
| Multicast / broadcast | One packet to many; TCP requires N individual connections |

**Never use raw UDP when:**
- Data must arrive completely (file transfer, API response, DB query)
- Order matters (any transactional flow)
- You can't handle loss in the application

### 🔍 Follow-up (Level 1)
**Q: QUIC runs over UDP but is reliable. Why not just use TCP?**

**A:**
QUIC solves problems TCP can't fix without breaking the internet:
1. **Head-of-line blocking:** TCP multiplexes streams over one connection. One lost packet blocks ALL streams waiting for retransmit. QUIC multiplexes streams independently — lost packet only blocks that stream.
2. **0-RTT connection resumption:** QUIC can send data on the first packet (no handshake RTT) using session resumption. TCP requires 1 RTT for handshake + 1 RTT for TLS = 2 RTTs before first byte.
3. **Connection migration:** QUIC connections are identified by a connection ID, not IP:port. Mobile client changes IP (WiFi → cellular) → QUIC connection survives. TCP connection breaks.
4. **Ossification:** middleboxes (firewalls, NATs) are hardcoded to parse and modify TCP headers. TCP can't add new features without breaking them. QUIC runs over UDP (middleboxes pass it through) → can evolve faster.

---

## Q44. What happens when you run `curl google.com`?

### 📊 Grading Applied (Operational — Classic Interview Question)

### 🏭 Production Context
This question maps the entire network stack. Every production network failure — DNS timeout, TCP reset, TLS error, routing asymmetry — is a failure at a specific step in this chain. Knowing the sequence lets you immediately narrow scope.

### ⚙️ The Complete Journey

**Step 1: URL Parsing**
```
curl parses: scheme=http, host=google.com, port=80, path=/
```

**Step 2: DNS Resolution**
```
gethostbyname("google.com") → glibc resolver
  → /etc/nsswitch.conf: files → dns
  → Check /etc/hosts: no match
  → Check resolver cache (nscd / systemd-resolved)
  → Query nameserver from /etc/resolv.conf (e.g., 8.8.8.8:53)
  → DNS query (UDP by default):
      Root NS → .com TLD NS → google.com authoritative NS
  → Response: A record = 142.250.80.46, TTL=300
```

**Step 3: TCP Connection**
```
socket(AF_INET, SOCK_STREAM, 0)    → create socket FD
connect(fd, {142.250.80.46, 80})   → 3-way handshake
  SYN → SYN-ACK → ACK
  RTT to Google: ~10ms
  Socket: ESTABLISHED
```

**Step 4: HTTP Request**
```
write(fd, "GET / HTTP/1.1\r\nHost: google.com\r\n...\r\n\r\n")
```

**Step 5: HTTP Response**
```
Google returns: HTTP/1.1 301 Moved Permanently
Location: https://www.google.com/

curl follows redirect (by default):
  → New DNS: www.google.com → 142.250.80.46
  → New TCP: connect to port 443
  → TLS handshake:
      ClientHello → ServerHello + Certificate
      → Verify cert chain against trusted CAs
      → ECDHE key exchange → session keys
      → TLS session ESTABLISHED
  → HTTP/2 (negotiated via ALPN in TLS)
  → GET / → 200 OK + HTML
```

**Step 6: Kernel network path (every packet)**
```
Outbound:
  App buffer → socket send buffer (tcp_wmem)
  → TCP segment → IP header (routing lookup) → ARP for gateway
  → DMA to NIC TX ring → wire

Inbound:
  Wire → NIC RX ring → DMA to kernel ring buffer
  → Softirq (NET_RX) → protocol parsing → socket recv buffer (tcp_rmem)
  → read() copies to app buffer
```

```bash
# Full timing breakdown:
curl -w "\nDNS:     %{time_namelookup}s\nConnect: %{time_connect}s\nTLS:     %{time_appconnect}s\nTTFB:    %{time_starttransfer}s\nTotal:   %{time_total}s\n" \
     -o /dev/null -s https://www.google.com

# Trace each layer:
strace -e trace=socket,connect,sendto,recvfrom,read,write curl google.com 2>&1 | head -30
tcpdump -i eth0 -nn 'host google.com' -w /tmp/curl.pcap
wireshark /tmp/curl.pcap   # Full protocol decode
```

### 🔍 Follow-up (Level 1)
**Q: `curl google.com` works from your laptop but hangs from a production pod. DNS resolves. TCP connect succeeds. What do you check next?**

**A:**
DNS works, TCP ESTABLISHED, hang = data path issue. Checklist:

```bash
# Inside the pod:
kubectl exec <pod> -- curl -v --max-time 10 google.com 2>&1

# 1. Does the GET request leave the pod?
tcpdump -i eth0 -nn 'host google.com and tcp'
# Sender: SYN out ✓, SYN-ACK in ✓, ACK out ✓, GET out ✓
# If GET goes out but no response → return path blocked

# 2. NetworkPolicy blocking egress?
kubectl describe networkpolicy -n <ns>
# Default: no policy = all egress allowed
# With Calico/Cilium: must have explicit egress allow for 0.0.0.0/0

# 3. SNAT/masquerade missing?
# Pod IP is private (10.x.x.x). External hosts can't route to it.
# Without masquerade, return traffic has no path back.
iptables -t nat -L POSTROUTING -n -v | grep MASQUERADE
# Missing → packets leave with pod IP, internet drops response

# 4. MTU issue?
kubectl exec <pod> -- ping -M do -s 1400 google.com
# If fails but small ping works → MTU mismatch (overlay adds headers)

# 5. HTTP proxy required in environment?
kubectl exec <pod> -- env | grep -i proxy
```

### 🔥 Follow-up (Level 2)
**Q: `curl -v` shows connection ESTABLISHED and GET sent, but no response arrives. `tcpdump` on the pod's node shows the GET leaving the node. Where do you look next?**

**A:**
Packet left the node. Doesn't return. Drop is somewhere between node and Google (or on the return path).

```bash
# Step 1: Trace the outbound path
traceroute -n -T -p 80 google.com   # TCP traceroute on port 80
# Find where hops stop responding

# Step 2: Is it the overlay encapsulation?
# VXLAN adds ~50 bytes. If effective pod MTU misconfigured at 1500:
# Outer IP header (20) + UDP (8) + VXLAN (8) + Inner (1500) = 1536 > wire MTU 1500
# Fragmentation occurs. If upstream router blocks fragments → silent drop
tcpdump -i eth0 -nn 'icmp and icmp[0]=3'   # ICMP fragmentation needed messages

# Step 3: Asymmetric routing
# Packet leaves via eth0. Response comes back to different interface (eth1).
# eth1 firewall policy drops return traffic.
ip route show   # Is there only one path out?
ip rule show    # Any policy routing?

# Step 4: Firewall egress policy on the node
iptables -L FORWARD -n -v    # FORWARD chain: does pod traffic get forwarded?
iptables -t nat -L -n -v | grep MASQUERADE   # SNAT present?

# Step 5: Packet capture on return path
# On the node's physical NIC — does the response packet ARRIVE?
tcpdump -i eth0 -nn 'src host 142.250.80.46 and dst port 12345'
# If yes → response arrives at node but doesn't reach pod → routing inside node
# If no → response never arrives → upstream issue
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
- Missing Happy Eyeballs (RFC 8305): curl races IPv4/IPv6 simultaneously — understanding this explains why `curl` is sometimes faster than `wget` for dual-stack hosts
- TLS section omits OCSP stapling and session tickets — relevant to why HTTPS to the same server is faster on second request
- Should mention `curl --resolve google.com:443:1.2.3.4` as a debugging technique to bypass DNS entirely

---

## Q45. How does DNS resolution work?

*(Covered within Q44 — supplementary depth below)*

### ⚙️ Key Details

**Recursive vs Iterative resolution:**
- Your stub resolver (glibc) sends to a **recursive resolver** (8.8.8.8, your router)
- The recursive resolver does **iterative lookups**: root → TLD → authoritative
- It caches results for TTL duration — you benefit from its cache

**TTL production impact:**
- Low TTL (60s): rapid failover — change DNS record, propagates in 1 minute
- High TTL (3600s): better cache hit rate, less DNS traffic, but slow failover
- During incidents: pre-lower TTL before planned failover, then execute change

**DNS in Kubernetes:**
```
Pod resolves "myservice" →
CoreDNS search path: myservice.default.svc.cluster.local → found → ClusterIP

Pod resolves "api.external.com" →
CoreDNS: not cluster-local → forward to upstream resolver (8.8.8.8)
```

```bash
# ndots:5 (K8s default): short names get search domain appended
# "api" → api.default.svc.cluster.local (1st attempt) → NX
#        → api.svc.cluster.local (2nd) → NX
#        → api.cluster.local (3rd) → NX
#        → api.ec2.internal (4th) → NX  
#        → api (5th, absolute) → NX → FAIL
# For external FQDNs: 5 queries before success!

# Fix: always use FQDN with trailing dot for external names
curl https://api.external.com./  # trailing dot = absolute, skips search domains
# Or set ndots:1 in pod resolv.conf
```

---

## Q46. How do you debug DNS issues?

### 📊 Grading Applied (Operational)

### 🔧 Debug Sequence

```bash
# ── Step 1: Bypass DNS to confirm it's actually DNS ──
curl --resolve api.service.com:443:10.0.1.50 https://api.service.com/health
# Works → DNS is the problem
# Fails → not a DNS problem

# ── Step 2: Basic resolution test ──
nslookup api.service.com            # System resolver
dig api.service.com                 # Detailed (TTL, server, query time)
dig api.service.com @8.8.8.8       # Test specific resolver
dig +short api.service.com          # Just the IP

# ── Step 3: Trace the full resolution path ──
dig +trace api.service.com          # Shows every delegation step: root → TLD → auth
# Where does it fail/return wrong answer?

# ── Step 4: Understand what the system is using ──
cat /etc/resolv.conf               # Nameserver + search domains + ndots
cat /etc/nsswitch.conf             # Resolution order (files dns mdns?)

# ── Step 5: Query reachability ──
nc -zuv <nameserver-ip> 53         # UDP port 53 open?
tcpdump -i eth0 'udp port 53'      # Are queries going out? Responses coming back?
# Query goes out, no response → resolver unreachable or dropping queries

# ── Step 6: Parse the DNS error ──
dig api.service.com | grep -E "status:|ANSWER"
# NOERROR + empty ANSWER → name exists but no A record
# NXDOMAIN → name doesn't exist at all
# SERVFAIL → resolver couldn't get authoritative answer (auth server down, DNSSEC fail)
# REFUSED → resolver refused to answer (access control)
# Timeout → resolver unreachable

# ── Step 7: Kubernetes-specific ──
# CoreDNS health
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50 | grep -E "SERVFAIL|error|plugin"

# Resolution from inside pod
kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local
# Must resolve to ~10.96.0.1

# CoreDNS metrics (if prometheus exposed)
kubectl exec -n kube-system <coredns-pod> -- wget -qO- localhost:9153/metrics | grep coredns_dns_requests_total

# ── Step 8: ndots cascade causing latency ──
kubectl exec <pod> -- time nslookup api.external.com
# Slow? → 5 failed search domain queries before hitting the real answer
# Fix: trailing dot in config, or lower ndots in pod spec
```

### 🔍 Follow-up (Level 1)
**Q: DNS works from a debug pod but fails intermittently from the production pod. Same namespace. What do you check?**

**A:**
Intermittent failure under load = CoreDNS saturation or conntrack table overflow:

```bash
# 1. CoreDNS resource pressure
kubectl top pods -n kube-system | grep dns
# CPU-throttled CoreDNS → queries time out sporadically

# 2. conntrack table for DNS
# CoreDNS uses UDP. Conntrack tracks UDP flows.
# Under high DNS query rate: conntrack table fills → new queries dropped silently
conntrack -S | grep insert_failed   # Non-zero = drops
cat /proc/sys/net/netfilter/nf_conntrack_count   # Current
cat /proc/sys/net/netfilter/nf_conntrack_max     # Max

# 3. UDP socket buffer overflow
# High DNS query rate fills UDP receive buffer → CoreDNS drops queries
netstat -su | grep "receive buffer errors"
# Fix: increase CoreDNS replicas or use NodeLocal DNSCache

# 4. NodeLocal DNSCache (DaemonSet on each node)
# Each node runs a local cache daemon
# Benefits: bypass conntrack for DNS, reduce CoreDNS load
kubectl get pods -n kube-system | grep node-local-dns

# 5. Application DNS caching
# Services making DNS query per HTTP call with TTL=300:
# 1000 req/s × 1 DNS query = 1000 DNS queries/s just for external names
# Fix: client-side DNS TTL caching (Java: networkaddress.cache.ttl=30)
```

### 🔥 Follow-up (Level 2)
**Q: `dig` shows correct answer. Application still can't resolve. Why?**

**A:**
`dig` uses its own resolver, not glibc. Application uses `getaddrinfo()` via glibc. Mismatch sources:

1. **nsswitch.conf override:** `hosts: files mdns4_minimal [NOTFOUND=return] dns` — `[NOTFOUND=return]` means if mdns can't find it, DON'T try DNS. `dig` always uses DNS directly.

2. **nscd (name service cache daemon) serving stale entries:** app hits nscd cache (wrong answer) while `dig` queries DNS fresh.
   ```bash
   nscd -i hosts   # Flush nscd hosts cache
   ```

3. **`/etc/hosts` override:** `grep api.service.com /etc/hosts` — a static entry pointing to old IP.

4. **Application-level DNS cache:** Java's DNS cache (`networkaddress.cache.ttl`) may hold stale entry. Default Java cache: 30s (if SecurityManager present) or infinity (if not).
   ```bash
   # Force Java DNS flush: restart the JVM (no API to flush at runtime without JMX)
   # Better: set networkaddress.cache.ttl=30 in security.properties
   ```

5. **IPv6 / Happy Eyeballs:** `dig` queries A only. `getaddrinfo()` queries both A and AAAA. If AAAA returns a bad address and app prefers IPv6 → fails despite correct A record.
   ```bash
   dig AAAA api.service.com   # Check IPv6 record too
   ```

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

**🔍 Critical Feedback:**
- `dig +trace` should be the first tool mentioned for tracing delegation — it's the most powerful DNS debugging tool
- Should mention `systemd-resolve --status` for systems using systemd-resolved (most modern Ubuntu/RHEL)
- DNSSEC validation failures (SERVFAIL) are increasingly common — worth a specific callout

---

## Q47. What is ARP and how does it work?

### ⚙️ Inner Workings

ARP (Address Resolution Protocol) maps IP → MAC addresses within a broadcast domain (L2 segment). Without ARP, L3 packets can't be placed in L2 frames.

```
Scenario: Host A (10.0.0.1) wants to send to Host B (10.0.0.2), same subnet

1. Kernel checks neighbor table: ip neigh show
   → 10.0.0.2 not in cache (or STALE)

2. Kernel sends ARP Request (Ethernet broadcast):
   Who has 10.0.0.2? Tell 10.0.0.1
   Ethernet: src=AA:BB:CC:11:22:33, dst=FF:FF:FF:FF:FF:FF

3. Host B responds (ARP Reply, unicast):
   10.0.0.2 is at DD:EE:FF:44:55:66
   Ethernet: src=DD:EE:FF:44:55:66, dst=AA:BB:CC:11:22:33

4. Kernel caches: 10.0.0.2 → DD:EE:FF:44:55:66 (REACHABLE)
   Entry ages: REACHABLE → STALE → FAILED (if no refresh)
```

**ARP cache states:**
```bash
ip neigh show
# 10.0.0.2 dev eth0 lladdr DD:EE:FF:44:55:66 REACHABLE  ← fresh, recently verified
# 10.0.0.3 dev eth0 lladdr AA:BB:CC:11:22:33 STALE       ← not recently used, will probe on next use
# 10.0.0.4 dev eth0 FAILED                               ← probe sent, no reply
```

**Gratuitous ARP:** unsolicited ARP reply — host announces its own IP/MAC.
Used by: Keepalived VIP takeover, VM live migration, bonding failover.
```bash
arping -U -I eth0 10.0.0.100   # Send gratuitous ARP for VIP
# Forces all hosts on segment to update their ARP cache
```

### 🔍 Follow-up (Level 1)
**Q: ARP cache shows FAILED for a host that's definitely alive. `ping` also fails. But you can reach it by hostname. What happened?**

**A:**
Hostname resolution works → it resolves to a different IP than you expect. Classic causes:

1. **DNS returns different IP than you're pinging:** hostname resolves to a different subnet (or different VLAN), so ARP is sent on the wrong interface. The target is reachable via routing, but ARP is for direct L2 — wrong approach for routed paths.
   ```bash
   dig +short hostname   # What IP does hostname resolve to?
   ip route get <that-ip>   # Which interface would send to it?
   # ARP is only for same-subnet IPs — different subnet → route through gateway, not ARP
   ```

2. **Proxy ARP:** router between VLANs responds to ARP on behalf of hosts in other segments. Working because router answers. Direct ARP failing because host is on different segment.

3. **ARP filtering:** target host has `arp_ignore` or `arp_filter` set:
   ```bash
   cat /proc/sys/net/ipv4/conf/eth0/arp_ignore
   # 0 = respond to any ARP (default)
   # 1 = only respond if target IP is on incoming interface
   # 2 = only respond if target IP is on incoming interface AND source in same subnet
   ```

### 💡 Real-World Insight
During a Keepalived VIP failover, the backup sent one gratuitous ARP. Most hosts updated their ARP cache immediately. But 3 servers had `neigh.default.gc_stale_time=600` and cache entries were pinned to the old MASTER's MAC. For 10 minutes, those 3 servers sent traffic to the dead MASTER. Fix: send 3 gratuitous ARPs on failover with 100ms intervals + reduce `gc_stale_time` on critical infrastructure.

---

## Q48. What is subnetting and CIDR?

### ⚙️ Inner Workings

CIDR (Classless Inter-Domain Routing) notation: `IP/prefix-length`

```
10.0.0.0/24:
  Prefix: 10.0.0   (24 bits fixed)
  Host:   .0-.255  (8 bits variable) = 256 addresses, 254 usable
  Mask: 255.255.255.0
  Network: 10.0.0.0, Broadcast: 10.0.0.255
  Usable: 10.0.0.1 – 10.0.0.254

10.0.0.0/16: 65,534 usable hosts
10.0.0.0/8:  16,777,214 usable hosts
10.0.0.0/28: 14 usable hosts (point-to-point or small office)
10.0.0.0/30: 2 usable hosts (router-to-router links)
10.0.0.0/32: 1 host (loopback, exact host route)
```

**Production CIDR design (Kubernetes):**
```
VPC:          10.0.0.0/16     (65k IPs total)
├── AZ-1:     10.0.1.0/24    (254 IPs)
├── AZ-2:     10.0.2.0/24    (254 IPs)
├── AZ-3:     10.0.3.0/24    (254 IPs)
Pod CIDR:     10.244.0.0/16  (65k pod IPs)
  Per node:   10.244.X.0/24  (254 pods per node)
Service CIDR: 10.96.0.0/12   (ClusterIPs)
```

```bash
# Is IP in subnet?
python3 -c "import ipaddress; print('10.0.1.5' in ipaddress.ip_network('10.0.0.0/16'))"
# True

# Which subnet would kernel route to?
ip route get 10.0.1.5   # Shows: route, interface, src IP used

# CIDR calculator
ipcalc 10.0.0.0/22   # Shows: range, broadcast, hosts count, mask
```

### 🔍 Follow-up (Level 1)
**Q: Pod CIDR is `10.244.0.0/16`. Each node gets a `/24`. After how many nodes does the pod CIDR run out?**

**A:**
`/16` has 65,536 addresses. Each node gets a `/24` = 256 addresses.
65,536 / 256 = **256 nodes** before pod CIDR exhaustion.

In practice: Kubernetes cluster size planning must account for pod CIDR. A `/16` pod CIDR limits you to 256 nodes maximum (with `/24` per node). For larger clusters:
- Use `/14` for pod CIDR → 1024 nodes
- Or use `/24` per node (more pods per node)

AWS EKS uses VPC CNI (no overlay) — pod IPs come directly from VPC subnet. Each node can only host as many pods as it has ENI slots × IPs per ENI (instance-type specific). A `t3.medium` has 3 ENIs × 6 IPs = 18 pod IPs max. This is a hard limit regardless of `--max-pods`.

---

## Q49. How does routing work in Linux?

### ⚙️ Inner Workings

The kernel's FIB (Forwarding Information Base) is a trie structure indexed by destination prefix. For each outgoing packet, kernel does:

**1. Lookup: longest prefix match**
```bash
ip route show
# 10.244.0.0/16  via 10.0.0.1  dev flannel.1   ← /16 match
# 10.0.0.0/24    dev eth0      scope link       ← /24 match (direct)
# default        via 10.0.0.1  dev eth0         ← /0 match (fallback)

# For 10.244.1.5: matches /16 → via flannel.1
# For 10.0.0.50:  matches /24 → direct on eth0
# For 8.8.8.8:    matches /0 (default) → via 10.0.0.1 on eth0
```

**2. Next hop resolution**
- Direct (`scope link`): no gateway → ARP for destination MAC
- Via gateway: ARP for gateway MAC → gateway forwards further

**3. Source IP selection**
```bash
ip route get 8.8.8.8
# 8.8.8.8 via 10.0.0.1 dev eth0 src 10.0.0.100 uid 0
# "src 10.0.0.100" → kernel selected this as source IP for outgoing packets
```

```bash
# Manipulate routes:
ip route add 192.168.100.0/24 via 10.0.0.1    # Add static route
ip route del 192.168.100.0/24                  # Delete route
ip route flush cache                            # Clear route cache

# ECMP (load balance across multiple next-hops):
ip route add default nexthop via 10.0.0.1 weight 1 nexthop via 10.0.0.2 weight 1
# Kernel distributes flows across both gateways via 5-tuple hash
```

### 🔍 Follow-up (Level 1)
**Q: A new route is added. Existing TCP connections use the old path. When do they switch?**

**A:**
Existing connections don't automatically switch. TCP sockets cache the route result (`sk_dst_cache` in `sock` struct). Route lookup is done at `connect()` time, not per-packet.

For a running TCP connection to use the new route:
- Connection must be destroyed and recreated (new `connect()`)
- OR kernel route cache is invalidated by a `NETLINK` route change event → kernel marks cached route stale → next packet triggers re-lookup

In practice: route changes take effect immediately for **new** connections. Long-lived connections (DB, keep-alive HTTP) continue on old path until they reconnect.

```bash
# Force route cache refresh (happens automatically on route change via ip route):
ip route flush cache
# Existing connections that do next send will re-lookup route
```

---

## Q50. What is a default gateway?

### ⚙️ Inner Workings

The default gateway is the router that handles all traffic with no more-specific route. It's your subnet's exit point to the world.

```bash
ip route show | grep default
# default via 192.168.1.1 dev eth0 metric 100
# → Send all unmatched packets to 192.168.1.1 via eth0
```

**Requirements for the gateway:**
- Must be L2-reachable (same subnet OR accessible via L2 path) — kernel ARPs for it
- Must be a router capable of forwarding beyond your subnet
- If gateway is unreachable → packets to external destinations silently blackholed

**Common production issues:**
```bash
# No default gateway → external traffic fails
ip route get 8.8.8.8
# RTNETLINK answers: Network is unreachable

# Wrong gateway (points to non-router host)
# Packets reach the gateway IP, but that host doesn't forward → blackhole
# Symptom: ping gateway works, ping beyond fails

# Multiple default gateways (causes asymmetric routing):
ip route show | grep default
# default via 10.0.0.1 dev eth0
# default via 10.0.0.2 dev eth1
# → Outbound may go eth0, return may arrive eth1 → stateful firewall drops

# Fix asymmetric: use policy routing (one table per interface)
ip rule add from 192.168.1.x table 1
ip route add default via <gw1> table 1
```

### 🔍 Follow-up (Level 1)
**Q: Container has no default gateway configured. It can talk to services on the same node (via veth/bridge) but nothing external. Why and how do you fix it?**

**A:**
Same-node traffic: pod → veth → cbr0/cni0 bridge → other pod's veth. No routing needed — L2 bridging. Works without a gateway.

External traffic: pod → veth → bridge → iptables FORWARD → eth0 → gateway. Kernel needs a route for external IPs → no default route → `RTNETLINK: Network is unreachable`.

Fix: CNI plugin's job to inject default route into pod's network namespace:
```bash
# Inside pod's netns:
ip route show
# default via 10.244.1.1 dev eth0   ← CNI should have added this
# If missing → CNI misconfigured

# Manually add (diagnostic only):
nsenter --target <pod-PID> --net ip route add default via 10.244.1.1

# Permanent fix: restart CNI plugin on the node, or check CNI config:
cat /etc/cni/net.d/10-flannel.conflist
```

---

## Q51–Q52. Firewalls — Stateful vs Stateless

### ⚙️ Inner Workings

**Stateless firewall:**
- Evaluates each packet independently
- Rules: src/dst IP, port, protocol flags
- Cannot distinguish SYN from ACK-only, or legitimate response from spoofed packet
- Fast (no state table lookup), but less secure

**Stateful firewall (Linux: netfilter + conntrack):**
- Tracks connection state in `nf_conntrack` table
- States: NEW (first packet), ESTABLISHED (part of tracked conn), RELATED, INVALID

```bash
# Stateful iptables rules:
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate NEW -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -j DROP   # Default deny

# vs stateless (weaker):
iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # Allows ANY packet to 443
iptables -A INPUT -p tcp --sport 443 -j ACCEPT   # Allows "responses" — but easily spoofed

# View tracked connections:
conntrack -L
conntrack -L | grep 10.0.1.5

# Conntrack table size:
cat /proc/sys/net/netfilter/nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count

# Critical: table full = new connections silently dropped
conntrack -S | grep insert_failed   # Non-zero = drops happening
```

### 🔍 Follow-up (Level 1)
**Q: IPVS mode in kube-proxy uses less conntrack than iptables mode. How?**

**A:**
iptables kube-proxy: every service connection requires conntrack entries for:
1. DNAT (ClusterIP → PodIP)
2. SNAT (masquerade for return path)
3. The TCP connection itself

For a busy service with 10k concurrent connections: 30k+ conntrack entries per service.

IPVS (IP Virtual Server) operates at L4 in the kernel with its own connection table — independent of conntrack. IPVS tracks connections in `ipvs_conn` structures, not conntrack. Only masquerade (SNAT) still uses conntrack.

Result: with IPVS, conntrack entries ≈ 1 per connection (SNAT only) instead of 3 per connection (DNAT + SNAT + TCP). At 100k concurrent service connections: `~100k` conntrack entries vs `~300k` in iptables mode.

```bash
# Switch kube-proxy to IPVS:
kubectl edit configmap kube-proxy -n kube-system
# mode: "ipvs"

# View IPVS table:
ipvsadm -L -n
# Shows virtual services and real servers with connection counts
```

---

## Q53. How does a packet travel from client to server?

### ⚙️ Full End-to-End Path (Internet scenario)

```
1. CLIENT APPLICATION
   connect("93.184.216.34", 80) → socket FD

2. CLIENT KERNEL
   TCP SYN segment built
   IP header: src=192.168.1.100, dst=93.184.216.34
   Routing: no specific route → default gateway 192.168.1.1
   ARP: resolve 192.168.1.1 → MAC aa:bb:cc:11:22:33
   Ethernet frame assembled: dst MAC = gateway's MAC
   DMA to NIC TX ring → out the wire

3. HOME ROUTER (NAT gateway)
   Receives frame, strips Ethernet
   IP dst = 93.184.216.34 → Internet (not local) → route via ISP uplink
   SNAT: rewrite src IP 192.168.1.100 → 203.0.113.5 (public IP)
   Record in NAT table: (203.0.113.5:54321 ↔ 192.168.1.100:54321)
   New Ethernet frame with ISP gateway MAC → out to ISP

4. INTERNET TRANSIT (BGP routing, multiple hops)
   Each router: decrement IP TTL, route lookup (BGP), forward to next AS

5. TARGET SERVER (93.184.216.34)
   Frame arrives → NIC DMA → kernel ring buffer
   Softirq (NET_RX) processes: Ethernet → IP → TCP
   iptables INPUT → ACCEPT
   TCP: SYN → allocate socket → SYN queue
   Send SYN-ACK

6. RETURN PATH (SYN-ACK back to client)
   src=93.184.216.34:80, dst=203.0.113.5:54321
   Traverses internet in reverse (BGP)
   Arrives at client's router
   NAT lookup: 203.0.113.5:54321 → 192.168.1.100:54321
   Rewrite dst, forward to client

7. CLIENT receives SYN-ACK → sends ACK → ESTABLISHED
```

---

## Q54. What is NAT?

### ⚙️ Inner Workings

NAT rewrites IP/port headers in transit. In Linux: `iptables -t nat`.

**Types:**
```
SNAT (Source NAT):
  Outbound: src IP rewritten (private → public)
  Tracking: NAT table maintains (private:port ↔ public:port) mapping
  Return: dst rewritten back to private IP

DNAT (Destination NAT / Port Forwarding):
  Inbound: dst IP:port rewritten (public:port → private:port)
  Used for: exposing internal services, load balancing

MASQUERADE (dynamic SNAT):
  Like SNAT but uses interface's current IP automatically
  Better for DHCP environments (IP can change)
```

```bash
# View NAT rules:
iptables -t nat -L -n -v

# Kubernetes DNAT for services:
iptables -t nat -L KUBE-SERVICES -n | head -10
# Each line: ClusterIP → KUBE-SVC-xxxx chain → one of N pod IPs (probabilistic)

# Add port forward:
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.1.5:80
iptables -t nat -A POSTROUTING -j MASQUERADE
```

### 🔍 Follow-up (Level 1)
**Q: NAT breaks some protocols. Which ones and why?**

**A:**
Protocols that embed IP addresses in their payload (not just headers) break with NAT:
1. **FTP (active mode):** client sends its IP:port in data channel negotiation via `PORT` command in payload. NAT rewrites outer IP but not payload → server connects to private IP (unreachable). Fix: passive FTP, or `nf_nat_ftp` kernel module (NAT helper that rewrites payload).
2. **SIP (VoIP):** SDP body contains IP:port for media stream. NAT rewrites outer IP, not SDP payload → media goes to wrong IP.
3. **IPSec (ESP mode):** ESP packets don't have TCP/UDP ports → NAT can't track them properly. Solution: NAT-T (NAT Traversal) wraps ESP in UDP.

---

## Q55. Where can a packet drop in the network path?

### 🏭 Production Context
Packet drops produce different symptoms depending on location: client-side drops cause `ENOBUFS` errors; firewall drops cause silent timeouts; server-side drops cause RST. Knowing the location is the diagnosis.

### ⚙️ Complete Drop Map

```
CLIENT:
  NIC TX queue full      → ethtool -S eth0 | grep tx_dropped
  Socket send buffer     → ss -nt | awk '{if($3>0) print}'   (Send-Q full)
  iptables OUTPUT        → iptables -L OUTPUT -n -v

NETWORK:
  Switch: STP port blocking, CAM overflow, VLAN mismatch
  Router: TTL=0 (traceroute hop), no route (ICMP unreachable), ACL
  Firewall: stateless DROP, conntrack invalid state
  MTU: DF bit set, packet > MTU, ICMP Frag Needed dropped (black hole)
  QoS/policing: traffic shaper drops excess traffic
  Congestion: buffer bloat → tail drop / ECN mark

SERVER:
  iptables INPUT DROP    → iptables -L INPUT -n -v
  SYN queue full         → netstat -s | grep "SYNs to LISTEN dropped"
  Accept queue full      → netstat -s | grep "listen queue overflow"
  Socket recv buffer     → ss -nt | awk '{if($2>0) print}'   (Recv-Q)
  conntrack table full   → conntrack -S | grep insert_failed
```

```bash
# Systematic drop detection:
# Client NIC:
ip -s link show eth0 | grep -A3 RX:   # RX errors/drops
ethtool -S eth0 | grep -i drop

# Firewall drops with counters:
iptables -L -n -v | awk '$1 > 0 && /DROP/'   # Rules with >0 drops

# TCP-level drops:
netstat -s | grep -E "failed|dropped|reset|overflow|ignored"

# Packet loss per hop:
mtr --report --no-dns -c 100 <destination>
# Shows % loss at each hop — if loss starts at hop N, drop is between N-1 and N
```

---

## Q56. Why does ping work but curl hangs?

### 🏭 Production Context
This is the canonical "partial connectivity" mystery. The gap between ICMP and TCP reveals exactly which layer is broken.

### ⚙️ Root Cause Tree

**1. Firewall allows ICMP, blocks TCP:**
```bash
# Stateless ACL:
iptables -A INPUT -p icmp -j ACCEPT   # ping works
iptables -A INPUT -j DROP             # TCP blocked (no port-specific rule)

# Diagnosis:
tcpdump -i eth0 'tcp and host <target>'
# SYN goes out, no SYN-ACK → server-side firewall dropping TCP
```

**2. MTU black hole (most common in overlays):**
```bash
# ping uses small ICMP (~64 bytes) — fits in any MTU → succeeds
# TCP MSS negotiated at 1460 bytes → data packets ~1500 bytes
# Somewhere in path: MTU is 1400 (VPN, VXLAN overhead)
# Router drops packet, should send ICMP "Fragmentation Needed" → blocked by firewall
# TCP sender never learns of smaller MTU → keeps sending too-large packets → hang

# Test:
ping -M do -s 1472 <target>   # 1472 + 28 (IP+ICMP) = 1500 byte packet
# FAILED → MTU issue between you and target

# Fix: set TCP MSS clamp
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
```

**3. Routing asymmetry:**
```bash
# SYN: client → router A → server (success)
# SYN-ACK: server → router B → client (router B drops — different firewall)
# Client: ESTABLISHED (thinks SYN-ACK arrived — actually it didn't)
# curl: sends GET → server sees ACK, but it never saw SYN-ACK arrive → RST
# Looks like: ping works, curl instant RST or hang

# Debug:
traceroute -n <target>           # Outbound path
traceroute -n -I <target>        # ICMP-based (may take different path)
# If paths differ → asymmetric routing
```

**4. Server's SYN backlog full:**
```bash
# ping: ICMP — no connection queue, always processed
# curl: TCP SYN → SYN queue full → dropped → client retries → eventual timeout

netstat -s | grep "SYNs to LISTEN"   # On server
```

**5. Application-level issue (not network):**
```bash
# TCP connects fine (curl -v shows "Connected")
# Hang is after connection — server isn't responding to HTTP
# ping: ICMP processed by kernel (always works if host alive)
# HTTP: processed by application (may be hung, slow, deadlocked)

# Test:
nc -v <target> 80        # TCP connect test without HTTP
echo -e "GET / HTTP/1.0\r\n\r\n" | nc <target> 80   # Raw HTTP
# If nc connects and hangs → application issue, not network
```

### 🔍 Follow-up (Level 1)
**Q: `tcpdump` shows SYN leaving the client and SYN-ACK returning. But curl still hangs. What's wrong?**

**A:**
SYN-ACK arrives → connection ESTABLISHED from kernel perspective → curl sends GET → GET leaves client → server never responds.

This points to the POST-handshake path:
1. **GET packet gets lost:** SYN-ACK arrived (maybe via a different path). GET takes different path that's blocked. Asymmetric routing post-handshake.
   ```bash
   tcpdump 'tcp and host <target> and tcp[13] & 8 != 0'  # Filter PSH flag (data packets)
   # Does GET (PSH+ACK) appear on the wire?
   ```

2. **Server receives GET but blocks the response:** application firewall (WAF), rate limit, auth failure that causes silent drop at app layer.

3. **HTTP proxy issue:** GET is sent correctly, but destination expects `CONNECT` (HTTPS proxy) or `GET http://...` (HTTP proxy). Plain `GET /` without proxy formatting → proxy drops it.

4. **Virtual hosting:** curl sends `Host: <IP>` if no hostname is specified. Server requires `Host: correct.hostname.com` or it silently closes connection.
   ```bash
   curl -v -H "Host: expected.hostname.com" http://<ip>/
   ```

---

## Q57. What is MTU and fragmentation?

### ⚙️ Inner Workings

MTU (Maximum Transmission Unit) = maximum bytes a network interface can transmit in one frame.

Standard: `1500 bytes` (Ethernet). VXLAN overlay adds `50 bytes` header → effective MTU = `1450` for inner packets.

**IP fragmentation:**
When IP packet > MTU and DF (Don't Fragment) bit NOT set:
- Router fragments packet into smaller pieces
- Each fragment has its own IP header (same IP ID, fragment offset field set)
- Reassembled at destination IP layer
- Cost: extra CPU at router and destination, increased packet count

**PMTUD (Path MTU Discovery):**
When DF bit IS set (TCP always sets it):
- If packet > MTU of any hop: router drops and sends `ICMP Type 3 Code 4: Fragmentation Needed`
- Sender reduces MSS (Max Segment Size) based on ICMP message
- Allows discovery of smallest MTU across the entire path

**MTU Black Hole:**
- `ICMP Fragmentation Needed` messages blocked by firewall
- Sender never learns of smaller MTU
- Large data packets silently dropped
- SYN-ACK works (small), TCP session establishes, data transfer hangs

```bash
# Test PMTUD:
ping -M do -s 1472 <target>   # DF bit set, 1472+28=1500 byte packet
ping -M do -s 1422 <target>   # Test VXLAN-adjusted MTU

# Check interface MTU:
ip link show eth0 | grep mtu
# In Kubernetes:
ip link show flannel.1   # overlay interface MTU

# TCP MSS in captures:
tcpdump -i eth0 -nn 'tcp[13] & 2 != 0'   # SYN packets
# Look for: MSS=1460 (Ethernet), MSS=1410 (VXLAN), MSS=1360 (VPN)
```

### 🔍 Follow-up (Level 1)
**Q: Service works for small requests but hangs for large responses (>4KB). You suspect MTU. How do you confirm?**

**A:**
Asymmetric MTU issue — small request (SYN, GET) fits in any MTU → works. Large response → response packets exceed overlay MTU → dropped. Classic MTU black hole.

```bash
# Confirm: test with different ping sizes
ping -M do -s 1400 <server-ip>   # Should succeed
ping -M do -s 1450 <server-ip>   # May fail if overlay MTU < 1478 (1450+28)

# Curl test with response size control:
curl -o /dev/null -v "http://<service>/endpoint?size=100"   # Small response → OK
curl -o /dev/null -v "http://<service>/endpoint?size=5000"  # Large response → hang

# Network-level proof:
tcpdump -i eth0 'icmp and icmp[0]=3 and icmp[1]=4'
# ICMP Type 3 Code 4 = Fragmentation Needed
# Presence of these → MTU mismatch confirmed, but ICMP is reaching you
# Absence → ICMP is being blocked → black hole

# Fix options:
# 1. Correct pod MTU:
kubectl edit configmap kube-flannel-cfg -n kube-system
# Change: "MTU": 1450  (instead of 1500)

# 2. MSS clamping (iptables):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
# Forces TCP to negotiate correct MSS during handshake
```

---

## Q58. How do you debug MTU issues?

### 📊 Grading Applied (Operational)

### 🔧 Debug Sequence

```bash
# ── Step 1: Gather MTU info at each layer ──
ip link show              # Physical interface MTU
ip link show flannel.1    # Overlay interface MTU (if K8s)
ip link show tunl0        # IPIP tunnel MTU

# Expected:
# Physical (eth0): 1500
# VXLAN (flannel.1): 1450 (1500 - 50 byte VXLAN overhead)
# Pod eth0: should match flannel.1 (1450)

# ── Step 2: Ping with DF bit to test path MTU ──
# From pod/host to destination:
ping -M do -s 1400 <dest>   # Slightly below max — should work
ping -M do -s 1452 <dest>   # Right at limit — should work
ping -M do -s 1453 <dest>   # 1 byte over — should fail if MTU is 1450+28=1478 total

# Binary search for actual path MTU:
for size in 1400 1420 1440 1450 1460 1472; do
    result=$(ping -M do -s $size -c 1 <dest> 2>&1)
    echo "$size bytes: $(echo $result | grep -o 'unreachable\|transmitted')"
done

# ── Step 3: Check if ICMP fragmentation needed is being blocked ──
tcpdump -i eth0 -nn 'icmp and icmp[0]=3 and icmp[1]=4'
# If no ICMP type 3 code 4 but ping fails → ICMP blocked (black hole)

# ── Step 4: TCP MSS inspection ──
tcpdump -i eth0 -nn 'tcp[13] & 2 != 0 and host <dest>'
# SYN packets (TCP flag SYN set)
# Parse MSS: tcpdump -vv shows "options [mss 1460,...]"
# If MSS is larger than (path MTU - 40) → will cause drops

# ── Step 5: Confirm with curl ──
curl -o /dev/null -w "size: %{size_download}\n" http://<dest>/large-endpoint
# If hangs on >1 packet response → MTU confirmed

# ── Step 6: Fix ──
# Option A: Correct MTU on CNI (Flannel):
kubectl edit configmap kube-flannel-cfg -n kube-system
# "MTU": 1450

# Option B: MSS clamping (immediate, no restart):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
# This tells TCP to negotiate the correct MSS during handshake

# Option C: Per-host route with MTU hint:
ip route add <dest> via <gw> mtu 1450
```

### 🔍 Follow-up (Level 1)
**Q: You set pod MTU to 1450. But jumbo frames (MTU 9000) are enabled on the physical switch. Is 1450 still correct?**

**A:**
With jumbo frames (MTU 9000) on physical switches AND all hosts configured with `MTU=9000` on their physical NICs:

```bash
ip link set eth0 mtu 9000   # Set NIC to 9000
```

The overlay overhead (50 bytes VXLAN) becomes negligible:
- Physical MTU: 9000
- VXLAN overhead: 50 bytes
- Effective pod MTU: 8950

Setting pod MTU to 1450 when physical is 9000 wastes ~84% of available bandwidth by unnecessarily fragmenting. Larger MTU = fewer packets = less CPU = higher throughput.

**However:** all nodes in the path must support jumbo frames. One misconfigured hop at 1500 → silent fragmentation or drops for packets >1500 bytes.

```bash
# Verify jumbo frames end-to-end:
ping -M do -s 8000 <other-node-ip>   # Does 8000+28=8028 byte packet get through?
```

If yes: set pod MTU to 8950 (or 8900 with safety margin). If any hop fails → conservative 1450.

---

### 📊 Grading

| Dimension | Score |
|---|---|
| Production Realism | 9 |
| Technical Depth | 9 |
| Debugging Practicality | 10 |
| Failure Awareness | 9 |
| Clarity & Signal | 9 |

**Total: 46/50**

**🔍 Critical Feedback:**
- Missing `ss -i` which shows per-socket MTU and TCP MSS — very useful for live connection MTU debugging
- Should mention `nmap --script path-mtu` for external MTU discovery
- The MSS clamping fix is correct but should mention it only helps for NEW connections — existing connections with wrong MSS aren't affected

---

## Q63–Q64 (Supplemental): Load Average & CPU-Bound vs IO-Bound

*(Core answers in Batch 5 — networking-relevant supplement here)*

### Real-World Networking Impact on Load Average

High load from network: `si` (softirq) — when a host receives millions of packets per second, the kernel's NET_RX softirq handler consumes CPU. This shows as load increase even though `%us` is low.

```bash
# Is load driven by softirq?
mpstat -P ALL 1 | grep -v idle | awk '$NF > 5'   # CPUs with >5% softirq

# Which softirq is the culprit?
watch -n1 'cat /proc/softirqs | head -6'
# NET_RX climbing → packet storm or microbursts
# TIMER climbing → timer wheel heavy use

# Pin NET_RX to specific CPUs (RSS — Receive Side Scaling):
ethtool -L eth0 combined $(nproc)   # Set RX queues = CPU count
service irqbalance restart           # Let irqbalance spread IRQs across CPUs
```

---

# 📋 Batch 6 Summary

| Q | Topic | Graded? |
|---|---|---|
| Q41 | TCP 3-way handshake | — |
| Q42 | TCP vs UDP | — |
| Q43 | When to use UDP | — |
| Q44 | curl google.com | Operational ✓ (45/50) |
| Q45 | DNS resolution | — |
| Q46 | DNS debugging | Operational ✓ (45/50) |
| Q47 | ARP | — |
| Q48 | Subnetting/CIDR | — |
| Q49 | Linux routing | — |
| Q50 | Default gateway | — |
| Q51–52 | Firewalls, stateful vs stateless | — |
| Q53 | Packet travel client→server | — |
| Q54 | NAT | — |
| Q55 | Packet drop points | — |
| Q56 | Ping works, curl hangs | — |
| Q57 | MTU and fragmentation | — |
| Q58 | MTU debugging | Operational ✓ (46/50) |

---

# 🏁 All Batches Complete

| Batch | Topic | Questions |
|---|---|---|
| Batch 1 | SRE Scenarios (L2/L3, K8s, HAProxy, CPU, FDs, Deploys) | 6 |
| Batch 2 | Process Management | 10 |
| Batch 3 | Memory Management | 10 |
| Batch 4 | File System, IPC & VFS | 17 |
| Batch 5 | Performance Debugging & Kernel Internals | 15 |
| Batch 6 | Networking Fundamentals | 18 |

**Total: 76 unique questions — all answered at Staff+ level with full self-interrogation loops.**
