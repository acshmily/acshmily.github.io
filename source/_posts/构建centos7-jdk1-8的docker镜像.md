---
title: 构建centos7+jdk1.8的docker镜像
date: 2020-11-26 10:51:27
tags:
    - java
    - docker
---

## 原因

由于Oracle JAVA不是开源的，所以你在Docker Hub是找不到基于Oracle JAVA 的开源容器，由于目前现状来说，多数开发者还是基于Oracle java版本进行开发和运行。所以为了避免发生未知的错误，多数企业都会自己构建基于Oracle JAVA的容器。

## 准备工作

1. 阿里云Docker镜像私有化空间
2. 一台已经安装Docker服务的机器
3. 下载最新的Oracle Jdk1.8 

## 开始构建

```dockerfile
# 依赖上层镜像
FROM centos:centos7
# Dockerfile作者
MAINTAINER acshmily "github.com/acshmily"
# 创建目录
RUN mkdir /usr/local/jdk
# 设置工作目录
WORKDIR /usr/local/jdk
# 解压到 /usr/local/jdk下
ADD jdk-8u271-linux-x64.tar.gz /usr/local/jdk


# 安装必要的语言环境以及网络工具，安装完毕后清理缓存
RUN yum update -y && \
yum reinstall -y glibc-common && \
yum install kde-l10n-Chinese -y && \
yum install -y telnet net-tools && \
yum clean all && \
rm -rf /tmp/* rm -rf /var/cache/yum/* && \
localedef -c -f UTF-8 -i zh_CN zh_CN.UTF-8 && \
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 设置环境变量
ENV JAVA_HOME /usr/local/jdk/jdk1.8.0_271
ENV JRE_HOME /usr/local/jdk/jdk1.8.0_271/jre
ENV PATH $JAVA_HOME/bin:$PATH
ENV LC_ALL zh_CN.UTF-8
```

## 发布镜像

1. 在阿里云创建一个空间（略）
2. 登录阿里云Docker Registry
3. `sudo docker tag [ImageId] Your Registry:[镜像版本号]`
4. `sudo docker push  Your Registry:[镜像版本号]`

