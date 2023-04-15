---
title: calico2-选择最佳网络方案
tags: Network
Categories:
  - Network
  - Calico
  - Kubernetes
date: 2023-04-16 00:18:00
---

## 重点
熟悉calico各种网络方案，可以根据自己需求选择最佳方案。
包含运行插件能力，像CNI和IPAM，还有底层网络类型，non-overlay或者overlay模型， 使用或者不使用BGP等。
## Kubernetes网络基础
Kubernetes网络模型有以下2个特点:
- 每个Pod有一个独立的IP
- Pod在每个节点上可以不使用NAT进与别的节点上的Pod通信
在这种模型之下，可以把Pod当作VM或者物理节点看待，实现一些基础的商品分配、服务发现、负载均等， 还能使用基础的网络策略来对流量进行管理。

## CNI插件
CNI(Container Network Interafce)是一种标准API规范，各种网络插件可以实现这些规范，提供给Kubernetes使用， Kubernetes在创建或者销毁Pod的时候会调用这些API. 有2种类型的CNI插件
- CNI Network Plugin: 负责添加或者删除Pod到Kubernetes网络里， 包含创建、删除Pod的网络设备
- CNI IPAM Plugin： 负责分配、回收IP资源，分配CIDR给每个Node, 获取underlay网络分给Pod的IP

## Cloud Provider integrations
Kubernetes cloud provider integrations 是云服务商的一些特殊控制器用来提供Kubernetes的Network. 基于不同的云提供商的编程实现，就能清楚Pod的流量。

## Kubenet
Kubenet是Kuernetes原生的网络插件，这个插件没有实现跨Node网络和策略，更多的时候这个插件和Cloud Provider integrations一起使用，能够在cloud provider network里设置路由用来节点之间的通信， 或者在一些单节点环境中工作，需要注意的是，Kubenet与calico不兼容。

## Overlay networks（覆盖网络)
什么是overlay network?
所谓overlay network就是基于别的网络上构建出来的上层网络就是overlay.在Kubernetes中， Underlay网络不清楚Pod ip也不清楚Pod在节点的运行情况，Overlay网络可以用来处理pod-to-pod的流量。

覆盖网络的工作原理是将底层网络不知道如何处理的网络数据包封装在底层网络知道如何处理的外层数据包中(例如使用 pod IP 地址)。有2种比较常见的网络协议用来封包：vxlan和IP-in-IP.

优点:
使用Overlay network不用太关心底层网络的实现。
缺点：
- 性能损耗， 由于多了一些封包拆包工作，消耗一些CPU。
- 另外就是Pod ip在集群之外没有路由

## Cross-subnet overlays(跨子网 overlay)
除了标准的overlay network(vxlan, ipip), calico也支持跨子网的vxlan和ipip.在这种模式下， underlaying network充当了二层网络角色，数据帧在同一个subnet里传输的时候不做封包， 所以性能没有损失。在不同subnet里做封包操作， 像是一个标准overlay network一样。

## Pod IP在集群之外的路由
没的Kubernetes network实现的区别可能是: Pod IP能否跨跃集群之外路由。
### 不可路由(Not routable)
如果不可路由，那么当一个Pod ip需要访问集群之外的地址时， Kubernetes会使SNAT(Source Network Address Translation)，目的是为了改变Pod的原始IP地址，相当于是宿主机地址到Pod地址的一种映射， 任何回包也会自动的通过DNAT回到Pod里.

外部的服务无法直接访问Pod里的服务，需要通过Kubernetes servcies或者Kubernetes Ingress.

### 可路由
内部Pod可以直接访问集群之外服务， 外部也可以直接访问Pod。有以下特点:
- 省去的SNAT，DNAT， 可以直接与边界网络集成，方便调试与观察日志
- 不再需要使用Kubernetes service, ingress.

缺点:
IP在不同的集群里要使用不同的CIDR，如果集群规模较大或者集群较多，会导致IP耗尽。

那是什么来决定是否可路由呢 ？
- 如果你使用了Overlay network, 则Pod无法路由
- 如果没有使用overlay network,则是否可以路由取决于你使用了CNI插件类型、云提供商集群，以及使用的BGP.

## BGP

## 关于Calico网络
Calico灵活的架构模块包含:
- Calico Network plugin: 使用veth pair来连接pods和宿主机的网络L3路由。这个L3架构避免了许多其它Kubernetes网络解决方案里L2桥的不必要的复杂性和性能开销。
- Calico CNI IPAM plugin
- Overlay network modes
- Non-overlay network modes
- Network policy enforcement： Calico 的网络策略执行引擎实现了 Kubernetes 网络策略的全部功能，加上 Calico 网络策略的扩展功能。这与 Calico 内置的网络模式，或任何其他 Calico 兼容的网络插件和云提供商集成相结合。


## 网络选项
预置: 
calico默认使用的是non-overlay， 使用BGP to Peer和宿主机物理网络使得Pod的IP可以在集群外路由(可以通过配置来禁止在集群外路由)。
如果不能将BGP同步到物理网络，你可以使用一个L2网络， Calico 只是在集群中的节点之间使用BGP互联。这不是一个overlay netwrok, 因为边界网络没有pod ip的路由规则.
另外calico 与其它云厂商提供的plugin兼容，比如: AWS、Azure、Google Cloud、IBM Cloud

## 结束
以上概念深入了解能帮助我们更好的来选择我们网络模式，如果还不清楚，初学者可以直接使用overlay netwrok vxlan，而不用太关心各种选项的不同。