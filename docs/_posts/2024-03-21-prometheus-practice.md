---
layout: post
title:  "prometheus-practice"
date:   2024-03-21 12:00:00 +0800
categories: kubernetes
---

# 使用Podman on Windows安装kind

Install Podman on Windows

```
https://github.com/containers/podman-desktop/releases/download/v1.8.0/podman-desktop-1.8.0-setup-x64.exe
```

get podman VM IP

```
podman run --rm --net host alpine ip a
```

e.g.

```
172.23.81.33/20
```

Install kubectl

点击settings->resources->kubectl

Install kind

点击左下角kind

Creating a kind cluster

点击settings->resources->kind

get kubernetes cluster master IP

```
podman inspect
```

e.g.

```
10.89.0.2
```

# 登录master安装kube-prometheus-stack

Get Helm Repository Info

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install Helm Chart

```
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack
```

# 使用grafana

暴露grafana服务,使用contour,参考

[contour-practice | 共倒金荷家万里 (wavebreake.github.io)](https://wavebreake.github.io/kubernetes/2024/03/21/contour-practice.html)

```
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: kube-prometheus-stack-grafana
  namespace: default
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: contour
    namespace: projectcontour
  hostnames:
  - "local.projectcontour.io"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - kind: Service
      name: kube-prometheus-stack-grafana
      port: 80
EOF
```

访问grafana服务,使用浏览器打开http://local.projectcontour.io:9090/

```
admin
prom-operator
```

进入Dashboards Kubernetes / Compute Resources / Pod,观察一下pod grafana的仪表盘

![prometheus](https://raw.githubusercontent.com/wavebreake/imagehosting/main/prometheus.jpeg)

进入Explore,查询一下pod grafana的CPU数据

![prometheus_1](https://raw.githubusercontent.com/wavebreake/imagehosting/main/prometheus_1.jpeg)

