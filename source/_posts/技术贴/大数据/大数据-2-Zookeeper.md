---
title: 大数据-2-Zookeeper
tags:
  - zookeeper
categories:
  - technical
  - zookeeper
toc: true
declare: true
date: 2020-11-23 14:54:27
---

# 三、 Zookeeper

## 3.1 概述

Zookeeper 是一个开源的<font color='red'>**分布式协调服务框架**</font> ，主要用来解决分布式集群中应用系统的**一致性问题**和**数据管理问题**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123153745.png)

<!-- more -->

在单机模式中，可以通过锁机制带实现对于共享资源的访问协调

但是在网络集群的多机模式下，每个主机都要通过网络去访问共享资源，这样实现的叫做**分布式锁**，具体的核心工作就是由**Zookeeper**来管理

对于网络中的**多个冗余存储的共享资源**，Zookeeper在于解决多机读写的数据同步性问题

举例来说：两个共享资源其中一个被写时，那么他的冗余备份不应当被其他主机读取，因为这会导致数据的不同步。所以这就是Zookeeper要解决的事。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123154359.png)

## 3.2 特点

Zookeeper 本质上是一个**分布式文件系统（本身就是一个集群）**, **适合存放小文件**，也可以理解为一个数据库

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123154441.png)

在上图左侧, Zookeeper 中存储的其实是一个又一个 **Znode**, Znode 是 Zookeeper 中的节点

Znode 是有路径的, 例如 /data/host1 , /data/host2 , 这个路径也可以理解为是Znode 的 Name
Znode 也可以携带数据, 例如说某个 Znode 的路径是 /data/host1 , 其值是一个字符串 "192.168.0.1"

<font color='green'>理解：Znode可以有文件的功能存储数据，也有文件夹的功能-能够发展期子节点</font>

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123155125.png)

正因为 Znode 的特性, 所以 **Zookeeper 可以对外提供出一个类似于文件系统的视图**, 可以
通过操作文件系统的方式操作 Zookeeper**使用路径获取 Znode**

* 获取 Znode 携带的数据

* 修改 Znode 携带的数据

* 删除 Znode

* 添加 Znode

## 3.3 架构

Zookeeper集群是一个基于<font color='red'>主从架构</font>的**高可用集群**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123155741.png)

每个服务器承担如下三种角色中的一种

* ***Leader*** 一个Zookeeper集群同一时间只会有一个实际工作的Leader，它会<u>发起并维护与各Follwer及Observer间的心跳</u>。**所有的写操作(<font color='green'>事务性操作</font>)必须要通过Leader完成**再由Leader将写操作广播给其它服务器。

* **Follower** 一个Zookeeper集群可能同时存在多个Follower，它会<u>响应Leader的心跳</u>。**Follower可直接处理并返回客户端的<font color='green'>读</font>请求**，同时会将**<font color='red'>写</font>请求<u>转发</u>给Leader处理**，并且<u>负责在Leader处理写请求时对请求进行投票</u>。

* Observer 角色与Follower类似，但是**无投票权**。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201123160201.png)

## 3.4 应用场景

### 3.4.1 数据发布/订阅

​	**数据发布/订阅系统**,需要发布者将数据发布到Zookeeper的节点上，供订阅者进行数据订阅，进而达到动态获取数据的目的，实现配置信息的集中式管理和数据的动态更新。
​	发布/订阅一般有两种设计模式：**推模式**和**拉模式**，服务端主动将数据更新发送给所有订阅的客户端称为推模式；客户端主动请求获取最新数据称为拉模式.
Zookeeper采用了推拉相结合的模式，客户端向服务端注册自己需要关注的节点，<u>一旦该节点数据发生变更，那么服务端就会向相应的客户端推送Watcher事件通知，客户端接收到此通知后，主动到服务端获取最新的数据。</u>

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125193114.png)

### 3.4.2 命名服务

