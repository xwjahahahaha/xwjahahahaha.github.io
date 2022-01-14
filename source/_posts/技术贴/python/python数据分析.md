---
title: python数据分析
tags:
  - python
categories:
  - technical
toc: true
declare: true
date: 2020-09-17 07:57:56
---

# 基本概念

## 数据分析是什么？

数据分析是用适当的方法对收集来的大量数据进行分析，**帮助人们作出判断，以便采取适当行动。**

<!-- more -->

## 流程

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200917080101.png)

# matplotlib可视化工具

## 简介

matplotlib: 最流行的Python底层绘图库，主要做数据可视化图表,名字取材于MATLAB，模仿MATLAB构建

## 作用

1.能将数据进行可视化,更直观的呈现

2.使数据更加客观、更具说服力

## 示例

### 1.折线图（连续的数据，感受变化趋势）

1.绘制了折线图(plt.plot)

2.设置了图片的大小和分辨率(plt.figure)

3.实现了图片的保存(plt.savefig)

4.设置了xy轴上的刻度和字符串(xticks)

5.解决了刻度稀疏和密集的问题(xticks)

6.设置了标题,xy轴的lable(title,xlable,ylable)

7.设置了字体(font_manager. fontProperties,matplotlib.rc)

8.在一个图上绘制多个图形(plt多次plot即可)

9.为不同的图形添加图例

> plt.plot()
>

```python
# 假设大家在30岁的时候,根据自己的实际情况,统计出来了你和你同桌各自从11岁到30岁
# 每年交的女(男)朋友的数量如列表a和b,请在一个图中绘制出该数据的折线图,
# 以便比较自己和同桌20年间的差异,同时分析每年交女(男)朋友的数量走势
# a = [1,0,1,1,2,4,3,2,3,4,4,5,6,5,4,3,3,1,1,1]
# b = [1,0,3,1,2,2,3,3,2,1 ,2,1,1,1,1,1,1,1,1,1]
# 要求:
#     y轴表示个数
#     x轴表示岁数,比如11岁,12岁等

from matplotlib import pyplot as plt, font_manager
import matplotlib

# 设置字体
matplotlib.rc("font", family="MicroSoft YaHei", weight="bold")
# 第二种设置字体的方式  fname参数是系统保存的字体路径
my_font = font_manager.FontProperties(fname="C:\Windows\Fonts\simfang.ttf") # 仿宋


# 数据
x = range(11, 31)
y1 = [1,0,1,1,2,4,3,2,3,4,4,5,6,5,4,3,3,1,1,1]
y2 = [1,0,3,1,2,2,3,3,2,1 ,2,1,1,1,1,1,1,1,1,1]

plt.figure(figsize=(20, 8), dpi=80)

plt.plot(x, y1, label="自己", color="orange", linestyle=":")
plt.plot(x, y2, label="同桌", color="cyan", linestyle="--")

# 设置横坐标
_xticks_lable = ["{}岁".format(i) for i in range(11, 31)]
plt.xticks(list(x), _xticks_lable, rotation=45)
plt.yticks(range(1, 9))

# 设置名称
plt.xlabel("年龄（岁）")
plt.ylabel("个数（个）")
plt.title("各个年龄交女(男)朋友的数量走势图")

# 显示不同折线标签  注意，legend添加字体参数使用prop而不是fontproperties
plt.legend(prop=my_font, loc="upper left")

plt.grid(alpha=0.5, linestyle=":")

plt.show()
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200917154051.png)



### 2. 散点图（分布规律）

> ```
> plt.scatter()
> ```

```python
# 假设通过爬虫你获取到了北京2016年3,10月份每天白天的最高气温(分别位于列表a,b),
# 那么此时如何寻找出气温和随时间(天)变化的某种规律?
# a = [11,17,16,11,12,11,12,6,6,7,8,9,12,15,14,17,18,21,16,17,20,14,15,15,15,19,21,22,22,22,23]
# b = [26,26,28,19,21,17,16,19,18,20,20,19,22,23,17,20,21,20,22,15,11,15,5,13,17,10,11,13,12,13,6]

from matplotlib import pyplot as plt
from matplotlib import font_manager as fm

