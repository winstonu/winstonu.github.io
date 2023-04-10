---
title: Harbor在MacOS下编译开发
date: 2023-04-07 14:56:52
tags: Harbor、registry
categories:
- Harbor
- 云原生
---
Harbor是一个云原生容器镜像仓库，Harbor比原生的registry多了用户管理与权限管理功能，所以使用起来还是很方便， 这里记录一下Harbor在MacOS下的编译与扩展开发. 本文是基于Harbor 2.7.0 release测试，别的版本可能会有出入。
## requirment
- Harbor在本地编译时，由于所需的依赖包过多，而且大部分是从dockerhub官方仓库下载，所以在国内有的时候拉取镜像很慢，需要搭配镜像加速器使用。
- 另外可能还需要https_proxy
- golang 1.20.x
- docker 23.x


需要注意的是，在macos中， docker与宿主机的通信不是通过Bridge，这个时候如果要使用到了https_proxy,且https_proxy是监听在宿主机上的127.0.0.1:1087这种地址的时候，需要使用host.docker.internal:1087来访问https_proxy（如果不用则不需要设置)

## 配置镜像加速器
```
vi ~/.config/daemon.json
加入: registry-mirrors: ["https://xxxx"]
```
重启docker，然后使用docker info检查镜像加速器是否设置成功。

## Build
```
mkdir -p github.com/goharbor
git clone 
mv harbor github.com/goharbor/
make install
```
上述过程如果完整运行完成，则就可以直接访问Harbor.

## M1上编译
需要注意的是，如果使用的是M1电脑，由于CPU是Arm64架构，上述镜像做完之后，是启动不起来的，因为编译出来的文件是arm64架构的，需要做交叉编译为amd64
在Makefile文件中修改 go build，需要修改的有：compile_core, compile_jobservice
直接在go build后边加上:
```
-e GOOS=linux -e GOARCH=amd64
```

## 需要注意的点
如果在对swagger api没有修改的情况下，因为编译compile_core之前每次都要执行: gen_api，如果可以将gen_api在第一次生成之后，就删除掉这一步骤，这样编译起来就会更快了。

下一篇将会Harbor源码做简要的分析