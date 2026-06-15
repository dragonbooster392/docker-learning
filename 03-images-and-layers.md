# 03 — Images & Layers

## What Is a Container Image?

An image is a **read-only package** containing everything your application needs to run:

```
┌──────────────────────────────┐
│     Your Application Code    │
├──────────────────────────────┤
│     Runtime (Python, Node)   │
├──────────────────────────────┤
│     System Libraries         │
├──────────────────────────────┤
│     OS Utilities (apt, ls)   │
├──────────────────────────────┤
│     Base OS (Ubuntu, Alpine) │
└──────────────────────────────┘
         Container Image
```

Key facts:
```
1. An image is READ-ONLY (immutable)
2. A container is a running instance of an image (with a writable layer on top)
3. One image → many containers (like a class → many objects)
4. Images are portable — build once, run anywhere with a container runtime
5. Images are stored locally and can be pushed to registries (Docker Hub, ECR, etc.)
```

---

## Images Are Made of Layers

This is the **most important concept** about images. An image isn't a single file — it's a **stack of layers**, each representing a change to the filesystem.

```
Image: python:3.11-slim

Layer 4:  Set ENTRYPOINT ["python3"]       (0 B — just metadata)
Layer 3:  Install pip, setuptools           (15 MB)
Layer 2:  Install Python 3.11               (45 MB)
Layer 1:  Install core packages             (20 MB)
Layer 0:  Debian Bookworm (base OS)         (50 MB — base layer)
          ─────────────────────────
          Total: ~130 MB
```

Each layer is a **diff** — it only contains what CHANGED from the layer below:

```
Layer 0 (base):  Contains /bin/ls, /etc/passwd, /usr/lib/...
Layer 1:         ADDS /usr/lib/libssl.so, /usr/bin/curl (doesn't copy base)
Layer 2:         ADDS /usr/bin/python3, /usr/lib/python3.11/...
Layer 3:         ADDS /usr/bin/pip, /usr/lib/python3.11/site-packages/pip/...
Layer 4:         CHANGES metadata only (entrypoint config)
```

---

## Why Layers Matter

### 1. Sharing — Layers Are Reused

If two images share the same base, the base layer is stored **only once**:

```
python:3.11-slim                  node:20-slim
┌───────────────────┐            ┌───────────────────┐
│ Python 3.11       │            │ Node.js 20        │
├───────────────────┤            ├───────────────────┤
│ Core packages     │            │ Core packages     │
├───────────────────┴────────────┴───────────────────┤
│           Debian Bookworm (SHARED!)                 │
└─────────────────────────────────────────────────────┘

On disk:
  Debian layer:  50 MB  ← stored ONCE
  Python layers: 80 MB
  Node layers:   70 MB
  Total: 200 MB (not 130 + 120 = 250 MB)
```

This is why `docker pull` is fast after the first image — most layers are already cached.

### 2. Caching — Builds Are Fast

Docker caches each layer. During a build, if a layer hasn't changed, Docker reuses the cache:

```
Dockerfile:
  FROM python:3.11-slim        ← cached ✓
  RUN pip install flask         ← cached ✓ (nothing changed)
  COPY app.py /app/             ← CHANGED (you edited app.py)
  CMD ["python", "/app/app.py"] ← must rebuild (layer above changed)

Docker rebuilds ONLY from the first changed layer downward.
Layers above the change are cached. This is why layer ORDER matters.
```

### 3. Efficiency — Pulling Is Layer-by-Layer

```
$ docker pull nginx:latest
latest: Pulling from library/nginx
a2abf6c4d29d: Already exists     ← layer cached from another image
a9edb18cadd1: Pull complete       ← only NEW layers downloaded
589b7251471a: Pull complete
186b1aaa4aa6: Pull complete
b4df32aa5a72: Pull complete
Digest: sha256:abc123...
Status: Downloaded newer image

Only 4 layers pulled — the base layer was already on disk!
```

---

## How Layers Work: Union Filesystem

Docker uses a **union filesystem** (usually OverlayFS on Linux) to stack layers:

