# Cluster Setup on Debian Trixie (riscv64)

This guide covers setting up a Kubernetes cluster on RISC-V hardware
running Debian Trixie (13) using kubeadm.

## Hardware Requirements

### Control Plane Node
- RISC-V board/server with 4+ cores
- 8 GB RAM minimum (16 GB recommended)
- 30 GB disk space
- Network connectivity

### Worker Nodes
- RISC-V board/server with 2+ cores
- 4 GB RAM minimum
- 20 GB disk space

### Tested Hardware
- StarFive VisionFive 2
- SiFive HiFive Unmatched
- Milk-V Pioneer
- QEMU riscv64 virtual machines

## OS Setup

Install Debian Trixie for riscv64 from the official installer:
https://www.debian.org/releases/trixie/

### System prerequisites (all nodes)

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Install Container Runtime

Debian Trixie ships containerd for riscv64. containerd includes the `ctr`
CLI. podman is optional and only useful for building or inspecting images
locally; it is not needed for cluster operation.

```bash
# Install containerd from Debian repos
sudo apt-get update
sudo apt-get install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

### Configure the sandbox (pause) image

Replace the default pause image reference with the community riscv64 image,
keeping the surrounding TOML quotes intact:

```bash
# containerd v1.x config (Debian Trixie ships 1.7.x):
sudo sed -i \
  's|sandbox_image = .*|sandbox_image = "ghcr.io/riseproject-dev/pause:3.10.2"|' \
  /etc/containerd/config.toml

# containerd v2.x config uses the "sandbox" key instead:
#   [plugins.'io.containerd.cri.v1.images'.pinned_images]
#     sandbox = 'ghcr.io/riseproject-dev/pause:3.10.2'

sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Install CNI Plugins

```bash
CNI_VERSION=$(curl -s https://api.github.com/repos/containernetworking/plugins/releases/latest | grep tag_name | cut -d '"' -f 4)
sudo mkdir -p /opt/cni/bin
curl -fsSL "https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-riscv64-${CNI_VERSION}.tgz" \
  | sudo tar -C /opt/cni/bin -xz
```

## Install Kubernetes Binaries

### From community builds

Community releases are tagged `${K8S_VERSION}-riscv64` and contain flat
tarballs of binaries (see the Releases page):

```bash
K8S_VERSION=v1.36.0

curl -fsSL "https://github.com/riseproject-dev/kubernetes-riscv/releases/download/${K8S_VERSION}-riscv64/kubernetes-node-linux-riscv64-${K8S_VERSION}.tar.gz" \
  | sudo tar -C /usr/local/bin -xz
sudo chmod +x /usr/local/bin/{kubelet,kubeadm,kubectl}

# Or copy binaries built from source (see building.md)
# sudo cp _output/local/bin/linux/riscv64/{kubelet,kubeadm,kubectl} /usr/local/bin/
# sudo chmod +x /usr/local/bin/{kubelet,kubeadm,kubectl}
```

### Configure kubelet systemd service

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo mkdir -p /etc/systemd/system/kubelet.service.d

cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
ExecStart=
ExecStart=/usr/local/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_KUBEADM_ARGS \$KUBELET_EXTRA_ARGS
EOF

sudo systemctl daemon-reload
sudo systemctl enable kubelet
```

## Initialize the Cluster (Control Plane Only)

The community images are published under `ghcr.io/riseproject-dev` with the
exact names and tags kubeadm derives (`kube-apiserver:${K8S_VERSION}`,
`etcd:3.6.8-0`, `coredns:v1.14.2`, `pause:3.10.2`), so pointing kubeadm's
`imageRepository` at the community registry is all that is needed — no
manual pulling or retagging.

etcd does not officially support riscv64 and refuses to start unless
`ETCD_UNSUPPORTED_ARCH=riscv64` is set; the `extraEnvs` entry below injects
it into the etcd static pod that kubeadm generates.

```bash
K8S_VERSION=v1.36.0

cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: ${K8S_VERSION}
imageRepository: ghcr.io/riseproject-dev
networking:
  podSubnet: 10.244.0.0/16
etcd:
  local:
    extraEnvs:
      - name: ETCD_UNSUPPORTED_ARCH
        value: riscv64
EOF

# Optional: pre-pull all images to fail fast on registry issues
sudo kubeadm config images pull --config kubeadm-config.yaml

sudo kubeadm init --config kubeadm-config.yaml
```

### Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### (Optional) Allow pods on control plane

For single-node clusters:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Install Pod Network (Flannel)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

The upstream Flannel images do not include riscv64, so patch the DaemonSet's
main container and both init containers (`install-cni-plugin` and
`install-cni`) to the community image, which also carries the CNI plugin
binary at the path the init container copies from:

```bash
FLANNEL_IMAGE=ghcr.io/riseproject-dev/flannel:v0.28.0-riscv64
kubectl -n kube-flannel set image daemonset/kube-flannel-ds \
  kube-flannel="${FLANNEL_IMAGE}" \
  install-cni-plugin="${FLANNEL_IMAGE}" \
  install-cni="${FLANNEL_IMAGE}"
```

## Join Worker Nodes

On the control plane, generate a join token:
```bash
kubeadm token create --print-join-command
```

On each worker node (after installing binaries, containerd with the
community sandbox image configured, and CNI plugins):
```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## Verify

```bash
# Check nodes
kubectl get nodes -o wide
# NAME     STATUS   ROLES           AGE   VERSION   INTERNAL-IP   OS-IMAGE           KERNEL-VERSION   CONTAINER-RUNTIME   ARCH
# node1    Ready    control-plane   5m    v1.36.0   10.0.0.1      Debian Trixie      6.x.x            containerd://1.7    riscv64

# Check system pods
kubectl get pods -n kube-system

# Verify architecture
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.architecture}'
# riscv64

# Deploy a test pod
kubectl run test --image=debian:trixie-slim --command -- sleep infinity
kubectl exec test -- uname -m
# riscv64
```
