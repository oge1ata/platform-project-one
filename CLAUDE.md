# Project 2: Understanding What I Built
## Platform Engineering Portfolio — Gap-Filling Phase

---

## What this project is

Project 1 (GitOps pipeline with Kubernetes + ArgoCD + GitHub Actions) is complete but
a lot of it felt like following steps rather than genuinely understanding what was happening.

Project 2 is NOT a new tool or stack. It is about going back into the cluster I already
built, breaking things on purpose, watching what happens, and building real understanding
of WHY everything works the way it does.

Goal: by the end of this project I should be able to answer these three questions
without hesitation:
1. "Walk me through what happens from the moment you push code to Git to when it's
   running in the cluster."
2. "What does ArgoCD actually do, and how is it different from just running kubectl apply?"
3. "Why did you use GitHub Actions and not something else?"

---

## Environment (same cluster from Project 1)

| Thing | Detail |
|---|---|
| Machine | Mac Mini (Apple Silicon, aarch64) |
| OS | macOS |
| VMs | Multipass (lightweight Ubuntu VMs) |
| Kubernetes | v1.29.15, bootstrapped with kubeadm |
| Container runtime | containerd (SystemdCgroup = true) |
| Network plugin | Flannel (pod CIDR: 10.244.0.0/16) |
| Container registry | Docker Hub (oge1ata/demo-app) |
| GitOps tool | ArgoCD |
| CI/CD | GitHub Actions |
| IDE | VS Code with Claude Code |

## Cluster nodes

| Node | IP | Role |
|---|---|---|
| k8s-control | 192.168.252.3 | Control plane |
| k8s-worker1 | 192.168.252.2 | Worker |
| k8s-worker2 | 192.168.252.4 | Worker |

Shell into nodes:
```bash
multipass shell k8s-control
multipass shell k8s-worker1
multipass shell k8s-worker2
```

---

## How the pipeline works (for reference)

```
git push to master
        ↓
GitHub Actions triggers
        ↓
Builds Docker image (cross-compiled for linux/arm64 using QEMU)
        ↓
Pushes image to Docker Hub tagged with git commit SHA
        ↓
Updates k8s/deployment.yaml with new image tag
        ↓
Commits updated deployment.yaml back to repo ([skip ci])
        ↓
ArgoCD polls GitHub, detects new commit
        ↓
ArgoCD syncs cluster to match manifest — redeploys app
        ↓
New version live. No manual kubectl commands.
```

---

## Project 2 structure — three weeks, one theme each

### Week 1 — Kubernetes
**Theme:** Understand what Kubernetes is actually doing to keep my app running.

### Week 2 — ArgoCD
**Theme:** Understand what GitOps actually means in practice, not just in theory.

### Week 3 — GitHub Actions
**Theme:** Understand the CI/CD pipeline step by step and make it more robust.

---

## Week 1 Exercises — Kubernetes

The goal is not to read docs. The goal is to break things and watch what happens.
After each exercise, write one or two sentences in the learning log explaining what
you observed and why it happened.

### Exercise 1.1 — Pod scheduling
**What to do:**
```bash
# On k8s-control — scale the app to 3 replicas
kubectl scale deployment demo-app --replicas=3 -n default

# Watch where each pod lands
kubectl get pods -o wide -n default
```
**Question to answer:** Which nodes did the pods land on? Did Kubernetes spread them
evenly? Why might it do that?

