---
title: uni-app的学习-1
tags:
  - uni-app
categories:
  - technical
  - uni-app
toc: true
declare: true
date: 2020-12-02 22:17:14
---

# 一、概念认识

官网地址：[什么是 uni-app - uni-app官网 (dcloud.io)](https://uniapp.dcloud.io/README)

`uni-app` 是一个使用 [Vue.js](https://vuejs.org/) 开发所有前端应用的框架，<font color='red'>开发者编写一套代码，可发布到iOS、Android、Web（响应式）、以及各种小程序（微信/支付宝/百度/头条/QQ/钉钉/淘宝）、快应用等多个平台</font>。

<!-- more -->

## 1.1 uni-app优势所在

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121175528.png)

## 2.2 uni-app和Vue的关系

* 使用Vue.js开发
* 发布到H5时，支持所有的Vue语法
* 发布到App和小程序时，支持部分Vue语法

## 3.3 uini-app和微信小程序的关系

* 组件标签靠近小程序规范
* 接口能力（js api）靠近小程序规范
* 完整的小程序周期

## 4.4 写法上的对比

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121180208.png)

# 二、重点总结

## 2.1 小程序重点

### 2.1.1 小程序的生命周期

* 小程序**应用**的生命周期

```js
App({

  /**
   * 当小程序初始化完成时，会触发 onLaunch（全局只触发一次）
   */
  onLaunch: function () {
    
  },

  /**
   * 当小程序启动，或从后台进入前台显示，会触发 onShow （应用进入前台）
   */
  onShow: function (options) {
    
  },

  /**
  * 当小程序从前台进入后台，会触发 onHide	（应用进入后台）
   */
  onHide: function () {
    
  },

  /**
   * 当小程序发生脚本错误，或者 api 调用失败时，会触发 onError 并带上错误信息
   */
  onError: function (msg) {
    
  }
})
```

**重要概念：**

1. 后台：当用户点击左上角关闭（或者右上角退出），或者按了home键离开微信，小程序并没有直接销毁，而是进入了后台。
2. 前台：当再次进入微信或者再次打开小程序，又会从后台进入前台。

* 小程序**单个页面**的生命周期 

```js
/**
   * 生命周期函数--监听页面加载
   */
  onLoad: function (options) {

  },

  /**
   * 生命周期函数--监听页面初次渲染完成
   */
  onReady: function () {

  },

  /**
   * 生命周期函数--监听页面显示(页面打开的时候触发)
   */
  onShow: function () {

  },

  /**
   * 生命周期函数--监听页面隐藏（页面隐藏的时候触发）(打开其他页面时当前页面就认为被隐藏)
   */
  onHide: function () {

  },

  /**
   * 生命周期函数--监听页面卸载（打开A页面进入B页面，当返回A页面的时候就是B页面的卸载）
   */
  onUnload: function () {

  },
      /**
   * 页面相关事件处理函数--监听用户下拉动作
   */
  onPullDownRefresh: function () {

  },

  /**
   * 页面上拉触底事件的处理函数
   */
  onReachBottom: function () {

  },

  /**
   * 用户点击右上角分享
   */
  onShareAppMessage: function () {

  }
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121200506.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121201057.png)

### 2.1.2 列表渲染

```html
<!--
  item 代表元素， index 代表下标
