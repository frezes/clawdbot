# ---------- Build stage ----------
FROM debian:bookworm-slim AS builder

# Do not specify default value as docker buildx will not set the arguments otherwise, see https://github.com/docker/buildx/issues/510
ARG TARGETOS
ARG TARGETARCH

ARG KUBECTL_VERSION=v1.33.4
ARG HELM_VERSION=v3.18.5


RUN  apt-get update && \
     DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
     ca-certificates curl tar gzip coreutils grep

WORKDIR /tmp

# ----- Download and verify Helm -----
RUN curl -fsSLO https://get.helm.sh/helm-${HELM_VERSION}-${TARGETOS}-${TARGETARCH}.tar.gz && \
    curl -fsSLO https://get.helm.sh/helm-${HELM_VERSION}-${TARGETOS}-${TARGETARCH}.tar.gz.sha256 && \
    echo "$(cat helm-${HELM_VERSION}-${TARGETOS}-${TARGETARCH}.tar.gz.sha256)  helm-${HELM_VERSION}-${TARGETOS}-${TARGETARCH}.tar.gz" | sha256sum -c - && \
    tar xf helm-${HELM_VERSION}-${TARGETOS}-${TARGETARCH}.tar.gz && \
    mv ${TARGETOS}-${TARGETARCH}/helm /tmp/helm

# ----- Download and verify kubectl -----
RUN curl -fsSLo /tmp/kubectl https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/${TARGETOS}/${TARGETARCH}/kubectl && \
    curl -fsSLo /tmp/kubectl.sha256 https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/${TARGETOS}/${TARGETARCH}/kubectl.sha256 && \
    echo "$(cat /tmp/kubectl.sha256)  /tmp/kubectl" | sha256sum -c - && \
    chmod +x /tmp/kubectl



FROM ghcr.io/openclaw/openclaw:2026.3.24

COPY --from=builder /tmp/helm /usr/local/bin/helm
COPY --from=builder /tmp/kubectl /usr/local/bin/kubectl

# Install additional system packages needed by your skills or extensions.
# Example: docker build --build-arg CUSTOM_PACKAGES="python3 wget" .
ARG CUSTOM_PACKAGES=""
RUN --mount=type=cache,id=openclaw-bookworm-apt-cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,id=openclaw-bookworm-apt-lists,target=/var/lib/apt,sharing=locked \
    if [ -n "$CUSTOM_PACKAGES" ]; then \
      apt-get update && \
      DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends $CUSTOM_PACKAGES; \
    fi
