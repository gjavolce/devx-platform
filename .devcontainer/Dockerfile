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