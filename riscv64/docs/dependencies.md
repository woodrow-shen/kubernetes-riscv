# Dependency Status Matrix

Status of all components required for a functional Kubernetes cluster on
`linux/riscv64` with Debian Trixie (13).

## Go Toolchain

| Component | Version | riscv64 Status |
|-----------|---------|---------------|
| Go compiler | 1.27.1 | First-class port since Go 1.21 |
| CGO | - | Not needed since v1.38 (all binaries static); `riscv64-linux-gnu-gcc` only for pause |

## Container Runtime

| Component | Version | riscv64 Status | Source |
|-----------|---------|---------------|--------|
| containerd | 2.x | Official riscv64 binaries | [github.com/containerd/containerd](https://github.com/containerd/containerd/releases) |
| runc | 1.2.x | Official riscv64 binaries | [github.com/opencontainers/runc](https://github.com/opencontainers/runc/releases) |
| crun | 1.x | Available in Debian Trixie | `apt-get install crun` |
| CNI plugins | 1.6.x | Official riscv64 binaries | [github.com/containernetworking/plugins](https://github.com/containernetworking/plugins/releases) |

## Kubernetes Components

| Component | Build Method | CGO Required | Notes |
|-----------|-------------|-------------|-------|
| kube-apiserver | `make all` | No | |
| kube-controller-manager | `make all` | No | |
| kube-scheduler | `make all` | No | |
| kube-proxy | `make all` | No | |
| kubelet | `make all` | No | static since v1.38 (upstream 81922ce9b57) |
| kubeadm | `make all` | No | |
| kubectl | `make all` | No | |

Cross builds use `KUBE_BUILD_PLATFORMS=linux/riscv64 make all WHAT=...`;
native builds on riscv64 use plain `make all`. Do not use `make cross` for
specific components -- it ignores `WHAT` (see
[known-issues.md](known-issues.md) and [building.md](building.md)).

## Kubernetes Build Images

| Image | Upstream riscv64 | Community Image |
|-------|-----------------|-----------------|
| kube-cross | Not available | `ghcr.io/riseproject-dev/kube-cross` |
| go-runner | Not available | `ghcr.io/riseproject-dev/go-runner` |
| distroless-iptables | Not available | `ghcr.io/riseproject-dev/distroless-iptables` |
| setcap | Not available | `ghcr.io/riseproject-dev/setcap` |
| pause | Not available | `ghcr.io/riseproject-dev/pause` |

## Distroless Base Images

| Image | riscv64 Status |
|-------|---------------|
| `gcr.io/distroless/static-debian13` | Available |
| `gcr.io/distroless/base-debian13` | Available |
| `gcr.io/distroless/cc-debian13` | Available |
| `gcr.io/distroless/static-debian12` | NOT available |
| `gcr.io/distroless/base-debian12` | NOT available |

Note: Only debian13 (Trixie) distroless images include riscv64 manifests.

The community etcd and CoreDNS images use the root variant
(`gcr.io/distroless/static-debian13:latest`, not `:nonroot`), matching the
upstream images: kubeadm static pods read root-owned paths such as
`/etc/kubernetes/pki` and `/var/lib/etcd`, and CoreDNS binds port 53.

## Ecosystem Components

| Component | Version | riscv64 Status | Notes |
|-----------|---------|---------------|-------|
| etcd | v3.7.1 | Compiles from source | Requires `ETCD_UNSUPPORTED_ARCH=riscv64`; published as `etcd:3.7.1-0` |
| CoreDNS | v1.14.7 | Compiles from source | Pure Go, no special handling; published as `coredns:v1.14.7` |
| Flannel | v0.28.0 | Compiles from source | Includes CNI plugin; published as `flannel:v0.28.0-riscv64` |
| Helm | v3.x | Official riscv64 binaries | [github.com/helm/helm](https://github.com/helm/helm/releases) |
| crictl | v1.x | Compiles from source | CRI debugging tool |

The etcd, CoreDNS, and pause image tags (`etcd:3.7.1-0`, `coredns:v1.14.7`,
`pause:3.10.2`) must track kubeadm's defaults in
`cmd/kubeadm/app/constants/constants.go` so that kubeadm with a custom
`imageRepository` pulls them directly.

## Debian Trixie Packages (riscv64)

All required system packages are available for `riscv64` in Debian Trixie:

| Package | Version | Status |
|---------|---------|--------|
| iptables | 1.8.11 | Available |
| nftables | - | Available |
| ipset | 7.22 | Available |
| conntrack-tools | 1.4.8 | Available |
| iproute2 | 6.15.0 | Available |
| libcap2-bin | 2.75 | Available |
| gcc-riscv64-linux-gnu | - | Available (cross-compiler) |

## QEMU Support

| Component | riscv64 Status |
|-----------|---------------|
| qemu-user-static | Supported (binfmt registered) |
| qemu-system-riscv64 | Supported |
| Docker buildx QEMU | Supported via `multiarch/qemu-user-static` |

The Kubernetes source tree includes QEMU binfmt registration for riscv64
in `third_party/multiarch/qemu-user-static/register/qemu-binfmt-conf.sh`.
