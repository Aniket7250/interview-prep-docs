# Networking Fundamentals — SRE Engineering Handbook
> Module: Day 3 — Networking Fundamentals
> Topics: TCP/IP, Three-Way Handshake, DNS, DHCP, ARP, Subnets, VLANs, Firewalls, Routers, Switches
> Audience: Senior SRE interview preparation
> Format: Deep mental models + production commands + debugging runbooks

---

## Mental Model — The Postal System Analogy

Before diving into protocols, lock in one analogy that maps to everything:

Think of networking as a postal system with multiple layers of routing:

- **Your application data** = the letter content
- **TCP/UDP** = the envelope rules (does recipient need to confirm receipt?)
- **IP address** = the city + street address on the envelope
- **MAC address** = the person's name at that address (local delivery only)
- **Router** = the regional sorting office (decides which direction to forward)
- **Switch** = the local postman (delivers within the same neighbourhood)
- **DNS** = the phone book (converts "google.com" to an IP address)
- **DHCP** = the housing office (assigns addresses to new residents)
- **Firewall** = the security checkpoint (inspects and allows/denies based on rules)
- **VLAN** = separate floors in the same building (same physical wire, logically isolated)

Every networking concept maps to this analogy. Use it to sanity-check explanations.

---

## 1. The OSI Model — Why It Exists

The OSI model is not just an academic exercise. It is the debugging framework every SRE uses to isolate which layer a problem lives in. When something is broken, you start at Layer 1 and work up — or start at Layer 7 and work down — until you find where data stops flowing correctly.

```
Layer 7 — Application    HTTP, DNS, SMTP, SSH         "Is the app itself broken?"
Layer 6 — Presentation   TLS/SSL, encoding            "Is encryption broken?"
Layer 5 — Session        Session management           (rarely debugged directly)
Layer 4 — Transport      TCP, UDP                     "Is the connection working?"
Layer 3 — Network        IP, ICMP, routing            "Can packets reach the host?"
Layer 2 — Data Link      Ethernet, MAC, ARP, VLANs   "Can frames reach next hop?"
Layer 1 — Physical       Cables, signals, NICs        "Is the wire connected?"
```

**Production debugging flow:**
```bash
# Layer 1: Is the interface up?
ip link show eth0
ethtool eth0          # shows link speed, duplex, cable detected

# Layer 2: Are frames getting through?
arp -n                # ARP table — can we reach the gateway's MAC?

# Layer 3: Can we reach the IP?
ping -c 4 <destination>
traceroute <destination>

# Layer 4: Is the port open and responding?
nc -zv <host> <port>
telnet <host> <port>

# Layer 7: Is the application responding correctly?
curl -v http://<host>:<port>/health
```

---

## 2. TCP/IP — The Foundation

### 2.1 What TCP/IP Actually Is

TCP/IP is not one protocol — it is a stack of two protocols working together:

**IP (Internet Protocol)** handles addressing and routing. It is a best-effort, connectionless protocol. It delivers packets but makes no promises about whether they arrive, arrive in order, or arrive without duplication. Every packet is independent — IP treats each one separately.

**TCP (Transmission Control Protocol)** sits on top of IP and adds reliability. It creates the illusion of a continuous byte stream over an unreliable packet network. It handles: connection establishment, ordering (sequence numbers), loss detection, retransmission, and flow control.

Together they enable: "I want to send this file to that server reliably, even though the underlying network might drop, reorder, or duplicate packets."

### 2.2 TCP vs UDP — The Real Trade-off

```
TCP                                    UDP
────────────────────────────────────   ────────────────────────────────────
Connection-oriented (handshake)        Connectionless (fire and forget)
Guaranteed delivery                    Best-effort delivery
Ordered delivery                       No ordering guarantee
Flow control (sender slows down)       No flow control
Congestion control                     No congestion control
Higher overhead (~20 byte header)      Lower overhead (~8 byte header)
Use: HTTP, SSH, database, file xfer   Use: DNS, video stream, VoIP, gaming
```

**Why UDP for DNS?** A DNS query is a single question and a single answer. If it gets lost, the client retries after a timeout. The overhead of establishing a TCP connection (3-way handshake = 1.5 RTT) before asking one question is wasteful. UDP sends the question directly and waits. For small request-response patterns where latency matters more than guaranteed delivery, UDP wins.

**Why TCP for HTTP?** You cannot afford to lose bytes from an HTML page or a database query. The application layer (HTTP) should not have to implement its own retry and ordering logic — TCP handles it transparently.

### 2.3 The Three-Way Handshake — Exact Mechanics

The three-way handshake establishes a TCP connection. It synchronises sequence numbers between client and server so both sides know where in the byte stream they are.

