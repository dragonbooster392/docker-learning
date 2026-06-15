# 05 — Container Lifecycle

## Container States

```
         docker create
              │
              ▼
         ┌─────────┐
         │ Created  │ ← exists but not running
         └────┬─────┘
              │ docker start
              ▼
         ┌─────────┐        docker pause       ┌─────────┐
         │ Running  │ ────────────────────────→ │ Paused  │
         │         │ ←──────────────────────── │         │
         └────┬─────┘      docker unpause       └─────────┘
              │
              ├── docker stop (graceful) ─────→ Stopped (exit 0)
              │
              └── process crashes ────────────→ Stopped (exit 1, 137, etc.)
                                                    │
                                                    ├── docker start → Running
                                                    ├── docker rm    → Removed (gone)
                                                    └── docker commit → new Image
```

---

## Core Commands

### `docker run` — Create + Start in One Command

```bash
docker run nginx
```

`docker run` is the command you'll use most. It's actually `docker create` + `docker start` combined.

```bash
# Basic run (foreground, ctrl+c to stop)
docker run nginx

# Detached mode (background)
docker run -d nginx

# With a name
docker run -d --name my-nginx nginx

# With port mapping
docker run -d -p 8080:80 nginx

# With environment variables
docker run -d -e MYSQL_ROOT_PASSWORD=secret mysql:8

# With a volume mount
docker run -d -v mydata:/var/lib/mysql mysql:8

# Interactive shell
docker run -it ubuntu bash

# Auto-remove when stopped
docker run --rm -it alpine sh

# All together
docker run -d \
  --name my-app \
  -p 8080:8000 \
  -e DB_HOST=db \
  -v ./data:/app/data \
  --restart unless-stopped \
  myapp:v1
```

### `docker run` Flags Explained

```
Flag                      What it does
────                      ────────────────────────────────────────────
-d, --detach              Run in background (returns container ID)
-it                       -i (interactive) + -t (terminal) — for shells
--name NAME               Give the container a human-readable name
-p HOST:CONTAINER         Map host port to container port
-e KEY=VALUE              Set environment variable
-v HOST:CONTAINER         Mount a volume or directory
--rm                      Auto-delete container when it exits
--restart POLICY          Restart behavior (see below)
-w /path                  Set working directory
--network NAME            Connect to a specific network
--cpus N                  Limit CPU (e.g., --cpus=2)
--memory SIZE             Limit memory (e.g., --memory=512m)
--user USER               Run as specific user
--read-only               Read-only root filesystem
--entrypoint CMD          Override ENTRYPOINT
```

### `-it` Explained

```
-i = --interactive    Keep STDIN open (so you can type)
-t = --tty            Allocate a pseudo-terminal (so you see a prompt)

Together (-it):
  docker run -it ubuntu bash
  root@abc123:/# _       ← you're inside the container with a shell

Without -it:
  docker run ubuntu bash
  (nothing happens — bash has no input and no terminal, so it exits)

Without -t (just -i):
  docker run -i ubuntu bash
  ls                      ← you can type commands
  bin  boot  dev  etc     ← output works, but no prompt shown

-it is only for interactive use. For background services, use -d.
```

---

## Managing Running Containers

### `docker ps` — List Containers

```bash
# Running containers
$ docker ps
CONTAINER ID   IMAGE   COMMAND                  STATUS          PORTS                  NAMES
a1b2c3d4e5f6   nginx   "/docker-entrypoint…"   Up 5 minutes    0.0.0.0:8080->80/tcp   my-nginx

# ALL containers (including stopped)
$ docker ps -a
CONTAINER ID   IMAGE    STATUS                     NAMES
a1b2c3d4e5f6   nginx    Up 5 minutes               my-nginx
f6e5d4c3b2a1   redis    Exited (0) 10 minutes ago  my-redis

# Just IDs (useful for scripting)
$ docker ps -q
a1b2c3d4e5f6

# Formatted output
$ docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
NAMES       IMAGE    STATUS
my-nginx    nginx    Up 5 minutes
```

### `docker stop` — Graceful Shutdown

```bash
docker stop my-nginx
```

```
What happens:
  1. Docker sends SIGTERM to PID 1 in the container
  2. The process has 10 seconds to shut down gracefully
     (close connections, save state, etc.)
  3. After 10 seconds, Docker sends SIGKILL (force kill)

Change the timeout:
  docker stop -t 30 my-nginx    ← give it 30 seconds

This matters for:
  - Web servers closing active connections
  - Databases flushing writes to disk
  - Queue workers finishing current jobs
```

