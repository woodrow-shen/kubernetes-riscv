# CoreDNS for linux/riscv64 on Debian Trixie
#
# CoreDNS is a pure Go application that compiles cleanly for riscv64.
#
# Build:
#   docker buildx build --platform linux/riscv64 \
#     --build-arg COREDNS_VERSION=v1.14.7 \
#     -t ghcr.io/riseproject-dev/coredns:v1.14.7 .
#
# The image tag must match kubeadm's CoreDNSVersion (e.g. v1.14.7) so that
# `kubeadm init --image-repository ghcr.io/riseproject-dev` resolves it.

ARG GO_VERSION=1.27.1
# Root variant required: CoreDNS binds :53 and Kubernetes does not grant
# ambient capabilities to non-root container users, matching the upstream
# coredns/coredns image which also runs as root.
ARG BASEIMAGE=gcr.io/distroless/static-debian13:latest

# Stage 1: Build CoreDNS from source
FROM --platform=$BUILDPLATFORM golang:${GO_VERSION}-trixie AS builder

ARG COREDNS_VERSION=v1.14.7
ARG TARGETOS=linux
ARG TARGETARCH=riscv64

RUN git clone --depth 1 --branch ${COREDNS_VERSION} \
    https://github.com/coredns/coredns.git /coredns

WORKDIR /coredns

RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -ldflags="-s -w -X github.com/coredns/coredns/coremain.GitCommit=$(git rev-parse --short HEAD)" \
    -o /out/coredns .

# Stage 2: Final image
FROM ${BASEIMAGE}

COPY --from=builder /out/coredns /coredns

EXPOSE 53 53/udp

ENTRYPOINT ["/coredns"]
