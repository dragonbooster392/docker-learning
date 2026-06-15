# 14 — Hands-On Lab: Docker on This Machine

> Every command below was run on this machine: **ubuntu-VMware-Virtual-Platform**
> IP: 10.12.24.141/23 | OS: Ubuntu Linux | Docker Engine 28.3.3

---

## Lab 1: Docker Installation Verification

### 1.1 — Docker Version

```bash
$ docker version
```

```
Client: Docker Engine - Community
 Version:           28.3.3
 API version:       1.51
 Go version:        go1.24.5
 Git commit:        980b856
 OS/Arch:           linux/amd64

Server: Docker Engine - Community
 Engine:
  Version:          28.3.3
  API version:      1.51 (minimum version 1.24)
  Go version:       go1.24.5
  Git commit:       bea959c
  OS/Arch:          linux/amd64
 containerd:
  Version:          1.7.27
 runc:
  Version:          1.2.5
 docker-init:
  Version:          0.19.0
```

**What to read here:**

```
Client:  the docker CLI you type commands into
Server:  the Docker daemon (dockerd) running in the background

Version 28.3.3:  both must match (or be compatible)
API 1.51:        the REST API version between CLI ↔ daemon
containerd:      the container runtime daemon (manages container lifecycle)
runc:            the OCI runtime (actually creates the Linux container)
docker-init:     tiny init process for PID 1 inside containers (--init flag)
```

### 1.2 — Docker System Info

```bash
$ docker info
```

```
Server:
 Containers: 15
  Running: 2
  Paused: 0
  Stopped: 13
 Images: 53
 Server Version: 28.3.3
 Storage Driver: overlay2            ← union filesystem (file 03)
  Backing Filesystem: extfs          ← host filesystem is ext4
 Logging Driver: json-file           ← default log driver
 Cgroup Driver: systemd              ← cgroup v2 with systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
```

**What to read here:**

```
15 containers total, only 2 running     ← exited containers consume disk
53 images                                ← many are dangling (<none>)
overlay2 on extfs                        ← the layer system from file 03
Cgroup v2 + systemd                      ← modern Linux cgroup management
Network plugins: bridge, host, overlay   ← all network drivers from file 06
Swarm: inactive                          ← we use Kubernetes, not Swarm
```

### 1.3 — Disk Usage

```bash
$ docker system df
```

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          53        5         37.92GB   34.81GB (91%)
Containers      15        2         69.6kB    2.712kB (3%)
Local Volumes   352       4         47.79GB   47.63GB (99%)
Build Cache     983       0         33.48GB   33.48GB
```

**What to read here:**

```
Images:     53 total, only 5 in use → 91% reclaimable (34.81 GB wasted!)
Containers: 15 total, only 2 running → 13 stopped containers sitting around
Volumes:    352 total, only 4 in use → 99% reclaimable (47.63 GB wasted!)
Build Cache: 983 entries, all reclaimable → 33.48 GB from previous builds

Total waste: ~115 GB that could be reclaimed with:
  docker system prune -a --volumes

⚠️  Don't run prune on a production machine without checking what's in use first!
```

---

## Lab 2: Inspecting Running Containers

### 2.1 — List Running Containers

```bash
$ docker ps
```

```
CONTAINER ID   IMAGE                                         COMMAND              CREATED        STATUS                 NAMES
856512cbd0fa   ...http-s3-ecg-receiver-bdd:ftr-k12l-addon…   "/bin/sh"           7 weeks ago    Up 7 weeks (healthy)   amazing_archimedes
a0498212186d   gitlab/gitlab-runner:v18.5.0                  "/usr/bin/dumb-in…"  4 months ago   Up 2 months            gitlab-runner-prod-v18.5.0
```

**What to read here:**

```
CONTAINER ID:  first 12 chars of the full SHA256 ID
IMAGE:         the image this container was created from
COMMAND:       the entrypoint + cmd (PID 1 process)
CREATED:       when the container was first created
STATUS:        Up 7 weeks (healthy)  ← running with passing health check
               Up 2 months           ← running, no health check defined
NAMES:         auto-generated or user-specified
```

### 2.2 — Include Stopped Containers

```bash
$ docker ps -a
```

Shows all 15 containers. The stopped ones show:

```
STATUS
  Exited (0) 7 weeks ago     ← exit code 0 = clean shutdown
  Exited (0) 2 months ago    ← these are old GitLab Runner job containers
