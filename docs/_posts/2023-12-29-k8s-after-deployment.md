---
layout: post
title:  "k8s-after-deployment"
date:   2023-12-29 12:00:00 +0800
categories: kubernetes
---

# 安装网络插件，flannel或者其他

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

# 设置认证

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

# 删除master污点

```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
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
