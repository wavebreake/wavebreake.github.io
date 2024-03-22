---
layout: post
title:  "zookeeper-practice"
date:   2024-01-26 12:00:00 +0800
categories: kubernetes
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
touch nfs-csi.yaml
cat <<EOF | tee ./nfs-csi.yaml
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
k apply -f nfs-csi.yaml
```

```
touch pvc-nfs-csi-dynamic.yaml
cat <<EOF | tee ./pvc-nfs-csi-dynamic.yaml
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
k apply -f pvc-nfs-csi-dynamic.yaml
```

Mark a StorageClass as default

```
kubectl patch storageclass nfs-csi -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

# 安装zookeeper

创建一个 ZooKeeper Ensemble

```
touch zookeeper.yaml
cat <<EOF | tee ./zookeeper.yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-hs
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "registry.k8s.io/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
          --servers=3 \
          --data_dir=/var/lib/zookeeper/data \
          --data_log_dir=/var/lib/zookeeper/data/log \
          --conf_dir=/opt/zookeeper/conf \
          --client_port=2181 \
          --election_port=3888 \
          --server_port=2888 \
          --tick_time=2000 \
          --init_limit=10 \
          --sync_limit=5 \
          --heap=512M \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
EOF
kubectl apply -f zookeeper.yaml
```

这里有一个问题(原因尚在研究),NFS CSI driver在/data创建的目录/data/pvc-xxx的权限拒绝zookeeper的POD在其中创建目录,临时解决办法,create statefulset后,手动将/data/pvc-xxx的权限调整为777,然后delete statefulset重新create

# 使用zookeeper

查看zookeeper状态

```
for i in 0 1 2; do kubectl exec zk-$i -- zkServer.sh status; done
```

测试zookeeper写读

```
kubectl exec zk-0 -- zkCli.sh create /hello world
kubectl exec zk-1 -- zkCli.sh get /hello
```
