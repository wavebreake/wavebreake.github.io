---
layout: post
title:  "k8s+docker-deployment"
date:   2023-12-29 12:00:00 +0800
categories: kubernetes
---

# 安装步骤

1. remove container-runtimes

2. init linux

3. deploy container-runtimes

4. deploy kubeadm、kubelet、kubectl

5. init k8s

# 使用脚本安装k8s+docke

```
- name: remove container-runtimes
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: apt remove docker.io
      become: true
      apt:
        name: docker.io
        state: absent

    - name: apt remove docker-doc
      become: true
      apt:
        name: docker-doc
        state: absent

    - name: apt remove docker-compose
      become: true
      apt:
        name: docker-docdocker-compose
        state: absent

    - name: apt remove podman-docker
      become: true
      apt:
        name: podman-docker
        state: absent

    - name: apt remove containerd
      become: true
      apt:
        name: containerd
        state: absent

    - name: apt remove runc
      become: true
      apt:
        name: runc
        state: absent

    - name: apt remove ca-certificates
      become: true
      apt:
        name: ca-certificates
        state: present

    - name: apt remove curl
      become: true
      apt:
        name: curl
        state: present

    - name: apt remove gnupg
      become: true
      apt:
        name: gnupg
        state: present

- name: init linux
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: setenforce 0;swapoff -a
      become: true
      shell: setenforce 0;swapoff -a

#    - name: sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
#      become: true
#      lineinfile:
#        dest: /etc/selinux/config
#        regexp: "^SELINUX="
#        line: "SELINUX=disabled"

    - name: sed -ri 's/.*swap.*/#&/' /etc/fstab
      become: true
      lineinfile:
        dest: /etc/fstab
        regexp: ".*swap"
        line: ""

    - name: /etc/modules-load.d/k8s.conf
      become: true
      blockinfile:
        path: /etc/modules-load.d/k8s.conf
        create: yes
        block: |
          overlay
          br_netfilter

    - name: modprobe overlay;modprobe br_netfilter
      become: true
      shell: modprobe overlay;modprobe br_netfilter

    - name: /etc/sysctl.d/k8s.conf
      become: true
      blockinfile:
        path: /etc/sysctl.d/k8s.conf
        create: yes
        block: |
          net.bridge.bridge-nf-call-ip6tables = 1
          net.bridge.bridge-nf-call-iptables = 1
          net.ipv4.ip_forward = 1

    - name: sysctl --system
      become: true
      shell: sysctl --system

- name: deploy container-runtimes
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16

  vars:
    ARCH: "amd64"
    JAMMY: "jammy"
    OS: "linux"
    CNI_VERSION: "v1.4.0"

  tasks:
    - name: apt install ca-certificates
      become: true
      apt:
        name: ca-certificates
        state: present

    - name: apt install curl
      become: true
      apt:
        name: curl
        state: present

    - name: install -m 0755 -d /etc/apt/keyrings
      become: true
      command: install -m 0755 -d /etc/apt/keyrings

    - name: curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
      become: true
      get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /etc/apt/keyrings/docker.asc

    - name: chmod a+r /etc/apt/keyrings/docker.asc
      become: true
      ansible.builtin.file:
        path: /etc/apt/keyrings/docker.asc
        mode: a+r

    - name: echo "deb [arch={{ARCH}} signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      become: true
      blockinfile:
        path: /etc/apt/sources.list.d/docker.list
        create: yes
        block: |
          deb [arch={{ARCH}} signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu {{JAMMY}} stable

    - name: apt update
      become: true
      ansible.builtin.apt:
        update_cache: yes

    - name: apt install docker-ce
      become: true
      apt:
        name: docker-ce
        state: present

    - name: apt install docker-ce-cli
      become: true
      apt:
        name: docker-ce-cli
        state: present

    - name: apt install containerd.io
      become: true
      apt:
        name: containerd.io
        state: present

    - name: apt install docker-buildx-plugin
      become: true
      apt:
        name: docker-buildx-plugin
        state: present

    - name: apt install docker-compose-plugin
      become: true
      apt:
        name: docker-compose-plugin
        state: present

    - name: /etc/systemd/system/docker.service.d/http-proxy.conf
      become: true
      blockinfile:
        path: /etc/systemd/system/docker.service.d/http-proxy.conf
        create: yes
        block: |
          [Service]
          Environment="HTTP_PROXY=http://127.0.0.1:10809"
          Environment="HTTPS_PROXY=http://127.0.0.1:10809"
          Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"

    - name: donwload cri-dockerd.deb
      get_url:
        url: https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.10/cri-dockerd_0.3.10.3-0.ubuntu-jammy_amd64.deb
        dest: ./cri-dockerd.deb

    - name: apt install cri-dockerd.deb
      become: true
      apt:
        deb: ./cri-dockerd.deb
        state: present

    - name: /lib/systemd/system/cri-docker.service
      become: true
      lineinfile:
        dest: /lib/systemd/system/cri-docker.service
        regexp: "ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd://"
        line: "ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=registry.k8s.io/pause:3.9"

    - name: donwload cni-plugins.tgz
      get_url:
        url: https://github.com/containernetworking/plugins/releases/download/{{CNI_VERSION}}/cni-plugins-{{OS}}-{{ARCH}}-{{CNI_VERSION}}.tgz
        dest: ./cni-plugins.tgz

    - name: mkdir -p /opt/cni/bin
      become: true
      file:
        path: /opt/cni/bin
        state: directory

    - name: tar Cxzvf /opt/cni/bin cni-plugins.tgz
      become: true
      unarchive: 
        src: /home/wavebreak/cni-plugins.tgz
        dest: /opt/cni/bin
        copy: no

    - name: systemctl daemon-reload
      become: true
      systemd:
        daemon_reload: yes

    - name: systemctl restart docker
      become: true
      systemd:
        name: docker
        state: restarted

    - name: systemctl restart cri-docker
      become: true
      systemd:
        name: cri-docker
        state: restarted

- name: deploy kubeadm、kubelet、kubectl
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: apt install apt-transport-https
      become: true
      apt:
        name: apt-transport-https
        state: present

    - name: apt install gpg
      become: true
      apt:
        name: gpg
        state: present

    - name: curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      become: true
      ansible.builtin.apt_key:
        url: https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key
        keyring: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    - name: echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
      become: true
      blockinfile:
        path:  /etc/apt/sources.list.d/kubernetes.list
        create: yes
        block: |
          deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /

    - name: apt update
      become: true
      ansible.builtin.apt:
        update_cache: yes

    - name: apt install kubelet
      become: true
      apt:
        name: kubelet
        state: present

    - name: apt install kubeadm
      become: true
      apt:
        name: kubeadm
        state: present

    - name: apt install kubectl
      become: true
      apt:
        name: kubectl
        state: present

    - name: apt-mark hold kubelet
      become: true
      dpkg_selections:
        name: kubelet
        selection: hold

    - name: apt-mark hold kubeadm
      become: true
      dpkg_selections:
        name: kubeadm
        selection: hold

    - name: apt-mark hold kubeadm
      become: true
      dpkg_selections:
        name: kubeadm
        selection: hold

- name: init k8s
  hosts: masters
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:    
    - name: kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
      become: true
      command: kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
      register: kubeadm_output

    - debug:
        var: kubeadm_output
```
