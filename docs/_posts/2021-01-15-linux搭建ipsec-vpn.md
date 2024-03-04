---
layout: post
title:  "linux搭建ipsec-vpn"
date:   2021-01-15 12:00:00 +0800
categories: linux
---

# 概述

需求：客户系统位于北京4节点，需要访问内蒙大数据节点服务，拟通过VPN打通链路，北京4节点提供VPN产品，可直接配置本端，内蒙大数据节点无VPN产品，拟通过弹性云主机部署VPN服务配置对端。

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/ipsec-src.png)

模拟实验：青岛节点模拟北京4节点开通VPN服务，XX节点模拟内蒙大数据节点开通弹性云主机，使用Openswan部署IPSec VPN。

# 青岛节点VPN配置

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/ipsec-des.png)

# XX节点弹性云主机配置（私有地址192.168.0.130，弹性公网地址121.36.84.102）

安装：

vim /etc/sysctl.conf  
net.ipv4.ip_forward = 1  
net.ipv4.conf.default.rp_filter = 0  
echo "0">/proc/sys/net/ipv4/conf/lo/rp_filter  
sysctl -a | egrep "ipv4.*(accept|send)_redirects" | awk -F "=" '{print$1"= 0"}' >> /etc/sysctl.conf  
sysctl -p  
sudo yum -y install openswan  
ipsec --version  
Linux Libreswan 3.15 (netkey) on 2.6.32-696.1.1.el6.x86_64

配置：

ipsec.conf

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/ipsec-conf.png)

ipsec.secrets

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/ipsec-secret.png)

手动和青岛节点建立连接

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/ipsec-connect.png)
