# 02 — Docker Architecture

## The Big Picture

When you type `docker run nginx`, five components work together:

```
  You type: docker run nginx
       │
       ▼
┌──────────────┐    REST API     ┌──────────────┐
│ Docker CLI   │ ──────────────→ │ Docker       │
│ (docker)     │  /var/run/      │ Daemon       │
│              │  docker.sock    │ (dockerd)    │
└──────────────┘                 └──────┬───────┘
                                        │
                                        │ gRPC
                                        ▼
                                 ┌──────────────┐
                                 │  containerd  │
                                 │              │
                                 └──────┬───────┘
                                        │
                                        │ OCI spec
                                        ▼
                                 ┌──────────────┐
                                 │    runc      │  ← creates the container
                                 │              │     (namespaces + cgroups)
                                 └──────┬───────┘
                                        │
                                        ▼
                                 ┌──────────────┐
                                 │  Container   │  ← your nginx process
                                 │  (running)   │
                                 └──────────────┘
```

Let's understand each component.

---

## Component 1: Docker CLI (`docker`)

The command-line tool you type commands into.

```
docker run nginx        ← CLI
docker build .          ← CLI
docker ps               ← CLI
docker logs my-app      ← CLI
```

The CLI does **nothing** by itself. It just:
1. Parses your command
2. Converts it to an API call
3. Sends it to the Docker daemon

```
docker run nginx
       │
       │ becomes:
       ▼
POST /v1.43/containers/create
{
  "Image": "nginx",
  "Cmd": null
}

POST /v1.43/containers/{id}/start
```

The CLI and daemon can be on **different machines**:
```
# Local (default) — CLI talks to daemon via Unix socket:
  /var/run/docker.sock

# Remote — CLI talks to daemon via TCP:
  DOCKER_HOST=tcp://remote-server:2376

This is how Docker Desktop works on Mac/Windows:
  CLI runs on your Mac → talks to daemon running in a Linux VM
```

---

## Component 2: Docker Daemon (`dockerd`)

The background service that manages everything Docker.

```
What dockerd does:
  ✓ Listens for API requests from the CLI
  ✓ Manages images (pull, build, store)
  ✓ Manages containers (create, start, stop, delete)
  ✓ Manages networks (create bridge, assign IPs)
  ✓ Manages volumes (create, mount)
  ✓ Handles image builds (Dockerfile → image)

What dockerd does NOT do:
  ✗ Actually create containers (that's containerd + runc)
  ✗ Run containers (containers are NOT child processes of dockerd)
```

```
Check if the daemon is running:
  $ systemctl status docker
  ● docker.service - Docker Application Container Engine
     Active: active (running)

The daemon runs as root (by default):
  $ ps aux | grep dockerd
  root  1234  dockerd --group docker

  ⚠ This is a security consideration — covered in file 09.
```

### The Docker Socket

```
/var/run/docker.sock

This Unix socket is HOW the CLI talks to the daemon.

Whoever has access to this socket has FULL control over Docker:
  - Start/stop any container
  - Mount any host directory
  - Effectively has ROOT access to the host

This is why:
  $ ls -la /var/run/docker.sock
  srw-rw---- 1 root docker /var/run/docker.sock

  Only root and members of the "docker" group can use it.
  Adding a user to the "docker" group = giving them root. (File 09)
```

---

## Component 3: containerd

The **container runtime** that actually manages the lifecycle of containers.

```
containerd is:
  - Originally part of Docker (extracted in 2016)
  - Now a standalone CNCF project
  - What Kubernetes uses directly (since K8s 1.24)
  - The layer between Docker and the low-level runtime (runc)
```

```
What containerd does:
  ✓ Pulls and stores images
  ✓ Creates container from image (sets up filesystem)
  ✓ Calls runc to actually start the container
  ✓ Manages container lifecycle (running, paused, stopped)
  ✓ Manages snapshots (filesystem layers)
  ✓ Manages networking (via CNI plugins)
```