```
Client                                          Server
  │                                               │
  │─── SYN (seq=x) ──────────────────────────────▶│
  │    "I want to connect,                         │  Server: allocates half-open
  │     my sequence starts at x"                  │  connection entry, replies
  │                                               │
  │◀── SYN-ACK (seq=y, ack=x+1) ─────────────────│
  │    "OK, I'm ready,                            │
  │     my sequence starts at y,                  │
  │     I acknowledge your x"                     │
  │                                               │
  │─── ACK (ack=y+1) ─────────────────────────────▶│
  │    "Acknowledged your y,                      │  Connection established
  │     connection established"                   │  Both sides ready to send data
  │                                               │
```

**Why three steps and not two?**
Two steps would only confirm that the server received the client's SYN. The third step (client's ACK) confirms that the server's SYN-ACK also got through — meaning both directions of the connection are verified. A two-way handshake only proves one-way communication works.

**Sequence numbers serve two purposes:**
1. Ordering: receiver can reorder out-of-sequence packets
2. Loss detection: gaps in sequence numbers indicate lost packets that need retransmission

**What happens next after handshake:**
```
Client                                          Server
  │─── DATA (seq=x+1, len=100) ─────────────────▶│
  │◀── ACK (ack=x+101) ──────────────────────────│
  │◀── DATA (seq=y+1, len=200) ──────────────────│
  │─── ACK (ack=y+201) ──────────────────────────▶│
```

**Connection teardown — four-way close:**
```
Client                     Server
  │─── FIN ────────────────▶│  Client done sending
  │◀── ACK ─────────────────│  Server acknowledges
  │◀── FIN ─────────────────│  Server done sending
  │─── ACK ────────────────▶│  Client acknowledges
```
Note: FIN and ACK from server are separate because server may still have data to send after acknowledging client's FIN. This is called half-close.

### 2.4 TCP States — What SREs Must Know

```bash
ss -tn              # show all TCP connections with state
ss -tlnp            # show listening sockets with process
netstat -tn         # alternative (older tool)
```

| State | Meaning | SRE significance |
|-------|---------|-----------------|
| LISTEN | Server waiting for connections | Normal for any service |
| SYN_SENT | Client sent SYN, waiting for SYN-ACK | If stuck here: server unreachable or firewall dropping SYN |
| SYN_RECV | Server received SYN, sent SYN-ACK | Many here = SYN flood attack |
| ESTABLISHED | Connection active | Normal |
| FIN_WAIT_1/2 | Client initiated close | Normal during shutdown |
| TIME_WAIT | Connection closed, waiting for late packets | Many here = high connection churn, tune tcp_tw_reuse |
| CLOSE_WAIT | Server received FIN, app hasn't closed socket | **Bug** — app not calling close() |
| LAST_ACK | Server sent FIN, waiting for final ACK | Transient |

**CLOSE_WAIT** is the most important state for SREs. It means the remote side closed the connection (sent FIN) but your application never called `close()` on the socket. This is a bug — file descriptor leak. Over time this accumulates and exhausts the FD limit.

```bash
# Find CLOSE_WAIT accumulation
ss -tn state close-wait
# If growing over time → application is not closing sockets

# Find which process owns these sockets
ss -tnp state close-wait
```

### 2.5 TCP Tuning — Production Knobs

```bash
# View current TCP settings
sysctl -a | grep net.ipv4.tcp

# SYN backlog — connections completed handshake but not yet accept()ed
cat /proc/sys/net/ipv4/tcp_max_syn_backlog     # default 128
sysctl -w net.ipv4.tcp_max_syn_backlog=2048

# Accept queue — same as above at socket level
cat /proc/sys/net/core/somaxconn               # kernel cap for listen backlog
sysctl -w net.core.somaxconn=65536

# TIME_WAIT reuse — allow reusing TIME_WAIT sockets for new connections
sysctl -w net.ipv4.tcp_tw_reuse=1              # safe for client-side
# Do NOT use tcp_tw_recycle — removed in Linux 4.12, breaks NAT

# Keep-alive settings
sysctl net.ipv4.tcp_keepalive_time             # default 7200 (2 hours!)
sysctl -w net.ipv4.tcp_keepalive_time=60       # detect dead connections faster
sysctl -w net.ipv4.tcp_keepalive_intvl=10
sysctl -w net.ipv4.tcp_keepalive_probes=3

# Connection queue full symptom in ss
ss -ltn
# Recv-Q: connections waiting to be accept()ed
# If Recv-Q = backlog limit → app not calling accept() fast enough
```

---

## 3. IP Addressing and Subnets

### 3.1 IPv4 Address Structure

An IPv4 address is 32 bits written as four decimal octets: `192.168.1.100`

