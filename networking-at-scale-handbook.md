# Networking at Scale — Kubernetes & On-Premise SRE Handbook
> Module: Day 3 Extension — Networking at Scale
> Topics: Kubernetes Networking, CNI, Service Mesh, On-Premise Scale, Load Balancing, BGP, Overlay Networks
> Audience: Senior SRE interview preparation
> Builds on: networking-handbook.md (TCP/IP, DNS, ARP, Subnets, VLANs)

---

## Mental Model — From One Server to a Thousand

In the basic networking handbook, one host communicates with another. The rules are simple: ARP finds the MAC, IP routes the packet, TCP ensures delivery.

At scale, three new problems emerge that the basic model does not solve:

**Problem 1 — Identity:** When you have 500 pods running the same service, which IP do clients connect to? IPs change every time a pod restarts. You cannot hardcode them.

**Problem 2 — Discovery:** How does Service A find Service B when both are running on dynamically scheduled containers that could be on any of 200 nodes?

**Problem 3 — Isolation:** How do you stop the payments service from talking directly to the internal admin API, across hundreds of nodes, without maintaining firewall rules for every combination?

Kubernetes networking, service meshes, and on-premise scale infrastructure exist to solve these three problems. Everything in this handbook maps back to one of them.

---

## Part 1 — Kubernetes Networking

### 1.1 The Kubernetes Networking Model — Four Rules

Kubernetes mandates a specific networking model. Any CNI (Container Network Interface) plugin must satisfy four rules:

1. **Every pod gets its own IP address** — no port sharing between pods on same host
2. **Pods on the same node can communicate without NAT**
3. **Pods on different nodes can communicate without NAT** — pod IP is routable cluster-wide
4. **The IP a pod sees itself as is the same IP others see it as** — no source NAT hiding the real pod IP

This is radically different from traditional Docker networking (which used NAT and port mapping). In Kubernetes, if a pod has IP `10.244.1.5`, any other pod in the cluster can reach it at `10.244.1.5:port` directly.

### 1.2 How a Pod Gets Its IP — The CNI Plugin

When kubelet creates a pod, it calls the CNI plugin to set up networking. The CNI plugin is responsible for:

```
kubelet creates pod
  │
  ▼
CNI plugin called with pod namespace
  │
  ├─▶ Creates a veth pair (virtual ethernet cable)
  │     veth0 ← inside pod namespace (becomes eth0 in pod)
  │     veth1 ← in host network namespace
  │
  ├─▶ Assigns IP from pod CIDR (e.g., 10.244.1.5/24)
  │
  ├─▶ Sets up routes so pod can reach other pods
  │
  └─▶ Sets up rules so other pods can reach this pod
```

**The veth pair analogy:** Think of a veth pair as a physical Ethernet cable with one end plugged into the pod and the other end plugged into a virtual bridge on the host. Traffic enters one end and exits the other.

```bash
# See veth pairs on a node
ip link show | grep veth

# See pod network namespace
# Each pod has its own network namespace
ls /var/run/netns/
ip netns list

# Inspect pod interface from host
ip netns exec <pod-netns> ip addr show
ip netns exec <pod-netns> ip route show

# From inside a pod
kubectl exec -it <pod> -- ip addr show
kubectl exec -it <pod> -- ip route show
kubectl exec -it <pod> -- cat /etc/resolv.conf
```

### 1.3 CNI Plugins — Which to Use and Why

| CNI Plugin | Approach | When to Use |
|-----------|---------|------------|
| **Flannel** | VXLAN overlay (default) | Simple clusters, learning, no BGP |
| **Calico** | BGP routing OR overlay | Production — policy enforcement + performance |
| **Cilium** | eBPF (kernel-level) | High performance, advanced observability |
| **Weave** | Mesh overlay + encryption | Encrypted multi-site |
| **Canal** | Flannel + Calico policy | Flannel networking + Calico NetworkPolicy |

**Calico with BGP** is the most common production choice for on-premise because it avoids overlay encapsulation overhead (no VXLAN wrapping) and integrates with physical network infrastructure via BGP peering.