​	命名服务是分步实现系统中较为常见的一类场景，分布式系统中，被命名的实体通常可以是<u>集群中的机器</u>、<u>提供的服务地址或远程对象</u>等，通过命名服务，客户端可以根据指定名字来获取资源的实体，在分布式环境中，上层应用仅仅需要一个**全局唯一**的名字。Zookeeper可以实现一套分布式全局唯一ID的分配机制。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125193331.png)

通过调用Zookeeper节点创建的API接口就可以创建一个顺序节点（后面介绍），并且在API返回值中会返回这个节点的完整名字，利用此特性，可以生成**全局ID**，其步骤如下
1.  客户端根据任务类型，在指定类型的任务下通过调用接口创建一个顺序节点，如"job-"。
2.  创建完成后，会返回一个完整的节点名，如"job-00000001"。
3.  客户端**拼接type类型和返回值**后，就可以作为全局唯一ID了，如"type2-job-00000001"。

### 3.4.3 分布式协调/通知

​	Zookeeper中特有的Watcher注册于异步通知机制，能够很好地实现分布式环境下不同机器，甚至不同系统之间的协调与通知，从而实现对数据变更的实时处理。通常的做法是不同的客户端都对Zookeeper上的同一个数据节点进行Watcher注册，监听数据节点的变化（包括节点本身和子节点），若数据节点发生变化，那么所有订阅的客户端都能够接收到相应的Watcher通知，并作出相应处理。在绝大多数分布式系统中，系统机器间的通信无外乎<font color='green'>**心跳检测、工作进度汇报和系统调度。**</font>

 * <font color='orange'>心跳检测</font>

   不同机器间需要**检测到彼此是否在正常运行**，可以使用Zookeeper实现机器间的心跳检测，基于其**临时节点**特性（临时节点的生存周期是客户端会话，客户端若宕机后，其临时节点自然不再存在），可以让不同机器都在Zookeeper的一个指定节点下创建临时子节点，不同的机器之间可以根据这个临时子节点来判断对应的客户端机器是否存活。通过Zookeeper可以大大减少系统耦合。

* <font color='orange'>工作进度汇报</font>

  通常任务被分发到不同机器后，需要实时地将自己的任务执行进度汇报给分发系统，<u>可以在Zookeeper上选择一个节点，每个任务客户端都在这个节点下面创建**临时子节点**</u>，这样**<font color='red'>不仅可以判断机器是否存活，同时各个机器可以将自己的任务执行进度写到该临时节点中去</font>**，以便中心系统能够实时获取任务的执行进度。

* <font color='orange'>系统调度</font>

  Zookeeper能够实现如下系统调度模式：分布式系统由<u>控制台和一些客户端系统两部分构成</u>，控制台的职责就是需要将一些指令信息发送给所有的客户端，以控制他们进行相应的业务逻辑，后台管理人员在控制台上做一些操作，实际上就是修改Zookeeper上某些节点的数据，Zookeeper可以把数据变更以时间通知的形式发送给订阅客户端

### 3.4.4 分布式锁

​	**分布式锁**用于控制分布式系统之间同步访问共享资源的一种方式，可以保证不同系统访问一个或一组资源时的一致性，**主要分为排它锁和共享锁**。

*  **排它锁又称为写锁或独占锁**，若事务T1对数据对象O1加上了排它锁，那么在整个加锁期间，<u>只允许事务T1对O1进行读取和更新操作</u>，其他任何事务都不能再对这个数据对象进行任何类型的操作，直到T1释放了排它锁。

* ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125194606.png)

  ① 获取锁，在需要获取排它锁时，所有客户端通过调用接口，在/exclusive_lock节点下创建临时子节点/exclusive_lock/lock。**Zookeeper可以保证只有一个客户端能够创建成功**，没有成功的客户端需要注册/exclusive_lock节点**监听**。
  ② 释放锁，当获取锁的客户端宕机或者正常完成业务逻辑都会导致临时节点的删除，此时，**所有在/exclusive_lock节点上注册监听的客户端都会收到通知**，可以重新发起分布式锁获取。

