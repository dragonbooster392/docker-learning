# 13 — From Docker to Kubernetes

## Why Kubernetes?

Docker runs containers on **one machine**. Kubernetes runs containers across a **cluster of machines** with:

```
Docker gives you:                  Kubernetes adds:
  ✓ Build images                     ✓ Run on MANY machines
  ✓ Run containers                   ✓ Auto-scaling (more traffic → more containers)
  ✓ Networks                         ✓ Self-healing (container dies → auto-restart)
  ✓ Volumes                          ✓ Rolling updates (zero-downtime deploys)
  ✓ Compose for multi-container      ✓ Service discovery (built-in DNS)
                                     ✓ Load balancing
                                     ✓ Secret management
                                     ✓ Resource scheduling
                                     ✓ Multi-team access control
```

---

## Architecture Mapping

```
Docker Architecture          → Kubernetes Architecture
──────────────────           → ──────────────────────────
Your laptop/server           → Cluster (many nodes)
docker CLI                   → kubectl CLI
Docker daemon                → kubelet (per node) + API server (control plane)
containerd                   → containerd (same!)
runc                         → runc (same!)
docker run                   → Pod (smallest deployable unit)
docker compose               → Deployment + Service (YAML manifests)
Docker Hub / ECR             → Same registries (OCI images work everywhere)
```

---

## The Complete Concept Map

### Container → Pod

```
Docker:
  docker run -d --name web nginx

Kubernetes:
  A Pod is one or more containers that share:
    - Network namespace (same IP, same ports)
    - Storage volumes
    - Lifecycle (started and stopped together)

  Usually 1 container per pod (sometimes sidecar pattern: 2+ containers)
```

```yaml
# Pod spec (you rarely create bare Pods — use Deployments instead)
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

### docker run flags → Pod spec

```
docker run -d \                        spec:
  --name web \                           containers:
  -p 8080:80 \                             - name: web
  -e DB_HOST=db \                            image: nginx:1.25
  -e DB_PORT=5432 \                          ports:
  --memory=512m \                              - containerPort: 80
  --cpus=2 \                               env:
  -v data:/app/data \                        - name: DB_HOST
  --restart=always \                           value: "db"
  --user 1000:1000 \                        - name: DB_PORT
  --read-only \                                value: "5432"
  --health-cmd="curl -f localhost/health" \  resources:
  nginx:1.25                                   limits:
                                                 memory: 512Mi
                                                 cpu: "2"
                                             volumeMounts:
                                               - name: data
                                                 mountPath: /app/data
                                             livenessProbe:
                                               httpGet:
                                                 path: /health
                                                 port: 80
                                             securityContext:
                                               runAsUser: 1000
                                               runAsGroup: 1000
                                               readOnlyRootFilesystem: true
                                           restartPolicy: Always
                                           volumes:
                                             - name: data
                                               persistentVolumeClaim:
                                                 claimName: data-pvc
```

### docker compose → Deployment + Service

```
Docker Compose:                    Kubernetes:

  services:                          Deployment (manages Pods)
    web:                               + Service (networking/load balancing)
      image: nginx:1.25               + ConfigMap (environment config)
      ports: ["8080:80"]              + Secret (passwords)
      environment:                     + PVC (storage)
        DB_HOST: db
      volumes:
        - data:/app/data
      depends_on: [db]
    db:
      image: postgres:16
```

---

## Networking Mapping

```
Docker Networking              → Kubernetes Networking
──────────────────             → ──────────────────────────────
Container IP (172.17.0.x)      → Pod IP (10.244.x.x)
docker0 bridge                 → CNI plugin (Calico, Flannel, Cilium)
Container name DNS             → Service name DNS
Custom network isolation       → NetworkPolicy

-p 8080:80 (port mapping)     → Service type: NodePort (host port)
                                 Service type: LoadBalancer (cloud LB)
                                 Ingress (HTTP routing)

Docker DNS (127.0.0.11)       → CoreDNS
"db" resolves to container IP  → "db-service" resolves to Service ClusterIP
                                 "db-service.namespace.svc.cluster.local"

Network: frontend, backend     → Namespace + NetworkPolicy
```

```
KEY DIFFERENCE:

Docker: containers need port mapping to be reachable from outside.
        Containers on different networks can't talk.

