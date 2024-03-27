---
layout: post
title:  "k8s+containerd-deployment"
date:   2023-12-29 12:00:00 +0800
categories: kubernetes
---

# 安装步骤

1. download tar

2. init linux

3. deploy container-runtimes

4. deploy kubeadm、kubelet、kubectl

5. init k8s

# 使用脚本安装k8s+containerd

```
- name: download tar
  hosts: masters
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16  

  vars:
    ARCH: "amd64"
    OS: "linux"
    CONTAINERD_VERSION: "1.7.13"
    RUNC_VERSION: "v1.1.12"
    CNI_VERSION: "v1.4.0"
    CRICTL_VERSION: "v1.28.0"
    RELEASE: "v1.29.2"
    RELEASE_VERSION: "v0.16.2"

  tasks:
    - name: donwload containerd
      get_url:
        url: https://github.com/containerd/containerd/releases/download/v{{CONTAINERD_VERSION}}/containerd-{{CONTAINERD_VERSION}}-{{OS}}-{{ARCH}}.tar.gz
        dest: ./containerd.tar.gz

    - name: donwload containerd.service
      get_url:
        url: https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
        dest: ./containerd.service

    - name: donwload runc
      get_url:
        url: https://github.com/opencontainers/runc/releases/download/{{RUNC_VERSION}}/runc.{{ARCH}}
        dest: .

    - name: donwload cni
      get_url:
        url: https://github.com/containernetworking/plugins/releases/download/{{CNI_VERSION}}/cni-plugins-{{OS}}-{{ARCH}}-{{CNI_VERSION}}.tgz
        dest: ./cni-plugins.tgz

    - name: donwload crictl
      get_url:
        url: https://github.com/kubernetes-sigs/cri-tools/releases/download/{{CRICTL_VERSION}}/crictl-{{CRICTL_VERSION}}-linux-{{ARCH}}.tar.gz
        dest: ./crictl.tar.gz

    - name: donwload kubeadm
      get_url:
        url: https://dl.k8s.io/release/{{RELEASE}}/bin/linux/{{ARCH}}/kubeadm
        dest: .

    - name: donwload kubelet
      get_url:
        url: https://dl.k8s.io/release/{{RELEASE}}/bin/linux/{{ARCH}}/kubelet
        dest: .

    - name: donwload kubelet.service
      get_url:
        url: https://raw.githubusercontent.com/kubernetes/release/{{RELEASE_VERSION}}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service
        dest: .

    - name: donwload 10-kubeadm.conf
      get_url:
        url: https://raw.githubusercontent.com/kubernetes/release/{{RELEASE_VERSION}}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf
        dest: .

    - name: donwload kubectl
      get_url:
        url: https://dl.k8s.io/release/{{RELEASE}}/bin/linux/{{ARCH}}/kubectl
        dest: .

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

    - name: modprobe
      become: true
      shell: modprobe overlay;modprobe br_netfilter

    - name:  /etc/sysctl.d/k8s.conf
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

    - name: install socat
      become: true
      apt:
        name: socat
        state: present

    - name: install conntrack
      become: true
      apt:
        name: conntrack
        state: present

- name: deploy container-runtimes
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: mkdir -P /usr/local/lib/systemd/system/
      become: true
      file:
        path: /usr/local/lib/systemd/system
        state: directory

    - name: cp containerd.service /usr/local/lib/systemd/system/
      become: true
      copy:
        src: ./containerd.service
        dest: /usr/local/lib/systemd/system

    - name:  tar Cxzvf /usr/local containerd.tar.gz
      become: true
      unarchive: 
        src: ./containerd.tar.gz
        dest: /usr/local
        copy: no

    - name: install -m 755 runc.amd64 /usr/local/sbin/runc
      become: true
      shell: install -m 755 runc.amd64 /usr/local/sbin/runc

    - name: mkdir -p /opt/cni/bin
      become: true
      file:
        path: /opt/cni/bin
        state: directory

    - name: tar Cxzvf /opt/cni/bin cni-plugins.tgz
      become: true
      unarchive: 
        src: ./cni-plugins.tgz
        dest: /opt/cni/bin
        copy: no

    - name: mkdir /etc/containerd
      become: true
      file:
        path: /etc/containerd/
        state: directory

    - name: containerd config default
      shell: containerd config default > ./config.toml

    - name: ./config.toml
      lineinfile:
        dest: ./config.toml
        regexp: "^            SystemdCgroup = "
        line: "            SystemdCgroup = true"

    - name: ./config.toml
      lineinfile:
        dest: ./config.toml
        regexp: "^    sandbox_image = "
        line: "    sandbox_image = \"registry.k8s.io/pause:3.9\""

    - name: cp ./config.toml /etc/containerd/
      become: true
      copy:
        src: ./containerd.service
        dest: /usr/local/lib/systemd/system    

    - name: /etc/systemd/system/containerd.service.d/http-proxy.conf
      become: true
      blockinfile:
        path: /etc/systemd/system/containerd.service.d/http-proxy.conf
        create: yes
        block: |
          [Service]
          Environment="HTTP_PROXY=http://127.0.0.1:10809"
          Environment="HTTPS_PROXY=http://127.0.0.1:10809"
          Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"

    - name: daemon_reload
      become: true
      systemd:
        daemon_reload: yes

    - name: restart containerd
      become: true
      systemd:
        name: containerd
        state: restarted

    - name: enable containerd
      become: true
      systemd:
        name: containerd
        enabled: yes

- name: deploy kubeadm、kubelet、kubectl
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: deploy crictl
      become: true
      unarchive: 
        src: ./crictl.tar.gz
        dest: /usr/local/bin
        copy: no

    - name: copy kubeadm
      become: true
      copy:
        src: ./kubeadm
        dest: /usr/local/bin

    - name: chmod kubeadm
      become: true
      file: 
        dest: /usr/local/bin/kubeadm
        mode: +x

    - name: copy kubelet
      become: true
      copy:
        src: ./kubelet
        dest: /usr/local/bin

    - name: chmod kubelet
      become: true
      file: 
        dest: /usr/local/bin/kubelet
        mode: +x

    - name: copy kubelet.service
      become: true
      copy:
        src: ./kubelet.service
        dest: /etc/systemd/system/

    - name: /etc/systemd/system/kubelet.service
      become: true
      lineinfile:
        dest: /etc/systemd/system/kubelet.service
        regexp: "^ExecStart=/usr/bin/kubelet"
        line: "ExecStart=/usr/local/bin/kubelet"

    - name: mkdir kubelet.service.d
      become: true
      file:
        path: /etc/systemd/system/kubelet.service.d
        state: directory

    - name: copy 10-kubeadm.conf
      become: true
      copy:
        src: ./10-kubeadm.conf
        dest: /etc/systemd/system/kubelet.service.d

    - name: /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
      become: true
      lineinfile:
        dest: /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
        regexp: "^ExecStart=/usr/bin/kubelet"
        line: "ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS"

    - name: copy kubectl
      become: true
      copy:
        src: ./kubectl
        dest: /usr/local/bin

    - name: chmod kubectl
      become: true
      file: 
        dest: /usr/local/bin/kubectl
        mode: +x

    - name: daemon_reload
      become: true
      systemd:
        daemon_reload: yes

    - name: restart kubelet
      become: true
      systemd:
        name: kubelet
        state: restarted

    - name: enable kubelet
      become: true
      systemd:
        name: kubelet
        enabled: yes

- name: init k8s
  hosts: masters
  gather_facts: no
  environment:
    http_proxy: http://127.0.0.1:10809
    https_proxy: http://127.0.0.1:10809
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:    
    - name: kubeadm init --pod-network-cidr=10.244.0.0/16
      become: true
      command: kubeadm init --pod-network-cidr=10.244.0.0/16
      register: kubeadm_output

    - debug:
        var: kubeadm_output
```
