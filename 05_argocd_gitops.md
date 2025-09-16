# Argo CD in Practice — GitOps, Drift, and Self‑Healing (with K3s + Pod Security Notes)

> **TL;DR**: Argo CD isn’t just a Continuous Delivery (CD) tool for containers. It’s a GitOps engine that continuously compares *what’s running* to *what’s declared in Git*, then synchronizes, audits, rolls back, and even **self‑heals** when things drift. It treats your **application configuration like application source code**—reviewed via PRs, versioned, and reproducible.

---

## What Argo CD Is (and Why It’s More Than “CD”)
Argo CD is a Kubernetes controller plus UI/CLI/API that:
- **Implements GitOps**: the desired state of your cluster (manifests, Helm, Kustomize) lives in Git. Argo pulls (not pushes) and applies it.
- **Tracks**: watches live cluster resources and compares them to Git, surfacing diffs and health.
- **Monitors**: evaluates resource health (Deployments, StatefulSets, CRDs) and shows status in the UI/CLI.
- **Audits**: every sync is recorded; commit SHAs and sync history form a durable audit trail.
- **Revokes/rolls back drift**: with **self‑heal**, any out‑of‑band change is reconciled back to Git’s truth.
- **Self‑heals**: if something (or someone) mutates a resource, Argo reverts it to the last declared state.
- **Multi‑cluster**: can manage multiple clusters from one control plane, with RBAC and per‑project guardrails.

**Key idea:** Your app **configuration** is handled like **code**. You build branches, open PRs, get reviews, run CI checks, and merge. Argo then deploys *the merged commit*. No click‑ops in prod; no snowflake clusters.

---

## Architecture at a Glance
- **argocd-repo-server** fetches and renders manifests (Helm/Kustomize/plain YAML).
- **argocd-application-controller** compares desired vs. live and drives syncs.
- **argocd-server** exposes the UI and gRPC/REST API for the CLI and web.
- **Projects & RBAC** constrain what repos, clusters, and namespaces an app may touch.

---

## Prerequisites
- Kubernetes cluster (K3s in this project).
- `kubectl` access with admin privileges.
- (Optional but recommended) A Git repo for your app manifests.

> In K3s, the in-cluster API endpoint is `https://kubernetes.default.svc`. If you use external contexts, make sure server certs include your chosen IP/DNS via `tls-san` in `/etc/rancher/k3s/config.yaml`.

---

## Hands‑On Steps (What I Did)

### 1) Install Argo CD (plain manifest)
```bash
# Create the Argo namespace
kubectl create namespace argocd

# Install the official stable manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# (Optional) Watch pods come up
kubectl -n argocd get pods -w
```

### 2) Install the Argo CD CLI (`argocd`)
Pick your platform; examples:
```bash
# ArchLinux (pacman)
pacman -S argocd
```
![argocd](./images/argocd.jpg) 

### 3) Log into Argo CD
Expose the API/UI. For a lab, port‑forward is simplest:
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:80
# UI at http://localhost:8080
```

Grab the initial admin password and log in:
```bash
# Username: admin
# Password comes from a secret created at install time
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Login via CLI
argocd login <argocd-host-server> --username admin --password <above> --insecure
```

### 4) Rotate credentials: delete the bootstrap secret & set a new password
```bash
# Change the admin password
argocd account update-password

# Delete the initial secret so it can’t be reused
kubectl -n argocd delete secret argocd-initial-admin-secret
```

> Tip: For production, create named users or SSO; avoid using `admin` except for break‑glass.

---

## Deploy a Sample App (and the Security Gotcha I Hit)

### 5) First attempt: deploy **guestbook** via CLI
```bash
# (You can point to the official example or your fork. Example below assumes example repo.)
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook \
  --project default \
  --sync-policy automated --auto-prune --self-heal \
  --sync-option CreateNamespace=true
```

### 6) Policy issue: K3s Pod Security (PSA) blocked the Pod
My cluster enforces **Pod Security** (to help pass CIS benchmarks). The `guestbook` Deployment lacked the hardened settings required by the **restricted** profile, so the ReplicaSet was created, but **no Pod** was admitted (`ReplicaFailure=True`).

### 7) Resolution: relax **only** the `guestbook` namespace to **baseline**
Keep strong policies globally; only loosen for the demo namespace:
```bash
# Ensure the namespace exists (CreateNamespace already covers it; this is idempotent)
kubectl create namespace guestbook --dry-run=client -o yaml | kubectl apply -f -

