# 08 — Docker Storage

## The Problem

By default, all files created inside a container are stored in the **container's writable layer**:

```
┌─────── Container ───────┐
│  R/W Layer               │ ← files written here
│  (deleted when container │
│   is removed!)           │
├──────────────────────────┤
│  Image layers (read-only)│
└──────────────────────────┘
```

```
Problems with the writable layer:
  1. DATA LOSS:    docker rm my-db → all data GONE
  2. NOT PORTABLE: data is tied to that specific container
  3. SLOW:         Copy-on-Write has overhead for write-heavy workloads
  4. NOT SHARED:   other containers can't access the data
```

Docker solves this with **three storage options**:

```
┌─────────────────────────────────────────────────────┐
│                                                      │
│  Volumes         Bind Mounts        tmpfs            │
│  ┌─────────┐    ┌──────────┐     ┌──────────┐      │
│  │ /var/lib │    │ /home/   │     │  RAM     │      │
│  │ /docker/ │    │ user/    │     │  (memory)│      │
│  │ volumes/ │    │ project/ │     │          │      │
│  └────┬─────┘    └────┬─────┘     └────┬─────┘      │
│       └───────────────┴────────────────┘             │
│                    │                                  │
│              ┌─────┴─────┐                           │
│              │ Container │                           │
│              │ /app/data │                           │
│              └───────────┘                           │
└──────────────────────────────────────────────────────┘
```

---

## Option 1: Volumes (Recommended)

Volumes are **managed by Docker**, stored in `/var/lib/docker/volumes/`.

```bash
# Create a named volume
docker volume create mydata

# Run with a volume mounted
docker run -d --name db \
  -v mydata:/var/lib/postgresql/data \
  postgres:16

# The -v flag:
#   mydata                        ← volume name
#   :                             ← separator
#   /var/lib/postgresql/data      ← mount point inside container
```

### How Volumes Work

```
Host filesystem:
  /var/lib/docker/volumes/mydata/_data/
  └── actual files stored here (owned by Docker)

Container sees:
  /var/lib/postgresql/data/
  └── same files (mounted into the container)

  Data in the volume SURVIVES:
    ✓ docker stop / docker start
    ✓ docker rm (container removed, volume stays)
    ✓ Docker daemon restart
    ✓ Host reboot

  Data is LOST only when:
    ✗ docker volume rm mydata (explicit deletion)
    ✗ docker system prune --volumes
```

### Named vs Anonymous Volumes

```bash
# Named volume — you choose the name
docker run -v mydata:/data myapp
# Volume: mydata (easy to find and manage)

# Anonymous volume — Docker generates a random name
docker run -v /data myapp
# Volume: 7a8b9c0d1e2f... (hard to identify)

# VOLUME in Dockerfile creates anonymous volumes:
# VOLUME ["/data"]
# docker run myapp → creates anonymous volume at /data

Always use NAMED volumes — anonymous volumes are orphaned easily.
```

### Volume Commands

```bash
# List all volumes
$ docker volume ls
DRIVER    VOLUME NAME
local     mydata
local     db-backup
local     7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d    ← anonymous (what is this?)

# Inspect a volume
$ docker volume inspect mydata
[{
    "Name": "mydata",
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
    "Labels": {},
    "Scope": "local"
}]

# Remove a volume
docker volume rm mydata

# Remove ALL unused volumes (⚠ data loss!)
docker volume prune

# See which volumes a container uses
docker inspect my-db | jq '.[0].Mounts'
```

### Sharing Volumes Between Containers

```bash
# Container 1: writes data
docker run -d --name writer \
  -v shared-data:/data \
  myapp

# Container 2: reads same data
docker run -d --name reader \
  -v shared-data:/data:ro \
  myapp

# Both containers see the same files at /data
# :ro means read-only for the reader container
```

```
Use cases for shared volumes:
  - Log aggregator reading app logs
  - Sidecar containers processing data
  - Init containers populating config

⚠ No file locking! If two containers write to the same file
  simultaneously, data corruption can occur. Use databases or
  message queues for concurrent writes.
```

---

## Option 2: Bind Mounts

Bind mounts map a **host directory** directly into the container.

