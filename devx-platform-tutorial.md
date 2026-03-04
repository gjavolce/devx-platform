# DevX Platform Tutorial — Building a Kubernetes-Native Internal Developer Platform

A hands-on, chapter-by-chapter guide to building a complete internal developer platform on Kubernetes. Each chapter introduces one component, builds on the previous chapters, and ends with verification steps and a git commit.

**What you'll build:** A local Kubernetes platform running Backstage (developer portal), ArgoCD (GitOps), Crossplane (infrastructure provisioning), Prometheus + Grafana (observability) — all managed as Infrastructure as Code with Terraform, packaged with Helm, and portable between macOS and GitHub Codespaces.

**Prerequisites:** A GitHub account with Codespaces access (or Docker Desktop on macOS).

---

## Chapter 1 — DevContainer & Tooling Setup

**Goal:** Create a reproducible development environment that works identically in GitHub Codespaces and local VS Code DevContainers. Every tool needed for the entire tutorial is installed here.

**Components introduced:** DevContainers, Docker-in-Docker, k3d, kubectl, Helm, Terraform, k9s, yq, ArgoCD CLI.

### 1.1 Create the repository

Create a new GitHub repository called `devx-platform` and clone it locally (or open it in Codespaces). Then create the initial structure:

```bash
mkdir -p .devcontainer scripts terraform/environments/{local,codespaces,cloud} \
  terraform/modules/{k3d-cluster,platform-base,backstage,crossplane,argocd,observability} \
  helm/charts/devx-app/templates
```

### 1.2 DevContainer Dockerfile

Create `.devcontainer/Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

# Avoid prompts during package install
ENV DEBIAN_FRONTEND=noninteractive

# k3d — lightweight k3s-in-Docker wrapper
RUN curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Terraform — infrastructure as code
RUN curl -fsSL https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
       > /etc/apt/sources.list.d/hashicorp.list \
    && apt-get update && apt-get install -y terraform \
    && rm -rf /var/lib/apt/lists/*

# k9s — Kubernetes TUI for cluster exploration
RUN K9S_VERSION=$(curl -s https://api.github.com/repos/derailed/k9s/releases/latest | grep tag_name | cut -d '"' -f 4) \
    && curl -sL "https://github.com/derailed/k9s/releases/download/${K9S_VERSION}/k9s_Linux_amd64.tar.gz" \
       | tar xz -C /usr/local/bin k9s

# yq — YAML processor (like jq for YAML)
RUN YQ_VERSION=$(curl -s https://api.github.com/repos/mikefarah/yq/releases/latest | grep tag_name | cut -d '"' -f 4) \
    && curl -sL "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64" \
       -o /usr/local/bin/yq && chmod +x /usr/local/bin/yq

# ArgoCD CLI — interact with ArgoCD from the command line
RUN ARGOCD_VERSION=$(curl -s https://api.github.com/repos/argoproj/argo-cd/releases/latest | grep tag_name | cut -d '"' -f 4) \
    && curl -sL "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64" \
       -o /usr/local/bin/argocd && chmod +x /usr/local/bin/argocd
```

### 1.3 DevContainer configuration

Create `.devcontainer/devcontainer.json`:

```json
{
  "name": "DevX Platform",
  "build": {
    "dockerfile": "Dockerfile"
  },
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {
      "kubectl": "latest",
      "helm": "latest",
      "minikube": "none"
    }
  },
  "forwardPorts": [80, 443, 3000, 7007, 8080, 9090],
  "portsAttributes": {
    "80":   { "label": "HTTP Ingress" },
    "443":  { "label": "HTTPS Ingress" },
    "3000": { "label": "Grafana" },
    "7007": { "label": "Backstage" },
    "8080": { "label": "ArgoCD" },
    "9090": { "label": "Prometheus" }
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "hashicorp.terraform",
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "redhat.vscode-yaml",
        "tim-koehler.helm-intellisense"
      ]
    }
  },
  "remoteUser": "vscode"
}
```

### 1.4 Repository boilerplate

Create `.gitignore`:

```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfplan
.terraform.lock.hcl

# IDE
.idea/
*.iml
.vscode/settings.json
.project
.classpath
.settings/

# OS
.DS_Store
Thumbs.db

# Logs
*.log

# Environment
.env
.env.local
```

Create `.editorconfig`:

```editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.{tf,hcl}]
indent_style = space
indent_size = 2

[*.{yaml,yml,json}]
indent_style = space
indent_size = 2

[*.{sh,bash}]
indent_style = space
indent_size = 2

[Makefile]
indent_style = tab

[*.md]
trim_trailing_whitespace = false
```

Create `README.md`:

```markdown
# DevX Platform

A local-first Kubernetes platform for developer experience applications.

## What's Inside

| Component   | Purpose                          |
|-------------|----------------------------------|
| k3d         | Lightweight Kubernetes cluster   |
| Terraform   | Infrastructure as Code           |
| Helm        | Kubernetes package management    |
| Crossplane  | Infrastructure provisioning      |
| ArgoCD      | GitOps continuous delivery       |
| Backstage   | Developer portal                 |
| Prometheus  | Metrics collection & alerting    |
| Grafana     | Dashboards & visualization       |

## Quick Start

### GitHub Codespaces (recommended)

1. Click **Code → Codespaces → Create codespace on main**
2. Wait for the DevContainer to build (~3 minutes first time)
3. Run `make up` in the terminal
4. All services are accessible via forwarded ports

### Local (macOS)

Requires Docker Desktop, then:

```bash
make up
```

## Repository Structure

```
devx-platform/
├── .devcontainer/       # DevContainer for Codespaces / VS Code
├── terraform/
│   ├── environments/    # Per-environment compositions (local, codespaces)
│   └── modules/         # Reusable Terraform modules
├── helm/charts/         # Custom Helm charts
├── scripts/             # Bootstrap, teardown, verify
└── Makefile             # Developer-facing commands
```
```

### 1.5 Verification

Open the repository in Codespaces (or rebuild the DevContainer locally). Then run:

```bash
# Each tool should print its version
docker --version
k3d version
kubectl version --client
helm version --short
terraform version
k9s version --short
yq --version
argocd version --client
```

All eight commands must succeed and print a version number.

### 1.6 Commit

```bash
git add -A
git commit -m "ch01: devcontainer and tooling setup

- DevContainer Dockerfile with k3d, terraform, k9s, yq, argocd CLI
- devcontainer.json with docker-in-docker, kubectl, helm features
- Port forwarding for all platform services
- Repository scaffolding: .gitignore, .editorconfig, README.md
- Empty directory structure for all future components"
```

---

## Chapter 2 — Your First k3d Cluster (Manual)

**Goal:** Understand what k3d does by creating a cluster manually. Explore the Kubernetes components that k3s provides, understand the kubeconfig, and see the local container registry in action.

**Components introduced:** k3d CLI, k3s, kubeconfig, local container registry.

### 2.1 Create a cluster

```bash
k3d cluster create devx \
  --servers 1 \
  --agents 0 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer" \
  --registry-create devx-registry:0.0.0.0:5111 \
  --wait
```

This creates:
- A single-node k3s cluster running inside a Docker container
- A load balancer container mapping ports 80 and 443 to your machine
- A container registry at `localhost:5111`

### 2.2 Explore the cluster

```bash
# Check Docker containers k3d created
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"

# k3d auto-configured your kubeconfig — verify
kubectl cluster-info

# List nodes (should show 1 server node)
kubectl get nodes -o wide

# See what k3s ships out of the box
kubectl get pods -A
```

You'll see k3s bundles: CoreDNS, Traefik ingress controller, local-path-provisioner (for PVCs), metrics-server, and the svclb daemonset (load balancer).

### 2.3 Test the local registry

```bash
# Pull a tiny image and push it to the local registry
docker pull nginx:alpine
docker tag nginx:alpine localhost:5111/test-nginx:latest
docker push localhost:5111/test-nginx:latest

# Verify the registry has it
curl -s http://localhost:5111/v2/_catalog
# Should output: {"repositories":["test-nginx"]}

# Deploy it from the local registry into the cluster
kubectl run test-nginx --image=k3d-devx-registry:5111/test-nginx:latest
kubectl get pods
```

Note the registry hostname difference: from your machine it's `localhost:5111`, from inside the cluster it's `k3d-devx-registry:5111` (the container name k3d assigns).

### 2.4 Clean up

```bash
kubectl delete pod test-nginx
k3d cluster delete devx
```

### 2.5 Verification

```bash
# Cluster should be gone
k3d cluster list
# Should show empty list

# Docker containers should be cleaned up
docker ps --filter "name=k3d" --format "{{.Names}}"
# Should show nothing
```

### 2.6 Commit

Nothing to commit to the repo this chapter — it was purely exploratory. But write down what you learned:

```bash
git add -A
git commit --allow-empty -m "ch02: explored k3d cluster manually

- Created k3d cluster with port mappings and local registry
- Explored k3s built-in components (CoreDNS, Traefik, local-path)
- Tested local registry push/pull cycle
- Verified cluster cleanup with k3d cluster delete"
```

---

## Chapter 3 — Terraform Fundamentals: The k3d-cluster Module

**Goal:** Recreate what you did by hand in Chapter 2, but declaratively with Terraform. Build a reusable module that any environment can consume.

**Components introduced:** Terraform providers, modules, `init`/`plan`/`apply`/`destroy` lifecycle.

### 3.1 The k3d-cluster module

Create `terraform/modules/k3d-cluster/variables.tf`:

```hcl
variable "cluster_name" {
  description = "Name of the k3d cluster"
  type        = string
  default     = "devx"
}

variable "servers" {
  description = "Number of server (control-plane) nodes"
  type        = number
  default     = 1
}

variable "agents" {
  description = "Number of agent (worker) nodes"
  type        = number
  default     = 0
}

variable "registry_name" {
  description = "Name of the local container registry"
  type        = string
  default     = "devx-registry"
}

variable "registry_port" {
  description = "Host port for the local container registry"
  type        = number
  default     = 5111
}

variable "http_port" {
  description = "Host port mapped to cluster port 80"
  type        = number
  default     = 80
}

variable "https_port" {
  description = "Host port mapped to cluster port 443"
  type        = number
  default     = 443
}
```

Create `terraform/modules/k3d-cluster/main.tf`:

```hcl
terraform {
  required_providers {
    k3d = {
      source  = "sneakybugs/k3d"
      version = "~> 1.0"
    }
  }
}

resource "k3d_cluster" "this" {
  name    = var.cluster_name
  servers = var.servers
  agents  = var.agents

  port {
    host_port      = var.http_port
    container_port = 80
    node_filters   = ["loadbalancer"]
  }

  port {
    host_port      = var.https_port
    container_port = 443
    node_filters   = ["loadbalancer"]
  }

  registry {
    name      = var.registry_name
    host      = "0.0.0.0"
    host_port = var.registry_port
  }

  k3s_config {
    # Keep default Traefik for simplicity
  }
}
```

