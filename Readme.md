# Hardened Arch Linux Kubernetes Control Plane Deployment

## Overview
This project builds a **production-inspired, security-hardened Kubernetes control plane** on Arch Linux, ready for scaling with additional worker nodes. The goal is to create a watertight system — from disk encryption and OS hardening to Kubernetes deployment, GitOps, and CIS compliance testing.

---

## Phase 1 — OS Base & Disk Security
**Goal:** Establish a secure, encrypted Arch Linux installation.

1. **Install Arch Linux** with:
   - **LVM with dm-crypt (LUKS)** for full disk encryption.
   - Verified ISO integrity with GPG before flashing.
   - Systemd-based initramfs hooks for encrypted boot:
     ```bash
     HOOKS=(base systemd autodetect modconf block sd-encrypt sd-vconsole filesystems fsck)
     ```
2. **Stretch Goal:** Configure **remote LUKS unlock** via:
   - Dropbear SSH in initramfs, or
   - Clevis + Tang for automated decryption.

---

## Phase 2 — Base Security Hardening
**Goal:** Secure kernel, system services, and network before installing Kubernetes.

1. **Firewall Configuration**
   - **UFW** (Uncomplicated Firewall):
     - Default deny inbound, allow outbound.
     - Allow SSH port (22) and future Kubernetes control plane ports (6443, etcd, etc.).
   
2. **Intrusion Prevention**
   - **Fail2Ban** to ban IPs with repeated SSH login failures.

3. **System Integrity & Malware Detection**
   - **rkhunter** — Detect rootkits and anomalies.
   - **chkrootkit** — Rootkit detection.
   - **clamav** — Antivirus scanning.

4. **System Auditing**
   - **auditd** — Kernel-level auditing for events.
   - **Arch-audit** — Detect vulnerable packages.

5. **Time Synchronization**
   - **chrony** for accurate NTP time (important for TLS, logs, cluster sync).

---

## Phase 3 — Kubernetes Installation & Hardening
**Goal:** Deploy a minimal, secure Kubernetes control plane.

1. **Install K3s** (lightweight Kubernetes):
   - Disable unused components (traefik, servicelb if using custom ingress).
   - Store etcd data on encrypted disk.

2. **API Server Security**
   - Enable `--authorization-mode=RBAC`.
   - Disable anonymous auth (`--anonymous-auth=false`).
   - Use TLS for all API traffic.

3. **Pod Security**
   - Deploy **Kyverno** for policy-as-code.
   - Enforce non-root containers, read-only filesystem, resource limits.

---

## Phase 4 — GitOps & Policy Management
**Goal:** Automate cluster configuration and enforce compliance.

1. **Deploy Argo CD** for GitOps:
   - Store manifests and Helm charts in a private Git repo.
   - Auto-sync cluster state with repo.

2. **Optional:** Use **FluxCD** instead of ArgoCD.

3. **Advanced Policy Enforcement**
   - Deploy **OPA Gatekeeper** for fine-grained Kubernetes policy control.

---

## Phase 5 — Observability & Benchmarking
**Goal:** Validate security posture and ensure CIS compliance.

1. **kube-bench** (by AquaSec) to run CIS Kubernetes Benchmark checks.

2. **Logging & Monitoring**
   - Enable audit logging in Kubernetes API server.
   - Install Prometheus + Grafana for metrics.

---

## Phase 6 — Networking & Access
**Goal:** Expose secure ingress and dashboard.

1. **Ingress Controller**
   - NGINX Ingress Controller.
   - Restrict access via:
     - Basic auth, or
     - Client certificate authentication.

2. **Secure Kubernetes Dashboard**
   - Deploy with restricted RBAC permissions.
   - Access via Ingress with authentication.

---

## Final Architecture Diagram
```
+---------------------------+
|  LUKS Encrypted Arch Linux|
+---------------------------+
| Hardened Kernel & OS      |
| - UFW, Fail2Ban           |
| - rkhunter, chkrootkit    |
| - clamav, auditd, chrony  |
+---------------------------+
| Kubernetes Control Plane  |
| - K3s (RBAC, TLS)         |
| - Kyverno Policies        |
| - OPA Gatekeeper          |
+---------------------------+
| GitOps Automation         |
| - ArgoCD / FluxCD         |
+---------------------------+
| Observability & Security  |
| - kube-bench              |
| - Prometheus, Grafana     |
+---------------------------+
| Ingress & Access Security |
+---------------------------+
```