* **共享锁又称为读锁**，若事务T1对数据对象O1加上共享锁，**那么当前事务只能对O1进行读取操作**，**其他事务也只能对这个数据对象加共享锁**，直到该数据对象上的所有共享锁都被释放。在需要获取共享锁时，所有客户端都会到/shared_lock下面创建一个临时顺序节点

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125194758.png)

### 3.4.5 分布式队列

​	有一些时候，多个团队需要共同完成一个任务，比如，A团队将Hadoop集群计算的结果交给B团队继续计算，B完成了自己任务再交给C团队继续做。这就有点像业务系统的**工作流一样，一环一环地传下去.**分布式环境下，我们同样需要**一个类似单进程队列的组件，用来实现跨进程、跨主机、跨网络的数据共享和数据传递**，这就是我们的分布式队列。

## 3.5 概念

### 3.5.1 Zookeeper数据模型

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125204937.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125204912.png)

### 3.5.2 Znode的节点类型

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125205046.png)

会话：客户端连接服务器的过程

**Znode的作用在于：**

1. **维系心跳，判断临时节点是否存在来判断主机是否宕机**
2. **工作进度汇报，存储任务执行进度**

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125205833.png)





## 3.6 Zookeeper的选举机制

Leader选举是保证分布式数据一致性的关键所在。当Zookeeper集群中的一台服务器出现以下两种情况之一时，需要进入Leader选举。

### 3.6.1. 启动时期的Leader选举

若进行Leader选举，则至少需要两台机器，这里选取3台机器组成的服务器集群为例。在集群初始化阶段，当有一台服务器Server1启动时，其单独无法进行和完成Leader选举，当第二台服务器Server2启动时，此时两台机器可以相互通信，每台机器都试图找到Leader，于是进入Leader选举过程。选举过程如下

* <font color='green'>(1)每个Server发出一个投票</font>。由于是初始情况，Server1和Server2都会将自己作为Leader服务器来进行投票，每次投票会包含所推举的服务器的**myid**和**ZXID**，使用(myid, ZXID)来表示，此时Server1的投票为(1, 0)，Server2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。

  **myid越大那么服务器的权重越大，ZXID越大说明节点越新，两者越大选中的概率越大**

* <font color='green'>(2) 接受来自各个服务器的投票</font>。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器。

* <font color='green'>(3) 处理投票</font>。针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK，PK

  规则如下

  · 优先检查ZXID。ZXID比较大的服务器优先作为Leader。

  · 如果ZXID相同，那么就比较myid。myid较大的服务器作为Leader服务器。

  对于Server1而言，它的投票是(1, 0)，接收Server2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时Server2的myid最大，于是更新自己的投票为(2, 0)，然后重新投票，对于Server2而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。

* <font color='green'>(4) 统计投票</font>。每次投票后，服务器都会统计投票信息，判断是否已经**有过半机器**接受到相同的投票信息，对于Server1、Server2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader。
* [(5) 改变服务器状态]()。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125200107.png)

**因为过半机制，所以一般Zookeeper集群都是<font color='red'>奇数</font>个节点**

### 3.6.2 运行时期的Leader选举

在Zookeeper运行期间，Leader与非Leader服务器各司其职，即便当有非Leader服务器宕机或新加入，此时也不会影响Leader，但是一旦Leader服务器挂了，那么整个集群将暂停对外服务，进入新一轮Leader选举，**其过程和启动时期的Leader选举过程基本一致过程相同。**

## 3.7 Zookeeper环境搭建

### 3.7.1 搭建环境流程

集群规划

| 主机名 | IP              | Mac地址           | Myid |
| :----- | --------------- | ----------------- | ---- |
| node1  | 192.168.178.100 | 00:50:56:24:BA:6B | 1    |
| node2  | 192.168.178.101 | 00:0c:29:b1:c1:ce | 2    |
| node3  | 192.168.178.102 | 00:50:56:24:7d:00 | 3    |

####  ***第一步：下载zookeeeper的压缩包，下载网址如下***

下载Zookeeper包：http://archive.apache.org/dist/zookeeper

我们在这个网址下载我们使用的zk版本为3.4.9