### `docker kill` — Immediate Termination

```bash
docker kill my-nginx
```

```
Sends SIGKILL immediately — no grace period.
The process is terminated instantly.

Use when:
  - Container is stuck/frozen
  - You don't care about graceful shutdown
  - docker stop timed out
```

### `docker start` / `docker restart`

```bash
docker start my-nginx       # Start a stopped container
docker restart my-nginx     # Stop + start (sends SIGTERM, then starts)
```

### `docker rm` — Remove a Container

```bash
# Remove a stopped container
docker rm my-nginx

# Force remove (even if running)
docker rm -f my-nginx

# Remove all stopped containers
docker container prune

# Remove all containers (running + stopped)
docker rm -f $(docker ps -aq)
```

```
⚠ Removing a container DELETES:
  - The container's writable layer (all files written inside the container)
  - The container's logs
  - The container's metadata

  It does NOT delete:
  - The image (still available for new containers)
  - Named volumes (data persists — file 08)
```

---

## Getting Inside a Running Container

### `docker exec` — Run a Command in Running Container

```bash
# Open a shell
docker exec -it my-nginx bash

# Run a single command
docker exec my-nginx cat /etc/nginx/nginx.conf

# Run as a specific user
docker exec -u root my-nginx whoami

# Set environment variables for the exec session
docker exec -e DEBUG=true my-nginx env
```

```
docker exec vs docker run:
  docker exec  = run command in an EXISTING running container
  docker run   = create a NEW container and run command in it

  docker exec -it my-nginx bash    ← shell in running container
  docker run -it nginx bash        ← NEW container with shell (different!)
```

### `docker attach` — Attach to Container's STDIO

```bash
docker attach my-nginx
```

```
docker attach connects to the container's PID 1 stdout/stderr.
You see what the main process outputs.

Ctrl+C = sends SIGINT to PID 1 (may STOP the container!)
Ctrl+P, Ctrl+Q = detach without stopping

docker attach vs docker exec:
  attach → connects to PID 1 (the main process)
  exec   → starts a NEW process in the container

  Usually exec is what you want.
```

---

## Logs

### `docker logs` — View Container Output

```bash
# All logs
docker logs my-nginx

# Follow (like tail -f)
docker logs -f my-nginx

# Last 100 lines
docker logs --tail 100 my-nginx

# Logs with timestamps
docker logs -t my-nginx

# Logs since a specific time
docker logs --since 1h my-nginx
docker logs --since "2024-01-15T10:00:00" my-nginx
```

```
Where do logs come from?
  Docker captures everything PID 1 writes to stdout and stderr.
  
  If your app writes to a LOG FILE (not stdout), docker logs shows NOTHING.
  
  This is why 12-factor apps log to stdout:
    Bad:  logging.FileHandler("/app/logs/app.log")
    Good: logging.StreamHandler(sys.stdout)
  
  nginx does this by default:
    access_log → symlinked to /dev/stdout
    error_log  → symlinked to /dev/stderr
```

---

## Environment Variables

The primary way to configure containers (no config files to manage):

```bash
# Pass individual variables
docker run -d \
  -e DATABASE_URL=postgres://db:5432/mydb \
  -e REDIS_URL=redis://cache:6379 \
  -e LOG_LEVEL=info \
  myapp:v1

# Pass from host environment
export API_KEY=abc123
docker run -d -e API_KEY myapp:v1    ← takes value from host

# Pass from a file
docker run -d --env-file .env myapp:v1
```

```
# .env file format:
DATABASE_URL=postgres://db:5432/mydb
REDIS_URL=redis://cache:6379
LOG_LEVEL=info
# Comments are supported
API_KEY=abc123
```

```
Priority order (highest to lowest):
  1. -e flag on docker run
  2. --env-file
  3. ENV in Dockerfile

If all three set the same variable, -e wins.
```

```
Inspect a container's environment:
  docker exec my-app env
  docker inspect my-app | jq '.[0].Config.Env'

⚠ Environment variables are visible in:
  - docker inspect output
  - /proc/1/environ inside the container
  - ps eww on the host

  For real secrets, use Docker secrets (file 09) or a secret manager.
```

---

## Resource Limits

Without limits, a container can consume ALL host resources:

### CPU Limits

