# 09 — Docker Security

## The Fundamental Security Model

Docker containers are isolated by the **Linux kernel**, not by a hypervisor. Understanding the isolation boundaries — and their limits — is critical.

```
VM Security:                          Container Security:
┌─────────────────┐                   ┌─────────────────┐
│  Application    │                   │  Application    │
├─────────────────┤                   ├─────────────────┤
│  Guest OS       │                   │  (no guest OS)  │
│  (own kernel)   │ ← hard boundary  │                 │
├─────────────────┤                   ├─────────────────┤
│  Hypervisor     │                   │  Container      │
│                 │                   │  Runtime        │
├─────────────────┤                   ├─────────────────┤
│  Host OS + HW   │                   │  Host OS + HW   │
└─────────────────┘                   └─────────────────┘

VM: separate kernel = very strong isolation
Container: shared kernel = isolation depends on kernel security features
```

---

## Layer 1: Linux Namespaces (Isolation)

Namespaces control **what a container can see**:

```
Namespace    What's isolated                 Security Impact
─────────    ───────────────────────────     ─────────────────────────────
PID          Process IDs                     Can't see/kill host processes
NET          Network interfaces, IPs,        Own network stack, can't sniff
             routes, iptables                host traffic
MNT          Filesystem mounts               Own root filesystem, can't see
                                             host files (unless mounted)
UTS          Hostname, domain name           Own hostname (cosmetic)
IPC          Shared memory, semaphores       Can't access host IPC
USER         User and group IDs              Can be "root" inside but
                                             unprivileged on host

These are the SAME namespaces from file 01 and 07 — now from a security lens.
```

### User Namespace Mapping

```
Without user namespaces (Docker default):
  Container root (UID 0) = Host root (UID 0)
  If a container escapes its namespace, it's ROOT on the host.

With user namespaces (rootless Docker):
  Container root (UID 0) → Host UID 100000 (unprivileged)
  Even if a container escapes, it's a regular user on the host.
  
  ┌── Container ──────────┐    ┌── Host ─────────────────┐
  │ root     (UID 0)      │ →  │ user     (UID 100000)   │
  │ appuser  (UID 1000)   │ →  │ user     (UID 101000)   │
  └────────────────────────┘    └─────────────────────────┘
```

---

## Layer 2: cgroups (Resource Limits)

Cgroups control **what a container can use**:

```
Without cgroups:
  A container can:
    - Use ALL CPU cores → starve other containers
    - Use ALL RAM → trigger host OOM killer
    - Fork infinitely → fork bomb takes down the host
    - Write unlimited I/O → saturate disk

With cgroups:
  --cpus=2         → max 2 CPU cores
  --memory=512m    → max 512 MB RAM (OOM-killed if exceeded)
  --pids-limit=100 → max 100 processes (prevents fork bombs)
  --blkio-weight   → limit disk I/O

ALWAYS set resource limits in production.
A single runaway container should NOT affect others.
```

### cgroups v1 vs v2

```
cgroups v1 (legacy):
  - Multiple hierarchies (one per resource: cpu, memory, etc.)
  - Harder to manage
  - Still default on older systems

cgroups v2 (modern, unified):
  - Single hierarchy for all resources
  - Better resource accounting
  - Supports eBPF and Pressure Stall Information (PSI)
  - Required for rootless Docker on some distributions

Check your system:
  $ mount | grep cgroup
  cgroup2 on /sys/fs/cgroup type cgroup2  ← v2
  (or)
  cgroup on /sys/fs/cgroup/cpu type cgroup ← v1
```

---

## Layer 3: Linux Capabilities

Root in Linux has ~40 capabilities. Docker drops most of them by default.

```
Container root DOES have (by default):
  CAP_CHOWN           Change file ownership
  CAP_DAC_OVERRIDE    Bypass file read/write permissions
  CAP_FOWNER          Bypass permission checks on file owner
  CAP_KILL            Send signals to any process (in container)
  CAP_SETGID          Change group IDs
  CAP_SETUID          Change user IDs
  CAP_NET_BIND_SERVICE Bind to ports < 1024
  CAP_NET_RAW         Use raw sockets (ping)
  CAP_SYS_CHROOT      Use chroot

Container root does NOT have (dropped by default):
  CAP_SYS_ADMIN       Mount filesystems, load kernel modules ← DANGEROUS
  CAP_NET_ADMIN       Configure network interfaces, iptables
  CAP_SYS_PTRACE      Trace/debug other processes
  CAP_SYS_MODULE      Load kernel modules
  CAP_SYS_RAWIO       Direct I/O access to hardware
  CAP_SYS_TIME        Change system clock
```