下载完成之后，上传到我们的linux的/export/softwares路径下准备进行安装

#### ***第二步：解压***

 解压zookeeper的压缩包到/export/servers路径下去，然后准备进行安装

```shell
cd /export/software
tar -zxvf zookeeper-3.4.9.tar.gz -C ../servers/ 
```

####  ***第三步：修改配置文件***

 第一台机器修改配置文件

 ```shell
cd /export/servers/zookeeper-3.4.9/conf/
cp zoo_sample.cfg zoo.cfg
mkdir -p /export/servers/zookeeper-3.4.9/zkdatas/
 ```

 `vim zoo.cfg`

```shell
dataDir=/export/servers/zookeeper-3.4.9/zkdatas
# 保留多少个快照  (保留多少个生成的日志文件数)
autopurge.snapRetainCount=3
# 日志多少小时清理一次
autopurge.purgeInterval=1
# 集群中服务器地址  集群中myid==1 的服务器为node01，并开启2888和3888两个端口
server.1=node01:2888:3888
server.2=node02:2888:3888
server.3=node03:2888:3888
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125203728.png)

####  ***第四步：添加myid配置***

 在第一台机器的

 /export/servers/zookeeper-3.4.9/zkdatas /这个路径（这个路径需要手动创建）下创建一个文件，文件名为myid ,文件内容为1

 `echo 1 > /export/servers/zookeeper-3.4.9/zkdatas/myid` 

####  ***第五步：安装包分发并修改myid的值***

 安装包分发到其他机器

 第一台机器上面执行以下两个命令

 `scp -r /export/servers/zookeeper-3.4.9/ node02:/export/servers/`

 `scp -r /export/servers/zookeeper-3.4.9/ node03:/export/servers/`

 第二台机器上修改myid的值为2

 `echo 2 > /export/servers/zookeeper-3.4.9/zkdatas/myid`

 第三台机器上修改myid的值为3

 `echo 3 > /export/servers/zookeeper-3.4.9/zkdatas/myid`

####  ***第六步***：三台机器启动zookeeper服务

 三台机器启动zookeeper服务

 这个命令三台机器都要执行

 `/export/servers/zookeeper-3.4.9/bin/zkServer.sh start`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125203914.png)

 查看启动状态

 `/export/servers/zookeeper-3.4.9/bin/zkServer.sh status`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125204146.png)

-----

## 3.8 Zookeeper的Shell 客户端操作

### 3.8.1 登录

登录Zookeeper客户端

```shell
bin/zkCli.sh -server node01:2181
```

客户端的端口就是在conf/zoo.cfg中的clientPoint字段

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125210551.png)

### 3.8.2 客户端操作命令

| 命令                             | 说明                                          | 参数                                                         |
| -------------------------------- | --------------------------------------------- | ------------------------------------------------------------ |
| `create [-s] [-e] path data acl` | 创建Znode                                     | -s 指定是顺序节点（永久序列化节点）<br>-e 指定是临时节点<br>两个参数都不加=>永久节点 |
| `ls path [watch]`                | 列出Path下所有子Znode                         |                                                              |
| `get path [watch]`               | 获取Path对应的Znode的数据和属性               |                                                              |
| `ls2 path [watch]`               | 查看Path下所有子Znode以及子Znode的属性        |                                                              |
| `set path data [version]`        | 更新节点                                      | version 数据版本                                             |
| `delete path [version]`          | 删除节点, 如果要删除的节点有子Znode则无法删除 | version 数据版本                                             |
| `rmr path`                       | 删除节点, 如果有子Znode则递归删除             |                                                              |
| `setquota -n|-b val path`        | 修改Znode配额                                 | -n 设置子节点最大个数<br>-b 设置节点数据最大长度             |
| `history`                        | 列出历史记录                                  |                                                              |

1：创建普通节点

 `create /app1 hello`

2: 创建顺序节点

`create -s /app3 world`

3:创建临时节点

`create -e /tempnode world`

4:创建顺序的临时节点

`create -s -e /tempnode2 aaa`

5:获取节点数据

   `get /app1`

6:修改节点数据

   `set /app1  xxx`

7:删除节点

   delete  /app1 删除的节点不能有子节点

​    rmr    /app1 递归删除

Znode 的特点

- 文件系统的核心是 `Znode`
- 如果想要选取一个 `Znode`, 需要使用路径的形式, 例如 `/test1/test11`
- Znode 本身并不是文件, 也不是文件夹, Znode 因为具有一个类似于 Name 的路径, 所以可以从逻辑上实现一个树状文件系统
- ZK 保证 Znode 访问的原子性, 不会出现部分 ZK 节点更新成功, 部分 ZK 节点更新失败的问题
- `Znode` 中数据是有大小限制的, 最大只能为`1M`
- `Znode`是由三个部分构成
  - `stat`: 状态, Znode的权限信息, 版本等
  - `data`: 数据, 每个Znode都是可以携带数据的, 无论是否有子节点
  - `children`: 子节点列表

Znode 的类型

- 每个`Znode`有两大特性, 可以构成四种不同类型的`Znode`
  - 持久性
    - `持久` 客户端断开时, 不会删除持有的Znode
    - `临时` 客户端断开时, 删除所有持有的Znode, **临时Znode不允许有子Znode**
  - 顺序性
    - `有序` 创建的Znode有先后顺序, 顺序就是在后面追加一个序列号, 序列号是由父节点管理的自增
    - `无序` 创建的Znode没有先后顺序
- `Znode`的属性
  - `dataVersion` 数据版本, 每次当`Znode`中的数据发生变化（只要使用set就会增加）的时候, `dataVersion`都会自增一下
  - `cversion` 节点版本, 每次当`Znode`的节点发生变化的时候, `cversion`都会自增
  - `aclVersion` `ACL(Access Control List)`的版本号, 当`Znode`的权限信息发生变化的时候aclVersion会自增
  - `zxid` 事务ID
  - `ctime` 创建时间
  - `mtime` 最近一次更新的时间
  - `ephemeralOwner` 如果`Znode`为临时节点, `ephemeralOwner`表示与该节点关联的`SessionId`（当前的会话ID）

## 3.10 Watch机制

### 3.10.1 概念

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125213704.png)

<font color='red'>注意:watch机制只能使用一次，想要使用多次必须多次添加</font>

Watch机制的两种作用：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125214309.png)

### 3.10.2 操作

添加Watch机制

`get /hello20000000003 watch`

当另一个主机改变watch的znode节点时，会触发**事件封装**:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125214836.png)

具体含义见下方通知机制

## 3.11 通知机制

- 通知类似于数据库中的触发器, 对某个Znode设置 `Watcher`, 当Znode发生变化的时候, `WatchManager`会调用对应的`Watcher`
- 当Znode发生删除, 修改, 创建, 子节点修改的时候, 对应的`Watcher`会得到通知
- `Watcher`的特点
  - **一次性触发** 一个 `Watcher` 只会被触发一次, 如果需要继续监听, 则需要再次添加 `Watcher`
  - 事件封装: `Watcher` 得到的事件是被封装过的, 包括三个内容 `keeperState, eventType, path`

| KeeperState       | EventType           | 触发条件                 | 说明                               |
| ----------------- | ------------------- | ------------------------ | ---------------------------------- |
|                   | None                | 连接成功                 |                                    |
| SyncConnected     | NodeCreated         | Znode被创建              | 此时处于连接状态                   |
| SyncConnected     | NodeDeleted         | Znode被删除              | 此时处于连接状态                   |
| **SyncConnected** | **NodeDataChanged** | Znode数据被改变          | 此时处于连接状态                   |
| SyncConnected     | NodeChildChanged    | Znode的子Znode数据被改变 | 此时处于连接状态                   |
| Disconnected      | None                | 客户端和服务端断开连接   | 此时客户端和服务器处于断开连接状态 |
| Expired           | None                | 会话超时                 | 会收到一个SessionExpiredException  |
| AuthFailed        | None                | 权限验证失败             | 会收到一个AuthFailedException      |

会话

- 在ZK中所有的客户端和服务器的交互都是在某一个`Session`中的, 客户端和服务器创建一个连接的时候同时也会创建一个`Session`
- `Session`会在不同的状态之间进行切换: `CONNECTING`, `CONNECTED`, `RECONNECTING`, `RECONNECTED`, `CLOSED`
- ZK中的会话两端也需要进行心跳检测, 服务端会检测如果超过超时时间没收到客户端的心跳, 则会关闭连接, 释放资源, 关闭会话

## 3.12 Zookeeper的JavaAPI操作

原生的java API较为麻烦，使用框架Curator较为方便

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125215118.png)

Curator有2.x.x和3.x.x两个系列的版本，支持不同版本的Zookeeper。其中Curator 2.x.x兼容Zookeeper的3.4.x和3.5.x。而Curator 3.x.x只兼容Zookeeper 3.5.x，并且提供了一些诸如动态重新配置、watch删除等新特性。

### 3.12.1 项目组件

| 名称      | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| Recipes   | Zookeeper典型应用场景的实现，这些实现是基于Curator Framework。 |
| Framework | Zookeeper API的高层封装，大大简化Zookeeper客户端编程，添加了例如Zookeeper连接管理、重试机制等。 |
| Utilities | 为Zookeeper提供的各种实用程序。                              |
| Client    | Zookeeper client的封装，用于取代原生的Zookeeper客户端（ZooKeeper类），提供一些非常有用的客户端特性。 |
| Errors    | Curator如何处理错误，连接问题，可恢复的例外等。              |

### 3.12.2 Maven依赖

Curator的jar包已经发布到Maven中心，由以下几个artifact的组成。根据需要选择引入具体的artifact。但大多数情况下只用引入curator-recipes即可。

| GroupID/Org        | ArtifactID/Name           | 描述                                                         |
| ------------------ | ------------------------- | ------------------------------------------------------------ |
| org.apache.curator |                           | 所有典型应用场景。需要依赖client和framework，需设置自动获取依赖。 |
| org.apache.curator | curator-framework         | 同组件中framework介绍。                                      |
| org.apache.curator | curator-client            | 同组件中client介绍。                                         |
| org.apache.curator | curator-test              | 包含TestingServer、TestingCluster和一些测试工具。            |
| org.apache.curator | curator-examples          | 各种使用Curator特性的案例。                                  |
| org.apache.curator | curator-x-discovery       | 在framework上构建的服务发现实现。                            |
| org.apache.curator | curator-x-discoveryserver | 可以喝Curator Discovery一起使用的RESTful服务器。             |
| org.apache.curator | curator-x-rpc             | Curator framework和recipes非java环境的桥接。                 |

根据上面的描述，开发人员大多数情况下使用的都是curator-recipes的依赖，此依赖的maven配置如下：

```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.12.0</version>
</dependency>
```

由于版本兼容原因，采用了2.x.x的最高版本。

项目中的maven依赖：

```xml
 <!--<repository>-->
        <!--<id>cloudera</id>-->
        <!--<name></name>-->
        <!--<url>https://repository.cloudera.com/artifactory/cloudera-repos</url>-->
    <!--</repository>-->

    <dependencies>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>2.12.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>2.12.0</version>
        </dependency>

        <dependency>
            <groupId>com.google.collections</groupId>
            <artifactId>google-collections</artifactId>
            <version>1.0</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>1.7.25</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