```
192      .  168      .  1        .  100
11000000    10101000    00000001    01100100
```

Every IP address has two logical parts:
- **Network portion**: identifies which network (same for all hosts on that network)
- **Host portion**: identifies the specific host within that network

The subnet mask (or CIDR prefix) tells you where the dividing line is.

### 3.2 Subnet Masks and CIDR — How to Think About Them

CIDR notation `/24` means "the first 24 bits are the network, the remaining 8 bits are the host."

```
192.168.1.0/24
│             │
│             └─ /24 = 24 bits for network, 8 bits for hosts
└─ Network address

Binary breakdown:
Network: 11000000.10101000.00000001  (24 bits — fixed)
Host:                               .xxxxxxxx (8 bits — varies)

Number of hosts: 2^8 = 256 addresses
Usable hosts: 256 - 2 = 254
  (subtract: .0 = network address, .255 = broadcast address)
```

**Common subnet sizes:**

| CIDR | Subnet Mask | Hosts | Use case |
|------|-------------|-------|----------|
| /8 | 255.0.0.0 | 16,777,214 | Large enterprises, ISPs |
| /16 | 255.255.0.0 | 65,534 | Large office networks |
| /24 | 255.255.255.0 | 254 | Standard LAN segment |
| /25 | 255.255.255.128 | 126 | Split a /24 in half |
| /26 | 255.255.255.192 | 62 | Smaller segments |
| /28 | 255.255.255.240 | 14 | Small server groups |
| /30 | 255.255.255.252 | 2 | Point-to-point links |
| /32 | 255.255.255.255 | 1 | Single host, loopback routes |

**How to calculate subnet quickly:**
```
/24 → 256 addresses → 254 usable
/25 → 128 addresses → 126 usable (splits /24 into two halves: .0-.127 and .128-.255)
/26 → 64 addresses → 62 usable
/27 → 32 addresses → 30 usable
/28 → 16 addresses → 14 usable
/29 → 8 addresses → 6 usable
/30 → 4 addresses → 2 usable

Rule: each step adds one to the prefix → halves the address space
```

**Private IP ranges (RFC 1918 — not routable on public internet):**
```
10.0.0.0/8          → 10.x.x.x           (16M addresses)
172.16.0.0/12       → 172.16.x.x - 172.31.x.x  (1M addresses)
192.168.0.0/16      → 192.168.x.x        (65K addresses)
```

### 3.3 How a Host Decides: Same Subnet or Different?

When process A on host `192.168.1.10/24` wants to send to `192.168.1.50`:
```
My IP:   192.168.1.10   AND with /24 mask = 192.168.1.0 (my network)
Dest IP: 192.168.1.50   AND with /24 mask = 192.168.1.0 (same network)
Same network → send directly, use ARP to find MAC address
```

When process A on `192.168.1.10/24` wants to send to `8.8.8.8`:
```
My IP:   192.168.1.10   AND with /24 mask = 192.168.1.0
Dest IP: 8.8.8.8        AND with /24 mask = 8.0.0.0 (different network)
Different network → send to default gateway (router)
```

This is the fundamental routing decision every host makes for every packet.

```bash
# View routing table — how your host makes this decision
ip route show
# or
route -n

# Output example:
# default via 192.168.1.1 dev eth0    ← default gateway for everything else
# 192.168.1.0/24 dev eth0             ← this subnet is directly reachable
# 10.0.0.0/8 via 10.1.1.1 dev eth1   ← this subnet via specific gateway

# Find which route a specific destination uses
ip route get 8.8.8.8
```

---

## 4. ARP — Address Resolution Protocol

### 4.1 The Problem ARP Solves

IP addresses are logical addresses (Layer 3). Ethernet frames are delivered using MAC addresses (Layer 2). When host A wants to send a packet to host B on the same subnet, it knows B's IP address but needs B's MAC address to construct the Ethernet frame.

ARP bridges this gap: it broadcasts "who has this IP address? Tell me your MAC."

### 4.2 ARP — Step by Step

```
Host A (192.168.1.10, MAC: AA:AA:AA:AA:AA:AA)
Host B (192.168.1.20, MAC: BB:BB:BB:BB:BB:BB)

Step 1: A wants to send to B's IP. Checks ARP cache — not found.

Step 2: A broadcasts ARP Request to all hosts on the LAN
  Src MAC:  AA:AA:AA:AA:AA:AA
  Dst MAC:  FF:FF:FF:FF:FF:FF  ← broadcast — everyone receives this
  Content:  "Who has 192.168.1.20? Tell 192.168.1.10"

Step 3: Every host receives it. Only B responds (it has 192.168.1.20).
  ARP Reply (unicast directly to A):
  "192.168.1.20 is at BB:BB:BB:BB:BB:BB"

Step 4: A caches the mapping: 192.168.1.20 → BB:BB:BB:BB:BB:BB
  Now A can construct Ethernet frames destined for BB:BB:BB:BB:BB:BB
  This cache entry expires after ~20 minutes (configurable)

Step 5: For all future packets to 192.168.1.20, A uses the cached MAC
  No ARP needed until cache expires
```

