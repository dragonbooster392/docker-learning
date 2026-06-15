# 11 — Container Registry

## What Is a Container Registry?

A registry is a **storage and distribution server** for container images. It's like GitHub, but for Docker images.

```
Developer builds image → pushes to registry → production pulls from registry

  docker build → docker push → [Registry] → docker pull (or K8s pulls)
```

```
Registry Types:

  Public registries (anyone can pull):
    Docker Hub        hub.docker.com         (default, free tier)
    GitHub GHCR       ghcr.io                (free for public repos)
    Quay.io           quay.io                (Red Hat)

  Cloud provider registries (private, integrated with cloud):
    AWS ECR           123456789.dkr.ecr.us-east-1.amazonaws.com
    Google GCR/AR     gcr.io / us-docker.pkg.dev
    Azure ACR         myregistry.azurecr.io

  Self-hosted (on your own infrastructure):
    Harbor            (CNCF project, enterprise features)
    Docker Registry   (official, minimal)
    Nexus/JFrog       (multi-format artifact managers)
```

---

## Image Naming and Registry Addresses

```
Full image reference:
  registry.example.com/namespace/repository:tag

Examples:
  docker.io/library/nginx:1.25           Docker Hub official image
  │         │       │     │
  │         │       │     └── tag (version)
  │         │       └── repository (image name)
  │         └── namespace (library = official)
  └── registry (docker.io = Docker Hub)

  ghcr.io/myorg/myapp:v2.1               GitHub Container Registry
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v2.1   AWS ECR
  myregistry.azurecr.io/backend/api:latest                Azure ACR

Docker Hub shortcuts:
  nginx           = docker.io/library/nginx:latest
  myuser/myapp    = docker.io/myuser/myapp:latest
```

---

## Push and Pull

### Pushing an Image

```bash
# 1. Build your image
docker build -t myapp:v1 .

# 2. Tag it for the registry
docker tag myapp:v1 docker.io/myuser/myapp:v1
#                   └── registry/namespace/repo:tag

# 3. Login to the registry
docker login
# Username: myuser
# Password: ****

# 4. Push
docker push docker.io/myuser/myapp:v1
```

```
What happens during push:
  1. Docker checks which layers the registry already has
  2. Only NEW layers are uploaded (others are skipped — "Layer already exists")
  3. The manifest (image config) is uploaded
  4. Registry acknowledges with the digest (sha256:abc...)

  $ docker push myuser/myapp:v1
  The push refers to repository [docker.io/myuser/myapp]
  5f70bf18a086: Layer already exists     ← registry has this layer
  a3ed95caeb02: Layer already exists     ← registry has this
  d8e1f35641ac: Pushed                   ← new layer uploaded
  v1: digest: sha256:abc123... size: 1234
```

### Pulling an Image

```bash
# Pull from Docker Hub (default registry)
docker pull nginx:1.25

# Pull from a specific registry
docker pull ghcr.io/myorg/myapp:v2.1

# Pull by digest (immutable — always the same image)
docker pull nginx@sha256:abc123def456...
```

```
What happens during pull:
  1. Docker resolves the tag to a manifest (via registry API)
  2. Manifest lists all required layers
  3. Docker checks which layers it already has locally
  4. Downloads only missing layers (parallel downloads)
  5. Assembles the image from layers
```

---

## Tagging Strategies

### The `:latest` Trap

```
:latest is NOT a version. It's just a tag name.
It doesn't mean "newest". It means "whatever was last pushed without a tag."

  docker build -t myapp .        ← tagged as myapp:latest
  docker push myapp              ← pushes myapp:latest

Problems:
  Monday:  docker pull myapp:latest → gets v1.0
  Tuesday: someone pushes v1.1 as :latest
  Wednesday: docker pull myapp:latest → gets v1.1 (DIFFERENT!)

  You can't reproduce Monday's deployment.
  You can't rollback ("which version was :latest on Monday?").
  If v1.1 has a bug, you're stuck.

RULE: Never use :latest in production. Always tag with a version.
```

### Semantic Versioning Tags

