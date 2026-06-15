# 12 — Best Practices & Troubleshooting

## Image Size Optimization

Smaller images = faster pulls, faster deploys, fewer vulnerabilities.

### Choose the Right Base Image

```
Image               Size     When to use
─────               ────     ────────────────────────────────
scratch             0 MB     Static Go/Rust binaries
alpine              7 MB     Small footprint, simple apps
debian:slim         74 MB    Python/Node with native deps
ubuntu              78 MB    Familiar, good tooling
python:3.11-slim    130 MB   Python apps (recommended)
python:3.11         920 MB   Includes gcc, dev headers (build only!)
node:20-slim        200 MB   Node.js apps (recommended)
node:20             1.1 GB   Includes build tools (build only!)

Rule: always use -slim or -alpine variants for runtime.
Use full images only in build stages (multi-stage builds).
```

### Multi-Stage Builds (Recap)

```dockerfile
# Stage 1: Build (big image, throw away)
FROM python:3.11 AS builder
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Runtime (small image, keep)
FROM python:3.11-slim
COPY --from=builder /install /usr/local
COPY . /app/
CMD ["python", "/app/app.py"]

# python:3.11 = 920 MB (build) → python:3.11-slim = 130 MB (runtime)
# Build tools are NOT in the final image.
```

### Reduce Layer Size

```dockerfile
# BAD — apt cache stays in the layer (50+ MB wasted)
RUN apt-get update
RUN apt-get install -y curl wget
RUN rm -rf /var/lib/apt/lists/*

# GOOD — one layer, cache cleaned before layer is saved
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl wget && \
    rm -rf /var/lib/apt/lists/*

# --no-install-recommends: skip optional packages (saves 20-50 MB)
```

```dockerfile
# BAD — pip cache stays in the layer
RUN pip install flask gunicorn requests

# GOOD — no pip cache saved
RUN pip install --no-cache-dir flask gunicorn requests
```

### Remove Unnecessary Files

```dockerfile
# Remove docs, man pages, locale files
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/* \
           /usr/share/doc \
           /usr/share/man \
           /usr/share/locale
```

### Use .dockerignore

```
# .dockerignore
.git
.gitignore
node_modules
__pycache__
*.pyc
.env
.venv
Dockerfile
docker-compose*.yml
README.md
tests/
docs/
*.md
.DS_Store
```

```
Without .dockerignore, COPY . . sends everything:
  .git/          → 100 MB+
  node_modules/  → 500 MB+
  .env           → secrets!

The build context is sent to the Docker daemon before build starts.
.dockerignore reduces the time to send the context AND keeps
junk out of your image.
```

---

## Dockerfile Best Practices Checklist

```
Image:
  □ Use specific base image version (python:3.11-slim, not python:latest)
  □ Use -slim or -alpine variants for runtime
  □ Multi-stage builds for compiled languages
  □ .dockerignore file

Layers:
  □ Combine related RUN commands (fewer layers)
  □ Clean up in the same layer (apt cache, pip cache)
  □ Order by change frequency (least → most changed)
  □ COPY requirements/package.json BEFORE source code

Security:
  □ Run as non-root USER
  □ No secrets in ENV or COPY
  □ Pin package versions
  □ Scan the final image

Metadata:
  □ LABEL for maintainer, version
  □ EXPOSE for documentation
  □ HEALTHCHECK for container health
```

---

## Logging

### The 12-Factor App Rule: Log to stdout

```
Docker captures stdout/stderr from PID 1.
This is what docker logs shows.

If your app writes to a FILE, docker logs shows NOTHING.

BAD:
  logging.FileHandler("/app/logs/app.log")

GOOD:
  logging.StreamHandler(sys.stdout)

Most frameworks support this:
  Flask: app.logger outputs to stderr by default ✓
  Django: configure LOGGING to use StreamHandler
  Node.js: console.log writes to stdout ✓
  nginx: access_log → /dev/stdout, error_log → /dev/stderr ✓
```

### Log Drivers

```bash
# See current log driver
docker info | grep "Logging Driver"
Logging Driver: json-file

# Change log driver per container
docker run --log-driver=json-file --log-opt max-size=10m --log-opt max-file=3 myapp

# Available drivers:
#   json-file   Default. Logs stored as JSON files on host.
#   local       Compressed, rotated. More efficient than json-file.
#   syslog      Send to syslog daemon.
#   journald    Send to systemd journal.
#   fluentd     Send to Fluentd (log aggregation).
#   awslogs     Send directly to CloudWatch (AWS).
#   none        No logs captured.
```

### Log Rotation (Prevent Disk Full!)

```
⚠ By default, Docker logs grow FOREVER and can fill your disk.

Fix globally in /etc/docker/daemon.json:
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

This limits each container to 3 files of 10 MB = 30 MB max.
Without this, a busy container can generate GBs of logs.
```

---

## Common Errors and Fixes

### Error: "port is already allocated"

```
$ docker run -p 8080:80 nginx
Error: Bind for 0.0.0.0:8080 failed: port is already allocated

Cause: Another container or process is using host port 8080.

Fix:
  # Find what's using the port
  ss -tlnp | grep 8080
  docker ps | grep 8080

  # Use a different host port
  docker run -p 8081:80 nginx

  # Or stop the other container
  docker stop the-other-container
```

### Error: "No space left on device"

```
$ docker build -t myapp .
Error: no space left on device

Cause: Docker storage (/var/lib/docker) is full.

Fix:
  # See what's using space
  docker system df

  # Clean up everything unused
  docker system prune -a --volumes

  # Check actual disk usage
  df -h /var/lib/docker
```

### Error: "OOMKilled" / Exit Code 137

