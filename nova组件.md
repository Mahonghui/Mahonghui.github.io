### Nova 子服务协作流程

![image-20190202181913045](https://ws1.sinaimg.cn/large/006tNc79gy1fzs8zueqvuj31ai0oeqbg.jpg)

消息队列使用**rabbitMQ**：OpenStack组件的各个子服务之间不直接调用，全部通过消息队列异步调用。

数据库使用**MySQL**：每个OpenStack组件在MySQL中都有自己的数据库。

### 1. nova api

nova api是nova服务的门户，所有对nova的调用都会经由 nova api 处理。nova api向外界暴露nova服务的HTTP REST api接口。

作为终端用户，不会直接调用api。命令行、dashboard负责调用这些api，进一步封装。

nova api 对 收到的 HTTP api 的处理：

1. 检查参数有效性
2. 调用nova其他子服务
3. 格式化调用结果返回给客户端

OpenStack使用 实例(Instance) 描述虚拟机。

### 2. nova-scheduler

创建instance时，用户会提出需要多少资源。OpenStack把这些资源需求定义在flavor中，flavor就是一个个资源套餐。

Flavor定义了 VCPU、RAM、DISK和Metadata。

在`/etc/nova/nova.conf`中，配置三个配置项：schedulerdriver, scheduleravailable_filters, schedulerdefaultfilters。

1. Filter scheduler 是nova-scheduler默认的调度器。
   - 通过过滤器(Filter)选择满足条件的计算节点
   - 通过权重(Weight)选择在权重最大的计算节点上创建instance

2. Filter

   scheduleravailable_filters 定义nova可用的过滤器，默认是nova自带的所有过滤器。

   schedulerdefaultfilters 则定义实际使用的过滤器。

   默认值：

   ![image-20190203104120126](https://ws1.sinaimg.cn/large/006tNc79gy1fzt1dq6nabj31uo09gkgv.jpg)

   - RetryFilter: 刷掉之前已经调度过的节点
   - AvailabilityZoneFilter: 考虑容灾性和隔离性，将计算节点划分在不同的Zone中。初始时将所有的节点放在默认的nova zone中。
   - RamFilter：将不满足 flavor 内存需求的计算节点过滤掉。OpenStack在计算节点内存时，认定可用内存是实际内存的倍数。
   - DiskFilter：将不满足 flavor 磁盘需求的计算节点过滤掉。同样允许超载。
   - CoreFilter：将不满足 flavor CPU需求的计算节点过滤掉。nova 默认的filters没有包含这个filter。
   - ComputeFilter：将 nova-compute 服务不能正常工作的节点过滤掉。
   - ComputeCapabilitiesFilter: 根据节点的计算特性筛选。
   - ImagePropertiesFilter: 根据镜像属性筛选节点。
   - ServerGroupAntiAffinityFilter：尽量让 instance 散落在不同的计算节点上。
   - ServerGroupAffinityFilter：尽量让 instance 聚集在一个计算节点上。

3. Weight

   经过第2步的过滤，scheduler过滤出一组符合各个过滤器条件的计算节点。

   如何确定最终在哪个节点上创建 instance？ **权重**。

   节点 i 的权重默认定义为 i 节点**空闲的内存量**。

   空闲内存越多，权重越大，被选中概率越大。

   

### 3. nova-compute

nova-compute 和 hypervisor 一起实现 OpenStack 对 instance 的管理。

1. driver 架构支持多种 hypervisor

   配置不同的driver， 使用不同的hypervisor。各自的hypervisor实现统一的接口，以driver的形式提供给OpenStack使用。

   nova-compute的作用有两个：报告计算节点状态； 管理 instance 生命周期。

   - 报告节点状态。nova-compute 通过**hypervisor**得知节点的资源使用情况，进而发送给OpenStack其他子服务(如 Filters) 使用。

   - 管理 instance。

     nova 创建 instance 分为4步：

     1. 按flavor套餐为 instance 准备资源
     2. 创建 instance的镜像文件，如果没有就从Glance处下载，**类似docker镜像的构建过程**
     3. 创建 instance的XML定义文件，**类似Dockerfile**
     4. 创建虚拟网络并启动虚拟机，创建NFV(网桥，NAT)

   

### 4. nova-conductor

nova-compute通过 nova-conductor 访问MySQL数据库。

这么做是因为：

1. 安全性：如果**计算节点**的nova-compute 直接访问**控制节点**上的 MySQL数据库，那么就要在计算节点上存储数据库的连接信息，如果任一计算节点被攻击，数据库位置也就暴露了。因此，compute 通过 conductor 访问MySQL，而conductor部署在控制节点上。

2. 伸缩性

   compute 与 conductor 通过消息队列通信，允许配置多个 conductor 实例，适应大量或少量数据库访问场景。



### 5. 日志相关

1. devstack日志同一放在`/opt/stack/logs`目录下。

![image-20190203120017662](https://ws1.sinaimg.cn/large/006tNc79gy1fzt3nwroepj31240lsazq.jpg)

每个服务都有自己的日志

Nova 以 ”**n-**“ 开头

Glance 以 “**g-**”开头

Cinder 以 “**c-**”开头

Neutron 以 “**q-**” 开头

2. 创建 instance 流程

   ![image-20190203121507027](https://ws4.sinaimg.cn/large/006tNc79gy1fzt438xt6pj30zw0u07hy.jpg)

3. 软/硬 启动
   - soft reboot 只重启操作系统，instance 仍在运行，相当于 Linux的`reboot`命令
   - hard reboot等效于 关机再开机

4. Lock/Unlock

   被加锁的 instance 不允许执行重启或删除等关键操作。

   注： admin用户不受限制，它相当于`root`用户。

5. suspend 和 pause比较

   pause 是短时间暂停，它把虚拟机状态保存在**内存**中。

   suspend 是长时间暂停，它把虚拟机状态保存在**磁盘**上。

   suspend后 instance 处于 shutdown 状态，但 hypervisor 仍为 instance 保留资源，为实现该 instance 的快速启动。若回收这些资源，使用`Shelve`操作。Shelve会将 instance 作为 image 上传到 Glance，然后在宿主机中删除该 instance。`Unshelve`还原被shelve的 instance。

6. Migrate

   Migrate 允许将 instance 从当前计算节点迁移到另一个计算节点。Migrate不要求源节点和目标节点共享存储，要求计算节点间配置ssh无密码访问。

7. Live Migrate

   Migrate 执行的是"冷迁移"，把 instance 关掉再迁移。而 Live Migrate 属于热迁移，instance 不会停机。

   Live Migrate分两种：

   - 源节点和目的节点没有共享存储， image + instance一起迁移。
   - 源节点和目的节点共享存储，只需要迁移 instance。

   热迁移的条件：

   - 源节点和目的节点的CPU类型一致
   - Libvirt 版本一致
   - 源节点和目标节点能互相识别对方的主机名
   - 源节点和目标节点的 `nova.conf`中指明热迁移使用 TCP 协议
   - instance 使用 config driver 保存其 metadata
   - 源和目标节点的 Libvirt TCP 远程监听服务得打开

8. Resize

Resize负责调整 instance 的vCPU、内存和磁盘资源。具体需要多少资源定义在 `flavor`。

**resize 通过选择新的flavor来调整资源的分配**。resize的本质是在 migrate 同时选择新的 flavor，而 migrate 是 resize 特例：flavor没有变化的resize。

9. Evacuate

   Rebuild 可以修复损坏的 image。

   Evacuate 可以在宿主机崩溃，nova-compute 无法工作的情况下将节点上的 instance 迁移到其他计算节点上，实际上在其他计算节点上重建 instance。前提是 image 的镜像文件必须共享存储。

### instance 操作总结

![image-20190203180050628](https://ws2.sinaimg.cn/large/006tNc79gy1fzte305qh8j318w0kqnjl.jpg)