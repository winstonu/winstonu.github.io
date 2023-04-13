---
title: calico学习(1)-搭建环境
tags:
---
## 搭建kubernets
我只有一个云服务器，2c2G，所以搭建一个单节点的Kubernetes，在这个单节点的Kubernetes开启我的calico之旅。
## 安装工具
安装Kubernetes必要的工具: kubectl、kubeadm、kubelet
kubeadm: k8s的安装工具
kubelet: 运行在k8s每个节点上，用来启动pod
kubectl : 用来与cluster交互

执行下边的命令:

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```
在腾讯云上竟然下载不了，所以把上边配置的官网yum源删除掉， 配置成阿里云的:
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```
重新执行安装，一切正常。（注意，安装时没有指定版本，默认是安装最新版本，我操作时最新版本是1.27.0)
## 配置cgroup driver
在v1.22版本，如果用户不配置KubeletConfiguration， kubeadm默认就就是systemd driver.
另外容器运行时建议使用systemd driver而不是cgroupfs driver，主要是因为kubeadm像管理systemd service一样管理kubelet
## 创建cluster
 ```
 kubeadm init
 ```
 我这里直接使用v1.18.5，因为1.24以后要安装containerd，我机器配置低，装太多东西就卡。
 
 ## 安装calico
 安装calico
 ```
 kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
 ```