```
Docker vs Kubernetes — both use containerd:

  Docker path:
    docker CLI → dockerd → containerd → runc → container

  Kubernetes path:
    kubelet → CRI → containerd → runc → container

  Notice: Kubernetes skips dockerd entirely.
  That's why K8s "dropped Docker" — it just talks to containerd directly.
  The containers are identical either way.
```

You can even use containerd without Docker:
```
# containerd has its own CLI: ctr
$ ctr images pull docker.io/library/nginx:latest
$ ctr run docker.io/library/nginx:latest my-nginx

# But nobody uses ctr in practice — it's low-level.
# Docker and Kubernetes are the user-friendly interfaces.
```

---

## Component 4: runc

The **lowest-level component** that creates containers.

```
runc is:
  - The reference implementation of the OCI Runtime Spec
  - Written in Go
  - A single binary (~10 MB)
  - Created by Docker, donated to OCI
```

```
What runc does (and ONLY this):
  1. Takes a filesystem bundle (root filesystem + config)
  2. Creates Linux namespaces (PID, NET, MNT, etc.)
  3. Sets up cgroups (CPU, memory limits)
  4. Configures security (capabilities, seccomp, AppArmor)
  5. Calls execve() to start the container process
  6. EXITS — runc is gone, the container process runs on its own

  ┌─────────────────────────────────────────────────┐
  │ runc does NOT stay running.                     │
  │ The container process is NOT a child of runc.   │
  │ This is why restarting Docker doesn't kill your │
  │ containers — they're independent processes.     │
  └─────────────────────────────────────────────────┘
```

---

## Component 5: The Container Shim (`containerd-shim`)

There's actually one more piece — the **shim** that sits between containerd and the container:

```
containerd → containerd-shim → container process

The shim:
  ✓ Keeps STDIN/STDOUT open for the container
  ✓ Reports exit status back to containerd
  ✓ Allows containerd to restart without killing containers
  ✓ One shim per running container
```

```
$ ps aux | grep shim
root  5678  containerd-shim-runc-v2 -namespace moby -id abc123...
root  5701  containerd-shim-runc-v2 -namespace moby -id def456...

Each running container has one shim process.
```

---

## The Complete `docker run` Flow

Let's trace exactly what happens when you type `docker run -d -p 80:80 nginx`:

```
Step 1: CLI parses the command
  docker CLI reads: run -d -p 80:80 nginx
  Translates to API calls to dockerd

Step 2: dockerd receives the request
  Checks if image "nginx:latest" exists locally
  If not → tells containerd to pull it

Step 3: containerd pulls the image
  Contacts Docker Hub registry (registry-1.docker.io)
  Downloads image manifest
  Downloads layers (only ones not already cached)
  Stores layers in content-addressable storage

Step 4: dockerd creates the container
  Creates a container record (ID, config, state)
  Sets up networking:
    - Creates a veth pair (file 07)
    - Connects one end to docker0 bridge
    - Assigns an IP from the bridge subnet
    - Sets up iptables rules for port mapping (-p 80:80)
  Sets up storage:
    - Creates a read-write layer on top of image layers
    - Sets up the overlay filesystem

Step 5: containerd starts the container
  Tells runc to create and start the container process

Step 6: runc creates the process
  Creates namespaces (PID, NET, MNT, UTS, IPC)
  Moves the process into the prepared network namespace
  Sets up cgroups
  Pivots root to the container filesystem
  Executes nginx (becomes PID 1 inside the container)
  runc exits

Step 7: containerd-shim takes over
  The shim monitors the nginx process
  Keeps stdout/stderr connected
  Reports back to containerd

Step 8: dockerd reports success
  Returns the container ID to the CLI
  CLI prints: a1b2c3d4e5f6...
```

```
Timeline:
  0ms    CLI sends request
  5ms    dockerd processes it
  50ms   Image already cached (skip pull)
  100ms  Network and storage set up
  150ms  runc creates the container
  200ms  nginx is running and accepting connections

  Total: ~200ms from command to running container.
  Compare this to a VM: 30-60 seconds.
```

---

## Docker Architecture Diagram — Everything Together