```

**Why stopped containers exist:**
The GitLab Runner creates a container per CI job, then exits. The containers
remain on disk until manually cleaned up. Each one is tiny (kilobytes) but
they clutter `docker ps -a`.

---

## Lab 3: Container Inspection Deep Dive

### 3.1 — Container State

```bash
$ docker inspect amazing_archimedes --format '{{.State.Status}} PID={{.State.Pid}} RestartCount={{.State.RestartCount}}'
```

```
running PID=3967997 RestartCount=0
```

**What this means:**

```
Status: running       ← one of: created, running, paused, restarting, exited, dead
PID: 3967997          ← the process ID on the HOST (not inside the container)
RestartCount: 0       ← this container never crashed and restarted
```

### 3.2 — Container Operating System

```bash
$ docker exec amazing_archimedes cat /etc/os-release
```

```
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.23.3
PRETTY_NAME="Alpine Linux v3.23"
```

The host runs Ubuntu, but the container runs Alpine Linux.
This is the whole point of containers — isolated userspace.

### 3.3 — Container Hostname

```bash
$ docker exec amazing_archimedes hostname
```

```
856512cbd0fa
```

By default, the container's hostname = its container ID.
You can override with `docker run --hostname myapp`.

### 3.4 — Container IP Address

```bash
$ docker inspect amazing_archimedes --format '{{.NetworkSettings.Networks.bridge.IPAddress}}'
```

```
172.17.0.3
```

This IP is on the docker0 bridge network (172.17.0.0/16).

### 3.5 — Container Security Settings

```bash
$ docker inspect amazing_archimedes --format 'Caps: {{.HostConfig.CapAdd}} CapDrop: {{.HostConfig.CapDrop}} Privileged: {{.HostConfig.Privileged}} ReadOnly: {{.HostConfig.ReadonlyRootfs}}'
```

```
Caps: [] CapDrop: [] Privileged: false ReadOnly: false
```

**What this means:**

```
CapAdd: []           ← no extra capabilities added
CapDrop: []          ← no capabilities dropped (gets DEFAULT set)
Privileged: false    ← good! not running in privileged mode
ReadOnly: false      ← root filesystem is writable (could be hardened)

Security improvement opportunities:
  --cap-drop ALL --cap-add <only-what-needed>
  --read-only
  --security-opt=no-new-privileges
```

---

## Lab 4: Namespaces — Proving Container Isolation

### 4.1 — Host Namespaces (PID 1 = systemd)

```bash
$ ls -la /proc/1/ns/
```

```
cgroup -> cgroup:[4026531835]
ipc    -> ipc:[4026531839]
mnt    -> mnt:[4026531841]
net    -> net:[4026531840]
pid    -> pid:[4026531836]
time   -> time:[4026531834]
user   -> user:[4026531837]
uts    -> uts:[4026531838]
```

### 4.2 — Container Namespaces (PID 3967997 on host)

```bash
$ ls -la /proc/3967997/ns/    # 3967997 is the container's PID on the host
```

```
cgroup -> cgroup:[4026532731]    ← DIFFERENT from host
ipc    -> ipc:[4026532729]       ← DIFFERENT from host
mnt    -> mnt:[4026532727]       ← DIFFERENT from host
net    -> net:[4026532732]       ← DIFFERENT from host
pid    -> pid:[4026532730]       ← DIFFERENT from host
time   -> time:[4026531834]      ← SAME as host (shared)
user   -> user:[4026531837]      ← SAME as host (shared)
uts    -> uts:[4026532728]       ← DIFFERENT from host
```

### 4.3 — Comparison Table

```
Namespace   Host ID         Container ID     Isolated?
─────────   ───────         ────────────     ─────────
cgroup      4026531835      4026532731       YES ← separate resource limits
ipc         4026531839      4026532729       YES ← separate shared memory
mnt         4026531841      4026532727       YES ← separate filesystem view
net         4026531840      4026532732       YES ← separate network stack
pid         4026531836      4026532730       YES ← separate process tree
time        4026531834      4026531834       NO  ← shares host clock
user        4026531837      4026531837       NO  ← shares host user table
uts         4026531838      4026532728       YES ← separate hostname
```

**Key insight:** 6 of 8 namespaces are different — the container has its own
filesystem, network, processes, hostname, IPC, and cgroups. Only time and user
namespaces are shared (user namespace isolation requires rootless Docker).

**This is what a container IS** — a process in separate namespaces.
Not a VM, not magic, just Linux kernel isolation.

### 4.4 — Cgroup Comparison

```bash
# Host process cgroup:
$ cat /proc/self/cgroup
0::/user.slice/user-0.slice/session-39871.scope

