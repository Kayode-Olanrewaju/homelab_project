# 02_kernel_os_hardening_pre_k3s.md

## Troubleshooting: IP Change and Static Assignment

During the initial setup, I established SSH connectivity from my Windows laptop into the Arch Linux system so I could manage it remotely from day one. This worked well until the Arch Linux machine’s IP address changed, which immediately broke SSH access—Windows was still pointing to the old address in its `hosts` file. The fix was to configure DHCP reservations on the router, assigning static IPs to both the Windows and Arch systems. With that in place, both machines now maintain consistent IP addresses across reboots, ensuring stable remote access.

Around the same time, I experimented with setting up an Ansible connection from Windows to Arch Linux, laying the groundwork for automated provisioning. That process is fully documented in the separate ansible/ folder.

##                                           sysctl: Kernel Parameter Hardening
`sysctl` is like system's control panel for Linux kernel's brain.It's a tool that let's you view and change kernel parameters on the fly,without rebooting.

###  How it sysctl works
- Every tunable kernel parameter is represented as a file in /proc/sys/.
- sysctl reads or writes values to these files.
- Changes made with sysctl -w apply instantly but are temporary (reset after reboot).
- Permanent changes are stored in /etc/sysctl.conf or in custom files under /etc/sysctl.d/, which get applied at boot by systemd-sysctl.

### Why it matters in Kubernetes / container world
Containers share the host’s kernel. If the kernel allows risky behaviors, a compromised container can exploit them.
For example:
- ICMP redirects can allow man-in-the-middle attacks on the host network.
- Unprivileged user namespaces can be abused to escalate privileges from inside a container
- Lack of SYN flood protection can allow cheap DoS attacks.
By hardening the kernel with sysctl, we shrink the attack surface before Kubernetes ever runs a pod.

I installed and configured security-focused sysctl settings to:
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
# ================================
# Kubernetes Host Hardening (sysctl)
# ================================
# This config locks down kernel and network behavior while keeping
# compatibility across CNIs (VXLAN overlays, BGP, eBPF datapaths).

# --- ICMP & Routing Protections ---
# Ignore broadcast ICMP (smurf attack prevention)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Enable reverse path filtering (anti-spoof)
# Mode 2 = Loose (safer for CNI plugins: VXLAN, eBPF, BGP)
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2

# Disable IP source routing (don’t allow attackers to specify routes)
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Disable ICMP redirects (prevent man-in-the-middle tricks)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# --- Logging & DoS Protections ---
# Log suspicious packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Enable SYN flood protection
net.ipv4.tcp_syncookies = 1

# --- Kubernetes Networking Prereqs ---
# Allow IP forwarding (pods need routing between nodes)
net.ipv4.ip_forward = 1

# Ensure bridged traffic passes through iptables (required by CNI plugins)
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# Increase conntrack table size (handle high pod/service traffic)
net.netfilter.nf_conntrack_max = 131072

# --- File System & Inotify ---
# Increase inotify limits (Kubernetes watches configs/logs heavily)
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024

# --- Kernel Hardening ---
# Restrict kernel logs to root (don’t leak info to unprivileged users)
kernel.dmesg_restrict = 1

# Protect against symlink/hardlink attacks
fs.protected_symlinks = 1
fs.protected_hardlinks = 1