```
$ docker inspect myapp | jq '.[0].State'
{
  "ExitCode": 137,
  "OOMKilled": true
}

Cause: Container exceeded its memory limit.

Fix:
  # Increase memory limit
  docker run --memory=1g myapp

  # Or fix the memory leak in your app
  # Monitor: docker stats myapp
```

### Error: "exec format error"

```
$ docker run myimage
exec /app/server: exec format error

Cause: The binary was built for a different architecture.
  e.g., Image built on ARM (Mac M1) but running on AMD64 (Linux server).

Fix:
  # Build for the target architecture
  docker buildx build --platform linux/amd64 -t myapp .

  # Or use multi-platform build
  docker buildx build --platform linux/amd64,linux/arm64 -t myapp .
```

### Error: "COPY failed: file not found"

```
$ docker build -t myapp .
COPY failed: file not found in build context

Cause: File doesn't exist in the build context, or .dockerignore excludes it.

Fix:
  # Check your .dockerignore
  cat .dockerignore

  # Check the file actually exists
  ls -la the-file

  # Remember: COPY can't access files ABOVE the build context
  COPY ../other-dir/file /app/   ← ✗ forbidden!
  # Solution: set build context to the parent directory
  docker build -t myapp -f subdir/Dockerfile ..
```

### Error: "permission denied"

```
$ docker run myapp
PermissionError: [Errno 13] Permission denied: '/app/data'

Cause: Container runs as non-root but files are owned by root.

Fix:
  # In Dockerfile: set ownership before switching user
  RUN chown -R appuser:appuser /app/data
  USER appuser

  # Or at runtime
  docker run --user root myapp chown -R 1000:1000 /app/data
```

### Error: Container Exits Immediately

```
$ docker run -d myapp
$ docker ps -a
STATUS: Exited (0) 1 second ago

Cause: The main process exited (no foreground process to keep container alive).

Common causes:
  1. CMD runs a background daemon that forks and exits
     Fix: run in foreground mode
     CMD ["nginx", "-g", "daemon off;"]

  2. CMD is a script that exits
     Fix: end script with exec or a long-running command

  3. The command crashes
     Check: docker logs myapp
```

### Error: "Cannot connect to the Docker daemon"

```
$ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock.
Is the docker daemon running?

Cause: dockerd is not running, or you don't have permission.

Fix:
  # Check if Docker is running
  systemctl status docker

  # Start it
  sudo systemctl start docker

  # Permission issue? Add yourself to docker group
  sudo usermod -aG docker $USER
  # Then log out and back in
```

---

## Debugging Strategies

### Strategy 1: Shell Into a Running Container

```bash
docker exec -it myapp bash
# or sh if bash isn't available
docker exec -it myapp sh

# Check processes
ps aux

# Check network
ip addr show
ss -tlnp
cat /etc/resolv.conf

# Check files
ls -la /app/
cat /app/config.yml
```

### Strategy 2: Shell Into a Crashed Container's Image

```bash
# Container won't start? Override the command:
docker run -it --entrypoint bash myapp:v1

# Now you're inside with the same filesystem as the container
# Check what's wrong: missing files, bad permissions, etc.
```

### Strategy 3: Copy Files Out

```bash
# Copy from a stopped container
docker cp myapp:/app/logs/error.log ./error.log

# Copy a config file to check it
docker cp myapp:/etc/nginx/nginx.conf ./nginx.conf
```

### Strategy 4: Check Resource Usage

```bash
# Live stats
docker stats

# One-shot
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### Strategy 5: Inspect Everything

```bash
# Full container config
docker inspect myapp

# Specific fields
docker inspect --format '{{.State.ExitCode}}' myapp
docker inspect --format '{{.Config.Env}}' myapp
docker inspect --format '{{.NetworkSettings.IPAddress}}' myapp
docker inspect --format '{{.HostConfig.Memory}}' myapp
```

---

## Production Deployment Patterns

### Health Check + Restart Policy

```bash
docker run -d \
  --name web \
  --restart unless-stopped \
  --health-cmd "curl -f http://localhost:8000/health || exit 1" \
  --health-interval 30s \
  --health-timeout 3s \
  --health-retries 3 \
  --memory 512m \
  --cpus 2 \
  --read-only \
  --tmpfs /tmp:rw,size=64m \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 1000:1000 \
  -p 8000:8000 \
  myapp:v2.1.3
```

### CI/CD Pipeline Flow

```
1. Developer pushes code to git
2. CI pipeline triggers:
   a. Run tests
   b. docker build (multi-stage)
   c. trivy image scan (fail if CRITICAL)
   d. docker tag with git SHA + version
   e. docker push to private registry
3. CD pipeline:
   a. Update K8s manifest (or Helm values) with new image tag
   b. kubectl apply / helm upgrade
   c. K8s does rolling update
   d. Health checks verify new pods are ready
   e. Old pods terminated
```

---

## Docker vs Kubernetes Decision Guide

```
Use Docker (Compose) when:
  ✓ Single server deployment
  ✓ Development environments
  ✓ CI/CD test environments
  ✓ Simple applications (1-5 services)
  ✓ No need for auto-scaling or multi-node

Use Kubernetes when:
  ✓ Multi-node cluster needed
  ✓ Auto-scaling (HPA, VPA)
  ✓ Self-healing (pod restarts, rescheduling)
  ✓ Rolling updates with zero downtime
  ✓ Complex networking (service mesh)
  ✓ Many teams deploying independently
  ✓ Cloud-native infrastructure
```

---

> **Next:** 13 — From Docker to Kubernetes → Complete mapping of every Docker concept to its Kubernetes equivalent.
