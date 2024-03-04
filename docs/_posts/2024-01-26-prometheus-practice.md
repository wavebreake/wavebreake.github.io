---
layout: post
title:  "prometheus-practice"
date:   2024-01-26 12:00:00 +0800
categories: prometheus
---

# 安装kube-prometheus-stack

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack
```
