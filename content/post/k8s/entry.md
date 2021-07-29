---
title: K8S setup
description: a tough start
date: 2021-07-13
slug: k8s/entry
image: img/2021/07/aqua.png
categories:
  - K8S
---

我的 **K8S 学习之旅** 01

## Kubernets 的官方定义

Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available.

## 单机搭建环境

鉴于本人的几台服务器配置都太差了，`kubectl`相关的实战都在 windows 上完成。

### 安装 Chocolatey

windows 环境下，还是比较推荐使用 Chocolatey 这个包管理工具的，很多程序都可以一键安装，升级也很方便。[官网在此](https://chocolatey.org/)

你也可以到[这里](https://community.chocolatey.org/packages)查询 Chocolatey 支持的程序。

注意：**`choco install`需要管理员权限。**

### 安装 Docker

1. 从[Docker 官网](https://www.docker.com/products/docker-desktop)下载安装文件，自行安装即可
2. 或者使用 choco 安装 `choco install docker-desktop`

注意：**minikube 依赖 docker 环境。**

### 使用 choco 安装 kubectl 和 minikube

```ps
choco install kubernetes-cli
choco install minikube

// 查看本地安装的程序
choco list --local
```

## 初步上手

使用`minikube start`，我们就获得了一个单节点的 kubernetes cluster

### 第一个应用

**nginx.yml 内容如下：**

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment # 定义pod名称
spec:
  selector:
    matchLabels:
      app: nginx # selector用于匹配template的label
  replicas: 2 # 两个副本
  template:
    metadata:
      labels:
        app: nginx # pod的label
    spec:
      volumes:
        - name: nginx-vol # 定义挂载容器
          emptyDir: {} # 自动生成的空路径，与Pod生命周期一致
      containers: # 定义容器内程序信息
        - name: nginx # image name
          image: nginx:1.20.0 # 具体的镜像信息
          ports:
            - containerPort: 80 # 端口
          resources: # 资源限制
            limits:
              memory: "128Mi"
              cpu: "500m"
          volumeMounts: # 容器挂载信息
            - mountPath: "/usr/share/nginx/html"
              name: nginx-vol
```

```ps
# 部署这个Pod
kubectl create -f nginx.yml
# 查看Pod内容器信息
kubectl get pods
# 查看Pod详情
kubectl describe pod <pod.name>
```

## 总结

暂时就这样