### Managing Capabilities

```bash
# Drop ALL capabilities, add back only what's needed (best practice)
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# Drop a specific dangerous capability
docker run --cap-drop NET_RAW myapp

# Add a capability (if you need it)
docker run --cap-add SYS_PTRACE myapp    # for debugging/strace

# See what capabilities a container has:
docker exec myapp grep Cap /proc/1/status
```

```
Principle of least privilege:
  Start with --cap-drop ALL
  Add back ONLY what your app actually needs
  Most apps need ZERO capabilities

Common capabilities needed:
  NET_BIND_SERVICE  → web servers binding to port 80/443
  CHOWN/SETUID/SETGID → apps that switch users at startup
  NET_RAW           → apps that need ping (usually not needed)
```

---

## Layer 4: Seccomp (System Call Filtering)

Seccomp limits which **kernel system calls** a container can make.

```
Linux has ~330+ system calls. A typical container needs ~100.
Docker's default seccomp profile blocks ~44 dangerous syscalls:

Blocked by default:
  mount          → can't mount filesystems
  umount         → can't unmount filesystems
  reboot         → can't reboot the host
  kexec_load     → can't load a new kernel
  swapon/off     → can't manage swap
  clock_settime  → can't change the clock
  bpf            → can't load eBPF programs
  ptrace         → can't trace other processes
  ...

This means even if an attacker gets root inside a container,
they can't perform these operations.
```

```bash
# Run with default seccomp (recommended)
docker run myapp

# Run with NO seccomp (dangerous — for debugging only)
docker run --security-opt seccomp=unconfined myapp

# Run with custom seccomp profile
docker run --security-opt seccomp=custom-profile.json myapp
```

---

## Layer 5: AppArmor / SELinux

Mandatory Access Control (MAC) adds another layer on top of everything:

```
AppArmor (Ubuntu/Debian default):
  - Profiles define what files a process can access
  - Docker applies a default AppArmor profile to containers
  - Blocks: writing to /proc, /sys, mounting filesystems

SELinux (RHEL/CentOS/Fedora):
  - Labels define access permissions
  - Containers get specific SELinux labels
  - Prevents container processes from accessing host files

Both work alongside namespaces and capabilities:
  Namespaces: "container can't SEE host processes"
  Capabilities: "container can't DO privileged operations"
  AppArmor/SELinux: "container can't ACCESS certain files/resources"
```

---

## The Docker Daemon Security Problem

```
The Docker daemon (dockerd) runs as ROOT.

/var/run/docker.sock is the API endpoint:
  $ ls -la /var/run/docker.sock
  srw-rw---- 1 root docker /var/run/docker.sock

Anyone who can access this socket can:
  1. Run any container
  2. Mount ANY host directory: -v /:/host
  3. Run as --privileged (full host access)
  4. Read any file on the host
  5. Effectively become ROOT on the host

This means:
  ✗ Adding a user to the "docker" group = giving them root
  ✗ Mounting docker.sock inside a container = giving it root
  ✗ Exposing the daemon on TCP without TLS = remote root access
```

### The Docker Socket Mount Problem

```bash
# VERY COMMON (and dangerous!) pattern:
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp

# Inside the container, you can now:
docker run -v /:/host --privileged alpine
cat /host/etc/shadow
# You just read the host's password file from inside a container!

When to mount the socket:
  ✓ CI/CD agents (Jenkins, GitLab Runner) — but with caution
  ✓ Docker management tools (Portainer, Watchtower)
  ✗ NEVER in untrusted containers

Mitigation:
  - Use Docker-in-Docker (DinD) with a dedicated daemon
  - Use Kaniko for building images (no daemon needed)
  - Use rootless Docker or Podman
```

---

## Root vs Rootless Docker

### Root Mode (Default)

