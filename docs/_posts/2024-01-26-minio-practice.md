---
layout: post
title:  "minio-practice"
date:   2024-01-26 12:00:00 +0800
categories: minio
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
KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster
```

get kubernetes cluster master IP

```
podman inspect
```

e.g.

```
10.89.0.2
```

# 安装MinIO

Download the MinIO Object

```
curl https://raw.githubusercontent.com/minio/docs/master/source/extra/examples/minio-dev.yaml -O
```

修改minio-dev.yaml

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: minio
  name: minio
  namespace: minio-dev # Change this value to match the namespace metadata.name
spec:
  containers:
  - name: minio
    image: quay.io/minio/minio:latest
    command:
    - /bin/bash
    - -c
    args:
    - minio server /data --console-address :9090
    volumeMounts:
    - mountPath: /data
      name: localvolume # Corresponds to the `spec.volumes` Persistent Volume
  volumes:
  - name: localvolume
    hostPath: # MinIO generally recommends using locally-attached volumes
      path: /data # Specify a path to a local drive or volume on the Kubernetes worker node
      type: DirectoryOrCreate # The path to the last directory must exist
```

Apply the MinIO Object Definition

```
kubectl apply -f minio-dev.yaml
```

Temporarily Access the MinIO S3 API and Console

```
kubectl port-forward pod/minio 9000 9090 -n minio-dev
```

# 使用minio

Connect your Browser to the MinIO Server

```
http://127.0.0.1:9090
minioadmin | minioadmin
```

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/minio.png)

Connect the MinIO Client

```
mc alias set k8s-minio-dev http://127.0.0.1:9000 minioadmin minioadmin
mc admin info k8s-minio-dev
```
