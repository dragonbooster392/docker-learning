# 07 — Container Networking Deep Dive

This file covers what Docker does **under the hood** to create container networks. Understanding this is essential for Kubernetes CNI plugins (Calico, Flannel, Cilium) — they use the same Linux primitives.

---

## The Three Linux Primitives

Everything Docker networking does is built on three kernel features:

```
1. Network Namespaces    → Isolation (each container gets its own network stack)
2. veth pairs            → Connectivity (virtual ethernet cables)
3. iptables / netfilter  → Routing and NAT (port mapping, internet access)
```

---

## Network Namespaces

A network namespace gives a process its **own isolated network stack**:

```
Host namespace:                    Container namespace:
┌──────────────────┐              ┌──────────────────┐
│ ens192: 10.12.24.141            │ eth0: 172.17.0.2 │
│ docker0: 172.17.0.1             │                  │
│ lo: 127.0.0.1    │              │ lo: 127.0.0.1    │
│ Routes: default   │              │ Routes: default   │
│   via 10.12.24.1  │              │   via 172.17.0.1  │
│ iptables: (rules) │              │ iptables: (empty) │
│ Ports: 22, 80...  │              │ Ports: 80         │
└──────────────────┘              └──────────────────┘

Each namespace has its own:
  - Interfaces (eth0, lo)
  - IP addresses
  - Routing table
  - iptables rules
  - Port space (both can use port 80 without conflict!)
```

### Creating a Network Namespace (Manual)

```bash
# Create a namespace
$ ip netns add my-namespace

# List namespaces
$ ip netns list
my-namespace

# Run a command inside the namespace
$ ip netns exec my-namespace ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

# Only loopback exists — no network connectivity!
```

```
Docker does this automatically for each container.
Docker stores its namespaces differently than ip netns (uses /proc/PID/ns/net),
but the kernel mechanism is identical.
```

### Seeing a Container's Namespace

```bash
# Get the container's PID on the host
$ docker inspect my-nginx --format '{{.State.Pid}}'
28451

# Enter the container's network namespace
$ nsenter -t 28451 -n ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8 scope host lo
2: eth0@if42: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 172.17.0.2/16 scope global eth0

# This is what the container sees.
# eth0@if42 means "this eth0 is paired with interface index 42 on the host"
```

---

## veth Pairs (Virtual Ethernet)

A veth pair is a **virtual cable** with two ends. What goes in one end comes out the other.

```
┌────────────────┐     veth pair     ┌────────────────┐
│  Container     │   (virtual cable)  │  Host          │
│                │                    │                │
│  eth0 ●────────┼────────────────────┼────● vethXXXX │
│  (172.17.0.2)  │                    │  (on docker0)  │
│                │                    │                │
└────────────────┘                    └────────────────┘
  Container namespace                   Host namespace
```

```
How Docker uses veth pairs:

1. Docker creates a veth pair: two virtual interfaces linked together
2. One end goes into the container's namespace → becomes eth0
3. Other end stays in the host namespace → attached to docker0 bridge
4. Traffic from eth0 comes out vethXXXX, enters the bridge, and vice versa
```

### Creating a veth Pair (Manual)

```bash
# Create a veth pair
$ ip link add veth-host type veth peer name veth-container

# Move one end into a namespace
$ ip link set veth-container netns my-namespace

# Configure the host end
$ ip addr add 172.17.0.1/16 dev veth-host
$ ip link set veth-host up

# Configure the container end (inside the namespace)
$ ip netns exec my-namespace ip addr add 172.17.0.2/16 dev veth-container
$ ip netns exec my-namespace ip link set veth-container up
$ ip netns exec my-namespace ip route add default via 172.17.0.1

# Now the namespace can reach the host, and vice versa!
$ ip netns exec my-namespace ping -c 1 172.17.0.1
64 bytes from 172.17.0.1: icmp_seq=1 ttl=64 time=0.031 ms
```

### Seeing veth Pairs on Your Machine

```bash
# On the host, see all veth interfaces
$ ip link show type veth
42: vethXXXXXX@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> master docker0
44: vethYYYYYY@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> master docker0

# The @if2 means "my peer is interface index 2 in another namespace"
# "master docker0" means "attached to the docker0 bridge"

# Match a veth to its container:
$ docker exec my-nginx cat /sys/class/net/eth0/iflink
42
# Container's eth0 is paired with host interface index 42 (vethXXXXXX)
```

---

## The Bridge (docker0)

A Linux bridge works like a **virtual network switch** — it connects multiple veth endpoints together:

```
                    docker0 bridge (172.17.0.1)
                    ┌──────────────────────────┐
                    │    Virtual Switch         │
                    │                           │
    vethA ──────────┤ port 1                    │
    (Container A)   │                           │
                    │                           │
    vethB ──────────┤ port 2                    │
    (Container B)   │                           │
                    │                           │
    vethC ──────────┤ port 3                    │
    (Container C)   │                           │
                    └──────────────────────────┘
                              │
                         IP: 172.17.0.1
                     (gateway for containers)
```