```
dockerd runs as:  root (PID 1's owner)
Container runs as: root inside container = root on host (same UID 0)

Attack scenario:
  1. Attacker exploits a vulnerability in your app
  2. Gets shell inside the container (as root)
  3. Finds a container escape vulnerability (kernel bug, misconfiguration)
  4. Escapes → they're ROOT on the host

Defense layers that must ALL fail:
  Namespace → Capabilities → Seccomp → AppArmor → Kernel
  
  If ANY layer holds, the attacker is contained.
  But defense in depth says: don't rely on a single layer.
```

### Rootless Mode

```
dockerd runs as:  regular user (UID 1000)
Container runs as: mapped UID (container root = host UID 100000)

Even if the attacker escapes ALL container isolation:
  They land as UID 100000 on the host — an unprivileged user.
  They can't read /etc/shadow, can't install packages, can't access
  other users' data.

Setup:
  $ dockerd-rootless-setuptool.sh install
  $ export DOCKER_HOST=unix:///run/user/1000/docker.sock

Limitations of rootless:
  ✗ Can't bind to ports < 1024 (use port mapping from high ports)
  ✗ Some storage drivers don't work (overlay2 needs kernel ≥ 5.11)
  ✗ Can't use --network host
  ✗ Some features require additional setup (cgroups v2)
```

### Podman (Rootless Alternative)

```
Podman = daemonless container runtime (Red Hat)
  - No daemon → no root daemon to attack
  - Compatible with Docker CLI (alias docker=podman)
  - Rootless by default
  - Used in RHEL, CentOS Stream, Fedora

$ podman run -d nginx   ← works like Docker, no daemon needed
```

---

## Running Containers as Non-Root

Even without rootless Docker, you should run your process as non-root INSIDE the container:

### In Dockerfile

```dockerfile
# Create a non-root user
RUN groupadd -r app && useradd -r -g app -d /app -s /sbin/nologin app

# Set ownership of app files
COPY --chown=app:app . /app/

# Switch to non-root user
USER app

# Now CMD runs as "app", not root
CMD ["python", "app.py"]
```

### At Runtime

```bash
# Override the user at runtime
docker run --user 1000:1000 myapp

# Verify
docker exec myapp whoami
# app (or "1000" if user doesn't exist in /etc/passwd)
docker exec myapp id
# uid=1000(app) gid=1000(app) groups=1000(app)
```

### Why Non-Root Matters

```
As root inside the container:
  - Can modify system files (/etc/passwd, /etc/shadow)
  - Can install packages (apt-get install nmap)
  - If escape happens: root on host
  - Can bind to ports < 1024

As non-root inside the container:
  - Can only modify app files (principle of least privilege)
  - Can't install packages
  - If escape happens: unprivileged on host
  - Can't bind to ports < 1024 (use EXPOSE 8000 instead of 80)
```

---

## The `--privileged` Flag

```bash
docker run --privileged myapp
```

```
--privileged gives the container EVERYTHING:
  ✓ ALL Linux capabilities
  ✓ Access to ALL host devices (/dev/*)
  ✓ Disabled seccomp filtering
  ✓ Disabled AppArmor/SELinux
  ✓ Can mount filesystems
  ✓ Can load kernel modules
  ✓ Full access to /proc and /sys

THIS IS EQUIVALENT TO RUNNING ON THE HOST AS ROOT.
There is almost no isolation left.

When it's "needed":
  - Docker-in-Docker (DinD) for CI/CD
  - Hardware access (GPU, USB devices)
  - Nested virtualization

Better alternatives:
  - Instead of --privileged, add only the specific capability needed:
    --cap-add SYS_ADMIN (still dangerous, but scoped)
  - Use Kaniko for container builds (no Docker daemon needed)
  - Use device flags: --device /dev/nvidia0 (specific device, not all)
```

---

## The Read-Only Filesystem

```bash
docker run --read-only myapp
```

```
Makes the container's root filesystem READ-ONLY.
The container can't write to any files except:
  - Mounted volumes
  - tmpfs mounts

This prevents:
  ✓ Attackers from installing tools (apt-get install nmap)
  ✓ Malware from modifying system binaries
  ✓ Applications from writing unexpected files

Common pattern:
  docker run --read-only \
    --tmpfs /tmp:rw,size=64m \
    --tmpfs /var/run:rw \
    -v mydata:/app/data \
    myapp

  /tmp and /var/run are writable (apps need them)
  Everything else is read-only
  Data volume for actual persistent data
```