-->
<view wx:for="{{nums}}">{{item}} - {{index}}</view>
<!-- 也可以自定义命名 -->
<view wx:for="{{nums}}" wx:for-item="t" wx:for-index="i">{{t}} - {{i}}</view>
```

## 2.2 uni-app核心知识点

### 2.2.1 Vue规范

* Vue单文件组件（SFC）规范

uni-app的所有页面都需要遵循SFC规范

使用类Html语法去描述一个Vue组件，没一个组件包含三种类型的语言块：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121203329.png)

* 组件标签类似小程序规范

具体可见4.4写法上的对比

* 接口能力类似微信小程序规范

例如直接使用uni的api

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121203622.png)

* 数据绑定和处理同Vue.js规范
* 为兼容多段运行，建议使用flex布局

### 2.2.2 uni-app特点

* 条件编译

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121203913.png)

* app端的Nvue的开发

weex为程序提供了原生的渲染能力，Nvue在weex的基础上提供了大量uniapp的组件与API

* HTML5+

协助直接调取原生的插件来实现某些功能，例如IOS或安卓。总的来说提升原生能力。

<font color='green'>注意：Nvue和Html5+**只能在App上使用**，在html和小程序上是无法使用的</font>

### 2.2.3 uni-app知识点

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121204629.png)

## 2.3 环境搭建

### 2.3.1 使用HbuilderX软件创建项目编写代码

推荐使用此方式！

### 2.3.2 使用vue-cli方式创建与运行项目

#### 1.安装vue-cli   

`cnpm install vue`  `cnpm install -g @vue/cli` 

详见https://www.jianshu.com/p/e5d1897e6439

#### 2.测试安装结果： 

`vue -V`  注意：V是大写

#### 3.安装uniapp工程：

`vue create -p dcloudio/uni-preset-vue 你的项目名`

注意create命令必须是vue 3.x版本之上，否则会报错:

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121222716.png)

#### 4.选择默认模式

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121223343.png)

#### 5.项目结构

创建的项目结构：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210121223550.png)

#### 6. 运行

`npm run serve`

## 2.4 uni-app开发基础

### 2.4.1 基础

#### **模板语法**

* 数据语法

```js
export default {
		//初始化数据的方法
		data() {
			return {
				title: 'Hello'
			}
		},
}
```

<font color='red'>**为什么不使用对象的方式去出初始化数据data呢？**</font>

即：

```js
data:{
    
},
```

**因为这种方法会保留上一次的值，不会每一次都初始化为原来的值，写成方法去Return这样使得每次页面的值都会被刷新，不会被之前的值影响**

* 事件语法

  事件语法与Vue相同，与小程序不同，注意区分

#### **数据绑定、条件判断、列表渲染**

语法同Vue

#### **基础组件使用**

见uniapp官方文档：https://uniapp.dcloud.io/component/README

#### <font color='red'>**自定义组件使用(重点)**</font>

**<font color='cornflowerblue'>第一步：注册自己的组件</font>**

创建components文件夹在其下创建组件	

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210122112809.png)

**<font color='cornflowerblue'>第二步：编写组件</font>**

```vue
<template>
	<view class="view-box">
		我自己创建的组件，哈哈哈
	</view>
</template>

<script>
	export default {
		data() {
			return {
				
			};
		}
	}
</script>

<style>
	.view-box{
		width: 100rpx;
		height: 500rpx;
		border: 1rpx #007AFF solid;
	}
</style>

```

**<font color='cornflowerblue'>第三步：导入组件使用</font>**

在**script标签首行位置**导入

```vue
<template>
	<view>
		<myView></myView>	
	</view>
</template>
<script>
	import myView from '@/components/myView/myView.vue'
	export default {
		//导入使用
		components:{
			myView
		},
		.....
	}
</script>
```

@表示项目的根目录，前面的是导入的别名

##### **<font color='cornflowerblue'>拓展1：自定义组件传入参数（页面向组件传参）</font>**

例1：

props与样式属性相结合：

在**自定义组件**vue文件中编辑props：

```vue
<template>
	<view class="view-box" :style="{color:color}">
		我自己创建的组件，哈哈哈
	</view>
</template>

<script>
	export default {
		props:{
			color:{
				default: 'red',		//参数的默认值
				type:String,		//参数的类型
				
			},
        ....
</script>
```

：style与v-bind类似，中括号中的两个都是属性名

**其他页面**使用自定义组件并传入参数：

`<myView color="blue"></myView>	`

```vue
<template>
	<view>
		<myView color="blue"></myView>	
	</view>
</template>
<script>
	import myView from '@/components/myView/myView.vue'
	export default {
		//导入使用
		components:{
			myView
		},
		.....
	}
</script>
```

目前的HbuilderX支持省略Import和components直接同名调用的方法

例二：

<font color='red'>**props可以直接接收父页面的参数, 并且可以用于类似data使用。**</font>

组件：

```vue
<template>
	<view class="bar">
		<scroll-view class="tab_scroll" scroll-x="true" enable-flex="true">
			<view v-for="item in categoryList" class="tab_scroll_item">{{item.name}}</view>
		</scroll-view>
	</view>
</template>

<script>
	export default {
        //props并没有作为样式属性
		props: {
			categoryList:{
				type: Array,
				default: [{
					name: '分类1'
				},{
					name: '分类2'
				},{
					name: '分类3'
				}]
			}
		},
		data() {
			return {

			};
		}
	}
</script>
```

父页面：

**以标签的方式载入，categoryList是传参对象**

```vue
<template>
	<view>
		<!-- 分类选项卡 -->
		<categoryBar :categoryList="categoryList"></categoryBar>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				categoryList: [{
					name: "类别1"
				},{
					name: "类别2"
				},{
					name: "类别3"
				},{
					name: "类别4"
				}]
			}
		},
        ...