**When sending to a different subnet:** ARP is sent for the **gateway's IP**, not the destination IP. The gateway then uses its own routing table to forward the packet further.

```bash
# View ARP cache
arp -n
ip neigh show

# Force ARP resolution
arping -I eth0 192.168.1.1

# Clear ARP cache entry
ip neigh del 192.168.1.20 dev eth0

# Gratuitous ARP — host announces its own IP/MAC mapping (used after IP change)
arping -A -I eth0 192.168.1.10

# ARP-related debugging
tcpdump -i eth0 arp               # capture ARP traffic
tcpdump -i eth0 arp and host 192.168.1.20
```

**ARP in production problems:**
- **ARP cache poisoning**: Malicious host sends fake ARP replies, redirecting traffic
- **Duplicate IP**: Two hosts with same IP both respond to ARP → intermittent connectivity
- **Stale ARP cache**: After changing a server's NIC, other hosts still have old MAC → send gratuitous ARP to force update
- **ARP storm**: Many hosts ARPing simultaneously on a large flat network → use VLANs to segment

---

## 5. DNS — Domain Name System

### 5.1 The Mental Model

DNS is a distributed, hierarchical database that translates human-readable names (google.com) into IP addresses (142.250.80.46). It is distributed because no single server knows all names — the resolution is delegated through a hierarchy.

```
Root (.)
  └─ .com  (Top Level Domain — TLD)
      └─ google.com  (Authoritative nameserver for google.com)
          └─ www.google.com  (specific record)
```

### 5.2 DNS Resolution — Complete Walk-through

When your browser requests `www.google.com`:

```
Browser
  │
  ▼
1. Check browser DNS cache (TTL-based, often 60 seconds)
   → Found? Return immediately.
   → Not found? Continue.
  │
  ▼
2. Check OS resolver cache (/etc/hosts first, then nscd/systemd-resolved cache)
   → Found? Return immediately.
   → Not found? Continue.
  │
  ▼
3. Query Recursive Resolver (your ISP's DNS or 8.8.8.8 or 1.1.1.1)
   → This is the server in /etc/resolv.conf
   → Recursive resolver does all the work on your behalf
  │
  ▼
4. Recursive Resolver checks its own cache
   → Found? Return to client.
   → Not found? Start recursive resolution:
  │
  ▼
5. Query Root Nameserver (13 root server clusters worldwide)
   "Who is responsible for .com?"
   → Response: "Ask the .com TLD nameservers at these IPs"
  │
  ▼
6. Query .com TLD Nameserver
   "Who is responsible for google.com?"
   → Response: "Ask Google's authoritative nameservers at ns1.google.com"
  │
  ▼
7. Query Google's Authoritative Nameserver
   "What is www.google.com?"
   → Response: "142.250.80.46, TTL=300"
  │
  ▼
8. Recursive resolver returns answer to client, caches it for 300 seconds
  │
  ▼
9. Browser connects to 142.250.80.46
```

### 5.3 DNS Record Types — Complete Reference

| Record | Purpose | Example |
|--------|---------|---------|
| A | IPv4 address | `www.example.com → 93.184.216.34` |
| AAAA | IPv6 address | `www.example.com → 2606:2800::1` |
| CNAME | Alias to another name | `blog.example.com → example.com` |
| MX | Mail server for domain | `example.com → mail.example.com (priority 10)` |
| NS | Authoritative nameservers | `example.com → ns1.example.com` |
| TXT | Arbitrary text (SPF, DKIM, verification) | `"v=spf1 include:_spf.google.com ~all"` |
| PTR | Reverse DNS (IP → name) | `34.216.184.93.in-addr.arpa → www.example.com` |
| SOA | Start of Authority (zone metadata) | Serial, refresh, retry, expire, TTL |
| SRV | Service location | `_http._tcp.example.com → host:port:priority` |

**CNAME trap:** A CNAME cannot coexist with other records at the same name. You cannot have `example.com` as both a CNAME and an MX record. This is why root domains (apex) use A records directly, and CDNs use tricks like ALIAS or ANAME records.

### 5.4 DNS Debugging — Production Commands

