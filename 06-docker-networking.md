# 06 — Docker Networking

## The Networking Problem

Each container has its **own network namespace** (from file 01). It sees its own interfaces, IP addresses, routing table, and ports. But containers need to:

```
1. Talk to the outside world (internet, APIs)
2. Talk to each other (app → database)
3. Be reachable from the host (browser → web server)
4. Be isolated from containers they shouldn't talk to
```

Docker solves this with **network drivers**.

---

## Docker Network Drivers

```
Driver      Description                      Use Case
──────      ────────────────────────────     ──────────────────────────
bridge      Default. Containers on a         Single-host container
            virtual bridge network.          communication (most common)

host        Container uses the HOST's        Maximum network performance.
            network directly. No isolation.  No port mapping needed.

none        No networking at all.            Security, batch processing.

overlay     Spans multiple Docker hosts.     Docker Swarm (skip for K8s)

macvlan     Container gets its own MAC       Legacy apps that need to
            on the physical network.         appear as physical devices.
```

We'll focus on **bridge** (90% of Docker usage), **host**, and **none**.

---

## Bridge Network (Default)

When you install Docker, it creates a virtual bridge called `docker0`:

```
Host Machine
┌───────────────────────────────────────────────────────────┐
│                                                            │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐           │
│   │Container │    │Container │    │Container │           │
│   │  App A   │    │  App B   │    │  App C   │           │
│   │172.17.0.2│    │172.17.0.3│    │172.17.0.4│           │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘           │
│        │ veth          │ veth          │ veth              │
│   ─────┴───────────────┴───────────────┴──────────        │
│              docker0 bridge (172.17.0.1)                   │
│                        │                                   │
│   ─────────────────────┴──────────────────────────        │
│              ens192 (10.12.24.141)                         │
│                        │                                   │
└────────────────────────┼───────────────────────────────────┘
                         │
                    Physical Network
```

```
How it works:
  1. docker0 bridge is created with IP 172.17.0.1/16
  2. Each container gets a veth pair (virtual ethernet cable):
     - One end inside the container (becomes eth0)
     - Other end connected to docker0 bridge
  3. Container gets IP from bridge subnet (172.17.0.2, .3, .4, ...)
  4. docker0 is the container's default gateway
  5. Docker sets up NAT (iptables) for outbound internet access
```

### Containers on the Default Bridge

```bash
# Start two containers on the default bridge
docker run -d --name web nginx
docker run -d --name db redis

# Check their IPs
docker inspect web | jq -r '.[0].NetworkSettings.IPAddress'
172.17.0.2

docker inspect db | jq -r '.[0].NetworkSettings.IPAddress'
172.17.0.3

# They can ping each other by IP
docker exec web ping -c 1 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.072 ms
```

### Default Bridge Limitations

```
⚠ The DEFAULT bridge (docker0) has problems:

1. No DNS resolution between containers:
   docker exec web ping db
   ping: db: Name or service not known    ← can't use container name!

2. ALL containers on the default bridge can talk to each other.
   No isolation.

3. You must use --link (deprecated) or IP addresses.

Solution: CREATE YOUR OWN bridge network.
```

---

## User-Defined Bridge Networks (Best Practice)

```bash
# Create a custom network
docker network create my-app-net

# Start containers on that network
docker run -d --name web --network my-app-net nginx
docker run -d --name db --network my-app-net redis
```

### DNS Works on Custom Networks!

```bash
# Containers can find each other BY NAME
docker exec web ping -c 1 db
PING db (172.18.0.3) 56(84) bytes of data.
64 bytes from db.my-app-net (172.18.0.3): icmp_seq=1 ttl=64 time=0.052 ms

# This is Docker's embedded DNS server (127.0.0.11)
docker exec web cat /etc/resolv.conf
nameserver 127.0.0.11
```

```
Docker's embedded DNS:
  Runs at 127.0.0.11 inside every container on custom networks.
  Resolves container names → container IPs automatically.

  "db"         → 172.18.0.3 (container named "db" on this network)
  "web"        → 172.18.0.2 (container named "web" on this network)
  "google.com" → forwarded to host DNS → 172.253.118.102

This is how microservices find each other:
  DATABASE_URL=postgres://db:5432/mydb    ← "db" resolves via Docker DNS
  REDIS_URL=redis://cache:6379            ← "cache" resolves via Docker DNS
```

### Network Isolation

```bash
# Create two separate networks
docker network create frontend
docker network create backend

# Web server on frontend
docker run -d --name web --network frontend nginx

# Database on backend
docker run -d --name db --network backend postgres

# App on BOTH networks
docker run -d --name app --network frontend myapp
docker network connect backend app
```

