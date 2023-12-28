# Kubernetes Cluster Setup Guide

This guide provides a step-by-step process to set up a Kubernetes cluster, detailing the configuration for the master node (node01) followed by the worker nodes (node02 and node03).

## Master Node Setup (node01)

### 1. Configure Firewall

Open necessary ports for the Kubernetes control plane:

```bash
firewall-cmd --permanent --add-port=6443/tcp  # Kubernetes API server
firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd server client API
firewall-cmd --permanent --add-port=10250/tcp  # Kubelet API
firewall-cmd --permanent --add-port=10257/tcp  # kube-controller-manager
firewall-cmd --permanent --add-port=10259/tcp  # kube-scheduler
firewall-cmd --reload
```

### 2. Disable Swap

Disable swap to ensure Kubernetes works correctly:

```bash
sed -i '/swap/d' /etc/fstab
swapoff -a
free -m
```

### 3. Install Docker

Remove Podman (if installed), add the Docker repository, and install Docker:

```bash
yum remove podman
yum install yum-utils -y
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl start docker
systemctl enable docker
```

### 4. Install cri-dockerd

Install `cri-dockerd` for Docker to work as the container runtime for Kubernetes:

```bash
sudo yum -y install git wget curl
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//g')
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
cri-dockerd --version
```

### 5. Configure cri-dockerd and Docker

Set up `cri-dockerd` and configure Docker daemon:

```bash
# Set up cri-dockerd service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
systemctl status cri-docker.socket

# Configure Docker daemon
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 6. Install Kubernetes Components

Install `kubeadm`, `kubelet`, and `kubectl`:

```bash
# Set up Kubernetes repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Install Kubernetes components
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

### 7. Initialize Kubernetes Cluster

Initialize the cluster with `kubeadm`:

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<NodeIP> --cri-socket=/var/run/cri-dockerd.sock
```

Replace `<NodeIP>` with the IP address of node01.

### 8. Configure Kubectl

Set up the kubeconfig file for `kubectl`:

```bash
mkdir -p

 $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 9. Deploy the Network Plugin

Deploy Flannel as the network plugin:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### 10. Generate Token for Worker Nodes

Generate a join token for the worker nodes:

```bash
kubeadm token create --print-join-command
```

---
# Worker Nodes Setup (node02 and node03)

This guide covers the steps to configure the worker nodes (node02 and node03) for a Kubernetes cluster. These steps should be performed on each worker node individually.

## 1. Configure Firewall

Set up necessary ports for Kubernetes worker nodes:

```bash
firewall-cmd --permanent --add-port=10250/tcp  # Kubelet API
firewall-cmd --permanent --add-port=30000-32767/tcp  # NodePort Services
firewall-cmd --reload
```

## 2. Disable Swap

Disabling swap is necessary for Kubernetes to function properly:

```bash
sed -i '/swap/d' /etc/fstab
swapoff -a
free -m
```

## 3. Install Docker

Install Docker, which will be used as the container runtime:

```bash
yum remove podman
yum install yum-utils -y
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl start docker
systemctl enable docker
```

## 4. Install cri-dockerd

Install `cri-dockerd` to integrate Docker with Kubernetes:

```bash
sudo yum -y install git wget curl
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//g')
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
cri-dockerd --version
```

## 5. Configure cri-dockerd and Docker

Set up the `cri-dockerd` service and configure the Docker daemon:

```bash
# Set up cri-dockerd service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
systemctl status cri-docker.socket

# Configure Docker daemon
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 6. Install Kubernetes Components

Install `kubeadm`, `kubelet`, and `kubectl`, which are essential for joining the Kubernetes cluster:

```bash
# Set up Kubernetes repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Install Kubernetes components
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

## 7. Join Cluster

Execute the join command generated by the master node. Add `--cri-socket /var/run/cri-dockerd.sock` at the end of the join command:

```bash
kubeadm join [MasterNodeIP]:6443 --token [YourToken] \
    --discovery-token-ca-cert-hash sha256:[YourHash] \
    --cri-socket /var/run/cri-dockerd.sock
```

Replace `[MasterNodeIP]`, `[YourToken]`, and `[YourHash]` with the actual values provided by the `kubeadm token create --print-join-command` output from the master node.

---

After completing these steps on each worker node, the nodes should be configured correctly and ready to join the Kubernetes cluster. Once joined, you can verify
