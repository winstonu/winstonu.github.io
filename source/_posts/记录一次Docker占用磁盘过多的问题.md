---
title: 记录一次Docker占用磁盘过多的问题
tags: Docker
date: 2023-04-10 16:43:03
categories:
- Docker
- 云原生
---

## 概述
项目中一直在使用Drone CI来做CICD，周末的时候突然发现CD失败了，登陆到机器中查看了一下，发现机器的磁盘空间满了，这里记录一下问题的排查过程。
## 问题
先直接来看一下问题现象:
![](/images/16811314262961.jpg)

可以看到，是由于磁盘空间不够导致CD失败。
## 排查过程
上来直接进入构建服务器，输入以下命令：
```
df -h
```
查看到磁盘占用情况:
![](/images/16811088150686.jpg)
现在空间是被我清掉了，在清掉之前， docker var/lib/docker/overlay2/xxx/merged其中一个文件夹里的磁盘占用高达32G.

查看Docker Volume
```
Docker system df
```
![](/images/16811089305443.jpg)
也可以清晰的看到docker的镜像与建立的volumes占用的磁盘空间，现在的图片是被我清掉以后，在清掉之前，volume占用空间32G， 50G大小的磁盘被占满，所以才会出现CD的时候写入报错的情况。

## 解决问题
既然我们已经发现了是由于Docker创建的volume里的数据占用太多导致的问题，那么问题就简单了，我们直接先使用docker volume prune来删除掉没有被容器使用的volume

通过
```
docker system df
```
可以查看volume的links是否为0,为0时说明没被使用
![](/images/16811173663912.jpg)

我们确定好哪一个volume占用过大，且LINKS=1时， 使用以下命名查看空上volume被哪个容器使用:
```
docker ps -a --filter volume=xxxxxxx
```
最终看到这个volume是我之前测试仓库时创建的registry仓库，已经停了好久了，所以直接将这个容器删除，重新执行:
```
docker volume prune
```
这样这个volume就被删除了，磁盘空间也释放了。