</script>
```





##### **<font color='cornflowerblue'>拓展2：自定义组件事件发送数据（组件向页面传参）</font>**

**自定义组件**编写

```vue
<template>
	<view class="view-box" :style="{color:color}" @click="myClick">
		我自己创建的组件，哈哈哈
	</view>
</template>

<script>
	export default {
		....
		methods:{
			myClick(){
				console.log("myView被点击")
				//发送事件，将当前页面的字体颜色发送
				this.$emit('change', this.color)
			}
		}
		
	}
</script>
```

`this.$emit()`第一个参数是其他页面调用的事件名，第二个参数是发送的参数数据

**其他页面**的调用与捕获数据：

```vue
<template>
	<view>
        <myView color="green" @change="change"></myView>
	</view>
</template>

<script>
	import myView from '@/components/myView/myView.vue'
	export default {
		//导入使用
		components:{
			myView
		},
		...
		methods: {
			...
			change(e){
				console.log("接收到了自定义组件的点击数据:", e)
			}
		}
	}
</script>
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210122115944.png)

**<font color='cornflowerblue'>拓展3：使用插槽实现调用者设置自定义组件内容</font>**

**自定义组件**使用slot

```vue
<template>
	<view class="view-box" :style="{color:color}" @click="myClick">
		<!-- 使用插槽实现调用者自定义内容 -->
		<slot></slot>
	</view>
</template>
```

其他页面调用时：

`<myView color="green" @change="change">我想写什么你就显示什么哈啊哈哈</myView>`

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210122120737.png)

#### **基础api用法**

详细见官网：https://uniapp.dcloud.io/api/README

例：调用uni.getSystemInfo

```vue
onLoad() {
			uni.getSystemInfo({
				success(res) {
					console.log("成功返回", res)
				},
				fail(err) {
					console.log("失败返回", err)
				},
				complete(res) {
					console.log("不管成功或者失败", res)
				}
			})
		},
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210122121402.png)

#### **条件编译**

只编译对应平台的代码显示对应的平台，不会编译冗余的代码，在组件的注释中实现：

```vue
<template>
	<view class="content" :class="className" v-on:click="open">
		{{title}}
		<!-- #ifdef H5 || APP-PLUS -->
		<view class="content">
			我只能在App上看见
		</view>
		<!-- #endif -->
    </view>
</template>
```

`<!-- #ifndef H5 -->` 加上一个n表示只有H5不显示，其他都显示

条件编译在模板、script、style都可以使用，都是包含在对应的注释中

官网文档：https://uniapp.dcloud.io/platform?id=%e6%9d%a1%e4%bb%b6%e7%bc%96%e8%af%91

#### **页面布局**

外部样式导入语法

```css
<style>
	@import url("./index.css");
	/* 或者 */
	@import "./index.css";
</style>
```

### 2.4.2 uni-app的生命周期

官网详细的文档：https://uniapp.dcloud.io/collocation/frame/lifecycle?id=%e5%ba%94%e7%94%a8%e7%94%9f%e5%91%bd%e5%91%a8%e6%9c%9f

#### 1 应用生命周期

应用的生命周期在**App.Vue**中：

```vue
<script>
	export default {
		// 应用初始化完成出发出发一次,全局只出发一次
		onLaunch: function() {
            //登录或一些全局变量的获取
			console.log('App Launch')
		},
		//应用从后台切换到前台触发一次
		onShow: function() {
			console.log('App Show')
		},
		//应用从前台切换到后台触发一次
		onHide: function() {
			console.log('App Hide')
		}
	}
</script>
```

#### 2 页面生命周期

在对应的页面的.vue文件中：

```js
export default {
		//监听页面加载，页面标签等加载之前触发
		onLoad() {
		},
		//监听页面初次渲染完成
		onReady() {
			//如果页面渲染的很快，那么有可能会在页面进入动画之前触发
		},
    	//监听页面显示，每次页面的显示都会触发onShow
		onShow() {
			
		},
		//监听页面的隐藏，每次页面被隐藏都会触发onHide
		onHide() {
			
		},
		//监听页面卸载
		onUnload() {
			
		},
    	//监听TabBar
		onTabItemTap() {
			
		},
		...

}
```

<font color='green'>**应用的生命周期与页面的生命周期总体上与小程序是相同的**</font>