### 3.12.3 创建节点

```java
@Test
    //创建节点
    public void createZnode() throws Exception {
        //1. 定制一个重试策略
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 1);     //参数1：重试间隔时间(毫秒)，参数2：重试最大次数
        //2. 获取一个客户端对象
        /**
         * 参数1：服务器连接字符串，多个逗号分隔
         * 参数2：会话超时时间
         * 参数3：连接超时时间
         * 参数4：上面的重试策略
         */
        String connectionStr = "192.168.178.100:2181,192.168.178.101:2181,192.168.178.102:2181";
        CuratorFramework client = CuratorFrameworkFactory.newClient(connectionStr, 5000, 8000, retryPolicy);
        //3. 开启客户端
        client.start();
        //4. 创建节点
        /**
            PERSISTENT(0, false, false),            永久
            PERSISTENT_SEQUENTIAL(2, false, true),  永久可序列化
            EPHEMERAL(1, true, false),              临时
            EPHEMERAL_SEQUENTIAL(3, true, true);    临时可序列化
            creatingParentsIfNeeded代表可以主动帮你创建父节点
            withMode代表创建节点的类型
            forPath两个参数就是路径+数据，一个参数就是路径
         */
        client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath("/xxx/abc", "xwj".getBytes()); //修改你的节点路径名
        //5. 关闭客户端  注意关闭会话就会导致临时节点消失
        client.close();
    }
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125225423.png)

打印日志基本就没问题

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201125225450.png)

### 3.12.4 修改节点数据

```java
//设置节点数据
    @Test
    public void setZnodeData() throws Exception {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 1);
        String connectionStr = "192.168.178.100:2181,192.168.178.101:2181,192.168.178.102:2181";
        CuratorFramework client = CuratorFrameworkFactory.newClient(connectionStr, 5000, 8000, retryPolicy);
        //3. 开启客户端
        client.start();
        /**
         * 使用setData方法
         */
        client.setData().forPath("/1126", "改掉你".getBytes());
        client.close();
    }