**Cilium** is the emerging choice for new clusters — it replaces kube-proxy with eBPF programs, reducing latency and enabling deeper observability without sidecar proxies.

### 1.4 Overlay vs Underlay Networking

**Overlay networking (VXLAN, Flannel default):**

The pod network runs on top of the physical network. Pod-to-pod packets are encapsulated inside UDP packets (VXLAN) before being sent over the physical network. The physical network only sees the node IPs.

```
Pod A (10.244.1.5) → Pod B (10.244.2.7) different nodes:

Original packet:
  src: 10.244.1.5  dst: 10.244.2.7  payload: HTTP data

VXLAN encapsulation on Node 1:
  Outer src: 192.168.1.10 (Node 1 IP)
  Outer dst: 192.168.1.20 (Node 2 IP)
  UDP port 4789 (VXLAN)
  Inner: original pod packet

Node 2 decapsulates, delivers to 10.244.2.7
```

**Overhead:** VXLAN adds ~50 bytes of header per packet. At high throughput this matters. For a 1500-byte MTU link, effective payload drops to ~1450 bytes.

**Underlay networking (BGP routing, Calico):**

Pod IPs are directly routable on the physical network. No encapsulation. Nodes advertise pod CIDR routes to the physical network via BGP. A router in the rack knows that `10.244.1.0/24` is reachable via Node 1.

```
Pod A (10.244.1.5) → Pod B (10.244.2.7) different nodes:

Physical router routing table:
  10.244.1.0/24 → Node 1 (192.168.1.10)
  10.244.2.0/24 → Node 2 (192.168.1.20)

Packet flows directly:
  Pod A → veth → bridge → Node 1 eth0 → physical switch → Node 2 → Pod B
  No encapsulation, no overhead
```

**Trade-off:** Underlay requires cooperation from physical network (BGP peering with ToR switches). Overlay works on any network that can route between nodes.

### 1.5 Kubernetes Services — The Identity and Discovery Solution

A Service gives a stable virtual IP (ClusterIP) in front of a set of pods. Even as pods die and are replaced with new IPs, the Service IP stays constant. This solves the identity problem.

```
Pods (ephemeral, changing IPs):        Service (stable):
  Pod A: 10.244.1.5  ───────────────▶  ClusterIP: 10.96.0.100
  Pod B: 10.244.2.3  ───────────────▶  (virtual IP, never changes)
  Pod C: 10.244.3.8  ───────────────▶

Client always connects to 10.96.0.100
kube-proxy distributes to healthy pods
```

**Service types:**

| Type | Scope | How it works |
|------|-------|-------------|
| ClusterIP | Cluster-internal only | Virtual IP, no external access |
| NodePort | External via node IP | Opens port on every node (30000-32767) |
| LoadBalancer | External via cloud LB | Provisions cloud load balancer |
| ExternalName | DNS CNAME alias | Maps service name to external DNS name |

### 1.6 kube-proxy — How Services Actually Work

kube-proxy runs on every node and implements Services by programming iptables (or IPVS) rules that redirect traffic destined for a ClusterIP to an actual pod IP.

```
Client pod sends packet to ClusterIP 10.96.0.100:80

iptables chain (programmed by kube-proxy):
  PREROUTING → KUBE-SERVICES
  KUBE-SERVICES: match dst=10.96.0.100, port=80
    → KUBE-SVC-<hash>
      → 33% chance: DNAT to 10.244.1.5:8080 (Pod A)
      → 33% chance: DNAT to 10.244.2.3:8080 (Pod B)
      → 33% chance: DNAT to 10.244.3.8:8080 (Pod C)

Packet's destination IP changes to selected pod IP
Response comes back through conntrack (stateful NAT)
```

**IPVS mode** (preferred for large clusters):
- iptables scales O(n) — each rule check is linear
- IPVS uses hash tables — O(1) lookup regardless of service count
- Supports more load balancing algorithms (round-robin, least-connection, source-hash)

