# k8s-cluster-with-kubeadm-calico
k8s Cluster Set Up guide with kubeadm and calico

# Budget Friendly Kubernetes Cluster Setup with Kubeadm and Calico for StartUps and SMEs.


𝐇𝐞𝐫𝐞’𝐬 𝐭𝐡𝐞 𝐯𝐢𝐛𝐞:
Lean Setup: Launched Ubuntu VMs in a 10.0.0.0/24 private subnet on Azure/AWS, with a jump server running an OpenVPN server for secure VPN access from my local machine (see my previous posts for the VPN setup!). 💻
Core Tech: Installed containerd and Kubernetes tools (Kubeadm, Kubelet, Kubectl) for a 2-node cluster (master + worker) on minimal VMs (2GB RAM, 2 CPUs).
Smart Networking: Used a 10.0.0.0/24 subnet to avoid Calico’s 192.168.0.0/16 CIDR, ensuring glitch-free networking with Calico’s VXLAN mode.

🔍𝐇𝐨𝐰 𝐈𝐭 𝐂𝐚𝐦𝐞 𝐓𝐨𝐠𝐞𝐭𝐡𝐞𝐫:
1. Set up Ubuntu VMs in a 10.0.0.0/24 private subnet, with a jump server for OpenVPN.
2. Installed containerd and Kubernetes tools, setting the stage for Kubeadm.
3. Initialized the master with Kubeadm, deployed Calico, and joined the worker node.
5. Checked with kubectl get nodes and kubectl get pods -A to confirm all’s good.

💡𝐏𝐫𝐨 𝐓𝐢𝐩𝐬:
Save cash with spot instances or reserved VMs.
Secure the subnet with tight security groups and VPN-only access.
Add Prometheus for lightweight cluster monitoring.


## Required Ports

### Control Plane Node (Master)
| Protocol | Port Range | Purpose |
|----------|------------|---------|
| TCP | 6443 | Kubernetes API Server |
| TCP | 2379-2380 | etcd server client API |
| TCP | 10250 | Kubelet API |
| TCP | 10259 | kube-scheduler |
| TCP | 10257 | kube-controller-manager |
| TCP | 179 | Calico BGP (if using BGP mode) |
| UDP | 4789 | Calico VXLAN |
| TCP | 5473 | Calico Typha (if used) |
| TCP | 22 | SSH access |

### Worker Node
| Protocol | Port Range | Purpose |
|----------|------------|---------|
| TCP | 10250 | Kubelet API |
| TCP | 30000-32767 | NodePort Services |
| TCP | 179 | Calico BGP (if using BGP mode) |
| UDP | 4789 | Calico VXLAN |
| TCP | 22 | SSH access |

## Step 1: Prepare All Nodes

Run on all nodes (master and workers):

```bash
# Update system and install prerequisites
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Set kernel parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install containerd using the correct method
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y containerd.io

# Configure containerd properly
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i '/disabled_plugins = \["cri"\]/d' /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install Kubernetes components using the updated repository for Ubuntu 24.04
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Step 2: Initialize Master Node

Run on master node only:

```bash
# Initialize the cluster with pod CIDR that doesn't overlap with Azure subnet (10.0.0.0/24)
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=$(kubelet --version | cut -d ' ' -f 2)

# If you encounter CRI issues, try specifying the socket explicitly:
# sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=$(kubelet --version | cut -d ' ' -f 2) --cri-socket unix:///run/containerd/containerd.sock

# Set up kubectl configuration
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 3: Deploy Calico Network Plugin

On master node:

```bash
# Apply Calico manifests
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

# Wait for Calico pods to be ready
kubectl wait --for=condition=Ready pods --all -n calico-system --timeout=300s
```

## Step 4: Join Worker Nodes

On worker nodes:

```bash
# Use the join command from the kubeadm init output
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

If you lost the token, run this on master:
```bash
kubeadm token create --print-join-command
```

## Step 5: Verify Cluster

On master node:

```bash
# Check node status
kubectl get nodes

# Verify pods are running
kubectl get pods --all-namespaces
```