# Container's cgroup:
$ docker exec amazing_archimedes cat /proc/1/cgroup
0::/
```

The container sees itself at the root of the cgroup hierarchy (`/`),
but on the host it's actually nested under
`/system.slice/docker-856512cbd0fa....scope`.

---

## Lab 5: Networking Internals

### 5.1 — The docker0 Bridge

```bash
$ ip link show docker0
```

```
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether ce:ee:db:eb:65:d8 brd ff:ff:ff:ff:ff:ff
```

**What to read here:**

```
Interface #3:  docker0 was the third network interface created
state UP:      bridge is active (containers are connected)
MTU 1500:      standard Ethernet frame size
MAC address:   ce:ee:db:eb:65:d8 (randomly assigned)

This is the virtual switch from file 07.
All containers on the default bridge connect here.
```

### 5.2 — Veth Pairs (Virtual Ethernet Cables)

```bash
$ ip link show type veth
```

```
15628: vetha5e096e@if2: master docker0 state UP
    link/ether c2:22:3f:d2:f4:b6 brd ff:ff:ff:ff:ff:ff link-netnsid 6

27558: veth764e743@if2: master docker0 state UP
    link/ether 16:67:d4:ca:bb:63 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

**What to read here:**

```
2 running containers → 2 veth pairs:

  vetha5e096e ←──────→ eth0 inside amazing_archimedes (link-netnsid 6)
  veth764e743 ←──────→ eth0 inside gitlab-runner       (link-netnsid 0)

Each veth is:
  - One end on the host (plugged into docker0 bridge)
  - Other end inside the container's network namespace (appears as eth0)
  - "master docker0" = connected to the bridge
  - @if2 = the other end is interface #2 inside the container's netns
```

```
Packet flow visualization (from file 07):

  ┌─────────────────────────────────────────────────────────────────┐
  │ HOST                                                            │
  │                                                                 │
  │  ┌──────────────┐     ┌──────────────┐                          │
  │  │ amazing_     │     │ gitlab-      │                          │
  │  │ archimedes   │     │ runner       │                          │
  │  │ 172.17.0.3   │     │ 172.17.0.2   │                          │
  │  │  eth0 (if2)  │     │  eth0 (if2)  │                          │
  │  └──────┬───────┘     └──────┬───────┘                          │
  │         │ veth pair          │ veth pair                         │
  │  vetha5e096e           veth764e743                               │
  │         │                    │                                   │
  │         └────────┬───────────┘                                   │
  │               docker0                                            │
  │            172.17.0.1/16                                         │
  │                  │                                               │
  │               ens160 ──────→ physical network (10.12.24.141)     │
  └──────────────────────────────────────────────────────────────────┘
```

### 5.3 — Bridge Network Details

```bash
$ docker network inspect bridge
```

```json
{
    "Name": "bridge",
    "Driver": "bridge",
    "Subnet": "172.17.0.0/16",
    "Gateway": "172.17.0.1",
    "Containers": {
        "856512cbd0fa...": {
            "Name": "amazing_archimedes",
            "IPv4Address": "172.17.0.3/16"
        },
        "a0498212186d...": {
            "Name": "gitlab-runner-prod-v18.5.0",
            "IPv4Address": "172.17.0.2/16"
        }
    }
}
```

**What to read here:**

```
Subnet: 172.17.0.0/16           ← all containers get IPs in this range
Gateway: 172.17.0.1             ← this is docker0's IP (the "router")
2 containers connected:
  amazing_archimedes  → 172.17.0.3
  gitlab-runner       → 172.17.0.2
```

### 5.4 — iptables NAT Rules (How Containers Reach the Internet)

```bash
$ sudo iptables -t nat -L -n
```