```bash
# Basic lookup
dig www.google.com                    # full DNS response with timing
dig www.google.com A                  # specific record type
dig www.google.com +short             # just the answer
nslookup www.google.com               # simpler alternative

# Query a specific nameserver directly (bypass your resolver)
dig @8.8.8.8 www.google.com           # query Google's DNS
dig @1.1.1.1 www.google.com           # query Cloudflare's DNS

# Trace the full resolution chain
dig +trace www.google.com             # shows every step from root → answer

# Reverse DNS lookup (IP → name)
dig -x 8.8.8.8
host 8.8.8.8

# Check authoritative nameservers for a domain
dig NS google.com
dig SOA google.com

# Check MX records
dig MX gmail.com

# Check TTL (important for propagation timing)
dig www.google.com | grep -A2 "ANSWER"

# View local DNS resolver config
cat /etc/resolv.conf                  # nameserver IPs, search domains
resolvectl status                     # systemd-resolved (modern systems)

# Flush DNS cache
# systemd-resolved:
resolvectl flush-caches
# nscd:
service nscd restart
# macOS:
sudo dscacheutil -flushcache

# DNS from /etc/hosts (always checked first)
cat /etc/hosts

# Debug DNS resolution order
cat /etc/nsswitch.conf | grep hosts
# typical: files dns  → check /etc/hosts first, then DNS
```

**Common DNS production issues:**

**High TTL + IP change:** Changed server IP but old IP still cached because TTL was 86400 (24 hours). Always lower TTL to 300 before making IP changes, change IP, wait 24 hours, then raise TTL again.

**Split-horizon DNS:** Internal DNS returns private IP for `app.company.com`, external DNS returns public IP. If your internal resolver is misconfigured to query external servers, internal clients get the wrong IP.

**DNS propagation:** Authoritative change propagates as TTLs expire — it is not instant. Check each level of the hierarchy with `dig @<specific-ns>` to confirm where the change has propagated.

```bash
# Check if DNS change has propagated to specific resolver
dig @8.8.8.8 yoursite.com     # Google's resolver has new value?
dig @1.1.1.1 yoursite.com     # Cloudflare's resolver has new value?
dig @<your-local-resolver> yoursite.com  # Your network has new value?
```

---

## 6. DHCP — Dynamic Host Configuration Protocol

### 6.1 The Problem DHCP Solves

When a new device joins a network, it needs four things before it can communicate: an IP address, a subnet mask, a default gateway, and a DNS server. Without DHCP, a network administrator would configure each of these manually on every device. DHCP automates this entirely.

### 6.2 The DORA Process — Four Steps

DHCP uses a four-message exchange remembered as **DORA**:

```
Client                                    DHCP Server
  │                                           │
  │─── DISCOVER (broadcast) ─────────────────▶│
  │    "I need an IP address, I'm new here"   │
  │    Src IP: 0.0.0.0                        │
  │    Dst IP: 255.255.255.255 (broadcast)    │
  │    Src MAC: client's MAC                  │
  │                                           │
  │◀── OFFER ─────────────────────────────────│
  │    "Here is 192.168.1.50,                │
  │     gateway 192.168.1.1,                 │
  │     DNS 8.8.8.8,                         │
  │     lease for 24 hours"                  │
  │                                           │
  │─── REQUEST (broadcast) ──────────────────▶│
  │    "I accept 192.168.1.50"               │
  │    (broadcast so other DHCP servers       │
  │     know this offer was selected)         │
  │                                           │
  │◀── ACK ───────────────────────────────────│
  │    "Confirmed, lease starts now"          │
  │    Client configures its interface        │
```

**Why REQUEST is broadcast again:** If multiple DHCP servers respond to DISCOVER with OFFERs, the client picks one and broadcasts the REQUEST. All other DHCP servers see the REQUEST and know their offer was rejected — they put their offered IP back in the pool.

**Lease renewal:** Client tries to renew at 50% of lease time (via unicast to original server), then at 87.5% (via broadcast to any server). If renewal fails, client must stop using the IP when lease expires.

```bash
# Request a new DHCP lease
dhclient eth0                         # request lease on eth0
dhclient -r eth0                      # release current lease
dhclient -v eth0                      # verbose — shows DORA messages

# View current DHCP lease info
cat /var/lib/dhclient/dhclient.leases
cat /var/lib/dhcp/dhclient.eth0.leases

# Modern systems (systemd-networkd)
networkctl status eth0

# DHCP server side — view active leases
cat /var/lib/dhcpd/dhcpd.leases
```

**DHCP in SRE scenarios:**
- **IP exhaustion**: DHCP pool runs out → new devices cannot get addresses. Check lease file for expired leases not released.
- **Rogue DHCP server**: Another device on the network answering DHCP requests and giving out wrong gateway/DNS → DHCP snooping on switches prevents this.
- **DHCP starvation attack**: Attacker sends thousands of DISCOVER messages with fake MACs, exhausting the pool.

