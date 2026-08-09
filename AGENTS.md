# AGENTS.md

Standalone Redis 8 (`redis:8-alpine`) in the `default` namespace. Backing store for queued jobs.

## What's here
- `k8s/redis-deployment.yaml` — Service (ClusterIP:6379) + Deployment. Runs with `--maxmemory-policy noeviction` and `--requirepass $(REDIS_PASS)`.
- `k8s/redis-secret.yaml` — Secret `redis-secret` with `REDIS_PASS`. Gitignored (`.gitignore: k8s/*-secret.yaml`). Passwords via env-var only, never hardcoded.
- `data/` — Host path `/mnt/workspaces/redis/data` mounted at `/data` in the container. Gitignored.

## Apply / redeploy (order matters)
```bash
kubectl apply -f k8s/redis-secret.yaml          # first — Deployment references it
kubectl apply -f k8s/redis-deployment.yaml
```

## Diagnostics
```bash
REDIS_PASS=$(kubectl get secret redis-secret -o jsonpath='{.data.REDIS_PASS}' | base64 -d)
POD=$(kubectl get pod -l app=redis -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD" -- redis-cli -a "$REDIS_PASS" PING
kubectl exec "$POD" -- redis-cli -a "$REDIS_PASS" INFO MEMORY
```

## Reset persistence
```bash
kubectl delete pod "$POD"                         # pod recreates, hostPath remounts
```

## Gotchas
- **Never edit `redis-secret.yaml` to supply passwords** — keep them in the Secret, never inline.
- **`data/` is gitignored.** Deleting it locally while a pod is mounted destroys state. Reset from the running pod instead.
- **Memory limit: 256Mi.** With `noeviction`, writes will be rejected once memory fills — expect this under heavy queue load.
- **`$REDIS_PASS` expands in the local shell, not the pod** — export it first or fetch from the Secret (see Diagnostics), else `redis-cli` runs with an empty password and gets `NOAUTH`.
- **`k8s/*-external-ingress.yaml` is gitignored** — external ingress manifests are expected to live uncommitted.