```bash
# Check kube-proxy mode
kubectl -n kube-system get configmap kube-proxy -o yaml | grep mode

# View iptables rules for a service
iptables -t nat -L KUBE-SERVICES -n | grep <service-cluster-ip>
iptables -t nat -L KUBE-SVC-<hash> -n

# View IPVS table (if using IPVS mode)
ipvsadm -L -n
ipvsadm -L -n --stats   # with connection counts

# Debug service connectivity
kubectl get endpoints <service-name>    # shows which pod IPs are behind service
# If endpoints is empty → label selector mismatch or pods not Ready
```

### 1.7 DNS in Kubernetes — CoreDNS

Every Kubernetes cluster runs CoreDNS (previously kube-dns) as the cluster DNS resolver. Every pod's `/etc/resolv.conf` points to the CoreDNS ClusterIP.

**DNS naming structure:**
```
<service-name>.<namespace>.svc.<cluster-domain>

Examples:
  my-service.default.svc.cluster.local       → ClusterIP of my-service in default namespace
  my-service.production.svc.cluster.local    → same service in production namespace
  my-service                                  → works within same namespace (search domain)
  my-service.production                       → works from any namespace

# Pod DNS
<pod-ip-dashes>.<namespace>.pod.cluster.local
  10-244-1-5.default.pod.cluster.local
```

**ndots:5 setting — why it matters:**

Every pod's `/etc/resolv.conf` contains `options ndots:5`. This means: if a hostname has fewer than 5 dots, try appending search domains before treating it as a fully-qualified domain name (FQDN).

```
Pod queries "api.external.com" (2 dots < 5):
  1. Try: api.external.com.default.svc.cluster.local  → NXDOMAIN
  2. Try: api.external.com.svc.cluster.local          → NXDOMAIN
  3. Try: api.external.com.cluster.local              → NXDOMAIN
  4. Try: api.external.com                             → found! returns IP

This causes 3 unnecessary DNS queries for every external name lookup.
```

**Fix:** Add a trailing dot to FQDNs in your application config: `api.external.com.` — the dot means "this is already fully qualified, don't append search domains."

Or set `ndots: 1` in pod DNS config for services that mostly call external APIs.

```bash
# Debug DNS in a pod
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl exec -it <pod> -- dig my-service.production.svc.cluster.local

# Check CoreDNS logs
kubectl -n kube-system logs -l k8s-app=kube-dns -f

# Check CoreDNS config
kubectl -n kube-system get configmap coredns -o yaml

# Test DNS from a debug pod
kubectl run debug --image=busybox --rm -it -- nslookup my-service
```

### 1.8 Ingress — External Traffic Entry

Ingress is a Kubernetes object that defines routing rules for external HTTP/HTTPS traffic into the cluster. An Ingress Controller (nginx, traefik, HAProxy, Istio) watches Ingress objects and programs itself accordingly.

```
External Client
      │
      ▼
Load Balancer (NodePort or cloud LB)
      │
      ▼
Ingress Controller Pod (nginx)
  ├─ /api/* → api-service:8080
  ├─ /web/* → frontend-service:3000
  └─ /admin/* → admin-service:9000 (with auth middleware)
      │
      ▼
Service → Pod
```

```bash
# View ingress objects
kubectl get ingress -A
kubectl describe ingress <name>

# Check ingress controller logs
kubectl -n ingress-nginx logs -l app=ingress-nginx -f

# View nginx config generated by ingress controller
kubectl -n ingress-nginx exec <nginx-pod> -- nginx -T | grep server_name

# Test ingress routing
curl -H "Host: myapp.example.com" http://<node-ip>:<nodeport>/api/health
```

### 1.9 NetworkPolicy — Kubernetes Firewall

NetworkPolicy is the Kubernetes equivalent of firewall rules. It controls which pods can communicate with which other pods (and external endpoints). Without NetworkPolicy, all pods can talk to all other pods by default.

```yaml
# Example: only allow payments service to talk to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-payments-only
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database          # this policy applies to database pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: payments      # only allow from payments pods
    ports:
    - protocol: TCP
      port: 5432
```

