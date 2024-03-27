---
layout: post
title:  "metallb-deployment"
date:   2023-12-29 12:00:00 +0800
categories: kubernetes
---

# 使用helm安装metallb

```
- name: Add-ons-Deployment
  hosts: masters
  gather_facts: no
  tasks:
    - name: helm repo add metallb https://metallb.github.io/metallb
      kubernetes.core.helm_repository:
        name: metallb
        repo_url: "https://metallb.github.io/metallb"
      environment:
        https_proxy: http://127.0.0.1:10809

    - name: helm install metallb metallb/metallb
      kubernetes.core.helm:
        name: metallb
        chart_ref: metallb/metallb
        release_namespace: metallb-system
        wait: true
      environment:
        https_proxy: http://127.0.0.1:10809

    - name: IPAddressPool
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: metallb.io/v1beta1
          kind: IPAddressPool
          metadata:
            name: first-pool
            namespace: metallb-system
          spec:
            addresses:
            - 172.29.62.240-172.29.62.250

    - name: L2Advertisement
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: metallb.io/v1beta1
          kind: L2Advertisement
          metadata:
            name: example
            namespace: metallb-system
          spec:
            ipAddressPools:
            - first-pool
```