# Disable unprivileged user namespaces (common container escape vector)      
# Modern container threats: Recent kernel exploits focus on info leak        
# (dmesg), namespace abuse, and filesystem tricks. Adding these kernel flags 
# provides high-value, low-cost defenses
kernel.unprivileged_userns_clone = 0
```
### This a CNI-agnostic sysctl config due to its loose rp_filter setting. I chose this because I may switch between a VXLAN, BGP, or eBPF network plugin.

3. Apply & Verify settings:
```bash
sudo sysctl --system
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.ip_forward
sysctl kernel.unprivileged_userns_clone
```
4. Expected:
```ini
net.ipv4.conf.all.rp_filter = 2
net.ipv4.ip_forward = 1
kernel.unprivileged_userns_clone = 0
```
5. Load required modules for K3s networking:
```bash
sudo modprobe br_netfilter overlay
```
Persist modules:
```bash
echo -e "br_netfilter\noverlay" | sudo tee /etc/modules-load.d/k8s.conf
```
##                                         UFW: Host-Level Firewall (North–South Traffic)

### What UFW Is
`ufw` (Uncomplicated Firewall) is a simple interface to Linux’s packet filter (iptables/nftables).  
- It manages **north–south traffic**: packets entering or leaving the host.  
- It does **not** control **east–west pod-to-pod traffic** inside the cluster — that’s handled by the CNI (Flannel, Calico, Cilium, etc.) and Kubernetes NetworkPolicies.  

Think of UFW as your **perimeter guard**: it decides which traffic from your LAN or the internet is allowed to reach the host. Once packets are inside the cluster overlay, UFW steps aside and lets the CNI handle routing.

---

### Why Use UFW Here
- Restrict access to the host: only trusted machines (like your Windows laptop) can SSH in.  
- Block unnecessary inbound traffic: IoT devices, guests on Wi-Fi, or scans from the internet won’t reach your node.  
- Future-proofing: as you add worker nodes or deploy an Ingress controller, you’ll extend UFW to open only the ports needed by Kubernetes control-plane, kubelet, or Ingress pods.  

---

### Installation and Base Rules (Single-Node Cluster)
1. Installation on Arch Linux:
```bash
sudo pacman -S ufw --needed
sudo systemctl enable --now ufw
Default policy: deny all inbound, allow all outbound.
```

2. Set default policy:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
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
### Extending UFW Later
#### Multi-node Cluster
When adding worker nodes:  
- **kube-apiserver (6443/tcp)** → allow from worker node IPs.  
- **kubelet API (10250/tcp)** → allow from the control-plane node.  
- **CNI overlay ports** (depends on plugin):  
  - Flannel (VXLAN): `8472/udp`  
  - Calico (BGP): `179/tcp`  
  - Cilium (eBPF): usually none beyond 6443  

Example (Flannel multi-node):
```bash
sudo ufw allow from <WORKER_NODE_IP> to any port 6443 proto tcp
sudo ufw allow from <CONTROL_PLANE_IP> to any port 10250 proto tcp
sudo ufw allow 8472/udp
```
#### Ingress (When ingress controller is deployed)
```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```
With this setup, only your Windows machine can ping or SSH into Arch, and all other inbound traffic is blocked. This is a strong security baseline for moving into Kubernetes (K3s) installation.

##                               Arch-Audit: Vulnerability Awareness
### What Arch-audit Is
`arch-audit` is a command-line tool that checks your installed Arch Linux packages against the Arch Security Team’s vulnerability advisories. It tells you if any package on your system is affected by known CVEs (Common Vulnerabilities and Exposures).

### How It Works
- It queries the [Arch Linux Security Tracker](https://security.archlinux.org/) database.  
- Compares installed package versions to known vulnerable versions.  
- Reports advisories by severity (Low, Medium, High, Critical).  
- Outputs results in human-readable text or machine-readable JSON.

### Why It Matters in a Kubernetes World
- Kubernetes nodes run lots of userland utilities (systemd, openssl, container runtimes). A single vulnerable package can compromise the host and all pods running on it.  
- By auditing regularly, you **catch risks before they get exploited**.  
- This is especially critical for a home lab that may run internet-facing workloads through an Ingress controller.

---
### Installation
```bash
sudo pacman -S arch-audit --needed
```
### Usage
```bash
arch-audit -u
```
### Update all packages
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


##                 Trivy: Container and Filesystem Vulnerability Scanning
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

## Security Hardening Stack Overview
- sysctl — tunes kernel parameters for network and process hardening.
- ufw — firewall to restrict inbound/outbound traffic.
- arch-audit — checks installed packages against known CVEs.
- trivy — scans container images/filesystems for vulnerabilities & secrets.
- chronyd — keeps system time accurate for TLS, logging, and cluster sync.
