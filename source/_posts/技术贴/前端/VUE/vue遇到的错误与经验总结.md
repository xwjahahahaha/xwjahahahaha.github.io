---
title: vue遇到的错误与经验总结
tags:
  - vue
categories:
  - technical
  - vue
toc: true
declare: true
date: 2020-11-08 21:26:56
---

# 总结Vue开发遇到的错误与经验

## 1. vue.js正确的放置位置

<font color='red'>vue报错：vue.js:634 [Vue warn]: Cannot find element: #app</font>

一般来说，这个问题就是new Vue({})定位ID的js代码位置在html ID元素<font color='red'>之前</font>构建的结果。

<!-- more -->

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<script src="https://unpkg.com/vue/dist/vue.js"></script>
<script src="https://unpkg.com/vue-router/dist/vue-router.js"></script>
    <!-- 自己写的vuejs要放在底下 放在这里是错误的-->
<script src="js/test01.js"></script>
<body>
<div id="app">
  <h1>Hello App!</h1>
  <p>
    <!-- 使用 router-link 组件来导航. -->
    <!-- 通过传入 `to` 属性指定链接. -->
    <!-- <router-link> 默认会被渲染成一个 `<a>` 标签 -->
    <router-link to="/foo">Go to Foo</router-link>
    <router-link to="/bar">Go to Bar</router-link>
  </p>      
  <!-- 路由出口 -->
  <!-- 路由匹配到的组件将渲染在这里 -->
  <router-view></router-view>
    <!-- 正确的位置 -->
   <script src="js/test01.js"></script> 
</div>
</body>
</html>
```

## 2. This和That

当在某一个函数中需要改变当前data中的某个字段，让页面改变时，获取变量需要使用this或者that引用。

具体的使用方法是：<font color='green'>**对于调用api的返回函数时，返回函数内部要使用that，因为this指代的对象已经改变，其他情况使用this**</font>

```js
created() {
			var that = this;
    		console.log("orgThis", this)  					//打印全局的this
			//实例化高得地图插件
			this.amapPlugin = new amap.AMapWX({key:this.amapKey});
			this.amapPlugin.getRegeo({
			  success: function(data){
				//成功回调
                //在这里要使用that，因为这里的this不在是全局的this
                console.log("funcThis:", this)				//打印函数内的this
				that.address_city = data[0].regeocodeData.addressComponent.city
			  },
			  fail: function(info){
				//失败回调
				console.log(info)
			  }
			});
		}
```

两个this的打印结果：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20210125124410.png)

显然是不同的，所以如果在返回函数中使用this是无法改变数据值的。

## 3. 父页面异步调用接口数据通过props传参给子组件无法响应

**症状描述：子组件得到的一直是空值，不管是否写了witch监听**

原因：异步函数请求未结束就已经开始渲染组件了

解决方法：通过async与await变成同步

```js
export async function  Xxxx(A, B){
	//这里要变成同步请求！不然子组件无法获得数据
	await ajax.get({url: url})
	.then(res =>{
		
	}).catch(err => {
		console.log(err);
	});
    return xxx;
}
```

改成同步的缺点就是有等待延迟，那么就使用v-if来让组件加载之前同步获取好数据，得到之后再渲染组件

```vue
<categoryBar :categoryList="categoryList" v-if="loading"></categoryBar>

export default {
		data() {
			return {
				categoryList:[],
				loading: false,
			}
		},
		async onLoad() {
			console.log("onload")
			this.categoryList = await getAllCategory(0, 0);
			this.loading=true
		},
```

当然这种方法也有缺点：不适用于一些必要的显示组件，自行选择使用。

另一种方法见：https://www.jianshu.com/p/4450b63a27fe  中的方法二

## 4. slot-scope="text, record" 参数顺序

使用table的`slot-scope="text, record"`参数的时候，如果想要在slot的渲染中的项中使用record，必须要写text和record，只写一个`slot-scope="record"`则其实默认是第一个参数即（`record`就是`text`）

同理`slot-scope="text, record, index"`，一定要按顺序写且不能省略

## 5. 如何在一些默认参数的响应事件中加入自己的参数

在使用Ant + Vue框架的时候，input输入框自带一个响应事件：`onChange`，其描述如下：

| 事件名称 | 说明                   | 回调参数    |
| :------- | :--------------------- | :---------- |
| change   | 输入框内容变化时的回调 | function(e) |

默认使用：

```vue
<a-input @change="change" />
...
change(e) {
	console.log(e)
}
```

但是如果想在输入框改变的同时传入一些额外的变量则需要像如下这样写：

```vue
<a-input @change="change(record, index, $event)" />
...
change(record, index, e) {
	console.log(record, index, e)
}
```

## 6. 父组件向子组件传递Number类型的值时的静态传递

> * https://www.cnblogs.com/saoge/p/15179659.html

例如父组件与子组件协商好了一个类型：（在子组件A中）

```vue
props: {
    direction: {
        type: Number
    }
},
```

那么父组件如下直接写数字其实传递的是String：

```vue
<A direction=1></A>
```

会报错：

`Invalid prop: type check failed for prop "direction". Expected Number with value 1, got String with value "1".`

要改为动态类型的值并用v-bind绑定：

```vue
<A :direction=inDirection></A>
...
data() {
    return {
        inDirection: -1,
    }
},
```

## 7. Vue数据更新视图不更新的几种解决方案

> * https://blog.csdn.net/bigbear00007/article/details/102594645

vue不能检测以下变动的数组：

1. 如果obj原本没有一个字段，新添加一个字段并改变值，那么是无法响应的
2. 当你利用索引直接设置一个项时，vm.items[indexOfItem] = newValue
3. 当你修改数组的长度时，例如： vm.items.length = newLength

对象属性的添加或删除
由于 Vue 会在初始化实例时对属性执行 getter/setter 转化过程，所以属性必须在 data 对象上存在才能让 Vue 转换它，这样才能让它是响应的。

解决办法：

使用 `Vue.set(object, key, value) `方法将响应属性添加到嵌套的对象上

`Vue.set(vm.someObject, 'b', 2) `或者` this.$set(this.someObject,'b',2) `（这也是全局 Vue.set 方法的别名

## 8. async/await关键字执行异步的时机

* https://blog.csdn.net/qq_30385099/article/details/125805192

总是在使用`async/await`配合使用的时候没有成功同步，其原因在于这对关键字只是在标记了`async`的函数内同步，而在调用该函数的地方是异步。

```js
// 模拟异步发数据
function req(str) {
	return new Promise((resolve, reject) => {
		resolve(str)
	})
}
// 请求方法,通过声明async/await,可等待异步完成之后再执行
async function update(data) {
	await req(data).then(result => {
		str = result;
		console.log("结果2:" + str); // 代码1
	});
	// 代码更新后区域
	console.log("结果3:" + str); // 代码2
}
let str = "你好"; // 代码3
update("hello"); // 代码4
console.log("结果1:" + str); // 代码5
```

<img src="http://xwjpics.gumptlu.work/image-20230227103355005.png" alt="image-20230227103355005" style="zoom:50%;" />

步骤：第一步将父级方法也就是update，声明为异步方法（async），然后再需要等待同步的地方标注为await，这样即可等待`req(data).then()`请求执行成功之后，再执行下面代码。注意：async必不可少，否则报错。

分析：代码执行完3时，因为代码4声明为异步了，故先执行了代码5，执行请求时，发现请求为等待，故等then执行结束后，即执行了代码1，最后执行了代码2。所以代码2区域能够解决实际问题。虽然这样是**解决了update方法内部的同步代码问题，但是变相的将异步问题向上拋了**。update方法为异步了。