x_1 = range(1, 32)
x_2 = range(51, 82)
y_1 = [11,17,16,11,12,11,12,6,6,7,8,9,12,15,14,17,18,21,16,17,20,14,15,15,15,19,21,22,22,22,23]
y_2 = [26,26,28,19,21,17,16,19,18,20,20,19,22,23,17,20,21,20,22,15,11,15,5,13,17,10,11,13,12,13,6]

my_font = fm.FontProperties(fname="C:\Windows\Fonts\simfang.ttf")

# 设置图像大小
plt.figure(figsize=(10,8), dpi=100)

plt.scatter(x_1, y_1, label="三月")
plt.scatter(x_2, y_2, label="十月")

plt.legend(prop=my_font)

# 调整x轴的刻度
_x = list(x_1) + list(x_2)
_xticks_label = ["三月{}日".format(i) for i in x_1]
_xticks_label += ["十月{}日".format(i-50) for i in x_2]
plt.xticks(_x[::3], _xticks_label[::3], rotation=45, fontproperties=my_font)

plt.show()

```



![](http://xwjpics.gumptlu.work/qiniu_picGo/20200917225736.png)



### 3. 条形图（离散数据）

> plt.bar()水平  plt.barh()垂直

```python
from matplotlib import pyplot as plt
from matplotlib import font_manager

# 假设你获取到了2017年内地电影票房前20的电影(列表a)和电影票房数据(列表b),
# 那么如何更加直观的展示该数据?
#
# a = ["战狼2","速度与激情8","功夫瑜伽","西游伏妖篇",
# "变形金刚5：最后的骑士","摔跤吧！爸爸","加勒比海盗5：死无对证",
# "金刚：骷髅岛","极限特工：终极回归","生化危机6：终章","乘风破浪","神偷奶爸3",
# "智取威虎山","大闹天竺","金刚狼3：殊死一战","蜘蛛侠：英雄归来","悟空传",
# "银河护卫队2","情圣","新木乃伊",]
# b=[56.01,26.94,17.53,16.49,15.45,12.96,11.8,11.61,11.28,11.12,10.49,10.3,8.75,7.55,7.32,6.99,6.88,6.86,6.58,6.23] 单位:亿

# 横着显示条形图

my_font = font_manager.FontProperties(fname="C:\Windows\Fonts\simfang.ttf")

a = ["战狼2","速度与激情8","功夫瑜伽","西游伏妖篇","变形金刚5：最后的骑士","摔跤吧！爸爸","加勒比海盗5：死无对证","金刚：骷髅岛","极限特工：终极回归","生化危机6：终章","乘风破浪","神偷奶爸3","智取威虎山","大闹天竺","金刚狼3：殊死一战","蜘蛛侠：英雄归来","悟空传","银河护卫队2","情圣","新木乃伊",]

b=[56.01,26.94,17.53,16.49,15.45,12.96,11.8,11.61,11.28,11.12,10.49,10.3,8.75,7.55,7.32,6.99,6.88,6.86,6.58,6.23]

plt.figure(figsize=(20,15), dpi=150)

plt.barh(range(len(a)), b, height=0.3, color="orange")  # 横着使用height而不是width

# 刻度
plt.yticks(range(len(a)), a, fontproperties=my_font)

plt.grid(alpha=0.3)

plt.show()
```

![image-20200917230026106](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200917230026106.png)





### 4. 柱状图（连续数据）

一般来说能够使用plt.hist方法的的是那些没有统计过的数据即**原始数据**，**不适合来显示已经统计过的数据**



```python
# 绘制统计直方图
from matplotlib import  pyplot as plt
from matplotlib import font_manager

