# 03_cis_benchmarking_k3s.md

## Goals
- Understand and apply the **CIS Kubernetes Benchmark** in the context of a single-node k3s cluster.  
- Install and configure **kube-bench** to validate k3s against CIS controls.  
- Tweak kube-bench configs to properly detect control plane and node components.  
- Apply **remediation steps** via k3s config changes to meet benchmark recommendations.  
- Document the process with before/after results as proof of hardening.

---

## What is the CIS Kubernetes Benchmark?
The **CIS (Center for Internet Security) Benchmark for Kubernetes** is a set of prescriptive, security-focused best practices for hardening Kubernetes clusters.  
It covers:

- Control plane components (API server, controller manager, scheduler).
- Worker node components (kubelet, proxy).
- etcd datastore (or equivalent in distros).
- Policies and configurations (RBAC, admission controllers, etc).

By running a CIS benchmark tool, such as [Aqua Security’s kube-bench](https://github.com/aquasecurity/kube-bench), we can check whether our cluster’s settings align with these security recommendations.  
According to the K3s CIS guideline, K3s has a number of security mitigations applied and turned on by default and will pass a number of the Kubernetes CIS controls without modification.

---

## Installing kube-bench Config (cfg/)
The kube-bench binary requires test files (`master.yaml`, `node.yaml`, etc.) and a dispatcher (`config.yaml`).  
I pulled these from the official kube-bench source tarball:

```bash
VER=v0.11.2  # known-good config release
curl -L "https://github.com/aquasecurity/kube-bench/archive/refs/tags/${VER}.tar.gz" -o /tmp/kb-src.tgz
tar -xzf /tmp/kb-src.tgz -C /tmp

# Copy config to system
sudo mkdir -p /etc/kube-bench/cfg
sudo cp /tmp/kube-bench-${VER#v}/cfg/config.yaml /etc/kube-bench/cfg/

# Copy the k3s CIS benchmark profile
sudo cp -r /tmp/kube-bench-${VER#v}/cfg/k3s-cis-1.8 /etc/kube-bench/cfg/
```

---

## Tweaking kube-bench Config for k3s

By default, `target_mapping` in `config.yaml` pointed only to labels (master, node), not actual files.  
This caused **Control Plane (Section 1.x)** tests to be skipped. I fixed this by explicitly pointing to the YAML files:

```yaml
# /etc/kube-bench/cfg/config.yaml
target_mapping:
  k3s-cis-1.8:
    master: k3s-cis-1.8/master.yaml
    node:   k3s-cis-1.8/node.yaml
    controlplane: k3s-cis-1.8/controlplane.yaml
    etcd:   k3s-cis-1.8/etcd.yaml
    policies: k3s-cis-1.8/policies.yaml
```

This ensured that when we run with `--benchmark k3s-cis-1.8`, kube-bench actually loads the control plane and node test files.

---

## Running kube-bench

```bash
sudo kube-bench   --config-dir=/etc/kube-bench/cfg   --config=/etc/kube-bench/cfg/config.yaml   --benchmark k3s-cis-1.8
```

![First result](./images/kube_first_result.jpg)

---

## Remediation steps

I followed most of the remediation steps listed here:  
https://docs.k3s.io/security/self-assessment-1.9  

Created a new config for the k3s binary with the below arguments:

```yaml
kube-apiserver-arg:
  - anonymous-auth=false
  - authorization-mode=Node,RBAC
  - admission-control-config-file=/var/lib/rancher/k3s/server/psa.yaml
  - enable-admission-plugins=NamespaceLifecycle,ServiceAccount,NodeRestriction,AlwaysPullImages,LimitRanger,ResourceQuota,PodSecurity
  - profiling=false
  - audit-log-path=/var/lib/rancher/k3s/server/logs/audit.log
  - audit-policy-file=/var/lib/rancher/k3s/server/audit.yaml
  - audit-log-maxage=30
  - audit-log-maxbackup=10
  - audit-log-maxsize=100
  - service-account-lookup=true
  - tls-cert-file=/var/lib/rancher/k3s/server/tls/serving-kube-apiserver.crt
  - tls-private-key-file=/var/lib/rancher/k3s/server/tls/serving-kube-apiserver.key
  - client-ca-file=/var/lib/rancher/k3s/server/tls/client-ca.crt
protect-kernel-defaults: true

kube-controller-manager-arg:
  - terminated-pod-gc-threshold=10

kubelet-arg:
  - anonymous-auth=false
  - tls-cert-file=/var/lib/rancher/k3s/agent/serving-kubelet.crt
  - tls-private-key-file=/var/lib/rancher/k3s/agent/serving-kubelet.key
  - client-ca-file=/var/lib/rancher/k3s/agent/client-ca.crt
  - streaming-connection-idle-timeout=5m
  - tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
```

---

## Final Result

I reran the kube-bench binary and results improved significantly.

![second result](./images/kube-bench-2.jpg)  
![second-2 result](./images/kube-bench-3.jpg)

---

## Stack
- **Arch Linux** — host OS.  
- **k3s** — lightweight Kubernetes distribution.  
- **aqua kube-bench** — CIS benchmark validation tool.  

---

## Achievements
- Successfully installed and configured **kube-bench** for k3s.  
- Fixed dispatcher (`target_mapping`) to enable full **Control Plane** and **Node** checks.  
- Integrated k3s-specific config paths to ensure kube-bench could locate kubeconfigs and certs.  
- Applied CIS remediation steps to harden API server, kubelet, and controller manager.  
- Produced before-and-after benchmark evidence to demonstrate hardening progress.  
