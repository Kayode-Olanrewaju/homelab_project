# Hardened Arch Linux Kubernetes Control Plane Deployment

## Overview
This project builds a **production-inspired, security-hardened Kubernetes control plane** on Arch Linux, ready for scaling with additional worker nodes. The goal is to create a watertight system — from disk encryption and OS hardening to Kubernetes deployment, GitOps, and CIS compliance testing.

---

## Phase 1 — OS Base & Disk Security
**Goal:** Establish a secure, encrypted Arch Linux installation.

1. **Install Arch Linux** with:
   - Verify ISO integrity with GPG before flashing.
   - **LVM with dm-crypt (LUKS)** for full disk encryption.
   - Systemd-based initramfs hooks for encrypted boot:


2. ## Removing Metadata from Project Images
To protect privacy and avoid leaking location or device information embedded in photos, all images included in this project are cleaned using `exiftool`.

---

## Phase 2 — Base Security Hardening (Container-Ready Edition)
**Goal:** Secure kernel, system services, and network before installing Kubernetes.

1. **Firewall Configuration**
   - **UFW** (Uncomplicated Firewall):
     - Default deny inbound, allow outbound.
     - SSH + ICMP allowed only from Windows IP
     - Allow SSH port (22) and future Kubernetes control plane ports (6443, etcd, etc.).
2. **Kernel & Network Hardening — sysctl**
   - Applied security-focused sysctl settings for network stack hardening and Kubernetes requirements.
3. **Vulnerability & Package Auditing**
   - Installed **arch-audit** for package vulnerability checks.
   - Installed **trivy** for container image and filesystem scanning.

4. **Time Synchronization**
   - **chrony** for accurate NTP time (important for TLS, logs, cluster sync).
5. **System Utilities for Ops & Troubleshooting**
   - Installed tools tcpdump bind-tools htop sysstat


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
---

## Phase 4 — GitOps & Kyverno for Policy Management
**Goal:** Automate cluster configuration and enforce compliance.

1. **Deploy Argo CD** for GitOps:
   - Store manifests and Helm charts in a Git repo.
   - Auto-sync cluster state with repo.

**Optional:** Use **FluxCD** instead of ArgoCD.

2. **Advanced Policy Enforcement**
   - Deploy **Kyverno** + base security policies

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
| - UFW,Sysctl         |
| - Trivy,arch-audit    |
| - chrony  |
+---------------------------+
| Kubernetes Control Plane  |
| - K3s (RBAC, TLS)         |
| -         |
          |
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