```
What you see (merged view):
┌──────────────────────────────────────────┐
│  /                                        │
│  ├── bin/    (from Layer 0 + Layer 1)    │
│  ├── usr/    (from all layers)           │
│  ├── app/    (from Layer 3)              │
│  └── etc/    (from Layer 0, modified 2)  │
└──────────────────────────────────────────┘

How it's actually stored:
┌─── Container R/W Layer ──┐  ← writable (new files, changes)
├─── Layer 3 (COPY app/) ──┤  ← read-only
├─── Layer 2 (RUN pip) ────┤  ← read-only
├─── Layer 1 (RUN apt) ────┤  ← read-only
├─── Layer 0 (base OS) ────┤  ← read-only
└──────────────────────────┘
```

```
How OverlayFS handles operations:

READ a file:
  Start from top layer, work down until file is found.
  /app/app.py → found in Layer 3 → return it.
  /bin/ls → not in 3, not in 2, not in 1, found in Layer 0 → return it.

WRITE a new file (in running container):
  Written to the R/W layer ONLY. Image layers untouched.
  touch /tmp/test.txt → goes to R/W layer.

MODIFY an existing file (in running container):
  Copy-on-Write (CoW): the ENTIRE file is copied to R/W layer, then modified.
  Edit /etc/nginx/nginx.conf → copied from Layer 1 to R/W → modified in R/W.
  Original in Layer 1 is unchanged.

DELETE a file:
  A "whiteout" file is created in R/W layer (marks the file as deleted).
  rm /bin/curl → whiteout created → file appears gone, but still in the layer.
```

### Why Copy-on-Write Matters

```
100 containers from the same nginx image:

  Image layers (read-only):  ~140 MB — shared by ALL 100 containers
  Container R/W layers:      ~1 MB each (only what each container changed)

  Total disk: 140 MB + (100 × 1 MB) = 240 MB
  WITHOUT sharing: 100 × 141 MB = 14.1 GB

  97% space saved!
```

---

## Image Identification

### Image Names

```
Full format:  registry/repository:tag

Examples:
  docker.io/library/nginx:1.25          ← Docker Hub official image
  │         │       │     │
  │         │       │     └── tag (version)
  │         │       └── repository (image name)
  │         └── namespace (library = official images)
  └── registry (docker.io = Docker Hub)

  gcr.io/google-containers/pause:3.9    ← Google Container Registry
  123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v2.1  ← AWS ECR

Shortcuts (Docker Hub defaults):
  nginx              = docker.io/library/nginx:latest
  nginx:1.25         = docker.io/library/nginx:1.25
  myuser/myapp       = docker.io/myuser/myapp:latest
```

### Tags

```
Tags are human-readable labels pointing to a specific image version:

  nginx:1.25.3       ← specific version (BEST for production)
  nginx:1.25         ← minor version (might update patch)
  nginx:latest       ← always the newest (DANGEROUS for production)
  nginx:alpine       ← variant (Alpine-based, smaller)

  ⚠ WARNING: "latest" is NOT special.
  It's just a tag that's automatically used when you don't specify one.
  It CAN be outdated. It CAN be overwritten. NEVER use it in production.

  docker pull nginx            ← pulls nginx:latest (you don't know which version!)
  docker pull nginx:1.25.3     ← pulls exactly this version (reproducible)
```

### Image Digest (SHA256)

```
Every image has a unique digest — a SHA256 hash of the image manifest:

  nginx@sha256:abc123def456...    ← immutable, cryptographic identifier

  Unlike tags, a digest CANNOT change.
  nginx:latest might point to different images over time.
  nginx@sha256:abc123... ALWAYS refers to the exact same image.

  Use digests for maximum reproducibility:
    FROM nginx@sha256:abc123def456...
```

---

## Inspecting Images

### `docker images` — List Local Images

```bash
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
nginx         1.25      abc123def456   2 weeks ago    187MB
python        3.11      789abc012def   3 weeks ago    1.01GB
alpine        3.19      321fed654cba   4 weeks ago    7.38MB
```

```
IMAGE ID:  First 12 chars of the SHA256 digest
SIZE:      Total size (all layers decompressed)
           Shared layers are counted in EACH image, so the total
           on disk is usually less than sum of all image sizes.
```

### `docker history` — See the Layers