```
What the bridge does:
  - Connects all veth host-side endpoints together
  - Acts as the default gateway for containers (172.17.0.1)
  - Forwards frames between containers (Layer 2, like a switch)
  - Enables host ↔ container communication

Container A → Container B (same bridge):
  A's eth0 → vethA → docker0 bridge → vethB → B's eth0
  Direct Layer 2 forwarding. No NAT. Fast.

Container A → Internet:
  A's eth0 → vethA → docker0 → iptables NAT → ens192 → internet
  NAT translates 172.17.0.2 to 10.12.24.141 (host IP)
```

### Seeing the Bridge

```bash
# Show bridge details
$ ip addr show docker0
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 172.17.0.1/16 scope global docker0
    ether 02:42:ce:eb:65:d8

# Show what's connected to the bridge
$ bridge link show
42: vethXXXXXX@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP>
44: vethYYYYYY@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP>

# Or with brctl (if installed)
$ brctl show
bridge name   bridge id            STP enabled   interfaces
docker0       8000.0242ceeb65d8    no            vethXXXXXX
                                                  vethYYYYYY
```

---

## iptables Rules — How Docker Routes Traffic

Docker creates iptables rules for two purposes:
1. **Outbound NAT** — containers reaching the internet
2. **Port mapping** — external traffic reaching containers

### Outbound NAT (MASQUERADE)

```bash
$ iptables -t nat -L POSTROUTING -n
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  0    --  172.17.0.0/16       0.0.0.0/0

# This rule says:
# Source: 172.17.0.0/16 (any Docker container)
# Destination: 0.0.0.0/0 (anywhere)
# Action: MASQUERADE (replace source IP with host IP)
```

```
Container sends a packet:
  Source: 172.17.0.2:54321 → Destination: 172.253.118.102:443 (Google)

After MASQUERADE:
  Source: 10.12.24.141:32789 → Destination: 172.253.118.102:443

Google sees the request from 10.12.24.141 (host IP), not 172.17.0.2.
Google sends the reply to 10.12.24.141:32789.
iptables translates it back to 172.17.0.2:54321 and sends to the container.

This is Source NAT (SNAT) — same as your home router (networking concept 09).
```

### Port Mapping (DNAT)

```bash
# When you run: docker run -p 8080:80 nginx
$ iptables -t nat -L DOCKER -n
Chain DOCKER (2 references)
target  prot opt source        destination
DNAT    tcp  --  0.0.0.0/0     0.0.0.0/0    tcp dpt:8080 to:172.17.0.2:80

# This rule says:
# Protocol: TCP
# Incoming port: 8080 (on the host)
# Action: DNAT to 172.17.0.2:80 (rewrite destination to container)
```

```
External request arrives:
  Source: 10.66.49.170:54321 → Destination: 10.12.24.141:8080 (host)

After DNAT:
  Source: 10.66.49.170:54321 → Destination: 172.17.0.2:80 (container)

The packet is forwarded to the container.
The response goes back through iptables, which reverses the translation.

This is Destination NAT (DNAT) — networking concept 09.
```

### FORWARD Chain

```bash
$ iptables -L FORWARD -n
Chain FORWARD (policy DROP)
target          prot opt source          destination
DOCKER-USER     0    --  0.0.0.0/0       0.0.0.0/0
DOCKER-FORWARD  0    --  0.0.0.0/0       0.0.0.0/0

# FORWARD policy is DROP — nothing is forwarded by default.
# DOCKER-FORWARD chain has rules to allow Docker container traffic.
# DOCKER-USER chain is for your custom rules (insert before Docker's rules).
```

---

## Putting It All Together: The Complete Flow

### Container → Internet

```
Container A (172.17.0.2) pings 8.8.8.8:

1. Container creates packet:
   src=172.17.0.2 dst=8.8.8.8

2. Container's route table: default via 172.17.0.1 (docker0)
   → packet goes to docker0

3. Packet exits container namespace via veth pair:
   eth0 (container) → vethXXXXXX (host) → docker0 bridge

4. Host routing: 8.8.8.8 is not local → default via 10.12.24.1 (gateway)
   → packet needs to be forwarded out ens192

5. iptables FORWARD chain: Docker allows this
   iptables POSTROUTING: MASQUERADE
   → src rewritten: 172.17.0.2 → 10.12.24.141

6. Packet leaves ens192:
   src=10.12.24.141 dst=8.8.8.8

7. Reply comes back:
   src=8.8.8.8 dst=10.12.24.141

8. iptables conntrack: this is a reply to our MASQUERADE
   → dst rewritten: 10.12.24.141 → 172.17.0.2

9. Packet forwarded to docker0 → vethXXXXXX → container's eth0

10. Container receives: src=8.8.8.8 dst=172.17.0.2 ✓
```

### External → Container (Port Mapping)

