# 01 — What Is a Container?

## The Problem Containers Solve

Before containers, deploying software looked like this:

```
Developer's laptop:  "It works on my machine!"
Test server:         Python 3.8, Ubuntu 18.04
Production server:   Python 3.6, CentOS 7  ← breaks

Developer: "But I tested it..."
Ops team:  "Your test environment doesn't match production."
```

The problem: applications depend on **specific versions** of libraries, runtimes, and OS packages. Different machines have different versions. Things break.

### The Old Solution: Virtual Machines

```
┌─────────────────────────────────┐
│         Your Application        │
├─────────────────────────────────┤
│    Libraries / Dependencies     │
├─────────────────────────────────┤
│        Guest OS (Ubuntu)        │  ← Full operating system (2-10 GB)
├─────────────────────────────────┤
│       Hypervisor (VMware)       │  ← Manages VMs
├─────────────────────────────────┤
│         Host OS (Linux)         │
├─────────────────────────────────┤
│           Hardware              │
└─────────────────────────────────┘
```

VMs work but are **heavy**:
- Each VM runs a **complete OS** (kernel, init system, everything)
- 2-10 GB per VM just for the OS
- Takes **30-60 seconds** to boot
- Uses significant CPU and RAM just to keep the guest OS running

### The New Solution: Containers

```
┌──────────┐ ┌──────────┐ ┌──────────┐
│  App A   │ │  App B   │ │  App C   │
│  Python  │ │  Node.js │ │  Go      │
│  3.11    │ │  20      │ │  1.22    │
├──────────┴─┴──────────┴─┴──────────┤
│         Container Runtime           │  ← Docker (thin layer)
├─────────────────────────────────────┤
│          Host OS (Linux)            │  ← ONE kernel, shared
├─────────────────────────────────────┤
│            Hardware                 │
└─────────────────────────────────────┘
```

