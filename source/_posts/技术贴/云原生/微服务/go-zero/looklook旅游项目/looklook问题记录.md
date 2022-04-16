---
title: looklook问题记录
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-04-08 21:06:13
---

# 一、提示DOCKER-ISOLATION-STAGE-1的RORWARD链找不到

问题描述：

`unable to insert jump to DOCKER-ISOLATION-STAGE-1 rule in FORWARD chain when using compse to create a container`

解决：

手动创建一个`DOCKER-ISOLATION-STAGE-1`链：`iptables -t filter -N DOCKER-ISOLATION-STAGE-1`

# 二、ElasticSearch服务容器一直启动失败

报错：

![image-20220409103904136](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220409103904136.png)

原因和解决：

https://techoverflow.net/2020/04/18/how-to-fix-elasticsearch-docker-accessdeniedexception-usr-share-elasticsearch-data-nodes/

卷映射文件宿主机文件权限不足的问题：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220409104211103.png" alt="image-20220409104211103" style="zoom:50%;" />

在宿主机上将这个文件权限打开：`sudo chown -R 1000:1000 ./data/elasticsearch/data/`

恢复正常：

![image-20220409104443238](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220409104443238.png)

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220409104507022.png" alt="image-20220409104507022" style="zoom:50%;" />

# 三、网络删除时虚拟网桥未删除导致容器服务与宿主机端口映射不通

错误描述：宿主机端口防火墙都已经打开，但是对应映射到容器中同端口的服务不通，导致所有服务都无法访问

例如prometheus的9090端口，宿主机本机curl访问此端口都被拒绝

经过排查`iptables -L` 链规则，以及网络端口转发`docker port `，发现均无问题：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220409235852062.png" alt="image-20220409235852062" style="zoom:50%;" />

最终发现问题在于某一次`docker-compose down`删除的时候创建网络在宿主机上创建对应的虚拟网桥没有被删除，手动删除这个遗留网桥后解决了问题，之前的原因也得到了解释：端口映射转发到上一个旧的网桥上去了，因为两个网桥同一个CIDR网段

```shell
$ ifconfig xxx down
$ brctl delbr xxx
```

# 四、ES存储提示watermark.high

错误描述：

首先是检查到`kibana`组件一直在重启，并在日志中抱如下错误：

FATAL Error: Unable to complete saved object migrations for the [.kibana_task_manager] index. Please check the health of your Elasticsearch cluster and try again. Error: [cluster_block_exception]: 

![image-20220410163409620](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220410163409620.png)

然后顺利成章的去检查ES：

> ... watermark.high controls the high watermark. It defaults to 90%, meaning ES will attempt to relocate shards to another node if the node disk usage rises above 90%.`

https://stackoverflow.com/questions/30289024/high-disk-watermark-exceeded-even-when-there-is-not-much-data-in-my-index

解决：

提升集群存储水位上限：

```shell
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "2gb",
    "cluster.routing.allocation.disk.watermark.high": "1gb",
    "cluster.routing.allocation.disk.watermark.flood_stage": "500mb",
    "cluster.info.update.interval": "1m"
  }
}
'
```

# 五、后台gin-vue-admin启动报错

错误描述：

![image-20220410171928617](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220410171928617.png)

解决方法：

https://www.jianshu.com/p/dc85066fe1fe

手动运行 `node node_modules/esbuild/install.js` 来解决`esbuild`安装问题

然后再`npm run serve`
