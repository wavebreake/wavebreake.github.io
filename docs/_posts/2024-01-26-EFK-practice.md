---
layout: post
title:  "EFK-practice"
date:   2024-01-26 12:00:00 +0800
categories: EFK
---

# 安装ECK+filebeat

Install ECK using the Helm chart

```
helm repo add elastic https://helm.elastic.co
helm repo update
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

Deploy an Elasticsearch cluster

```
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 8.12.2
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
EOF
```

Deploy a Kibana instance

```
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 8.12.2
  count: 1
  elasticsearchRef:
    name: quickstart
EOF
```

Run Beats on ECK

```
cat <<EOF | kubectl apply -f -
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: quickstart
spec:
  type: filebeat
  version: 8.12.2
  elasticsearchRef:
    name: quickstart
  config:
    filebeat.inputs:
    - type: container
      paths:
      - /var/log/containers/*.log
  daemonSet:
    podTemplate:
      spec:
        dnsPolicy: ClusterFirstWithHostNet
        hostNetwork: true
        securityContext:
          runAsUser: 0
        containers:
        - name: filebeat
          volumeMounts:
          - name: varlogcontainers
            mountPath: /var/log/containers
          - name: varlogpods
            mountPath: /var/log/pods
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
        volumes:
        - name: varlogcontainers
          hostPath:
            path: /var/log/containers
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
EOF
```

# 使用Kibana

暴露Kibana服务,暂且使用最简单的nodeport

访问Kibana服务,使用浏览器打开https://127.0.0.1:30011

```
user: elastic
password: kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

进入discove

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/efk.png)

创建数据视图

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/efk_1.png)

搜索日志

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/efk_2.png)
