# Kubernetes Cluster Setup Guide

This guide provides step-by-step instructions to set up a Kubernetes cluster with one master (node01) and two worker nodes (node02 and node03) using Docker as the container runtime. Ensure each step is followed on the respective nodes as described.

## Step 1: Configure Firewall on All Nodes

### On the Master Node (node01)

Open necessary ports for the Kubernetes control plane:

```bash
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10257/tcp
firewall-cmd --permanent --add-port=10259/tcp
firewall-cmd --reload
```

### On Worker Nodes (node02 & node03)

Open ports for worker node communication:

```bash
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --reload
```

## Step 2: Disable Swap and Install Docker (All Nodes)

### Disable Swap

```bash
sed -i '/swap/d' /etc/fstab
swapoff -a
free -m
```

### Install Docker

Remove Podman if installed, add Docker repository, and install Docker:

```bash
yum remove podman
yum install yum-utils -y
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl start docker
systemctl enable docker
```

## Step 3: Install cri-dockerd (All Nodes)

### Download and Install cri-dockerd

```bash
sudo yum -y install git wget curl
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//g')
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
cri-dockerd --version
```

### Set up systemd Service for cri-dockerd

```bash
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
systemctl status cri-docker.socket
```

### Configure Docker Daemon

```bash
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

## Step 4: Install Kubernetes Components (All Nodes)

### Set up Kubernetes Repository

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

### Install and Start kubelet

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

## Step 5: Initialize Kubernetes Cluster on Master (node01)

### Initialize the Kubernetes Cluster

Replace `<NodeIP>` with the IP address of node01.

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<NodeIP> --cri-socket=/var/run/cri-dockerd.sock
```

### Set up K

ubeconfig

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Deploy the Network Plugin (Flannel)

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### Generate Token for Worker Nodes

```bash
kubeadm token create --print-join-command
```

## Step 6: Join Worker Nodes to the Cluster (node02 and node03)

On each worker node, execute the join command generated by the master node. Add `--cri-socket /var/run/cri-dockerd.sock` at the end of the join command.

```bash
kubeadm join [MasterNodeIP]:6443 --token [YourToken] \
    --discovery-token-ca-cert-hash sha256:[YourHash] \
    --cri-socket /var/run/cri-dockerd.sock
```

Replace `[MasterNodeIP]`, `[YourToken]`, and `[YourHash]` with the actual values provided by the `kubeadm token create --print-join-command` output from the master node.

---

After completing these steps, you should have a functional Kubernetes cluster with one master and two worker nodes. You can verify the status of your cluster with `kubectl get nodes`.
