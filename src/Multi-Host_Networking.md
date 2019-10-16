### 第三章 跨主机网络

------

只在单个主机上使用容器，第2章中介绍的技术就足够了。然而，如果单个主机的容量不足以处理您的工作负载，或者您想要更多的弹性，那么您需要水平扩展。

#### 多主机容器网络

当水平扩展时，最终会得到一个跨机器网络，也称为机器集群，简称集群。现在，有一些问题：不同主机上的容器如何互相通信？容器间如何通讯，并且和外部网络通讯？如何保持集群中的状态(例如IP地址分配)一致?与现有网络基础设施的集成点是什么?那么安全策略呢?

为了解决这些问题，我们将在本章的剩余部分中回顾多主机托管网络的技术。由于不同的用例和环境有不同的需求，我将避免为特定的项目或产品提供推荐。你应该意识到权衡利弊，做出明智的决定。

#### 多主机网络的选择

简而言之，Docker本身提供了对覆盖网络(在主机特定网络上创建跨主机的分布式网络)的支持，以及为第三方提供商提供的网络工作插件。

实践中经常使用许多多主机容器联网选项，尤其是在Kubernetes环境中。这些包括:

+ Flannel by CoreOS 
+ Weave Net by Weaveworks
+ Metaswitch’s Project Calico
+ Open vSwitch from the OpenStack project
+ OpenVPN

另外，Docker本身提供了多主机联网。

##### flannel

CoreOS的flannel是一个虚拟网络，为每个主机分配子网，供容器运行时使用。对于Kubernetes来说，每个容器豆荚在集群中都有一个惟一的、可路由的IP。flannel支持一系列后端，比如VXLAN、AWS VPC和默认L2 UDP网络。flannel的优点是它减少了做端口映射的复杂性。Red Hat的项目Atomic使用flannel。

##### Weave Net

Weaveworks的WeaveNet创建了一个虚拟网络，将部署在多台主机上的Docker包含服务器连接起来。应用程序使用网络就像所有的容器都插入同一个网络交换机一样，不需要考虑端口映射和链接。Weave网络上的应用程序容器提供的服务可以让外界访问，而不管这些容器在哪里运行。

类似地，现有的内部系统也可以暴露在应用程序容器中，以反映其位置。Weave可以穿越防火墙，在部分连接的网络中运行。流量可以加密，允许主机通过不受信任的网络连接。您可以在Alvaro Saurin的博客文章“使用Weave discovery自动部署Docker主机”中了解更多Weave的发现。

##### Project Calico

Metaswitch的Project Calico使用标准IP路由来精确定位，如RFC 1105中定义的值得尊敬的边界网关协议(BGP)和网络工具来提供第三层解决方案。相比之下，大多数其他网络解决方案通过将第2层流量封装到更高的层来构建覆盖网络。

主操作模式不需要封装，它是为数据中心设计的，组织可以在数据中心控制物理网络结构。还可以查看Canal，它将Calico的网络政策执行与丰富的Calico和法兰绒覆盖和非覆盖网络连接结合在一起。

##### Open vSwitch

Open vSwitch是一种多层虚拟交换机，旨在通过编程扩展实现网络自动化，同时支持标准管理接口和协议，如NetFlow、IPFIX、LACP和802.1ag。另外，它被设计用于支持跨多个物理服务器的分发，并用于Red Hat的Kubernetes发行版OpenShift中，这是Xen、KVM、Proxmox VE和VirtualBox中的默认开关。它还被集成到许多私有云系统中，例如OpenStack和oVirt。

##### OpenVPN

OpenVPN是另一个具有商业功能的OSS项目，它允许您使用TLS创建虚拟专用网(vpn)。这些vpn还可以用于通过公共internet安全地将容器连接到彼此。如果您想尝试基于Docker的设置，我建议您看看DigitalOcean的“[如何在Ubuntu 14.04的Docker容器中运行OpenVPN](https://www.digitalocean.com/community/tutorials/how-to-run-openvpn-in-a-docker-container-on-ubuntu-14-04)”演练教程。

#### Docker Networking

Docker 1.9引入了一个新的Docker命令`network`。这样，容器就可以动态地连接到其他网络，每个网络都可能由不同的网络驱动程序支持。

2015年3月，Docker公司收购了SDN创业公司SocketPlane，并将其产品更名为Overlay Driver。从Docker 1.9开始，这是多主机网络的默认设置。覆盖驱动程序通过对等通信扩展了普通的桥接模式，并使用可插入的键-值存储后端来分发集群状态、支持领事、etcd和ZooKeeper。

为了了解更多，我建议看看以下的博客文章:

+ Aleksandr Tarasov’s “[Splendors and Miseries of Docker Network](http://developerblog.info/2015/11/16/splendors-and-miseries-of-docker-network/)”
+ Project Calico’s “[Docker 1.9 Includes Network Plugin Support and Calico Is Ready](https://www.projectcalico.org/docker-libnetwork-is-almost-here-and-calico-is-ready)”

#### 管理方面的考虑

在本章的最后一节中，我们将讨论一些您应该注意的管理方面:

***IPVLAN***

Linux内核版本3.19引入了IP-per-container特性。这将为主机上的每个容器分配一个惟一的、可路由的IP地址。高效地说，IPVLAN采用一个网络接口，并创建多个虚拟网络接口，这些虚拟网络接口具有分配给它们的不同MAC地址。

这个功能是由谷歌的Mahesh Bandewar贡献的，它在本质上类似于macvlan驱动程序，但更灵活，因为它同时适用于L2和L3。如果您的Linux发行版已经有了内核> 3.19，那么您就很幸运了。否则，您还不能从这个特性中获益。

***IP address management (IPAM)***

多主机联网的关键挑战之一是将IP地址分配给集群中的容器。可以采取两种策略:要么在你现有的(公司)网络中找到实现它的方法，要么衍生出一个正交的、实际上是隐藏的网络层(即一个覆盖网络)。注意，使用IPv6时这种情况比较宽松，因为找到一个空闲地址空间应该容易得多。

***Orchestration tool compatibility***

本章中讨论的许多多主机网络解决方案实际上都是封装Docker API并为您配置网络工作。这意味着在选择一个工具之前，您应该确保检查与您正在使用的容器编排工具的兼容性问题。你会在第四章找到更多关于这个主题的内容。

***IPv4 versus IPv6***

到目前为止，大多数Docker部署都使用标准的IPv4，但是IPv6却有一些优势。Docker自2015年2月发布v1.5以来一直支持IPv6;然而，IPv6在Kubernetes的支持还没有完成。IPv4-land日益增长的地址短缺可能会鼓励更多的IPv6部署，也会消除网络地址转换(NAT)，但目前还不清楚[何时会达到临界点](https://www.youtube.com/watch?v=eF50OxZ5u4o)。

#### 总结

在本章中，我们回顾了多主机网络选项，并讨论了管理问题，如IPAM和编配。在这个关键时刻，您应该很好地理解低级别的单主机和多主机联网选项及其挑战。现在让我们继续讨论容器编排，看看它如何依赖于网络以及如何与之交互。