# Build helm-repo-sync: a tiny Go CLI that materializes helm's repositories.yaml
# from ArgoCD repository Secrets, so private helm deps resolve inside the CMP
# sidecar. Pure stdlib, no external Go deps.
FROM golang:1.26-alpine AS build-helm-repo-sync
WORKDIR /src
COPY helm-repo-sync/go.mod helm-repo-sync/main.go ./
RUN CGO_ENABLED=0 go build -ldflags "-s -w" -o /helm-repo-sync .

FROM ubuntu:25.10 AS download
WORKDIR /custom-tools/
COPY  sops-generate.sh .
COPY  --from=build-helm-repo-sync /helm-repo-sync /custom-tools/helm-repo-sync
RUN   apt-get update && apt-get install -y --no-install-recommends curl ca-certificates git && rm -rf /var/lib/apt/lists/*
RUN   curl -fSL --retry 5 --retry-delay 3 https://get.helm.sh/helm-v4.3.2-linux-amd64.tar.gz | tar -xz -C /usr/local/bin --strip-components=1 linux-amd64/helm
RUN   chmod +x  /usr/local/bin/helm /custom-tools/sops-generate.sh /custom-tools/helm-repo-sync

# helm-secrets plugin: install into a fixed, uid-999-readable location.
# Cache/config live in /tmp (mounted in the sidecar, writable under uid 999).
# ENV is set here for `helm plugin install` AND re-declared in the final stage
# (multi-stage does not carry ENV across stages).
ENV HELM_DATA_HOME=/opt/helm \
  HELM_CACHE_HOME=/tmp/helm-cache \
  HELM_CONFIG_HOME=/tmp/helm-config \
  XDG_CACHE_HOME=/tmp/helm-cache \
  XDG_CONFIG_HOME=/tmp/helm-config
RUN   mkdir -p /opt/helm-plugins && \
  helm plugin install --verify=false https://github.com/jkroepke/helm-secrets/releases/download/v4.7.4/secrets-4.7.4.tgz

FROM akopyanlabs/argocd-hardened:v3.6.1

COPY --from=download /custom-tools/ /usr/local/bin/
COPY --from=download /opt/helm/plugins /opt/helm/plugins
# Script/image version, surfaced by `sops-generate.sh --version` and in every
# generate log line. Pass at build time, e.g.:
#   docker build --build-arg CMP_SOPS_VERSION=$(git describe --tags --always) ...
ARG CMP_SOPS_VERSION=dev
ENV CMP_SOPS_VERSION=${CMP_SOPS_VERSION}
# Re-declare ALL helm env vars in the final stage (ENV does not carry across
# multi-stage builds). Without HELM_CONFIG_HOME helm looks for repositories.yaml
# under the default ~/.config/helm and never finds the one helm-repo-sync writes,
# so private helm dependencies would fail to resolve.
ENV HELM_DATA_HOME=/opt/helm  \
  HELM_CACHE_HOME=/tmp/helm-cache \
  HELM_CONFIG_HOME=/tmp/helm-config \
  XDG_CACHE_HOME=/tmp/helm-cache \
  XDG_CONFIG_HOME=/tmp/helm-config
