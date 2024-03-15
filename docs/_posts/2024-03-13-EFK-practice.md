---
layout: post
title:  "EFK-practice"
date:   2024-03-13 12:00:00 +0800
categories: EFK
---

# 安装NFS

Installation

```
sudo apt install nfs-kernel-server
sudo systemctl start nfs-kernel-server.service
```

Configuration

```
/data    *(rw,sync,no_subtree_check)
sudo mkdir /data
sudo exportfs -a
```

NFS Client Configuration

```
sudo apt install nfs-common
sudo mkdir ~/data
sudo mount 127.0.0.1:/data ~/data
```

# 安装NFS CSI driver for Kubernetes

Install driver on a Kubernetes cluster

```
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system --version v4.6.0
```

CSI driver Usage

```
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 127.0.0.1
  share: /data
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1
EOF
```

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-dynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs-csi
EOF
```

Mark a StorageClass as default

```
kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

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

# 生产建议:filebeat->Kafka->logstash->elastic search->kibana

安装ECK

参考[EFK-practice | 共倒金荷家万里](https://wavebreake.github.io/efk/2024/03/13/EFK-practice.html)

安装kafka

参考[strimzi-practice | 共倒金荷家万里](https://wavebreake.github.io/kafka/2024/03/13/strimzi-practice.html)

安装filebeat

```
cat <<EOF | kubectl apply -f -
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: quickstart
spec:
  type: filebeat
  version: 8.12.2
  config:
    filebeat.inputs:
    - type: container
      paths:
      - /var/log/containers/*.log
    output.kafka:
      hosts: ["my-cluster-kafka-bootstrap.default.svc:9092"]
      topic: 'filebeat'
      partition.round_robin:
        reachable_only: false
      required_acks: 1
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

安装logstash

```
cat <<EOF | kubectl apply -f -
apiVersion: logstash.k8s.elastic.co/v1alpha1
kind: Logstash
metadata:
  name: quickstart
spec:
  count: 1
  elasticsearchRefs:
    - name: quickstart
      clusterName: qs
  version: 8.12.2
  pipelines:
    - pipeline.id: main
      config.string: |
        input {
          kafka {
            bootstrap_servers => "my-cluster-kafka-bootstrap.default.svc:9092"
            topics => ["filebeat"]
          }
        }
        output {
          elasticsearch {
            hosts => [ "${QS_ES_HOSTS}" ]
            user => "${QS_ES_USER}"
            password => "${QS_ES_PASSWORD}"
            ssl_certificate_authorities => "${QS_ES_SSL_CERTIFICATE_AUTHORITY}"
          }
        }
EOF
```

这里不知道为什么通过elasticsearchRefs的传参在logstash启动时总是报错，参数直接写入后成功，具体原因尚待排查。

```
cat <<EOF | kubectl apply -f -
apiVersion: logstash.k8s.elastic.co/v1alpha1
kind: Logstash
metadata:
  name: quickstart
spec:
  count: 1
  elasticsearchRefs:
    - name: quickstart
      clusterName: qs
  version: 8.12.2
  pipelines:
    - pipeline.id: main
      config.string: |
        input {
          kafka {
            bootstrap_servers => "my-cluster-kafka-bootstrap.default.svc:9092"
            topics => ["filebeat"]
          }
        }
        output {
          elasticsearch {
            hosts => [ "https://quickstart-es-http.default.svc:9200" ]
            user => "elastic"
            password => "9K3h38lc4cM6xB2tt81mD2VA"
            ssl_certificate_authorities => "/mnt/elastic-internal/elasticsearch-association/default/quickstart/certs/ca.crt"
          }
        }
EOF
```

暴露Kibana服务,暂且使用最简单的nodeport

访问Kibana服务,使用浏览器打开[https://127.0.0.1:30011](https://127.0.0.1:30011)

```
user: elastic
password: kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

进入discove

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/elk_3.png)

创建数据视图

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/elk_4.png)

搜索日志

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/elk_5.png)