**Important:** NetworkPolicy requires the CNI plugin to enforce it. Flannel does not enforce NetworkPolicy. Calico, Cilium, and Weave do. If you apply NetworkPolicy with Flannel, it is silently ignored.

```bash
# View network policies
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <namespace>

# Test if NetworkPolicy is working
# From payments pod → database should work
kubectl exec -it <payments-pod> -- nc -zv database-service 5432

# From other pod → database should be blocked
kubectl exec -it <other-pod> -- nc -zv database-service 5432
# Should timeout if NetworkPolicy is enforced
```

### 1.10 Service Mesh — Istio and the Sidecar Model

A service mesh solves three problems that neither Kubernetes Services nor NetworkPolicy fully address:
1. **mTLS between all services** — encrypt and authenticate every pod-to-pod call
2. **Traffic management** — canary deployments, circuit breaking, retries, timeouts at the proxy layer
3. **Observability** — automatic distributed tracing, metrics, access logs without changing application code

**How Istio works:**

Istio injects a sidecar container (Envoy proxy) into every pod. All traffic in and out of the pod is transparently redirected through the sidecar using iptables rules. The application thinks it is talking directly to the other service — the sidecar handles TLS, retry, and telemetry invisibly.

```
Pod A                              Pod B
┌─────────────────┐                ┌─────────────────┐
│  App Container  │                │  App Container  │
│  (port 8080)    │                │  (port 8080)    │
│       │         │                │       ▲         │
│  iptables       │                │  iptables       │
│  redirect       │                │  redirect       │
│       │         │                │       │         │
│  Envoy Sidecar  │──── mTLS ─────▶│  Envoy Sidecar  │
│  (port 15001)   │                │  (port 15001)   │
└─────────────────┘                └─────────────────┘
```

```bash
# Check if sidecar injected
kubectl get pod <name> -o jsonpath='{.spec.containers[*].name}'
# Should show: your-app istio-proxy

# View Envoy configuration
kubectl exec -it <pod> -c istio-proxy -- pilot-agent request GET /config_dump

# Check mTLS status
kubectl exec -it <pod> -c istio-proxy -- openssl s_client -connect <service>:8080

# Istio traffic management
kubectl get virtualservice -A
kubectl get destinationrule -A
kubectl get gateway -A

# Debug Istio routing
istioctl proxy-config routes <pod>
istioctl proxy-config clusters <pod>
istioctl analyze                      # check for config issues
```

**eBPF alternative (Cilium):** Cilium achieves similar results to a service mesh without sidecar injection — it uses eBPF programs in the kernel to enforce policy, encrypt traffic (WireGuard), and collect metrics. Lower overhead, simpler operations, no sidecar restart cycles.

---

## Part 2 — On-Premise Scale Networking

### 2.1 Data Centre Network Architecture — The Spine-Leaf Model

On-premise at scale uses a **spine-leaf topology** rather than the traditional three-tier (core-distribution-access) model. Spine-leaf is designed for east-west traffic (server-to-server, the dominant pattern in microservices) rather than north-south traffic (client-to-server).

```
                    ┌──────────────┐    ┌──────────────┐
  SPINE layer       │   Spine 1    │────│   Spine 2    │
  (no servers)      └──────┬───────┘    └───────┬──────┘
                           │    ╲       ╱        │
                      ┌────┘     ╲   ╱          └────┐
                      │           ╲ ╱                 │
                 ┌────▼────┐   ┌───▼────┐   ┌────▼────┐
  LEAF layer     │ Leaf 1  │   │ Leaf 2 │   │ Leaf 3  │
  (ToR switches) └────┬────┘   └───┬────┘   └────┬────┘
                      │            │               │
                  [Servers]    [Servers]       [Servers]
```

**Properties:**
- Every leaf is connected to every spine (full mesh)
- Any server can reach any other server in exactly 2 hops (leaf → spine → leaf)
- Predictable, consistent latency across the fabric
- Scale by adding more leaf switches without redesigning the network
- No spanning tree (ECMP routing instead)

