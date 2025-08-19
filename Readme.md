# Hardened Arch Linux Kubernetes Control Plane Deployment

## Overview
This project builds a **production-inspired, security-hardened Kubernetes control plane** on Arch Linux, ready for scaling with additional worker nodes. The goal is to create a watertight system — from disk encryption and OS hardening to Kubernetes deployment, CIS benchmarking, GitOps, and observability.

---

## Phase 1 — OS Base & Disk Security
**Goal:** Establish a secure Arch Linux installation and Disk encryption.

1. **Install Arch Linux**.
   - Verify ISO integrity with GPG before flashing.
   - **LVM with dm-crypt (LUKS)** for full disk encryption.
   - Systemd-based initramfs hooks for encrypted boot.

2. **Removing Metadata from Project Images**
   - To protect privacy, all images in this project are cleaned using `exiftool` to strip GPS and device metadata.

---

## Phase 2 — Base Security Hardening
**Goal:** Secure kernel, system services, and network before installing Kubernetes.

1. **Firewall Configuration (UFW)**
   - Default deny inbound, allow outbound.
   - Allow SSH + Kubernetes control plane ports (22, 6443, etcd).

2. **Kernel & Network Hardening (sysctl)**
   - Harden sysctl for safe networking defaults and Kubernetes readiness.

3. **Vulnerability & Package Auditing**
   - **arch-audit** for system packages.
   - **trivy** for container images and filesystem scanning.

4. **Time Sync**
   - chrony for NTP (crucial for TLS, logs, cluster sync).

5. **Utilities**
   - Installed tcpdump, bind-tools, htop, sysstat.

---

## Phase 3 — Kubernetes Installation & Hardening
**Goal:** Deploy a minimal, secure Kubernetes control plane.

1. **Install K3s**
   - Lightweight Kubernetes distribution.
   - Disable unused components (traefik, servicelb if unneeded).
   - etcd data stored on encrypted disk.

2. **API Server Security**
   - `--authorization-mode=RBAC` enabled.
   - `--anonymous-auth=false` (no anonymous access).
   - TLS enabled for all API traffic.

---

## Phase 4 — Benchmarking & CIS Hardening
**Goal:** Validate Kubernetes security posture before layering automation.

1. **Run kube-bench** (by AquaSec)
   - Execute CIS Kubernetes Benchmark checks on control plane and nodes.
   - Identify misconfigurations in kubelet, API server, etc.

2. **Fix CIS Issues**
   - Harden systemd unit for K3s.
   - Adjust RBAC roles.
   - Apply secure file permissions.

**Reasoning:** Measuring and fixing baseline cluster security first ensures future automation (GitOps, policies) enforces *secure defaults*, not insecure ones.

---

## Phase 5 — Policy Enforcement with Kyverno
**Goal:** Guardrail workloads with admission policies.

1. **Deploy Kyverno**  
   - Base policies:  
     - Require signed images.  
     - Block privileged pods.  
     - Forbid `:latest` image tags.  
     - Enforce readOnlyRootFilesystem.  

2. **Ongoing Compliance**
   - Workload security guaranteed at admission time.

---

## Phase 6 — GitOps with Argo CD
**Goal:** Continuous, automated delivery with security baked in.

1. **Deploy Argo CD**
   - Git repo as source of truth.
   - Auto-sync manifests and Helm charts.
   - Drift detection: prevents insecure manual edits.

**Reasoning:** Applied after kube-bench fixes + Kyverno policies, so GitOps enforces hardened configs.

---

## Phase 7 — Observability & Monitoring
**Goal:** Ensure visibility into cluster health and security.

1. **Prometheus**
   - Scrape metrics from kubelets, cAdvisor, and K3s.

2. **Grafana**
   - Dashboards for cluster state, workloads, CIS compliance.  
   - Security-focused panels (Kyverno violations, kube-bench results).

3. **Audit Logging**
   - API server logs enabled for security auditing.

---

## Phase 8 — Networking & Access
**Goal:** Expose services securely.

1. **Ingress Controller**
   - NGINX ingress with restricted access.  
   - Client certificate auth or basic auth for dashboards.

2. **Secure Kubernetes Dashboard**
   - Restricted RBAC roles.  
   - Access via Ingress + authentication.

---

## Final Architecture Diagram
```
+-----------------------------+
| LUKS Encrypted Arch Linux   |
+-----------------------------+
| Hardened Kernel & OS        |
| - UFW, Sysctl               |
| - Trivy, arch-audit         |
| - chrony                    |
+-----------------------------+
| Kubernetes Control Plane    |
| - K3s (RBAC, TLS)           |
| - CIS hardening (kube-bench)|
+-----------------------------+
| Policy Management           |
| - Kyverno policies          |
+-----------------------------+
| GitOps Automation           |
| - ArgoCD (Git-driven state) |
+-----------------------------+
| Observability & Monitoring  |
| - Prometheus, Grafana       |
| - API Audit Logs            |
+-----------------------------+
| Secure Ingress & Dashboard  |
+-----------------------------+
```