```bash
# Mount current directory into the container
docker run -d --name dev \
  -v $(pwd)/src:/app/src \
  myapp

# Or with the --mount flag (more explicit):
docker run -d --name dev \
  --mount type=bind,source=$(pwd)/src,target=/app/src \
  myapp
```

### How Bind Mounts Work

```
Host filesystem:
  /home/user/project/src/
  ├── app.py
  ├── config.py
  └── utils.py

Container sees:
  /app/src/
  ├── app.py       ← same files!
  ├── config.py
  └── utils.py

Changes go BOTH ways:
  Edit app.py on host → container sees the change immediately
  Container creates log.txt → appears on host immediately

No copy. No sync. It's the SAME directory.
```

### Bind Mounts vs Volumes

```
Feature              Volume                     Bind Mount
───────              ──────                     ──────────
Managed by           Docker                     You
Location             /var/lib/docker/volumes/    Anywhere on host
Portable             ✓ (just volume name)       ✗ (path varies per host)
Pre-populated        ✓ (image data copied in)   ✗ (host dir overrides)
Backup               docker volume commands      cp, rsync, tar
Permission control   Docker manages              Your responsibility
Performance          Best (native storage)       Same (native fs)
Development          ✗ (hidden in /var/lib)     ✓ (edit files with IDE)
Production           ✓ (preferred)              ✗ (path dependency)
```

```
When to use what:

Volumes:
  ✓ Database storage (postgres, mysql data)
  ✓ Application state that must survive container restart
  ✓ Data shared between containers
  ✓ Production

Bind mounts:
  ✓ Development (edit code on host, run in container)
  ✓ Config files (mount a config file into the container)
  ✓ Build contexts (mount source code for compilation)
  ✗ Not for production (path might not exist on another machine)
```

### Bind Mount Gotchas

```
1. HOST DIRECTORY OVERRIDES CONTAINER DIRECTORY

   Image has: /app/config/default.yml
   You mount: -v ./myconfig:/app/config
   Container sees: ONLY your myconfig/ — default.yml is HIDDEN

   Volumes don't have this problem — Docker copies image data to the volume
   if it's empty.

2. FILE PERMISSIONS

   Host file owned by: user (UID 1000)
   Container runs as:  appuser (UID 999)
   → Container can't read the file!

   Fix: match UIDs or use --user flag:
   docker run --user 1000:1000 -v ./src:/app/src myapp

3. PATH MUST BE ABSOLUTE

   docker run -v ./src:/app/src myapp    ← ERROR on some systems
   docker run -v $(pwd)/src:/app/src myapp  ← ✓ absolute path
```

### Development Workflow with Bind Mounts

```bash
# Mount your code into the container — hot reload!
docker run -d --name dev \
  -p 8000:8000 \
  -v $(pwd)/src:/app/src \
  -v $(pwd)/requirements.txt:/app/requirements.txt \
  myapp:dev

# Edit src/app.py in your IDE → container picks up changes instantly
# (if your framework supports hot reload: Flask debug, nodemon, etc.)
```

---

## Option 3: tmpfs Mounts

tmpfs stores data **in memory only** — nothing written to disk.

```bash
docker run -d --name secure \
  --tmpfs /app/secrets:rw,size=64m \
  myapp

# Or:
docker run -d --name secure \
  --mount type=tmpfs,destination=/app/secrets,tmpfs-size=67108864 \
  myapp
```

```
tmpfs:
  ✓ Fast (RAM speed)
  ✓ Data never touches disk (security)
  ✓ Gone when container stops
  ✗ Limited by available RAM
  ✗ Not shared between containers

Use cases:
  - Temporary secrets in memory
  - Scratch space for processing
  - Session storage
  - /tmp for security-sensitive apps
```

---

## The `-v` Flag vs `--mount` Flag

```bash
# -v (short form) — most common
docker run -v mydata:/data myapp
docker run -v $(pwd)/src:/app/src myapp
docker run -v mydata:/data:ro myapp

# --mount (long form) — more explicit, recommended for complex mounts
docker run --mount type=volume,source=mydata,target=/data myapp
docker run --mount type=bind,source=$(pwd)/src,target=/app/src myapp
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m myapp
```