**ECMP (Equal-Cost Multi-Path):** Multiple paths of equal cost exist between any two points. Traffic is distributed across all paths using hashing (typically on src/dst IP + port). Provides both load balancing and redundancy without STP's limitations.

### 2.2 BGP at Scale — The Routing Protocol of the Internet

BGP (Border Gateway Protocol) is used at scale for two reasons:
1. **Inter-AS routing** — how the internet works between ISPs and enterprises
2. **Data centre routing** — how Calico and large-scale on-premise networks distribute pod routes

**BGP fundamentals:**

BGP is a path-vector protocol. Routers (peers) exchange reachability information: "I can reach these prefixes, via this path." Unlike OSPF (which converges on shortest path), BGP is policy-driven — you control which routes are accepted, preferred, and advertised.

```
AS 65001 (Your DC)           AS 65002 (ISP)
┌─────────────────┐          ┌─────────────────┐
│  Border Router  │──eBGP────│   ISP Router    │
│  10.0.0.1       │          │   203.0.113.1   │
│                 │          │                 │
│  Announces:     │          │  Announces:     │
│  203.0.113.0/24 │          │  0.0.0.0/0      │
└─────────────────┘          └─────────────────┘

iBGP inside your AS (between internal routers):
  All internal routers learn routes from border router
  ASN stays the same inside
```

**Calico BGP in Kubernetes:**
```bash
# View BGP peers (Calico)
calicoctl get bgppeer
calicoctl get bgpconfiguration

# View routes being advertised
calicoctl node status
# Shows: BGP state, peers, routes learned/advertised

# View BGP route table on a node
bird -r show protocol all          # BIRD is Calico's BGP daemon
birdc show route                   # all routes

# Check if pod routes are being advertised
ip route show | grep bird          # routes installed by BIRD
```

### 2.3 Load Balancing at Scale

**Layer 4 vs Layer 7 load balancing:**

| | L4 (TCP/UDP) | L7 (HTTP/gRPC) |
|---|---|---|
| Visibility | IP + Port | HTTP headers, path, method, cookies |
| Speed | Faster (no protocol parsing) | Slower (must parse payload) |
| Sticky sessions | By src IP or src IP+port | By cookie or header |
| Health check | TCP connect | HTTP response code |
| Use case | Database connections, raw TCP | Web services, APIs, gRPC |
| Examples | HAProxy TCP, IPVS, AWS NLB | nginx, HAProxy HTTP, AWS ALB |

**HAProxy — production-grade load balancer:**

```
# /etc/haproxy/haproxy.cfg

frontend web_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/app.pem
    default_backend web_servers

backend web_servers
    balance roundrobin                    # algorithm: roundrobin, leastconn, source
    option httpchk GET /health            # health check endpoint
    http-check expect status 200
    server web1 10.0.1.10:8080 check inter 2s rise 2 fall 3
    server web2 10.0.1.11:8080 check inter 2s rise 2 fall 3
    server web3 10.0.1.12:8080 check inter 2s rise 2 fall 3

# Sticky sessions (by cookie)
backend sticky_backend
    balance roundrobin
    cookie SERVERID insert indirect nocache
    server web1 10.0.1.10:8080 cookie web1 check
    server web2 10.0.1.11:8080 cookie web2 check
```

```bash
# HAProxy runtime stats
echo "show stat" | socat stdio /var/run/haproxy/admin.sock | cut -d',' -f1,2,18,19
# Shows: backend name, server name, status, active connections

echo "show info" | socat stdio /var/run/haproxy/admin.sock
# Shows: uptime, current connections, max connections, requests/sec

# Drain a server (remove from rotation without dropping existing connections)
echo "set server web_backend/web1 state drain" | socat stdio /var/run/haproxy/admin.sock

# Put server back
echo "set server web_backend/web1 state ready" | socat stdio /var/run/haproxy/admin.sock
```

**Keepalived — Virtual IP for HA load balancers:**

