---
layout: post
title:  "strimzi-practice"
date:   2024-03-13 12:00:00 +0800
categories: kafka
---

# 使用strimz部署kafka

Deploying Strimzi using Helm

```
helm install strimzi-cluster-operator oci://quay.io/strimzi-helm/strimzi-kafka-operator
```

Deploying a ZooKeeper-based Kafka cluster+the Topic Operator+the User Operator

```
cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 3.6.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
      inter.broker.protocol.version: "3.6"
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 1Gi
        deleteClaim: false
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 1Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF
```

# 使用kafka

Send messages

```
kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:0.40.0-kafka-3.6.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic 
```

receive messages

```
kubectl run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.40.0-kafka-3.6.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic filebeat --from-beginning
```

查看topic

```
k exec my-cluster-kafka-0 -- bin/kafka-topics.sh --list --bootstrap-server my-cluster-kafka-bootstrap:9092
```

使用KafkaTopic创建topic

```
cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: filebeat
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 1
  replicas: 1
EOF
```
