# Platform Project 1 — GitOps with ArgoCD

A self-hosted Kubernetes cluster (kubeadm, on Multipass VMs) running a minimal FastAPI
app, deployed and kept in sync entirely through GitOps: push to `master`, and the
running app updates itself with no manual `kubectl` involved.

## Architecture

```
 ┌─────────────┐   git push   ┌──────────────────┐
 │  Developer  │ ────────────▶│  GitHub repo      │
 └─────────────┘               │  (this repo)      │
                                └─────────┬─────────┘
                                          │ triggers
                                          ▼
                                ┌───────────────────┐
                                │  GitHub Actions    │
                                │  - build image      │
                                │  - push to Docker Hub│
                                │  - update k8s/       │
                                │    deployment.yaml    │
                                │  - commit back        │
                                └─────────┬─────────┘
                                          │ new commit
                                          ▼
                                ┌───────────────────┐
                                │  ArgoCD             │
                                │  watches k8s/ on     │
                                │  master, auto-syncs  │
                                └─────────┬─────────┘
                                          │ applies manifests
                                          ▼
                     ┌────────────────────────────────────┐
                     │  Kubernetes cluster (kubeadm)         │
                     │  k8s-control (control plane)          │
                     │  k8s-worker1, k8s-worker2 (workers)   │
                     │  CNI: Flannel                          │
                     └────────────────────────────────────┘
```

## Stack

- **App**: FastAPI (`app.py`) with a `/health` endpoint, containerized with a plain
  `python:3.11-slim` Dockerfile.
- **Cluster**: 3 Ubuntu VMs on Multipass (Apple Silicon / arm64), bootstrapped with
  `kubeadm` — one control plane, two workers. CNI is Flannel.
- **Registry**: Docker Hub (`oge1ata/demo-app`).
- **GitOps**: ArgoCD, installed in-cluster, watching this repo's `k8s/` folder with
  automated sync + self-heal enabled.
- **CI**: GitHub Actions (`.github/workflows/deploy.yml`) builds a `linux/arm64` image
  (matching the cluster's architecture, cross-built with QEMU since GitHub's runners
  are amd64), pushes it tagged with the commit SHA, updates `k8s/deployment.yaml`, and
  commits that change back to the repo.

## Repo layout

```
app.py                     the FastAPI app
Dockerfile
requirements.txt
k8s/deployment.yaml         Deployment + Service — what ArgoCD syncs
argocd/application.yaml     the ArgoCD Application definition (applied once, manually)
.github/workflows/deploy.yml  CI: build, push, update manifest, commit
learning_log.md              running notes from building this
```

## How a change ships

1. Push a commit to `master`.
2. GitHub Actions builds the image, tags it with the commit SHA, pushes to Docker Hub,
   rewrites the image line in `k8s/deployment.yaml`, and commits that back (with
   `[skip ci]` so it doesn't retrigger itself).
3. ArgoCD notices the new commit on its next poll (default ~3 min, or force it with
   `kubectl -n argocd annotate application demo-app argocd.argoproj.io/refresh=hard`)
   and rolls out the new image.
4. No manual `kubectl apply` at any point.

## One-time setup this required

- `kubeadm init` on the control plane, `kubeadm join` on both workers.
- containerd's cgroup driver had to be set to `systemd` (`SystemdCgroup = true` in
  `/etc/containerd/config.toml`) to match kubelet — a mismatch here silently
  crash-loops every control-plane static pod. See `learning_log.md` for the full
  debugging story.
- ArgoCD installed with `kubectl apply --server-side` (its CRDs are too large for a
  normal client-side `kubectl apply`).
- GitHub repo settings: `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets, and
  Settings → Actions → "Read and write permissions" so the workflow can commit back.

## Local dev

```
pip install -r requirements.txt
uvicorn app:app --reload
curl http://localhost:8000/health
```