a=[131,  98, 125, 131, 124, 139, 131, 117, 128, 108, 135, 138, 131, 102, 107, 114, 119, 128, 121, 142, 127, 130, 124, 101, 110, 116, 117, 110, 128, 128, 115,  99, 136, 126, 134,  95, 138, 117, 111,78, 132, 124, 113, 150, 110, 117,  86,  95, 144, 105, 126, 130,126, 130, 126, 116, 123, 106, 112, 138, 123,  86, 101,  99, 136,123, 117, 119, 105, 137, 123, 128, 125, 104, 109, 134, 125, 127,105, 120, 107, 129, 116, 108, 132, 103, 136, 118, 102, 120, 114,105, 115, 132, 145, 119, 121, 112, 139, 125, 138, 109, 132, 134,156, 106, 117, 127, 144, 139, 139, 119, 140,  83, 110, 102,123,107, 143, 115, 136, 118, 139, 123, 112, 118, 125, 109, 119, 133,112, 114, 122, 109, 106, 123, 116, 131, 127, 115, 118, 112, 135,115, 146, 137, 116, 103, 144,  83, 123, 111, 110, 111, 100, 154,136, 100, 118, 119, 133, 134, 106, 129, 126, 110, 111, 109, 141,120, 117, 106, 149, 122, 122, 110, 118, 127, 121, 114, 125, 126,114, 140, 103, 130, 141, 117, 106, 114, 121, 114, 133, 137,  92,121, 112, 146,  97, 137, 105,  98, 117, 112,  81,  97, 139, 113,134, 106, 144, 110, 137, 137, 111, 104, 117, 100, 111, 101, 110,105, 129, 137, 112, 120, 113, 133, 112,  83,  94, 146, 133, 101,131, 116, 111,  84, 137, 115, 122, 106, 144, 109, 123, 116, 111,111, 133, 150]

# 计算组数
bin_width = 3   # 设置组距为3
num_bins = (max(a)-min(a)) // bin_width  # 计算组数  //是整数除法（向下取整）

plt.figure(figsize=(10, 8), dpi=200)

# 第一个参数是数据，第二个参数是分的组数
plt.hist(a, num_bins, density=True)     #density属性是显示频率

plt.xticks(range(min(a), max(a)+bin_width, bin_width))

plt.grid()

plt.show()
```



![](http://xwjpics.gumptlu.work/qiniu_picGo/20200917230240.png)

### 5. 总结

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200917155359.png)

## 使用流程

1.明确问题 

2.选择图形的呈现方式

3.准备数据

4.绘图和图形完善

# Numpy

一个在Python中做科学计算的基础库，重在**数值计算**，也是大部分**PYTHON科学计算库的基础库**，多用于在大型、多维数组上执行数值运算

![](http://xwjpics.gumptlu.work/qiniu_picGo/numpy.png)

# Pandas

![](http://xwjpics.gumptlu.work/qiniu_picGo/pandas.png)

# Tips

## 1. 项目环境

每次创建一个项目都是**根据项目来创建一个其特有的环境**，单纯的为这个项目服务。

## 2. numpy的注意点copy和view

1. a=b 完全不复制，a和b相互影响

2. a = b[:],**视图**的操作，一种切片，会**创建新的对象a**，**但是a的数据完全由b保管**，他们**两个的数据变化是一致**的，注意和python的区别！！

3. a = b.copy(),**复制**，a和b互不影响

## 3. numpy使用布尔选择筛选部分数据

注意直接将不满足的整行去除！ 	

sample：

```python
import numpy as np
from matplotlib import pyplot as plt
from matplotlib import font_manager

# 希望了解英国的youtube中视频的评论数和喜欢数的关系，应该如何绘制改图

my_font = font_manager.FontProperties(fname="C:\Windows\Fonts\simfang.ttf")
# 提取数据
file_path = "../data/GB_video_data_numbers.csv"

t = np.loadtxt(fname=file_path, delimiter=",", dtype=int)
print(t)

# 去除极端数据，在总表上操作而不是单列！不然会导致数量不一致
t = t[t[:,1]<=60000] # 如果不满足条件，去除的是整行

t_comment_col = t[:,-1] # 评论数列
t_like_col = t[:,1] #喜欢数列

# 去除极端数据,把极端数据变成平均数
# t_comment_col[t_comment_col>500000] = np.mean(t_comment_col)
# t_like_col[t_like_col>500000] = np.mean(t_like_col)




# 画散点图
plt.scatter(t_comment_col, t_like_col)
plt.xlabel("评论数", fontproperties=my_font)
plt.ylabel("喜欢数", fontproperties=my_font)

# 显示
plt.show()
```

## 4. pandas和numpy的切片方法中的':'

1. numpy中切片使用的：连续是不到达后面的数字的

   sample:

   ```python
   t1 = np.arange(12).reshape(3, 4)
   print(t1)
   print("*"*100)
   print(t1[0:2,:]) # 这是第一行到第二行的数据
   print("*"*100)
   df1 = pd.DataFrame(np.arange(12).reshape(3, 4))
   print(df1)
   print("*"*100)
   print(df1.loc[0:2,:]) # 取第一行、第二行、第三行的数据
   ```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200919200154.png)