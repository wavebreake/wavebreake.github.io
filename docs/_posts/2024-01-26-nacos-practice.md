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

# 登陆master安装nacos

Clone Project

```
git clone https://github.com/nacos-group/nacos-k8s.git
```

Simple Start

```
cd nacos-k8s
chmod +x quick-startup.sh
./quick-startup.sh
```

由于当前版本使用headless services，不太方便使用nodeport观察UI，所以将POD增加hostport以暴露

```
k edit statefulsets.apps nacos
```

```
spec:
  template:
    spec:
      containers:
      - ports:
        - containerPort: 8848
          name: client
          protocol: TCP
          hostPort: 8848
```

使用浏览器访问127.0.0.1:8848/nacos

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/nacos.png)

# 使用nacos

Service registration

```
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'
```

Service discovery

```
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName'
```

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/nacos_1.png)

Publish config

```
curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=helloWorld"
```

Get config

```
curl -X GET "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"
```

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/nacos_2.png)
