---
title: MetaMask或外部账户导入到Geth私链的方法（相同地址）小结
tags:
  - geth
categories:
  - technical
toc: true
declare: true
date: 2020-09-03 20:45:41
---

# 获取MetaMask上账户的私钥

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjE1NDAzLnBuZw?x-oss-process=image/format,png)

点击账户详情，导出私钥：

<!-- more -->

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjE1NDQ4LnBuZw?x-oss-process=image/format,png)

拿到私钥以后，创建一个文本文件例如fk.txt放进去

然后输入`geth account import 你的私钥文件路径`

会提示你输入密码，这个密码是在geth控制台使用的密码

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjE1NzEzLnBuZw?x-oss-process=image/format,png)

发现生成的账户就是在MetaMask上的账户。

此时查看当前生成密钥文件位置：`geth account list`

找到对应账户后面的存储位置，把文件放到项目中的keystore文件夹中即可。

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjIwNDQ3LnBuZw?x-oss-process=image/format,png)

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjIwNjA5LnBuZw?x-oss-process=image/format,png)
在geth中输入`eth.accounts`查看

导入完成。

![](https://imgconvert.csdnimg.cn/aHR0cDovL3h3anBpY3MuZ3VtcHRsdS53b3JrL3Fpbml1X3BpY0dvLzIwMjAwOTAzMjIwNjU4LnBuZw?x-oss-process=image/format,png)

**这种方法同样适用于任何已知私钥的外部账户导入geth**