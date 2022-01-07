---
title: 大数据-3-Hadoop的环境搭建
tags:
  - hadoop
categories:
  - technical
  - hadoop
toc: true
declare: true
date: 2020-11-28 08:12:09
---

# 四、Hadoop

## 4.1 Hadoop介绍

### 4.1.1 起源

​	Hadoop最早起源于Nutch。Nutch的设计目标是构建一个大型的<u>全网搜索引擎</u>, 包括网页抓取、索引、查询等功能，但随着抓取网页数量的增加，遇到了严重的可扩展性问题。

​	Google提出了两种解决方案：

* 分布式文件系统（GFS），可用于处理海量网页的**存储**
* 分布式计算框架MapReduce，可用于处理海量网页的**索引计算**问题

=> Nutch的开发人员完成了相应的**HDFS**和**MapReduce**，并从Nutch中剥离成为独立的项目**Hadoop**

<!-- more -->

**狭义上来说，Hadoop就是指代Hadoop这个软件**

简单介绍：

* HDFS分布式文件系统

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128123140.png)

* MapReduce分布式计算系统

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128123347.png)

* Yarn分布式样集群资源管理系统（以后再学习）

**广义上Hadoop是指Hadoop生态圈**：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128124344.png)

### 4.1.2 历史版本与发行公司

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128124453.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128124724.png)

我们学习的是apache的版本

### 4.1.3 hadoop的架构模型

* 1.x的版本架构模型

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128124948.png)

  * 分布式文件存储系统

    主从架构：

    NameNode：集群中的<font color='orange'>主节点</font>，管理元数据（文件的大小、位置、权限）主要管理集群中的各种数据

    SecondaryNameNode：主要用于hadoop当中元数据信息的**辅助**管理

    DataNode: 集群当中的<font color='green'>从节点</font>，主要用于集群中的各种数据

    元数据：<font color='green'>文件的大小、位置、权限；例如记录数据被切成几片，分别的存储位置等等。</font>

  * 分布式计算系统

    JobTracker:接收用户的计算请求任务，并**分配**任务给从节点

    TaskTracker: 负责**执行**主节点JobTracker分配的任务

* 2.x架构

  * NameNode与ResourceManager单节点架构模型

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128125839.png)

    * yarn数据计算核心模块

      ResourceManager: 接收用户的计算请求任务，并负责集群的资源分配

      NodeManager：负责执行主节APPmaster分配的任务

  * NameNode单节点与ResourceManager**高可用**架构模型

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128130531.png)

  * NameNode高可用与ResourceManager单节点架构模型

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128130737.png)

    Journal Node是为了当一个NameNode挂掉之后，元数据通过这些JournalNode实现数据的恢复

  * NameNode与ResourceManage高可用架构模型

    ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128130818.png)

## 4.2 Hadoop的安装

### 4.2.1 前提工作（自己编译安装包）

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128131100.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128133813.png)

<font color='red'>注意Hadoop-2.7.5这个版本的编译，只能使用jdk1.7，不能使用jdk1.8否则会报错，所以要重新安装jdk-1.7</font>

#### 4.2.1.1 CentOS使用多个版本的java

* 下载jdk1.7包：[Java Archive Downloads - Java SE 7 (oracle.com)](https://www.oracle.com/java/technologies/javase/javase7-archive-downloads.html#jdk-7u80-oth-JPR)

* 移动到/export/softwares,解压到/export/servers中

* 配置环境变量

* ```shell
  export JAVA18_HOME=/export/servers/jdk1.8.0_271
  export JAVA17_HOME=/export/servers/jdk1.7.0_80
  export PATH=:$JAVA18_HOME/bin:$PATH
  export PATH=:$JAVA17_HOME/bin:$PATH
  # 退出保存
  source /etc/profile
  ```

* 放置到java bin中

  ```shell
  alternatives --install /usr/bin/java java /export/servers/jdk1.7.0_80/bin/java 4
  # 如果之前的1.8没放置可以也放置
  alternatives --install /usr/bin/java java /export/servers/jdk1.8.0_271/bin/java 4
  ```

* 切换版本以及设置默认版本

  ```shell
  alternatives --config java
  # 选择数字
  ```

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128141024.png)

#### 4.2.1.2 安装编译需要的依赖包