```

### 3.12.5 获取节点数据

```java
//获取节点数据
    @Test
    public void getZnodeData() throws Exception {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 1);
        String connectionStr = "192.168.178.100:2181,192.168.178.101:2181,192.168.178.102:2181";
        CuratorFramework client = CuratorFrameworkFactory.newClient(connectionStr, 5000, 8000, retryPolicy);
        client.start();
        /**
         * getData
         */
        byte[] bytes = client.getData().forPath("/1126");
        System.out.println(new String(bytes));
        client.close();
    }
```

### 3.12.6 Watch机制

```java
@Test
    public void watchZnode() throws Exception {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 1);
        String connectionStr = "192.168.178.100:2181,192.168.178.101:2181,192.168.178.102:2181";
        CuratorFramework client = CuratorFrameworkFactory.newClient(connectionStr, 5000, 8000, retryPolicy);
        client.start();

        //创建一个TreeCache对象（相当于对节点树形结构的一个映射），指定要监控的节点路径
        TreeCache treeCache = new TreeCache(client, "/hello");
        //自定义一个监听器
        treeCache.getListenable().addListener(new TreeCacheListener() {     //需要实现一个监听器接口
            @Override
            public void childEvent(CuratorFramework curatorFramework, TreeCacheEvent treeCacheEvent) throws Exception {
                //自定义监听对象
                ChildData data = treeCacheEvent.getData();  //事件结果
                if (data != null){                          //不为null则代表有事件发生
                    switch (treeCacheEvent.getType()){      //判断事件类型
                        case NODE_ADDED:
                            System.out.println("监控到有新增节点！");
                            break;
                        case NODE_UPDATED:
                            System.out.println("监控到节点被更新！");
                            break;
                        case NODE_REMOVED:
                            System.out.println("监控到有节点被移除！");
                            break;
                        default:
                            break;
                    }
                }
            }
        });
        //开始监听
        treeCache.start();

        //测试环境代码不能结束
        Thread.sleep(1000000000);

        client.close();
    }
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201126231232.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201126231147.png)

### 3.12.7 删除节点

```java
 //删除节点
    @Test
    public void deleteZnode() throws Exception {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 1);
        String connectionStr = "192.168.178.100:2181,192.168.178.101:2181,192.168.178.102:2181";
        CuratorFramework client = CuratorFrameworkFactory.newClient(connectionStr, 5000, 8000, retryPolicy);
        client.start();
        /**
         * delete
         * deletingChildrenIfNeeded  有孩子一起删除
         */
        client.delete().deletingChildrenIfNeeded().forPath("/hello");
        client.close();
    }
```