```
┌─── User Space ──────────────────────────────────────────────────────┐
│                                                                      │
│   docker CLI                                                         │
│       │                                                              │
│       │ REST API (/var/run/docker.sock)                              │
│       ▼                                                              │
│   dockerd (Docker Daemon)                                            │
│       │                                                              │
│       ├── Image management (build, pull, push)                       │
│       ├── Network management (bridge, overlay, host)                 │
│       ├── Volume management (local, NFS, etc.)                       │
│       │                                                              │
│       │ gRPC                                                         │
│       ▼                                                              │
│   containerd                                                         │
│       │                                                              │
│       ├── Image storage (content-addressable)                        │
│       ├── Snapshot management (overlay layers)                       │
│       │                                                              │
│       │ OCI Runtime Spec                                             │
│       ▼                                                              │
│   runc (creates container, then exits)                               │
│       │                                                              │
│       ▼                                                              │
│   containerd-shim ─── Container Process (nginx, python, node, etc.) │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
┌─── Kernel Space ────────────────────────────────────────────────────┐
│                                                                      │
│   Namespaces (PID, NET, MNT, UTS, IPC, USER)                       │
│   Cgroups (CPU, Memory, I/O, PIDs)                                  │
│   Overlay filesystem (UnionFS)                                       │
│   Netfilter / iptables (port mapping, NAT)                          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Why This Layered Architecture?

```
"Why not just have Docker do everything in one binary?"

The layers exist for good reasons:

1. RESTART DOCKER WITHOUT KILLING CONTAINERS
   Because containers aren't children of dockerd, you can:
     systemctl restart docker
   And all running containers keep running.

2. KUBERNETES DOESN't NEED ALL OF DOCKER
   K8s only needs containerd (the runtime).
   It doesn't need the CLI, the build system, or the daemon.
   
   Docker path:    CLI → dockerd → containerd → runc
   K8s path:       kubelet → containerd → runc  (simpler!)

3. SWAP COMPONENTS
   Don't like runc? Use crun (written in C, faster).
   Don't like containerd? Use CRI-O (built for K8s).
   The OCI standard ensures compatibility.

4. SECURITY BOUNDARIES
   containerd runs as root but has a smaller attack surface than dockerd.
   runc runs briefly and exits — minimal exposure.
```

---

## Checking Your Docker Setup

```bash
# Docker version (client + server)
$ docker version
Client: Docker Engine - Community
  Version: 24.0.7
Server: Docker Engine - Community
  Version: 24.0.7
  OS/Arch: linux/amd64

# System info
$ docker info
  Containers: 5
  Running: 2
  Paused: 0
  Stopped: 3
  Images: 12
  Server Version: 24.0.7
  Storage Driver: overlay2
  Cgroup Driver: systemd
  Docker Root Dir: /var/lib/docker

# Check containerd
$ systemctl status containerd
  Active: active (running)

# See the components
$ which docker dockerd containerd runc
  /usr/bin/docker
  /usr/bin/dockerd
  /usr/bin/containerd
  /usr/bin/runc
```

---

## Key Takeaways

```
1. docker CLI    → just a client, sends API requests
2. dockerd       → manages images, networks, volumes, orchestrates everything
3. containerd    → manages container lifecycle (what K8s uses directly)
4. runc          → creates the container (namespaces + cgroups), then exits
5. shim          → keeps container connected after runc exits

The container is NOT a child process of Docker.
It's an independent process managed by the shim.
Restarting Docker doesn't kill your containers.

Kubernetes uses the same containerd + runc stack.
Your images work everywhere because of OCI standards.
```

---

## How This Connects to Kubernetes

```
Docker Architecture          →  Kubernetes Architecture
──────────────────           →  ──────────────────────────
docker CLI                   →  kubectl
dockerd                      →  kubelet (per node)
containerd                   →  containerd (same!)
runc                         →  runc (same!)
/var/run/docker.sock         →  CRI (Container Runtime Interface)
docker pull                  →  kubelet pulls via containerd
docker network               →  CNI (Container Network Interface)
docker volume                →  CSI (Container Storage Interface)
```

---

> **Next:** 03 — Images & Layers → How container images are built, stored, and shared using the layer system.
