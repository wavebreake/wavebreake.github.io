---
layout: post
title:  "k8s-after-deployment"
date:   2023-12-29 12:00:00 +0800
categories: kubernetes
---

# 设置认证

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

# 设置快捷命令

```
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
k get nodes
k get pod -A
```

# 删除master污点

```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

# 设置containerd proxy(用于pull image)

```
mkdir /etc/systemd/system/containerd.service.d/
touch /etc/systemd/system/containerd.service.d/http-proxy.conf
cat > /etc/systemd/system/containerd.service.d/http-proxy.conf << EOF
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:10809"
Environment="HTTPS_PROXY=http://127.0.0.1:10809"
Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
EOF
systemctl daemon-reload
systemctl restart containerd
```

# 安装helm

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

# 安装网络插件，flannel或者其他

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
