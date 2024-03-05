---
layout: post
title:  "prometheus-practice"
date:   2024-01-26 12:00:00 +0800
categories: prometheus
---

# 安装kube-prometheus-stack

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

暴露grafana服务,暂且使用最简单的nodeport

访问grafana服务,使用浏览器打开https://127.0.0.1:30011

```
admin
prom-operator
```

dashboards/node explore/nodes

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/prometheus.png)