Containers are **lightweight**:
- **No guest OS** — containers share the host kernel
- 5-500 MB per container (just your app + its dependencies)
- Starts in **milliseconds** (it's just a process)
- Near-zero overhead (no hypervisor, no guest OS)

---

## A Container Is Just a Process

This is the most important mental model:

> **A container is a normal Linux process with isolation and limits.**

When you run `docker run nginx`, Docker doesn't start a virtual machine. It starts a **regular Linux process** (nginx) and applies two things:

### 1. Isolation (Linux Namespaces)

Namespaces make the process **think it's alone** on the machine:

```
Namespace    What it isolates             Container sees
─────────    ─────────────────────────    ──────────────────────
PID          Process IDs                  Its own PID 1 (not host's)
NET          Network stack                Its own IP, ports, routes
MNT          Filesystem mounts            Its own root filesystem
UTS          Hostname                     Its own hostname
IPC          Inter-process communication  Its own shared memory
USER         User/group IDs              Can be "root" inside but
                                          unprivileged on the host
```

Example — the container sees PID 1 (like it's the only process):
```
# Inside the container:
$ ps aux
PID   USER    COMMAND
1     root    nginx: master process    ← "I'm PID 1!"

# On the host:
$ ps aux | grep nginx
PID     USER    COMMAND
28451   root    nginx: master process  ← Actually PID 28451
```

The container thinks nginx is PID 1 (init process). On the host, it's just process 28451. **Same process, different views.**

### 2. Resource Limits (Linux cgroups)

Cgroups **limit how much** the process can use:

```
Resource    Limit                 Why it matters
────────    ─────────────────     ──────────────────────────
CPU         --cpus=2              Can't use more than 2 CPU cores
Memory      --memory=512m         Can't use more than 512 MB RAM
                                  (OOM-killed if it tries)
Disk I/O    --blkio-weight        Limits read/write speed
PIDs        --pids-limit=100      Can't fork-bomb the host
```

Without cgroups, a container could use ALL the host's CPU and RAM — crashing other containers and the host itself.

---

## Containers vs VMs — The Complete Comparison

```
Feature          Container                    Virtual Machine
───────────      ────────────────────         ─────────────────────
What it is       Process with isolation       Complete OS instance
Start time       Milliseconds                 30-60 seconds
Size             5-500 MB                     2-10 GB
Overhead         Near zero                    Hypervisor + guest OS
Isolation        Process-level (namespaces)   Hardware-level (hypervisor)
Kernel           Shared with host             Own kernel
Density          100s per host                10s per host
Security         Good (but shared kernel)     Stronger (separate kernel)
Use case         Microservices, CI/CD         Legacy apps, different OS
```

### When to Use Containers

```
✅ Microservices (each service in its own container)
✅ CI/CD pipelines (consistent build environments)
✅ Development (match production environment locally)
✅ Scaling (spin up 50 containers in seconds)
✅ Cloud-native applications
```

### When to Use VMs

```
✅ Running different OS kernels (Windows on Linux host)
✅ Stronger security isolation (multi-tenant, untrusted workloads)
✅ Legacy applications that need a full OS
✅ Compliance requirements that mandate VM-level isolation
```

### The Real World: Both Together

In production, you typically use **containers running inside VMs**:

```
┌──────────────────────── AWS EC2 Instance (VM) ─────────────────────┐
│                                                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │
│  │ App     │  │ App     │  │ Redis   │  │ Nginx   │              │
│  │ (Python)│  │ (Node)  │  │         │  │         │              │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘              │
│       └─────────┬──┴───────────┬┘            │                    │
│            Docker Engine / Kubernetes                              │
│            Linux Kernel                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

  VM provides: hardware isolation from other customers (security)
  Containers provide: lightweight isolation between your apps (efficiency)
```

Kubernetes does exactly this: your Pods (containers) run on Nodes (VMs).

---

## The OCI Standard

Docker created containers, but now containers are an **open standard** managed by the **Open Container Initiative (OCI)**:

```
OCI defines three specs:

1. Image Spec       What a container image looks like
                    (layers, config, manifest)

2. Runtime Spec     How to run a container
                    (namespaces, cgroups, filesystem)

3. Distribution     How to push/pull images
   Spec             (registry API)
```

Why this matters:

```
Before OCI:   Only Docker could run Docker containers
After OCI:    Any compliant runtime can run any OCI image

Container runtimes (all OCI-compatible):
  docker       ← what you'll learn first
  containerd   ← what Kubernetes uses (Docker's runtime, extracted)
  cri-o        ← lightweight runtime built specifically for Kubernetes
  podman       ← daemonless Docker alternative (Red Hat)
```

When Kubernetes "dropped Docker support" in v1.24, it actually just stopped using Docker's extra layer and talks directly to **containerd** — the same runtime Docker uses internally. Your images still work everywhere.

---

## Container Lifecycle — The Big Picture

```
         Image (blueprint)
              │
              │ docker run
              ▼
   ┌─── Created ────┐
   │  (not running)  │
   │                 │
   │  docker start   │
   ▼                 │
Running ◄────────────┘
   │
   ├── docker pause  → Paused → docker unpause → Running
   │
   ├── docker stop   → Stopped (exit code 0)
   │
   └── process dies  → Stopped (exit code 1, 137, etc.)
                            │
                            ├── docker start  → Running (restart)
                            │
                            └── docker rm     → Removed (gone forever)
```

```
Key insight:
  Image     = Class       (blueprint, read-only)
  Container = Instance    (running thing, has state)

  You can create 100 containers from 1 image.
  Each container is independent with its own writable layer.
```

---

## What Docker Actually Does

Docker is a **platform** that makes containers easy to use. Without Docker, you'd have to manually configure namespaces, cgroups, and filesystem overlays.

```
Without Docker (the hard way):
  1. Download a root filesystem
  2. Create namespaces: unshare --pid --net --mount ...
  3. Set up cgroups: mkdir /sys/fs/cgroup/..., echo limits
  4. Set up network: ip netns, veth pairs, bridge
  5. Mount the filesystem: mount --bind, overlay
  6. Execute the process: chroot, exec

With Docker (the easy way):
  docker run nginx
```

Docker handles all the kernel plumbing so you don't have to. But understanding what's underneath (namespaces, cgroups, layers) is crucial — that's what file 07 and file 09 will cover.

---

## Key Terms

```
Term              Definition
────              ──────────────────────────────────────────────────
Container         A process running in an isolated environment with
                  resource limits
Image             A read-only template used to create containers
                  (your app + dependencies, packaged)
Docker            The platform/tooling that builds, runs, and manages
                  containers
Container         The software that actually creates and runs containers
Runtime           (containerd, runc, cri-o)
OCI               Open Container Initiative — the standard that ensures
                  images and runtimes are compatible
Namespace         Linux kernel feature that isolates what a process can SEE
Cgroup            Linux kernel feature that limits what a process can USE
Host              The physical or virtual machine running the containers
```

---

## How This Connects to Kubernetes

```
Docker concept          → Kubernetes equivalent
──────────────────      → ──────────────────────────────
Container               → Container (inside a Pod)
docker run              → kubectl run / Pod spec
Image                   → Same OCI images
Container runtime       → containerd / CRI-O (via CRI)
--cpus, --memory        → resources.requests / resources.limits
Namespaces (Linux)      → Same — Pods use the same kernel features
Host machine            → Node (worker node in the cluster)
```

Kubernetes doesn't replace Docker — it **orchestrates** containers that Docker (or containerd) creates.

---

> **Next:** 02 — Docker Architecture → How Docker's components work together, from `docker run` to a running container.
