# Building Kubernetes Binaries for riscv64

This guide covers three methods for building Kubernetes binaries for `linux/riscv64`.

## Prerequisites

All methods require:
- Git
- Go 1.27.1+ (must match the `.go-version` in the Kubernetes source tree)
- The Kubernetes source checked out at the target version

```bash
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes
git checkout v1.38.0  # or your target version
```

## Method 1: Cross-Compile from amd64/arm64 (Recommended)

The simplest approach. Builds on your existing x86_64 or arm64 workstation.

### Toolchain

No C cross-compiler is required for the Kubernetes binaries: as of the
v1.38 development cycle every component, including kubelet, is built as
a static CGO-free binary upstream (kubernetes/kubernetes commit
81922ce9b57, "Build kubelet as a static binary"). A Go toolchain
matching `.go-version` is all you need.

The C toolchain below is only needed if you also build the pause image
(a small static C program):

```bash
# Debian/Ubuntu (gcc plus static glibc for linking pause)
sudo apt-get install gcc-riscv64-linux-gnu libc6-dev-riscv64-cross
```

### Build all server binaries

Use `make all` (not `make cross`) when targeting a specific set of binaries.
`make cross` ignores `WHAT` and builds all targets for all platforms.

```bash
KUBE_BUILD_PLATFORMS=linux/riscv64 make all \
  WHAT="cmd/kubelet cmd/kubectl cmd/kubeadm cmd/kube-apiserver \
        cmd/kube-controller-manager cmd/kube-scheduler cmd/kube-proxy"
```

All binaries, kubelet included, come out statically linked.

### Build specific binaries

```bash
# Just kubectl
KUBE_BUILD_PLATFORMS=linux/riscv64 make all WHAT="cmd/kubectl"
```

### Output location

Binaries are placed in:
```
_output/local/bin/linux/riscv64/
```

### Verify

```bash
file _output/local/bin/linux/riscv64/kubelet
# ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, ...

file _output/local/bin/linux/riscv64/kubectl
# ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, ...
```

## Method 2: Native Build on riscv64 Hardware

Build directly on a RISC-V machine running Debian Trixie.

### Hardware requirements

- RISC-V board or server (StarFive VisionFive 2, SiFive HiFive, Milk-V, etc.)
- At least 8 GB RAM (16 GB recommended)
- At least 30 GB free disk space
- Debian Trixie (13) for riscv64

### Install build dependencies

```bash
sudo apt-get update
sudo apt-get install -y git make gcc rsync
```

### Install Go

Install the Go version pinned by the Kubernetes source tree (run from the
source checkout):

```bash
GO_VERSION=$(cat .go-version)
curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-riscv64.tar.gz" | sudo tar -C /usr/local -xz
export PATH="/usr/local/go/bin:$PATH"
```

### Build

On native hardware, no cross-compilation flags are needed:

```bash
make all \
  WHAT="cmd/kubelet cmd/kubeadm cmd/kubectl cmd/kube-apiserver \
        cmd/kube-controller-manager cmd/kube-scheduler cmd/kube-proxy"
```

Note: Native builds on current RISC-V hardware are significantly slower than
cross-compilation. Expect 30-60 minutes for a full build depending on hardware.

## Method 3: Dockerized Cross-Compile with kube-cross

Use the community kube-cross image for a fully containerized build environment.

### Build using the community kube-cross image

```bash
docker run --rm \
  -v "$PWD":/go/src/k8s.io/kubernetes \
  -w /go/src/k8s.io/kubernetes \
  -e KUBE_BUILD_PLATFORMS=linux/riscv64 \
  ghcr.io/riseproject-dev/kube-cross:v1.38.0-go1.27.1-trixie.0 \
  make all WHAT="cmd/kubelet cmd/kubeadm cmd/kubectl"
```

Note: use `make all`, not `make cross` -- `make cross` ignores `WHAT` and
builds every target for every platform (see [known-issues.md](known-issues.md)).

### Or use the Kubernetes dockerized build system

Set the `KUBE_CROSS_IMAGE` override to point to the community image:

```bash
export KUBE_CROSS_IMAGE=ghcr.io/riseproject-dev/kube-cross
export KUBE_CROSS_VERSION=v1.38.0-go1.27.1-trixie.0
export KUBE_BUILD_PLATFORMS=linux/riscv64

build/run.sh make all WHAT="cmd/kubelet cmd/kubeadm cmd/kubectl"
```

## CGO Considerations

None for the Kubernetes binaries: since the v1.38 development cycle all
components build with `CGO_ENABLED=0` (kubelet was the last holdout;
upstream made it static in commit 81922ce9b57). The historical guidance
about a CGO kubelet for seccomp support, `KUBE_STATIC_OVERRIDES`, and a
riscv64 glibc sysroot applies only to pre-v1.38 trees.

If you do force a CGO build explicitly, the build system still honors
`KUBE_LINUX_RISCV64_CC` (default: `riscv64-linux-gnu-gcc`).

## Troubleshooting

### "riscv64-linux-gnu-gcc: command not found"

Only the pause image build needs this compiler:
```bash
sudo apt-get install gcc-riscv64-linux-gnu libc6-dev-riscv64-cross
```

### "unsupported GOOS/GOARCH pair"

Ensure you are using Go 1.21 or later. RISC-V (riscv64) has been a first-class
Go port since Go 1.21.

### Build runs out of memory on riscv64 hardware

Limit parallelism:
```bash
make all WHAT="cmd/kubelet" GOFLAGS="-p=2"
```

### KUBE_FASTBUILD not recognizing riscv64

Ensure the build system patches from this repository have been applied. The
`hack/lib/golang.sh` file must include `riscv64` in the `KUBE_FASTBUILD`
host architecture check.
