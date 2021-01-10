### 背景
容器云开发环境昨天遇到了个问题，在master（node1）节点的应用连接mongodb突然crash掉了，尝试去node1的pod去ping，但是ping不通，然后去node1的主机上ping，但是能ping通。 因为我们用的Ovn网络组件来做cni。所以先考虑是不是网络组件出问题了。但是其他主机上的机器没有影响。


### 理解kube-ovn网络组件的数据流向

- eth0:主机网卡
- genev_sys_6081：ovn产生的tunnel设备(各个节点之间通讯依赖这个tunnel)
- ovn0: eth0的veth，挂载在ovs br-int.

三种大场景：
Pod<->Pod同主机/跨主机
Pod<->ServiceIP(在ovn0做了lb)
Pod<->Host 主机host(内部集群)/(外部集群)外部地址

同主机Pod数据流向：PodNetVeth--->ovs-br-int---->PodNetVeth
跨主机主机Pod数据流向：PodNetVeth--->ovs-br-int---->genev_sys_6081(封装)---->eth0----->eht0(跨主机的)---->genev_sys_6081(解封装)---->ovs-br-int---->PodNetVeth(跨主机的)

Pod<->Host数据流向: PodNetVeth---->ovs-br-int---->ovn0---->eth0----->外部Host(策略路由，这里和`分布式网关`和`集中式网关`有直接的关系，默认用的是分布式网关:认走的ovn0出口到主机上的默认路由出去，集中式网关：它的路由策略会走向一台特定的主机(不一定是集群中的主机))。


## 如何排查问题这个网络问题？ 

1. 首先通过容器运提供的容器debug功能，进入容器尝试`ping` Mongodb的那台主机.但是ping不通。
   - 提出假设一:是否是coredns有问题或者apiserver的问题。
   - 提出假设二:是不是ovs组件假死了。
   - 提出假设三:是不是ovn0网络的包没到eth0（执行`traceroute 10.1.1.7`看到最后一跳是在ovn0上就跳不下去了。并且在host层主机上ping到mongodb的那台服务器是可以的）

>通过看日志和实验去检验自己的假设

### 是否是coredns有问题或者apiserver的问题？
首先,coredns组件一般会和nameserver相关，不过还是想尝试拍排查一下，万一是了呢？所以，查看coredns在node上的相应的日志，看到访问一些域名timeout(我们ping的是ip所以和一些域名的超时不相关）。所以尝试重启node1节点的容器，看到日志正常启动。然后再尝试ping节200的mongodb服务器。还是ping不通。然后查看coredns的日志，是否输出错误。看了，并没有。然后我们又查看对应的apiserver的日志，然后也没有相关的错误日志。和预想的假设完全不符合所以继续排查。

### 是不是ovs组件假死了。
那么尝试启动ovs这个网络组件吧。重启了，再继续ping还是不行。



### 是不是ovn0网络的包没到eth0？

那么看ping的包到没有到eth0，利用tcpdump ICMP协议的包`tcpdump -i eth0 icmp -vvvnn or src host 10.1.1.7`，然后去随便找一个node1的pod，去ping`10.1.1.7`的ip是能到达eth0的，但是目标端没有返回icmp的包。那么考虑是否是iptable的问题呢？当时我在node1上的找了一个pod的ip是`10.16.0.106`,那么我尝试增加一条iptable的规则`iptables -t nat -I POSTROUTING -s 10.16.0.106/16 -j MASQUERADE`,尝试去ping，执行了ping能通。那么这是iptable的问题了。想到了iptable那么就会想到k8s的kube-proxy组件问题，那么通过容器云查看node1上的kube-proxy的日志查询，查看日志报错，日志是报没找到serviceaccount，第一反应查看service-account的是否存在，查看了是存在的，那么重启当前的当前node1上的kube-proxy的pod，然后尝试去ping` 10.1.1.7` 或者其他的ip，尝试都是可以的，那么都是可以的。



### 总结
这个问题，从一开始就应该看kube-proxy的日志的。当网络出现问题，那么就要看对应的主机上的日志，包括网络组件的日志，还有就是coredns/apiserver/kube-proxy。从这些组件去着手，看日志排查问题。整体排查下来也就10分钟左右，这次写成文章的形式没有图文并茂，下次会把遇到的问题截图下来，形成blog。


