# 04 — Dockerfile

## What Is a Dockerfile?

A Dockerfile is a **text file with instructions** that Docker reads to build an image, line by line, top to bottom.

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "app.py"]
```

```
Each instruction:
  1. Starts from the previous layer
  2. Makes a change (adds files, runs a command, sets metadata)
  3. Creates a new layer (or metadata entry)
  4. That layer becomes the starting point for the next instruction
```

---

## Every Dockerfile Instruction

### FROM — The Starting Point

```dockerfile
FROM python:3.11-slim
```

```
Every Dockerfile MUST start with FROM (or ARG before FROM).
This sets the base image — the foundation everything else builds on.

FROM ubuntu:24.04          ← start from Ubuntu
FROM python:3.11-slim      ← start from Python (already includes Debian + Python)
FROM node:20-alpine        ← start from Node.js on Alpine
FROM scratch               ← start from nothing (empty filesystem)
```

### RUN — Execute a Command During Build

```dockerfile
RUN apt-get update && apt-get install -y curl
RUN pip install flask
```

```
RUN executes a command inside a temporary container and saves
the result as a new layer.

Two forms:
  RUN apt-get update              ← shell form (runs in /bin/sh -c)
  RUN ["apt-get", "update"]      ← exec form (runs directly, no shell)

Each RUN = one layer. Combine related commands to reduce layers:

BAD (3 layers):
  RUN apt-get update
  RUN apt-get install -y curl wget
  RUN rm -rf /var/lib/apt/lists/*

GOOD (1 layer):
  RUN apt-get update && \
      apt-get install -y curl wget && \
      rm -rf /var/lib/apt/lists/*

WHY combine?
  Deleting files in a LATER layer doesn't remove them from earlier layers.
  The apt cache from "apt-get update" is baked into that layer forever.
  Cleaning up in the SAME RUN command means the cache never appears in any layer.
```

### COPY — Add Files from Build Context

```dockerfile
COPY app.py /app/
COPY . /app/
COPY --chown=appuser:appuser config/ /app/config/
```

```
COPY takes files from the BUILD CONTEXT (the directory you pass to docker build)
and adds them to the image.

  docker build -t myapp .
                         └── "." = build context = current directory

COPY . /app/            ← copies everything from build context to /app/ in image
COPY app.py /app/       ← copies just one file
COPY *.py /app/         ← wildcard — copies all .py files
COPY --chown=1000:1000  ← sets ownership (avoids running as root)
```

### COPY vs ADD ⚠️ (Interview Question!)

```dockerfile
COPY app.tar.gz /app/          # Copies the tar file as-is
ADD  app.tar.gz /app/          # Auto-extracts the tar into /app/!
ADD  https://example.com/f.txt /app/  # Downloads from URL!
```

```
COPY:
  ✓ Copies files. That's it. Simple and predictable.

ADD:
  ✓ Does everything COPY does, PLUS:
    - Auto-extracts .tar, .tar.gz, .tar.bz2 archives
    - Can download from URLs (but DON'T — no caching, no checksums)

Rule: ALWAYS use COPY unless you specifically need tar extraction.
ADD's implicit behaviors cause confusion and unexpected results.

  COPY = explicit, predictable ✓
  ADD  = magical, surprising ✗
```

### WORKDIR — Set the Working Directory

```dockerfile
WORKDIR /app
```

```
Sets the working directory for all subsequent RUN, CMD, ENTRYPOINT,
COPY, and ADD instructions.

  WORKDIR /app
  RUN pwd            ← prints /app
  COPY . .           ← copies to /app/
  CMD ["python", "app.py"]  ← runs in /app/

If the directory doesn't exist, WORKDIR creates it.

DON'T do this:
  RUN cd /app && python app.py    ← cd only works for that one RUN

DO this:
  WORKDIR /app
  RUN python app.py               ← now /app is the working directory
```

### ENV — Set Environment Variables

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV FLASK_APP=app.py
ENV DB_HOST=localhost DB_PORT=5432
```

```
ENV sets environment variables that persist in the image AND running container.

  ENV MY_VAR=hello
  RUN echo $MY_VAR     ← "hello" (available during build)

  docker run myimage
  $ echo $MY_VAR       ← "hello" (available at runtime too)

⚠ NEVER put secrets in ENV:
  ENV DB_PASSWORD=secret123    ← BAD! Visible in docker history and inspect!

  Use runtime injection instead:
  docker run -e DB_PASSWORD=secret123 myimage
```

### ARG — Build-time Variables

```dockerfile
ARG PYTHON_VERSION=3.11
FROM python:${PYTHON_VERSION}-slim
ARG APP_VERSION=1.0
RUN echo "Building version ${APP_VERSION}"
```

```
ARG sets variables available ONLY during build (not at runtime).

  ARG PYTHON_VERSION=3.11         ← default value
  FROM python:${PYTHON_VERSION}   ← used in FROM

  Override at build time:
  docker build --build-arg PYTHON_VERSION=3.12 .
```

### ARG vs ENV ⚠️ (Interview Question!)

```
                  Available during build?    Available at runtime?    In docker history?
ARG               ✓                          ✗ (gone)                 ✓ (⚠ visible!)
ENV               ✓                          ✓ (persists)             ✓ (⚠ visible!)

Neither is safe for secrets! Both appear in image metadata.

For secrets:
  Build-time → Docker BuildKit secrets:
    docker build --secret id=mysecret,src=./secret.txt .
  Runtime → docker run -e or Docker secrets (file 09)
```

### EXPOSE — Document the Port

```dockerfile
EXPOSE 8000
EXPOSE 8000/tcp 9000/udp
```

```
EXPOSE does NOT actually publish the port.
It's DOCUMENTATION — tells users which ports the app listens on.

  EXPOSE 8000    ← "this app expects traffic on 8000"

You still need -p to actually publish:
  docker run -p 8000:8000 myimage    ← NOW port is accessible

Think of EXPOSE like a comment for ports:
  # This app uses port 8000    ← same thing, just structured metadata
```

### CMD — Default Command

```dockerfile
CMD ["python", "app.py"]
CMD python app.py
```

```
CMD sets the DEFAULT command that runs when a container starts.

Two forms:
  CMD ["python", "app.py"]      ← exec form (preferred — no shell wrapper)
  CMD python app.py             ← shell form (runs as /bin/sh -c "python app.py")

CMD can be OVERRIDDEN at runtime:
  docker run myimage                    ← runs CMD: python app.py
  docker run myimage bash               ← overrides CMD: runs bash instead
  docker run myimage python test.py     ← overrides CMD: runs python test.py

Only the LAST CMD in a Dockerfile takes effect.
```

### ENTRYPOINT — The Fixed Command

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

```
ENTRYPOINT sets the command that ALWAYS runs. CMD becomes arguments to it.

  ENTRYPOINT ["python"]
  CMD ["app.py"]

  docker run myimage                  ← runs: python app.py
  docker run myimage test.py          ← runs: python test.py
  docker run myimage -c "print(1)"   ← runs: python -c "print(1)"

The user can change arguments (CMD) but NOT the base command (ENTRYPOINT).
```

### ENTRYPOINT vs CMD ⚠️ (Top Interview Question!)

```
Scenario 1: CMD only
  CMD ["python", "app.py"]
  docker run myimage              → python app.py
  docker run myimage bash         → bash (CMD completely replaced)

Scenario 2: ENTRYPOINT only
  ENTRYPOINT ["python", "app.py"]
  docker run myimage              → python app.py
  docker run myimage bash         → python app.py bash (bash appended as arg!)

Scenario 3: ENTRYPOINT + CMD (most common pattern)
  ENTRYPOINT ["python"]
  CMD ["app.py"]
  docker run myimage              → python app.py (CMD = default arg)
  docker run myimage test.py      → python test.py (CMD overridden)

Summary:
  CMD alone        → easy to override entire command (flexible)
  ENTRYPOINT alone → hard to change (rigid)
  ENTRYPOINT + CMD → fixed base command, flexible arguments (BEST PATTERN)
```

```
Real-world patterns:

# Web server — always runs gunicorn, default args overridable
ENTRYPOINT ["gunicorn"]
CMD ["--workers=4", "--bind=0.0.0.0:8000", "app:app"]

# CLI tool — always runs the tool, user provides arguments
ENTRYPOINT ["aws"]
CMD ["--help"]

# Simple app — just run it
CMD ["python", "app.py"]
```

### USER — Run as Non-Root

```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
```

```
USER sets which user runs the subsequent RUN, CMD, ENTRYPOINT.

Default is root (UID 0) — this is a security risk (file 09).

Best practice:
  # Create a non-root user
  RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser

  # Switch to it
  USER appuser

  # Now CMD runs as appuser, not root
  CMD ["python", "app.py"]
```

### HEALTHCHECK — Is the Container Healthy?

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

```
HEALTHCHECK tells Docker how to check if your application is working.

  --interval=30s   Check every 30 seconds
  --timeout=3s     If check takes longer than 3s, it failed
  --retries=3      After 3 failures, mark container as "unhealthy"
  CMD              The command to run (exit 0 = healthy, exit 1 = unhealthy)

Container states:
  starting    → first health check hasn't run yet
  healthy     → last check succeeded
  unhealthy   → last N checks failed

Docker shows this in:
  $ docker ps
  STATUS: Up 5 minutes (healthy)

⚠ In Kubernetes, DON'T use HEALTHCHECK — use livenessProbe / readinessProbe instead.
  K8s ignores Docker HEALTHCHECK.
```

### LABEL — Metadata

```dockerfile
LABEL maintainer="team@example.com"
LABEL version="1.0" description="My web app"
```

```
LABEL adds metadata to the image (key=value pairs).
No effect on the container — just informational.

Inspect labels:
  docker inspect myimage | jq '.[0].Config.Labels'
```

### VOLUME — Declare a Mount Point

```dockerfile
VOLUME ["/data"]
```

```
VOLUME creates a mount point and marks it for external storage.
When a container starts, Docker creates an anonymous volume at this path.

Any data written to /data by the container goes to the volume,
not the container's R/W layer.

⚠ Better to use explicit volumes at runtime:
  docker run -v mydata:/data myimage
  (Covered in file 08)
```

### STOPSIGNAL — Graceful Shutdown Signal

```dockerfile
STOPSIGNAL SIGTERM
```

```
What signal docker stop sends to PID 1 in the container.
Default is SIGTERM (graceful shutdown).

  docker stop mycontainer
    1. Sends SIGTERM (or your STOPSIGNAL)
    2. Waits 10 seconds (configurable with -t)
    3. Sends SIGKILL (force kill)
```

### SHELL — Change the Default Shell

```dockerfile
SHELL ["/bin/bash", "-c"]
RUN echo "now using bash, not sh"
```

```
Changes the shell used for shell-form RUN, CMD, ENTRYPOINT.
Default is ["/bin/sh", "-c"].
Useful when you need bash features (arrays, pipefail, etc.)
```

---

## .dockerignore — What NOT to Copy

```
# .dockerignore

.git
.gitignore
node_modules
__pycache__
*.pyc
.env
.venv
docker-compose*.yml
README.md
Dockerfile
.dockerignore
```

```
Without .dockerignore, COPY . . sends EVERYTHING to Docker daemon:
  .git/        → 100+ MB of git history
  node_modules → 500+ MB of dependencies
  .env         → YOUR SECRETS!

With .dockerignore:
  - Faster builds (less data sent to daemon)
  - Smaller images (no junk files)
  - Safer builds (no secrets accidentally copied)
  - Better caching (irrelevant file changes don't bust cache)
```

---

## Multi-Stage Builds

The most powerful Dockerfile feature for **image size optimization**.

### The Problem

```dockerfile
# BAD — everything ends up in the final image
FROM python:3.11
RUN apt-get update && apt-get install -y gcc   # 200 MB of build tools
RUN pip install --no-cache-dir cython myapp
# gcc is still in the image but NOT needed at runtime!
# Final image: 1.2 GB
```

### The Solution: Multi-Stage

```dockerfile
# Stage 1: BUILD (heavy, with build tools)
FROM python:3.11 AS builder
RUN apt-get update && apt-get install -y gcc
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: RUNTIME (slim, no build tools)
FROM python:3.11-slim
COPY --from=builder /install /usr/local
COPY app.py /app/
CMD ["python", "/app/app.py"]
```

```
What happens:
  Stage 1 (builder):
    - Full Python image with gcc
    - Installs dependencies (might need gcc to compile C extensions)
    - pip install goes to /install (isolated directory)

  Stage 2 (final):
    - Slim Python image (no gcc, no apt cache)
    - COPY --from=builder copies ONLY the installed packages
    - Your app code
    - Everything else from Stage 1 is THROWN AWAY

  Result:
    Without multi-stage: 1.2 GB (includes gcc, apt cache, etc.)
    With multi-stage:    180 MB (just runtime + your code)
```

### Go Example (Even More Dramatic)

```dockerfile
# Stage 1: Build the binary
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o myapp .

# Stage 2: Run the binary (from scratch!)
FROM scratch
COPY --from=builder /app/myapp /myapp
CMD ["/myapp"]
```

```
  Stage 1 image: ~1.2 GB (Go compiler, source code, dependencies)
  Final image:   ~10 MB  (just the compiled binary, no OS!)

  scratch = empty filesystem. The Go binary is statically linked,
  so it needs NOTHING else. No bash, no ls, no libc — just the binary.
```

### Node.js Example

```dockerfile
# Stage 1: Install dependencies and build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Runtime
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## Layer Cache Optimization

### The Golden Rule

```
Put instructions that change LEAST at the TOP.
Put instructions that change MOST at the BOTTOM.

Because: Docker rebuilds from the FIRST changed layer downward.
         Everything above the change is cached.
```

### Python Example

```dockerfile
# ✅ OPTIMAL layer order
FROM python:3.11-slim

# 1. System deps (rarely change)
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# 2. Python deps (change sometimes)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 3. Your code (changes often)
COPY . /app/
WORKDIR /app

# 4. Metadata (never causes rebuilds)
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

```
When you change ONLY your code:
  FROM python:3.11-slim      ← cached ✓
  RUN apt-get...             ← cached ✓
  COPY requirements.txt .    ← cached ✓ (requirements.txt didn't change)
  RUN pip install...         ← cached ✓
  COPY . /app/               ← CHANGED (your code changed)
  WORKDIR /app               ← rebuilds (below the change)
  EXPOSE 8000                ← rebuilds
  CMD [...]                  ← rebuilds

  Only 3 layers rebuild, pip install is CACHED.
  Build time: seconds instead of minutes.
```

```dockerfile
# ❌ BAD layer order — pip install runs every time you change code
FROM python:3.11-slim
COPY . /app/                          ← changes every time
RUN pip install -r /app/requirements.txt  ← cache busted! rebuilds every time!
CMD ["python", "/app/app.py"]
```

---

## Complete Real-World Examples

### Python Flask App

```dockerfile
FROM python:3.11-slim

# Don't run as root
RUN groupadd -r app && useradd -r -g app -d /app -s /sbin/nologin app

WORKDIR /app

# Install dependencies first (cache optimization)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY --chown=app:app . .

# Switch to non-root user
USER app

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:app"]
```

### Node.js Express App

```dockerfile
FROM node:20-alpine

RUN addgroup -S app && adduser -S app -G app

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --chown=app:app . .

USER app

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

### Go Microservice (Multi-Stage)

```dockerfile
# Build
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache git
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server .

# Runtime
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

```
Notes:
  -ldflags="-s -w"    → strips debug info (smaller binary)
  CGO_ENABLED=0        → fully static binary (no C dependencies)
  ca-certificates.crt  → needed if your app makes HTTPS calls
  scratch              → ~8 MB final image
```

---

## Dockerfile Linting

Use `hadolint` to catch common mistakes:

```bash
$ hadolint Dockerfile
DL3008 Pin versions in apt get install
DL3009 Delete the apt-get lists after installing
DL3025 Use JSON form for CMD/ENTRYPOINT
DL4006 Set SHELL with pipefail for pipe commands
SC2086 Double quote to prevent globbing
```

---

## Instruction Summary Table

```
Instruction    Creates Layer?    Purpose
───────────    ──────────────    ────────────────────────────────────
FROM           -                 Sets base image
RUN            ✓                 Executes command, saves result
COPY           ✓                 Copies files from build context
ADD            ✓                 COPY + auto-extract + URL download
WORKDIR        ✗ (metadata)      Sets working directory
ENV            ✗ (metadata)      Sets environment variables (persist!)
ARG            ✗ (metadata)      Sets build-time variables (gone at runtime)
EXPOSE         ✗ (metadata)      Documents port (doesn't publish it)
CMD            ✗ (metadata)      Default command (overridable)
ENTRYPOINT     ✗ (metadata)      Fixed command (arguments overridable)
USER           ✗ (metadata)      Sets runtime user
HEALTHCHECK    ✗ (metadata)      Defines health check command
LABEL          ✗ (metadata)      Adds key=value metadata
VOLUME         ✗ (metadata)      Declares mount point
STOPSIGNAL     ✗ (metadata)      Shutdown signal
SHELL          ✗ (metadata)      Changes default shell
```

---

## How This Connects to Kubernetes

```
Dockerfile concept           → Kubernetes equivalent
──────────────────           → ──────────────────────────────
FROM                         → image: in Pod spec
CMD / ENTRYPOINT             → command: / args: in container spec
ENV                          → env: in container spec, or ConfigMap
EXPOSE                       → containerPort: (also documentation)
HEALTHCHECK                  → livenessProbe / readinessProbe (K8s ignores HEALTHCHECK)
USER                         → securityContext.runAsUser
VOLUME                       → volumeMounts + PVC
--cpus, --memory (run-time)  → resources.requests / resources.limits
Multi-stage build            → Same — CI/CD builds image, pushes to registry
.dockerignore                → Same — used during docker build in CI/CD
```

---

> **Next:** 05 — Container Lifecycle → run, stop, rm, exec, logs, environment variables, resource limits, debugging.