```bash
# Limit to 2 CPU cores
docker run -d --cpus=2 myapp

# Limit to 50% of one core
docker run -d --cpus=0.5 myapp

# CPU shares (relative weight — default 1024)
docker run -d --cpu-shares=512 myapp     ← half the default weight
docker run -d --cpu-shares=2048 myapp    ← double the default weight

# Pin to specific CPU cores
docker run -d --cpuset-cpus="0,1" myapp  ← only use cores 0 and 1
```

```
--cpus vs --cpu-shares:

  --cpus=2           Hard limit. Container CANNOT use more than 2 cores.
                     Even if the host has idle CPUs.

  --cpu-shares=512   Soft limit. Only matters when CPUs are CONTESTED.
                     If host is idle, container can use ALL CPUs.
                     If host is busy, containers get CPU proportional to shares.

In Kubernetes:
  --cpus      → resources.limits.cpu
  --cpu-shares → resources.requests.cpu
```

### Memory Limits

```bash
# Limit to 512 MB
docker run -d --memory=512m myapp

# Limit to 1 GB
docker run -d --memory=1g myapp

# Memory + swap limit
docker run -d --memory=512m --memory-swap=1g myapp
# Container gets 512 MB RAM + 512 MB swap = 1 GB total

# No swap
docker run -d --memory=512m --memory-swap=512m myapp
```

```
What happens when a container exceeds its memory limit?

  The kernel OOM-killer terminates the process.
  
  $ docker inspect my-app | jq '.[0].State'
  {
    "Status": "exited",
    "ExitCode": 137,        ← 128 + 9 = SIGKILL (OOM killed)
    "OOMKilled": true       ← confirms it was out-of-memory
  }

  Exit code 137 = the container was killed by SIGKILL (signal 9).
  This is the #1 reason containers mysteriously die.

In Kubernetes:
  --memory → resources.limits.memory
  OOM → pod gets OOMKilled status, may be restarted by kubelet
```

---

## Restart Policies

What happens when a container stops?

```bash
docker run -d --restart=always nginx
```

```
Policy               When it restarts
──────               ────────────────────────────────────────────
no                   Never restart (default)
on-failure           Only if exit code ≠ 0 (crashed)
on-failure:5         Same, but max 5 retries
always               Always restart, regardless of exit code
                     Also starts on Docker daemon boot
unless-stopped       Like "always", but if you manually stop it,
                     it stays stopped (even after daemon restart)
```

```
Common patterns:

  Production services:
    docker run -d --restart=unless-stopped my-web-app

  One-shot tasks (migrations, backups):
    docker run --rm my-migration-script    ← run once, auto-remove

  Development:
    docker run -d --restart=no my-dev-app  ← don't auto-restart
```

```
In Kubernetes:
  --restart=always      → restartPolicy: Always (default for Pods)
  --restart=on-failure  → restartPolicy: OnFailure (for Jobs)
  --restart=no          → restartPolicy: Never (for one-shot Jobs)
```

---

## Health Checks at Runtime

```bash
docker run -d \
  --health-cmd="curl -f http://localhost:8000/health || exit 1" \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  --health-start-period=10s \
  myapp:v1
```

```
--health-start-period=10s
  Wait 10 seconds before first check (app startup time).

$ docker ps
STATUS: Up 2 minutes (healthy)

$ docker inspect my-app | jq '.[0].State.Health'
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2024-01-15T10:00:30Z",
      "End": "2024-01-15T10:00:30Z",
      "ExitCode": 0,
      "Output": "OK"
    }
  ]
}
```

---

## Inspecting Containers

### `docker inspect` — Everything About a Container

```bash
# Full JSON output
docker inspect my-nginx

# Specific fields with Go templates
docker inspect --format '{{.State.Status}}' my-nginx
docker inspect --format '{{.NetworkSettings.IPAddress}}' my-nginx
docker inspect --format '{{.HostConfig.Memory}}' my-nginx

# With jq (more readable)
docker inspect my-nginx | jq '.[0].State'
docker inspect my-nginx | jq '.[0].Config.Env'
docker inspect my-nginx | jq '.[0].NetworkSettings.Networks'
```

### `docker stats` — Live Resource Usage

```bash
$ docker stats
CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O        BLOCK I/O
my-nginx    0.01%   5.3 MiB / 512 MiB   1.04%   1.2kB / 648B   0B / 0B
my-redis    0.10%   8.1 MiB / 1 GiB     0.79%   2.3kB / 1.1kB  4.1kB / 0B
my-app      2.34%   128 MiB / 256 MiB   50.0%   50kB / 25kB    0B / 8.2kB

This is real-time monitoring — like top for containers.
Press Ctrl+C to exit.

# One-shot stats (no streaming)
docker stats --no-stream
```

