---
title: Docker安装
date: 2023-04-07 10:29:01
tags: Docker、云原生
categories:
- Docker
- 云原生
---

Docker在随着云原生热潮下，是所有开发者机器上必不可少的一个工具，今天记录一下docker的安装过程. 本安装教程是基于连网情况下进行安装的，离线安装请移步互联网搜索.

<!-- more -->

## requirement
Docker 要求 CentOS 系统的内核版本高于 3.10 ，查看本页面的前提条件来验证你的CentOS 版本是否支持 Docker 。
通过 uname -r 命令查看你当前的内核版本
```
$ uname -r
```
使用 root 权限登录 Centos。确保 yum 包更新到最新。
```
$ sudo yum update
```
卸载旧版本(如果安装过旧版本的话)
```
$ sudo yum remove docker  docker-common docker-selinux docker-engine
```
安装需要的软件包， yum-utils 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的
```
$ sudo yum install -y yum-utils 
```
设置yum源
```
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
可以查看所有仓库中所有docker版本，并选择特定版本安装
```
$ yum list docker-ce --showduplicates | sort -r
```

## install
```
$ sudo yum install docker-ce  #由于repo中默认只开启stable仓库，故这里安装的是最新稳定版17.12.0
$ sudo yum install <FQPN>  # 例如：sudo yum install docker-ce-17.12.0.ce
yum install docker-ce-20.10.9
```

 

启动并加入开机启动
```
$ sudo systemctl start docker
$ sudo systemctl enable docker
```
验证安装是否成功(有client和service两部分表示docker安装启动都成功了)
```
$ docker version
```