---

## 7. Routers and Switches — The Difference

### 7.1 Switches — Layer 2 Devices

A switch forwards Ethernet frames within the same network (same broadcast domain). It learns which MAC addresses are reachable through which port by observing traffic. This learning process builds the MAC address table (also called CAM table).

```
Switch MAC Table:
Port 1 → AA:AA:AA:AA:AA:AA  (Host A)
Port 2 → BB:BB:BB:BB:BB:BB  (Host B)
Port 3 → CC:CC:CC:CC:CC:CC  (Host C)

When Host A sends frame to BB:BB:BB:BB:BB:BB:
  Switch looks up BB:BB... → Port 2
  Forwards frame ONLY to Port 2 (not broadcast)
  Host B and C do not receive it
```

**Unknown MAC → flood:** If the switch does not have a MAC in its table, it sends the frame out ALL ports (except the receiving port). This is why new connections cause a brief broadcast until the switch learns the MAC.

**Switches operate at Layer 2.** They have no concept of IP addresses in their forwarding decision (unless they are Layer 3 switches, which can also route).

### 7.2 Routers — Layer 3 Devices

A router forwards packets between different networks (different subnets). It makes decisions based on the destination IP address and its routing table.

```
Routing Table Example:
Destination     Gateway         Interface
192.168.1.0/24  directly        eth0      ← local LAN
10.0.0.0/8      10.1.1.1        eth1      ← reach 10.x.x.x via 10.1.1.1
0.0.0.0/0       192.168.1.1    eth0      ← default: send everything else here
```

**Every hop changes MAC addresses, never IP addresses:**
```
Host A (192.168.1.10) sending to 8.8.8.8:

Hop 1 (A → Router):
  IP src: 192.168.1.10  IP dst: 8.8.8.8   ← unchanged
  MAC src: AA:AA:AA:AA  MAC dst: Router MAC ← A's frame to router

Hop 2 (Router → ISP):
  IP src: 192.168.1.10  IP dst: 8.8.8.8   ← still unchanged
  MAC src: Router WAN   MAC dst: ISP MAC   ← new frame, new MACs

This repeats at every hop. IP addresses are end-to-end. MAC addresses are hop-by-hop.
```

### 7.3 Switch vs Router — Quick Reference

| Feature | Switch | Router |
|---------|--------|--------|
| Layer | 2 (Data Link) | 3 (Network) |
| Addresses | MAC addresses | IP addresses |
| Scope | Same network | Between networks |
| Broadcast domain | Single domain | Separates domains |
| Table | MAC address table (CAM) | Routing table |
| Use | Connect hosts on LAN | Connect LANs, internet access |

---

## 8. VLANs — Virtual LANs

### 8.1 The Problem VLANs Solve

Without VLANs, every device on a switch is in the same broadcast domain. A broadcast from any host goes to every host. As networks grow, this creates three problems:
1. **Security**: All hosts can see all broadcast traffic (ARP, DHCP, etc.)
2. **Performance**: Large broadcast domains generate excessive broadcast traffic
3. **Management**: No way to logically separate departments or services without separate physical switches

VLANs partition a single physical switch into multiple logical switches. Traffic in VLAN 10 cannot reach VLAN 20 unless it passes through a router (or Layer 3 switch).

### 8.2 How VLANs Work

IEEE 802.1Q standard adds a 4-byte VLAN tag to Ethernet frames:

```
Normal Ethernet Frame:
[Dst MAC | Src MAC | EtherType | Payload]

802.1Q Tagged Frame:
[Dst MAC | Src MAC | 802.1Q Tag (VLAN ID 1-4094) | EtherType | Payload]
```

**Port types on a switch:**
- **Access port**: Belongs to one VLAN. End devices (servers, PCs) connect here. The switch adds/removes VLAN tags transparently — the end device never sees the tag.
- **Trunk port**: Carries multiple VLANs. Switch-to-switch or switch-to-router links. Frames carry the 802.1Q tag so the receiving device knows which VLAN it belongs to.

```
VLAN Design Example:
VLAN 10 — Web servers     (192.168.10.0/24)
VLAN 20 — App servers     (192.168.20.0/24)
VLAN 30 — DB servers      (192.168.30.0/24)
VLAN 100 — Management     (10.0.100.0/24)

Web → App: must go through router/firewall (different VLANs)
App → DB: must go through router/firewall
All three → Management: firewall rules control access
```

**Inter-VLAN routing** requires either:
1. A dedicated router port for each VLAN ("router on a stick" — single trunk port, subinterfaces per VLAN)
2. A Layer 3 switch with routing capability