Without Keepalived, a single load balancer is a single point of failure. Keepalived uses VRRP (Virtual Router Redundancy Protocol) to maintain a virtual IP (VIP) that floats between two HAProxy nodes.

```
HAProxy Primary (192.168.1.10) ──VRRP──▶ HAProxy Secondary (192.168.1.11)
        │
        │ VIP: 192.168.1.100
        │ (clients connect here)
        │
  If primary fails:
        │
        ▼
HAProxy Secondary takes VIP 192.168.1.100
  (clients reconnect, ~1-2 second failover)
```

```bash
# keepalived.conf (primary)
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100                          # higher = preferred master
    advert_int 1                          # VRRP advertisement interval
    authentication {
        auth_type PASS
        auth_pass secretpassword
    }
    virtual_ipaddress {
        192.168.1.100/24                  # the VIP
    }
}

# Check VRRP state
systemctl status keepalived
ip addr show eth0 | grep 192.168.1.100   # VIP present = this node is master
journalctl -u keepalived -f
```

### 2.4 MetalLB — LoadBalancer Services for On-Premise Kubernetes

In cloud environments, `Service type: LoadBalancer` provisions a cloud load balancer automatically. On-premise, this does nothing by default. MetalLB fills this gap — it gives bare-metal Kubernetes clusters a LoadBalancer implementation.

**MetalLB modes:**

**L2 mode (simpler):** MetalLB assigns an IP from a configured pool and one node announces that IP via ARP (for IPv4) or NDP (for IPv6). Traffic arrives at that node, then kube-proxy distributes to pods. Single-node ingress — no actual load balancing at L2.

**BGP mode (production):** MetalLB peers with your ToR switches via BGP and advertises Service IPs. The physical network load-balances across nodes using ECMP. True multi-path load balancing.

```bash
# View MetalLB config
kubectl -n metallb-system get configmap config -o yaml

# View IP address pools
kubectl get ipaddresspool -A      # MetalLB v0.13+
kubectl get l2advertisement -A
kubectl get bgpadvertisement -A

# Check MetalLB speaker logs (handles BGP/ARP)
kubectl -n metallb-system logs -l component=speaker

# Verify service got external IP
kubectl get svc <name>
# EXTERNAL-IP should show an IP from your MetalLB pool
```

### 2.5 Network Observability at Scale

When you have hundreds of nodes and thousands of pods, "I can't reach service X" requires systematic observability tools, not manual packet tracing.

```bash
# Cilium Hubble — eBPF-based network visibility
hubble observe --namespace production --last 100
hubble observe --pod payments-xxx --verdict DROPPED
hubble observe --to-service database --protocol TCP

# Visualise traffic flows
hubble observe --output json | jq '.flow | select(.verdict=="DROPPED")'

# Traditional packet capture at scale
# Capture on specific node
tcpdump -i any -n port 8080 -w /tmp/capture.pcap

# Distributed capture with ksniff (kubectl plugin)
kubectl sniff <pod> -p -f "port 8080" -o /tmp/capture.pcap

# Network performance baseline
# iperf3 between nodes (run server on node1, client on node2)
iperf3 -s                              # on node 1
iperf3 -c <node1-ip> -t 30 -P 4       # on node 2 — 4 parallel streams, 30 sec

# Check MTU across the path (VXLAN overhead)
ping -M do -s 1472 <destination>       # 1472 + 28 header = 1500 MTU
# If VXLAN: max payload is 1450 (50 bytes VXLAN overhead)
ping -M do -s 1422 <vxlan-destination> # test VXLAN path

# Conntrack table (stateful connection tracking)
conntrack -L                           # list tracked connections
conntrack -L | wc -l                   # total tracked connections
cat /proc/sys/net/netfilter/nf_conntrack_count    # current
cat /proc/sys/net/netfilter/nf_conntrack_max      # max
# If count approaches max → connections start failing silently
sysctl -w net.netfilter.nf_conntrack_max=524288   # increase limit
```

### 2.6 Common At-Scale Network Problems and Fixes

**Problem 1: Conntrack table exhaustion**

