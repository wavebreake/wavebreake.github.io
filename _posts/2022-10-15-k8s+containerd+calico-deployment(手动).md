---
layout: post
title:  "k8s+containerd+calico-deployment(手动)"
date:   2022-10-15 12:00:00 +0800
categories: kubernetes

---

# linux init

```
setenforce 0
sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
getenforce
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
free

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

apt update

apt autoremove -y
```

# deploy container-runtimes

```
wget https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.2-linux-amd64.tar.gz

tar Cxzvf /usr/local containerd-1.7.2-linux-amd64.tar.gz

wget -P /usr/local/lib/systemd/system/ https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

wget https://github.com/opencontainers/runc/releases/download/v1.1.8/runc.amd64

install -m 755 runc.amd64 /usr/local/sbin/runc

wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz

mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz

mkdir /etc/containerd/
containerd config default > /etc/containerd/config.toml

vim /etc/containerd/config.toml

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9"

# mkdir /etc/systemd/system/containerd.service.d

# vim /etc/systemd/system/containerd.service.d/http-proxy.conf

# [Service]
# Environment="HTTP_PROXY=http://172.21.40.19:7890"
# Environment="HTTPS_PROXY=http://172.21.40.19:7890"
# Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"

systemctl daemon-reload
systemctl restart containerd
systemctl enable containerd
```

# deploy kubeadm、kubelet、kubectl

```
apt install -y apt-transport-https ca-certificates curl

mkdir /etc/apt/keyrings

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list

apt update

apt install -y kubelet kubeadm kubectl

apt-mark hold kubelet kubeadm kubectl
```

# init k8s

```
kubeadm config images pull --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc

wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml

kubectl create -f tigera-operator.yaml

wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml

vim custom-resources.yaml

kubectl create -f custom-resources.yaml

curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o calicoctl

chmod +x ./calicoctl
```

# join k8s

略