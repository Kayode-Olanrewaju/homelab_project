# Argo CD + Helm: Migrating Raw Manifests to a Pluggable Chart


## Goals
Template my app's configuration into **Helm packages** and let **Argo CD** handle deployments.

## Quick Game Plan
1. Make the Helm chart render what I already have (**names, selectors**).
2. Point the **Argo CD Application** at the chart (Git path or Helm repo/OCI).
3. **Dry‑run** the diff.
4. **Sync** with pruning, watching for immutables.

---

## What is Helm?
Yes, Helm is a **package manager for Kubernetes**, **and** a **templating engine** that makes app deployments **modular** and **pluggable**—similar to how **Terraform modules** let you reuse infrastructure blueprints. With Helm, a single chart can be reused across environments by swapping **values** instead of copy‑pasting YAML.

### Helm’s Two Personalities
- **Client-side tool** — Create charts, lint, render (`helm template`), package `.tgz`, push/pull from chart repos (GitHub Pages, OCI/GHCR, Harbor). **No cluster needed.**
- **Cluster-facing tool** — `helm install/upgrade` talks to Kubernetes via `kubeconfig`. In this setup, **Argo CD** does all applying, so Helm stays client-only.

> **My choice today:** Use Helm **client-side only**. Argo CD remains the deployer-of-record.

---

## Steps: Re-architect Plain YAML into a Helm Template (in Git)
### 1) Install Helm (Windows)
```powershell
choco install kubernetes-helm
```

### 2) Create and refactor into a chart in my Git repo
```bash
helm create guestbook
```
I updated the chart to keep names stable and selectors consistent so Argo can **adopt** existing objects.(replica counts changed to 4)

