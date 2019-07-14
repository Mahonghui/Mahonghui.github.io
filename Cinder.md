### 功能

- 提供REST API 使用户能够查询和管理 volume、volume snapshot 和 volume type
- 提供 scheduler 调度 volume 创建请求
- 通过 driver 架构支持多种后端存储方式，包括 LVM、NFS。

### 架构

![image-20190203182000314](https://ws4.sinaimg.cn/large/006tNc79gy1fztemw10jfj30fy0jm7ax.jpg)

- cinder-api

  接收 API 请求，调用 cinder 子服务处理请求

- cinder-scheduler

选择合适存储节点创建 volume

- cinder-volume

  管理 volume 的服务，与 provider 协作管理 volume的生命周期。运行 cinder-volume 的节点叫存储节点

- volume provider

  数据的存储设备，为 volume 提供物理存储空间

  cinder-volume 支持多种 provider

- Message Queue

  各个子服务通过消息队列协作

- database

  存放Cinder的必要数据，一般是MySQL。和nova一样，数据库全部安装在**控制节点**上。

  ### 物理部署方案

  OpenStack是个分布式系统，理论上所有服务部署在任何地方，只要网络能够互相联通。

  cinder-api是控制节点帽子，cinder-volume是存储节点帽子。可以在同一台机器上安装两个服务，让这台机器既是控制节点，也是存储节点。RabbitMQ 和 MySQL 通常部署在控制节点上。

  cinder-provider放在哪里？

  一般，volume-provider 是独立的。cinder-volume 使用 **driver**与 volume-provider 通信并协调工作。所以只需要把driver 和 cinder-volume 放在一起。

  #### 子服务流程图

  ![image-20190203190441715](https://ws4.sinaimg.cn/large/006tNc79gy1fztfxen0d4j30ni0ken18.jpg)

  

  #### 设计思想

  Cinder 沿用 Nova相同的设计思想。

  **API 前端服务； Scheduler调度服务；Worker 工作服务； Driver 框架**



### Cinder 组件

1. cinder-api

2. cinder-scheduler

3. cinder-volume

   cinder-volume本身不存储物理块，它通过 driver 与 volume-provider 通信，共同负责卷的生命周期管理

#### volume操作

**Attach** 

​	*存储节点上本地的逻辑卷通过 attach 操作挂载到计算节点上的 instance。*

​	而计算节点和存储节点通常位于不同物理机，采用**iSCSI协议**在主机间传输卷块。

​	![image-20190205121609322](https://ws3.sinaimg.cn/large/006tNc79gy1fzvfd0lgdnj30wc0u01kx.jpg)

​	其中，

 - target： 提供 iSCSI 资源的设备，iSCSI 服务器

 -  initiator：使用 iSCSI 资源的设备，iSCSI 客户端

   

**Detach**

解除 volume 和 instance 的关联



**Extend**

扩大 volume 的容量，状态为 available 才能被 extend。正在被 attach 的volume要先 detach。

![image-20190205122928904](https://ws3.sinaimg.cn/large/006tNc79gy1fzvfqtsdthj30qw0lgadm.jpg)

​	extend 操作不需要 scheduler 的介入，因为要被拓展的 volume 肯定已经被挂载在某个 instance 上了。



**Delete**

状态为 available 的 volume 才能被 delete。

cinder-volume 执行的是“安全”删除：将 volume 数据抹掉后才删除。LVM 使用`dd`操作将 LV 的数据清零。



**Snapshot**

Snapshot 可以为 volume 创建快照，快照保存了 volume 当前的状态，以后可以通过快照恢复。

如果一个 volume 存在快照，则这个 volume 不能被删除。



**Backup**

将 volume 备份到别的地方(备份设备)，将来通过`restore`操作恢复。

Backup 和 Snapshot 的区别：

- Snapshot 依赖于源，不能独立存在； Backup 不依赖源 volume，即使源 volume 不存在也可以进行 restore。
- Snapshot 与源 volume 通常存放在一起；而 Backup 存放在独立的备份设备中，有自己的备份方案和实现。
- Snapshot 提供快捷的回溯功能； Backup 具有容灾功能。



**Restore**

1. 在存储节点上创建一个空白的 volume

2. 将 backup 的数据 copy 到空白的 volume

   ![image-20190205124909850](https://ws1.sinaimg.cn/large/006tNc79gy1fzvgbbr70aj311i0u0n99.jpg)



**Boot from Volume**

Volume 除了当做 instance 的数据盘，也可以作为启动盘。