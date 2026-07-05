# Deploy NanoClaw to Kubernetes (single pod + PVC + DinD)

This directory deploys the NanoClaw **host** as a single-replica Deployment with
a **Docker-in-Docker (DinD)** sidecar and one **RWO PVC**, targeting the
`cankp-primary` cluster (NKP / containerd nodes).

## Why DinD?

NanoClaw's host process is a Docker orchestrator: for every session it runs
`docker run ... -v <hostPath>:<containerPath> ...` to spawn a per-session agent
container (`src/container-runner.ts`), and `CONTAINER_RUNTIME_BIN` is hardcoded
to `docker`. The nodes run **containerd, not Docker**, so we give the pod its own
Docker daemon via a privileged DinD sidecar.

Because agent containers use **bind mounts** rooted under `PROJECT_ROOT`
(`data/`, `groups/`, `container/`), the daemon that runs them must see those
exact paths. So the host and DinD containers both mount the PVC at the **same**
path (`/data/nanoclaw`), and the repo itself lives on the PVC.

```
Pod (replicas: 1, Recreate)
├── initContainer seed   -> copies /opt/nanoclaw-src to PVC:app, chown 1000
├── container host        -> node dist/index.js, cwd=/data/nanoclaw,
│                            DOCKER_HOST=tcp://127.0.0.1:2375
└── container dind        -> docker:27-dind (privileged)
                             /data/nanoclaw  (PVC:app, SAME path as host)
                             /var/lib/docker (PVC:dind, agent image persists)
```

## Files

| File | Purpose |
|------|---------|
| `Dockerfile.host` | NanoClaw host image: Node 22 + pnpm + docker CLI + repo (deps installed, `dist/` built). |
| `../.github/workflows/build-host-image.yml` | GitHub Actions: builds `Dockerfile.host` and pushes to GHCR. |
| `k8s/namespace.yaml` | `nanoclaw` namespace, privileged Pod Security profile. |
| `k8s/pvc.yaml` | Single RWO PVC on `nutanix-volume`. |
| `k8s/secret.yaml` | Env (`ASSISTANT_NAME`, `TZ`, `ONECLI_URL`, `ONECLI_API_KEY`). |
| `k8s/deployment.yaml` | Seed init + host + privileged DinD sidecar, pulling from GHCR. |

## Prerequisites

- The forked repo on GitHub (`script-repo/nanoclaw`) with the included Actions
  workflow — this builds the image in the cloud, so no local Docker is needed.
- `kubectl` access to `cankp-primary`.
- Cluster egress to `ghcr.io` and `pkg-containers.githubusercontent.com`.

## 1. Build & publish the host image (via GitHub Actions)

Push these files to your fork's `main` branch (or trigger the workflow
manually). `.github/workflows/build-host-image.yml` builds `linux/amd64` and
pushes to `ghcr.io/script-repo/nanoclaw-host:latest` (plus a `sha-<short>` tag).

After the **first** successful run, make the package **public** so the cluster
can pull it without a secret: GitHub → your profile/org → Packages →
`nanoclaw-host` → Package settings → Change visibility → Public.

> Prefer building locally instead? `docker build -f deploy/Dockerfile.host -t ghcr.io/script-repo/nanoclaw-host:latest .` then `docker push ...` (after `docker login ghcr.io`).

If you keep the package **private**, create a pull secret and reference it in
`deployment.yaml` under `spec.template.spec.imagePullSecrets`:

```bash
kubectl -n nanoclaw create secret docker-registry ghcr \
  --docker-server=ghcr.io --docker-username=script-repo --docker-password=<PAT-with-read:packages>
```

## 2. Apply the manifests

```bash
kubectl apply -f deploy/k8s/namespace.yaml
kubectl apply -f deploy/k8s/pvc.yaml
kubectl apply -f deploy/k8s/secret.yaml
kubectl apply -f deploy/k8s/deployment.yaml
```

Watch it come up (the seed init copies the repo onto the PVC on first start):

```bash
kubectl -n nanoclaw get pods -w
kubectl -n nanoclaw logs deploy/nanoclaw -c host -f
```

You should see `DinD reachable. Starting NanoClaw host.` once the sidecar is up.

## 3. First-time setup (interactive, via exec)

Agents cannot spawn until the OneCLI credential vault is configured (the host
refuses to spawn without it). Run the setup wizard inside the host container:

```bash
kubectl -n nanoclaw exec -it deploy/nanoclaw -c host -- bash
# inside the container (cwd = /data/nanoclaw):
pnpm setup
```

The wizard will: register your Anthropic credential with OneCLI, build the
agent-runner image inside DinD (persisted on the PVC via `/var/lib/docker`), and
pair your first channel (Telegram/Discord/etc.). Any env it writes lands in
`/data/nanoclaw/.env` on the PVC.

## 4. Verify

```bash
# From the host container, confirm the daemon and (after setup) the agent image:
kubectl -n nanoclaw exec deploy/nanoclaw -c host -- docker info
kubectl -n nanoclaw exec deploy/nanoclaw -c host -- docker images
```

Send a test message on the paired channel and confirm an agent container appears:

```bash
kubectl -n nanoclaw exec deploy/nanoclaw -c host -- docker ps
```

## Operational notes

- **State** persists on the PVC: app data/DBs/groups under subPath `app`, the
  built agent image + layers under subPath `dind`.
- **Restarts** use `Recreate` so two hosts never touch the PVC at once.
- **Node affinity**: the image is pulled from GHCR (any node), but the RWO PVC
  still ties scheduling to the node its volume attaches to. That's handled
  automatically by the CSI driver on reschedule.
- **Privileged**: the DinD sidecar requires a privileged container; if cluster
  policy forbids this, this approach won't work as-is.
