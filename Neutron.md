### 一  概述

Neutron 为 OpenStack 提供网络支持，包括二层交换、三层路由、负载均衡、防火墙和 VPN。

#### 1. 功能

- 二层交换

  Nova 的 instance 通过**虚拟交换机**连接到虚拟局域网。

  Neturon 支持多种交换机，如 Linux 的 虚拟网桥 和 open vswitch。

- 三层路由

  instance 可以配置不同网段的 IP，Neturon 的**虚拟路由**实现跨网段通信。 router 通过 IP 转发、IPtable等技术实现路由和 NAT。

- 负载均衡

- 防火墙

  - Security Group

    通过 IPtable 限制进出 instance 的网络包

  - Firewall-as-a-service

    限制进出虚拟路由器的网络包，也通过 IPtable 实现

#### 2. 基本概念

 1. network：隔离的二层广播域

    - local：主机 instance 只能与同一节点上的同一网络通信

    - flat：不带 tag 的网络，宿主机的物理网卡直接与 Linux Bridge 连接，每个flat 网络独占一块物理网卡；能跨节点与位于同一网络的 instance 通信，

    - vlan：不同vlan的 instance 需通过 router 通信

    - vxlan： 基于隧道技术的 overlay 网络，VNI 标识网络，基于 UDP，二层包通过封装在三层传输

    - gre：与  vxlan 类似的 overlay 网络，区别在于使用 IP 封装而非 UDP

      

    不同 network 在二层上是隔离的。要想通信就要借助路由器。

    network 必须属于某个 project， 一个 project 可以拥有多个 network。即，一对多的关系

    

2. subnet

   一个 IP网络地址结合掩码标识一个子网，一个网络地址可以划分成多个不重叠的子网

   不同 network 里可以配置相同的子网 CIDR 地址块，因为 Neutron的router 是通过 Linux network namespace 实现的，每个router隔离，维护自己的路由表

   subnet 和 network 是一对多关系

   

3. port

   虚拟交换机上的一个端口，**不是传输层的端口**。port上定义了MAC 地址和IP地址，当 instance 的虚拟网卡 VIF(Virtual Interface)绑定到 port 时，port 会将 Mac 和 IP 地址分配给 VIF。

   port 和 subnet 是一对多关系，一个subnet 可以有多个port

### 二、架构

