---
layout: post
title:  "jenkins-practice"
date:   2024-01-26 12:00:00 +0800
categories: jenkins
---

# 使用Docker Desktop安装kind

Install Docker Desktop on Mac

```
https://desktop.docker.com/mac/main/arm64/Docker.dmg?utm_source=docker&utm_medium=webreferral&utm_campaign=docs-driven-download-mac-arm64&_gl=1*1v6jo26*_ga*ODU5ODY3MDEyLjE3MDYyNDc1NTQ.*_ga_XJWPQMJYHQ*MTcwNjI0NzU1NC4xLjEuMTcwNjI0NzYwOC42LjAuMA..
```

get docker desktop VM IP

```
docker run --rm --net host alpine ip a
```

e.g.

```
192.168.65.3
```

Install kubectl on macOS

```
brew install kubectl
```

Install kind on macOS

```
brew install kind
```

Creating a kubernetes cluster

```
kind create cluster
```

get kubernetes cluster master IP

```
docker inspect
```

e.g.

```
172.18.0.2
```

# 安装Jenkins

Installing Jenkins in Docker

Customize the official Jenkins Docker image, by executing the following two steps:

Create a Dockerfile with the following content:

```
FROM jenkins/jenkins:2.426.3-jdk17
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
```

Build a new docker image from this Dockerfile, and assign the image a meaningful name, such as "myjenkins-blueocean:2.426.3-1":

docker build -t myjenkins-blueocean:2.426.3-1 .

Run your own myjenkins-blueocean:2.426.3-1 image as a container in Docker using the following docker run command:

```
docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network kind --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.426.3-1
```

get jenkins IP

```
docker inspect
```

e.g.

```
172.18.0.3
```

Unlocking Jenkins

```
https://www.jenkins.io/doc/book/installing/docker/#setup-wizard
```

# 使用jenkins

1. jenkins提供多种任务模式,可以理解为不同语法的脚本(Jenkinsfile),包括freestyle project、pipeline、multi configuration pipeline等

2. Jenkinsfile可以直接编辑文档,也可以通过UI生成,下一代UI blue ocean可以通过表单直接生成multi configuration pipeline类型的Jenkinsfile

3. jenkins任务运行在所谓agent,有多种方式临时部署agent:docker container、kubernetes POD、node。如果将docker engine或者kubernetes cluster添加成cloud,貌似他们是以node方式部署agent,如何以container或者POD方式部署agent有待研究

e.g.

将kubernetes cluster添加成cloud

```
https://plugins.jenkins.io/kubernetes/
```

将docker engine添加成cloud

```
https://plugins.jenkins.io/docker-plugin/
```

将docker engine添加成cloud这里有一个问题,macOS+docker desktop环境下,docker engine port无法通过daemon.json或者docker.service文件配置暴露,比较简单的一个办法是创建一个container将docker engine所在VM的port映射出来,比如我们创建一个IP为172.18.0.4的container:

```
docker run --net kind \
  -it \
  --rm \
  --name=api-forwarder \
  -p 2375:2375 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  alpine sh -c 'apk add --no-cache socat && socat TCP-LISTEN:2375,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock'
```

创建一个POD template(定义当需要一个agent时,如何启动一个node)

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/jenkins_manage_cloud_kind_configure.png)

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/jenkins_manage_cloud_kind_template.png)

创建一个pipeline

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/jenkins_blue_create-pipeline.png)

创建一个step

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/jenkins_pipeline-editor_devops_main.png)

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/jenkins_pipeline-editor_devops_main_%201.png)

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/jenkins_pipeline-editor_devops_main_%202.png)

运行pipeline

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/jenkins_manage_computer_.png)

结果

![](https://raw.githubusercontent.com/wavebreake/imagehosting/main/jenkins_devops_detail_main_2_pipeline.png)