```
Best practice: tag with semantic version AND git SHA:

  myapp:2.1.3                ← exact version (most reliable)
  myapp:2.1                  ← minor version (auto-gets latest patch)
  myapp:2                    ← major version (auto-gets latest minor)
  myapp:a1b2c3d             ← git commit SHA (exact source mapping)
  myapp:2.1.3-a1b2c3d      ← both version and commit

  Push ALL tags at once:
  docker tag myapp:2.1.3 myuser/myapp:2.1.3
  docker tag myapp:2.1.3 myuser/myapp:2.1
  docker tag myapp:2.1.3 myuser/myapp:2
  docker tag myapp:2.1.3 myuser/myapp:a1b2c3d
  docker push myuser/myapp --all-tags
```

### Tag Immutability

```
Tags are MUTABLE by default:
  docker push myapp:v1     ← pushes image A
  docker push myapp:v1     ← pushes image B (A is now GONE!)

Digests are IMMUTABLE:
  myapp@sha256:abc123      ← always the same image, forever

Some registries support tag immutability:
  AWS ECR: imageTagMutability = IMMUTABLE
  Harbor: tag retention policies
  Docker Hub: no built-in immutability

For maximum safety:
  - Use immutable tags in production (ECR immutability)
  - Or reference images by digest in K8s manifests
```

---

## Authentication

### Docker Login

```bash
# Interactive login (prompts for username/password)
docker login

# Login to a specific registry
docker login ghcr.io
docker login 123456789.dkr.ecr.us-east-1.amazonaws.com

# Login with credentials (CI/CD)
echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
```

```
Credentials are stored in:
  ~/.docker/config.json

  {
    "auths": {
      "https://index.docker.io/v1/": {
        "auth": "base64(username:password)"    ← ⚠ NOT encrypted!
      }
    }
  }

For better security, use a credential helper:
  "credHelpers": {
    "123456789.dkr.ecr.us-east-1.amazonaws.com": "ecr-login"
  }
```

### Cloud Registry Authentication

```bash
# AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Google GCR (with gcloud)
gcloud auth configure-docker

# Azure ACR
az acr login --name myregistry
```

---

## Private Registries

### Why Private Registries?

```
1. Security:     Your code/images aren't public
2. Speed:        Images stored close to your infrastructure
3. Control:      Vulnerability scanning, signing, access policies
4. Compliance:   Data residency requirements
5. Cost:         No Docker Hub rate limits
```

### Docker Hub Rate Limits

```
Anonymous pulls:    100 pulls / 6 hours (per IP)
Authenticated:      200 pulls / 6 hours
Paid plans:         5000+ pulls / day

In CI/CD with many jobs, you'll hit these limits fast.
Solution: use a private registry or Docker Hub paid plan.
```

### AWS ECR (Most Common in AWS)

```bash
# Create a repository
aws ecr create-repository --repository-name myapp

# Login
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Tag for ECR
docker tag myapp:v1 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1

# Push
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

```
ECR features:
  ✓ Image scanning (CVE detection)
  ✓ Tag immutability
  ✓ Lifecycle policies (auto-delete old images)
  ✓ Cross-region replication
  ✓ IAM-based access control
  ✓ Integrated with ECS, EKS, Lambda
```

### Self-Hosted Registry (Harbor)

```bash
# Harbor is a CNCF project — enterprise-grade registry
# Features:
#   ✓ Vulnerability scanning (Trivy built-in)
#   ✓ Image signing (Notary/Cosign)
#   ✓ RBAC (role-based access control)
#   ✓ Replication between registries
#   ✓ Audit logs
#   ✓ Garbage collection
#   ✓ OCI-compliant
```

---

## Image Distribution — How Registries Work

```
Registry API (OCI Distribution Spec):

  1. Check if image exists:
     HEAD /v2/myapp/manifests/v1 → 200 (exists) or 404 (not found)

  2. Pull an image:
     GET /v2/myapp/manifests/v1 → returns manifest (list of layers)
     GET /v2/myapp/blobs/sha256:abc123 → download a layer

  3. Push an image:
     POST /v2/myapp/blobs/uploads/ → initiate layer upload
     PUT  /v2/myapp/blobs/uploads/{id}?digest=sha256:abc → upload layer
     PUT  /v2/myapp/manifests/v1 → upload manifest (finalizes push)

  4. List tags:
     GET /v2/myapp/tags/list → {"tags": ["v1", "v2", "latest"]}
```

### The Image Manifest

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:config...",
    "size": 7023
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:layer1...",
      "size": 32654
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:layer2...",
      "size": 16724
    }
  ]
}
```