#### 3 组件生命周期

在组件的vue文件中:

```js
		//在实例初始化之后，数据观测 (data observer) 和 event/watcher 事件配置之前被调用。
		beforeCreate() {
			
		},
		//在实例创建完成后被立即调用。
		created() {
			
		},
		//挂载上实例之后调用
		mounted() {
			
		},
		//Vue实例销毁后调用
		destroyed() {
			
		}
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210123110719.png)

**当组件被使用v-if等隐藏之后，组件就会触发destroyed**

### 2.4.3 目录结构

一般项目的目录结构如下：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210123120056.png)

### 2.4.4 uniCloud

#### 1. 概述

可以作为前端实现页面逻辑的一个临时后端数据接口

集成了阿里云以及腾讯云，基于serverless模式和js编程的云开发平台

#### 2. uniCloud的价值

* 使用js开发前后台整体业务
* 专注于业务
* 敏捷性项目不需要前后端分离
* 前端不需要会关系SQL，数据库采用NOSQL

#### 3. 开发流程

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210123152225.png)

#### 4. 构成

* 云函数

  只需要编写云函数即可对数据库进行CRUD

  ```js
  'use strict';
  exports.main = async (event, context) => {
  	//event为客户端上传的参数
  	console.log('event : ', event)
  	//返回数据给客户端
  	return event
  };
  ```

* 云数据库

  云数据库采用NoSql数据库，前端工作者不需要考虑SQL语句的书写

* 云存储与CDN

  云存储可以提供存储一些图片以及文件

  CDN全称Content Delivery Network，即内容分发网络。其基本思路是尽可能避开互联网上有可能影响数据传输速度和稳定性的瓶颈和环节，使内容传输的更快、更稳定。

#### 5. 使用

在HbuilderX中使用:

1. 首先登陆HbuilderX，没有实名认证的话需要先实名认证

2. 创建项目时选择使用uniCloud，没选择的话右击项目=>创建uniCloud云开发即可

3. 准备工作

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210126162027.png)

4. 在cloud文件夹上右键，创建云空间

   ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210126162428.png)

5. 创建完毕后，在HbuilderX中关联创建的云空间

6. 右击cloud文件夹创建云函数进行部署使用

因为项目使用前后端分离，所以此处不再深究，详细的云函数CRUD编写等见官方文档：

https://uniapp.dcloud.io/uniCloud/README

### 2.4.5 uniapp网络请求

##### 官方原生的request：

https://uniapp.dcloud.io/api/request/request

```js
uni.request({
    url: 'https://www.example.com/request', //仅为示例，并非真实接口地址。
    data: {
        text: 'uni.request'
    },
    header: {
        'custom-header': 'hello' //自定义请求头信息
    },
    success: (res) => {
        console.log(res.data);
        this.text = 'request success';
    }
});
```

##### uni-ajax

具体操作步骤见使用文档：https://uniajax.ponjs.com/quickstart.html

插件市场：https://ext.dcloud.net.cn/plugin?id=2351

## 2.5 实战项目

### 2.5.1 ginblog博客项目重点记录

#### 1. 使用scss样式

```css
<style lang="scss">
	...
</style>
```

添加lang字段然后编译提示需要安装scss插件，进行安装

基本用法：

https://www.jianshu.com/p/3259976b414b

#### 2. 使用高德地图获取定位

* 首先去高德地图网站注册以及实名认证：

  https://lbs.amap.com/api/wx/summary/

* 进入控制台创建自己的应用获取Key

  ![](http://xwjpics.gumptlu.work/qiniu_picGo/20210125110542.png) 

* 下载sdk文件

  https://lbs.amap.com/api/wx/download

* 将其中的`amap-wx.130.js`文件复制到uniapp的common文件夹下

* 引入SDK以及使用

  ```js
  <script>
  	import amap from "./amap-wx.130.js"
  	export default {
  		data() {
  			return {
  				amapPlugin: null,							//高德地图插件实例
                  amapKey: '申请的Key',							//高德地图的Key 
  				
  			};
  		},
  		created() {
  			//实例化高得地图插件
  			this.amapPlugin = new amap.AMapWX({key:this.amapKey});
  			this.amapPlugin.getRegeo({
  			  success: function(data){
  				//成功回调
  				console.log(data)
  			  },
  			  fail: function(info){
  				//失败回调
  				console.log(info)
  			  }
  			})
  		}
  	}
  </script>
  ```

  

  
  