---
layout: post
title:  "k8s+containerd+calico-deployment(脚本)"
date:   2023-04-15 12:00:00 +0800
categories: kubernetes
---

# deploy ansible

略

# download tar

```
- name: container-runtimes-download
  hosts: masters
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16  

  vars:
    CRICTL_VERSION: "v1.27.0"
    RELEASE: "v1.27.4"
    ARCH: "amd64"
    RELEASE_VERSION: "v0.15.1"

  tasks:
    - name: mkdir download
      file:
        path: /root/1
        state: directory

    - name: donwload containerd
      get_url:
        url: https://github.com/containerd/containerd/releases/download/v1.7.3/containerd-1.7.3-linux-amd64.tar.gz
        dest: /root/1/containerd-linux-amd64.tar.gz

    - name: donwload containerd.service
      get_url:
        url: https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
        dest: /root/1/containerd.service

    - name: donwload runc
      get_url:
        url: https://github.com/opencontainers/runc/releases/download/v1.1.8/runc.amd64
        dest: /root/1

    - name: donwload cni
      get_url:
        url: https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
        dest: /root/1/cni-plugins-linux-amd64.tgz

    - name: donwload crictl
      get_url:
        url: https://github.com/kubernetes-sigs/cri-tools/releases/download/{{CRICTL_VERSION}}/crictl-{{CRICTL_VERSION}}-linux-{{ARCH}}.tar.gz
        dest: /root/1/crictl-linux-amd64.tar.gz

    - name: donwload kubeadm
      get_url:
        url: https://dl.k8s.io/release/{{RELEASE}}/bin/linux/{{ARCH}}/kubeadm
        dest: /root/1

    - name: donwload kubelet
      get_url:
        url: https://dl.k8s.io/release/{{RELEASE}}/bin/linux/{{ARCH}}/kubelet
        dest: /root/1

    - name: donwload kubelet.service
      get_url:
        url: https://raw.githubusercontent.com/kubernetes/release/{{RELEASE_VERSION}}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service
        dest: /root/1

    - name: donwload 10-kubeadm.conf
      get_url:
        url: https://raw.githubusercontent.com/kubernetes/release/{{RELEASE_VERSION}}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf
        dest: /root/1

    - name: donwload kubectl
      get_url:
        url: https://dl.k8s.io/release/{{RELEASE}}/bin/linux/{{ARCH}}/kubectl
        dest: /root/1

    - name: donwload tigera-operator.yaml
      get_url:
        url: https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
        dest: /root/1

    - name: donwload custom-resources.yaml
      get_url:
        url: https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
        dest: /root/1

    - name: donwload calicoctl
      get_url:
        url: https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64
        dest: /root/1/calicoctl
```

# init linux

```
- name: container-runtimes-prerequisites
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: 即时生效
      shell: setenforce 0;swapoff -a

#    - name: 关闭selinux
#      lineinfile:
#        dest: /etc/selinux/config
#        regexp: "^SELINUX="
#        line: "SELINUX=disabled"

    - name: 关闭swap
      lineinfile:
        dest: /etc/fstab
        regexp: ".*swap"
        line: ""

    - name: /etc/modules-load.d/k8s.conf
      blockinfile:
        path: /etc/modules-load.d/k8s.conf
        create: yes
        block: |
          overlay
          br_netfilter

    - name: modprobe
      shell: modprobe overlay;modprobe br_netfilter
        
    - name:  /etc/sysctl.d/k8s.conf
      blockinfile:
        path: /etc/sysctl.d/k8s.conf
        create: yes
        block: |
          net.bridge.bridge-nf-call-ip6tables = 1
          net.bridge.bridge-nf-call-iptables = 1
          net.ipv4.ip_forward = 1

    - name: sysctl
      shell: sysctl --system

    - name: apt update
      shell: apt update

    - name: apt autoremove -y
      shell: apt autoremove -y

    - name: install socat
      apt:
        name: socat 
        state: present

    - name: install conntrack
      apt:
        name: conntrack
        state: present
```

# deploy container-runtimes

