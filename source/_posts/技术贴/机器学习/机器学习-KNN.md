---
title: 机器学习_KNN
tags:
  - machine_learn
categories:
  - technical
  - machine_learn
toc: true
declare: true
date: 2020-09-25 14:36:51
---

# K-临近算法



<!-- more -->



## 一、理论部分

### 1. 概念

K Nearest Neighbor算法又叫KNN算法

指如果一个样本在特征空间中的**k个最相似(即特征空间中最邻近)的样本中的大多数属于某一个类别**，则该样本也属于这个类别。

例：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200927192340.png)

判断小明所属的区名称，可以找和小明距离最小的人的区域来判断

### 2. 距离公式

#### 2.1 欧氏距离

欧氏距离是最容易直观理解的距离度量方法，我们小学、初中和高中接触到的两个点在空间中的距离一般都是指欧氏距离。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200927193322.png)

举例:

```
X=[[1,1],[2,2],[3,3],[4,4]];
经计算得:
d = 1.4142    2.8284    4.2426    1.4142    2.8284    1.4142
```



#### 2.2  小结

- 1.欧式距离(Euclidean Distance)【知道】：
  - 通过距离平方值进行计算
- 2.曼哈顿距离(Manhattan Distance)【知道】：
  - 通过距离的绝对值进行计算
- 3.切比雪夫距离 (Chebyshev Distance)【知道】：
  - 维度的最大值进行计算
- 4.闵可夫斯基距离(Minkowski Distance)【知道】：
  - 当p=1时，就是曼哈顿距离；
  - 当p=2时，就是欧氏距离；
  - 当p→∞时，就是切比雪夫距离。
- **前四个距离公式小结:前面四个距离公式都是把单位相同看待了,所以计算过程不是很科学**
- 5.标准化欧氏距离 (Standardized EuclideanDistance)【知道】：
  - 在计算过程中添加了标准差,对量刚数据进行处理
- 6.余弦距离(Cosine Distance)【知道】：
  - 通过cos思想完成
- 7.汉明距离(Hamming Distance)【了解】：
  - 一个字符串到另一个字符串需要变换几个字母,进行统计
- 8.杰卡德距离(Jaccard Distance)【了解】：
  - 通过交并集进行统计
- 9.马氏距离(Mahalanobis Distance)【了解】
  - 通过样本分布进行计算



### 3. 算法流程

1）计算已知类别数据集中的点与当前点之间的距离

2）按距离递增次序排序

3）选取与当前点距离最小的k个点

4）统计前k个点所在的类别出现的频率

5）返回前k个点出现频率最高的类别作为当前点的预测分类



## 二、实战部分

### 1. API

sklearn.neighbors.KNeighborsClassifier(n_neighbors=5)

- n_neighbors：int,可选（默认= 5），k_neighbors查询默认使用的邻居数

## Tips