Create `terraform/modules/k3d-cluster/outputs.tf`:

```hcl
output "cluster_name" {
  description = "Name of the k3d cluster"
  value       = k3d_cluster.this.name
}

output "kubeconfig" {
  description = "Raw kubeconfig for the cluster"
  value       = k3d_cluster.this.kubeconfig
  sensitive   = true
}

output "host" {
  description = "Kubernetes API server host"
  value       = k3d_cluster.this.host
}

output "client_certificate" {
  description = "Client certificate for Kubernetes auth"
  value       = k3d_cluster.this.client_certificate
  sensitive   = true
}

output "client_key" {
  description = "Client key for Kubernetes auth"
  value       = k3d_cluster.this.client_key
  sensitive   = true
}

output "cluster_ca_certificate" {
  description = "CA certificate for the cluster"
  value       = k3d_cluster.this.cluster_ca_certificate
  sensitive   = true
}

output "registry_url" {
  description = "URL of the local container registry"
  value       = "k3d-${var.registry_name}:${var.registry_port}"
}
```

### 3.2 Test the module standalone

Create a temporary test file to try it out. Create `terraform/modules/k3d-cluster/test.tf`:

```hcl
provider "k3d" {}

module "test_cluster" {
  source = "."
}
```

Run the Terraform lifecycle:

```bash
cd terraform/modules/k3d-cluster

# Download the k3d provider
terraform init

# Preview what will be created
terraform plan

# Create the cluster
terraform apply -auto-approve

# Verify the cluster is running
k3d cluster list
kubectl get nodes
kubectl get pods -A

# Destroy the cluster
terraform destroy -auto-approve

# Verify clean teardown
k3d cluster list
```

Now remove the test file — we won't commit it:

```bash
rm test.tf terraform.tfstate* .terraform.lock.hcl
rm -rf .terraform
```

### 3.3 Verification

```bash
# The module files should exist and be valid HCL
cd terraform/modules/k3d-cluster
terraform fmt -check
terraform validate 2>&1 | grep -q "Success" || echo "Note: validate may fail without provider init, that's OK for module-only validation"
```

### 3.4 Commit

```bash
cd ../../..  # back to repo root
git add terraform/modules/k3d-cluster/
git commit -m "ch03: terraform k3d-cluster module

- Reusable module to create k3d cluster with registry and port mappings
- Variables for cluster name, node counts, ports, registry config
- Outputs: kubeconfig, host, certificates, registry URL
- Uses sneakybugs/k3d terraform provider"
```

---

## Chapter 4 — Kubernetes Namespaces: The platform-base Module

**Goal:** Build a second Terraform module that creates the namespaces all platform services will live in. Learn to use the Kubernetes Terraform provider and module-to-module dependencies.

**Components introduced:** Terraform Kubernetes provider, namespaces as organizational units.

### 4.1 The platform-base module

Create `terraform/modules/platform-base/variables.tf`:

```hcl
variable "host" {
  description = "Kubernetes API server host"
  type        = string
}

variable "client_certificate" {
  description = "Client certificate for Kubernetes auth (PEM)"
  type        = string
  sensitive   = true
}

variable "client_key" {
  description = "Client key for Kubernetes auth (PEM)"
  type        = string
  sensitive   = true
}

variable "cluster_ca_certificate" {
  description = "CA certificate for the cluster (PEM)"
  type        = string
  sensitive   = true
}

variable "namespaces" {
  description = "List of namespaces to create"
  type        = list(string)
  default = [
    "backstage",
    "devx-apps",
    "observability",
    "crossplane-system",
    "argocd",
  ]
}
```

Create `terraform/modules/platform-base/main.tf`:

```hcl
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

provider "kubernetes" {
  host                   = var.host
  client_certificate     = var.client_certificate
  client_key             = var.client_key
  cluster_ca_certificate = var.cluster_ca_certificate
}

resource "kubernetes_namespace" "this" {
  for_each = toset(var.namespaces)

  metadata {
    name = each.value
    labels = {
      "managed-by" = "terraform"
      "part-of"    = "devx-platform"
    }
  }
}
```

Create `terraform/modules/platform-base/outputs.tf`:

```hcl
output "namespace_names" {
  description = "List of created namespace names"
  value       = [for ns in kubernetes_namespace.this : ns.metadata[0].name]
}
```

### 4.2 Verification

```bash
cd terraform/modules/platform-base
terraform fmt -check
```

The module can't be tested in isolation without a running cluster — that's expected. It will be wired up in the next chapter.

### 4.3 Commit

```bash
cd ../../..
git add terraform/modules/platform-base/
git commit -m "ch04: terraform platform-base module

- Creates namespaces: backstage, devx-apps, observability, crossplane-system, argocd
- Accepts Kubernetes connection details as variables (host, certs)
- Labels all namespaces with managed-by=terraform
- Uses for_each for clean namespace management"
```

---

## Chapter 5 — Environment Composition: Wiring Modules Together

**Goal:** Create the local and codespaces environment directories that compose the k3d-cluster and platform-base modules. A single `terraform apply` creates everything; a single `destroy` tears it down.

**Components introduced:** Terraform environment pattern, module composition, provider configuration chains.

### 5.1 Local environment