```
                frontend network              backend network
              ┌─────────────────┐          ┌─────────────────┐
              │  web     app    │          │  app      db    │
              │  (nginx) (myapp)│          │  (myapp)  (pg)  │
              └─────────────────┘          └─────────────────┘

  web can reach app ✓       (both on frontend)
  app can reach db ✓        (both on backend)
  web CANNOT reach db ✗     (different networks, no route)

This is network segmentation — same concept as VLANs in networking (file 04).
```

### Custom Bridge vs Default Bridge

```
Feature                 Default bridge    User-defined bridge
──────                  ──────────────    ──────────────────
DNS resolution          ✗ (IP only)       ✓ (by container name)
Network isolation       ✗ (all can talk)  ✓ (per network)
Connect/disconnect      ✗                 ✓ (on the fly)
Best practice           ✗                 ✓ ALWAYS use this
```

---

## Port Mapping

Containers have their own network namespace — ports inside a container are NOT accessible from the host by default.

### `-p` Flag: Publish a Port

```bash
docker run -d -p 8080:80 nginx
#              ─────┬────
#                    │
#           HOST:CONTAINER
#           8080 → 80
```

```
-p 8080:80 means:
  "Forward traffic arriving on HOST port 8080 to CONTAINER port 80"

  Browser → http://localhost:8080 → Docker → Container:80 → nginx

How it works (under the hood):
  Docker creates an iptables NAT rule:
  -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80

  DNAT = Destination NAT (networking concept 09!)
  Incoming traffic on port 8080 is rewritten to go to the container IP:port.
```

### Port Mapping Variations

```bash
# Specific host port → container port
docker run -d -p 8080:80 nginx
# Host:8080 → Container:80

# Random host port (Docker picks one)
docker run -d -p 80 nginx
# Host:32768 → Container:80  (check with docker ps)

# Bind to specific host IP
docker run -d -p 127.0.0.1:8080:80 nginx
# ONLY localhost:8080 → Container:80 (not reachable from network!)

docker run -d -p 0.0.0.0:8080:80 nginx
# ALL interfaces:8080 → Container:80 (reachable from network)

# Multiple ports
docker run -d -p 8080:80 -p 8443:443 nginx
# Host:8080 → Container:80
# Host:8443 → Container:443

# Port range
docker run -d -p 8000-8010:8000-8010 myapp
```

### Common Port Mapping Patterns

```
Service         Container Port    Typical Host Mapping
───────         ──────────────    ────────────────────
nginx/httpd     80, 443           -p 80:80 -p 443:443
Node.js         3000              -p 3000:3000
Python/Flask    5000              -p 5000:5000
Python/Django   8000              -p 8000:8000
MySQL           3306              -p 3306:3306
PostgreSQL      5432              -p 5432:5432
Redis           6379              -p 6379:6379
MongoDB         27017             -p 27017:27017
```

### Port Conflicts

```
$ docker run -d -p 8080:80 --name web1 nginx
$ docker run -d -p 8080:80 --name web2 nginx
Error: Bind for 0.0.0.0:8080 failed: port is already allocated

Two containers CANNOT use the same host port.
Solutions:
  1. Use different host ports: -p 8081:80
  2. Use random ports: -p 80 (Docker assigns one)
  3. Use host networking (container uses host ports directly)
```

---

## Host Network

```bash
docker run -d --network host nginx
```

```
Host networking = container uses the HOST's network stack directly.
No network namespace isolation. No port mapping needed.

  Container listens on port 80 → Host port 80 is used directly.
  No NAT, no bridge, no veth pairs.

┌─────────────────────────────────────┐
│  Host + Container (shared network)  │
│                                      │
│  nginx:80 ← directly on host        │
│  ens192: 10.12.24.141               │
│  docker0: doesn't apply             │
│                                      │
└─────────────────────────────────────┘

Pros:
  ✓ Best network performance (no NAT overhead)
  ✓ Container sees real host IP (useful for service discovery)
  ✓ No port mapping complexity

Cons:
  ✗ No port isolation (two containers can't use the same port)
  ✗ No network namespace isolation (security risk)
  ✗ Only works on Linux (not Mac/Windows Docker Desktop)
```

```
When to use host networking:
  ✓ Performance-critical applications (avoid NAT overhead)
  ✓ Applications that need to bind many ports
  ✓ Monitoring tools that need to see host network
  ✗ NOT for general use — use bridge for isolation
```

---

## None Network

```bash
docker run -d --network none myapp
```

```
No network at all. Container only has a loopback interface (127.0.0.1).
Cannot reach the internet or other containers.

$ docker exec myapp ip addr
1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8 scope host lo
    # That's it — no eth0, no external connectivity

When to use:
  ✓ Batch processing (no network needed)
  ✓ Security-sensitive workloads (zero network attack surface)
  ✓ Generating certificates or processing files
```

---

## Network Commands

