---
layout: post
title:  "ansible-practice"
date:   2023-12-29 12:00:00 +0800
categories: kubernetes
---

# 安装ansible

```
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py --user
python3 -m pip install --user ansible
```

# 使用ansible
免密登陆

```
ssh-keygen
ssh-copy-id xxx.xxx.xxx.xxx
```

创建inventory

```
vim inventory.yaml
```

```
masters:
  hosts:
    master:
      ansible_host: xxx.xxx.xxx.xxx

nodes:
  hosts:
    node:
      ansible_host: xxx.xxx.xxx.xxx

k8s:
  children:
    masters:
    nodes:
```

创建playbook

```
vim playbook.yaml
```

```
- name: XXX
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: apt update
      become: true
      ansible.builtin.apt:
        update_cache: yes  
```

运行playbook

```
ansible-playbook playbook.yaml -i inventory.yaml -K
```




