---

## Image Security

### The Base Image Problem

```
Your Dockerfile:
  FROM some-random-user/python-app:latest

What you're trusting:
  1. The base image author (do you know them?)
  2. Every dependency in the image
  3. Docker Hub's integrity (was it tampered with?)
  4. The "latest" tag (could change to anything!)

Supply chain attack: attacker pushes a malicious image to Docker Hub
with the same name. You pull it. You're compromised.
```

### Choosing Safe Base Images

```
Trust level    Source                        Example
──────────     ──────                        ──────────────────
Official       Docker Hub "official" images  python:3.11-slim
                (maintained by Docker/vendor) nginx:1.25
                                              postgres:16

Verified       Publisher verified by Docker   bitnami/redis
               Hub (✓ badge)

Your own       Built by you/your org         mycompany/base:v1
               from a trusted base

Random         Unknown publisher             randomuser/myapp
               ⚠ HIGH RISK                  ← never use in production
```

### Image Scanning

Scan images for known vulnerabilities (CVEs):

```bash
# Docker Scout (built into Docker Desktop)
docker scout cves myapp:v1

# Trivy (open-source, widely used)
trivy image myapp:v1

# Example output:
$ trivy image python:3.11-slim
python:3.11-slim (debian 12.4)

Total: 85 (CRITICAL: 2, HIGH: 15, MEDIUM: 45, LOW: 23)

┌──────────────┬──────────────┬──────────┬─────────────────────┐
│   Library    │ Vulnerability│ Severity │ Fixed Version       │
├──────────────┼──────────────┼──────────┼─────────────────────┤
│ libssl3      │ CVE-2024-XXX │ CRITICAL │ 3.0.13-1~deb12u1   │
│ libsystemd0  │ CVE-2024-YYY │ HIGH     │ 252.22-1~deb12u1   │
└──────────────┴──────────────┴──────────┴─────────────────────┘
```

```
Scanning best practices:
  1. Scan in CI/CD pipeline (block deployment if CRITICAL found)
  2. Scan regularly (new CVEs are published daily)
  3. Use minimal base images (fewer packages = fewer vulnerabilities)
  4. Update base images regularly
```

### Secrets in Images

```
NEVER put secrets in Docker images:

  # BAD — secret is in a layer (visible in docker history!)
  ENV DB_PASSWORD=mysecretpassword
  COPY .env /app/.env
  RUN --mount=type=bind,source=.env,target=/app/.env pip install

  # Even if you delete it later, it's still in the layer:
  COPY secret.key /app/
  RUN use-secret.sh /app/secret.key
  RUN rm /app/secret.key    ← STILL in the COPY layer!

  How to check:
  $ docker history myapp | grep secret
  $ docker save myapp | tar x && grep -r "password" .

Safe alternatives:
  # Build-time (BuildKit secrets — never saved in layers):
  RUN --mount=type=secret,id=mytoken cat /run/secrets/mytoken

  # Runtime (environment variable or file mount):
  docker run -e DB_PASSWORD=secret myapp
  docker run -v /path/to/secrets:/run/secrets:ro myapp
```

---

## No New Privileges

```bash
docker run --security-opt no-new-privileges myapp
```

```
Prevents the process from gaining MORE privileges after starting:
  - Can't use setuid/setgid binaries (like su, sudo)
  - Can't escalate via exec

Even if an attacker finds a setuid binary inside the container,
they can't use it to become root.

This should be ON by default in production.
```

---

## Attack Scenarios

### Scenario 1: Container Escape via --privileged

```
Attack:
  1. Vulnerable web app runs in a --privileged container
  2. Attacker exploits the app, gets shell
  3. With --privileged, attacker can mount host filesystem:
     mount /dev/sda1 /mnt
     cat /mnt/etc/shadow
     echo "attacker ALL=(ALL) NOPASSWD:ALL" >> /mnt/etc/sudoers
  4. Attacker now has root access to the host

Prevention:
  ✗ Don't use --privileged
  ✓ Use specific --cap-add only for what's needed
  ✓ Run as non-root user
```

### Scenario 2: Docker Socket Exploitation