```bash
# List all networks
$ docker network ls
NETWORK ID     NAME          DRIVER    SCOPE
abc123def456   bridge        bridge    local     ← default
789abc012def   host          host      local
321fed654cba   none          null      local
456abc789def   my-app-net    bridge    local     ← user-defined

# Create a network
docker network create my-net
docker network create --subnet=192.168.1.0/24 my-net

# Inspect a network (see connected containers)
docker network inspect my-app-net

# Connect a running container to a network
docker network connect my-app-net my-container

# Disconnect a container from a network
docker network disconnect my-app-net my-container

# Remove a network (must have no connected containers)
docker network rm my-net

# Remove all unused networks
docker network prune
```

### Inspect a Network

```bash
$ docker network inspect my-app-net | jq '.[0].Containers'
{
  "abc123...": {
    "Name": "web",
    "IPv4Address": "172.18.0.2/16"
  },
  "def456...": {
    "Name": "db",
    "IPv4Address": "172.18.0.3/16"
  }
}
```

---

## Container DNS Deep Dive

### How DNS Resolution Works

```
Container makes a DNS query for "db":

1. Container's /etc/resolv.conf points to 127.0.0.11 (Docker DNS)
2. Docker DNS checks: is "db" a container name on this network?
   - YES → return the container's IP (172.18.0.3)
   - NO  → forward the query to the host's DNS
3. Host DNS resolves external names (google.com → 172.253.118.102)
```

```
Container DNS resolution order:
  1. Container name    → "db" → 172.18.0.3
  2. Service alias     → (Docker Compose service names)
  3. Network alias     → docker run --network-alias myalias
  4. External DNS      → forwarded to host DNS resolver
```

### Network Aliases

```bash
# Give a container extra DNS names
docker run -d --name db-primary \
  --network my-net \
  --network-alias db \
  --network-alias database \
  postgres

# Now all three names resolve to the same container:
#   db-primary → 172.18.0.3
#   db         → 172.18.0.3
#   database   → 172.18.0.3
```

---

## Communication Patterns

### Container → Internet

```
Works by default (NAT via docker0):

Container (172.17.0.2) → docker0 (172.17.0.1) → iptables MASQUERADE → 
  ens192 (10.12.24.141) → internet

The container's private IP is hidden behind the host's IP.
This is NAT — exactly like your home router (networking concept 09).
```

### Container → Container (Same Network)

```
Direct communication on the bridge — no NAT:

Container A (172.18.0.2) → bridge → Container B (172.18.0.3)

Use container names as hostnames:
  curl http://web:80
  psql -h db -U postgres
  redis-cli -h cache
```

### Container → Container (Different Networks)

```
By default, containers on different networks CANNOT communicate.

  frontend network: web (172.18.0.2)
  backend network:  db (172.19.0.2)
  
  web cannot reach db — no route between the networks.

Solutions:
  1. Put both on the same network
  2. Connect a container to multiple networks
  3. Use port mapping (container → host → container)
```

### External → Container

```
Only works with port mapping (-p):

  Browser → http://host:8080 → iptables DNAT → Container:80

Without -p:
  The container has no published ports.
  It's invisible from outside the host.
```

---

## Summary

```
Network Type     Isolation    DNS    Performance    Use Case
────────────     ─────────    ───    ───────────    ──────────────
Default bridge   Low          ✗      Good           Don't use
Custom bridge    High         ✓      Good           ✓ Most apps
Host             None         N/A    Best           Performance-critical
None             Total        N/A    N/A            Security, batch

Always use user-defined bridge networks.
Never use the default bridge in production.
```

---

## How This Connects to Kubernetes

```
Docker Networking              → Kubernetes equivalent
──────────────────             → ──────────────────────────────
docker0 bridge                 → CNI plugin (Calico, Flannel, Cilium)
Container IP (172.17.0.x)      → Pod IP (every pod gets a unique IP)
Custom bridge network          → Namespace + Network Policy
Docker DNS (127.0.0.11)        → CoreDNS (cluster DNS)
Container name → IP            → Service name → ClusterIP
-p HOST:CONTAINER              → Service type: NodePort / LoadBalancer
Port mapping (iptables DNAT)   → kube-proxy (iptables or IPVS rules)
Network isolation (networks)   → NetworkPolicy (Calico, Cilium)
--network host                 → hostNetwork: true in Pod spec
```

```
Key difference:
  Docker: containers get IPs from bridge subnet (172.17.0.x/16)
  K8s:    pods get IPs from a cluster-wide CIDR (e.g., 10.244.0.0/16)
          Every pod can reach every other pod by IP (flat network).
          NetworkPolicies restrict what's allowed.
```

---

> **Next:** 07 — Container Networking Deep Dive → veth pairs, network namespaces, iptables rules — how bridge networking actually works under the hood (this is how K8s CNI works).
