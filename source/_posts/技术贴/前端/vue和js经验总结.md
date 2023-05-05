---
title: vue和js经验总结
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2023-03-01 10:52:28
---

[TOC]


<!-- more -->

# vue和js经验总结

## 1. slot-scope="text, record" 参数顺序

使用table的`slot-scope="text, record"`参数的时候，如果想要在slot的渲染中的项中使用record，必须要写text和record，只写一个`slot-scope="record"`则其实默认是第一个参数即（`record`就是`text`）

同理`slot-scope="text, record, index"`，一定要按顺序写且不能省略

## 2. 如何在一些默认参数的响应事件中加入自己的参数

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

## 3. 父组件向子组件传递Number类型的值时的静态传递

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

## 4. Vue数据更新视图不更新的几种解决方案

> * https://blog.csdn.net/bigbear00007/article/details/102594645
> * https://www.cnblogs.com/ypSharing/p/updataHandler.html

### 情况分类

vue页面视图不更新的情况如下：

- [Vue 无法检测实例被创建时不存在于 data 中的 属性](https://www.cnblogs.com/ypSharing/p/updataHandler.html#1vue-无法检测实例被创建时不存在于-data-中的-属性)
- [ Vue 无法检测‘对象属性’的添加或移除](https://www.cnblogs.com/ypSharing/p/updataHandler.html#2-vue-无法检测对象属性的添加或移除)
- [Vue 不能检测利用数组索引直接修改一个数组项](https://www.cnblogs.com/ypSharing/p/updataHandler.html#3vue-不能检测利用数组索引直接修改一个数组项)
- [Vue 不能监测直接修改数组长度的变化](https://www.cnblogs.com/ypSharing/p/updataHandler.html#4vue-不能监测直接修改数组长度的变化)
- [在异步更新执行之前操作 DOM 数据不会变化](https://www.cnblogs.com/ypSharing/p/updataHandler.html#5在异步更新执行之前操作-dom-数据不会变化)
- [循环嵌套层级太深，视图不更新？](https://www.cnblogs.com/ypSharing/p/updataHandler.html#6循环嵌套层级太深视图不更新)
- [路由参数变化时，页面不更新（数据不更新）](https://www.cnblogs.com/ypSharing/p/updataHandler.html#7路由参数变化时页面不更新数据不更新)
- [使用keep-alive之后数据无法实时更新问题](https://www.cnblogs.com/ypSharing/p/updataHandler.html#8使用keep-alive之后数据无法实时更新问题)

### 解决办法

#### 1. Vue.set

使用 `Vue.set(object, key, value) `方法将响应属性添加到嵌套的对象上

`Vue.set(vm.someObject, 'b', 2) `或者` this.$set(this.someObject,'b',2) `(这也是全局 Vue.set 方法的别名)

#### 2. 计算属性

还可以使用计算属性解决。例如场景是：

```js
<a-step v-for="item in steps" :key="item.title" :title="item.title" />
...
data() {
      return {
          steps: [{ title: this.$t('sgroup.addnew.basicinfo') }, { title: this.$t('sgroup.addnew.relateVms') }, { title: this.$t('sgroup.addnew.rules') }],
      }
},
```

vue中使用国际化，动态返回的steps数组中的对象值要根据中英文选项动态变化，但是页面无法感知title的变化

解决：

删掉data中的steps，写一个同名的计算属性:

```js
computed: {
    steps() {
        return [
            { title: this.$t('sgroup.addnew.basicinfo') },
            { title: this.$t('sgroup.addnew.relateVms') },
            { title: this.$t('sgroup.addnew.rules') }
        ]
    }
},
```

## 5. async/await关键字执行异步的时机

* https://blog.csdn.net/qq_30385099/article/details/125805192

### 基本概念

#### Promise

> 相关网站：
>
> * Promise的基本概念：https://zh.javascript.info/promise-basics

首先一定需要明白的是，promise 的处理程序 `.then`、`.catch` 和 `.finally` 都是异步的。即便一个 promise 立即被 resolve，`.then`、`.catch` 和 `.finally` **下面** 的代码也会在这些处理程序之前被执行。

所以下面的代码会先执行（即使立即resolve）。

例如：

```js
let promise = new Promise((resolve, reject) => {
  console.log(1)
  resolve()
})
promise.then(() => console.log(3))

console.log(2)
```

输出的结果是

```text
1， 2， 3
```

**调用resolve()会触发异步操作，传入的then()方法的函数会被添加到任务队列并异步执行**，具体内部实现的队列可以参考：[微任务队列](https://zh.javascript.info/microtask-queue#wei-ren-wu-dui-lie-microtaskqueue)

简单地说，当一个 promise 准备就绪时，它的 `.then/catch/finally` 处理程序就会被放入队列中：但是它们**不会立即被执行**。当 JavaScript 引擎执行完当前的代码，它会从队列中获取任务并执行它。

#### async/await

在函数前面的 “async” 这个单词表达了一个简单的事情：即这个函数总是返回一个 promise。其他值将自动被包装在一个 resolved 的 promise 中。

关键字 `await` 让 JavaScript 引擎等待直到 promise 完成（settle）并返回结果。

一个简单的例子：

```js
async function f() {

  let promise = new Promise((resolve, reject) => {
    setTimeout(() => resolve("done!"), 1000)
  });

  let result = await promise; // 等待，直到 promise resolve (*)

  alert(result); // "done!"
}

f();
```

这个函数在执行的时候，“暂停”在了 `(*)` 那一行，并在 promise settle 时，拿到 `result` 作为结果继续往下执行。所以上面这段代码在一秒后显示 “done!”。

### 为什么总是无法同步？

总是在使用`async/await`配合使用的时候没有成功同步，关键在于两点：

#### 1. await要等待一个promise才有用

当你在await一个函数的结果时，那个函数必须要返回的是一个promise，这样await才有用，例如：

```js
function f1() {
  return new Promise(resolve => {			// return 不写那么就是 1111 -> 3333 -> 2222
    console.log('1111')
    setTimeout(() => resolve('done'), 1000)
  }).then(() => {
    console.log('2222')
  })
}

async function f2() {
  await f1()			// 等到一个promise，如果不写await，那么顺序就是 1111 -> 3333 -> 2222
  console.log('3333')
}

f2()		// 1111 -> 2222 -> 3333
```

> 为什么先输出1111再输出3333？
>
> * [Promise对象的构造函数中的函数（executor）是同步执行的，也就是说，它会在Promise对象创建后立即运行](https://towardsdev.com/promises-in-javascript-285f523c3e8d)[4](https://towardsdev.com/promises-in-javascript-285f523c3e8d)。但是，executor中可以包含异步操作，比如网络请求或者定时器，所以如果有这些异步操作，如果不使用await就会先执行promise后面的代码。

#### 2. await的同步作用只保留在当前函数内部，async让其异步上抛了

`await`关键字只是在标记了`async`的函数内同步，而`async`字段的作用是表明此函数是异步函数（默认给此函数返回一个promise），所以在调用该函数的地方是异步。

```js
// 模拟异步发数据
function req(str) {
	return new Promise((resolve, reject) => {
		resolve(str)
	})
}
// 请求方法,通过声明async/await,可等待异步完成之后再执行
async function update(data) {
	await req(data).then(result => {		// 只能做到对req函数的同步效果
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

上述代码的解决方法就是：

1. 再对update返回的Promise使用await 
2. 对update之后要做的事放到`.then()`处理 （也就是promise的基础用法）

## 6. Promise链式调用的i一个误解

以下两种调用的代码看似相同，其实差别很大

第一种：

```js
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000); // (*)

}).then(function(result) { // (**)

  alert(result); // 1
  return result * 2;

}).then(function(result) { // (***)

  alert(result); // 2
  return result * 2;

}).then(function(result) {

  alert(result); // 4
  return result * 2;

});
```

这样是Promise的链式调用，每一次then的返回结果都会向下传递，核心原因在于：**then返回的还是一个Promise**

<img src="http://xwjpics.gumptlu.work/image-20230301150527027.png" alt="image-20230301150527027" style="zoom:50%;" />

第二种：

```js
let promise = new Promise(function(resolve, reject) {
  setTimeout(() => resolve(1), 1000);
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});
```

这是一种典型的**伪链式调用**，其不是链式调用，所做的就仅仅是添加promise的几个处理程序而已，所以，在上面的代码中，所有 `alert` 都显示相同的内容：`1`。

<img src="http://xwjpics.gumptlu.work/image-20230301150644010.png" alt="image-20230301150644010" style="zoom:50%;" />

## 7. 计算属性如何带参数

> * https://blog.csdn.net/qq_42988836/article/details/106542901

在使用计算属性的时候如果返回的是一个字符串值而不是一个函数是无法携带参数的，会报如下错：

![image-20230427105909629](http://xwjpics.gumptlu.work/image-20230427105909629.png)

解决方法：

return一个函数

```js
// 计算select的样式
computeSelectClass() {
    return (record) => {
        if (this.onlyShowMode) {
            return 'special-select-showmode-style'
        } else {
            if (record.disabled) {
                return 'special-select-disable-style'
            } else {
                return 'special-select-style'
            }
        }
    }
}
```