At scale, Linux's connection tracking table fills up. New connections are silently dropped — no error, just timeout.

```bash
# Symptom: intermittent connection failures, no obvious cause
dmesg | grep "nf_conntrack: table full"
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Fix
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_buckets=262144
echo "net.netfilter.nf_conntrack_max=1048576" >> /etc/sysctl.conf

# Reduce timeout for TIME_WAIT connections
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30  # default 120s
```

**Problem 2: MTU mismatch with VXLAN**

VXLAN adds 50 bytes of overhead. If the physical network MTU is 1500 and pods try to send 1500-byte packets, they get silently dropped after encapsulation (1550 > 1500).

```bash
# Symptom: small packets work, large packets silently dropped
# HTTP works, file transfers fail, database connections drop randomly

# Test MTU
ping -M do -s 1472 <destination>       # should succeed (1472 + 28 = 1500)
ping -M do -s 1422 <destination>       # should succeed through VXLAN

# Fix: set pod network MTU to 1450 (1500 - 50 VXLAN overhead)
# In Flannel: edit kube-flannel ConfigMap, set "MTU": 1450
# In Calico: calicoctl patch felixconfiguration default --patch '{"spec":{"mtu":1450}}'

# Check current MTU
ip link show | grep mtu
kubectl exec -it <pod> -- ip link show eth0
```

**Problem 3: DNS storm (ndots:5 causing excessive queries)**

```bash
# Symptom: CoreDNS CPU high, external service latency high
# Check CoreDNS query rate
kubectl -n kube-system top pod -l k8s-app=kube-dns

# View CoreDNS metrics
kubectl -n kube-system port-forward svc/kube-dns 9153:9153
curl localhost:9153/metrics | grep coredns_dns_request_count

# Fix option 1: increase CoreDNS replicas
kubectl -n kube-system scale deployment coredns --replicas=4

# Fix option 2: enable NodeLocal DNSCache (DNS agent on every node)
# Intercepts DNS queries at the node level, reduces CoreDNS load by 60-70%

# Fix option 3: reduce ndots in pod spec
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "1"
```

**Problem 4: kube-proxy iptables at scale (1000+ services)**

```bash
# Symptom: service latency increases as more services are added
# iptables has O(n) lookup — 1000 services = 10000+ rules

# Check rule count
iptables -t nat -L | wc -l

# Check kube-proxy sync time
kubectl -n kube-system logs -l k8s-app=kube-proxy | grep "sync took"

# Fix: switch to IPVS mode
kubectl -n kube-system edit configmap kube-proxy
# Set: mode: "ipvs"
kubectl -n kube-system rollout restart daemonset kube-proxy

# Verify IPVS
ipvsadm -L -n | head -20
```

---

## Part 3 — On-Premise vs Cloud Networking Comparison

| Concern | On-Premise | Cloud (AWS/GCP/Azure) |
|---------|-----------|----------------------|
| Load Balancer | MetalLB + HAProxy + Keepalived | Native LB service (managed) |
| Ingress IP | MetalLB pool from your IPAM | Automatic from cloud |
| BGP peering | Manual with ToR switches | Not required (overlay) |
| Network policy | Calico/Cilium (self-managed) | VPC Security Groups + K8s policy |
| DNS | CoreDNS + internal bind/unbound | Route 53 or Cloud DNS |
| Private connectivity | MPLS, VPN, dark fibre | VPC Peering, PrivateLink |
| MTU management | Manual (VXLAN overhead) | Usually handled by cloud |
| Conntrack limits | Must tune manually | Scaled automatically |
| Monitoring | Prometheus + Grafana (self-managed) | Cloud-native (CloudWatch etc.) |

---

## Part 4 — Production Debugging Runbook

### "Pod cannot reach another pod"