```bash
$ docker history nginx:1.25
IMAGE          CREATED BY                                      SIZE
<missing>      CMD ["nginx" "-g" "daemon off;"]                0B
<missing>      EXPOSE 80                                       0B
<missing>      STOPSIGNAL SIGQUIT                              0B
<missing>      COPY 30-tune-worker-processes.sh /docker-e…    4.62kB
<missing>      COPY 20-envsubst-on-templates.sh /docker-e…    3.02kB
<missing>      COPY 10-listen-on-ipv6-by-default.sh /dock…    2.12kB
<missing>      COPY docker-entrypoint.sh / RUN chmod +x /…    1.62kB
<missing>      RUN /bin/sh -c set -x && addgroup --system…    61.1MB
<missing>      ENV DYNPKG_RELEASE=2~bookworm                   0B
<missing>      ENV NJS_VERSION=0.8.2                           0B
<missing>      ENV NGINX_VERSION=1.25.3                        0B
<missing>      RUN /bin/sh -c apt-get update && apt-get i…    92.2MB
<missing>      /bin/sh -c #(nop) ADD file:a5a...              74.8MB
```

```
Reading this (bottom to top, that's how layers stack):
  74.8 MB  — Base OS layer (Debian Bookworm)
  92.2 MB  — System packages (apt-get install)
  0 B      — ENV instructions (metadata only, no layer created)
  61.1 MB  — nginx binary and dependencies
  ~10 KB   — COPY scripts (entrypoint, config)
  0 B      — EXPOSE, CMD, STOPSIGNAL (metadata only)
```

### `docker inspect` — Full Details

```bash
$ docker inspect nginx:1.25 | jq '.[0].RootFS'
{
  "Type": "layers",
  "Layers": [
    "sha256:571ade696b26...",    ← Layer 0
    "sha256:b4a6eb95a06f...",    ← Layer 1
    "sha256:d3f5f6b97a8a...",    ← Layer 2
    "sha256:97576f5c3b0c...",    ← Layer 3
    "sha256:1f41b2f23e8e...",    ← Layer 4
    "sha256:6e4c56282c4c...",    ← Layer 5
    "sha256:5ce6debc5b44..."     ← Layer 6
  ]
}
```

---

## Image Storage on Disk

```
Docker stores everything under /var/lib/docker/:

/var/lib/docker/
├── overlay2/          ← layer contents (OverlayFS)
│   ├── abc123.../     ← layer 1 data
│   │   ├── diff/      ← actual files in this layer
│   │   ├── link       ← shortened link name
│   │   └── lower      ← pointer to layer below
│   ├── def456.../     ← layer 2 data
│   └── ...
├── image/             ← image metadata
│   └── overlay2/
│       ├── imagedb/   ← image configs (JSON)
│       └── layerdb/   ← layer chain metadata
├── containers/        ← container configs and R/W layers
└── volumes/           ← named volumes
```

```bash
# See total disk usage
$ docker system df
TYPE            TOTAL   ACTIVE  SIZE     RECLAIMABLE
Images          12      5       4.2GB    2.1GB (50%)
Containers      8       3       120MB    80MB (66%)
Local Volumes   4       2       500MB    200MB (40%)
Build Cache     15      0       1.2GB    1.2GB (100%)

# Clean up unused images, containers, and cache
$ docker system prune -a
```

---

## Base Images

Every image starts from a base. Choosing the right base matters:

```
Base Image        Size      Use Case
──────────        ────      ───────────────────────────────────
scratch           0 B       Static Go/Rust binaries (no OS at all)
alpine:3.19       7 MB      Minimal Linux (musl libc, busybox)
debian:bookworm   124 MB    Standard Linux (glibc, apt)
  -slim variant   74 MB     Debian without docs/man pages
ubuntu:24.04      78 MB     Popular Linux (apt, good community)
```

### The `scratch` Image

```
scratch is literally NOTHING — empty filesystem.
Only works if your binary is fully static (no shared libraries).

FROM scratch
COPY myapp /myapp
CMD ["/myapp"]

This produces the smallest possible image (just your binary).
Common for Go and Rust applications.
```

### Alpine vs Debian

```
Alpine (7 MB):
  ✅ Tiny image size
  ✅ Minimal attack surface (fewer packages = fewer vulnerabilities)
  ⚠  Uses musl libc (not glibc) — some software has compatibility issues
  ⚠  Uses busybox (simplified versions of GNU tools)
  ⚠  Debugging is harder (no strace, gdb by default)

Debian Slim (74 MB):
  ✅ Standard glibc — everything just works
  ✅ Easier debugging
  ✅ Better compatibility with C extensions (Python, Node native modules)
  ⚠  Larger image

Rule of thumb:
  Go/Rust static binary → scratch or alpine
  Python/Node/Java       → debian slim (alpine can cause headaches with native deps)
  Quick prototype         → ubuntu (most familiar, largest but most tooling)
```

