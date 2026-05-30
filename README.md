# ☸️ Kubernetes Cluster Setup on AWS EC2 (Ubuntu)

A step-by-step guide to setting up a Kubernetes cluster with one **Master Node** and one **Worker Node** on AWS EC2 using `kubeadm`, `containerd`, and `Flannel CNI`.

---

## 📋 Prerequisites

- AWS account with EC2 access
- Two Ubuntu EC2 instances (recommended: `c7` family or similar)
- Basic familiarity with Linux CLI
- SSH access to both instances

---

## 🏗️ Architecture

```
┌─────────────────────┐          ┌─────────────────────┐
│    Master Node      │◄────────►│    Worker Node      │
│  (Control Plane)    │          │     (Slave)         │
│                     │          │                     │
│  kubeadm init       │          │  kubeadm join       │
│  kubectl            │          │  kubelet            │
│  Flannel CNI        │          │  containerd         │
└─────────────────────┘          └─────────────────────┘
```

---

## 🚀 Step 1: AWS EC2 Setup

### 1.1 Launch EC2 Instances

- Launch **two Ubuntu EC2 instances** (one for master, one for worker)
- Recommended instance type: `c7` family or `t3.medium` minimum

### 1.2 Configure Security Groups

Open the following ports in the Security Groups for **both instances**:

| Port | Protocol | Purpose |
|------|----------|---------|
| `6443` | TCP | Kubernetes API Server |
| `10250` | TCP | Kubelet API |
| `22` | TCP | SSH Access |

### 1.3 Set Hostnames

SSH into each instance and set descriptive hostnames:

**On Master Node:**
```bash
sudo hostnamectl set-hostname master
bash
```

**On Worker Node:**
```bash
sudo hostnamectl set-hostname slave
bash
```

---

## 📦 Step 2: Install Common Dependencies (Both Nodes)

Run the following on **both master and worker nodes**.

### 2.1 Create the Setup Script

```bash
vim k8s-common-containerd.sh
```

Paste the script below:

```bash
#!/bin/bash
set -e

echo "==== COMMON: Starting Kubernetes base setup (containerd + kubeadm) ===="

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
echo "Swap disabled."

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Apply sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Install dependencies
sudo apt-get update -y
sudo apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gpg \
  lsb-release

# Remove old container runtimes
sudo apt-get remove -y docker docker-engine docker.io containerd runc || true

# Add Docker repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y

# Install containerd
sudo apt-get install -y containerd.io

# Configure containerd with SystemdCgroup
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
echo "Containerd configured successfully."

# Add Kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y

# Install Kubernetes packages
sudo apt-get install -y kubelet kubeadm kubectl cri-tools
sudo apt-mark hold kubelet kubeadm kubectl

echo "==== COMMON: Completed successfully ===="
```

### 2.2 Run the Script

```bash
sudo chmod +x k8s-common-containerd.sh
sudo ./k8s-common-containerd.sh
```

> ✅ Run this on **both master and worker nodes** before proceeding.

---

## 🎛️ Step 3: Initialize the Master Node

Run the following **only on the Master Node**.

### 3.1 Create the Master Init Script

```bash
vim k8s-master-init.sh
```

Paste the script below:

```bash
#!/bin/bash
set -e

echo "==== MASTER: Initializing Kubernetes Control Plane ===="

# Initialize Kubernetes cluster
sudo kubeadm init \
  --cri-socket unix:///run/containerd/containerd.sock \
  --pod-network-cidr=10.244.0.0/16

echo "==== Setting up kubeconfig ===="

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo "Kubeconfig setup complete."

echo "==== Installing Flannel CNI Plugin ===="
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

echo "==== Kubernetes Control Plane is Ready ===="
echo ""
echo "Run the following to check cluster status:"
echo "  kubectl get nodes"
echo "  kubectl get pods -A"
echo ""
echo "Generate worker join command:"
echo "  kubeadm token create --print-join-command"
```

### 3.2 Run the Script

```bash
sudo chmod +x k8s-master-init.sh
sudo ./k8s-master-init.sh
```

### 3.3 Generate Worker Join Command

```bash
kubeadm token create --print-join-command
```

> 📋 **Copy the output** — you'll need it in the next step.

---

## 🔗 Step 4: Join the Worker Node

Run the following **only on the Worker Node**.

### 4.1 Create the Worker Join Script

```bash
vim k8s-worker-join.sh
```

Paste the script below:

```bash
#!/bin/bash
set -e

echo "==== WORKER: Joining Kubernetes Cluster ===="

# Prompt for join command from master
read -p "Paste the kubeadm join command from master: " joincmd

# Execute with containerd CRI socket
sudo ${joincmd} --cri-socket unix:///run/containerd/containerd.sock

echo "==== WORKER: Successfully joined the Kubernetes Cluster! ===="
```

### 4.2 Run the Script

```bash
chmod +x k8s-worker-join.sh
sudo ./k8s-worker-join.sh
```

When prompted, paste the join command copied from the master node.

---

## ✅ Step 5: Verify the Cluster (Master Node)

Run on the **Master Node**:

```bash
kubectl get nodes -o wide
```

Expected output — both nodes should show `Ready`:

```
NAME     STATUS   ROLES           AGE   VERSION   INTERNAL-IP   ...
master   Ready    control-plane   Xm    v1.29.x   10.x.x.x      ...
slave    Ready    <none>          Xm    v1.29.x   10.x.x.x      ...
```

---

## 🛠️ Troubleshooting

### ❌ "The connection to the server localhost:8080 was refused"

Run these commands on the **master node** to fix kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Then retry:
```bash
kubectl get nodes -o wide
```

---

## 📁 File Summary

| File | Used On | Purpose |
|------|---------|---------|
| `k8s-common-containerd.sh` | Both nodes | Installs containerd, kubeadm, kubelet, kubectl |
| `k8s-master-init.sh` | Master only | Initializes control plane, installs Flannel CNI |
| `k8s-worker-join.sh` | Worker only | Joins worker to the cluster |

---

## 🧰 Tech Stack

- **OS:** Ubuntu (AWS EC2)
- **Container Runtime:** containerd
- **Kubernetes Version:** v1.29
- **CNI Plugin:** Flannel
- **Tools:** kubeadm, kubectl, kubelet

---

## 📄 License

This project is open-source and available under the [MIT License](LICENSE).
