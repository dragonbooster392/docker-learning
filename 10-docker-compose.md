# 10 — Docker Compose

## What Is Docker Compose?

Docker Compose lets you define and run **multi-container applications** with a single YAML file.

```
Without Compose (managing 3 containers manually):

  docker network create myapp
  docker volume create pgdata
  docker run -d --name db --network myapp -v pgdata:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret postgres:16
  docker run -d --name cache --network myapp redis:7
  docker run -d --name web --network myapp -p 8000:8000 \
    -e DATABASE_URL=postgres://db:5432/mydb \
    -e REDIS_URL=redis://cache:6379 \
    myapp:v1

  That's 5 commands. Now imagine 10 services. Or running this on another machine.

With Compose (one file, one command):

  docker compose up -d
```

---

## The `docker-compose.yml` File

```yaml
# docker-compose.yml

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgres://postgres:secret@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    restart: unless-stopped

volumes:
  pgdata:
```

```
docker compose up -d

This single command:
  1. Creates a network (myproject_default)
  2. Creates a volume (myproject_pgdata)
  3. Pulls images (postgres:16, redis:7-alpine)
  4. Builds your app image (from Dockerfile)
  5. Starts all 3 containers
  6. Connects them to the same network
  7. Sets up DNS (web can reach "db" and "cache" by name)
```

---

## Services, Networks, and Volumes

A Compose file has three top-level sections:

```yaml
services:    # ← Containers (the things that run)
  web:
    ...
  db:
    ...

networks:    # ← Networks (how they communicate)
  frontend:
  backend:

volumes:     # ← Volumes (persistent data)
  pgdata:
  uploads:
```

---

## Service Configuration — Every Key

### Image vs Build

```yaml
services:
  # Use a pre-built image
  db:
    image: postgres:16

  # Build from a Dockerfile
  web:
    build: .                      # Dockerfile in current directory

  # Build with options
  api:
    build:
      context: ./api              # Build context directory
      dockerfile: Dockerfile.prod # Specific Dockerfile
      args:                       # Build arguments (ARG in Dockerfile)
        - APP_VERSION=2.1
      target: production          # Multi-stage build target
```

### Ports

```yaml
services:
  web:
    ports:
      - "8080:80"          # HOST:CONTAINER
      - "8443:443"         # Multiple ports
      - "127.0.0.1:9090:9090"  # Only localhost
      - "3000"             # Random host port → container 3000

    expose:
      - "9090"             # Exposed to other containers ONLY
                           # (not to the host — no port mapping)
```

### Environment Variables

```yaml
services:
  web:
    # Inline
    environment:
      - DATABASE_URL=postgres://db:5432/mydb
      - LOG_LEVEL=info
      - DEBUG=false

    # Or as a map (same thing):
    environment:
      DATABASE_URL: postgres://db:5432/mydb
      LOG_LEVEL: info

    # From a file
    env_file:
      - .env
      - .env.local        # Can use multiple files

    # Pass from host (no value = take from host environment)
    environment:
      - API_KEY            # Uses $API_KEY from the host
```

### Volumes

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data   # Named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # Bind mount
      - /tmp/cache:/app/cache             # Absolute path bind mount

  web:
    volumes:
      - ./src:/app/src                    # Development: mount source code
      - /app/node_modules                 # Anonymous volume (prevents override)

volumes:
  pgdata:                 # Declare named volumes
    driver: local         # Default driver (optional)
```

### Dependencies

```yaml
services:
  web:
    depends_on:
      - db
      - cache
    # Compose starts db and cache BEFORE web

  # Better: wait for service to be READY (not just started)
  web:
    depends_on:
      db:
        condition: service_healthy    # Wait for health check to pass
      cache:
        condition: service_started    # Just wait for container to start
```

### Health Checks

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  web:
    depends_on:
      db:
        condition: service_healthy
    # web starts ONLY after db's health check passes
```

### Restart Policies

```yaml
services:
  web:
    restart: unless-stopped   # Same as docker run --restart

# Options:
#   "no"              — never restart (default)
#   always            — always restart
#   on-failure        — restart only on non-zero exit
#   unless-stopped    — restart unless manually stopped
```

### Resource Limits

```yaml
services:
  web:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

### Networks

```yaml
services:
  web:
    networks:
      - frontend
      - backend        # Connected to both networks

  db:
    networks:
      - backend        # Only on backend (web can reach db, but
                       # external can't reach db directly)

networks:
  frontend:
  backend:
```

### Command and Entrypoint Override

```yaml
services:
  web:
    image: python:3.11
    command: python app.py              # Overrides CMD
    entrypoint: ["python"]              # Overrides ENTRYPOINT
    working_dir: /app                   # Overrides WORKDIR
    user: "1000:1000"                   # Overrides USER
```

---

## Compose Commands

```bash
# Start all services (detached)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Stop all services
docker compose down

# Stop and remove volumes too (⚠ data loss!)
docker compose down -v

# View running services
docker compose ps

# View logs
docker compose logs
docker compose logs -f web        # Follow specific service

# Execute command in a service
docker compose exec web bash

# Run a one-off command
docker compose run --rm web python manage.py migrate

# Scale a service (multiple instances)
docker compose up -d --scale web=3

# Restart a specific service
docker compose restart web

# Pull latest images
docker compose pull

# View configuration (merged, resolved)
docker compose config
```

---

## Networking in Compose

Compose creates a **default network** for your project automatically:

```yaml
# docker-compose.yml in /home/user/myproject/
services:
  web:
    image: nginx
  db:
    image: postgres