K8s:    Every pod gets a unique, routable IP.
        ANY pod can reach ANY other pod by default (flat network).
        NetworkPolicies RESTRICT traffic (deny by default when applied).
```

### Port Publishing

```
Docker:
  docker run -p 8080:80 nginx           ← maps host port to container

Kubernetes (3 options):

  1. ClusterIP (default — internal only):
     Only reachable within the cluster.
     Other pods access via: http://nginx-service:80

  2. NodePort (expose on every node):
     Cluster picks a port (30000-32767) on every node.
     External: http://node-ip:30123

  3. LoadBalancer (cloud load balancer):
     Cloud provider creates an actual LB (AWS ALB/NLB, GCP LB).
     External: http://my-lb.elb.amazonaws.com

  4. Ingress (HTTP routing):
     Routes based on hostname/path to different services.
     app.example.com/api → api-service
     app.example.com/web → web-service
```

---

## Storage Mapping

```
Docker Storage               → Kubernetes Storage
──────────────────           → ──────────────────────────────
Named volume                 → PersistentVolume (PV) + PersistentVolumeClaim (PVC)
docker volume create         → PV created by admin or StorageClass (dynamic)
-v mydata:/data              → volumeMounts in Pod spec
Volume driver (local)        → StorageClass (gp3, io2, efs, nfs)
Bind mount                   → hostPath (same caveats: not portable)
tmpfs                        → emptyDir with medium: Memory
Read-only mount (:ro)        → readOnly: true in volumeMount
Shared volume                → ReadWriteMany PV (EFS, NFS, CephFS)
```

```
K8s storage flow:

  1. Admin creates StorageClass:
     "Use AWS gp3 SSD for dynamic provisioning"

  2. Developer creates PVC:
     "I need 10Gi of storage with ReadWriteOnce access"

  3. K8s auto-provisions PV:
     Creates a 10Gi gp3 EBS volume in AWS

  4. Pod mounts the PVC:
     volumeMounts: [{name: data, mountPath: /app/data}]
     volumes: [{name: data, persistentVolumeClaim: {claimName: my-pvc}}]
```

---

## Security Mapping

```
Docker Security               → Kubernetes Security
──────────────────            → ──────────────────────────────
USER appuser                  → runAsNonRoot: true
                                runAsUser: 1000
--cap-drop ALL                → capabilities: {drop: ["ALL"]}
--cap-add NET_BIND_SERVICE    → capabilities: {add: ["NET_BIND_SERVICE"]}
--privileged                  → privileged: true (AVOID!)
--read-only                   → readOnlyRootFilesystem: true
no-new-privileges             → allowPrivilegeEscalation: false
--memory, --cpus              → resources.limits
Image scanning                → Admission controllers (block vulnerable images)
Docker Content Trust          → Image signing + verification policies
```

```yaml
# Kubernetes secure Pod spec (maps to Docker security best practices)
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  containers:
    - name: app
      image: myapp:v2.1.3
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "1"
          memory: "512Mi"
```

```
Pod Security Standards (PSS) — cluster-wide security enforcement:

  Privileged:   No restrictions (dev only)
  Baseline:     Blocks known escalation vectors
                (no privileged, no hostNetwork, no hostPID)
  Restricted:   Maximum security
                (non-root, drop ALL, read-only, no escalation)
                This maps to ALL the Docker security best practices from file 09.
```

---

## Configuration Mapping

```
Docker Config                → Kubernetes Config
──────────────────           → ──────────────────────────────
-e KEY=VALUE                 → env: in container spec
--env-file .env              → ConfigMap (non-sensitive config)
ENV in Dockerfile            → Also env: (but configurable at deploy time)
Secrets (runtime injection)  → Secret (base64 encoded, mountable)
docker-compose.override.yml  → Kustomize overlays / Helm values
```

```yaml
# ConfigMap (replaces .env files)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  DB_HOST: "db-service"
  DB_PORT: "5432"

# Secret (replaces docker secrets / runtime env vars)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0MTIz    # base64 encoded

