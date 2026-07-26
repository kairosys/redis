# AGENTS.md

This repo is a standalone Redis instance for local development, deployed into the `furseal` namespace on Kubernetes.

## What's here
- `k8s/redis-deployment.yaml` — Service + Deployment (image: `redis:8-alpine`). The server runs with `--maxmemory-policy noeviction` and `--requirepass $(REDIS_PASS)`.
- `k8s/redis-secret.yaml` — Secret named `redis-secret` exposing `REDIS_PORT=...`/`REDIS_PASS` (`stringData`, plaintext in the YAML, `.gitignore`'d). Passwords are referenced by env-var only, never hardcoded in manifests.
- `data/` — local directory bound into the container as a `hostPath` volume at `/mnt/workspaces/redis/data`. Redis persistence files live here.

## Apply / redeploy
```bash
kubectl apply -f k8s/redis-secret.yaml   # first (Deployment references it)
kubectl apply -f k8s/redis-deployment.yaml
```

## Important gotchas for agents
- **Do not edit `redis-secret.yaml` to supply production passwords** — keep them in the Secret (`REDIS_PASS`), never inline.
- **`data/` is gitignored.** Wiping or moving it destroys persisted Redis state; if you must reset, do it from the running pod, not by deleting the local files while a session is mounted.
- **Network policy:** `maxmemory-policy noeviction`, so once memory fills the server will reject writes rather than evict — expect this under heavy queue load.

## Namespace note
Resources use namespace `furseal` and may need `kubectl -n furseal ...` on clusters where default differs, or your user lacks RBAC for it.