```
Differences:
  -v creates the volume if it doesn't exist (silently!)
  --mount FAILS if the source doesn't exist (safer!)

  For production, --mount is recommended because it's explicit
  and won't silently create empty volumes.
```

### Mount Options

```bash
# Read-only mount
-v mydata:/data:ro
--mount type=volume,source=mydata,target=/data,readonly

# Read-write (default)
-v mydata:/data:rw

# Specific mount options
--mount type=volume,source=mydata,target=/data,volume-opt=type=nfs
```

---

## Storage Drivers (Underlying Mechanism)

Docker uses storage drivers to manage image layers and the container R/W layer:

```
Storage Driver     How It Works                  Default On
──────────────     ──────────────────────────    ──────────
overlay2           OverlayFS (union filesystem)  Modern Linux (recommended)
aufs               Advanced Union FS             Older Ubuntu
devicemapper       Block-level snapshots          CentOS/RHEL (legacy)
btrfs              Btrfs snapshots               Btrfs filesystems
zfs                ZFS snapshots                 ZFS filesystems
```

```
Check your storage driver:
  $ docker info | grep "Storage Driver"
  Storage Driver: overlay2

overlay2 is the default and best choice for most setups.
Don't change it unless you have a specific reason.
```

### overlay2 Deep Dive

```
/var/lib/docker/overlay2/

Each directory is a layer:

abc123.../
├── diff/          ← the actual files in this layer
│   ├── usr/
│   │   └── bin/
│   │       └── python3   ← added by this layer
│   └── etc/
│       └── pip.conf      ← added by this layer
├── link           ← shortened symlink name (for performance)
├── lower          ← pointer to layer below (parent)
├── merged/        ← union view (all layers merged — only for containers)
└── work/          ← OverlayFS workspace (internal)

For a running container:
  merged/ shows the unified filesystem (all layers + R/W)
  diff/   shows only what THIS container has written
  lower   points to the image's top layer
```

---

## Data Patterns

### Database Container

```bash
# PostgreSQL with named volume
docker run -d --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16

# Data survives container removal:
docker rm -f postgres
docker run -d --name postgres-new \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
# All your data is still there!
```

### Backup a Volume

```bash
# Backup: run a temporary container that mounts the volume and creates a tar
docker run --rm \
  -v pgdata:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata-backup.tar.gz -C /source .

# Restore: run a temporary container that extracts the tar into the volume
docker run --rm \
  -v pgdata:/target \
  -v $(pwd):/backup:ro \
  alpine tar xzf /backup/pgdata-backup.tar.gz -C /target
```

### Config File Injection

```bash
# Mount a single config file (not the whole directory)
docker run -d --name nginx \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -p 80:80 \
  nginx

# :ro prevents the container from modifying your config
```

---

## Storage Summary

```
Type          Stored Where           Survives rm?    Shared?    Speed
────          ────────────           ────────────    ───────    ─────
R/W Layer     /var/lib/docker/       ✗               ✗          Medium
Volume        /var/lib/docker/vol/   ✓               ✓          Fast
Bind Mount    Anywhere on host       ✓               ✓          Fast
tmpfs         RAM (memory)           ✗               ✗          Fastest
```

---

## How This Connects to Kubernetes

```
Docker Storage               → Kubernetes equivalent
──────────────────           → ──────────────────────────────
Named volume                 → PersistentVolume (PV)
docker volume create         → PV created by admin or dynamic provisioning
-v mydata:/data              → volumeMounts in Pod spec
Volume claim ("I need 10Gi") → PersistentVolumeClaim (PVC)
Volume driver (local, nfs)   → StorageClass (gp3, io2, nfs, ceph)
Bind mount                   → hostPath (same warnings: not portable!)
tmpfs                        → emptyDir with medium: Memory
Shared volume                → ReadWriteMany PV (NFS, EFS, CephFS)
:ro mount                    → readOnly: true in volumeMount

Key K8s concepts:
  PV           = the actual storage (disk, NFS share)
  PVC          = "I need X amount of storage with Y properties"
  StorageClass = "Use this provisioner to create PVs automatically"

  Pod → PVC → PV → actual disk (EBS, NFS, local, etc.)
```

---

> **Next:** 09 — Docker Security → Namespaces, cgroups, capabilities, rootless Docker, image scanning, attack scenarios.