# Pod using ConfigMap and Secret
spec:
  containers:
    - name: app
      envFrom:
        - configMapRef:
            name: app-config
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
```

---

## Lifecycle Mapping

```
Docker Lifecycle             → Kubernetes Lifecycle
──────────────────           → ──────────────────────────────
docker run                   → Pod creation (by Deployment controller)
docker stop                  → Pod termination (SIGTERM + grace period)
docker restart               → Pod restart (kubelet)
--restart=always             → restartPolicy: Always (default)
--restart=on-failure         → restartPolicy: OnFailure (Jobs)
HEALTHCHECK                  → livenessProbe (restart if unhealthy)
                               readinessProbe (remove from Service if not ready)
                               startupProbe (slow-starting apps)
depends_on                   → Init containers + readinessProbe
docker run --rm              → Job (run once, clean up)
```

```
K8s has THREE probe types (Docker only has one):

  livenessProbe:    "Is the container alive?"
                    If fails → K8s RESTARTS the container
                    Like Docker HEALTHCHECK + restart policy

  readinessProbe:   "Is the container ready to receive traffic?"
                    If fails → removed from Service (no traffic sent)
                    Container keeps running, just no traffic

  startupProbe:     "Has the container finished starting?"
                    Disables liveness/readiness during startup
                    For slow-starting apps (Java, heavy initialization)
```

---

## Command Mapping

```
Docker CLI                    → kubectl
──────────────────            → ──────────────────────────────
docker run -d nginx           → kubectl create deployment nginx --image=nginx
docker ps                     → kubectl get pods
docker ps -a                  → kubectl get pods --show-all
docker logs myapp             → kubectl logs pod-name
docker logs -f myapp          → kubectl logs -f pod-name
docker exec -it myapp bash    → kubectl exec -it pod-name -- bash
docker stop myapp             → kubectl delete pod pod-name
docker inspect myapp          → kubectl describe pod pod-name
docker stats                  → kubectl top pods
docker images                 → (check registry — K8s doesn't store images)
docker network ls             → kubectl get services / networkpolicies
docker volume ls              → kubectl get pv,pvc

docker compose up -d          → kubectl apply -f ./manifests/
docker compose down           → kubectl delete -f ./manifests/
docker compose logs           → kubectl logs -l app=myapp
docker compose scale web=3    → kubectl scale deployment web --replicas=3
```

---

## What Kubernetes Adds (Beyond Docker)

```
Feature                      Docker              Kubernetes
──────                       ──────              ──────────
Self-healing                 restart policy      Restart + reschedule to healthy node
Auto-scaling                 manual              HPA (Horizontal Pod Autoscaler)
Rolling updates              manual              Built-in (Deployment strategy)
Service discovery            Docker DNS           CoreDNS + Service abstraction
Load balancing               manual/nginx         Service + Ingress
Multi-node                   Swarm (deprecated)   Native — the whole point
Role-based access            docker group         RBAC (fine-grained per-namespace)
Configuration management     env files            ConfigMap + Secret
Package management           Compose              Helm charts
Multiple environments        Override files        Kustomize overlays / Helm values
Observability                docker stats          Prometheus + Grafana (metrics)
                                                   EFK/Loki (logs)
Network policies             Docker networks       NetworkPolicy (Calico, Cilium)
```

---

## Learning Path: Docker → Kubernetes

```
Now that you know Docker, here's what to learn for Kubernetes:

Foundation (you already know these from Docker):
  ✓ Containers, images, registries
  ✓ Networking (bridge, DNS, port mapping)
  ✓ Storage (volumes, bind mounts)
  ✓ Security (namespaces, cgroups, capabilities)

Kubernetes concepts to learn next:
  1. Pods                    (container wrapper — file 01 maps here)
  2. Deployments             (manage Pod replicas — docker compose maps here)
  3. Services                (networking + load balancing — port mapping maps here)
  4. ConfigMaps & Secrets    (configuration — env vars map here)
  5. Namespaces              (multi-tenant isolation)
  6. PV, PVC, StorageClass   (storage — volumes map here)
  7. Ingress                 (HTTP routing)
  8. RBAC                    (access control)
  9. NetworkPolicy           (network security — Docker networks map here)
  10. Helm                   (package management — compose maps here)
  11. HPA                    (auto-scaling)
  12. Pod Security Standards (security — file 09 maps directly)

Everything in this Docker course feeds directly into K8s.
You won't start from zero — you're starting from Docker.
```

---

> **Next:** 14 — Hands-On Lab → Real Docker commands on this machine with output and explanations.
