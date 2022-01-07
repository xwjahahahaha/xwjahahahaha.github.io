---
title: css样式的坑
tags:
  - css
categories:
  - technical
  - css
toc: true
declare: true
date: 2021-01-27 00:32:09
---

# 一、就是不居中

设置了如下两个样式参数，元素很居中了但是不是完全居中，但是父容器高度较小时溢出

```css
display: flex
align-items: center
```

解决方法：

子元素一定要设置高度！！！（即使给子元素高度为0也可）

```css
		.tab_scroll{
			display: flex;
			align-items: center;
			height: 30px;
			flex-wrap: nowrap;
			box-sizing: border-box;
			.tab_scroll_item{
				height: 0px;	//子元素要设置高度！！！
				flex-shrink: 0;	//不让元素收缩挤压
				padding: 0px 10px;
				color: #333333;
				font-size: 20px;
				background-color: #007AFF;
			}
			
		}
```


<!-- more -->

