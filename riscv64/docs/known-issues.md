# Known Issues and Workarounds

## etcd: Unsupported Architecture Flag

etcd does not officially support riscv64. The `ETCD_UNSUPPORTED_ARCH=riscv64`
environment variable must be set at runtime. Without it, etcd refuses to start.

This is set automatically in the community etcd Dockerfile. When using
kubeadm, the clean fix is the kubeadm v1beta4 `etcd.local.extraEnvs` field
in the ClusterConfiguration, which injects the variable into the generated
etcd static pod manifest (see [cluster-setup.md](cluster-setup.md)):

```yaml
etcd:
  local:
    extraEnvs:
    - name: ETCD_UNSUPPORTED_ARCH
      value: "riscv64"
```

Alternatively (for clusters not managed by kubeadm), add the environment
variable to the etcd static pod manifest by hand:

```yaml
# /etc/kubernetes/manifests/etcd.yaml
spec:
  containers:
  - name: etcd
    env:
    - name: ETCD_UNSUPPORTED_ARCH
      value: "riscv64"
```

## Build Time on Native Hardware

Building Kubernetes natively on current RISC-V hardware is approximately
9x slower than on x86_64. Cross-compilation from amd64 is strongly
recommended for regular builds.

| Method | Approximate Time |
|--------|-----------------|
| Cross-compile from amd64 | ~10-15 minutes |
| Native on StarFive VisionFive 2 | ~60-90 minutes |
| Native on Milk-V Pioneer | ~30-45 minutes |

To reduce native build time, limit parallelism to avoid OOM:
```bash
make all WHAT="cmd/kubelet" GOFLAGS="-p=2"
```

## Docker Setup Action (GitHub CI)

The `docker/setup-docker-action` GitHub Action does not support riscv64
runners. When using RISE Project riscv64 runners (`ubuntu-24.04-riscv`),
skip this action and rely on the container engine pre-installed on the
runner. Condition the step on however your workflow selects the runner --
for example, a matrix variable holding the `runs-on` label:

```yaml
- uses: docker/setup-docker-action@v4
  if: matrix.runner != 'ubuntu-24.04-riscv'
```

## Pause Image Override

The default pause image (`registry.k8s.io/pause`) does not include an riscv64
variant. All nodes must be configured to use the community pause image. CI
publishes it both as `pause:3.10.2-linux-riscv64` (upstream naming) and as
the unsuffixed `ghcr.io/riseproject-dev/pause:3.10.2` used below.

For containerd v2.x, set in `/etc/containerd/config.toml`:
```toml
[plugins."io.containerd.cri.v1.images"]
  pinned_images = ["ghcr.io/riseproject-dev/pause:3.10.2"]
```

For containerd v1.x:
```toml
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "ghcr.io/riseproject-dev/pause:3.10.2"
```

## kubelet CGO and seccomp (resolved in v1.38)

Historical issue for pre-v1.38 trees only: kubelet used to require CGO
(and therefore a riscv64 glibc sysroot when cross-compiling) for seccomp
support. Since the v1.38 development cycle upstream builds kubelet as a
static CGO-free binary (kubernetes/kubernetes commit 81922ce9b57), so no
cross C toolchain or `KUBE_STATIC_OVERRIDES` handling is needed anymore.

## `make cross` ignores WHAT

The `make cross` target always builds all server, node, client, and test targets
for all supported platforms. The `WHAT` variable is ignored. Use `make all` with
`KUBE_BUILD_PLATFORMS=linux/riscv64` to build specific binaries:

```bash
KUBE_BUILD_PLATFORMS=linux/riscv64 make all WHAT="cmd/kubectl cmd/kubeadm"
```

## Fedora: Missing riscv64 glibc Sysroot

Fedora's `gcc-riscv64-linux-gnu` package is a bare cross-compiler without
glibc. Since v1.38 this no longer affects the Kubernetes binaries (all
static, no CGO); it only matters for cross-building the pause image,
which links against static glibc. Workarounds: build pause via the
dockerized kube-cross method, or use Debian/Ubuntu, which ships
`libc6-dev-riscv64-cross`.

## No Official Release Artifacts

The Kubernetes project does not produce official binaries or container images
for riscv64. All artifacts must be built from source using this guide or
obtained from community builds. This is consistent with Tier 3 (Community
Supported) status.

## Conformance Test Status

Full Kubernetes conformance test results for riscv64 are not yet available.
Basic cluster operations (pod scheduling, service networking, DNS resolution)
have been validated. Conformance testing is an ongoing effort.

## Protobuf Compiler on riscv64

The protobuf project does not ship pre-built `protoc` binaries for riscv64.
When building on native riscv64 hardware, install protobuf from Debian
packages:
```bash
sudo apt-get install protobuf-compiler
```
