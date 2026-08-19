#What I've built so far

##🗂️ The App
Created a project folder with 4 files: app.py, requirements.txt, Dockerfile, deployment.yaml
app.py is a minimal FastAPI app with a /health endpoint — just enough to have something to deploy
Pushed the whole folder to GitHub — this repo is what ArgoCD will watch later

##🐳 Docker
Confirmed Docker Desktop and Homebrew were already installed
Built the Docker image locally with docker build
Ran the container and confirmed curl http://localhost:8000/health returned {"status":"ok"}
Proved the app works inside a container exactly as it will in Kubernetes

##☸️ Kubernetes VMs
Installed Multipass — the tool that runs lightweight Ubuntu VMs on your Mac Mini
Spun up 3 VMs:
k8s-control — will be the Kubernetes control plane (the brain of the cluster)
k8s-worker1 — will run your actual workloads
k8s-worker2 — will run your actual workloads
On all 3 VMs installed:
kubeadm — the tool that bootstraps the cluster
kubelet — the agent that runs on every node
kubectl — the CLI for talking to the cluster
containerd — the container runtime (what actually runs your containers inside Kubernetes)


##🐛 Debugging: API server refusing connections
After kubeadm init, kubectl couldn't reach the API server at all — connection refused.
Two separate bugs stacked on top of each other:
1. **Cgroup driver mismatch** — kubelet was set to manage containers using `systemd` cgroups, but containerd was still configured for the older `cgroupfs` style. That mismatch made every core Kubernetes component (etcd, the API server, the scheduler, the controller manager) get killed within seconds of starting, over and over — which is why containers kept flashing in and out of `crictl ps`. Fixed by setting `SystemdCgroup = true` in containerd's config on all 3 VMs and restarting containerd.
2. **Stale kubeconfig** — even after the fix, kubectl still failed, this time with a certificate error. My local `~/.kube/config` still had the CA certificate from an earlier, abandoned `kubeadm init` attempt, so it didn't trust the cluster's actual certificate anymore. Fixed by re-copying the current `/etc/kubernetes/admin.conf` over it.
Lesson: "connection refused" right after kubeadm init is almost always the control plane itself crash-looping, not a networking problem — check `crictl ps -a` and the component logs before touching the network.

##☸️ Completing the cluster
Applied Flannel (the CNI plugin — it's what gives pods IP addresses and lets them talk to each other across nodes) once the control plane was actually stable.
Ran the kernel prerequisites on both workers (`br_netfilter`, IP forwarding) and joined them with `kubeadm join`.
All 3 nodes — k8s-control, k8s-worker1, k8s-worker2 — showed `Ready`.

##📦 Docker Hub
Built the image for `linux/arm64` (matches the Apple Silicon VMs) and pushed it as `oge1ata/demo-app:latest`.
Moved `deployment.yaml` into a `k8s/` folder and pointed its image field at the pushed image — ArgoCD watches this folder.

##🚀 Installing ArgoCD
ArgoCD is the tool that watches the GitHub repo and keeps the cluster in sync with it automatically — that's the "GitOps" part of the project.
Installed it into its own `argocd` namespace. Its CRDs (custom resource definitions) were too large for a normal `kubectl apply`, so used `--server-side` apply instead.
Instead of clicking through the ArgoCD UI to create the app, wrote it as a YAML file (`argocd/application.yaml`) committed to the repo, with automatic sync + self-heal turned on. Applied it once, and ArgoCD deployed the app itself.

##⚙️ GitHub Actions CI/CD
Added `.github/workflows/deploy.yml`. On every push to `master` it:
1. Builds the Docker image (cross-building for arm64 using QEMU, since GitHub's runners are normally a different chip architecture)
2. Pushes it to Docker Hub tagged with the git commit SHA
3. Updates `k8s/deployment.yaml` with that new tag
4. Commits that change back to the repo itself (using `[skip ci]` so it doesn't trigger itself again in a loop)
Needed two things set up on GitHub's side first: repo secrets for `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`, and "Read and write permissions" turned on for the workflow so it's allowed to push commits back.

##✅ Proved the whole loop works
Made a one-line change to `app.py`, pushed to `master`, and watched it flow through automatically:
GitHub Actions built + pushed the image → committed the updated `deployment.yaml` → ArgoCD picked up the new commit and redeployed → curled the app from inside the cluster and got the new response back.
No manual kubectl commands anywhere in that chain — that was the whole point.

Where I am in the overall journey
✅ App created and containerised
✅ Pushed to GitHub
✅ 3 VMs running with Kubernetes packages installed
✅ Control plane initialised (and the cgroup/kubeconfig bugs fixed)
✅ Workers joined — all 3 nodes Ready
✅ ArgoCD installed
✅ ArgoCD connected to the GitHub repo (declaratively, via YAML)
✅ Full GitOps pipeline working — push to master → build → deploy, automatically

##🔜 Possible next steps (optional, project already meets the goal)
- Rotate the ArgoCD admin password (still the auto-generated initial one)
- Write a README explaining the architecture for the portfolio
- Add resource requests/limits to deployment.yaml
- Swap the `kubectl port-forward` for a proper Ingress if I want the ArgoCD UI reachable without a manual command each time