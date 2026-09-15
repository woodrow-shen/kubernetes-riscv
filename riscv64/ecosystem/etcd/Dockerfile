# etcd for linux/riscv64 on Debian Trixie
#
# etcd does not officially support riscv64, but compiles and runs with
# ETCD_UNSUPPORTED_ARCH=riscv64 set at runtime.
#
# Build:
#   docker buildx build --platform linux/riscv64 \
#     --build-arg ETCD_VERSION=v3.6.8 \
#     -t ghcr.io/riseproject-dev/etcd:3.6.8-0 .
#
# The image tag must match kubeadm's DefaultEtcdVersion (e.g. 3.6.8-0) so
# that `kubeadm init --image-repository ghcr.io/riseproject-dev` resolves it.

ARG GO_VERSION=1.26.2
# Root variant required: kubeadm's etcd static pod writes to a root-owned
# hostPath (/var/lib/etcd), matching upstream registry.k8s.io/etcd.
ARG BASEIMAGE=gcr.io/distroless/static-debian13:latest

# Stage 1: Build etcd from source
FROM --platform=$BUILDPLATFORM golang:${GO_VERSION}-trixie AS builder

ARG ETCD_VERSION=v3.6.8
ARG TARGETOS=linux
ARG TARGETARCH=riscv64

RUN git clone --depth 1 --branch ${ETCD_VERSION} \
    https://github.com/etcd-io/etcd.git /etcd

WORKDIR /etcd

RUN mkdir -p /out && \
    cd server  && CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -o /out/etcd .    && cd .. && \
    cd etcdctl && CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -o /out/etcdctl . && cd .. && \
    cd etcdutl && CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -o /out/etcdutl . && cd ..

# Stage 2: Final image
FROM ${BASEIMAGE}

COPY --from=builder /out/etcd /usr/local/bin/etcd
COPY --from=builder /out/etcdctl /usr/local/bin/etcdctl
COPY --from=builder /out/etcdutl /usr/local/bin/etcdutl

ENV ETCD_UNSUPPORTED_ARCH=riscv64

EXPOSE 2379 2380

ENTRYPOINT ["/usr/local/bin/etcd"]