```bash
# View VLAN configuration on Linux (trunked interface)
ip link show
# Look for: eth0.10 (VLAN 10 subinterface on eth0)

# Create a VLAN interface on Linux
ip link add link eth0 name eth0.10 type vlan id 10
ip addr add 192.168.10.1/24 dev eth0.10
ip link set eth0.10 up

# View VLAN interfaces
cat /proc/net/vlan/config
```

---

## 9. Firewalls

### 9.1 What Firewalls Do

A firewall inspects network traffic and allows or denies it based on rules. Rules typically match on: source IP, destination IP, source port, destination port, protocol (TCP/UDP/ICMP), and connection state.

**Stateful vs stateless:**
- **Stateless**: Each packet evaluated independently. Rule must explicitly allow both directions. Simpler but less secure.
- **Stateful**: Tracks connection state. If you allow outbound TCP to port 80, the firewall automatically allows the return traffic for established connections. Most production firewalls are stateful.

### 9.2 iptables — Linux Firewall

```bash
# View current rules
iptables -L -v -n                     # list all rules with counters
iptables -L INPUT -v -n               # just the INPUT chain
iptables -t nat -L -v -n              # NAT table

# Basic rule structure
iptables -A <chain> -p <proto> --dport <port> -j <action>
# Chains: INPUT (incoming), OUTPUT (outgoing), FORWARD (routing through)
# Actions: ACCEPT, DROP, REJECT, LOG

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow established connections (stateful)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Drop everything else
iptables -A INPUT -j DROP

# Allow specific IP range
iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# Port forwarding (NAT)
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.10:8080

# Save rules (survives reboot)
iptables-save > /etc/iptables/rules.v4

# View with nftables (modern replacement for iptables)
nft list ruleset
```

### 9.3 Firewall Debugging

```bash
# Is traffic being dropped? Add logging rule
iptables -A INPUT -j LOG --log-prefix "DROPPED: " --log-level 4
# Then check: dmesg | grep DROPPED

# Trace packet through rules
iptables -t raw -A PREROUTING -p tcp --dport 80 -j TRACE
# Then check: /var/log/kern.log or journalctl -k | grep TRACE

# Test if a port is reachable from the outside
nc -zv <target-host> <port>           # TCP connection test
nc -zvu <target-host> <port>          # UDP test

# Test if firewall is blocking vs service not listening
# If: connection refused → service not listening (firewall passed it through)
# If: connection timeout → firewall dropping packets (no response at all)
```

---

## 10. Network Debugging — Production Runbooks

### 10.1 "I Cannot Reach Host X" — Systematic Debug

```bash
# Step 1: Can I reach the gateway? (Layer 3 local)
ip route show                         # find default gateway
ping -c 3 <gateway-ip>
# Fail → local network issue (cable, switch, VLAN, IP config)

# Step 2: Can I reach a known public IP? (Layer 3 routing)
ping -c 3 8.8.8.8
# Fail → routing issue, ISP problem, firewall blocking ICMP

# Step 3: Can I reach the destination IP?
ping -c 3 <destination-ip>
# Fail → destination unreachable (firewall, routing, host down)

# Step 4: Can I resolve the hostname?
dig <hostname> +short
# Fail → DNS issue (see Section 5)

# Step 5: Can I reach the specific port?
nc -zv <hostname> <port>
# Connection refused → host reachable, port not listening
# Timeout → firewall dropping, host unreachable

# Step 6: Trace the path
traceroute <destination>              # shows each hop and RTT
traceroute -T <destination>           # TCP traceroute (bypasses ICMP blocks)
mtr <destination>                     # live traceroute with statistics
```

### 10.2 "Connection Keeps Dropping"

```bash
# Check for packet loss along the path
mtr --report --report-cycles 100 <destination>
# Look for packet loss % at each hop

# Check interface errors
ip -s link show eth0
# Look for: RX errors, TX errors, dropped, overruns

ethtool -S eth0                       # detailed NIC statistics
# Look for: rx_missed_errors, rx_crc_errors, tx_errors

# Check for TCP retransmissions
ss -ti dst <destination-ip>
# Look for: retrans:X

# System-wide TCP stats
netstat -s | grep -i retransmit
cat /proc/net/netstat | grep RetransSegs
```

### 10.3 "Port is Not Reachable"

```bash
# Is the service listening?
ss -tlnp | grep <port>
netstat -tlnp | grep <port>
# Not listed → service not running or listening on wrong interface

# Is firewall blocking?
iptables -L -n -v | grep <port>
# Or test from the server itself vs from outside

# Is the service binding to 0.0.0.0 or 127.0.0.1?
ss -tlnp | grep <port>
# 0.0.0.0 = all interfaces (reachable from outside)
# 127.0.0.1 = loopback only (not reachable from outside)
# :::port = IPv6 all interfaces

# Capture traffic to confirm packets arriving
tcpdump -i eth0 -n port <port>
# If packets arrive but no response → service not picking them up
# If no packets arrive → firewall/routing dropping before reaching host
```