maven3.0.5: [Index of /dist/maven/maven-3/3.0.5/binaries (apache.org)](https://archive.apache.org/dist/maven/maven-3/3.0.5/binaries/)

tomcat6.0.53: [Index of /dist/tomcat/tomcat-6/v6.0.53/bin (apache.org)](https://archive.apache.org/dist/tomcat/tomcat-6/v6.0.53/bin/)

findbugs1.3.9:[FindBugs - Browse /findbugs/1.3.9 at SourceForge.net](https://sourceforge.net/projects/findbugs/files/findbugs/1.3.9/)

hadoop-2.7.5-src.tar.gz(源文件):[Index of /dist/hadoop/core/hadoop-2.7.5 (apache.org)](http://archive.apache.org/dist/hadoop/core/hadoop-2.7.5/)

mvnrepository：暂未找到

protobuf-2.5.0:[Release Protocol Buffers v2.5.0 · protocolbuffers/protobuf (github.com)](https://github.com/protocolbuffers/protobuf/releases/tag/v2.5.0)

snappy-1.1.3: [Release Snappy 1.1.3 · google/snappy (github.com)](https://github.com/google/snappy/releases/tag/1.1.3)

##### 安装Maven

```shell
tar -xvf apache-maven-3.0.5-bin.tar.gz -C ../servers/

# 在/etc/profile
export MAVEN_HOME=/export/servers/apache-maven-3.0.5
export MAVEN_OPTS="-Xms4096m -Xmx4096m"
export PATH=:$MAVEN_HOME/bin:$PATH

source /etc/profile
```

解压Maven的仓库

```shell
# 同理解压到servers中
tar -zxvf ../softwares/mvnrepository.tar.gz -C ./
# 修改maven的配置文件
cd /export/servers/apache-maven-3.0.5/conf
vim settings.xml
# 修改为仓库的路径 见下图

# 修改完了之后再添加一个阿里镜像
<mirror>
<id>alimaven</id>
<name>aliyun maven</name>
<url>http://maven.aliyun.com/nexus/content/groups/public</url>
<mirrorOf>central</mirrorOf>
</mirror>
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128145143.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128145622.png)

##### 安装findbugs

```shell
tar -zxvf /export/softwares/findbugs-1.3.9.tar.gz  -C /export/servers/

export FINDBUGS_HOME=/export/servers/findbugs-1.3.9
export PATH=:$FINDBUGS_HOME/bin:$PATH
source /etc/profile
```

##### 在线安装一些依赖包

```shell
yum install autoconf automake libtool cmake
yum install ncurses-devel
yum install openssl-devel
yum install lzo-devel zlib-devel gcc gcc-c++

yum install -y bzip2-devel
```

##### 安装protobuf

解压protobuf并进行编译

```shell
tar -zxvf protobuf-2.5.0.tar.gz -C ../servers/
cd ../servers/protobuf-2.5.0/
./configure				# 配置
make && make install  	# 编译并安装  要一些时间等会....
```

##### 安装snappy  

<font color='red'>（编译就是为了支持snappy压缩，snappy是一种压缩算法）</font>

```shell
tar -zxvf snappy-1.1.1.tar.gz  -C ../servers/
cd ../servers/snappy-1.1.1/
./configure
make && make install
```

#### 4.2.1.3 编译Hadoop源码

```shell
tar -zxvf hadoop-2.7.5-src.tar.gz -C  ../servers/
cd ../servers/hadoop-2.7.5-src/
# 编译支持snappy压缩
mvn package -DskipTests -Pdist,native -Dtar -Drequire.snappy -e -X
```

漫长的等待。。。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128164643.png)

出现SUCCESS就代表ok了

找到编译好的文件:

```shell
cd /export/servers/hadoop-2.7.5-src/hadoop-dist/target
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201128221537.png)

这个安装包已经加入了常用的压缩算法，至此hadoop的编译就结束啦！

### 4.2.2 安装Hadoop

集群规划

| 服务器IP          | 192.168.178.100 | 192.168.178.101 | 192.168.178.102 |
| ----------------- | --------------- | --------------- | --------------- |
| 主机名            | node01          | node02          | node03          |
| NameNode          | 是              | 否              | 否              |
| SecondaryNameNode | 是              | 否              | 否              |
| dataNode          | 是              | 是              | 是              |
| ResourceManager   | 是              | 否              | 否              |
| NodeManager       | 是              | 是              | 是              |

#### 第一步：上传hadoop安装包

<font color='red'>注意这里上传的安装包是上面编译过的安装包，支持一些底层的压缩算法</font>

对比两个安装包的区别：

使用命令：

```shell
# 进入到Hadoop文件夹下
bin/hadoop checknative
```

直接解压官方的hadoop压缩包文件：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209164613.png)

检查结果显示一些压缩算法是没有实现的

而自己编译的压缩包中压缩算法是实现的：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209165330.png)

<font color='green'>主要的还是要支持snappy压缩算法</font>

#### 第二步：修改配置文件

```shell
# 在Hadoop文件夹中进入配置文件文件夹
cd etc/hadoop
```

使用Notepad++远程登录Linux

在其中使用第三方插件NppFTP，具体看：https://www.cnblogs.com/JBLi/p/10691540.html

不过Notepad支持港独，啧啧，垃圾软件，建议使用mobaxterm

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209172436.png)

-----

##### 修改core-site.xml

修改如下：

```xml
<configuration>

  <property>
    <!-- 指定Hadoop集群的文件系统类型：分布式文件系统 -->
    <name>fs.default.name</name>
    <value>hdfs://node01:8020</value>
  </property>
  
  
  <property>
    <!-- 指定临时存储空间 -->
    <name>hadoop.tmp.dir</name>
    <value>/export/servers/hadoop-2.7.5/hadoopDatas/tempDatas</value>
  </property>
  
  <!-- 缓冲区大小，实际工作中根据服务器的性能动态调整 -->
  <property>
    <name>io.file.buffer.size</name>
    <value>4096</value>
  </property>

   <!-- 开启hdfs的垃圾桶机制，删除掉的数据可以从垃圾桶中回收，单位分钟 -->
  <property>
    <name>fs.trash.interval</name>
    <!-- 7天 -->
    <value>10080</value>
  </property>


</configuration>
```

##### 修改hdfs-site.xml

```xml
<configuration>
        <!-- 指定secondaryNameNode的地址 -->
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>node01:50090</value>
        </property>

        <!-- 指定namenode的访问地址和端口 -->
        <property>
                <name>dfs.namenode.http-address</name>
                <value>node01:50070</value>
        </property>
        <!-- 指定namenode元数据的存放位置 -->
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:///export/servers/hadoop-2.7.5/hadoopDatas/namenodeDatas,file:///export/servers/hadoop-2.7.5/hadoopDatas/namenodeDatas2</value>
        </property>
        <!--  定义dataNode数据存储的节点位置，实际工作中，一般先确定磁盘的挂载目录，然后多个目录用，进行分割  -->
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:///export/servers/hadoop-2.7.5/hadoopDatas/datanodeDatas,file:///export/servers/hadoop-2.7.5/hadoopDatas/datanodeDatas2</value>
        </property>

        <!-- 指定namenode日志文件的存放目录 -->
        <property>
                <name>dfs.namenode.edits.dir</name>
                <value>file:///export/servers/hadoop-2.7.5/hadoopDatas/nn/edits</value>
        </property>


        <property>
                <name>dfs.namenode.checkpoint.dir</name>
                <value>file:///export/servers/hadoop-2.7.5/hadoopDatas/snn/name</value>
        </property>
        <property>
                <name>dfs.namenode.checkpoint.edits.dir</name>
                <value>file:///export/servers/hadoop-2.7.5/hadoopDatas/dfs/snn/edits</value>
        </property>
        <!-- 文件切片的副本个数,也就是每个切片冗余存储的个数-->
        <property>
                <name>dfs.replication</name>
                <value>3</value>
        </property>

        <!-- 设置HDFS的文件权限-->
        <property>
                <name>dfs.permissions</name>
                <value>false</value>
        </property>

        <!-- 设置一个文件切片的大小：128M-->
        <property>
                <name>dfs.blocksize</name>
                <value>134217728</value>
        </property>
</configuration>
```

##### 修改hadoop-env.sh

只要修改JAVA_HOME的路径即可

```shell
# The java implementation to use.
export JAVA_HOME=/export/servers/jdk1.8.0_271
```

注意：配置自己安装的jdk版本！

##### 修改mapred-site.xml

**这个没有源文件，而是有一个模板文件（mapred-site.xml.template），在模板文件上修改再重命名mapred-site.xml即可**

```xml
<configuration>
        <!-- 指定分布式计算使用的框架是yarn -->
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>

        <!-- 开启MapReduce小任务模式 -->
        <property>
                <name>mapreduce.job.ubertask.enable</name>
                <value>true</value>
        </property>

        <!-- 设置历史任务的主机和端口 -->
        <property>
                <name>mapreduce.jobhistory.address</name>
                <value>node01:10020</value>
        </property>

        <!-- 设置网页访问历史任务的主机和端口 -->
        <property>
                <name>mapreduce.jobhistory.webapp.address</name>
                <value>node01:19888</value>
        </property>
</configuration>
```

##### 修改yarn-site.xml

```xml
<configuration>
        <!-- 配置yarn主节点的位置 -->
        <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>node01</value>
        </property>

        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>

        <!-- 开启日志聚合功能 -->
        <property>
                <name>yarn.log-aggregation-enable</name>
                <value>true</value>
        </property>
        <!-- 设置聚合日志在hdfs上的保存时间 -->
        <property>
                <name>yarn.log-aggregation.retain-seconds</name>
                <value>604800</value>
        </property>
        <!-- 设置yarn集群的内存分配方案 -->
        <property>
                <name>yarn.nodemanager.resource.memory-mb</name>
                <value>20480</value>
        </property>

        <property>
                 <name>yarn.scheduler.minimum-allocation-mb</name>
                <value>2048</value>
        </property>
        <property>
                <name>yarn.nodemanager.vmem-pmem-ratio</name>
                <value>2.1</value>
        </property>

</configuration>
```

##### 修改mapred-env.sh

同样的修改JAVA_HOME

```shell
export JAVA_HOME=/export/servers/jdk1.8.0_271
```

##### 修改slaves

把localhost改为如下：

这一步主要是配置slave

```shell
node01
node02
node03
```

#### 第三步：修改Hadoop的环境变量

第一台机执行以下命令，创建文件夹，以此来适配之前的配置文件路径

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209210614.png)

分发整个hadoop到node02、node03

```shell
scp -r hadoop-2.7.5 root@node02:/export/servers/
scp -r hadoop-2.7.5 root@node03:/export/servers/
```

**配置Hadoop的环境变量**

**三台主机都需要配置**

```shell
vim /etc/profile

export HADOOP_HOME=/export/servers/hadoop-2.7.5
export PATH=:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH


source /etc/profile
```

#### 第四步：启动集群

<font color='red'>注意在启动Hadoop之前，要先启动Zookeeper</font>

主要启动两个模块：HDFS和YARN

```shell
cd /ecport/servers/hadoop-2.7.5
bin/hdfs namenode -format  	# 注意：这一步是对namenode的格式化，创建一些文件与目录，只需要创建一次即可，如果再次使用则会导致数据丢失
sbin/start-dfs.sh			# 启动hdfs
sbin/stop-dfs.sh			# 关闭hdfs
sbin/start-yarn.sh			# 启动yarn
sbin/stop-yarn.sh			# 关闭yarn
sbin/mr-jobhistory-daemon.sh start historyserver
```

如果文件中的中文编码格式没有设置好，格式化的时候可能会错误，修改文件编码格式或者删除文字即可。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209212219.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209212554.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209212718.png)

如果NameNode起不起来，那么可以参考：

https://www.cnblogs.com/ya-qiang/p/9494986.html

或者检查配置文件，可以查看对应的日志发现错误（那台机有问题就去那台机去看log文件）

最终的启动效果：

node01：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209220551.png)

node02：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209220205.png)

node03：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201209220219.png)

**web浏览器访问端口，查看界面**

http://node01:50070/explorer.html#/	查看hdfs集群

http://node01:8080/cluster  查看yarn集群

http://node01:19888/jobhistory



windows配置host映射：https://blog.csdn.net/uhana/article/details/60867861

<font color='green'>注意：上方网址的node01，在宿主机上访问需要先配置host文件，或者替换为对应的ip访问</font>