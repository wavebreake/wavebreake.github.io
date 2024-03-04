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

```
mkdir -p ~/minio/data
podman run \
  --network kind \
  -p 9000:9000 \
  -p 9001:9001 \
  -v ~/minio/data:/data \
  -e "MINIO_ROOT_USER=ROOTNAME" \
  -e "MINIO_ROOT_PASSWORD=CHANGEME123" \
  quay.io/minio/minio server /data --console-address ":9001"
```

# 使用minio

Connect your Browser to the MinIO Server

```
http://127.0.0.1:9000
```