# Remove any old restrictive labels
kubectl label ns guestbook \
  pod-security.kubernetes.io/enforce- \
  pod-security.kubernetes.io/audit- \
  pod-security.kubernetes.io/warn- || true

# Set 'baseline' to admit simple workloads
kubectl label ns guestbook \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline \
  --overwrite
```

> Why baseline? It’s still safer than “privileged,” but lenient enough for vanilla sample apps. In real apps, prefer keeping `restricted` and hardening manifests (non‑root, no‑priv‑esc, seccomp, drop caps, resource requests, etc.).

### 8) Clean up the failed attempt and re‑deploy cleanly
```bash
# Option A: let Argo reconcile after the namespace tweak
argocd app sync guestbook
argocd app wait guestbook --sync --health --timeout 180

# If you created resources by hand and want them gone first:
kubectl -n guestbook delete deploy guestbook-ui --ignore-not-found
kubectl -n guestbook delete svc guestbook-ui --ignore-not-found
# then re-sync
argocd app sync guestbook
```

### 9) Verify the rollout
```bash
kubectl -n guestbook get deploy,rs,pods,svc -o wide
kubectl -n guestbook get events --sort-by=.metadata.creationTimestamp | tail -n 25

# Quick smoke test
kubectl -n guestbook port-forward svc/guestbook-ui 8080:80
# visit http://localhost:8080
```
![argocd2](./images/argocd-2.jpg) 

---

### 10) Scale-Up to confirm GitOps
I edited the deployment manifest in the github to scale to 2 replicas.

![argocd3](./images/argocd-3.jpg)

---

## Why This Workflow Matters (GitOps Payoffs)
- **Change review**: All cluster changes happen via commits and PRs—peer review beats ad‑hoc kubectl.
- **Auditability**: Every deploy is tied to a Git SHA; Argo keeps sync history with who/what/when.
- **Rollbacks**: Revert the commit; Argo reconciles back automatically.
- **Drift control**: Turn on `--self-heal` to continuously snap live state to Git.
- **Reproducibility**: New clusters can be brought up by pointing Argo at the same repo.
- **Separation of duties**: App teams own repos; platform teams own projects, policies, and cluster access.

---

## Goals & Achievements

### Goals
- Stand up Argo CD using upstream manifests.
- Authenticate securely and rotate the bootstrap admin secret.
- Demonstrate GitOps with a sample application.
- Preserve cluster security posture (CIS/PSA) while enabling controlled exceptions.
- Document a clean path for namespace‑scoped policy relaxation vs. global disable.

### Achievements
- **Argo CD installed** and reachable via CLI/UI.
- **CLI installed** and authenticated (`argocd login`).
- **Initial password rotated** and bootstrap secret removed.
- **Guestbook deployed** via GitOps and made to pass security by relaxing **only** the `guestbook` namespace to **baseline**.
- **Redeployed cleanly** after cleaning old pods and syncing the App.
- Established a repeatable, auditable workflow for future apps.

---

## Stack

- **OS/Cluster**: Arch Linux host, **K3s** (Kubernetes lightweight distro)
- **GitOps**: **Argo CD** (controller, repo-server, API/UI)
- **Manifests**: Plain YAML / Kustomize / Helm (repo‑based)
- **Security**: Pod Security Admission (PSA) — cluster default; namespace exception = `baseline`
- **CLI**: `kubectl`, `argocd`
- **Optional**: Kyverno/Gatekeeper for policy as code, Argo Projects for multi‑tenant guardrails

---

## Next Steps (Optional but Recommended)
- Re‑tighten `guestbook` to **restricted** and harden manifests (set container `securityContext`, `seccompProfile`, resource requests/limits).
- Use **Argo Projects** to restrict which repos/clusters/namespaces each app may touch.
- Add **notifications** (Slack/Email) and **SSO**.
- Consider **App of Apps** pattern for layered, multi‑env deployments.