**`templates/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "guestbook.fullname" . }}
  labels:
    {{- include "guestbook.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  revisionHistoryLimit: 4
  selector:
    matchLabels:
      {{- include "guestbook.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "guestbook.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ coalesce .Values.service.targetPort .Values.service.port }}
              protocol: TCP
          env:
            {{- with .Values.env }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**`templates/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "guestbook.fullname" . }}
  labels:
    {{- include "guestbook.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "guestbook.selectorLabels" . | nindent 4 }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ coalesce .Values.service.targetPort .Values.service.port }}
      protocol: TCP
```

### 3) `_helpers.tpl` (what helpers are and why use them)
Helpers are small **reusable template functions**. Instead of sprinkling the same naming/label logic across many files, we **define once** and **include** everywhere. That’s safer than raw `{{ .Release.Name }}` because helpers keep names/selectors **consistent** (critical for Argo to adopt resources rather than recreate them).

**This chart’s helpers**
```gotemplate
{{- define "guestbook.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "guestbook.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}

{{- define "guestbook.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
app.kubernetes.io/name: {{ include "guestbook.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{- define "guestbook.selectorLabels" -}}
app.kubernetes.io/name: {{ include "guestbook.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

### 4) `values.schema.json` (what it does)
A JSON Schema that **validates `values.yaml`** during `helm lint`. It catches typos and bad types (e.g., `"eighty"` instead of `80`) before anything reaches the cluster.

```json
{{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "title": "guestbook values",
  "type": "object",
  "properties": {{
    "replicaCount": {{
      "type": "integer",
      "minimum": 1,
      "description": "Number of pod replicas for the Deployment."
    }},
    "image": {{
      "type": "object",
      "additionalProperties": false,
      "properties": {{
        "repository": {{ "type": "string", "minLength": 1 }},
        "tag": {{ "type": "string", "minLength": 1 }},
        "pullPolicy": {{ "type": "string", "enum": ["Always", "IfNotPresent", "Never"] }}
      }},
      "required": ["repository", "tag"]
    }},
    "service": {{
      "type": "object",
      "additionalProperties": false,
      "properties": {{
        "type": {{ "type": "string", "enum": ["ClusterIP", "NodePort", "LoadBalancer"] }},
        "port": {{ "type": "integer", "minimum": 2, "maximum": 65535 }},
        "targetPort": {{
          "type": ["integer", "null"],
          "minimum": 1,
          "maximum": 65535,
          "description": "When null, defaults to service.port."
        }}
      }},
      "required": ["type", "port"]
    }},
    "nameOverride": {{
      "type": "string",
      "maxLength": 63,
      "pattern": "^[a-z0-9]([-a-z0-9]*[a-z0-9])?$",
      "description": "Short name override (DNS-1123)."
    }},
    "fullnameOverride": {{
      "type": "string",
      "maxLength": 63,
      "pattern": "^[a-z0-9]([-a-z0-9]*[a-z0-9])?$",
      "description": "Full resource name override (DNS-1123)."
    }},
    "env": {{
      "type": "array",
      "description": "Container environment variables.",
      "items": {{
        "type": "object",
        "additionalProperties": false,
        "properties": {{
          "name": {{
            "type": "string",
            "pattern": "^[A-Za-z_][A-Za-z0-9_]*$"
          }},
          "value": {{ "type": "string" }}
        }},
        "required": ["name", "value"]
      }}
    }},
    "resources": {{
      "type": "object",
      "description": "Kubernetes resource requests/limits map."
    }}
  }},
  "required": ["replicaCount", "image", "service"],
  "additionalProperties": false
}}
```

### 5) `Chart.yaml`
```yaml
apiVersion: v2
name: guestbook
description: A clean, pluggable Helm chart for the Guestbook frontend
type: application
version: 0.1.0
appVersion: "1.0.0"
```

---

## Local “vanity checks” before pushing
### 3) Lint + schema validation
```bash
helm lint ./guestbook
```
**Explainer:** Validates chart structure and `values.yaml` **against `values.schema.json`**. Stops silly mistakes early.

![No error lint](./images/1_helm.jpg) 

### 4) Render with current values (no cluster needed)
```bash
helm template ./guestbook -f ./guestbook/values.yaml --namespace my-namespace
```
**Explainer:** Renders final Kubernetes YAML locally so you can compare to what’s running (names + selectors).

---

## Argo CD Commands (CLI-only)
### 5) Point the App to the chart path (Git)
```bash
argocd app set guestbook \
  --repo https://github.com/<owner>/<repo>.git \
  --path helm/guestbook \
  --revision main \
  --sync-option CreateNamespace=true \
  --sync-option ServerSideApply=true
```
**Explainer:** Repoints the *same* Application to the chart path on branch `main`, and enables namespace creation + server-side apply.

### 6) Dry-run the diff
```bash
argocd app diff guestbook
```
**Explainer:** Shows differences between Git desired state and live cluster. If nothing prints, it’s already in sync or you haven’t pushed changes.

### 7) Sync with pruning (and handle immutables)
```bash
argocd app sync guestbook --prune --replace
```
**Explainer:** Applies changes, **prunes** objects no longer in Git, and uses **replace** for immutable field changes (e.g., Service `clusterIP`, selector tweaks).

> **Adoption tip:** Set `fullnameOverride: "guestbook"` and keep selectors unchanged so Argo adopts old resources instead of creating duplicates.


### 8) Cluster is working fine

![Cluster responding as should ](./images/2_helm.jpg) 

![Resources running](./images/3_helm.jpg) 
---

## Stacks
- **OS/Cluster:** Arch Linux + K3s (UEFI, LUKS/LVM), hardened
- **GitOps:** Argo CD (CLI)
- **Templating/Packaging:** Helm (client-only), JSON Schema
- **SCM/Registry:** GitHub repo now; **OCI Helm repo (GHCR)** next
- **Security/Policy (roadmap):** Kyverno, kube-bench remediations

## Goals & Achievements (today)
- Converted raw YAML → reusable Helm chart
- Introduced helpers for stable naming/labels
- Added schema validation for safe values
- Repointed Argo CD to chart path; prepared for prune + replace cutover

## Roadmap: Switch from Git Path → Helm Repo (OCI)
As apps multiply, move to versioned chart releases:
```bash
# Package
helm package ./guestbook  # -> guestbook-0.1.0.tgz

# Push to GHCR (OCI)
echo $GITHUB_TOKEN | helm registry login ghcr.io -u <you> --password-stdin
helm push ./guestbook-0.1.0.tgz oci://ghcr.io/<you-or-org>

# Update Argo CD to use OCI source
argocd repo add oci://ghcr.io/<you-or-org> --name ghcr --enable-oci --username <you> --password <token>
argocd app set guestbook --repo oci://ghcr.io/<you-or-org> --helm-chart guestbook --revision 0.1.0
```

---

## Troubleshooting
- **No diff printed:** push changes, try `--hard-refresh`, or `--local` diff for uncommitted files.
- **Old pods linger:** they weren’t owned by this App; adopt via same names/selectors or delete once.
- **Lint fails on `nameOverride`:** allow `""`/`null` in schema or remove the key.
- **WSL/Windows paths:** run Helm where the files live (WSL vs Windows).

**Outcome:** App is modular & pluggable (Terraform‑like), Argo CD controls lifecycle, and we’re ready to scale via OCI Helm repos without YAML sprawl.




