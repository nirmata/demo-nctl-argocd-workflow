FROM rust:1-slim-buster AS build

# https://docs.docker.com/reference/dockerfile/#automatic-platform-args-in-the-global-scope
ARG TARGETARCH

# create a new empty shell project
RUN USER=root cargo new --bin argocd-diff-preview
WORKDIR /argocd-diff-preview

# copy over your manifests
COPY ./Cargo.lock ./Cargo.lock
COPY ./Cargo.toml ./Cargo.toml

# this build step will cache your dependencies
RUN cargo build --release
RUN rm src/*.rs

# copy your source tree
COPY ./src ./src

# install kind
RUN apt-get update && apt-get install -y curl
RUN curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.26.0/kind-linux-${TARGETARCH}
RUN chmod +x ./kind

# install helm
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install kubectl
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/${TARGETARCH}/kubectl \
    && chmod +x ./kubectl

# Install Argo CD
RUN curl -sSL -o argocd-linux-${TARGETARCH} https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-${TARGETARCH}
RUN install -m 555 argocd-linux-${TARGETARCH} /usr/local/bin/argocd
RUN rm argocd-linux-${TARGETARCH}

# run test
RUN cargo test

# build for release
RUN rm ./target/release/deps/argocd_diff_preview-*
RUN cargo build --release

# our final base
FROM debian:bookworm-slim

COPY --from=docker:dind /usr/local/bin/docker /usr/local/bin/

COPY --from=build /argocd-diff-preview/kind /usr/local/bin/kind
COPY --from=build /argocd-diff-preview/kubectl /usr/local/bin/kubectl
COPY --from=build /usr/local/bin/helm /usr/local/bin/helm
COPY --from=build /usr/local/bin/argocd /usr/local/bin/argocd
COPY --from=build /argocd-diff-preview/target/release/argocd-diff-preview .

RUN apt-get update && \
    apt-get install -y git && \
    rm -rf /var/lib/apt/lists/*

# copy argocd helm chart values
COPY ./argocd-config ./argocd-config

# set the startup command to run your binary
ENTRYPOINT ["./argocd-diff-preview"]