```
The manifest is the "table of contents" for an image:
  - config: image metadata (env vars, CMD, labels, etc.)
  - layers: list of layer digests (each is a tar.gz of filesystem changes)

The image tag points to a manifest.
The manifest digest (sha256 of the manifest) is the TRUE image identity.
```

---

## Multi-Architecture Images

```
Modern images support multiple architectures:

  docker pull nginx:1.25
  → pulls the correct architecture for YOUR machine:
    - linux/amd64 (Intel/AMD)
    - linux/arm64 (Apple Silicon, AWS Graviton)
    - linux/arm/v7 (Raspberry Pi)

This works via a MANIFEST LIST (or OCI Image Index):

  nginx:1.25 manifest list:
    ├── linux/amd64  → sha256:abc... (x86 image)
    ├── linux/arm64  → sha256:def... (ARM image)
    └── linux/arm/v7 → sha256:ghi... (32-bit ARM image)

  Docker automatically selects the right one.
```

```bash
# Build for multiple architectures
docker buildx build --platform linux/amd64,linux/arm64 \
  -t myuser/myapp:v1 --push .

# Inspect manifest list
docker manifest inspect nginx:1.25
```

---

## Image Signing and Trust

### Docker Content Trust (DCT)

```bash
# Enable content trust (images must be signed)
export DOCKER_CONTENT_TRUST=1

# Now docker pull only accepts signed images:
docker pull nginx:1.25        # ✓ if signed
docker pull randomuser/myapp  # ✗ if not signed
```

### Cosign (Modern Alternative)

```bash
# Sign an image
cosign sign myregistry.com/myapp:v1

# Verify signature before deployment
cosign verify myregistry.com/myapp:v1

# In K8s: admission controllers can enforce signature verification
# (e.g., Kyverno policy: "only deploy signed images")
```

---

## CI/CD Pipeline Integration

```yaml
# GitHub Actions example
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Build and tag
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker tag ghcr.io/${{ github.repository }}:${{ github.sha }} \
                     ghcr.io/${{ github.repository }}:latest

      - name: Scan for vulnerabilities
        run: trivy image ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Push
        run: docker push ghcr.io/${{ github.repository }} --all-tags
```

```
CI/CD best practices:
  1. Build in CI (not on developer machines)
  2. Tag with git SHA (traceability)
  3. Scan before pushing (block vulnerable images)
  4. Push to private registry
  5. K8s pulls from registry on deployment
```

---

## Registry Cleanup

```bash
# AWS ECR: lifecycle policy (auto-delete old images)
aws ecr put-lifecycle-policy --repository-name myapp --lifecycle-policy-text '{
  "rules": [{
    "rulePriority": 1,
    "description": "Keep last 10 images",
    "selection": {
      "tagStatus": "any",
      "countType": "imageCountMoreThan",
      "countNumber": 10
    },
    "action": { "type": "expire" }
  }]
}'

# Docker Hub: manual or use retention tools
# Harbor: built-in garbage collection and tag retention
```

---

## How This Connects to Kubernetes

```
Registry Concept             → Kubernetes Equivalent
──────────────────           → ──────────────────────────────
docker pull                  → kubelet pulls automatically
docker login                 → imagePullSecrets in Pod spec
Private registry auth        → K8s Secret (type: kubernetes.io/dockerconfigjson)
Image tag                    → image: myapp:v1 in container spec
Image digest                 → image: myapp@sha256:abc...
Docker Hub rate limits       → Use private registry + pull-through cache
Image scanning               → Admission controller (block unscanned images)
Content trust / signing      → Kyverno / OPA policy (verify signatures)
Multi-arch images            → Automatic — K8s nodes pull correct arch
```

```yaml
# K8s: pulling from a private registry
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v2.1
  imagePullSecrets:
    - name: ecr-credentials    # Secret with registry auth
```

```
imagePullPolicy:
  Always          → always pull (checks for new image, uses cached layers)
  IfNotPresent    → only pull if not already on the node (DEFAULT for tagged)
  Never           → never pull (must be pre-loaded on node)

  :latest tag → defaults to Always (always checks for updates)
  :v1.2.3 tag → defaults to IfNotPresent (assumes tag is immutable)

  This is another reason to NEVER use :latest in production.
```

---

> **Next:** 12 — Best Practices & Troubleshooting → Image optimization, security checklist, logging, common errors and fixes.
