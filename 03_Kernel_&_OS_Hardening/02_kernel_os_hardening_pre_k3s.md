# 02_kernel_os_hardening_pre_k3s.md

## Troubleshooting: IP Change and Static Assignment
We initially lost SSH access when the Arch Linux IP changed, breaking the Windows local DNS entry.  
**Solution**: reserved static IPs for both systems at the router (DHCP reservation), ensuring consistent addressing.

---

## sysctl: Kernel Parameter Hardening
`sysctl` is used to configure kernel parameters for network hardening and Kubernetes readiness.  
Security settings include ICMP handling, anti-spoofing, disabling source routing, blocking redirects, logging suspicious packets, and SYN flood protection.  
K3s prerequisites (IP forwarding, bridge networking, conntrack tuning, and inotify limits) were also configured.

---

## UFW: Firewall Configuration
`ufw` manages `iptables` rules with a simple syntax. Configured to:
- Deny all inbound traffic by default.
- Allow only SSH and ICMP from the Windows laptop IP via `/etc/ufw/before.rules`.
- Remove broad allow rules and restrict access.

---

## arch-audit: Vulnerability Awareness
`arch-audit` checks installed packages against Arch Security Team advisories.

**Why now**: Before installing K3s, we verify that the base OS is not running packages with unpatched, high-risk vulnerabilities.

**Install**:
```bash
sudo pacman -S arch-audit
```

**Check unpatched vulnerabilities**:
```bash
arch-audit -u
```

**Detailed JSON output**:
```bash
arch-audit -j
```

**Update packages**:
```bash
sudo pacman -Syu
```

### Vulnerability Review
Recent output flagged several packages. Using the Arch Security Tracker:
- **libxml2 (High, DoS)** — Fixed in `2.14.4-1`. Upgrade if below this version.
- **pam (High, Arbitrary Filesystem Access)** — Fixed in `1.7.1-1`. Upgrade if below this version.
- **Other packages** (coreutils, giflib, libtiff, openssl, perl, systemd, wget, linux) — Advisories present but often patched in current versions or require specific conditions to exploit.

**Action Workflow**:
1. Run `arch-audit -u`.
2. For any High severity packages, check `security.archlinux.org` to confirm if your version is patched.
3. If patched version available, `pacman -Syu` immediately.
4. If no patch yet, monitor tracker for updates.

---

## Trivy: Container and Filesystem Vulnerability Scanning
Trivy scans container images, filesystems, and repositories for vulnerabilities, misconfigurations, and secrets.

**Why now**: Having it ready before K3s means we can scan base images before deploying them.

**Install**:
```bash
sudo pacman -S trivy
```

**Usage**:
```bash
trivy image alpine:latest
trivy fs /
```

---

## Chrony: Time Synchronization
Chrony keeps system time accurate, which is crucial for Kubernetes cluster health, TLS certificates, and log integrity.

**Why now**: Time drift can cause authentication failures and scheduling issues in K3s.

**Install**:
```bash
sudo pacman -S chrony
```

**Enable and start**:
```bash
sudo systemctl enable --now chronyd
```

**Check status**:
```bash
chronyc tracking
```

---

With sysctl hardening, UFW restrictions, vulnerability scanning (arch-audit, trivy), and reliable time sync (chrony) in place, the host OS is fully prepared for secure K3s deployment.