![image-20190205203213864](https://ws3.sinaimg.cn/large/006tNc79gy1fzvtp3swhrj30sw0ly13x.jpg)

其中，

- server：对外提供 网络 API，接受请求，调用 plugin 处理请求
- plugin：处理来自 server 的请求，维护 OpenStack 网络状态，并调用 agent 处理请求，它解决的是“**What**”问题，配置什么样的网络？
- agent：处理来自 plugin 的请求，负责在 provider 上实现真正的网络功能，它解决“**How**”问题，怎么配置这些网络？
- provider：提供网络服务的虚拟或物理设备，如 Linux 网桥， open vswitch等物理交换机
- queue：上面几个子服务的通信中介，默认是 RabbitMQ
- database：存放 OpenStack 网络状态

plugin 按功能分两类：**core plugin**和**service plugin**。core plugin 负责维护 network、subnet、port等资源状态，及对应的 agent(如 Linux Bridge)；service plugin 提供 Firewalls、load-balance等服务，以及对应的 agent。

#### 1. 部署方案

- 控制节点 + 计算节点
- 控制节点 + 网络节点 + 计算节点

#### 2. Neutron Server = APIs + Plugins

![image-20190205205255715](https://ws3.sinaimg.cn/large/006tNc79gy1fzvuamwxmaj30qc0jwk0i.jpg)

#### 3. ML2 Core Plugin

目的：解决 OpenStack 只能有一个 core plugin 从而不支持多个 provider 的问题

传统问题：

1. network、subnet等全局信息只能由一个 core plugin 来维护，而 plugin 和 agent 是一一对应的，如果只有一个 core plugin 那么就只能有一个 core agent，也就不支持多个网络方案。
2. 开发新的 core plugin 工作量大，编写重复的数据库访问代码



ML2 是OpenStack 统一的 Core plugin。

ML2 允许同时使用多种二层网络技术，不同节点使用不同的网络实现技术

![image-20190205211042537](https://ws1.sinaimg.cn/large/006tNc79gy1fzvut5113kj30pk0fkwpz.jpg)



ML2对二层网络进行抽象和建模，引入 type driver 和 mechanism driver，解耦了 Neutron 支持的网络类型和访问这些类型网络的机制，使得 ML2 伸缩性更好，灵活支持各种 type 和 mechanism。

![image-20190205211601564](https://ws2.sinaimg.cn/large/006tNc79gy1fzvuyocil7j30js0mw79z.jpg)

​	

#### 4. Service Plugins 和 Service Agent

service plugin/agent 提供更丰富高级的拓展功能。

![image-20190205212156675](https://ws3.sinaimg.cn/large/006tNc79gy1fzvv4tft9qj313m0i2guy.jpg)

<hr>
* 实线矩形是服务，跟着的虚线是实现



Firewalls 和 security group 的区别在于, 

Firewalls 安全策略位于router，保护整个 project 的 networks；security group 安全策略在于 instance，保护单个 instance。



#### 5. 展开的架构图

![image-20190205213121658](https://ws3.sinaimg.cn/large/006tNc79gy1fzvvemyej8j30w80p4apz.jpg)



### 三、Linux Bridge 实现 Neutron 网络

#### 0. 必要性

Neutron ML2 plugin 默认使用的 mechanism driver 是 OVS，而不是 Linux Bridge。

但是 OVS 的虚拟网络比较复杂，不易理解。而 Linux Bridge 实现的方案更加直观。了解 Linux Bridge 的应用可以对比着理解 OVS 的虚拟网络。



Linux Bridge 的大致原理： 在创建虚拟机时，Hypervisor 会在 host 主机上创建一个虚拟网络设备，这个虚拟网设备就是 Linux Bridge，叫 br0； 接着为虚拟机实例创建一对虚拟接口 veth pair，一个接口位于实例中，充当实例与外界通信的进出口，另一个放在虚拟交换机 br0 上。





#### 1. 配置 Linux-bridge mechanism driver

首先在  `/etc/neutron/neutron.conf` 配置core_plugin

```
core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
```

注：控制节点和计算节点都要这样配置。



然后修改 ML2 的配置文件 `/etc/neutron/plugins/ml2/ml2_conf.ini`

```
mechanism_driver = linuxbridge
```

这里可以指定多种驱动，ML2 会按顺序依次加载。同样，控制节点和计算节点也要这样配置。



Neutron 服务正常启动后，所有节点都会运行 neutron-linuxbridge-agent。



#### 2. Linux Bridge 网络环境中的几种设备

1. tap interface，靠 instance 最近的虚拟设备
2. Linux Bridge，虚拟交换机
3. vlan interface，在 vlan 网络中使用
4. vxlan interface，跨主机多子网场景中使用
5. 物理 interface， 实际网卡

#### 3. DHCP 服务

DHCP 服务由 DHCP agent 提供， DHCP agent 在网络节点上运行，默认通过 dnsmasq 实现 DHCP 功能。

##### 1. 配置 DHCP 服务

`/etc/neutron/dhcp_agent.ini`

```
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
```

创建 network 并在 subnet 上启用 DHCP 时，网络节点上的 DHCP agent 会启动一个 dnsmasq 进程为该 network 提供 DHCP 服务。

启动 dnsmasq 的两个重要参数

-  --dhcp-hostsfile：记录 instance IP 与 MAC 的映射关系，信息来自 Neutron 数据库
- --interface ： 指定 DHCP 服务监听的端口

##### 2. Linux Network Namespace 隔离 dnsmasq 服务

VLAN 在二层网络上将一个物理交换机分隔成几个独立的虚拟交换机；

Network Namespace 在三层上将一个物理三层网络分割成几个独立的虚拟三层网络，每个 Namespace 都有自己的 IPtables、IP、gateway、Firewalls 和 虚拟网络设备。



Neutron 通过 namespace 为每个 network 提供各自的 DHCP 服务和路由服务。每个 dnsmasq 进程位于独立的 namespace 里面，命名为 qdhcp-<network_id>。



事实上，host 主机上也有一个 namespace，叫 root namespace，拥有所有物理和虚拟的设备。

**物理设备只能位于 root namespace 里面**。

新建的 namespace 里面默认只有一个 loopback 接口，docker 的 none 网络模式的原理就在于此。



##### 3. 如何从 dnsmasq 获取 IP

创建 instance 时， Neutron 会为 instance 分配一个 port，包含 MAC 和 IP 地址信息。这些信息会同步到 dnsmasq 的 hosts 文件里。

启动 instance 后

1. instance 广播 DHCPDISCOVER，广播到达 veth 的本地端，然后传给对端。
2. dnsmasq 监听在 veth 的这一对端，收到广播后，检查 hosts 文件，发现有该 instance 的记录，于是把资源信息(IP、子网掩码、网关、dns、租约) 通过 DHCPOFFER 单播给源 instance。
3. instance 收到回应后，通过 DHCPREQUEST 广播给目的 dnsmasq 进程。
4. dnsmasq 单播确认信息 DHCPACK，过程结束。

#### 4. vlan network

vlan 是带 tag 的网络。

在某个物理接口上可以创建多个 vlan 网络(只要 vlan_id 没用完)；创建一个 vlan 会在物理接口上生成一个虚拟的 **vlan interface**，充当该 vlan 的二层"网关"； 但是，instance 并不是直接与 vlan interface 相连，而是通过 **Linux Bridge** 作为中介，这一过程中两次出现了虚拟网络接口对 **veth pair**。

总之，创建一个新的 vlan 不仅会在物理接口上创建一个 vlan interface，还会为新 vlan 里的 instance 创建一个 Linux Bridge。

![image-20190212135616416](https://ws3.sinaimg.cn/large/006tNc79gy1g03llcepnej310m0oa1kx.jpg)

​	图中，eth1.100, eth1.101 是两个 vlan interface; brqXXXX，brqYYYY是对应的 Linux Bridge。

#####  配置 vlan

`/etc/neutron/plugins/ml/ml_conf.ini`

```
[ml2]
tenant_network_type = vlan
...
[ml2_type_vlan]
network_vlan_ranges = default:3001:4000 // 指定用户创建 vlan 的可用 vlan_id
...
[linux_bridge]
physical_interface_mappings = default:eth1
```



#### 5. Routing

不同 vlan 的 instance 无法通过虚拟交换机直接通信，只能在三层通过路由转发。

路由实现了跨子网互联互通的功能。

Neutron 支持物理 router 和虚拟 router 两种方案。物理 router 就是传统的三层通信模式。

##### 虚拟 router

虚拟 router 由 L3 agent 提供。除此之外，它还通过 iptables 提供 Firewalls 和 floating ip 的服务。



L3 agent 的配置

控制节点和网络节点上的`/etc/neutron/l3_agent.ini`

如果 ml2 的 `mechanism_driver` 是 linuxbridge，则

```
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
```

如果是默认的 openvswitch，则

```
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
```



虚拟路由的位置：

![image-20190212143859897](../../../Library/Application%20Support/typora-user-images/image-20190212143859897.png)

上图中的两个 TAP 设备上并没有配置网关，router 怎么完成路由？

L3 agent 为每个 router 创建一个 namespace，通过 veth pair 与 TAP 相连，然后将网关地址配置在位于 namespace 的 veth 上，这样到达 TAP 的流量自然地流到了 veth 上，也就到达虚拟路由器上，由路由器进行后续路由。

router 的 namespace 命名格式是  qrouter-<router_id>



更为细致的网络结构

![image-20190212153217781](https://ws2.sinaimg.cn/large/006tNc79gy1g03oda77ifj310c0u0e81.jpg)



在这里，不用 namespace， 直接为 TAP 配置网关也能使两个 vlan 通信，做法是在控制节点上做路由。

既然这样，为什么还要加一层 namespace ？ 

--> 支持网络重叠。namespace使得每个router拥有自己的路由表，不会与其他路由器冲突，更好支持网络重叠。

不同的用户配置各自的网络时可能产生网络地址重叠，比如配置完全相同的两个子网。而路由表是为**一个网络集合**服务的路由规则，对于完全重叠的网络，路由器不知道该把流量传给哪个网络，或者遵循最长优先匹配原则，把数据发到错误的网段上，故无法应对重叠网络的路由需求。



##### 访问外网

外部网络的类型一般是 flat 或 vlan。

配置的话，和上面类似，先为 flat 或 vlan 网络起一个 label (flat_networks |vlan_networks)，然后在 `network_vlan_ranges`中指明这个 label，最后把这个 label 和 与外网连接的物理接口建立映射(physical_interface_mappings)，重启 Neutron。	



在配置外网 network 的 subnet 时，subnet 的 IP 要和通向外网的网关地址一致。



创建好外网 network 后，要把这个 flat 网络添加(Add Interface)到虚拟router上。



此时，instance 可以直接访问外网，但外网还不能直接访问 instance，因为 instance 没有外网 IP。

Floating IP 解决这个问题



##### Floating IP

当虚拟 router 收到来自 instance 的数据包时，会修改数据包的源地址为自己的外网地址，确保外部数据包能找到自己，这叫做 *Source NAT*。

floating IP 提供静态 NAT 功能，建立外网 IP 与 instance IP 的一一对应关系。floating IP 是在 router 的外网 interface 上配置的。



配置位置： Project --> Compute --> Access&Security



#### Vxlan

##### 原理

vxlan 和 gre 都是一种 overlay 网络。overlay 网络是指建立在其他网络上的网路。类似的概念还有 overlayfs，docker用的就是这种文件系统。该网络中的节点可以看做通过虚拟链路连接，底层可能实现由若干物理链路组成，但这些链路对节点是透明的。

P2P 是 overlay 网络，隧道也是。 vxlan 和 gre 是基于隧道的，自然也属于 overlay 网络。

目前 Linux Bridge 只支持vxlan，OVS 支持 vxlan 和 gre。

**vxlan 是拓展性更强的 LAN**，使用 "MAC-in-UDP"的封装格式，在三层网络上建立一条二层隧道。

优点有

- 支持更多的二层网段。VLAN 用12bit标识 VLAN ID；而 vxlan 用 24bit 标识。
- 更好利用已有的网络路径。 VLAN 使用 STP 协议避免环路，会导致一半的网络路径被 prune 掉；vxlan 使用 UDP 封装传输数据，可以使用所有数据
- 避免 MAC 表耗尽。



VXLAN 使用 VTEP 处理 VXLAN 的封装和解封，**它是隧道两端的进出口**。

每个 VTEP 有一个 IP 接口，配有 IP 地址。VTEP 使用 IP 封装二层帧，并通过隧道在 IP 网路中传输。

二层数据封装在 UDP 中，意味着包含实际主机的 IP - MAC 数据帧基础上，额外从里到外再封装 **VXLAN 头、UDP头、IP 头 和 MAC 头**，解封顺序相反。



##### L2population

它的作用是在 VTEP 上提供 ARP 代理功能，使得 VTEP 能预先知道 VXLAN 网络中 ”instance-VETP 和 instance IP-MAC “的映射关系，避免网络规模变大时产生 ARP 广播风暴。



配置，在 `/etc/neutron/ml2/ml2_conf.ini` 中的 `mechanism_driver`项指定`l2population`；[VXLAN] 块 启用 `l2_population`。



#### Security Group

Neutron 使用 **安全组**和**虚拟防火墙**两种网络安全策略。它们本质上都是利用 iptables 控制网络流量。

##### 安全组

每个 project 默认处在一个默认的安全组中。默认安全组有四条规则：允许所有外出流量，禁止所有进入流量。

想让 instance 被外部访问，可以修改默认安全组或增加新组。

安全组有以下特性：

- 通过宿主机上的 iptables 规则控制进出 instance 的流量
- 安全组作用在 instance 的 port 上
- 安全组的规则都是 allow，不能定义 deny 规则
- instance 可应用多个安全组叠加使用这些安全组的规则



#### FWaaS

##### 原理

Firewall as a Service 是 Neutron 的高级服务。在 subnet 边界上创建防火墙，控制三四层的流量。

和传统防火墙一样， Neutron 在虚拟 router 上部署规则控制流量。

三个概念：

- Firewall
- policy，rule 的集合
- rule，规则条目，包括源和目的 的IP、端口、协议、deny|allow 等动作



firewall 和安全组的区别：

安全组应用在虚拟网卡上，由 L2 agent 实现，安全组保护的是 instance；

Firewall 应用在虚拟 router 上，作用位置比安全组更靠前，Firewall 保护的是 subnet。



##### 启用 Firewalls

`/etc/neutron/fwaas_driver.ini`中

```
driver = neutron_fwaas.services.firewall.drivers.linux:iptables_fwaas.IptablesFwaasDriver
enabled = True
```



`/etc/neutron/neutron.conf`中

```
service_plugins = ..., neutron_fwaas.services.firewall.fwaas_plugin.FirewallPlugin
```

注意，刚创建的防火墙默认阻止所有跨子网流量，需要自己添加访问规则(rule)。



#### Load Balancing as a Service

##### 原理

负载均衡接收前端发来的请求，然后按照某种均衡策略转发给后端资源池中的某个处理单元，以完成处理。

三个概念

- Pool Member。四层实体，拥有IP 和 监听服务的端口
- Pool。由一组 Pool Member 组成，这些 members 通常提供同一类服务
- Virtual IP。定义在 Load Balancer 上的 IP。每个 Member 可能有自己的 IP，但对外服务则是通过 Virtual IP。

OpenStack 默认通过 HAProxy 软件实现负载均衡。

##### 启用 LBaas

`/etc/neutron/services/loadbalancer/haproxy/lbaas_agent.ini`中，

```
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
```

`/etc/neutron/neutron.conf`中,

```
[DEFAULT]
service_plugins = ..., neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPlugin
```

`/etc/neutron/neutron_lbaas.conf`

![image-20190213172117950](https://ws2.sinaimg.cn/large/006tNc79gy1g04x52by05j316w0cmtus.jpg)



LBaas 支持三种均衡策略

- ROUND_ROBIN: 将请求顺序轮询给 Pool 中的 members 处理
- LEAST_CONNECTIONS: 选择当前连接数最少的 member
- SOURCE_IP： 将源 IP 相同的请求发给同一个 member，实现 "sticky session"。

