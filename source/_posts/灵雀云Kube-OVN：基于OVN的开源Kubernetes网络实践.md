---
title: 灵雀云Kube-OVN：基于OVN的开源Kubernetes网络实践
date: 2023-04-13 09:36:04
tags: kubeovn
categories:
- kubeovn
- 网络
---
[原文](https://zhuanlan.zhihu.com/p/464815452)
近日，灵雀云发布了基于OVN的Kubernetes网络组件Kube-OVN，并正式将其在Github上开源。Kube-OVN提供了大量目前Kubernetes不具备的网络功能，并在原有基础上进行增强。通过将OpenStack领域成熟的网络功能平移到Kubernetes，来应对更加复杂的基础环境和应用合规性要求。

目前Kube-OVN项目代码已经在Github 上开源，项目地址为：https://github.com/alauda/kube-ovn。项目使用宽松的Apache 2.0 协议，欢迎更多技术开发者和爱好者前去试用和使用。

网络插件那么多，为什么还需要Kube-OVN？

<!-- more -->

网络插件千千万，为什么还要开发Kube-OVN？从当前Kubernetes网络现状来看，Kubernetes 网络相关的组件非常分散。例如，CNI 负责基础容器网络，它本身只是个接口标准，社区和市场上都有很多各自的实现；集群内的服务发现网络需要依赖 kube-proxy，而 kube-proxy 又有 iptables 和 ipvs 两种实现；集群内的 DNS 需要依赖额外组件kube-dns 或coredns；集群对外访问的负载均衡器服务需要依赖各个云厂商提供的Cloud-Provider；网络策略的 NetworkPolicy 本身只是一个标准接口，社区中也有各自不同的实现。此外还有 ingress，Kubernetes提供的只是一个标准接口，社区中同样有各自的实现。

分散的网络组件导致容器网络流量被分散到了不同的网络组件上，一旦出现问题需要在多个组件间游走逐个排查。在实际运维过程中网络问题通常是最难排查的，需要维护人员掌握全链路上所有组件的使用、原理以及排查方式。因此，如果有一个网络方案能够将所有数据平面统一，那么出现问题时只需要排查一个组件即可。

其次，现有网络插件种类繁多，但是在落地时会发现，每个网络插件由于覆盖的功能集合不同，很难在所有场景使用同一套方案。为了满足不同的客户需求，很多厂商一度同时支持多种网络方案，给自身带来很大负担。在落地过程中，还发现很多传统的网络方案在容器网络中是缺失的。一些高级功能是所有网络插件都无法满足的，比如：子网划分、vlan 绑定、nat、qos、固定 IP、基于acl的网络策略等等。

现有 Kubernetes网络能力是否足够？答案很明显，如果真的已经做够强大落地的时候就不会出现这么多的问题。从更大格局来看，Kubernetes本质上是提供了一层虚拟化网络。而虚拟化网络并不是一个新问题。在OpenStack社区，虚拟网络已经有了长足的发展，方案成熟，OVS 基本已经成为网络虚拟化的标准。于是，灵雀云开始把目光投向OVS 以及 OVS 的控制器OVN。

OVN 简介

OVS 是一个单机的虚拟网络交换机，同时支持 OpenFlow 可以实现复杂的网络流量编程，这也是网络虚拟化的基础。通过OVS 和 OpenFlow 可以实现细粒度的流量控制。如果做个类比，OVS 相当于网络虚拟化里的 Docker。

OVS 只是个单机程序，想生成一个集群规模的虚拟网络就需要一个控制器，这就是 OVN。OVN 和 OVS 的关系就好比 Kubernetes 和 Docker 的关系。OVN 会将高层次的网络抽象转换成具体的网络配置和流表，下发到各个节点的OVS上，实现集群网络的管理。

由于 OVN 最初是为 OpenStack 网络功能设计的，提供了大量 Kubernetes 网络目前不存在的功能：

L2/L3 网络虚拟化包括：

分布式交换机，分布式路由器
L2到L4的ACL
内部和外部负载均衡器
QoS，NAT，分布式DNS
Gateway
IPv4/IPv6 支持
此外 OVN 支持多平台，可以在Linux，Windows，KVM，XEN，Hyper-V 以及 DPDK 的环境下运行。

综上可以看出 OVN 可以覆盖 CNI, Kube-Proxy, LoadBalancer, NetworkPolicy, DNS 等在内的所有 Kubernetes 网络功能，并在原有基础上有所增强。将之前在OpenStack领域内成熟的网络功能往 Kubernetes 平移，也就诞生了灵雀云的开源项目 Kube-OVN.

Kube-OVN 重要功能及实现

目前大部分网络插件的网络拓扑都是一个大二层网络，通过防火墙做隔离，这种形式非常类似公有云早期的经典网络。Kube-OVN在设计网络支出时同时把多租户的场景考虑进来，方便之后更好的扩展高级功能，因此采用了不同的网络拓扑。
![](/images/16813498789050.jpg)
Kube-OVN 网络拓扑

在Kube-OVN的网络拓扑里，不同 Namespace 对应着不同的子网。子网是其中最为重要的概念，之后的内置负载均衡器，防火墙，路由策略以及DNS的网络功能都是和子网绑定的。我们做到了同一台机器的不同 pod 可以使用不同的子网，多个子网之间使用一个虚拟路由器进行网络的联通。为了满足Kubernetes中主机可以和容器互通的网络要求，Kube-OVN在每个主机新增一块虚拟网卡，并接入一个单独的虚拟交换机和集群的虚拟路由器相连，来实现主机和容器网络的互通。

在容器访问外部网络的策略中，Kube-OVN设计了两种方案：一种是分布式 Gateway，每台主机都可以作为当前主机上 Pod 的出网节点，做到出网的分布式。针对企业需要对流量进行审计，希望使用固定IP出网的场景，还做了和 namespace 绑定的 Gateway，可以做到一个 namespace 下的 Pod 使用一个集中式的主机出网。在整个网络拓扑中，除了集中式网关，交换机，路由器，防火墙，负载均衡器，DNS都是分布在每个节点上的，不存在网络的单点。
![](/images/16813498987048.jpg)

Kube-OVN架构图

在实现的过程中，Kube-OVN对组件架构进行了大幅的简化，整个流程中只需要一个全局Controller 和分布在每台机器上的 CNI 插件，就可以组建网络，整体安装十分简单。其中全局Controller 负责监听 APIServer 事件，针对 Namespace，Pod，Service，Endpoint 等和网络相关的资源变化对 OVN 进行操作。同时 Controller 会把操作 OVN 的结果，例如 IP ，Mac，网关，路由等信息回写到对应资源的 annotation 中。而 CNI 插件则根据回写的 annotation 信息操作本机的 OVS 以及主机网络配置。通过这种架构Kube-OVN将 CNI 插件和、Controller 和 OVN 解耦，通过 Apiserver 传递所需的网络信息，方便快速开发迭代。

Kube-OVN选择使用最基础的 annotation ，而不是 CRD、API Aggregation 或者 Operator，也是出于降低复杂度的考虑，使用户和开发者都能够快速上手。

Kube-OVN主要具备五大主要功能：

Namespace 和子网的绑定，以及子网间访问控制；
静态IP分配；
动态QoS；
分布式和集中式网关；
内嵌 LoadBalancer；
子网是Kube-OVN中最重要的概念，由于子网和 namespace 关联，因此只需要在 namespace 中设置对应的 annotation 就可以完成子网的功能。Kube-OVN支持子网cidr，gateway，exclude_ips 以及 switch_name 的设置。同时支持基于子网的IP隔离，用户可以轻松实施基本的网络隔离策略。在后台实现上，Kube-OVN会监听 namespace 的变化，并根据变化在 ovn 中创建并设置虚拟交换机，将其和集群路由器关联，设置对应的 acl，dns 和 lb。

静态IP的使用可以直接在 Pod 中加入对应的 annotation，Controller 在发现相关 annotation 时会跳过自动分配阶段，直接使用用户指定的 IP/Mac。对应多 Pod 的工作负载，例如 Deployment、DaemonSet，可以指定一个 ip-pool，工作负载下的 Pod 会自动使用ip-pool中未使用的地址。

在QoS功能中，分别实现了 ingress 和 egress 的带宽限制，用户可以在 Pod 运行时通过动态调整 annotation 来实现 QoS 的动态调整，而无需重启 Pod。在后台的实现中， OVN 自带的 QoS 功能工作在 Tunnel 端口，无法对同主机间 Pod 的互访做 QoS 控制。因此Kube-OVN最终通过操作 OVS 的 ingress_policing_rate 和 port qos 字段来实现 QoS 的控制。

在网关设计中，OVN的网关功能有一些使用限制，需要单独的网卡来做 overlay 和 underlay 的流量交换，使用起来比较复杂，为了能够适应更广泛的网络条件并简化用户使用，Kube-OVN对网关部分进行了调整。使用策略路由的方式根据网络包的源 IP 选择下一跳的机器，通过这种方式将流量导入所希望的边界网关节点，然后在网关节点通过 SNAT 的方式对外进行访问。这种方式用户只需要在 namespace 中配置一个网关节点的 annotation 就可以配置对应的流量规则。此外，Kube-OVN也支持非SNAT将容器IP直接暴露给外网的场景，这种情况下只需要外部添加一条静态路由指向容器网络，就可以实现 Pod IP 直接和外部互通。

内嵌的 LoadBalancer 使用 OVN 内置的 L2 LB，这样集群内部的服务发现功能可以直接在 OVS 层面完成，不需要走到宿主机的 iptable 或者 ipvs 规则，可以将 kube-porxy 的功能整合到 Kube-OVN 中。

开源计划 & RoadMap

目前Kube-OVN已经在 github 上开源。OVN 安装比较繁琐，Kube-OVN特意做了安装的简化，现在只需要两个 yaml 就可以部署一个完整的 Kube-OVN。在使用方面也做了优化，通过直观的 annotation 即可对网络进行配置。此外还针对不使用任何 annotation的情况内置了一套默认配置。用户可以使用默认的子网，默认的IP分配策略，默认的分布式网关以及内嵌的负载均衡器，这些都不需要任何配置就可以默认启用。欢迎大家体验试用，多给我们提供反馈和意见。

关于Kube-OVN，近期灵雀云将主要着力于实现三大目标：第一，集中式网关的高可用，消灭整个架构中最后一个单点；第二，内嵌 DNS，去除 Kube-DNS/CoreDNS 的依赖，将整个数据平面用 OVN 进行统一；第三，NetworkPolicy实现。

长期来看，Kube-OVN未来将实现对DPDK 和 Hardware Offload 的支持，解决 Overlay 网络性能问题；将更多的 OVS 监控和链路追踪工具引入 Kubernetes；将OpenStack社区的网络功能向Kubernetes平移，打造更完整的网络体系。