### 10.4 "DNS Is Broken"

```bash
# Test DNS resolution
dig google.com @<your-resolver>      # test specific resolver
dig google.com @8.8.8.8              # test with known-good resolver

# If @8.8.8.8 works but @your-resolver fails:
# → your DNS server is broken or unreachable

# Check resolv.conf
cat /etc/resolv.conf
# nameserver <ip>     ← DNS server(s) to query
# search company.com  ← appended to unqualified names
# options ndots:5     ← how many dots before treating as FQDN

# Check if DNS traffic reaching the server
tcpdump -i eth0 -n port 53

# Check /etc/hosts first
grep <hostname> /etc/hosts
```

### 10.5 Key Tools Summary

```bash
# Connectivity
ping <host>                           # ICMP reachability
traceroute <host>                     # path and hop latency
mtr <host>                            # live traceroute with stats
nc -zv <host> <port>                  # TCP port test

# Interface and addressing
ip addr show                          # IP addresses on all interfaces
ip link show                          # interface state
ip route show                         # routing table
ip neigh show                         # ARP cache (neighbours)

# Connections
ss -tlnp                              # listening TCP sockets with process
ss -tn                                # established TCP connections
ss -tn state established              # only established
ss -tn state close-wait               # problematic CLOSE_WAIT

# DNS
dig <name>                            # full DNS query with details
dig +trace <name>                     # trace full resolution chain
dig @<server> <name>                  # query specific nameserver
host <name>                           # simple lookup

# Packet capture
tcpdump -i eth0 -n                    # all traffic on eth0
tcpdump -i eth0 -n port 80            # HTTP traffic
tcpdump -i eth0 -n host 8.8.8.8       # traffic to/from specific host
tcpdump -i eth0 -n tcp and port 443 -w /tmp/capture.pcap  # save to file

# Traffic analysis
iftop -i eth0                         # live bandwidth by connection
nethogs eth0                          # bandwidth by process
sar -n DEV 1 5                        # interface statistics over time

# ARP
arp -n                                # ARP cache
arping -I eth0 <ip>                   # send ARP request

# Firewall
iptables -L -v -n                     # current rules with counters
nft list ruleset                      # nftables equivalent
```

---

## 11. Interview One-Liners

| Question | Answer |
|----------|--------|
| What is the three-way handshake? | SYN → SYN-ACK → ACK. Client proposes sequence number, server acknowledges and proposes its own, client acknowledges. Proves both directions work. |
| Why three steps not two? | Two steps only proves one-way. Third step (client ACK) confirms server's SYN-ACK was received — verifies both directions. |
| TCP vs UDP? | TCP: reliable, ordered, connection-based, slower. UDP: unreliable, unordered, connectionless, faster. Use TCP for correctness, UDP for latency. |
| What is CLOSE_WAIT? | Remote closed connection (sent FIN) but local app never called close(). It is a bug — FD leak. Find with ss -tn state close-wait. |
| What does ARP do? | Resolves IP addresses to MAC addresses within a subnet by broadcasting "who has this IP?" |
| What if ARP target is on different subnet? | ARP is sent for the gateway's IP, not the destination. Gateway forwards the packet. |
| DNS resolution order? | Browser cache → OS cache → /etc/hosts → recursive resolver → root → TLD → authoritative. |
| DHCP DORA? | Discover (broadcast) → Offer → Request (broadcast) → ACK. Client gets IP, gateway, DNS, lease time. |
| Switch vs router? | Switch forwards frames by MAC address within a subnet (L2). Router forwards packets by IP address between subnets (L3). |
| What is a VLAN? | Logical partition of a physical switch into separate broadcast domains. Inter-VLAN traffic must go through a router. |
| /24 vs /25 vs /30? | /24 = 254 hosts. /25 = 126 hosts (half of /24). /30 = 2 hosts (point-to-point links). Each step halves the space. |
| Connection timeout vs refused? | Timeout = firewall dropping packets silently. Refused = host reachable, nothing listening on that port. |
| How to check if DNS change propagated? | dig @8.8.8.8 domain.com and dig @<your-resolver> domain.com. Compare answers. Check TTL. |
| IP addresses change at each hop? | No. IP addresses are end-to-end. Only MAC addresses change at each hop. |
| What is a default gateway? | The router your host sends packets to when the destination is on a different subnet. Found via ip route show. |
| What is /etc/resolv.conf? | Configures which DNS servers the OS queries. Also contains search domains and ndots options. |
