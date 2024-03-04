---
layout: post
title:  "nacos-practice"
date:   2024-01-26 12:00:00 +0800
categories: nacos
---

# 使用Podman on Mac安装kind

Install Podman on Mac

```
https://github.com/containers/podman-desktop/releases/download/v1.7.0/podman-desktop-1.7.0-universal.dmg
```

get podman VM IP

```
podman run --rm --net host alpine ip a
```

e.g.

```
192.168.127.2
```

Install kubectl on macOS

```
brew install kubectl
```

Install kind on macOS

```
brew install kind
```

Creating a kind cluster with Rootless Podman

```
KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster  --config=config.yaml
```

Extra mounts can be used to pass through storage on the host to a kind node for persisting data, mounting through code etc.

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
  - hostPath: /nacos-k8s
    containerPath: /nacos-k8s
```

get kubernetes cluster master IP

```
podman inspect
```

e.g.

```
10.89.0.2
```

# 登陆container安装nacos

```
cd nacos-k8s/
chmod 700 get_helm.sh
./get_helm.sh
```