```
Browser at 10.66.49.170 hits http://10.12.24.141:8080 (mapped to nginx:80):

1. Packet arrives at host:
   src=10.66.49.170:54321 dst=10.12.24.141:8080

2. iptables PREROUTING/DOCKER chain: DNAT rule matches port 8080
   → dst rewritten: 10.12.24.141:8080 → 172.17.0.2:80

3. Packet now: src=10.66.49.170:54321 dst=172.17.0.2:80
   → needs to be forwarded to docker0

4. iptables FORWARD chain: Docker allows this

5. Packet reaches docker0 → vethXXXXXX → container's eth0

6. nginx receives: src=10.66.49.170:54321 dst=172.17.0.2:80

7. nginx sends response: src=172.17.0.2:80 dst=10.66.49.170:54321

8. iptables conntrack: reverse the DNAT
   → src rewritten: 172.17.0.2:80 → 10.12.24.141:8080

9. Packet leaves host: src=10.12.24.141:8080 dst=10.66.49.170:54321 ✓
```

### Container → Container (Same Bridge)

```
Container A (172.17.0.2) talks to Container B (172.17.0.3):

1. A sends: src=172.17.0.2 dst=172.17.0.3

2. A's route table: 172.17.0.0/16 is on the local bridge
   → ARP for 172.17.0.3

3. ARP request goes through: eth0 → vethA → docker0 bridge

4. Bridge broadcasts ARP to all ports (like a switch)

5. Container B receives ARP, responds with its MAC

6. Packet: eth0(A) → vethA → docker0 → vethB → eth0(B)

7. Direct Layer 2 forwarding — no NAT, no iptables.
   This is why same-bridge communication is fast.
```

---

## Custom Bridge Networks Under the Hood

```bash
# When you create: docker network create my-net
# Docker creates a NEW bridge:

$ ip addr show br-abc123def456
5: br-abc123def456: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 172.18.0.1/16 scope global br-abc123def456

# Each custom network = separate bridge = separate IP range = isolation
#
# docker0:          172.17.0.0/16
# br-abc123def456:  172.18.0.0/16
# br-789def012abc:  172.19.0.0/16
#
# No routing between bridges by default = containers on different
# networks are isolated.
```

---

## Network Namespace Inspection Cheat Sheet

```bash
# Find container PID
PID=$(docker inspect --format '{{.State.Pid}}' CONTAINER)

# See the container's interfaces
nsenter -t $PID -n ip addr show

# See the container's route table
nsenter -t $PID -n ip route show

# See the container's ARP table
nsenter -t $PID -n ip neigh show

# See the container's iptables (usually empty)
nsenter -t $PID -n iptables -L -n

# See the container's listening ports
nsenter -t $PID -n ss -tlnp

# Trace a packet (on the host)
tcpdump -i docker0 -n host 172.17.0.2

# Trace on a specific veth
tcpdump -i vethXXXXXX -n
```

---

## How This Maps to Kubernetes CNI

```
Docker Primitive           → Kubernetes CNI Equivalent
──────────────────         → ──────────────────────────────
Network namespace          → Pod network namespace (one per pod)
veth pair                  → Same — CNI creates veth pairs
docker0 bridge             → cni0 / cbr0 bridge (or replaced by routes)
172.17.0.0/16              → Pod CIDR (e.g., 10.244.0.0/16)
iptables MASQUERADE        → Same — pods need NAT for external traffic
iptables DNAT (port map)   → kube-proxy iptables rules (Service → Pod)
docker network create      → CNI plugin auto-configures per node
Docker embedded DNS        → CoreDNS (runs as pods in kube-system)

Key differences:
  Docker: each container gets an IP, bridge-per-host
  K8s:    each POD gets an IP, CNI handles cross-node routing

  Docker: cross-host = overlay network (Swarm, not needed for K8s)
  K8s:    cross-node = CNI handles it:
          - Flannel: VXLAN tunnels between nodes
          - Calico:  BGP routing between nodes (no overlay)
          - Cilium:  eBPF-based networking (replacing iptables)
```

```
Calico example (most common in enterprise K8s):

  Node 1 (10.0.1.10)          Node 2 (10.0.2.20)
  Pod A: 10.244.0.5            Pod B: 10.244.1.8
  ┌──────────────┐             ┌──────────────┐
  │ eth0         │             │ eth0         │
  └──┬───────────┘             └──┬───────────┘
     │ veth pair                   │ veth pair
  ───┴────── cni0 ────            ─┴────── cni0 ────
         │                              │
    Route: 10.244.1.0/24          Route: 10.244.0.0/24
    via 10.0.2.20                 via 10.0.1.10
         │                              │
    ─────┴──────── Physical Network ────┴─────

  Pod A → Pod B:
    src=10.244.0.5 dst=10.244.1.8
    Node 1 route table: 10.244.1.0/24 via 10.0.2.20
    Packet sent to Node 2 → delivered to Pod B

  Same primitives (namespaces, veth, routes) — just scaled across nodes.
```

---

> **Next:** 08 — Docker Storage → Volumes, bind mounts, tmpfs — how containers persist data.
