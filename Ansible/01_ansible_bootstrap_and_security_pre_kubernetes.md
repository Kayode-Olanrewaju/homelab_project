# Ansible Bootstrap & Security Preparation for Kubernetes Home Lab
*Arch Linux + Windows (WSL) Integration — Pre-Kernal and OS-Hardening Phase*

---

## Overview
This document captures the early stages of setting up my Kubernetes home lab:
1. Pairing **Windows (WSL/Ubuntu)** and **Arch Linux** for remote automation.
2. Using **Ansible raw manifests** to bootstrap Python dependencies on Arch Linux.
3. Preparing for **system hardening** before Kubernetes installation.

The focus here is on **minimal Ansible use** (only raw manifests) and establishing a secure baseline for Kubernetes deployment.

---

## Architecture Diagram

```plaintext
+-------------------+             +------------------+
| Windows Laptop    |             | Arch Linux Host  |
| (WSL - Ubuntu)    |  <------>   | Bare Install     |
| Control Machine   |   SSH       | K3s Target Node  |
+-------------------+             +------------------+
```

---

## What is a "Raw Manifest" in Ansible?
In Ansible:
- **Raw tasks** are commands run on the target host **without using Python or Ansible modules**.
- Useful when the target system does not yet have Python installed.
- Often used for bootstrapping and pre-configuration.

Example:
```yaml
- name: Install Python on Arch Linux
  raw: pacman -Sy --noconfirm python3 python-pip python-six
```

---

## Environment Setup

### 1. Installing WSL/Ubuntu on Windows
```bash
wsl --version
lsb_release -a
```

### 2. Editing `/etc/hosts` on Both Systems
**WSL (control node):**
```
10.0.0.141   <agent's hostname>
```
**Arch Linux:**
```
10.0.0.102   <control's hostname>
```

### 3. Setting Up SSH Key Authentication
**From WSL:**
```bash
ssh-keygen -t ed25519 -C "arch-linux-key"
ssh-copy-id user@host
```
If `ssh-copy-id` is unavailable:
```bash
scp ~/.ssh/id_ed25519.pub user@host:~/.ssh/authorized_keys
```

### 4. Adjusting Windows Firewall for ICMP
- Enabled inbound **ICMPv4 Echo Requests** to allow `ping`.

---

## Bootstrap Playbook
**bootstrap.yml**
```yaml
- hosts: all
  gather_facts: no
  tasks:
    - name: Install Python requirements for Ansible on Arch
      raw: sudo pacman -Sy --noconfirm python3 python-pip python-six

    - name: Install six using pip3 (fallback)
      raw: pip3 install six
```

---

## Good Practices for GitHub
- Push:
  - `bootstrap.yml`
  - `host.ini` (placeholders only)
  - Documentation
- Avoid pushing:
  - SSH keys
  - Real IPs
  - Sensitive firewall configs

---

## Goals Achieved
✅ SSH pairing between WSL & Arch Linux  
✅ Hostname resolution via `/etc/hosts`  
✅ Python bootstrapped on Arch via Ansible raw tasks  
✅ Firewall adjusted for connectivity tests  

---

## Next Step: System Hardening Before Kubernetes


---

## Final Notes
This file is part of my **pre-Kubernetes setup phase**.  
Security steps are performed **before** introducing cluster components to reduce attack surface.