```
- name: container-runtimes-deployment
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: copy containerd
      copy:
        src: /root/1/containerd-linux-amd64.tar.gz
        dest: /root

    - name: mkdir systemd
      file:
        path: /usr/local/lib/systemd/system
        state: directory

    - name: copy containerd.service
      copy:
        src: /root/1/containerd.service
        dest: /usr/local/lib/systemd/system

    - name: copy runc
      copy:
        src: /root/1/runc.amd64
        dest: /root

    - name: copy cni
      copy:
        src: /root/1/cni-plugins-linux-amd64.tgz
        dest: /root

    - name: deploy containerd
      unarchive: 
        src: /root/containerd-linux-amd64.tar.gz
        dest: /usr/local
        copy: no

    - name: deploy runc
      shell: install -m 755 runc.amd64 /usr/local/sbin/runc

    - name: mkdir cni
      file:
        path: /opt/cni/bin
        state: directory

    - name: deploy cni
      unarchive: 
        src: /root/cni-plugins-linux-amd64.tgz
        dest: /opt/cni/bin
        copy: no

    - name: mkdir containerd
      file:
        path: /etc/containerd/
        state: directory

    - name: containerd config default
      shell: containerd config default > /etc/containerd/config.toml

    - name: /etc/containerd/config.toml
      lineinfile:
        dest: /etc/containerd/config.toml
        regexp: "^            SystemdCgroup = "
        line: "            SystemdCgroup = true"

    - name: /etc/containerd/config.toml
      lineinfile:
        dest: /etc/containerd/config.toml
        regexp: "^    sandbox_image = "
        line: "    sandbox_image = \"registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9\""

    - name: daemon_reload
      systemd:
        daemon_reload: yes

    - name: restart containerd
      systemd:
        name: containerd
        state: restarted

    - name: enable containerd
      systemd:
        name: containerd
        enabled: yes
```

# deploy kubeadm、kubelet、kubectl

```
- name: kubeadm-deployment
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: copy crictl
      copy:
        src: /root/1/crictl-linux-amd64.tar.gz
        dest: /root

    - name: deploy crictl
      unarchive: 
        src: /root/crictl-linux-amd64.tar.gz
        dest: /usr/local/bin
        copy: no

    - name: copy kubeadm
      copy:
        src: /root/1/kubeadm
        dest: /usr/local/bin

    - name: chmod kubeadm
      file: 
        dest: /usr/local/bin/kubeadm
        mode: +x

    - name: copy kubelet
      copy:
        src: /root/1/kubelet
        dest: /usr/local/bin

    - name: chmod kubelet
      file: 
        dest: /usr/local/bin/kubelet
        mode: +x

    - name: copy kubelet.service
      copy:
        src: /root/1/kubelet.service
        dest: /etc/systemd/system/

    - name: /etc/systemd/system/kubelet.service
      lineinfile:
        dest: /etc/systemd/system/kubelet.service
        regexp: "^ExecStart=/usr/bin/kubelet"
        line: "ExecStart=/usr/local/bin/kubelet"

    - name: mkdir kubelet.service.d
      file:
        path: /etc/systemd/system/kubelet.service.d
        state: directory

    - name: copy 10-kubeadm.conf
      copy:
        src: /root/1/10-kubeadm.conf
        dest: /etc/systemd/system/kubelet.service.d

    - name: /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
      lineinfile:
        dest: /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
        regexp: "^ExecStart=/usr/bin/kubelet"
        line: "ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS"

    - name: copy kubectl
      copy:
        src: /root/1/kubectl
        dest: /usr/local/bin

    - name: chmod kubectl
      file: 
        dest: /usr/local/bin/kubectl
        mode: +x

    - name: daemon_reload
      systemd:
        daemon_reload: yes

    - name: restart kubelet
      systemd:
        name: kubelet
        state: restarted

    - name: enable kubelet
      systemd:
        name: kubelet
        enabled: yes
```

# init k8s

```
- name: kubeadm-init
  hosts: masters
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
    KUBECONFIG: /etc/kubernetes/admin.conf
  tasks:    
    - name: Initialize Kubernetes master
      command: kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers
      register: kubeadm_output

    - debug:
        var: kubeadm_output
```

# calico-deployment

```
- name: calico-deployment
  hosts: masters
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
    KUBECONFIG: /etc/kubernetes/admin.conf
  tasks:
    - name: copy tigera-operator.yaml
      copy:
        src: /root/1/tigera-operator.yaml
        dest: /root

    - name: Install tigera-operator 
      command: kubectl create -f tigera-operator.yaml

    - name: copy custom-resources.yaml
      copy:
        src: /root/1/custom-resources.yaml
        dest: /root

    - name: /root/custom-resources.yaml
      lineinfile:
        dest: /root/custom-resources.yaml
        regexp: "^      cidr:"
        line: "      cidr: 10.244.0.0/16"

    - name: /root/custom-resources.yaml
      lineinfile:
        dest: /root/custom-resources.yaml
        regexp: "^      encapsulation: VXLANCrossSubnet"
        line: "      encapsulation: None"

    - name: Install custom-resources.yaml
      command: kubectl create -f custom-resources.yaml

    - name: copy calicoctl
      copy:
        src: /root/1/calicoctl
        dest: /root

    - name: chmod calicoctl
      file: 
        dest: /root/calicoctl
        mode: +x
```

# join k8s

```
- name: kubeadm-join
  hosts: nodes
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: Join Kubernetes nodes to the cluster
      command: ""
```