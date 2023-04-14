---
title: calico学习(1)-搭建环境
date: 2023-04-14 15:00:43
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
 kubeadm init --pod-network-cidr=192.168.0.0/16 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.18.5
 ```
 我这里直接使用v1.18.5，因为1.24以后要安装containerd，我机器配置低，装太多东西就卡。
 
 ## 安装calico
 安装calico，需要注意的是，最新版本的calico是3.25，面向的是kubernetes v1.23以后的版本， 去官网查阅以后，发现1.18.5版本的k8s安装calico 3.15即可。
 ```
 kubectl create -f https://docs.projectcalico.org/archive/v3.15/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/archive/v3.15/manifests/custom-resources.yaml

 ```
 街tigera-operator下载完镜像，启动之后，就可以使用以下命令来查看calico组件安装状态:
 ```
 watch kubectl get po -n calico-system
 ```
 ![](/images/16814384249497.jpg)
当所有组件都Running之后，可看kubernetes节点状态:
```
kubectl get node
```
![](/images/16814384614003.jpg)
可以看到节点状态Ready， 说明我们calico安装成功了。后边所有的calico测试都在这个环境里进行。

## 安装calicoctl
calicoctl用来管理calico资源对象的安装、删除、更新等操作。calico对象一般保存在etcd或者Kubernetes里，跟你安装的选项有关，一般默认在Kubernetes,可以使用以下命令安装:
```
curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.15.5/calicoctl
chmod +x calicoctl
```
你还可以安装calicaoctl的容器，使用容器来代替工具，效果是一样的，可以参考官方文档，这里不介绍了。

## 配置calicoctl
calicoctl目标是使用命令行访问calico datastore. 在大多数情况下，calicotl默认不能连接到datastore， 所以需要提供一些连接信息。
1. 默认情况calicoctl会寻找/etc/calico/calioctl.cfg配置文件， 同时你也可以使用--config标志传递相应配置来代替这个配置文件。这个文件的例子:
```
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "etcdv3"
  etcdEndpoints: "http://etcd1:2379,http://etcd2:2379"
  ...

```
2. 同时可以通过环境变量来设置连接信息
### etcd datastore
例:
```
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  etcdEndpoints: https://etcd1:2379,https://etcd2:2379,https://etcd3:2379
  etcdKeyFile: /etc/calico/key.pem
  etcdCertFile: /etc/calico/cert.pem
  etcdCACertFile: /etc/calico/ca.pem

```
通过环境变量:
```
ETCD_ENDPOINTS=http://localhost:2379 calicoctl get bgppeers
```
由于我的机器默认datastore是kubernetes datastore，所以这里不展示内容。
### Kubernetes API datastore
```
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get nodes
```
执行上边的命令就会返回信息如下:
![](/images/16814555314472.jpg)
可以看到没有使用到ssl证书，如果使用到了在/etc/calico/calico.cfg里配置即可。
这样calico的环境就搭建好了