Create `terraform/environments/local/providers.tf`:

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    k3d = {
      source  = "sneakybugs/k3d"
      version = "~> 1.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

provider "k3d" {}

provider "kubernetes" {
  host                   = module.cluster.host
  client_certificate     = module.cluster.client_certificate
  client_key             = module.cluster.client_key
  cluster_ca_certificate = module.cluster.cluster_ca_certificate
}

provider "helm" {
  kubernetes {
    host                   = module.cluster.host
    client_certificate     = module.cluster.client_certificate
    client_key             = module.cluster.client_key
    cluster_ca_certificate = module.cluster.cluster_ca_certificate
  }
}
```

Create `terraform/environments/local/variables.tf`:

```hcl
variable "cluster_name" {
  description = "Name of the k3d cluster"
  type        = string
  default     = "devx"
}

variable "environment" {
  description = "Environment name (local, codespaces, cloud)"
  type        = string
  default     = "local"
}
```

Create `terraform/environments/local/main.tf`:

```hcl
# --- Cluster ---
module "cluster" {
  source = "../../modules/k3d-cluster"

  cluster_name = var.cluster_name
}

# --- Platform Base (namespaces) ---
module "platform_base" {
  source = "../../modules/platform-base"

  host                   = module.cluster.host
  client_certificate     = module.cluster.client_certificate
  client_key             = module.cluster.client_key
  cluster_ca_certificate = module.cluster.cluster_ca_certificate
}
```

Create `terraform/environments/local/outputs.tf`:

```hcl
output "cluster_name" {
  value = module.cluster.cluster_name
}

output "registry_url" {
  value = module.cluster.registry_url
}

output "namespaces" {
  value = module.platform_base.namespace_names
}
```

Create `terraform/environments/local/terraform.tfvars`:

```hcl
cluster_name = "devx"
environment  = "local"
```

### 5.2 Codespaces environment

The Codespaces environment is structurally identical but will diverge later when we configure Backstage URLs. For now, create it as a copy:

Create `terraform/environments/codespaces/providers.tf`:

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    k3d = {
      source  = "sneakybugs/k3d"
      version = "~> 1.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

provider "k3d" {}

provider "kubernetes" {
  host                   = module.cluster.host
  client_certificate     = module.cluster.client_certificate
  client_key             = module.cluster.client_key
  cluster_ca_certificate = module.cluster.cluster_ca_certificate
}

provider "helm" {
  kubernetes {
    host                   = module.cluster.host
    client_certificate     = module.cluster.client_certificate
    client_key             = module.cluster.client_key
    cluster_ca_certificate = module.cluster.cluster_ca_certificate
  }
}
```

Create `terraform/environments/codespaces/variables.tf`:

```hcl
variable "cluster_name" {
  description = "Name of the k3d cluster"
  type        = string
  default     = "devx"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "codespaces"
}

variable "codespace_name" {
  description = "GitHub Codespace name (auto-detected from env)"
  type        = string
  default     = ""
}
```

Create `terraform/environments/codespaces/main.tf`:

```hcl
# --- Cluster ---
module "cluster" {
  source = "../../modules/k3d-cluster"

  cluster_name = var.cluster_name
}

# --- Platform Base (namespaces) ---
module "platform_base" {
  source = "../../modules/platform-base"

  host                   = module.cluster.host
  client_certificate     = module.cluster.client_certificate
  client_key             = module.cluster.client_key
  cluster_ca_certificate = module.cluster.cluster_ca_certificate
}
```

Create `terraform/environments/codespaces/outputs.tf`:

```hcl
output "cluster_name" {
  value = module.cluster.cluster_name
}

output "registry_url" {
  value = module.cluster.registry_url
}

output "namespaces" {
  value = module.platform_base.namespace_names
}
```

Create `terraform/environments/codespaces/terraform.tfvars`:

```hcl
cluster_name = "devx"
environment  = "codespaces"
```

Add a placeholder for the future cloud environment:

Create `terraform/environments/cloud/.gitkeep` (empty file).

### 5.3 Test the full lifecycle

```bash
cd terraform/environments/local

# Initialize (downloads providers)
terraform init

# Preview
terraform plan

# Create cluster + namespaces
terraform apply -auto-approve

# Verify cluster
k3d cluster list
kubectl get nodes

# Verify namespaces
kubectl get namespaces -l managed-by=terraform
# Should show: argocd, backstage, crossplane-system, devx-apps, observability

# Verify registry
curl -s http://localhost:5111/v2/_catalog

# Destroy everything
terraform destroy -auto-approve

# Verify clean teardown
k3d cluster list
```

### 5.4 Verification

After running apply, these must all pass:

```bash
# Cluster has 1 ready node
[ "$(kubectl get nodes --no-headers | wc -l)" -eq 1 ] && echo "PASS: 1 node" || echo "FAIL"

# All 5 namespaces exist
for ns in backstage devx-apps observability crossplane-system argocd; do
  kubectl get namespace "$ns" > /dev/null 2>&1 && echo "PASS: $ns" || echo "FAIL: $ns"
done

# Registry responds
curl -sf http://localhost:5111/v2/_catalog > /dev/null && echo "PASS: registry" || echo "FAIL: registry"
```

After running destroy:

```bash
# No k3d clusters remain
[ "$(k3d cluster list --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo "PASS: clean teardown" || echo "FAIL"
```

### 5.5 Commit

```bash
cd ../../..
# Don't commit .terraform dirs or state files (.gitignore handles this)
git add terraform/environments/
git commit -m "ch05: environment composition (local + codespaces)

- terraform/environments/local/ composes k3d-cluster + platform-base
- terraform/environments/codespaces/ mirrors local with env-specific vars
- Single 'terraform apply' creates cluster + all namespaces
- Single 'terraform destroy' tears everything down
- Kubernetes and Helm providers wired to cluster outputs"
```

---

## Chapter 6 — Helm Charts: Understanding Packaging

**Goal:** Build a minimal Helm chart from scratch to understand chart structure, templates, values, and releases. You'll deploy a simple nginx-based app that we'll reuse later when testing ArgoCD.

**Components introduced:** Helm chart structure, Go templates, values.yaml, `helm install`/`upgrade`/`uninstall`.

### 6.1 Chart metadata

Create `helm/charts/devx-app/Chart.yaml`:

```yaml
apiVersion: v2
name: devx-app
description: Generic chart for DevX platform applications
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### 6.2 Default values

Create `helm/charts/devx-app/values.yaml`:

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: alpine
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  host: ""
  annotations: {}

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 128Mi

config: {}
```

### 6.3 Templates

Create `helm/charts/devx-app/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

Create `helm/charts/devx-app/templates/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
      protocol: TCP
  selector:
    app.kubernetes.io/instance: {{ .Release.Name }}
```

Create `helm/charts/devx-app/templates/ingress.yaml`:

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

Create `helm/charts/devx-app/templates/configmap.yaml`:

```yaml
{{- if .Values.config }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

### 6.4 Test the chart

First, bring the cluster up (if not already running):

```bash
cd terraform/environments/local
terraform init
terraform apply -auto-approve
cd ../../..
```

Then lint and install the chart:

```bash
# Lint the chart
helm lint helm/charts/devx-app/

# Dry-run to see rendered YAML
helm template test-app helm/charts/devx-app/ --namespace devx-apps

# Install into the devx-apps namespace
helm install test-app helm/charts/devx-app/ \
  --namespace devx-apps \
  --set ingress.enabled=true \
  --set ingress.host=test-app.localhost

# Watch the pod come up
kubectl get pods -n devx-apps -w
# Press Ctrl+C once STATUS is Running

# Test it
curl -s -H "Host: test-app.localhost" http://localhost | head -5
```

Clean up the test release:

```bash
helm uninstall test-app --namespace devx-apps
```

Leave the cluster running — we'll use it in the next chapters.

### 6.5 Verification

```bash
# Chart lints cleanly
helm lint helm/charts/devx-app/ 2>&1 | grep -q "1 chart(s) linted, 0 chart(s) failed" && echo "PASS: lint" || echo "FAIL: lint"

# Template renders valid YAML
helm template test helm/charts/devx-app/ --namespace devx-apps | kubectl apply --dry-run=client -f - > /dev/null 2>&1 \
  && echo "PASS: template renders valid K8s resources" || echo "FAIL"
```

### 6.6 Commit

```bash
git add helm/
git commit -m "ch06: custom devx-app Helm chart

- Generic chart with deployment, service, ingress, configmap
- Configurable via values.yaml: replicas, image, resources, ingress
- Liveness and readiness probes configured
- Tested with helm install/uninstall against live cluster"
```

---

## Chapter 7 — Crossplane: Kubernetes-Native Infrastructure Management

**Goal:** Install Crossplane into your cluster and understand its architecture. Use `provider-kubernetes` to manage Kubernetes resources declaratively through Crossplane, demonstrating the concept of an infrastructure control plane.

**Components introduced:** Crossplane, Providers, Managed Resources.

### 7.1 Why Crossplane?

Terraform manages infrastructure from outside the cluster. Crossplane manages infrastructure from inside — it extends the Kubernetes API with custom resources that represent infrastructure. When a developer creates a YAML file requesting a database, Crossplane provisions it. This is the foundation for self-service infrastructure.

### 7.2 Crossplane Terraform module

Create `terraform/modules/crossplane/variables.tf`:

```hcl
variable "namespace" {
  description = "Namespace to install Crossplane into"
  type        = string
  default     = "crossplane-system"
}

variable "chart_version" {
  description = "Crossplane Helm chart version"
  type        = string
  default     = "1.17.0"
}
```

Create `terraform/modules/crossplane/main.tf`:

```hcl
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

resource "helm_release" "crossplane" {
  name       = "crossplane"
  namespace  = var.namespace
  repository = "https://charts.crossplane.io/stable"
  chart      = "crossplane"
  version    = var.chart_version

  wait          = true
  wait_for_jobs = true
  timeout       = 300

  set {
    name  = "args"
    value = "{--enable-usages}"
  }
}
```

Create `terraform/modules/crossplane/outputs.tf`:

```hcl
output "release_name" {
  description = "Name of the Crossplane Helm release"
  value       = helm_release.crossplane.name
}

output "release_namespace" {
  description = "Namespace where Crossplane is installed"
  value       = helm_release.crossplane.namespace
}

output "release_version" {
  description = "Version of the Crossplane chart deployed"
  value       = helm_release.crossplane.version
}
```

### 7.3 Wire Crossplane into the local environment

Update `terraform/environments/local/main.tf` — add after the platform_base module:

```hcl
# --- Crossplane ---
module "crossplane" {
  source = "../../modules/crossplane"

  namespace = "crossplane-system"

  depends_on = [module.platform_base]
}
```

### 7.4 Apply and explore

```bash
cd terraform/environments/local
terraform init -upgrade
terraform apply -auto-approve

# Watch Crossplane pods come up
kubectl get pods -n crossplane-system -w
# Wait until all pods show Running/Completed, then Ctrl+C

# Crossplane extends the Kubernetes API — see what it added
kubectl api-resources | grep crossplane
```

### 7.5 Install provider-kubernetes

Crossplane uses Providers to interact with external systems. `provider-kubernetes` lets Crossplane manage Kubernetes resources — perfect for learning without needing cloud credentials.

Create a file called `crossplane-provider-k8s.yaml` in the repo root (we'll move it later):

```bash
cat <<'EOF' > crossplane-provider-k8s.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.14.1
  runtimeConfigRef:
    name: provider-kubernetes
---
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: provider-kubernetes
spec:
  serviceAccountTemplate:
    metadata:
      name: provider-kubernetes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: provider-kubernetes-cluster-admin
subjects:
  - kind: ServiceAccount
    name: provider-kubernetes
    namespace: crossplane-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

Apply it:

```bash
kubectl apply -f crossplane-provider-k8s.yaml

# Wait for the provider to become healthy
kubectl get providers -w
# Wait until HEALTHY shows True, then Ctrl+C

# Create a ProviderConfig that uses in-cluster credentials
cat <<'EOF' | kubectl apply -f -
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity
EOF
```

### 7.6 Create your first Managed Resource

Use Crossplane to create a ConfigMap — this proves Crossplane can manage Kubernetes resources:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: kubernetes.crossplane.io/v1alpha2
kind: Object
metadata:
  name: crossplane-demo-configmap
spec:
  forProvider:
    manifest:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: managed-by-crossplane
        namespace: devx-apps
      data:
        message: "This ConfigMap was created by Crossplane"
        managed-by: "crossplane/provider-kubernetes"
  providerConfigRef:
    name: default
EOF

# Verify Crossplane created it
kubectl get configmap managed-by-crossplane -n devx-apps -o yaml

# Check the Object status
kubectl get objects.kubernetes.crossplane.io crossplane-demo-configmap
```

### 7.7 Clean up the demo resources

```bash
# Delete the managed resource — Crossplane will delete the ConfigMap too
kubectl delete objects.kubernetes.crossplane.io crossplane-demo-configmap

# Verify ConfigMap was cleaned up
kubectl get configmap managed-by-crossplane -n devx-apps 2>&1 | grep -q "not found" && echo "PASS: cleanup" || echo "FAIL"

# Keep the provider installed — we'll use it in the next chapter
rm crossplane-provider-k8s.yaml
```

### 7.8 Verification

```bash
# Crossplane pods are running
kubectl get pods -n crossplane-system --no-headers | grep -v Completed | awk '{print $3}' | sort -u
# Should show only: Running

# Provider is healthy
kubectl get providers provider-kubernetes -o jsonpath='{.status.conditions[?(@.type=="Healthy")].status}'
# Should print: True

echo ""
echo "PASS: Crossplane installed with provider-kubernetes"
```

### 7.9 Commit

```bash
git add terraform/modules/crossplane/ terraform/environments/local/main.tf
git commit -m "ch07: crossplane installation with provider-kubernetes

- Terraform module for Crossplane Helm chart deployment
- Wired into local environment with depends_on platform_base
- Installed provider-kubernetes for in-cluster resource management
- Tested managed resource lifecycle (create ConfigMap, verify, delete)
- Crossplane extends K8s API for declarative infrastructure"
```

---

## Chapter 8 — Crossplane Compositions: Building Your Own Platform API

**Goal:** Define a Composite Resource Definition (XRD) and Composition that abstracts multiple Kubernetes resources behind a single, simple Claim. This is where Crossplane becomes a platform building block.

**Components introduced:** XRD, Composition, Composite Resource, Claim.

### 8.1 The concept

Instead of developers knowing they need a Namespace + ConfigMap + ResourceQuota, you define a single "DevEnvironment" API. They create a Claim saying "I want a dev environment called team-alpha" and Crossplane provisions everything.

### 8.2 Create the XRD

This defines the schema — what developers can request:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdevenvironments.platform.devx.io
spec:
  group: platform.devx.io
  names:
    kind: XDevEnvironment
    plural: xdevenvironments
  claimNames:
    kind: DevEnvironment
    plural: devenvironments
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                teamName:
                  type: string
                  description: "Name of the team"
                cpuLimit:
                  type: string
                  default: "2"
                  description: "CPU limit for the resource quota"
                memoryLimit:
                  type: string
                  default: "4Gi"
                  description: "Memory limit for the resource quota"
              required:
                - teamName
EOF
```

### 8.3 Create the Composition

This defines what gets created when someone submits a Claim:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: devenvironment-composition
  labels:
    crossplane.io/xrd: xdevenvironments.platform.devx.io
spec:
  compositeTypeRef:
    apiVersion: platform.devx.io/v1alpha1
    kind: XDevEnvironment
  resources:
    # 1. Create a Namespace for the team
    - name: team-namespace
      base:
        apiVersion: kubernetes.crossplane.io/v1alpha2
        kind: Object
        spec:
          forProvider:
            manifest:
              apiVersion: v1
              kind: Namespace
              metadata:
                name: ""  # patched below
                labels:
                  managed-by: crossplane
                  part-of: devx-platform
          providerConfigRef:
            name: default
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.teamName
          toFieldPath: spec.forProvider.manifest.metadata.name
          transforms:
            - type: string
              string:
                fmt: "team-%s"

    # 2. Create a ConfigMap with team info
    - name: team-config
      base:
        apiVersion: kubernetes.crossplane.io/v1alpha2
        kind: Object
        spec:
          forProvider:
            manifest:
              apiVersion: v1
              kind: ConfigMap
              metadata:
                name: ""
                namespace: ""
              data:
                team: ""
          providerConfigRef:
            name: default
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.teamName
          toFieldPath: spec.forProvider.manifest.metadata.namespace
          transforms:
            - type: string
              string:
                fmt: "team-%s"
        - type: FromCompositeFieldPath
          fromFieldPath: spec.teamName
          toFieldPath: spec.forProvider.manifest.metadata.name
          transforms:
            - type: string
              string:
                fmt: "%s-team-config"
        - type: FromCompositeFieldPath
          fromFieldPath: spec.teamName
          toFieldPath: spec.forProvider.manifest.data.team

    # 3. Create a ResourceQuota
    - name: resource-quota
      base:
        apiVersion: kubernetes.crossplane.io/v1alpha2
        kind: Object
        spec:
          forProvider:
            manifest:
              apiVersion: v1
              kind: ResourceQuota
              metadata:
                name: default-quota
                namespace: ""
              spec:
                hard:
                  requests.cpu: "1"
                  requests.memory: 2Gi
                  limits.cpu: ""
                  limits.memory: ""
          providerConfigRef:
            name: default
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.teamName
          toFieldPath: spec.forProvider.manifest.metadata.namespace
          transforms:
            - type: string
              string:
                fmt: "team-%s"
        - type: FromCompositeFieldPath
          fromFieldPath: spec.cpuLimit
          toFieldPath: spec.forProvider.manifest.spec.hard["limits.cpu"]
        - type: FromCompositeFieldPath
          fromFieldPath: spec.memoryLimit
          toFieldPath: spec.forProvider.manifest.spec.hard["limits.memory"]
EOF
```

### 8.4 Submit a Claim

Now act as a developer requesting an environment:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: platform.devx.io/v1alpha1
kind: DevEnvironment
metadata:
  name: alpha-env
  namespace: devx-apps
spec:
  teamName: alpha
  cpuLimit: "4"
  memoryLimit: "8Gi"
EOF

# Watch Crossplane provision everything
kubectl get devenvironments -n devx-apps -w
# Wait until SYNCED and READY both show True, then Ctrl+C
```

### 8.5 Verify what was created

```bash
# The namespace
kubectl get namespace team-alpha --show-labels

# The ConfigMap
kubectl get configmap alpha-team-config -n team-alpha -o yaml

# The ResourceQuota
kubectl get resourcequota default-quota -n team-alpha -o yaml

# The Composite Resource (cluster-scoped parent)
kubectl get xdevenvironments
```

### 8.6 Clean up

```bash
# Delete the claim — Crossplane deletes everything it created
kubectl delete devenvironment alpha-env -n devx-apps

# Wait a moment, then verify
sleep 10
kubectl get namespace team-alpha 2>&1 | grep -q "not found" && echo "PASS: namespace cleaned up" || echo "Still deleting..."
```

### 8.7 Verification

```bash
# XRD is established
kubectl get xrd xdevenvironments.platform.devx.io -o jsonpath='{.status.conditions[?(@.type=="Established")].status}'
echo ""
# Should print: True

# Composition exists
kubectl get compositions devenvironment-composition > /dev/null 2>&1 && echo "PASS: composition exists" || echo "FAIL"

echo "PASS: Crossplane platform API working"
```

### 8.8 Commit

```bash
git add -A
git commit -m "ch08: crossplane compositions — platform API

- XRD defines DevEnvironment claim with teamName, cpuLimit, memoryLimit
- Composition provisions: namespace, configmap, resource quota per team
- Patches transform teamName into resource names and namespaces
- Full lifecycle tested: claim → provision → verify → delete → cleanup
- Developers get self-service infrastructure via simple YAML claims"
```

---

## Chapter 9 — ArgoCD: GitOps Delivery

**Goal:** Install ArgoCD into the cluster, understand the Application CRD and sync model, and deploy the devx-app chart from Chapter 6 using GitOps.

**Components introduced:** ArgoCD, Application CRD, sync policies, ArgoCD UI.

### 9.1 ArgoCD Terraform module

Create `terraform/modules/argocd/variables.tf`:

```hcl
variable "namespace" {
  description = "Namespace to install ArgoCD into"
  type        = string
  default     = "argocd"
}

variable "chart_version" {
  description = "ArgoCD Helm chart version"
  type        = string
  default     = "7.3.11"
}

variable "server_service_type" {
  description = "Service type for the ArgoCD server"
  type        = string
  default     = "ClusterIP"
}
```

Create `terraform/modules/argocd/main.tf`:

```hcl
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

resource "helm_release" "argocd" {
  name       = "argocd"
  namespace  = var.namespace
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = var.chart_version

  wait    = true
  timeout = 600

  # Minimal resource footprint for local dev
  set {
    name  = "server.service.type"
    value = var.server_service_type
  }

  # Disable dex (we'll use admin login for dev)
  set {
    name  = "dex.enabled"
    value = "false"
  }

  # Insecure mode (no TLS termination in ArgoCD, Traefik handles it)
  set {
    name  = "configs.params.server\\.insecure"
    value = "true"
  }

  # Resource limits for local dev
  set {
    name  = "server.resources.requests.cpu"
    value = "100m"
  }
  set {
    name  = "server.resources.requests.memory"
    value = "128Mi"
  }
  set {
    name  = "server.resources.limits.cpu"
    value = "500m"
  }
  set {
    name  = "server.resources.limits.memory"
    value = "512Mi"
  }
}
```

Create `terraform/modules/argocd/outputs.tf`:

```hcl
output "release_name" {
  description = "Name of the ArgoCD Helm release"
  value       = helm_release.argocd.name
}

output "release_namespace" {
  description = "Namespace where ArgoCD is installed"
  value       = helm_release.argocd.namespace
}

output "release_version" {
  description = "Version of the ArgoCD chart deployed"
  value       = helm_release.argocd.version
}
```

### 9.2 Wire ArgoCD into the local environment

Add to `terraform/environments/local/main.tf`:

```hcl
# --- ArgoCD ---
module "argocd" {
  source = "../../modules/argocd"

  namespace = "argocd"

  depends_on = [module.platform_base]
}
```

### 9.3 Apply

```bash
cd terraform/environments/local
terraform init -upgrade
terraform apply -auto-approve

# Watch ArgoCD pods
kubectl get pods -n argocd -w
# Wait until all pods are Running, then Ctrl+C
```

### 9.4 Access the ArgoCD UI

```bash
# Get the initial admin password
ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD admin password: $ARGO_PWD"

# Port-forward the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGO_PF_PID=$!

# Wait a moment for port-forward to establish
sleep 3

# Test it (--insecure because it's self-signed TLS)
curl -sk https://localhost:8080 | head -5

# Login with the CLI
argocd login localhost:8080 --insecure --username admin --password "$ARGO_PWD"

# List applications (should be empty)
argocd app list
```

In Codespaces, the port 8080 will be auto-forwarded — you can open it in the browser.

### 9.5 Deploy devx-app via ArgoCD

First, ArgoCD needs to know about the Helm chart. Since our chart is in the same repo, we'll point ArgoCD at the repo. For this to work, push your code to GitHub first:

```bash
# Stop port-forward for now
kill $ARGO_PF_PID 2>/dev/null

# Push what we have so far
cd ../../..
git add -A
git commit -m "ch09-wip: argocd module and environment wiring"
git push origin main
```

Now create an ArgoCD Application that deploys the devx-app chart:

```bash
# Replace YOUR_GITHUB_USERNAME with your actual GitHub username
export GITHUB_REPO="https://github.com/YOUR_GITHUB_USERNAME/devx-platform.git"

cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: ${GITHUB_REPO}
    targetRevision: main
    path: helm/charts/devx-app
    helm:
      valuesObject:
        replicaCount: 1
        ingress:
          enabled: true
          host: sample-app.localhost
  destination:
    server: https://kubernetes.default.svc
    namespace: devx-apps
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
EOF

# Watch the sync
argocd app get sample-app --refresh
kubectl get pods -n devx-apps -w
# Wait until the pod is Running, then Ctrl+C

# Test the deployed app
curl -s -H "Host: sample-app.localhost" http://localhost | head -5
```

### 9.6 Observe GitOps in action

```bash
# Check the app status
argocd app get sample-app

# Manually delete the pod — ArgoCD self-heals
kubectl delete pod -n devx-apps -l app.kubernetes.io/instance=sample-app
kubectl get pods -n devx-apps -w
# ArgoCD detects drift and recreates the pod
```

### 9.7 Clean up the sample app

```bash
argocd app delete sample-app --yes
```

### 9.8 Verification

```bash
# ArgoCD server is running
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server --no-headers | grep Running > /dev/null \
  && echo "PASS: argocd-server running" || echo "FAIL"

# ArgoCD API responds
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGO_PF_PID=$!
sleep 3
curl -sk https://localhost:8080/api/version > /dev/null 2>&1 \
  && echo "PASS: argocd API responding" || echo "FAIL"
kill $ARGO_PF_PID 2>/dev/null

echo "PASS: ArgoCD operational"
```

### 9.9 Commit

```bash
git add terraform/modules/argocd/ terraform/environments/local/main.tf
git commit -m "ch09: argocd installation and gitops deployment

- Terraform module for ArgoCD Helm chart
- Wired into local environment with depends_on platform_base
- Insecure mode for local dev (no TLS in ArgoCD)
- Tested GitOps flow: Application CRD → auto-sync → self-heal
- Deployed devx-app chart via ArgoCD Application
- ArgoCD UI accessible via port-forward on 8080"
```

---

## Chapter 10 — ArgoCD App-of-Apps: Managing Multiple Deployments

**Goal:** Structure an app-of-apps pattern where a single root Application manages child Applications. This is how you'd manage all platform services declaratively.

**Components introduced:** App-of-apps pattern, ArgoCD Application sets.

### 10.1 Create the apps directory

```bash
mkdir -p argocd/apps
```

### 10.2 Root application manifest

Create `argocd/apps/sample-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/devx-platform.git
    targetRevision: main
    path: helm/charts/devx-app
    helm:
      valuesObject:
        replicaCount: 1
        ingress:
          enabled: true
          host: sample-app.localhost
  destination:
    server: https://kubernetes.default.svc
    namespace: devx-apps
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Create `argocd/root-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/devx-platform.git
    targetRevision: main
    path: argocd/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

> **Note:** Replace `YOUR_GITHUB_USERNAME` in both files with your actual GitHub username before committing.

### 10.3 Deploy the app-of-apps

```bash
# Push the new files to GitHub first (ArgoCD reads from Git)
git add argocd/
git commit -m "ch10-wip: app-of-apps manifests"
git push origin main

# Apply the root application
kubectl apply -f argocd/root-app.yaml

# Watch ArgoCD discover and sync the child apps
argocd app get root-app --refresh

# The sample-app should appear and sync automatically
argocd app list
kubectl get pods -n devx-apps -w
# Wait for Running, then Ctrl+C
```

### 10.4 Add a second app via Git

Create `argocd/apps/second-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: second-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_GITHUB_USERNAME/devx-platform.git
    targetRevision: main
    path: helm/charts/devx-app
    helm:
      valuesObject:
        replicaCount: 1
        ingress:
          enabled: true
          host: second-app.localhost
  destination:
    server: https://kubernetes.default.svc
    namespace: devx-apps
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
git add argocd/apps/second-app.yaml
git commit -m "ch10-wip: add second-app to app-of-apps"
git push origin main

# ArgoCD auto-syncs and deploys the second app
sleep 30
argocd app list
# Should show: root-app, sample-app, second-app

curl -s -H "Host: second-app.localhost" http://localhost | head -5
```

### 10.5 Clean up demo apps

Remove the demo apps but keep the structure:

```bash
# Delete the root app — cascades to all children
argocd app delete root-app --cascade --yes

# Remove demo manifests (keep the directory)
rm argocd/apps/sample-app.yaml argocd/apps/second-app.yaml
```

### 10.6 Verification

```bash
# ArgoCD app list should be clean
[ "$(argocd app list -o name 2>/dev/null | wc -l)" -eq 0 ] \
  && echo "PASS: no lingering apps" || echo "INFO: some apps remain (may still be deleting)"

# ArgoCD server still healthy
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server --no-headers | grep Running > /dev/null \
  && echo "PASS: argocd healthy after cleanup" || echo "FAIL"

echo "PASS: App-of-apps pattern working"
```

### 10.7 Commit

```bash
git add -A
git commit -m "ch10: argocd app-of-apps pattern

- Root Application in argocd/root-app.yaml manages child apps
- Child Application manifests in argocd/apps/
- Adding a YAML file to argocd/apps/ auto-deploys via GitOps
- Cascade delete cleans up all children
- Pattern ready for managing Backstage, Crossplane, etc."
```

---

## Chapter 11 — Deploying Backstage with Terraform + Helm

**Goal:** Deploy Backstage — the developer portal — using the official Helm chart. PostgreSQL is included as a subchart. Configure ingress and guest authentication for development.

**Components introduced:** Backstage, Bitnami PostgreSQL subchart, ConfigMap-based app-config.

### 11.1 Backstage Helm values

Create `terraform/modules/backstage/values/base.yaml`:

```yaml
backstage:
  image:
    registry: ghcr.io
    repository: backstage/backstage
    tag: latest

  extraEnvVars:
    - name: APP_CONFIG_app_baseUrl
      value: "http://backstage.localhost"
    - name: APP_CONFIG_backend_baseUrl
      value: "http://backstage.localhost"
    - name: APP_CONFIG_auth_providers_guest_dangerouslyAllowOutsideDevelopment
      value: "true"

  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 1000m

ingress:
  enabled: true
  host: backstage.localhost

postgresql:
  enabled: true
  auth:
    password: backstage-dev
  primary:
    persistence:
      size: 1Gi
```

Create `terraform/modules/backstage/values/local.yaml`:

```yaml
# Local overrides — localhost access
backstage:
  extraEnvVars:
    - name: APP_CONFIG_app_baseUrl
      value: "http://backstage.localhost"
    - name: APP_CONFIG_backend_baseUrl
      value: "http://backstage.localhost"
    - name: APP_CONFIG_auth_providers_guest_dangerouslyAllowOutsideDevelopment
      value: "true"

ingress:
  host: backstage.localhost
```

Create `terraform/modules/backstage/values/codespaces.yaml`:

```yaml
# Codespaces overrides — dynamic URL
# The actual baseUrl is set via environment variable at deploy time
backstage:
  extraEnvVars:
    - name: APP_CONFIG_app_baseUrl
      value: "${backstage_base_url}"
    - name: APP_CONFIG_backend_baseUrl
      value: "${backstage_base_url}"
    - name: APP_CONFIG_auth_providers_guest_dangerouslyAllowOutsideDevelopment
      value: "true"

ingress:
  enabled: false  # Codespaces uses port forwarding, not ingress
```

### 11.2 Backstage Terraform module

Create `terraform/modules/backstage/variables.tf`:

```hcl
variable "namespace" {
  description = "Namespace to deploy Backstage into"
  type        = string
  default     = "backstage"
}

variable "chart_version" {
  description = "Backstage Helm chart version"
  type        = string
  default     = "1.9.2"
}

variable "environment" {
  description = "Environment name (local, codespaces)"
  type        = string
  default     = "local"
}

variable "backstage_base_url" {
  description = "Base URL for Backstage (used in Codespaces)"
  type        = string
  default     = "http://backstage.localhost"
}
```

Create `terraform/modules/backstage/main.tf`:

```hcl
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

resource "helm_release" "backstage" {
  name       = "backstage"
  namespace  = var.namespace
  repository = "https://backstage.github.io/charts"
  chart      = "backstage"
  version    = var.chart_version

  wait    = true
  timeout = 600

  values = [
    file("${path.module}/values/base.yaml"),
    file("${path.module}/values/${var.environment}.yaml"),
  ]
}
```

Create `terraform/modules/backstage/outputs.tf`:

```hcl
output "release_name" {
  description = "Name of the Backstage Helm release"
  value       = helm_release.backstage.name
}

output "release_namespace" {
  description = "Namespace where Backstage is deployed"
  value       = helm_release.backstage.namespace
}
```

### 11.3 Wire Backstage into the local environment

Add to `terraform/environments/local/main.tf`:

```hcl
# --- Backstage ---
module "backstage" {
  source = "../../modules/backstage"

  namespace   = "backstage"
  environment = var.environment

  depends_on = [module.platform_base]
}
```

### 11.4 Deploy

```bash
cd terraform/environments/local
terraform init -upgrade
terraform apply -auto-approve

# Watch pods in the backstage namespace
kubectl get pods -n backstage -w
# Wait for both backstage and postgresql pods to show Running
# This may take 2-5 minutes for image pulls
```

### 11.5 Access Backstage

```bash
# Via ingress (if Traefik is routing backstage.localhost)
curl -s -H "Host: backstage.localhost" http://localhost | head -20

# Or via port-forward (more reliable fallback)
kubectl port-forward svc/backstage -n backstage 7007:7007 &
BS_PF_PID=$!
sleep 3

curl -s http://localhost:7007 | head -20
# Should return HTML with "Backstage" in the content

kill $BS_PF_PID 2>/dev/null
```

In Codespaces, open the forwarded port 7007 in the browser.

### 11.6 Verification

```bash
# Backstage pod is running
kubectl get pods -n backstage -l app.kubernetes.io/name=backstage --no-headers | grep Running > /dev/null \
  && echo "PASS: backstage pod running" || echo "FAIL"

# PostgreSQL pod is running
kubectl get pods -n backstage -l app.kubernetes.io/name=postgresql --no-headers | grep Running > /dev/null \
  && echo "PASS: postgresql pod running" || echo "FAIL"

# Backstage responds via port-forward
kubectl port-forward svc/backstage -n backstage 7007:7007 &
BS_PF_PID=$!
sleep 5
HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:7007)
kill $BS_PF_PID 2>/dev/null
[ "$HTTP_CODE" = "200" ] && echo "PASS: backstage responding (HTTP $HTTP_CODE)" || echo "FAIL: HTTP $HTTP_CODE"

echo "PASS: Backstage deployed with PostgreSQL"
```

### 11.7 Commit

```bash
cd ../../..
git add terraform/modules/backstage/ terraform/environments/local/main.tf
git commit -m "ch11: backstage deployment with postgresql

- Terraform module using official backstage/backstage Helm chart
- Base values + environment-specific overrides (local, codespaces)
- PostgreSQL subchart enabled for persistence
- Guest auth for development
- Ingress configured for backstage.localhost
- Port-forward fallback on 7007"
```

---

## Chapter 12 — Prometheus: Metrics Collection & Alerting

**Goal:** Install the kube-prometheus-stack (Prometheus + Grafana + alerting rules) into the observability namespace. Verify it's scraping metrics from all running services.

**Components introduced:** Prometheus, ServiceMonitor, kube-prometheus-stack, alerting rules.

### 12.1 Observability Terraform module

Create `terraform/modules/observability/variables.tf`:

```hcl
variable "namespace" {
  description = "Namespace for observability stack"
  type        = string
  default     = "observability"
}

variable "prometheus_chart_version" {
  description = "kube-prometheus-stack Helm chart version"
  type        = string
  default     = "61.7.0"
}

variable "grafana_admin_password" {
  description = "Grafana admin password"
  type        = string
  default     = "devx-admin"
  sensitive   = true
}

variable "grafana_ingress_host" {
  description = "Hostname for Grafana ingress"
  type        = string
  default     = "grafana.localhost"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "local"
}
```

Create `terraform/modules/observability/main.tf`:

```hcl
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

resource "helm_release" "kube_prometheus_stack" {
  name       = "kube-prometheus-stack"
  namespace  = var.namespace
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "kube-prometheus-stack"
  version    = var.prometheus_chart_version

  wait    = true
  timeout = 600

  # Grafana configuration
  set {
    name  = "grafana.adminPassword"
    value = var.grafana_admin_password
  }

  set {
    name  = "grafana.ingress.enabled"
    value = var.environment == "local" ? "true" : "false"
  }

  set {
    name  = "grafana.ingress.hosts[0]"
    value = var.grafana_ingress_host
  }

  set {
    name  = "grafana.service.type"
    value = "ClusterIP"
  }

  # Resource limits for local dev
  set {
    name  = "prometheus.prometheusSpec.resources.requests.cpu"
    value = "100m"
  }
  set {
    name  = "prometheus.prometheusSpec.resources.requests.memory"
    value = "256Mi"
  }
  set {
    name  = "prometheus.prometheusSpec.resources.limits.cpu"
    value = "500m"
  }
  set {
    name  = "prometheus.prometheusSpec.resources.limits.memory"
    value = "512Mi"
  }

  # Retention
  set {
    name  = "prometheus.prometheusSpec.retention"
    value = "24h"
  }

  # Storage (small for local dev)
  set {
    name  = "prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage"
    value = "2Gi"
  }

  # Scrape all namespaces (needed to discover ArgoCD, Backstage, etc.)
  set {
    name  = "prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues"
    value = "false"
  }
  set {
    name  = "prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues"
    value = "false"
  }

  # Disable components that are heavy and not needed locally
  set {
    name  = "alertmanager.enabled"
    value = "true"
  }

  set {
    name  = "alertmanager.alertmanagerSpec.resources.requests.cpu"
    value = "10m"
  }
  set {
    name  = "alertmanager.alertmanagerSpec.resources.requests.memory"
    value = "32Mi"
  }
}
```

Create `terraform/modules/observability/outputs.tf`:

```hcl
output "release_name" {
  description = "Name of the kube-prometheus-stack release"
  value       = helm_release.kube_prometheus_stack.name
}

output "grafana_url" {
  description = "Grafana access URL"
  value       = var.environment == "local" ? "http://${var.grafana_ingress_host}" : "http://localhost:3000 (port-forward)"
}

output "prometheus_url" {
  description = "Prometheus access URL (port-forward)"
  value       = "http://localhost:9090 (via kubectl port-forward svc/kube-prometheus-stack-prometheus -n ${var.namespace} 9090:9090)"
}
```

### 12.2 Wire into the local environment

Add to `terraform/environments/local/main.tf`:

```hcl
# --- Observability (Prometheus + Grafana) ---
module "observability" {
  source = "../../modules/observability"

  namespace   = "observability"
  environment = var.environment

  depends_on = [module.platform_base]
}
```

### 12.3 Deploy

```bash
cd terraform/environments/local
terraform init -upgrade
terraform apply -auto-approve

# This takes a while — kube-prometheus-stack is a large chart
kubectl get pods -n observability -w
# Wait until all pods are Running/Completed (may take 3-5 minutes)
```

### 12.4 Access Prometheus

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus -n observability 9090:9090 &
PROM_PF_PID=$!
sleep 3

# Check Prometheus targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets | length'
# Should show multiple active scrape targets

# Query a metric
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result | length'
# Shows how many targets are reporting "up"

kill $PROM_PF_PID 2>/dev/null
```

### 12.5 Verification

```bash
# Prometheus pod is running
kubectl get pods -n observability -l app.kubernetes.io/name=prometheus --no-headers | grep Running > /dev/null \
  && echo "PASS: prometheus running" || echo "FAIL"

# Grafana pod is running
kubectl get pods -n observability -l app.kubernetes.io/name=grafana --no-headers | grep Running > /dev/null \
  && echo "PASS: grafana running" || echo "FAIL"

# Alertmanager pod is running
kubectl get pods -n observability -l app.kubernetes.io/name=alertmanager --no-headers | grep Running > /dev/null \
  && echo "PASS: alertmanager running" || echo "FAIL"

# Prometheus is scraping targets
kubectl port-forward svc/kube-prometheus-stack-prometheus -n observability 9090:9090 &
PROM_PF_PID=$!
sleep 3
TARGET_COUNT=$(curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets | length')
kill $PROM_PF_PID 2>/dev/null
[ "$TARGET_COUNT" -gt 0 ] && echo "PASS: prometheus scraping $TARGET_COUNT targets" || echo "FAIL"

echo "PASS: Observability stack operational"
```

### 12.6 Commit

```bash
cd ../../..
git add terraform/modules/observability/ terraform/environments/local/main.tf
git commit -m "ch12: prometheus + grafana via kube-prometheus-stack

- Terraform module for kube-prometheus-stack Helm chart
- Prometheus configured to scrape all namespaces
- Grafana with ingress on grafana.localhost
- Alertmanager enabled
- 24h retention, 2Gi storage for local dev
- Resource limits tuned for local development"
```

---

## Chapter 13 — Grafana: Dashboards & Visualization

**Goal:** Access Grafana, explore built-in Kubernetes dashboards, and understand the datasource configuration. Grafana was deployed in Chapter 12 as part of kube-prometheus-stack — this chapter focuses on using it.

**Components introduced:** Grafana UI, datasources, dashboard exploration.

### 13.1 Access Grafana

```bash
# Via ingress
curl -s -H "Host: grafana.localhost" http://localhost | head -5

# Or via port-forward
kubectl port-forward svc/kube-prometheus-stack-grafana -n observability 3000:80 &
GRAF_PF_PID=$!
sleep 3

# Login credentials
echo "URL: http://localhost:3000"
echo "Username: admin"
echo "Password: devx-admin"

curl -s http://localhost:3000/api/health | jq .
# Should show: {"commit":"...","database":"ok","version":"..."}
```

### 13.2 Explore built-in dashboards

Open `http://localhost:3000` in a browser (or the Codespaces forwarded port). Log in with `admin` / `devx-admin`.

Navigate to **Dashboards** — kube-prometheus-stack ships with many pre-built dashboards:

- **Kubernetes / Compute Resources / Cluster** — cluster-wide CPU, memory, network
- **Kubernetes / Compute Resources / Namespace (Pods)** — per-namespace breakdown
- **Kubernetes / Networking / Cluster** — network I/O across the cluster
- **Node Exporter / Nodes** — host-level metrics from k3s nodes

### 13.3 Verify Prometheus as a datasource

```bash
# Check datasources via API
curl -s -u admin:devx-admin http://localhost:3000/api/datasources | jq '.[].name'
# Should show: "Prometheus"

# Query Prometheus through Grafana
curl -s -u admin:devx-admin \
  'http://localhost:3000/api/datasources/proxy/1/api/v1/query?query=up' | jq '.data.result | length'

kill $GRAF_PF_PID 2>/dev/null
```

### 13.4 Verify all platform metrics are flowing

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus -n observability 9090:9090 &
PROM_PF_PID=$!
sleep 3

# Check for ArgoCD metrics
curl -s 'http://localhost:9090/api/v1/query?query=argocd_app_info' | jq '.data.result | length'

# Check for Kubernetes node metrics
curl -s 'http://localhost:9090/api/v1/query?query=node_cpu_seconds_total' | jq '.data.result | length'

# Check for container metrics (our running workloads)
curl -s 'http://localhost:9090/api/v1/query?query=container_memory_working_set_bytes{namespace="backstage"}' | jq '.data.result | length'

kill $PROM_PF_PID 2>/dev/null
```

### 13.5 Verification

```bash
# Grafana health check
kubectl port-forward svc/kube-prometheus-stack-grafana -n observability 3000:80 &
GRAF_PF_PID=$!
sleep 3

GRAF_HEALTH=$(curl -s http://localhost:3000/api/health | jq -r '.database')
[ "$GRAF_HEALTH" = "ok" ] && echo "PASS: grafana healthy" || echo "FAIL"

# Datasource configured
DS_COUNT=$(curl -s -u admin:devx-admin http://localhost:3000/api/datasources | jq 'length')
[ "$DS_COUNT" -ge 1 ] && echo "PASS: $DS_COUNT datasource(s) configured" || echo "FAIL"

# Dashboards available
DASH_COUNT=$(curl -s -u admin:devx-admin 'http://localhost:3000/api/search?type=dash-db' | jq 'length')
[ "$DASH_COUNT" -gt 0 ] && echo "PASS: $DASH_COUNT dashboards available" || echo "FAIL"

kill $GRAF_PF_PID 2>/dev/null

echo "PASS: Grafana operational with Prometheus datasource"
```

### 13.6 Commit

```bash
git add -A
git commit -m "ch13: grafana dashboards and visualization

- Verified Grafana UI access (ingress + port-forward)
- Prometheus auto-configured as datasource
- Built-in Kubernetes dashboards available
- Metrics flowing from all namespaces (backstage, argocd, etc.)
- Login: admin / devx-admin"
```

---

## Chapter 14 — Networking & Ingress: Traefik + Port Mapping

**Goal:** Understand how all the networking pieces fit together — k3d port mapping, Traefik ingress controller, `*.localhost` DNS, and Codespaces port forwarding. Configure ingress for all services.

**Components introduced:** Traefik IngressRoutes, service mesh, DNS resolution.

### 14.1 How it all connects

```
Browser → localhost:80 → k3d loadbalancer → Traefik pod → Ingress rules → Service → Pod
```

k3d maps port 80 on your machine to port 80 on the k3d loadbalancer container. Traefik runs inside k3s and reads Kubernetes Ingress resources to route traffic.

### 14.2 View all ingress rules

```bash
kubectl get ingress -A
```

You should see ingresses for Backstage and Grafana. ArgoCD and Prometheus are accessed via port-forward (they don't have ingress by default in our setup).

### 14.3 Test all service access

```bash
echo "=== Testing service access ==="

# Backstage via ingress
HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' -H "Host: backstage.localhost" http://localhost)
echo "Backstage (ingress): HTTP $HTTP_CODE"

# Grafana via ingress
HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' -H "Host: grafana.localhost" http://localhost)
echo "Grafana (ingress): HTTP $HTTP_CODE"

# ArgoCD via port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
PF1=$!
sleep 3
HTTP_CODE=$(curl -sk -o /dev/null -w '%{http_code}' https://localhost:8080)
echo "ArgoCD (port-forward): HTTP $HTTP_CODE"
kill $PF1 2>/dev/null

# Prometheus via port-forward
kubectl port-forward svc/kube-prometheus-stack-prometheus -n observability 9090:9090 &
PF2=$!
sleep 3
HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:9090)
echo "Prometheus (port-forward): HTTP $HTTP_CODE"
kill $PF2 2>/dev/null
```

### 14.4 Understand Codespaces networking

In Codespaces, `*.localhost` doesn't resolve to anything useful. Instead:

- Ports are auto-forwarded to `https://<codespace>-<port>.app.github.dev`
- Ingress-based routing won't work — use port-forward for all services
- The bootstrap script (Chapter 15) will detect Codespaces and adjust

### 14.5 Create a networking reference document

Create `docs/NETWORKING.md`:

```markdown
# Networking Reference

## Service Access

### Local (macOS / Linux)

| Service    | URL                          | Method        |
|------------|------------------------------|---------------|
| Backstage  | http://backstage.localhost   | Traefik ingress |
| Backstage  | http://localhost:7007        | port-forward  |
| Grafana    | http://grafana.localhost     | Traefik ingress |
| Grafana    | http://localhost:3000        | port-forward  |
| ArgoCD     | https://localhost:8080       | port-forward  |
| Prometheus | http://localhost:9090        | port-forward  |
| Registry   | http://localhost:5111        | direct        |
| K8s API    | https://localhost:6443       | kubeconfig    |

### GitHub Codespaces

All services accessed via port-forwarding. Codespaces auto-forwards ports and provides HTTPS URLs.

| Service    | Port  | Codespace URL pattern                         |
|------------|-------|----------------------------------------------|
| Backstage  | 7007  | https://CODESPACE-7007.app.github.dev        |
| Grafana    | 3000  | https://CODESPACE-3000.app.github.dev        |
| ArgoCD     | 8080  | https://CODESPACE-8080.app.github.dev        |
| Prometheus | 9090  | https://CODESPACE-9090.app.github.dev        |

## How It Works

### k3d Port Mapping

k3d creates a loadbalancer container that maps host ports to cluster ports:
- Host :80 → Loadbalancer :80 → Traefik → Ingress rules
- Host :443 → Loadbalancer :443 → Traefik → Ingress rules

### Traefik Ingress Controller

k3s ships Traefik as the default ingress controller. It watches for Kubernetes
Ingress resources and configures routes automatically.

### *.localhost DNS

On most systems, `*.localhost` resolves to `127.0.0.1` without any
/etc/hosts configuration. This lets us use `backstage.localhost`,
`grafana.localhost`, etc., for local development.

### Port Forwarding

For services without ingress (ArgoCD, Prometheus), use `kubectl port-forward`:
```
kubectl port-forward svc/SERVICE -n NAMESPACE LOCAL_PORT:REMOTE_PORT
```
```

### 14.6 Verification

```bash
# At least 2 ingress resources exist
INGRESS_COUNT=$(kubectl get ingress -A --no-headers 2>/dev/null | wc -l)
[ "$INGRESS_COUNT" -ge 2 ] && echo "PASS: $INGRESS_COUNT ingress resources" || echo "INFO: $INGRESS_COUNT ingresses (some may use port-forward only)"

# Traefik is running
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik --no-headers | grep Running > /dev/null \
  && echo "PASS: traefik running" || echo "FAIL"

echo "PASS: Networking documented and verified"
```

### 14.7 Commit

```bash
mkdir -p docs
git add docs/NETWORKING.md
git commit -m "ch14: networking and ingress documentation

- Documented all service access methods (ingress + port-forward)
- Explained k3d port mapping → Traefik → Ingress chain
- Codespaces networking reference with port patterns
- Verified Traefik and ingress resources for all services"
```

---

## Chapter 15 — Automation: Makefile, Bootstrap & Verify Scripts

**Goal:** Build the automation layer — scripts that detect the environment, bootstrap everything, verify health, and tear it down. The Makefile provides the developer-facing interface.

**Components introduced:** Bootstrap scripts, health checks, idempotent operations.

### 15.1 Bootstrap script

Create `scripts/bootstrap.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

info()  { echo -e "${GREEN}[INFO]${NC} $*"; }
warn()  { echo -e "${YELLOW}[WARN]${NC} $*"; }
error() { echo -e "${RED}[ERROR]${NC} $*"; exit 1; }

# --- 1. Detect environment ---
if [ -n "${CODESPACES:-}" ]; then
  ENVIRONMENT="codespaces"
  info "Detected GitHub Codespaces environment"
else
  ENVIRONMENT="local"
  info "Detected local environment"
fi

# --- 2. Check prerequisites ---
info "Checking prerequisites..."
MISSING=""
for cmd in docker k3d kubectl helm terraform; do
  if ! command -v "$cmd" &> /dev/null; then
    MISSING="$MISSING $cmd"
  fi
done

if [ -n "$MISSING" ]; then
  error "Missing required tools:$MISSING"
fi
info "All prerequisites found"

# --- 3. Check Docker is running ---
if ! docker info &> /dev/null; then
  error "Docker is not running. Please start Docker first."
fi
info "Docker is running"

# --- 4. Terraform init + apply ---
TF_DIR="terraform/environments/${ENVIRONMENT}"
info "Using Terraform environment: $TF_DIR"

cd "$TF_DIR"

info "Running terraform init..."
terraform init -input=false

info "Running terraform apply..."
terraform apply -auto-approve -input=false

cd - > /dev/null

# --- 5. Wait for all pods to be ready ---
info "Waiting for pods to be ready (timeout: 5 minutes)..."

NAMESPACES=("crossplane-system" "argocd" "backstage" "observability")
for ns in "${NAMESPACES[@]}"; do
  info "  Waiting for pods in $ns..."
  kubectl wait --for=condition=ready pod \
    --all -n "$ns" \
    --timeout=300s 2>/dev/null || warn "  Some pods in $ns may not be ready yet"
done

# --- 6. Print access URLs ---
echo ""
info "========================================="
info "  DevX Platform is ready!"
info "========================================="
echo ""

if [ "$ENVIRONMENT" = "codespaces" ]; then
  CODESPACE_URL="https://${CODESPACE_NAME}"
  info "Backstage:  ${CODESPACE_URL}-7007.app.github.dev"
  info "ArgoCD:     ${CODESPACE_URL}-8080.app.github.dev"
  info "Grafana:    ${CODESPACE_URL}-3000.app.github.dev"
  info "Prometheus: ${CODESPACE_URL}-9090.app.github.dev"
else
  info "Backstage:  http://backstage.localhost  (or port-forward 7007)"
  info "ArgoCD:     https://localhost:8080       (port-forward)"
  info "Grafana:    http://grafana.localhost     (or port-forward 3000)"
  info "Prometheus: http://localhost:9090        (port-forward)"
fi

echo ""
info "ArgoCD admin password:"
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" 2>/dev/null | base64 -d && echo ""
info "Grafana login: admin / devx-admin"
echo ""

# --- 7. Run verify ---
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
if [ -f "${SCRIPT_DIR}/verify.sh" ]; then
  info "Running verification..."
  bash "${SCRIPT_DIR}/verify.sh"
fi
```

### 15.2 Teardown script

Create `scripts/teardown.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

info()  { echo -e "${GREEN}[INFO]${NC} $*"; }
error() { echo -e "${RED}[ERROR]${NC} $*"; exit 1; }

# Detect environment
if [ -n "${CODESPACES:-}" ]; then
  ENVIRONMENT="codespaces"
else
  ENVIRONMENT="local"
fi

TF_DIR="terraform/environments/${ENVIRONMENT}"
info "Tearing down environment: $ENVIRONMENT"

if [ -d "$TF_DIR/.terraform" ]; then
  cd "$TF_DIR"
  info "Running terraform destroy..."
  terraform destroy -auto-approve -input=false
  cd - > /dev/null
else
  info "No Terraform state found. Attempting manual cleanup..."
  k3d cluster delete devx 2>/dev/null || true
fi

# Verify cleanup
if k3d cluster list 2>/dev/null | grep -q devx; then
  error "Cluster 'devx' still exists after teardown"
fi

info "Teardown complete"
```

### 15.3 Verify script

Create `scripts/verify.sh`:

```bash
#!/usr/bin/env bash
set -uo pipefail

GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

PASS=0
FAIL=0

check() {
  local name="$1"
  shift
  if "$@" > /dev/null 2>&1; then
    echo -e "  ${GREEN}✓${NC} $name"
    ((PASS++))
  else
    echo -e "  ${RED}✗${NC} $name"
    ((FAIL++))
  fi
}

echo "=== DevX Platform Health Check ==="
echo ""

# Cluster
echo "Cluster:"
check "k3d cluster exists" k3d cluster list -o json | grep -q devx
check "node is Ready" kubectl get nodes --no-headers | grep -q " Ready"
check "registry accessible" curl -sf http://localhost:5111/v2/_catalog

echo ""

# Namespaces
echo "Namespaces:"
for ns in backstage devx-apps observability crossplane-system argocd; do
  check "$ns namespace" kubectl get namespace "$ns"
done

echo ""

# Crossplane
echo "Crossplane:"
check "crossplane pods running" kubectl get pods -n crossplane-system --no-headers | grep -v Completed | grep -q Running
check "provider-kubernetes healthy" kubectl get providers provider-kubernetes -o jsonpath='{.status.conditions[?(@.type=="Healthy")].status}' | grep -q True

echo ""

# ArgoCD
echo "ArgoCD:"
check "argocd-server running" kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server --no-headers | grep -q Running

echo ""

# Backstage
echo "Backstage:"
check "backstage pod running" kubectl get pods -n backstage -l app.kubernetes.io/name=backstage --no-headers | grep -q Running
check "postgresql pod running" kubectl get pods -n backstage -l app.kubernetes.io/name=postgresql --no-headers | grep -q Running

echo ""

# Observability
echo "Observability:"
check "prometheus running" kubectl get pods -n observability -l app.kubernetes.io/name=prometheus --no-headers | grep -q Running
check "grafana running" kubectl get pods -n observability -l app.kubernetes.io/name=grafana --no-headers | grep -q Running

echo ""
echo "=== Results: ${PASS} passed, ${FAIL} failed ==="

if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

Make scripts executable:

```bash
chmod +x scripts/bootstrap.sh scripts/teardown.sh scripts/verify.sh
```

### 15.4 Makefile

Create `Makefile`:

```makefile
.PHONY: help up down status verify logs reset plan backstage argocd grafana prometheus

.DEFAULT_GOAL := help

help: ## Show available targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'

up: ## Bootstrap everything (terraform init + apply + verify)
	@./scripts/bootstrap.sh

down: ## Destroy cluster and all resources
	@./scripts/teardown.sh

status: ## Show cluster status and pod health
	@echo "=== Cluster ==="
	@k3d cluster list 2>/dev/null || echo "No clusters found"
	@echo ""
	@echo "=== Nodes ==="
	@kubectl get nodes 2>/dev/null || echo "Cannot reach cluster"
	@echo ""
	@echo "=== Pods (all namespaces) ==="
	@kubectl get pods -A 2>/dev/null || echo "Cannot reach cluster"

verify: ## Run health checks on all services
	@./scripts/verify.sh

logs: ## Tail logs for a service (usage: make logs SVC=backstage NS=backstage)
	@kubectl logs -f -l app.kubernetes.io/name=$(or $(SVC),backstage) -n $(or $(NS),backstage)

reset: ## Destroy and recreate everything
	@./scripts/teardown.sh
	@./scripts/bootstrap.sh

plan: ## Run terraform plan (dry run)
	@cd terraform/environments/$$([ -n "$${CODESPACES:-}" ] && echo codespaces || echo local) && \
		terraform plan

backstage: ## Port-forward Backstage (http://localhost:7007)
	@echo "Backstage available at http://localhost:7007"
	@echo "Press Ctrl+C to stop"
	@kubectl port-forward svc/backstage -n backstage 7007:7007

argocd: ## Port-forward ArgoCD (https://localhost:8080)
	@echo "ArgoCD available at https://localhost:8080"
	@echo "Admin password:"
	@kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo ""
	@echo "Press Ctrl+C to stop"
	@kubectl port-forward svc/argocd-server -n argocd 8080:443

grafana: ## Port-forward Grafana (http://localhost:3000)
	@echo "Grafana available at http://localhost:3000"
	@echo "Login: admin / devx-admin"
	@echo "Press Ctrl+C to stop"
	@kubectl port-forward svc/kube-prometheus-stack-grafana -n observability 3000:80

prometheus: ## Port-forward Prometheus (http://localhost:9090)
	@echo "Prometheus available at http://localhost:9090"
	@echo "Press Ctrl+C to stop"
	@kubectl port-forward svc/kube-prometheus-stack-prometheus -n observability 9090:9090

registry: ## Show local registry URL and contents
	@echo "Registry URL: localhost:5111"
	@echo "Repositories:"
	@curl -s http://localhost:5111/v2/_catalog | jq . 2>/dev/null || echo "(registry not accessible)"
```

### 15.5 Verification

```bash
# Test the Makefile help
make help

# If cluster is running, test verify
make verify
```

### 15.6 Commit

```bash
git add scripts/ Makefile
git commit -m "ch15: automation — makefile, bootstrap, teardown, verify

- scripts/bootstrap.sh: detect env, check prereqs, terraform apply, wait, print URLs
- scripts/teardown.sh: terraform destroy with manual fallback
- scripts/verify.sh: comprehensive health checks for all services
- Makefile: up, down, status, verify, logs, reset, plan, port-forward targets
- All scripts are idempotent and environment-aware"
```

---

## Chapter 16 — Codespaces Environment & End-to-End Verification

**Goal:** Wire up the Codespaces-specific Terraform environment, update the DevContainer to auto-bootstrap, and run the complete end-to-end flow.

**Components introduced:** Codespaces environment detection, postCreateCommand, end-to-end testing.

### 16.1 Update the Codespaces Terraform environment

The Codespaces environment needs Crossplane, ArgoCD, Backstage, and Observability modules — same as local. Update `terraform/environments/codespaces/main.tf`:

```hcl
# --- Cluster ---
module "cluster" {
  source = "../../modules/k3d-cluster"

  cluster_name = var.cluster_name
}

# --- Platform Base (namespaces) ---
module "platform_base" {
  source = "../../modules/platform-base"

  host                   = module.cluster.host
  client_certificate     = module.cluster.client_certificate
  client_key             = module.cluster.client_key
  cluster_ca_certificate = module.cluster.cluster_ca_certificate
}

# --- Crossplane ---
module "crossplane" {
  source = "../../modules/crossplane"

  namespace = "crossplane-system"

  depends_on = [module.platform_base]
}

# --- ArgoCD ---
module "argocd" {
  source = "../../modules/argocd"

  namespace = "argocd"

  depends_on = [module.platform_base]
}

# --- Backstage ---
module "backstage" {
  source = "../../modules/backstage"

  namespace   = "backstage"
  environment = var.environment

  depends_on = [module.platform_base]
}

# --- Observability (Prometheus + Grafana) ---
module "observability" {
  source = "../../modules/observability"

  namespace   = "observability"
  environment = var.environment

  depends_on = [module.platform_base]
}
```

### 16.2 Update DevContainer for auto-bootstrap

Update `.devcontainer/devcontainer.json` — add the `postCreateCommand`:

```json
{
  "name": "DevX Platform",
  "build": {
    "dockerfile": "Dockerfile"
  },
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {
      "kubectl": "latest",
      "helm": "latest",
      "minikube": "none"
    }
  },
  "postCreateCommand": "./scripts/bootstrap.sh",
  "forwardPorts": [80, 443, 3000, 7007, 8080, 9090],
  "portsAttributes": {
    "80":   { "label": "HTTP Ingress" },
    "443":  { "label": "HTTPS Ingress" },
    "3000": { "label": "Grafana", "onAutoForward": "notify" },
    "7007": { "label": "Backstage", "onAutoForward": "notify" },
    "8080": { "label": "ArgoCD", "onAutoForward": "notify" },
    "9090": { "label": "Prometheus" }
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "hashicorp.terraform",
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "redhat.vscode-yaml",
        "tim-koehler.helm-intellisense"
      ]
    }
  },
  "remoteUser": "vscode"
}
```

### 16.3 End-to-end test (local)

From a clean state:

```bash
# Start from scratch
make down 2>/dev/null || true

# Full bootstrap
make up

# Verify everything
make verify

# Test each service
make backstage &  # Ctrl+C after testing
make argocd &     # Ctrl+C after testing
make grafana &    # Ctrl+C after testing

# Clean teardown
make down
```

### 16.4 End-to-end test (Codespaces)

1. Push all code to GitHub: `git push origin main`
2. Go to the repo on GitHub
3. Click **Code → Codespaces → Create codespace on main**
4. Wait for the DevContainer to build and `postCreateCommand` to finish
5. The terminal should show all services coming up
6. Open the **Ports** tab in VS Code — you should see forwarded ports
7. Click the Backstage port (7007) to open in browser
8. Run `make verify` in the terminal

### 16.5 Final verification checklist

```bash
echo "=== Final End-to-End Verification ==="
echo ""

# 1. Cluster
echo "1. Cluster"
k3d cluster list | grep devx && echo "   ✓ Cluster exists" || echo "   ✗ FAIL"

# 2. All namespaces
echo "2. Namespaces"
for ns in backstage devx-apps observability crossplane-system argocd; do
  kubectl get ns "$ns" > /dev/null 2>&1 && echo "   ✓ $ns" || echo "   ✗ $ns MISSING"
done

# 3. All core pods running
echo "3. Core Services"
for pair in "crossplane-system:crossplane" "argocd:argocd-server" "backstage:backstage" "backstage:postgresql" "observability:prometheus" "observability:grafana"; do
  NS="${pair%%:*}"
  APP="${pair##*:}"
  kubectl get pods -n "$NS" -l "app.kubernetes.io/name=$APP" --no-headers 2>/dev/null | grep -q Running \
    && echo "   ✓ $APP in $NS" || echo "   ✗ $APP in $NS NOT RUNNING"
done

# 4. Registry
echo "4. Registry"
curl -sf http://localhost:5111/v2/_catalog > /dev/null && echo "   ✓ Registry accessible" || echo "   ✗ FAIL"

# 5. Crossplane provider
echo "5. Crossplane"
kubectl get providers provider-kubernetes -o jsonpath='{.status.conditions[?(@.type=="Healthy")].status}' 2>/dev/null | grep -q True \
  && echo "   ✓ provider-kubernetes healthy" || echo "   ✗ provider-kubernetes NOT healthy"

echo ""
echo "=== Full stack verification complete ==="
```

### 16.6 Commit

```bash
git add -A
git commit -m "ch16: codespaces environment and end-to-end verification

- Codespaces Terraform environment with all modules
- DevContainer postCreateCommand runs bootstrap automatically
- Port auto-forwarding with notify for key services
- End-to-end verification for both local and Codespaces
- Full stack: k3d + Crossplane + ArgoCD + Backstage + Prometheus + Grafana"
```

---

## Summary

You've built a complete internal developer platform from scratch:

| Chapter | Component | What you learned |
|---------|-----------|-----------------|
| 1  | DevContainer | Reproducible dev environments, tooling installation |
| 2  | k3d (manual) | Kubernetes fundamentals, k3s architecture, registries |
| 3  | Terraform + k3d | IaC basics, providers, modules, plan/apply/destroy |
| 4  | Namespaces module | Kubernetes provider, module inputs/outputs |
| 5  | Environment composition | Module wiring, multi-environment Terraform |
| 6  | Helm chart | Chart structure, templates, values, install lifecycle |
| 7  | Crossplane | Infrastructure control plane, Providers, Managed Resources |
| 8  | Crossplane Compositions | XRDs, Compositions, Claims — platform API design |
| 9  | ArgoCD | GitOps delivery, Application CRD, sync + self-heal |
| 10 | App-of-Apps | Multi-app management, declarative app deployment |
| 11 | Backstage | Developer portal, PostgreSQL, Helm subchart |
| 12 | Prometheus | Metrics collection, ServiceMonitors, alerting |
| 13 | Grafana | Dashboards, datasources, visualization |
| 14 | Networking | Traefik ingress, k3d port mapping, DNS |
| 15 | Automation | Makefile, bootstrap/teardown/verify scripts |
| 16 | Codespaces | Environment portability, end-to-end testing |

### Architecture

```
┌─────────────────────────────────────────────────┐
│                  Developer                       │
│         (Codespaces or Local macOS)              │
└───────────────┬─────────────────────────────────┘
                │ make up / make down
                ▼
┌─────────────────────────────────────────────────┐
│              Terraform (IaC)                     │
│  ┌──────────┐ ┌────────────┐ ┌───────────────┐  │
│  │ k3d-     │ │ platform-  │ │ Helm releases │  │
│  │ cluster  │ │ base       │ │ (all modules) │  │
│  └──────────┘ └────────────┘ └───────────────┘  │
└───────────────┬─────────────────────────────────┘
                ▼
┌─────────────────────────────────────────────────┐
│          k3d Cluster (k3s in Docker)             │
│                                                  │
│  ┌─────────────┐  ┌──────────────┐              │
│  │  Traefik    │  │   Registry   │              │
│  │  (ingress)  │  │ :5111        │              │
│  └─────────────┘  └──────────────┘              │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ crossplane-system                         │   │
│  │  Crossplane + provider-kubernetes         │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │ argocd                                    │   │
│  │  ArgoCD Server + Repo Server + Redis      │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │ backstage                                 │   │
│  │  Backstage + PostgreSQL                   │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │ observability                             │   │
│  │  Prometheus + Grafana + Alertmanager      │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │ devx-apps                                 │   │
│  │  (your applications deployed via ArgoCD)  │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Next Steps

- **Connect Backstage to ArgoCD**: Install the ArgoCD plugin in a custom Backstage image to see deployments in the portal
- **Connect Backstage to Crossplane**: Create Software Templates that submit Crossplane Claims for self-service infrastructure
- **Add cloud providers to Crossplane**: Install `provider-aws` or `provider-gcp` for real cloud resource provisioning
- **GitOps everything**: Move Crossplane, Backstage, and Observability deployments from Terraform Helm releases to ArgoCD Applications
- **Custom Backstage image**: Build a Backstage image with your organization's plugins, catalog, and templates
