---
layout: post
title:  "istio-practice"
date:   2024-03-25 12:00:00 +0800
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

# 登录master安装istio

Download Istio

```
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.21.0
export PATH=$PWD/bin:$PATH
```

Install Istio using the demo profile

```
istioctl install -f samples/bookinfo/demo-profile-no-gateways.yaml -y
```

Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later:

```
kubectl label namespace default istio-injection=enabled
```

Install Kiali and the other addons and wait for them to be deployed.

```
kubectl apply -f samples/addons
```

安装Gateway API

> In addition to its own traffic management API, Istio includes beta support for the Kubernetes Gateway API and intends to make it the default API for traffic management in the future

看起来云原生生态正在拥抱Gateway API(HTTPRoute)，因此我们使用Gateway API(HTTPRoute)代替Istio APIs(VirtualService)

创建gatewayclass

```
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=444631bfe06f3bcca5d0eadf1857eac1d369421d" | kubectl apply -f -; }
```

创建istio的Gateway

```
kubectl create namespace istio-ingress
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: istio-gateway
  namespace: istio-ingress
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
EOF
```

由于istio-gateway的service运行在kind中，我们无法直接从个人电脑访问，简单地修改istio-gateway的deployment(containerPort: 8080映射到hostPort: 80,containerPort: 8443映射到hostPort: 443)，再通过podman的http映射(containerPort: 80映射到hostPort: 9090,containerPort: 443映射到hostPort: 9443)访问

# 使用kiali

创建kiali的HTTPRoute

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: kiali-httproute
  namespace: istio-system
spec:
  parentRefs:
  - name: istio-gateway
    namespace: istio-ingress
  hostnames: 
  - "local.projectcontour.io"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /kiali
    backendRefs:
    - name: kiali
      port: 20001
EOF
```

访问kiali服务,使用浏览器打开http://local.projectcontour.io:9090/kiali

![istio_3](https://raw.githubusercontent.com/wavebreake/imagehosting/main/istio_3.jpeg)

# 使用jaeger

创建jaeger的HTTPRoute

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: jaeger-httproute
  namespace: istio-system
spec:
  parentRefs:
  - name: istio-gateway
    namespace: istio-ingress
  hostnames: 
  - "local.projectcontour.io"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /jaeger
    backendRefs:
    - name: tracing
      port: 80
EOF
```

访问jaeger服务,使用浏览器打开http://local.projectcontour.io:9090/jaeger

![istio_1](https://raw.githubusercontent.com/wavebreake/imagehosting/main/istio_1.jpeg)

![istio_2](https://raw.githubusercontent.com/wavebreake/imagehosting/main/istio_2.jpeg)

# 上面测试用的应用bookinfo

安装bookinfo

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

创建bookinfo的HTTPRoute

```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo-httproute
spec:
  parentRefs:
  - name: istio-gateway
    namespace: istio-ingress
  hostnames: 
  - "local.projectcontour.io"
  rules:
  - matches:
    - path:
        type: Exact
        value: /productpage
    - path:
        type: PathPrefix
        value: /static
    - path:
        type: Exact
        value: /login
    - path:
        type: Exact
        value: /logout
    - path:
        type: PathPrefix
        value: /api/v1/products
    backendRefs:
    - name: productpage
      port: 9080
EOF
```

访问bookinfo服务,使用浏览器打开http://local.projectcontour.io:9090/productpage

![istio](https://raw.githubusercontent.com/wavebreake/imagehosting/main/istio.jpeg)

不停使用F5刷新，观察kiali和jaeger