### `docker top` — Processes Inside a Container

```bash
$ docker top my-nginx
PID     USER    TIME    COMMAND
28451   root    0:00    nginx: master process nginx -g daemon off;
28502   nginx   0:00    nginx: worker process
28503   nginx   0:00    nginx: worker process
```

### `docker diff` — What Changed in the Container

```bash
$ docker diff my-nginx
C /var               ← Changed
C /var/cache         ← Changed
A /var/cache/nginx   ← Added
C /run               ← Changed
A /run/nginx.pid     ← Added
```

```
A = Added (new file)
C = Changed (modified directory/file)
D = Deleted

This shows what the container's R/W layer contains —
everything that differs from the original image.
```

---

## Exit Codes

```
Exit Code    Meaning                          Common Cause
─────────    ────────────────────────────     ─────────────────────────
0            Success (clean exit)             docker stop, process finished
1            General error                    Application crash, exception
2            Shell misuse                     Bad command syntax
126          Command not executable           Permission denied
127          Command not found                Typo in CMD/ENTRYPOINT
128 + N      Killed by signal N               SIGKILL=9, SIGTERM=15
  137        SIGKILL (128+9)                  OOM killed, docker kill
  143        SIGTERM (128+15)                 docker stop (graceful)
```

```bash
# Check exit code
$ docker inspect my-app | jq '.[0].State.ExitCode'
137

# Check if OOM killed
$ docker inspect my-app | jq '.[0].State.OOMKilled'
true
```

---

## Debugging Crashed Containers

When a container exits unexpectedly:

```bash
# Step 1: Check what happened
docker ps -a                           # See exit code in STATUS column
docker logs my-app                     # Check application logs
docker inspect my-app | jq '.[0].State'  # Full state details

# Step 2: If OOM (exit 137 + OOMKilled=true)
docker stats --no-stream               # Check current resource usage
docker inspect my-app | jq '.[0].HostConfig.Memory'  # Check memory limit
# Fix: increase --memory or fix memory leak in your app

# Step 3: If exit 1 (application error)
docker logs --tail 50 my-app           # Last 50 lines of logs
docker run -it myapp:v1 bash           # Start fresh container with shell
# Debug inside: check files, run the command manually, etc.

# Step 4: If exit 127 (command not found)
docker inspect myapp:v1 | jq '.[0].Config.Cmd'       # Check CMD
docker inspect myapp:v1 | jq '.[0].Config.Entrypoint' # Check ENTRYPOINT
docker run -it myapp:v1 ls /usr/local/bin/             # Check if binary exists

# Step 5: Look at what changed in the container
docker diff my-app                     # Files added/changed/deleted
```

```
Pro tip: debug a crashed container without restarting it:

  # Export the container's filesystem
  docker export my-crashed-app > debug.tar
  
  # Or copy a specific file out
  docker cp my-crashed-app:/app/logs/error.log ./error.log
  
  docker cp works even on stopped containers!
```

---

## Cleaning Up

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove unused volumes (⚠ data loss!)
docker volume prune

# Remove EVERYTHING unused (containers, images, volumes, networks)
docker system prune -a --volumes

# Nuclear option: remove ALL containers and images
docker rm -f $(docker ps -aq)
docker rmi -f $(docker images -q)
```

---

## How This Connects to Kubernetes

```
Docker command               → Kubernetes equivalent
──────────────────           → ──────────────────────────────
docker run -d                → kubectl create deployment
docker run -it ... bash      → kubectl exec -it POD -- bash
docker stop                  → kubectl delete pod (K8s recreates it)
docker logs                  → kubectl logs
docker ps                    → kubectl get pods
docker stats                 → kubectl top pods
docker inspect               → kubectl describe pod
-e KEY=VALUE                 → env: in Pod spec, or ConfigMap/Secret
--cpus, --memory             → resources.requests / resources.limits
--restart=always             → restartPolicy: Always (default)
HEALTHCHECK                  → livenessProbe / readinessProbe
exit 137 (OOM)               → OOMKilled in pod status
docker exec                  → kubectl exec
docker cp                    → kubectl cp
```

---

> **Next:** 06 — Docker Networking → bridge, host, none, port mapping, container DNS, network isolation.