```
Chain POSTROUTING (policy ACCEPT)
target      source           destination
MASQUERADE  172.17.0.0/16    0.0.0.0/0

Chain DOCKER (2 references)
target      source           destination
RETURN      0.0.0.0/0        0.0.0.0/0
```

**What to read here:**

```
MASQUERADE rule:
  Source: 172.17.0.0/16 (container traffic)
  Destination: 0.0.0.0/0 (anywhere)
  Action: MASQUERADE (replace source IP with host IP)

This is how containers reach the internet:
  Container (172.17.0.3) → docker0 → iptables MASQUERADE → ens160 (10.12.24.141) → internet

The external server sees traffic from 10.12.24.141, not 172.17.0.3.
Same as your home router's NAT from file 07.

No DNAT rules:
  No port mappings are published (-p flag was not used for these containers).
  If we ran: docker run -p 8080:80 nginx
  We'd see:  DNAT tcp 0.0.0.0:8080 → 172.17.0.x:80
```

---

## Lab 6: Image and Layer Inspection

### 6.1 — List Images

```bash
$ docker images | head -10
```

```
REPOSITORY                                TAG                           SIZE
eu-artifactory/.../greengrass             integ-53dd16df-260417121701   3.98GB
amazoncorretto                            17                            465MB
greengrass-ubuntu-test                    latest                        921MB
amazonlinux                               2                             165MB
amazonlinux                               2023                          149MB
dashboard-backend                         latest                        441MB
grafana/grafana                           latest                        761MB
```

**Things to notice:**

```
greengrass: 3.98GB! ← very large image, likely has too many layers or a fat base
amazoncorretto:17: 465MB ← Java runtime, expected size
amazonlinux:2 vs 2023: 165MB vs 149MB ← newer base = smaller
<none>:<none> images (not shown): dangling images from rebuilds
```

### 6.2 — Image Layer History

```bash
$ docker history amazoncorretto:17
```

```
IMAGE          CREATED       CREATED BY                                      SIZE
4e674c28378b   11 days ago   ENV JAVA_HOME=/usr/lib/jvm/java-17-amazon-co…   0B
<missing>      11 days ago   ENV LANG=C.UTF-8                                0B
<missing>      11 days ago   RUN |1 version=17.0.18.9-1 /bin/sh -c set -e…   299MB
<missing>      11 days ago   ARG version=17.0.18.9-1                         0B
<missing>      11 days ago   CMD ["/bin/bash"]                               0B
<missing>      11 days ago   COPY /rootfs/ / # buildkit                      165MB
```

**What to read here:**

```
2 actual layers (the rest are metadata with 0B size):

  Layer 1: COPY /rootfs/ /        → 165MB (this is the Amazon Linux base)
  Layer 2: RUN ... install java   → 299MB (JDK 17 installation)
  Total:                            465MB

The 0B entries (ENV, ARG, CMD) don't create layers — they only add metadata.
This matches what we learned in file 03 about Dockerfile instructions and layers.
```

### 6.3 — Image Digest (Content-Addressable ID)

```bash
$ docker image inspect amazoncorretto:17 --format '{{.Id}}'
```

```
sha256:4e674c28378b0dce8e86b5ae5f2ddfba1ff097b1aa234c300dd4dbab68014a21
```

This SHA256 is computed from the image's content. If even one byte changes,
the hash changes. This is how registries detect duplicates and verify integrity.

### 6.4 — Layer Count

```bash
$ docker image inspect amazoncorretto:17 --format '{{len .RootFS.Layers}} layers'
```

```
2 layers
```

Matches the `docker history` output — only RUN and COPY created actual layers.

---

## Lab 7: Overlay2 Storage on Disk

### 7.1 — Container's Filesystem Layers

```bash
$ docker inspect amazing_archimedes --format '{{.GraphDriver.Name}}'
```

```
overlay2
```

### 7.2 — Merged Directory (What the Container Sees)

```bash
$ docker inspect amazing_archimedes --format '{{.GraphDriver.Data.MergedDir}}'
```

```
/var/lib/docker/overlay2/f0b193ad43572bbc1fb132e319c5f506e23786743fc9ee860320cb6d502b2669/merged
```

This is the **union mount point** — where all layers are stacked and presented
as a single filesystem to the container. The container's `/` points here.

### 7.3 — Layer Count on Disk

