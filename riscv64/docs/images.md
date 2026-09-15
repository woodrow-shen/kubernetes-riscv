# Building Container Images for riscv64

This guide covers building Kubernetes component container images for
`linux/riscv64` using the community dependency images on Debian Trixie.

## Prerequisites

- podman with podman-docker (recommended), or Docker with buildx plugin
- QEMU user-static for cross-architecture builds (if building on amd64):
  ```bash
  podman run --rm --privileged multiarch/qemu-user-static --reset -p yes
  ```
- Kubernetes source tree with riscv64 binaries already built (see [building.md](building.md))

Install podman:
```bash
# Debian/Ubuntu
sudo apt-get install podman podman-docker

# Fedora
sudo dnf install podman podman-docker
```

## Dependency Images

The Kubernetes component images depend on base images. The official base images
do not include riscv64 variants, so we provide community equivalents:

| Upstream Image | Community riscv64 Image | Purpose |
|---------------|------------------------|---------|
| `registry.k8s.io/build-image/go-runner` | `ghcr.io/riseproject-dev/go-runner` | Base for static binaries |
| `registry.k8s.io/build-image/distroless-iptables` | `ghcr.io/riseproject-dev/distroless-iptables` | Base for kube-proxy |
| `registry.k8s.io/build-image/setcap` | `ghcr.io/riseproject-dev/setcap` | Capabilities for kube-apiserver |
| `registry.k8s.io/build-image/kube-cross` | `ghcr.io/riseproject-dev/kube-cross` | Cross-compilation environment |

### Build the dependency images

```bash
# go-runner
docker buildx build --platform linux/riscv64 \
  -t ghcr.io/riseproject-dev/go-runner:v2.4.0-go1.26.2-trixie.0 \
  riscv64/images/go-runner/

# setcap
docker buildx build --platform linux/riscv64 \
  -t ghcr.io/riseproject-dev/setcap:trixie-v1.0.0 \
  riscv64/images/setcap/

# distroless-iptables (depends on go-runner)
docker buildx build --platform linux/riscv64 \
  --build-arg GORUNNERIMAGE=ghcr.io/riseproject-dev/go-runner:v2.4.0-go1.26.2-trixie.0 \
  -t ghcr.io/riseproject-dev/distroless-iptables:v0.9.0-trixie.0 \
  riscv64/images/distroless-iptables/
```

## Building Kubernetes Component Images

Once binaries are built and dependency images are available, build the
component images using the Kubernetes build system with env var overrides:

```bash
export KUBE_BUILD_PLATFORMS=linux/riscv64
export KUBE_DOCKER_REGISTRY=ghcr.io/riseproject-dev
export KUBE_GORUNNER_IMAGE=ghcr.io/riseproject-dev/go-runner:v2.4.0-go1.26.2-trixie.0
export KUBE_PROXY_BASE_IMAGE=ghcr.io/riseproject-dev/distroless-iptables:v0.9.0-trixie.0
export KUBE_APISERVER_BASE_IMAGE=ghcr.io/riseproject-dev/go-runner:v2.4.0-go1.26.2-trixie.0
export KUBE_BUILD_SETCAP_IMAGE=ghcr.io/riseproject-dev/setcap:trixie-v1.0.0

make release-images
```

The kube-controller-manager and kube-scheduler images use the go-runner base
selected by `KUBE_GORUNNER_IMAGE`; there are no per-component base image
overrides for them in `build/common.sh`.

The release scripts name the component images with an architecture suffix
(`kube-apiserver-riscv64:${K8S_VERSION}`), following the upstream
`release.sh` convention. CI additionally tags and pushes unsuffixed aliases
(`kube-apiserver:${K8S_VERSION}`) because kubeadm with
`imageRepository: ghcr.io/riseproject-dev` resolves unsuffixed names:

```bash
K8S_VERSION=v1.36.0
for img in kube-apiserver kube-controller-manager kube-scheduler kube-proxy; do
  docker tag ghcr.io/riseproject-dev/${img}-riscv64:${K8S_VERSION} \
             ghcr.io/riseproject-dev/${img}:${K8S_VERSION}
done
```

### Or build individual component images manually

Each component image is a simple copy of the binary into the base image:

```bash
# Example: kube-apiserver
docker buildx build --platform linux/riscv64 \
  --build-arg BASEIMAGE=ghcr.io/riseproject-dev/go-runner:v2.4.0-go1.26.2-trixie.0 \
  --build-arg BINARY=kube-apiserver \
  -t ghcr.io/riseproject-dev/kube-apiserver:v1.36.0 \
  -f build/server-image/Dockerfile \
  _output/local/bin/linux/riscv64/
```

## Building the Pause Image

