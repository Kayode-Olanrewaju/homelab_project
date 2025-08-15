# K3s Installation & DNS Troubleshooting

## 1. K3s Installation
We installed K3s with the default Flannel CNI, disabling unnecessary components only when needed for our GitOps setup.  
The command used:
```bash
curl -sfL https://get.k3s.io | sh -
```
Later adjustments may be made to disable the Helm controller if desired.

---

## 2. DNS Resolution Issue

### Symptoms
After K3s installation, pods were stuck in `ContainerCreating` state.  
`kubectl describe pod` showed repeated errors pulling the `rancher/mirrored-pause:3.6` image, indicating DNS resolution failures.

Running:
```bash
nslookup registry-1.docker.io
```
returned:
```
;; communications error to ::1#53: connection refused
;; communications error to 127.0.0.1#53: connection refused
;; no servers could be reached
```

---

### Root Cause
The system's `/etc/resolv.conf` was pointing DNS queries to:
```
nameserver 127.0.0.1
nameserver ::1
```
But **no DNS service** (`systemd-resolved`, `dnsmasq`, `unbound`, etc.) was listening on those addresses.  
This meant:
- IP-based networking worked fine (`ping 8.8.8.8` was successful)
- Any hostname lookup failed instantly, breaking image pulls and pod startups.

This is a common situation on **minimal OS installs** like Arch Linux, Alpine, or container-focused distributions. They often assume you'll manually configure DNS or rely on DHCP to supply a valid `/etc/resolv.conf`.

---

### Resolution
We enabled `systemd-resolved` and linked its managed resolver configuration:
```bash
sudo systemctl enable --now systemd-resolved
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```
After this change:
- Host DNS resolution worked.
- Pods were able to pull required images.
- K3s cluster became fully operational.

---

### Lessons Learned
For Kubernetes hosts, DNS must be functional **before** installing K3s.  
If `/etc/resolv.conf` points to `127.0.0.1` or `::1`, ensure a DNS service is listening there or replace it with valid upstream DNS servers.