---

## Image vs Container — The Key Difference

```
              Image (read-only)
              ┌──────────────┐
              │ Layer 3      │
              │ Layer 2      │
              │ Layer 1      │
              │ Layer 0      │
              └──────┬───────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Container A   Container B   Container C
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ R/W Layer│ │ R/W Layer│ │ R/W Layer│  ← unique per container
   ├──────────┤ ├──────────┤ ├──────────┤
   │ (shared  │ │ (shared  │ │ (shared  │  ← shared image layers
   │  image   │ │  image   │ │  image   │
   │  layers) │ │  layers) │ │  layers) │
   └──────────┘ └──────────┘ └──────────┘
```

```
Image:
  - Read-only
  - Has no state
  - Can exist without containers
  - Can be pushed to a registry

Container:
  - Has a writable layer on top of the image
  - Has state (running processes, modified files)
  - Created from an image
  - Writable layer is LOST when container is removed
    (unless you use volumes — file 08)
```

---

## Building Your Own Image

You define an image with a **Dockerfile** (covered in detail in file 04):

```dockerfile
# Start from a base
FROM python:3.11-slim

# Each instruction creates a LAYER
RUN pip install flask            # Layer: installs flask
COPY app.py /app/                # Layer: adds your code
WORKDIR /app                     # Metadata: sets working directory
CMD ["python", "app.py"]         # Metadata: default command
```

```
Build it:
  $ docker build -t my-app:v1 .

What happens:
  Layer 0: python:3.11-slim (pulled from Docker Hub if not cached)
  Layer 1: RUN pip install flask (runs pip, captures filesystem changes)
  Layer 2: COPY app.py /app/ (copies file from build context)
  Metadata: WORKDIR, CMD (no filesystem changes, stored in image config)
```

---

## Layer Best Practices (Preview)

```
BAD — creates 3 layers, intermediate apt cache wastes space:
  RUN apt-get update
  RUN apt-get install -y curl
  RUN rm -rf /var/lib/apt/lists/*

GOOD — single layer, cache cleaned in same layer:
  RUN apt-get update && \
      apt-get install -y curl && \
      rm -rf /var/lib/apt/lists/*

WHY?
  Deleting files in a LATER layer doesn't shrink the image.
  The file still exists in the earlier layer.
  You must clean up in the SAME layer where you created it.
```

```
BAD — COPY before pip install (busts cache on every code change):
  COPY . /app/
  RUN pip install -r requirements.txt

GOOD — install deps first, then copy code:
  COPY requirements.txt /app/
  RUN pip install -r requirements.txt
  COPY . /app/

WHY?
  requirements.txt rarely changes → pip install layer stays cached.
  Code changes often → only the last COPY layer rebuilds.
```

More on this in file 04 (Dockerfile).

---

## Key Takeaways

```
1. An image is a stack of READ-ONLY layers
2. Each Dockerfile instruction can create a new layer
3. Layers are shared between images (saves disk space and download time)
4. Containers add a thin WRITABLE layer on top of image layers
5. Copy-on-Write: modified files are copied to R/W layer
6. Deleting in a later layer doesn't shrink the image — clean up in same layer
7. OverlayFS merges all layers into a single unified filesystem view
8. Use tags for human-readable versions, digests for immutable references
9. NEVER use :latest in production
```

---

## How This Connects to Kubernetes

```
Docker Images concept        → Kubernetes equivalent
──────────────────           → ──────────────────────────────
docker pull                  → kubelet pulls automatically
docker build                 → CI/CD pipeline builds, then pushes
Image layers + caching       → Same — K8s uses containerd's cache
Image tags                   → Pod spec: image: nginx:1.25
Image digests                → image: nginx@sha256:abc...
Local images                 → Must be in a registry for K8s to pull
docker.io (Docker Hub)       → Any OCI registry (ECR, GCR, ACR, etc.)
```

---

> **Next:** 04 — Dockerfile → Every instruction explained, multi-stage builds, and image optimization.
