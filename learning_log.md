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

---

# Project 2 — Understanding What I Built

## 2026-08-30 — Exercise 1.1 (Pod scheduling) — but it turned into Exercise 2.2 first

**What I did:** Ran `kubectl scale deployment demo-app --replicas=3 -n default` directly against the cluster to see where the new pods would land.

**What I observed:** Two of the three pods went `Terminating` within about 20 seconds of being created — the deployment settled back down to 1 pod almost immediately, undoing my scale command.

**Why it happened:** I hadn't touched Git at all — `kubectl scale` only changes the *live* cluster, not the desired state recorded in `k8s/deployment.yaml` (which still said `replicas: 1`). ArgoCD continuously polls the repo and compares it against the live cluster; since ArgoCD's `syncPolicy.automated.selfHeal` is turned on, it treated my manual change as drift and reverted it back to match Git. First checked my guess that it was a resource limit problem — but the pods showed `Terminating`, not `Pending`, which was the tell that something *actively killed* them rather than Kubernetes failing to schedule them.

**What surprised me:** I ran head-first into Exercise 2.2 (self-healing) by accident while trying to do Exercise 1.1 (scheduling). Good reminder that in a real GitOps setup, `kubectl` changes made directly against the cluster don't stick — Git is the actual source of truth, not whatever's currently running.

## 2026-08-30 — Bonus lesson: pull before you push

**What I did:** After confirming the self-heal theory, edited `k8s/deployment.yaml` myself to set `replicas: 3`, committed, and tried to push.

**What I observed:** `git push` was rejected — "Updates were rejected because the remote contains work that you do not have locally." Diverged branches.

**Why it happened:** My last push before this was days ago. When that push landed, GitHub Actions ran and committed its own change back to `k8s/deployment.yaml` (updating the image tag) — a commit that only exists on GitHub, since I never pulled it down locally. So my new local commit was built on an outdated base. Ran `git pull --no-rebase` to merge both histories — since the CI commit only touched the `image:` line and mine only touched `replicas:`, Git merged them with zero conflicts.

**What surprised me:** The CI pipeline committing back to the repo isn't a one-time thing that only matters right after I set it up — it means my local clone can silently fall behind GitHub *every single time the pipeline runs*, not just when I personally push. Worth remembering to `git pull` before starting new work, not just before pushing.

## 2026-08-30 — Exercise 1.1 (Pod scheduling) — for real this time

**What I did:** Pushed the `replicas: 3` change to Git properly this time, then forced an ArgoCD refresh instead of waiting for its poll interval.

**What I observed:** All 3 pods came up and stayed `Running` (no more `Terminating`). Placement: 2 pods on `k8s-worker2`, 1 pod on `k8s-worker1` — not a perfectly even split.

**Why it happened:** This time the change came through Git, so ArgoCD saw it as the new desired state and applied it instead of reverting it. The uneven placement is the scheduler doing its normal job — it picks nodes based on available resources and a scoring algorithm, not a strict round-robin. With no anti-affinity rules or topology spread constraints set on this deployment, there's nothing forcing it to spread pods evenly across nodes — it's free to put more than one pod on the same node if that node scores best.

**What surprised me:** How much more convincing this is as a live demo than reading about it — watching the exact same command produce a "wait what" result the first time (self-heal reverted it) and a "yes that's expected" result the second time (Git-driven change stuck), back to back, on the same cluster.

## 2026-09-01 — Exercise 1.2 (Self-healing)

**What I did:** Deleted one running pod directly with `kubectl delete pod` and watched with `kubectl get pods -w` to see what happened next.

**What I observed:** A replacement pod started almost immediately — nowhere near the few-minutes delay ArgoCD would take.

**Why it happened:** This isn't ArgoCD at all. Deployments actually work through a small hierarchy: **Deployment → ReplicaSet → Pods**. The ReplicaSet is the object that holds "there should be exactly N pods matching this spec," and a dedicated Kubernetes controller (the ReplicaSet controller, running inside kube-controller-manager) watches that continuously and creates a replacement the instant the count drops — completely independent of Git or ArgoCD. The speed was the actual evidence: ArgoCD polls every few minutes, this was near-instant, so it had to be a different, faster, always-on mechanism.

