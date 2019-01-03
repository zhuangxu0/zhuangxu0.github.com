---
layout: post
title: OpenStack-Networking(1)
date: 2018-12-21 17:05:07
image: '/assets/img/'
description: OpenStack中网络服务基础，两种网络类型provider or self-service network,网络隔离技术vlan and vxlan.
main-class: shu
color: '#7AAB13'
tags:
- OpenStack
- Network
categories:
twitter_text: OpenStack networking, vlan and vxlan
introduction: OpenStack中网络服务基础，两种网络类型provider or self-service network,网络隔离技术vlan and vxlan.
---

# I.OpenStack中的网络服务
Openstack的网络服务提供API,供用户在云环境下建立网络连接和寻址，处理虚拟网络资源（networks、switch、subnets、router、port）的创建和管理。

# II.网络类型
OpenStack中的网络服务可以允许，创建、配置networks and subnets。OpenStack networking可以使一个tenant创建多个networks,允许tenant自定义network的ip分配，
由于Openstack通过user namespace做了多个tenant之间的隔离，所以允许不同tenant的ip address overlap。存在两种类型的网络 `tenant network` 和 `provider network`。

## 2.1 Provider networks
Provider networks提供layer-2的联通，可以选择DHCP和metadata service的支持。这些networks使用VLAN来identify and separate. Provider networks会连接或
映射到数据中心的真实存在的layer-2 network。只要administrators可以管理provider network,因为需要一些配置来连接或映射到真实的physical network infrastructure.

Provider netwoks提供layer-2的联通性，没有routers支持的功能。通常OpenStack networking中的处理layer-3操作的组件最能影响performance and reliability.

## 2.2 Self-service networks
Self-service networks是普通tenant可以使用创建不需要administrators的权限。这些网络是完全虚拟化的，需要virtual router连接到provider networks或external networks。
Self-service networks通常提供DHCP和metadata service to instances。

这些networks大多情况下使用overlay protocols像vxlan，gre，可以支持多少个networks。IPv4 self-service使用私有ip段，通过源地址NAT去访问provider networks. 
Floating IP 使得通过目的地址NAT可以从provider networks访问instance.

OpenStack中网络服务使用layer-3 agent来实现routers,layer-3 agent运行的好坏影响self-service networks的质量以及使用它的instances。

用户为projects中的连接创建tenant networks，默认情况下projects间的网络是隔离的。

# III.网络isolation和overlay技术
## 3.1 VLAN
VLAN的出现是为了解决网络中广播域（传统情况下router接口分割出的网络）过大的问题，通过Vlan可以自定义广播域，隔离layer-2网络。VLAN虚拟局域网是一组逻辑上的devices，
根据功能将它们组织起来，使相互之间通信好像它们在同一个网段中一样。VLANs定义了layer-2中的广播域，可以在switch中定义virtual bridge，
每一个virtual bridge对应定义一个新的广播域vlan,不同的vlan间traffic不能directly pass，需要经过router.
![Vlan定义网络的例子](/assets/img/net/1-2.png)

VLAN真实的组网形式，以及VLAN内流量通信与VLAN间流量通信的真实过程,网络物理结构（物理布线形态）和逻辑结构（从网络层以上的层面观察到的网络结构）
[参考]http://network.51cto.com/art/201409/450885.htm

**Vlan的网络物理结构**
router为了支持Vlans，上面的端口必须支持汇聚链路，也就是router与switch连接的一个物理端口，可以分割成多个虚拟端口。VLAN将交换机从逻辑上分割成了多台，
因而用于VLAN间路由的路由器，也必须拥有分别对应各个VLAN的虚拟接口。
![Vlan的物理网络支持](/assets/img/net/1-3.png)

Vlan间流量帧的转发需要通过router,三层网络的转发
![Vlan间流量转发](/assets/img/net/1-4.png)

**Vlans间虚拟机迁移**
在不使用vlan的原始情况下，需要改变网络物理拓扑结构（机器所连的网段接口）。想将192.168.1.0/24这个网络上的计算机A转移到192.168.2.0/24上去
![原始情况下虚机迁移](/assets/img/net/1-5.png)

当使用vlan后，可以通过划分switch上端口所属的vlan，然后更改主机A的ip和默认网关，不用更改物理网络拓扑。
![Vlan场景下虚机迁移](/assets/img/net/1-6.png)

**网络的物理结构和逻辑结构**
![网络的物理布线结构](/assets/img/net/1-7.png)

网络的逻辑结构是以router为中心划分的不同网段
![网络的逻辑结构](/assets/img/net/1-8.png)

## 3.2 GRE and VXLAN
GRE和VXLAN是封装协议，来创建overlay networks。VXLAN是网络虚拟化技术，用来解决云计算中网络的扩展问题。它使用layer 4的UDP包封装layer 2
的以太网frames,来实现大二层网络。

**VXLAN的作用**
VXLAN的出现是为了解决1)原有技术（VLAN支持4096个）有限的网络隔离；2)虚机迁移范围受限问题（云计算中虚机迁移是常态性业务）。

`虚机迁移的要求`
为了保证虚机迁移过程中业务不中断，需要保证虚机IP、MAC地址不变，这就要求虚机迁移必须在一个二层网络中（不同网段组成的网络是同一个三层网络，
一个网段组成的网络是同一个二层网络）。传统VLAN划分网络的情况下，虚机的迁移限制在比较小的地域范围。

对于第1）VXLAN将使网络工程师更容易扩展云计算环境，同时在逻辑上隔离云应用和tenant。云环境是多tenant，每个tenant需要自己所有的logical network,
也就是有自己的网络标识（Network id).传统上在云计算中使用VLANs来隔离tenant和apps，不过vlan最多支持4096个network id.Vxlan扩展了virtual LAN的地址范围。
对于第2）虚机迁移，vxlan将vm发出的报文封装后通过vxlan隧道（两端的vtep）传输，隧道两端的vm不感知传输网络的物理架构，这样对于同一网段的vm而言，
即使物理位置不在同一个二层网络，但网络的逻辑结构上处于同一个二层网络，这样解决虚拟迁移范围受限问题。

> http://network.51cto.com/art/201409/450885.htm
> https://www.zhihu.com/question/22907100
> http://network.51cto.com/art/201811/587745.htm