### Exercise 1.2 — Self-healing
**What to do:**
```bash
# Get the name of one running pod
kubectl get pods -n default

# Delete it — kill it manually
kubectl delete pod <pod-name> -n default

# Watch immediately
kubectl get pods -n default -w
```
**Question to answer:** What happened after you deleted it? How fast did a new one
appear? What part of Kubernetes is responsible for that? (Hint: it's not ArgoCD.)

### Exercise 1.3 — Node failure simulation
**What to do:**
```bash
# From your Mac — stop a worker node
multipass stop k8s-worker1

# Watch what happens to the pods that were on it
kubectl get pods -o wide -n default -w
```
**Question to answer:** Did the pods move? How long did it take? What does this tell
you about how Kubernetes thinks about availability?

Don't forget to bring the node back:
```bash
multipass start k8s-worker1
```

### Exercise 1.4 — Resource limits in action
**What to do:**
Look at your current deployment.yaml. Find the resources section. It will look something like:
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "200m"
```
**Question to answer:** What is the difference between a request and a limit? What
happens if a pod uses more memory than its limit? What happens if it uses more than
its request but less than its limit?

You don't need to break this one — just read your own file and explain it back in
plain English in your learning log.

### Exercise 1.5 — Look inside a pod
**What to do:**
```bash
# Shell directly into your running app
kubectl exec -it <pod-name> -n default -- /bin/sh

# Once inside — look around
ls
ps aux
env
exit
```
**Question to answer:** What is running inside the container? What environment
variables exist? Does anything surprise you?

---

## Week 2 Exercises — ArgoCD

The goal is to understand what GitOps actually means by watching ArgoCD do its job
and by deliberately breaking the sync.

### Exercise 2.1 — Watch a real sync happen
**What to do:**
Make a small visible change to your app. Something simple like changing the message
in the root endpoint:
```python
return {"message": "hello from version 2"}
```
Push it. Then open the ArgoCD UI and watch the sync happen in real time.

ArgoCD UI is at: `http://192.168.252.3:<nodeport>`
(Check the service port: `kubectl get svc -n argocd`)

**Question to answer:** How long did ArgoCD take to notice the change after you pushed?
Why is there a delay? (Hint: ArgoCD polls rather than receiving a webhook by default.)

### Exercise 2.2 — Self-healing in action
**What to do:**
```bash
# Make a direct change to the cluster — bypass Git entirely
kubectl set image deployment/demo-app demo-app=nginx -n default

# Watch what ArgoCD does
kubectl get pods -n default -w
```
**Question to answer:** Did ArgoCD revert your change? How long did it take? What does
this prove about the difference between ArgoCD and just running kubectl apply yourself?

### Exercise 2.3 — Break the Git connection
**What to do:**
In the ArgoCD application.yaml, change the repoURL to a repo that doesn't exist.
Apply it with:
```bash
kubectl apply -f argocd/application.yaml -n argocd
```
Then look at the ArgoCD UI.

**Question to answer:** What status does ArgoCD show? What error message appears?
How do you fix it and bring it back to healthy?

### Exercise 2.4 — Understand every ArgoCD status
**What to do:**
In the ArgoCD UI, find where it shows the sync status and health status.

**Question to answer:** What do each of these statuses mean in plain English?
- Synced
- OutOfSync
- Healthy
- Degraded
- Progressing
- Unknown

Write a one-line plain English definition for each one in your learning log.

### Exercise 2.5 — Explain ArgoCD vs kubectl apply
**What to do:**
This is a writing exercise, not a terminal exercise.

In your learning log, write the answer to this question in plain English, as if
explaining to someone who has never heard of ArgoCD:

"What does ArgoCD actually do, and why would I use it instead of just running
kubectl apply every time I deploy?"

Your answer should be at least 3 sentences. No jargon allowed.

---

## Week 3 Exercises — GitHub Actions

The goal is to understand each step of the pipeline and make it catch real problems.

### Exercise 3.1 — Read the pipeline file out loud
**What to do:**
Open `.github/workflows/deploy.yml`. Read every line. For every step you don't
fully understand, write the step name in your learning log with a question mark
next to it.

Then bring those questions to Claude Code or to Claude.ai and get each one answered.

**Question to answer:** After going through it — which step was most confusing?
Which step would break the whole pipeline if it failed?

### Exercise 3.2 — Introduce a deliberate bug
**What to do:**
Break your FastAPI app on purpose. For example, add a syntax error:
```python
def health()
    return {"status": "ok"}
```
Push it. Watch what happens in GitHub Actions.

**Question to answer:** Did the pipeline catch the bug? Did it still deploy? If it
deployed a broken app, what does that tell you about what's missing from your pipeline?

### Exercise 3.3 — Add a basic test
**What to do:**
Create a test file:
```python
# test_app.py
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)

def test_health():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

Add a test step to your GitHub Actions pipeline before the build step:
```yaml
- name: Run tests
  run: |
    pip install fastapi uvicorn httpx pytest
    pytest test_app.py
```

Now introduce the bug from Exercise 3.2 again and push. Does the pipeline stop before
deploying?

**Question to answer:** Where in the pipeline does the failure happen now? Why is it
better to catch this before the build step rather than after?

### Exercise 3.4 — Understand QEMU
**What to do:**
Find the QEMU step in your pipeline. It will look something like:
```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v3
```

**Question to answer:** Why does this step exist? What would happen if you removed it
and tried to build on GitHub Actions without it? (Hint: think about the difference
between the CPU in your Mac Mini and the CPU in GitHub's runners.)

### Exercise 3.5 — Explain the full pipeline
**What to do:**
Writing exercise again. In your learning log, write a step-by-step explanation of
your entire pipeline as if describing it to someone in a job interview.

Start with "When I push code to the master branch..." and end with "...and the new
version is live in the cluster."

No bullet points. Write it as a story. It should take about a paragraph.

---

## Learning log instructions

After every exercise, add an entry to learning_log.md in this format:

```
## [Date] — [Exercise number and name]

**What I did:** [One sentence]
**What I observed:** [What actually happened]
**Why it happened:** [Your explanation in plain English]
**What surprised me:** [Anything unexpected]
```

This log becomes part of your portfolio and feeds your LinkedIn posts.

---

## Definition of Done for Project 2

Project 2 is complete when you can do all of the following without looking anything up:

- [ ] Explain what happens to pods when a node goes down
- [ ] Explain the difference between a resource request and a resource limit
- [ ] Explain what ArgoCD's self-healing actually does, with a real example
- [ ] Explain why ArgoCD detects changes with a delay (polling vs webhooks)
- [ ] Explain what QEMU does in your pipeline and why it's needed on Apple Silicon
- [ ] Walk through the entire pipeline end-to-end in plain English
- [ ] Answer "what does ArgoCD do differently from kubectl apply?" in three sentences

---

## Key commands cheat sheet

```bash
# Cluster overview
kubectl get nodes
kubectl get pods -o wide -n default
kubectl get pods -o wide -n argocd

# Watch things happen in real time
kubectl get pods -n default -w

# Scale replicas
kubectl scale deployment demo-app --replicas=3 -n default

# Shell into a pod
kubectl exec -it <pod-name> -n default -- /bin/sh

# Check ArgoCD app status
kubectl get application -n argocd

# Check ArgoCD UI port
kubectl get svc -n argocd

# Shell into nodes
multipass shell k8s-control
multipass shell k8s-worker1
multipass shell k8s-worker2

# Stop/start a node (for failure simulation)
multipass stop k8s-worker1
multipass start k8s-worker1
```

---

## How to work with Claude Code on this project

When you open this project in VS Code with Claude Code, this file gives Claude full
context. You can ask things like:

- "I just did exercise 1.2 and this happened — can you explain why?"
- "Help me write the learning log entry for today's exercise"
- "I don't understand why ArgoCD took 3 minutes to sync — explain it"
- "Draft a LinkedIn post based on what I learned today"

Claude Code will read this file automatically and have all the context it needs.