The pause image is built from the `build/pause/` directory using the
cross-compilation triple added by the riscv64 patches:

```bash
cd build/pause
make ARCH=riscv64 TAG=3.10.2 REGISTRY=ghcr.io/riseproject-dev
```

This produces `pause:3.10.2-linux-riscv64` (upstream naming). CI also tags
and pushes the unsuffixed alias `pause:3.10.2`, which is the name kubeadm
and containerd's `sandbox_image` expect under the image repository:

```bash
docker tag ghcr.io/riseproject-dev/pause:3.10.2-linux-riscv64 \
           ghcr.io/riseproject-dev/pause:3.10.2
```

This requires the kube-cross image with `riscv64-linux-gnu-gcc`.
Override if needed:

```bash
make ARCH=riscv64 TAG=3.10.2 REGISTRY=ghcr.io/riseproject-dev \
  KUBE_CROSS_IMAGE=ghcr.io/riseproject-dev/kube-cross \
  KUBE_CROSS_VERSION=v1.36.0-go1.26.2-trixie.0
```

## Building Ecosystem Component Images

The etcd and CoreDNS image tags must match kubeadm's defaults
(`DefaultEtcdVersion` and `CoreDNSVersion` in
`cmd/kubeadm/app/constants/constants.go`), so they are published unsuffixed
as `etcd:3.6.8-0` and `coredns:v1.14.2`. Both images use the
`gcr.io/distroless/static-debian13:latest` base (root variant, not
`:nonroot`): the upstream images run as root, kubeadm static pods read
root-owned paths such as `/etc/kubernetes/pki` and `/var/lib/etcd`, and
CoreDNS binds port 53.

```bash
# etcd (tag matches kubeadm's DefaultEtcdVersion)
docker buildx build --platform linux/riscv64 \
  --build-arg ETCD_VERSION=v3.6.8 \
  -t ghcr.io/riseproject-dev/etcd:3.6.8-0 \
  riscv64/ecosystem/etcd/

# CoreDNS (tag matches kubeadm's CoreDNSVersion)
docker buildx build --platform linux/riscv64 \
  --build-arg COREDNS_VERSION=v1.14.2 \
  -t ghcr.io/riseproject-dev/coredns:v1.14.2 \
  riscv64/ecosystem/coredns/

# Flannel
docker buildx build --platform linux/riscv64 \
  --build-arg FLANNEL_VERSION=v0.28.0 \
  -t ghcr.io/riseproject-dev/flannel:v0.28.0-riscv64 \
  riscv64/ecosystem/flannel/
```

## Pushing Images

```bash
# Login to GHCR
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

REGISTRY=ghcr.io/riseproject-dev
K8S_VERSION=v1.36.0
GO_VERSION=$(cat .go-version)

# Kubernetes components: arch-suffixed (upstream convention) plus
# unsuffixed aliases for kubeadm's imageRepository resolution
for img in kube-apiserver kube-controller-manager kube-scheduler kube-proxy; do
  docker push ${REGISTRY}/${img}-riscv64:${K8S_VERSION}
  docker push ${REGISTRY}/${img}:${K8S_VERSION}
done

# Pause: suffixed and unsuffixed
docker push ${REGISTRY}/pause:3.10.2-linux-riscv64
docker push ${REGISTRY}/pause:3.10.2

# Dependency images
docker push ${REGISTRY}/go-runner:v2.4.0-go${GO_VERSION}-trixie.0
docker push ${REGISTRY}/distroless-iptables:v0.9.0-trixie.0
docker push ${REGISTRY}/setcap:trixie-v1.0.0

# Ecosystem (etcd and coredns are unsuffixed only)
docker push ${REGISTRY}/etcd:3.6.8-0
docker push ${REGISTRY}/coredns:v1.14.2
docker push ${REGISTRY}/flannel:v0.28.0-riscv64
```

## Multi-Arch Manifests (Optional)

The unsuffixed `pause:3.10.2` tag is now tagged and pushed automatically by
CI (`.github/workflows/build-riscv64.yml`), so no manual manifest work is
needed for a riscv64-only registry. To instead create a manifest list that
supports both amd64 and riscv64 under the unsuffixed tag:

```bash
docker manifest create ghcr.io/riseproject-dev/pause:3.10.2 \
  ghcr.io/riseproject-dev/pause:3.10.2-linux-amd64 \
  ghcr.io/riseproject-dev/pause:3.10.2-linux-riscv64

docker manifest annotate --os linux --arch riscv64 \
  ghcr.io/riseproject-dev/pause:3.10.2 \
  ghcr.io/riseproject-dev/pause:3.10.2-linux-riscv64

docker manifest push ghcr.io/riseproject-dev/pause:3.10.2
```
