# 02_kernel_os_hardening_pre_k3s.md

## Troubleshooting: IP Change and Static Assignment
During initial setup, I ran into an issue where the Arch Linux IP address changed. This broke SSH access from the Windows system because the Windows `hosts` file still pointed to the old IP. The solution was to assign static IPs to both machines at the router (DHCP reservation). This ensured both systems maintained consistent IP addresses across reboots.

## sysctl: Kernel Parameter Hardening
`sysctl` allows runtime modification of Linux kernel parameters and persistent configuration via `/etc/sysctl.d/*.conf`. I installed and configured security-focused sysctl settings to:
- Harden the network stack against spoofing and scans.
- Limit information leakage.
- Prepare for Kubernetes networking requirements.

### Steps:
1. Verify `procps-ng` installed:
```bash
pacman -Qi procps-ng
```
2. Create `/etc/sysctl.d/99-security.conf` with security and K3s-required settings:
```ini
# Ignore ICMP broadcast requests
net.ipv4.icmp_echo_ignore_broadcasts = 1
# Ignore bogus ICMP errors
net.ipv4.icmp_ignore_bogus_error_responses = 1
# Reverse path filtering (anti-spoof)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
# Disable source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
# Log suspicious packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
# SYN flood protection
net.ipv4.tcp_syncookies = 1
# K3s prerequisites
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.netfilter.nf_conntrack_max = 131072
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024
```
3. Apply settings:
```bash
sudo sysctl --system
```
4. Load required modules for K3s networking:
```bash
sudo modprobe br_netfilter overlay
```
Persist modules:
```bash
echo -e "br_netfilter\noverlay" | sudo tee /etc/modules-load.d/k8s.conf
```

## UFW: Firewall Configuration
`ufw` (Uncomplicated Firewall) manages `iptables` rules in a user-friendly way. Our goals:
- Default deny all incoming traffic.
- Allow only ICMP (ping) and SSH from the Windows laptop’s IP.
- Block all other inbound traffic.

### Steps:
1. Set default policy:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
2. Remove unsafe broad rules:
```bash
sudo ufw delete allow 22/tcp
sudo ufw delete allow from <10.*.*.*/24>
sudo ufw delete allow 22
```
3. Allow SSH only from Windows:
```bash
sudo ufw allow from <window IP> to any port 22 proto tcp
```
4. Allow ICMP only from Windows by editing `/etc/ufw/before.rules` before the `# End required lines` marker:
```
-A ufw-before-input -p icmp --icmp-type echo-request -s 10.0.0.142 -j ACCEPT
```
Reload UFW:
```bash
sudo ufw reload
```
5. Verify rules:
```bash
sudo ufw status verbose
```

With this setup, only your Windows machine can ping or SSH into Arch, and all other inbound traffic is blocked. This is a strong security baseline for moving into Kubernetes (K3s) installation.
## arch-audit: Vulnerability Awareness
`arch-audit` checks installed packages against Arch Security Team advisories.
- **Why now**: Before installing K3s, we verify that the base OS is not running packages with unpatched, high-risk vulnerabilities.
- **Install**:
```bash
sudo pacman -S arch-audit
```
- **Check unpatched vulnerabilities**:
```bash
arch-audit -u
```
- **Detailed JSON output**:
```bash
arch-audit -j
```
- **Update packages**:
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
