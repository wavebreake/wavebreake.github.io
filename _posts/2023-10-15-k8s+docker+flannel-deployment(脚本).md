---
layout: post
title:  "k8s+docker+flannel-deployment(脚本)"
date:   2023-10-15 12:00:00 +0800
categories: kubernetes
---

# deploy ansible

略

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
    - name: setenforce 0;swapoff -a
      become: true
      shell: setenforce 0;swapoff -a

#    - name: sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
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

    - name: apt update
      become: true
      ansible.builtin.apt:
        update_cache: yes
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

    - name: install -m 0755 -d /etc/apt/keyrings
      become: true
      command: install -m 0755 -d /etc/apt/keyrings

    - name: curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      become: true
      ansible.builtin.apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        keyring: /etc/apt/keyrings/docker.gpg

    - name: chmod a+r /etc/apt/keyrings/docker.gpg
      become: true
      ansible.builtin.file:
        path: /etc/apt/keyrings/docker.gpg
        mode: a+r

    - name: echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      become: true
      blockinfile:
        path: /etc/apt/sources.list.d/docker.list
        create: yes
        block: |
          deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu   jammy stable

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
          Environment="HTTP_PROXY=http://172.21.40.19:7890"
          Environment="HTTPS_PROXY=http://172.21.40.19:7890"
          Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"

    - name: donwload cri-dockerd.deb
      get_url:
        url: https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.4/cri-dockerd_0.3.4.3-0.ubuntu-jammy_amd64.deb
        dest: /home/wavebreak/cri-dockerd.deb

    - name: apt install cri-dockerd.deb
      become: true
      apt:
        deb: /home/wavebreak/cri-dockerd.deb
        state: present

    - name: /lib/systemd/system/cri-docker.service
      become: true
      lineinfile:
        dest: /lib/systemd/system/cri-docker.service
        regexp: "ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd://"
        line: "ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=registry.k8s.io/pause:3.9"

    - name: donwload cni-plugins.tgz
      get_url:
        url: https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
        dest: /home/wavebreak/cni-plugins.tgz

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
```

# deploy kubeadm、kubelet、kubectl

```
- name: kube-deployment
  hosts: k8s
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:
    - name: apt install apt-transport-https
      become: true
      apt:
        name: apt-transport-https
        state: present

    - name: curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      become: true
      ansible.builtin.apt_key:
        url: https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key
        keyring: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    - name: echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
      become: true
      blockinfile:
        path:  /etc/apt/sources.list.d/kubernetes.list
        create: yes
        block: |
          deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /

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
```

# init k8s

```
- name: kubeadm init
  hosts: masters
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
  tasks:    
    - name: kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
      become: true
      command: kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
      register: kubeadm_output

    - debug:
        var: kubeadm_output
```

# flannel-deployment

```
- name: flannel-deployment
  hosts: masters
  gather_facts: no
  environment:
    http_proxy: http://172.21.40.19:7890
    https_proxy: http://172.21.40.19:7890
    no_proxy: localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
    KUBECONFIG: /etc/kubernetes/admin.conf
  tasks:
    - name: flannel-deployment
      become: true
      kubernetes.core.k8s:
        state: present
        src: https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
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