```bash
# How many overlay2 layers exist for this container's image:
$ docker inspect amazing_archimedes --format '{{.GraphDriver.Data.LowerDir}}' | tr ':' '\n' | wc -l
```

```
18
```

18 read-only layers stacked together, plus 1 writable layer on top = the
container's complete filesystem.

### 7.4 — Total overlay2 Directories

```bash
$ ls /var/lib/docker/overlay2/ | wc -l
```

```
1375
```

1375 overlay2 directories from 53 images and 15 containers. Each image layer
gets its own directory. Shared layers between images only exist once on disk
(content-addressable storage from file 03).

---

## Lab 8: Docker System Health

### 8.1 — Full Disk Usage Breakdown

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          53        5         37.92GB   34.81GB (91%)
Containers      15        2         69.6kB    2.712kB (3%)
Local Volumes   352       4         47.79GB   47.63GB (99%)
Build Cache     983       0         33.48GB   33.48GB (100%)
```

```
Total disk used by Docker:   ~119 GB
Actually needed:             ~3.3 GB (active images + containers + volumes)
Wasted:                      ~116 GB

Breakdown:
  Images:      48 unused images                → 34.81 GB reclaimable
  Containers:  13 stopped containers           → 2.7 KB (tiny)
  Volumes:     348 unused GitLab Runner caches → 47.63 GB reclaimable
  Build Cache: 983 unused build layers         → 33.48 GB reclaimable
```

### 8.2 — Why So Much Waste?

```
This machine runs a GitLab Runner for CI/CD:

  1. Each CI job creates a container + volumes for build caches
  2. Jobs finish → containers exit (but aren't removed)
  3. Each pipeline may pull different images
  4. Build cache accumulates from every `docker build`

This is NORMAL for CI machines. Production machines should be leaner.
```

### 8.3 — How to Clean Up (DON'T run on prod without checking!)

```bash
# Remove stopped containers:
docker container prune

# Remove unused images:
docker image prune -a

# Remove unused volumes:
docker volume prune

# Remove everything unused (images + containers + volumes + build cache):
docker system prune -a --volumes

# See what WOULD be removed (dry run):
docker system prune -a --volumes --dry-run
```

---

## Lab 9: Quick Reference — Commands Used in This Lab

```
INFORMATION COMMANDS:
  docker version                          → client + server versions
  docker info                             → system-wide config
  docker system df                        → disk usage

CONTAINER COMMANDS:
  docker ps                               → running containers
  docker ps -a                            → all containers (including stopped)
  docker inspect <name>                   → full JSON details
  docker inspect <name> --format '...'    → extract specific fields
  docker exec <name> <command>            → run command inside container
  docker logs <name>                      → container stdout/stderr

NETWORK COMMANDS:
  docker network ls                       → list networks
  docker network inspect bridge           → bridge details + connected containers
  ip link show docker0                    → bridge interface on host
  ip link show type veth                  → veth pairs connecting containers
  iptables -t nat -L -n                   → NAT rules for container traffic

IMAGE COMMANDS:
  docker images                           → list images
  docker history <image>                  → layer history
  docker image inspect <image>            → full image metadata

STORAGE COMMANDS:
  docker volume ls                        → list volumes
  docker system df                        → disk usage summary

SECURITY / INTERNALS:
  ls /proc/<PID>/ns/                      → namespace IDs
  cat /proc/<PID>/cgroup                  → cgroup placement
  docker inspect --format '...HostConfig...'  → security settings
```

---

## What You Proved in This Lab

```
File 01 (Containers):     Containers are processes in separate namespaces (Lab 4)
File 02 (Architecture):   containerd 1.7.27 + runc 1.2.5 visible in docker version (Lab 1)
File 03 (Layers):         overlay2 layers visible on disk, history shows layer creation (Lab 6, 7)
File 06 (Networking):     docker0 bridge, container IPs on 172.17.0.0/16 (Lab 5)
File 07 (Deep Dive):      veth pairs, iptables MASQUERADE rule, packet flow (Lab 5)
File 08 (Storage):        overlay2 driver, 1375 dirs, MergedDir mount point (Lab 7)
File 09 (Security):       Namespace isolation, capability settings, cgroup separation (Lab 3, 4)
File 12 (Best Practices): 116 GB reclaimable disk, cleanup commands (Lab 8)
```

Every concept from the course is visible right here on this machine.
