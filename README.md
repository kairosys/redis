# Redis in Kubernetes (Local Development)

Standalone single-instance **Redis 8** (`redis:8-alpine`) for local development, deployed into the `default` namespace. It serves as a backing store for queued jobs and is configured with password authentication plus a non-evicting eviction policy so queue data does not disappear under load.

## Architecture & Specs

| Aspect | Detail |
|---|---|
| Image | `redis:8-alpine` |
| Namespace | `default` |
| Replicas | 1 (Deployment) |
| Service | `ClusterIP` on port **6379** → targetPort **6379** (`app=redis`) |
| Storage | Inline `hostPath` volume, mountPath `/data`; host directory `/mnt/workspaces/redis/data`, type `DirectoryOrCreate`. *No PVC/PV exists in these manifests.* Local `data/` is gitignored. |
| Requests | cpu: `50m`, memory: `64Mi` |
| Limits | cpu: `500m`, memory: `256Mi` |

> No `livenessProbe`/`readinessProbe` are defined, so the Deployment reports ready as soon as the container starts. No PodDisruptionBudget is configured; a rolling restart may briefly evict in-flight traffic.

## Configuration & Secrets

Only one secret value is referenced by these manifests:

| Field | Source | Notes |
|---|---|---|
| `REDIS_PASS` | Injected via env-var into `--requirepass $(REDIS_PASS)` from Kubernetes Secret `redis-secret`. Not committed: see `.gitignore` (`k8s/*-secret.yaml`). Password is plaintext in the checked-in YAML under `stringData.REDIS_PASS`. Keep out of version control. |

Redis runtime flags (passed as a container command override) are not environment variables but still shape behavior:
```yaml
command: ["redis-server", "--maxmemory-policy", "noeviction", "--requirepass", "$(REDIS_PASS)"]
```
- `REDIS_PORT` is **not** set in any Secret or manifest; the Service uses the default Redis port 6379.
- Database index defaults to Redis's standard DB `0`; no `--databases`/`SELECT` override appears in these manifests.

> ⚠️ Network-policy note: `maxmemory-policy=noeviction` means once **`256Mi`** memory usage exceeds limit, writes will fail with errors instead of evicting keys — expect this when queues fill rapidly during local load tests.

## Deployment Guide

Apply in order; the Secret must exist before the Deployment starts because the pod command references `$(REDIS_PASS)` at container launch:
```bash
kubectl apply -f k8s/redis-secret.yaml      # creates Secret in default namespace
kubectl apply -f k8s/redis-deployment.yaml  # creates Service + Deployment
```

## Health Checks & Diagnostics

Validate the deployment and reachability (the password below is read from Secret `redis-secret`; never hardcode it into your commands in committed docs):
```bash
# 1. Confirm pod is running and its name:
POD=$(kubectl get pods -l app=redis -o jsonpath='{.items[0].metadata.name}') && \
kubectl wait --for=condition=ready "pod/$POD"

# 2. From inside the pod, test Redis auth + reachability:
PASS=$(kubectl get secret redis-secret -o jsonpath='{.data.REDIS_PASS}' | base64 -d) && \
kubectl exec "pod/$POD" -- redis-cli -a "$PASS" ping      # expect: PONG

# Inspect memory and config (same $PASS pattern applies):
kubectl exec "$POD" -- redis-server --version             # Redis 8.x
kubectl exec "$POD" -- redis-cli -a "$PASS" INFO MEMORY
kubectl exec "$POD" -- redis-cli -a "$PASS" CONFIG GET maxmemory-policy    # noeviction

# Local data directory is gitignored — to reset persistence:
kubectl delete pod "$POD"             # Pod will recreate with hostPath remount
rm /mnt/workspaces/redis/data/dump.rdb                # clears the local mount source if you restart minikube/kind afterward
```