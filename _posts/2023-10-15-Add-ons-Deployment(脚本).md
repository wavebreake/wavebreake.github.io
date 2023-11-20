---
layout: post
title:  "Add-ons-Deployment(脚本)"
date:   2023-10-15 12:00:00 +0800
categories: kubernetes
---

# deploy ansible

略

# deploy kubernetes-dashboard、metallb

```
- name: Add-ons-Deployment
  hosts: masters
  gather_facts: no
  tasks:
    - name: kubectl create ns kubernetes-dashboard
      kubernetes.core.k8s:
        name: kubernetes-dashboard
        api_version: v1
        kind: Namespace
        state: present

    - name: helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
      kubernetes.core.helm_repository:
        name: kubernetes-dashboard
        repo_url: "https://kubernetes.github.io/dashboard/"
      environment:
        https_proxy: http://127.0.0.1:7890

    - name: helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --namespace kubernetes-dashboard
      kubernetes.core.helm:
        name: kubernetes-dashboard
        chart_ref: kubernetes-dashboard/kubernetes-dashboard
        release_namespace: kubernetes-dashboard
      environment:
        https_proxy: http://127.0.0.1:7890

    - name: ServiceAccount
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: ServiceAccount
          metadata:
            name: admin-user
            namespace: kubernetes-dashboard

    - name: ClusterRoleBinding
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: rbac.authorization.k8s.io/v1
          kind: ClusterRoleBinding
          metadata:
            name: admin-user
          roleRef:
            apiGroup: rbac.authorization.k8s.io
            kind: ClusterRole
            name: cluster-admin
          subjects:
          - kind: ServiceAccount
            name: admin-user
            namespace: kubernetes-dashboard

    - name: Secret
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Secret
          metadata:
            name: admin-user
            namespace: kubernetes-dashboard
            annotations:
              kubernetes.io/service-account.name: "admin-user"   
          type: kubernetes.io/service-account-token  

    - name: kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"}
      command: kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"}
      register: pw

    - debug:
        msg: "{{ pw.stdout | b64decode }}"

    - name: kubectl create ns kubernetes-dashboard
      kubernetes.core.k8s:
        name: metallb-system
        api_version: v1
        kind: Namespace
        state: present

    - name: helm repo add metallb https://metallb.github.io/metallb
      kubernetes.core.helm_repository:
        name: metallb
        repo_url: "https://metallb.github.io/metallb"
      environment:
        https_proxy: http://127.0.0.1:7890

    - name: helm install metallb metallb/metallb
      kubernetes.core.helm:
        name: metallb
        chart_ref: metallb/metallb
        release_namespace: metallb-system
        wait: true
      environment:
        https_proxy: http://127.0.0.1:7890

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