---
layout: post
title:  "contour-practice"
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

# 登录master安装contour

Deploy the Gateway provisioner:

```
kubectl apply -f https://projectcontour.io/quickstart/contour-gateway-provisioner.yaml
```

Verify the Gateway provisioner deployment is available:

```
kubectl -n projectcontour get deployments
```

Create a GatewayClass:

```
kubectl apply -f - <<EOF
kind: GatewayClass
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: contour
spec:
  controllerName: projectcontour.io/gateway-controller
EOF
```

Create a Gateway:

```
kubectl apply -f - <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: contour
  namespace: projectcontour
spec:
  gatewayClassName: contour
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
EOF
```

Verify the `Gateway` is available (it may take up to a minute to become available):

```
kubectl -n projectcontour get gateways
```

Verify the Contour pods are ready by running the following:

```
kubectl -n projectcontour get pods
```

# 使用ingress暴露nginx服务

创建一个nginx deployment

```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```

创建一个nginx service

```
kubectl expose deployment/my-nginx
```

 创建一个nginx ingress

```
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  defaultBackend:
    service:
      name: my-nginx
      port:
        number: 80
EOF
```

由于envoy的service运行在kind中，我们无法直接从个人电脑访问，简单地修改envoy的daemonset(containerPort: 8080映射到hostPort: 80)，再通过podman的http映射(containerPort: 80映射到hostPort: 9090)访问

访问nginx服务,使用浏览器打开http://local.projectcontour.io:9090/

![contour](https://raw.githubusercontent.com/wavebreake/imagehosting/main/contour.jpeg)

# 使用Gateway API(HTTPRoute` and `TLSRoute)暴露nginx服务

删除nginx ingress

```
kubectl delete ingress my-nginx
```

创建nginx HTTPRoute

```
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: my-nginx
  namespace: default
  labels:
    run: my-nginx
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
      name: my-nginx
      port: 80
EOF
```

访问nginx服务,使用浏览器打开http://local.projectcontour.io:9090/

![contour](https://raw.githubusercontent.com/wavebreake/imagehosting/main/contour.jpeg)