**What surprised me:** There are two separate self-healing systems stacked on top of each other in this setup, operating at completely different layers and speeds — the ReplicaSet controller keeps the *pod count* correct in real time, while ArgoCD keeps the *whole cluster* matching Git every few minutes. Exercise 1.1's accidental detour and this exercise are really the same lesson from two different angles: something is always watching, but which "something" depends on what changed.

## 2026-09-01 — Exercise 1.3 (Node failure simulation)

**What I did:** Ran `multipass stop k8s-worker1` to kill an entire worker VM while 3 app pods were running, then watched `kubectl get nodes` and `kubectl get pods -o wide -n default` at the same time.

**What I observed:** Two separate, very different timers. `k8s-worker1` flipped from `Ready` to `NotReady` in under a minute. But the pod that had been running on it (`5hmfv`) kept showing `Running` for several more minutes after that — the AGE just kept ticking — before it finally flipped to `Terminating`, and only then did a brand new pod get created (`Pending` → `ContainerCreating` → `Running`) on `k8s-worker2`. The old pod stayed stuck in `Terminating` limbo even after that. Once I ran `multipass start k8s-worker1` and it came back, the stuck pod cleared on its own with no manual force-delete needed — but none of the 3 running pods moved back to it. All 3 stayed piled up on `k8s-worker2`, leaving `k8s-worker1` completely empty despite being healthy again.

**Why it happened:** Node failure detection is a two-stage process, not one event. Stage 1: the control plane marks a node `NotReady` once it stops hearing heartbeats — fast, tens of seconds. Stage 2: it waits a separate, much longer grace period (default 5 minutes) before actually evicting the pods that were on it. That gap is deliberate — a `NotReady` node might just be a brief network blip that resolves itself, and rescheduling pods instantly on every blip would cause unnecessary churn across a real cluster. The pod staying stuck in `Terminating` makes sense too: the API server can't confirm a pod is actually gone until the kubelet running on that pod's node acknowledges it — and that kubelet was unreachable, because the whole VM was down. And the fact that nothing moved back to `k8s-worker1` once it recovered is because Kubernetes only makes placement decisions when a pod is *created* — it never rebalances pods that are already running just because a better node becomes available. To prove this, deleted one of the pods piled on `k8s-worker2` manually, and its replacement landed on `k8s-worker1` — because deleting it forced a brand new scheduling decision, and with `k8s-worker1` empty, the scheduler's default spread-across-nodes preference kicked in.

**What surprised me:** That recovery isn't symmetric with failure. A node going down and a node coming back sound like they should be mirror-image events, but they're not — going down triggers an active, timed response (detect → wait → evict → reschedule), while coming back triggers almost nothing on its own. The cluster just quietly accepts the node is available again and waits for some future scheduling decision to actually use it. If I hadn't forced that by deleting a pod, `k8s-worker1` could have sat empty indefinitely.

## 2026-09-22 — Unplanned lesson: kernel settings don't survive a reboot

**What I did:** Came back to the cluster after about 3 weeks away and found 3 app pods stuck — two `Unknown`, one stuck in `ContainerCreating` for 21 days straight.

**What I observed:** Flannel's pods on both worker nodes were in `CrashLoopBackOff` with thousands of restarts. Their logs showed `Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables: no such file or directory`. The `br_netfilter` kernel module was missing on both workers (confirmed with `lsmod`), even though it was present on `k8s-control`.

**Why it happened:** Back in Project 1 setup, I loaded `br_netfilter` with `sudo modprobe br_netfilter` and set the related settings with `echo 1 | sudo tee /proc/sys/net/...`. Neither of those is persistent — they only last until the next reboot. Sometime in the 3 weeks since we last touched this, the worker VMs restarted (most likely the Mac Mini sleeping or restarting), the module unloaded, and Flannel couldn't function without it. That cascaded into every app pod on the affected nodes failing to get network sandboxes at all. Fixed it properly this time: reloaded the module, but also added it to `/etc/modules-load.d/k8s.conf` (loads automatically on every boot from now on) and moved the sysctl settings into `/etc/sysctl.d/k8s.conf` instead of the one-off `/proc` writes.

**What surprised me:** This wasn't caused by anything I did in Project 2 — it was a latent problem from Project 1's setup that had been sitting there the whole time, waiting for the first reboot to expose it. A "one-time setup command" and a "permanent setting" look identical the moment you type them, but behave completely differently the next time the machine restarts. Also a good reminder that "it worked when I set it up" and "it's actually configured correctly" aren't the same claim.