```
Attack:
  1. Container has docker.sock mounted (for CI/CD)
  2. Attacker gets shell in the container
  3. Uses Docker to start a new privileged container:
     docker run -v /:/host --privileged alpine
  4. Full host access via /host

Prevention:
  ✓ Don't mount docker.sock
  ✓ Use Kaniko for image builds
  ✓ If you must mount it, use --read-only and limit the user
```

### Scenario 3: Compromised Base Image

```
Attack:
  1. You use FROM randomuser/python-app:latest
  2. Attacker gains access to randomuser's Docker Hub account
  3. Pushes a version with a cryptocurrency miner
  4. Your next build pulls the compromised image
  5. Miner runs in your production containers

Prevention:
  ✓ Use official/verified base images
  ✓ Pin image versions with digest: FROM python@sha256:abc123...
  ✓ Scan images in CI/CD pipeline
  ✓ Use private registry with approved base images
```

### Scenario 4: Resource Abuse (Crypto Mining)

```
Attack:
  1. Attacker gets shell in container (e.g., via web vulnerability)
  2. No CPU/memory limits set
  3. Downloads and runs a crypto miner
  4. Miner uses 100% of ALL host CPUs
  5. Your other containers are starved

Prevention:
  ✓ Set --cpus and --memory on every container
  ✓ Set --pids-limit (prevents fork bombs)
  ✓ Use --read-only filesystem (can't download miner binary)
  ✓ Network egress rules (can't reach mining pools)
```

---

## Docker Security Checklist

```
Image Security:
  □ Use official/verified base images
  □ Pin image versions (never :latest in production)
  □ Scan images for vulnerabilities (Trivy, Docker Scout)
  □ No secrets in images (no ENV passwords, no COPY .env)
  □ Minimal base image (alpine or slim)
  □ Multi-stage builds (no build tools in final image)

Container Security:
  □ Run as non-root user (USER in Dockerfile)
  □ Drop capabilities: --cap-drop ALL, add back only needed
  □ Set --security-opt no-new-privileges
  □ Set resource limits: --cpus, --memory, --pids-limit
  □ Use --read-only filesystem + tmpfs for /tmp
  □ Never use --privileged
  □ Never mount docker.sock (unless absolutely necessary)

Runtime Security:
  □ Use custom bridge networks (not default bridge)
  □ Network segmentation (frontend/backend separate networks)
  □ Mount volumes as :ro when possible
  □ Use secrets management (not environment variables for passwords)
  □ Enable Docker Content Trust (image signing)
  □ Log and monitor container activity

Daemon Security:
  □ Don't expose Docker daemon on TCP without TLS
  □ Limit docker group membership
  □ Consider rootless Docker or Podman
  □ Keep Docker updated
```

---

## How This Connects to Kubernetes

```
Docker Security               → Kubernetes Equivalent
──────────────────            → ──────────────────────────────
USER (non-root)               → securityContext.runAsNonRoot: true
                                securityContext.runAsUser: 1000
--cap-drop ALL                → securityContext.capabilities.drop: ["ALL"]
--cap-add                     → securityContext.capabilities.add: [...]
--privileged                  → securityContext.privileged: true (avoid!)
--read-only                   → securityContext.readOnlyRootFilesystem: true
no-new-privileges             → securityContext.allowPrivilegeEscalation: false
--cpus, --memory              → resources.requests / resources.limits
Resource limits               → LimitRange (namespace defaults)
Seccomp profile               → seccompProfile in securityContext
AppArmor                      → apparmor annotation on Pod
Image scanning                → Admission controllers (OPA/Gatekeeper, Kyverno)
                                Block unscanned or vulnerable images
Don't run as root             → Pod Security Standards (PSS):
                                Restricted / Baseline / Privileged
Network isolation             → NetworkPolicy (Calico, Cilium)
Docker Content Trust          → Image signing + Sigstore/Cosign
--pids-limit                  → PID limits per pod (kubelet config)
```

```
Kubernetes Pod Security Standards (PSS):

  Privileged:  No restrictions (development only)
  Baseline:    Blocks known privilege escalations
               (no --privileged, no hostNetwork, no hostPID)
  Restricted:  Maximum security
               (non-root, drop ALL caps, read-only root, no escalation)

  Most production clusters should enforce Baseline or Restricted.
```

---

> **Next:** 10 — Docker Compose → Multi-container applications, service discovery, and development environments.