```

```
docker compose up creates:

  Network: myproject_default
  Containers:
    myproject-web-1 (172.20.0.2) — reachable as "web"
    myproject-db-1  (172.20.0.3) — reachable as "db"

  DNS works automatically:
    web → 172.20.0.2
    db  → 172.20.0.3

  The service NAME becomes the hostname.
  This is Docker's embedded DNS at work (file 06).
```

### Multi-Network Architecture

```yaml
services:
  # Public-facing web server
  nginx:
    image: nginx
    ports:
      - "80:80"
    networks:
      - frontend

  # Application (connects to both)
  app:
    build: .
    networks:
      - frontend
      - backend

  # Database (backend only — not reachable from nginx directly)
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend

  # Cache (backend only)
  cache:
    image: redis:7
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  pgdata:
```

```
Traffic flow:
  Internet → nginx (port 80) → app → db
                                    → cache

  nginx CANNOT reach db directly (different networks).
  This is the same network segmentation from file 06.
```

---

## The `.env` File

Compose automatically reads a `.env` file in the same directory:

```bash
# .env (loaded automatically by docker compose)
POSTGRES_VERSION=16
REDIS_VERSION=7
APP_PORT=8000
DB_PASSWORD=mysecretpassword
```

```yaml
# docker-compose.yml — uses variables from .env
services:
  db:
    image: postgres:${POSTGRES_VERSION}
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  cache:
    image: redis:${REDIS_VERSION}

  web:
    ports:
      - "${APP_PORT}:8000"
```

```
Variable substitution:
  ${VARIABLE}           Use the variable
  ${VARIABLE:-default}  Use "default" if VARIABLE is unset
  ${VARIABLE:?error}    Fail with "error" if VARIABLE is unset
```

---

## Profiles

Run different sets of services for different environments:

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    # No profile = always starts

  db:
    image: postgres:16
    # No profile = always starts

  debug:
    image: busybox
    profiles:
      - debug              # Only starts with --profile debug

  monitoring:
    image: grafana/grafana
    profiles:
      - monitoring         # Only starts with --profile monitoring
```

```bash
# Normal start (web + db only)
docker compose up -d

# Start with debug tools
docker compose --profile debug up -d

# Start with monitoring
docker compose --profile monitoring up -d

# Start everything
docker compose --profile debug --profile monitoring up -d
```

---

## Common Patterns

### Pattern 1: Web App + Database + Cache

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgres://postgres:secret@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    restart: unless-stopped

volumes:
  pgdata:
```

### Pattern 2: Development with Hot Reload

```yaml
services:
  web:
    build:
      context: .
      target: development        # Multi-stage: dev target
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src           # Mount source code (hot reload!)
      - /app/node_modules        # Don't override node_modules
    environment:
      - DEBUG=true
      - LOG_LEVEL=debug
    command: npm run dev          # Dev server with hot reload

  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=dev
    ports:
      - "5432:5432"             # Exposed for local DB tools
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### Pattern 3: Override Files (Dev vs Prod)

```yaml
# docker-compose.yml (base — shared config)
services:
  web:
    image: myapp:latest
    restart: unless-stopped

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```yaml
# docker-compose.override.yml (dev — auto-loaded!)
services:
  web:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src
    environment:
      - DEBUG=true
```

```yaml
# docker-compose.prod.yml (production — explicit)
services:
  web:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
    environment:
      - DEBUG=false
      - LOG_LEVEL=warning
```

```bash
# Dev (automatically merges base + override):
docker compose up -d

# Production (explicit file selection):
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Compose v1 vs v2

```
v1 (legacy):
  Command: docker-compose (hyphen, separate binary)
  File: docker-compose.yml with version: "3.8"
  Status: DEPRECATED (end of life June 2023)

v2 (current):
  Command: docker compose (space, built into Docker CLI)
  File: docker-compose.yml (no version field needed)
  Status: ACTIVE — use this

If you see tutorials with "docker-compose" (hyphen), replace with "docker compose" (space).
The version: field at the top of the file is optional in v2 and ignored.
```

---

## How This Connects to Kubernetes

```
Compose Concept              → Kubernetes Equivalent
──────────────────           → ──────────────────────────────
Service (in Compose)         → Deployment + Service (in K8s)
docker-compose.yml           → Multiple YAML manifests (or Helm chart)
docker compose up            → kubectl apply -f ./ (or helm install)
docker compose down          → kubectl delete -f ./
ports:                       → Service type: NodePort / LoadBalancer
environment:                 → env: in Pod spec, ConfigMap, Secret
volumes:                     → PVC + volumeMounts
depends_on                   → Init containers, readiness probes
healthcheck:                 → livenessProbe / readinessProbe
restart: unless-stopped      → restartPolicy: Always (default)
networks: (isolation)        → NetworkPolicy
deploy.resources.limits      → resources.limits
--scale web=3                → replicas: 3 in Deployment
.env file                    → ConfigMap / Secret
profiles:                    → Kustomize overlays / Helm values
Override files               → Kustomize overlays

Key difference:
  Compose = single host, declarative multi-container management
  K8s     = multi-host cluster, declarative everything management

Compose is great for:
  - Local development
  - CI/CD test environments
  - Simple single-host deployments

Kubernetes is for:
  - Production at scale
  - Multi-node clusters
  - Auto-scaling, self-healing, rolling updates
```

---

> **Next:** 11 — Container Registry → Push, pull, tagging strategies, private registries, and image distribution.