```bash
# Step 1: Confirm both pods are Running and Ready
kubectl get pod <pod-a> <pod-b> -o wide   # also shows node assignment

# Step 2: Are they on same or different nodes?
# Same node → veth/bridge issue
# Different nodes → overlay/routing issue

# Step 3: Test connectivity from pod
kubectl exec -it <pod-a> -- ping <pod-b-ip>
kubectl exec -it <pod-a> -- nc -zv <pod-b-ip> <port>

# Step 4: Check NetworkPolicy
kubectl get networkpolicy -n <namespace>
# Any policy that might be blocking?

# Step 5: Check CNI plugin health
kubectl get pod -n kube-system | grep -E "calico|flannel|cilium|weave"
# Any not Running?

# Step 6: Check node routes
ip route show | grep <pod-cidr>         # on the source node
# Route for destination pod's subnet should exist

# Step 7: Packet capture
tcpdump -i any -n host <pod-b-ip>       # on source node — do packets leave?
tcpdump -i any -n host <pod-b-ip>       # on destination node — do packets arrive?
```

### "Service is not reachable"

```bash
# Step 1: Does the service have endpoints?
kubectl get endpoints <service>
# Empty endpoints → pods not matching label selector or not Ready

# Step 2: Can we reach the ClusterIP directly?
kubectl exec -it <pod> -- nc -zv <clusterip> <port>

# Step 3: Check kube-proxy rules
iptables -t nat -L KUBE-SERVICES -n | grep <clusterip>
# If no rule → kube-proxy not synced, try: kubectl rollout restart -n kube-system daemonset kube-proxy

# Step 4: DNS resolving correctly?
kubectl exec -it <pod> -- nslookup <service-name>
# Should return ClusterIP

# Step 5: Check pod health
kubectl describe endpoints <service>    # which pods are included/excluded and why
```

---

## Interview One-Liners — Scale and Kubernetes

| Question | Answer |
|----------|--------|
| How do pods on different nodes communicate? | CNI plugin sets up routes (overlay: VXLAN encapsulation, underlay: BGP routing). Pod IPs are cluster-wide routable. |
| What is a ClusterIP? | Virtual IP for a Service. kube-proxy programs iptables/IPVS to DNAT traffic to a real pod IP. Never actually exists on any interface. |
| VXLAN overhead problem? | Adds 50 bytes per packet. Set pod network MTU to 1450 (not 1500) to avoid fragmentation. |
| How does Kubernetes DNS work? | CoreDNS runs as a pod with a ClusterIP. All pods have resolv.conf pointing to it. Resolves service-name.namespace.svc.cluster.local. |
| ndots:5 problem? | External DNS names with fewer than 5 dots get 3 unnecessary search-domain attempts before resolving. Fix: use FQDNs with trailing dot or reduce ndots. |
| NetworkPolicy not working? | Flannel does not enforce NetworkPolicy. Need Calico, Cilium, or Weave. Policy is silently ignored otherwise. |
| How to give on-premise K8s a LoadBalancer service? | MetalLB. L2 mode uses ARP (single node ingress). BGP mode peers with physical switches for true ECMP load balancing. |
| Spine-leaf vs three-tier? | Spine-leaf: any-to-any in 2 hops, consistent latency, scales horizontally, designed for east-west. Three-tier: designed for north-south, spanning tree limitations, unequal path costs. |
| What is conntrack table exhaustion? | Linux stateful connection tracking table fills up. New connections silently dropped. Fix: increase nf_conntrack_max, reduce TIME_WAIT timeout. |
| kube-proxy performance problem at scale? | iptables is O(n) per service — 1000 services = 10000+ rules per packet. Fix: switch kube-proxy to IPVS mode (O(1) hash lookup). |
| What is a service mesh? | Sidecar proxies (Envoy) injected into every pod handle mTLS, retries, circuit breaking, and observability without app code changes. |
| eBPF vs sidecar mesh? | eBPF (Cilium) achieves mesh features in kernel without sidecar injection. Lower overhead, no sidecar lifecycle issues, no 2x context switches per request. |
| How does Calico BGP work? | Each node runs BIRD BGP daemon. Advertises pod CIDR (e.g., 10.244.1.0/24) to ToR switches via iBGP or eBGP. Physical network installs routes. No encapsulation. |
