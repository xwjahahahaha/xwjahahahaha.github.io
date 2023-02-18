---
title: vue快速入门
tags:
  - vue
categories:
  - technical
  - vue
toc: true
declare: true
date: 2020-11-07 21:46:21
---

官方文档地址：https://cn.vuejs.org/v2/guide/

# 一、Vue基础

## 1.1 第一个Vue程序

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Vue基础</title>
</head>
<body> 
    <div id="app">
        {{message}}
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
                message:"Hello Vue！"
            }
        })
    </script>
</body>
</html>
```

<!-- more -->

## 1.2 el挂载点

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201107221343.png)

el的作用范围：在el命中的标签**内部**均可被替换，外部不行

el选择器：id:#xxx    class：**.**xxx   标签选择器则原样   （推荐使用ID选择器）

el挂载的标签：注意不建议挂载在其他标签，**一般挂载在div上**，特别的挂载在body、html上会报错

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201107222414.png)

## 1.3 data数据对象

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201107223028.png)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>data数据对象</title>
</head>
<body> 
    <div id="app">
        {{message}}
        <h2>{{ school.name }} {{ school.mobile }}</h2>
        <ul>
            <li>{{ campus[0] }}</li>
            <li>{{ campus[1] }}</li>
            <li>{{ campus[2] }}</li>
        </ul>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
                message:"Hello Vue！",
                school:{
                    name: "黑马程序员",
                    mobile:"400-618-9090"
                },
                campus:["北京校区", "上海", "广州"]
            }
        })
    </script>
</body>
</html>
```

# 二、本地应用

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201107223337.png)

## 2.1 v-text指令

```html
<body> 
    <div id="app">
        <h2 v-text="message + '!!!'">哈哈哈</h2>
        {{ message + "!!!"}}
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
                message:"Hello Vue！",
                info:"lala"
            }
        })
    </script>
</body>
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201107224103.png)

## 2.2 v-html指令

```html
<body> 
    <div id="app">
        <!-- html会把html渲染成为标签 -->
       <p v-html="content"></p>
        <!-- text不会把html渲染成为标签 -->
       <p v-text="content"></p>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
               content: "<a href='#'>黑马程序员</a>"
            }
        })
    </script>
</body>
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201107224801.png)

## 2.3 v-on指令

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201107225140.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201107230126.png)

```html
<body> 
    <div id="app">
       <input type="button" value="v-on指令" v-on:click="doIt">
       <input type="button" name="" id="" value="点我"   @click="doIt">
       <!-- 双击事件 -->
       <input type="button" name="" id="" value="点我"   @dblclick="doIt">
       <!-- 事件与数据处理 -->
       <h2 @click="change"> {{ food }} </h2>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
                food:"西蓝花炒鸡蛋"
            },
            methods:{
                doIt:function(){
                    alert("做IT");
                },
                change:function(){
                   this.food += "好好吃哦！"
                }
            },

        })
    </script>
</body>
```

v-on带参数：

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108101815.png)

```html
<body> 
    <div id="app">
        {{message}}
        <button v-on:click="click('老铁', '666')">点击</button>
        <input type="text" @keyup.enter="sayHi">
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
                message:"Hello Vue！"
            },
            methods:{
                click:function(p1, p2){
                    console.log(p1 + p2);
                },
                sayHi:function(){
                    alert("吃了吗？");
                }
            }
        })
    </script>
</body>
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108103032.png)



## 2.4  v-show

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108005545.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108010428.png)

```html
<body> 
    <div id="app">
        <img src="./meixi.png" alt="" width="80px" v-show="isShow">
        <button @click="changeShow">点我</button>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
               isShow: true
            },
            methods:{
                changeShow:function(){
                    this.isShow = !this.isShow
                }
            }
        })
    </script>
</body>
```

## 2.5 v-if

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108011143.png)

```html
<body> 
    <div id="app">
        <img src="./meixi.png" alt="" width="80px" v-show="isShow">
        <img src="./meixi.png" alt="" width="80px" v-if="isShow">
        <button @click="changeShow">点我</button>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
               isShow: true
            },
            methods:{
                changeShow:function(){
                    this.isShow = !this.isShow
                }
            }
        })
    </script>
</body>
```

## 2.6 v-bind

注意是**单**大括号

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108011441.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108013137.png)

```html
<body> 
    <div id="app">
        <img v-bind:src="imgSrc" v-bind:alt="imgAlt" v-bind:width="imgWidth" v-bind:title="imgTitle">
        <!-- v-bind 简写的方式 : -->
        <br>
        <!-- 使用三元表达式 -->
        <img :src="imgSrc" :alt="imgAlt" :width="imgWidth" :title="imgTitle" :class="isActive?'active':''" @click="chActive">
        <br>
        <!-- 用对象的方式而不使用三元表达式 -->
        <img :src="imgSrc" :alt="imgAlt" :width="imgWidth" :title="imgTitle" :class="{active: isActive}" @click="chActive">
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
               imgSrc: "./meixi.png",
               imgWidth: "80px",
               imgAlt: "梅西",
               imgTitle: "梅西",
               isActive: false
            },
            methods:{
                chActive:function(){
                    this.isActive = !this.isActive
                }
            }
        })
    </script>
</body>
```

## 2.7 v-for

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108100341.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108101340.png)



```html
<body> 
    <div id="app">
        <ul>
            <li v-for="(item, index) in objArr">{{index + "-" +  item.name +  item.age}}</li>
        </ul>
        <button @click="add">添加</button>
        <button @click="sub">删除</button>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
               objArr:[
                   {"name": "xwj", "age": 15},
                   {"name": "rt", "age": 18},
                   {"name": "lal", "age": 85},
                   {"name": "hj", "age": 566},
                   {"name": "ty", "age": 15}
               ]
            },
            methods:{
                add:function(){
                    this.objArr.push({"name":"济良", "age": 18})
                },
                sub:function(){
                    this.objArr.shift();
                }
            }
        })
    </script>
</body>
```

## 2.8 v-model（双向数据绑定）

双向数据绑定：更改双方的任何一方，双方都会同步的更改

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108104104.png)

```html
<body> 
    <div id="app">
        <input type="button" value="点击" @click="setM">
       <input type="text" v-model="message" @keyup.enter="getM">
       <h2>{{message}}</h2>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var app = new Vue({
            el:"#app",
            data:{
               message: "你好哈哈哈哈哈",
            },
            methods:{
              getM:function(){
                  alert(this.message);
              },
              setM:function(){
                  this.message = "改了"
              }
            }
        })
    </script>
</body>
```

## 2.9 插槽slot、slot-scope、v-slot

> * https://juejin.cn/post/6844903555837493256
> * https://www.bilibili.com/video/BV1qC4y147UR/?spm_id_from=333.337.search-card.all.click&vd_source=52e3776de8b0541a4e4db8921bcd8d6d

<img src="http://xwjpics.gumptlu.work/image-20230215143541085.png" alt="image-20230215143541085" style="zoom:50%;" />

**插槽，也就是slot，是组件的一块HTML模板，这块模板显示不显示、以及怎样显示由父组件来决定。**子组件中写了插槽，父组件则负责往插槽中添加内容

### 匿名插槽 slot

**单个插槽**是vue的官方叫法，但是其实也可以叫它默认插槽或者与具名插槽相对，我们可以叫它匿名插槽。因为它不用设置name属性。子组件中。单个插槽可以放置在组件的任意位置，但是就像它的名字一样，一个组件中只能有一个该类插槽。

例子：

<img src="http://xwjpics.gumptlu.work/image-20230215144318216.png" alt="image-20230215144318216" style="zoom: 43%;" />

效果如图：

<img src="http://xwjpics.gumptlu.work/image-20230215144341697.png" alt="image-20230215144341697" style="zoom:50%;" />

### 具名插槽 slot

匿名插槽没有name属性，所以是匿名插槽，那么，插槽**加了name属性**，就变成了具名插槽。具名插槽可以在一个组件中出现N次，出现在不同的位置。下面的例子，就是一个有两个**具名插槽**和**单个插槽**的组件，这三个插槽被父组件用同一套css样式显示了出来，不同的是内容上略有区别。

例子：

父组件：

```vue
<template>
  <div class="father">
    <h3>这里是父组件</h3>
    <child>
      <div class="tmpl" slot="up">
        <span>菜单1</span>
        <span>菜单2</span>
        <span>菜单3</span>
        <span>菜单4</span>
        <span>菜单5</span>
        <span>菜单6</span>
      </div>
      <div class="tmpl" slot="down">
        <span>菜单-1</span>
        <span>菜单-2</span>
        <span>菜单-3</span>
        <span>菜单-4</span>
        <span>菜单-5</span>
        <span>菜单-6</span>
      </div>
      <div class="tmpl">
        <span>菜单->1</span>
        <span>菜单->2</span>
        <span>菜单->3</span>
        <span>菜单->4</span>
        <span>菜单->5</span>
        <span>菜单->6</span>
      </div>
    </child>
  </div>
</template>
```

子组件：

```vue
<template>
  <div class="child">
    // 具名插槽
    <slot name="up"></slot>
    <h3>这里是子组件</h3>
    // 具名插槽
    <slot name="down"></slot>
    // 匿名插槽
    <slot></slot>
  </div>
</template>
```

### 作用域插槽 | 带数据的插槽 slot-scope

最后，就是我们的作用域插槽。这个稍微难理解一点。官方叫它作用域插槽，实际上，对比前面两种插槽，我们可以叫它带数据的插槽。什么意思呢，就是前面两种，都是在组件的template里面写, 但是**作用域插槽要求，在slot上面绑定数据**。也就是你得写成大概下面这个样子：

```vue
<slot name="up" :data="data"></slot>
 export default {
    data: function(){
      return {
        data: ['zhangsan','lisi','wanwu','zhaoliu','tianqi','xiaoba']
      }
    },
}
```

我们前面说了，插槽最后显示不显示是看父组件有没有在child下面写模板，像下面那样：

```vue
<child>
   html模板
</child>
```

写了，插槽就总得在浏览器上显示点东西，东西就是html该有的模样，没写，插槽就是空壳子，啥都没有。 OK，我们说有html模板的情况，就是父组件会往子组件插模板的情况，那到底插一套什么样的样式呢，这由父组件的html+css共同决定，但是这套样式里面的内容呢？

正因为作用域插槽绑定了一套数据，父组件可以拿来用。于是，情况就变成了这样：**样式父组件说了算，但内容可以显示子组件插槽绑定的**

我们再来对比，作用域插槽跟单个插槽和具名插槽的区别，因为单个插槽和具名插槽不绑定数据，所以父组件提供的模板一般要既包括样式又包括内容，上面的例子中，你看到的文字，“菜单1”，“菜单2”都是父组件自己提供的内容；**而作用域插槽，父组件只需要提供一套样式（在确实用作用域插槽绑定的数据的前提下）**

下面的例子，你就能看到，父组件提供了三种样式(分别是flex、ul、直接显示)，都没有提供数据，数据使用的都是子组件插槽自己绑定的那个数组（一堆人名的那个数组）

父组件：

```vue
<template>
  <div class="father">
    <h3>这里是父组件</h3>
    <!--第一次使用：用flex展示数据-->
    <child>
      <template slot-scope="user">
        <div class="tmpl">
          <span v-for="item in user.data">{{item}}</span>
        </div>
      </template>

    </child>

    <!--第二次使用：用列表展示数据-->
    <child>
      <template slot-scope="user">
        <ul>
          <li v-for="item in user.data">{{item}}</li>
        </ul>
      </template>

    </child>

    <!--第三次使用：直接显示数据-->
    <child>
      <template slot-scope="user">
       {{user.data}}
      </template>

    </child>

    <!--第四次使用：不使用其提供的数据, 作用域插槽退变成匿名插槽-->
    <child>
      我就是模板
    </child>
  </div>
</template>
```

子组件：

```vue
<template>
  <div class="child">
    <h3>这里是子组件</h3>
    // 作用域插槽
    <slot  :data="data"></slot>
  </div>
</template>

 export default {
    data: function(){
      return {
        data: ['zhangsan','lisi','wanwu','zhaoliu','tianqi','xiaoba']
      }
    }
}
```

效果：

<img src="http://xwjpics.gumptlu.work/image-20230215145149939.png" alt="image-20230215145149939" style="zoom: 50%;" />













## 2.10 备忘录

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108130342.png)

```html
<body>
    <!-- 主体区域 -->
    <section id="todoapp">
      <!-- 输入框 -->
      <header class="header">
        <h1>备忘录</h1>
        <input
          autofocus="autofocus"
          autocomplete="off"
          placeholder="请输入任务"
          class="new-todo"
          v-model="userInput"
          @keyup.enter="add"
        />
      </header>
      <!-- 列表区域 -->
      <section class="main">
        <ul class="todo-list">
          <li class="todo" v-for="(item, index) in things">
            <div class="view">
              <span class="index">{{index + 1}}</span> <label>{{item}}</label>
              <button class="destroy" @click="remove(index)"></button>
            </div>
          </li>
        </ul>
      </section>
      <!-- 统计和清空 -->
      <footer class="footer">
        <span class="todo-count"  v-show="things.length>0"> <strong>{{things.length}}</strong> items left </span>
        <button class="clear-completed" @click="removeAll"  v-show="things.length>0">
          Clear
        </button>
      </footer>
    </section>
    <!-- 开发环境版本，包含了有帮助的命令行警告 -->
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script>
        var todoapp = new Vue({
            el: "#todoapp",
            data: {
                userInput:"",
                things: ["恰饭"]
            },
            methods: {
                add:function(){
                    this.things.push(this.userInput);
                    this.userInput = "";
                },
                removeAll:function(){
                    this.things = [];
                },
                remove:function(index){
                this.things.splice(index, 1);
            }
            },
            
        }) 
    </script>
  </body>
```

# 三、网络应用

**Vue结合网络数据开发应用**

axios： 小巧的轻量级网络请求库

axios + vue

## 3.1 axios的基本使用



![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108132931.png)

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108135213.png)

```html
<body>
    <input type="button" value="get请求" class="get">
    <input type="button" value="post请求" class="post">
    <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
    <script>
        document.querySelector(".get").onclick = function(){
            axios.get("https://autumnfish.cn/api/joke/list?num=3")
            .then(function(res){
                console.log(res.data.jokes)
            }),function(err){
                console.log(err)
            }
        }

        document.querySelector(".post").onclick = function(){
            axios.post(
            "https://autumnfish.cn/api/user/reg123123",
            {username:"xwj"})
            .then(function(res){
                console.log(res)
            }, function(err){
                console.log(err)
            })
        }

       
    </script>

</body>
```

## 3.2 axios + Vue

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108141006.png)



```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>axios+vue</title>
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
</head>
<body>
    <div id="app">
        <input type="button" value="获取笑话" @click="getJoke">
        <p>{{message}}</p>
    </div>
    
</body>
</html>

<script>
    var app = new Vue({
        el:"#app",
        data: {
            message: "笑话"
        },
        methods: {
           getJoke:function(){
            //    这一步很关键，在axios里面this都是undefind
              var that = this
              axios.get("https://autumnfish.cn/api/joke")
              .then(function(res){
                 that.message = res.data 
              }, function(err){
                 console.log(err)
              })
           }
        }
    })
</script>
```

## 3.3 天知道案例

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>天知道</title>
    <link rel="stylesheet" href="css/reset.css" />
    <link rel="stylesheet" href="css/index_tq.css" />
  </head>

  <body>
    <div class="wrap" id="app">
      <div class="search_form">
        <div class="logo"><img src="images/logo.png" alt="logo" /></div>
        <div class="form_group">
          <input
            type="text"
            class="input_txt"
            placeholder="请输入查询的天气"
            v-model="input"
            @keyup.enter="query"
          />
          <button class="input_sub" @click="query">
            搜 索
          </button>
        </div>
        <!-- <div class="hotkey">
          <a href="javascript:;" @click="chooiseCity('北京')">北京</a>
          <a href="javascript:;" @click="chooiseCity('上海')">上海</a>
          <a href="javascript:;" @click="chooiseCity('广州')">广州</a>
          <a href="javascript:;" @click="chooiseCity('深圳')">深圳</a>
          <a href="javascript:;" @click="chooiseCity('重庆')">重庆</a>
        </div> -->

        <div class="hotkey" >
            <a href="javascript:;" v-for="(v, i) in simpleArray" @click="chooiseCity(v)">{{v}}</a>
        </div>
      </div>
      <ul class="weather_list">
        <li v-for="(v, i) in dataArray">
          <div class="info_type"><span class="iconfont">{{v.type}}</span></div>
          <div class="info_temp">
            <b>{{v.low}}</b>
            ~
            <b>{{v.high}}</b>
          </div>
          <div class="info_date"><span>{{v.date}}</span></div>
        </li>
      </ul>
    </div>
    <!-- 开发环境版本，包含了有帮助的命令行警告 -->
    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <!-- 官网提供的 axios 在线地址 -->
    <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
    <!-- 自己的js -->
    <script src="js/main.js"></script>
  </body>
</html>

```

```js
var app = new Vue({
    el: "#app",
    data: {
        input: "",
        dataArray: [],
        simpleArray: ["北京", "上海", "广州", "重庆", "常熟", "台湾"]
    },
    methods:{
        query(){
            var url = "http://wthrcdn.etouch.cn/weather_mini" + "?city=" + this.input;
            var that = this
            axios.get(url)
            .then(function(res){
                that.dataArray = res.data.data.forecast
            }, function(err){
                alert(err) 
            })
        },
        chooiseCity(city){
            this.input = city
            // 调用同级函数
            this.query()
        }
    }
})
```

![](http://xwjpics.gumptlu.work/qiniu_picGo/20201108151414.png)

# 四、综合应用-悦听音乐播放器

**开发一款音乐播放器**功能如下：

 ![image-20201108151627916](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20201108151627916.png)

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta http-equiv="X-UA-Compatible" content="ie=edge" />
  <title>悦听player</title>
  <!-- 样式 -->
  <link rel="stylesheet" href="./css/index_mp3.css">
</head>

<body>
  <div class="wrap">
    <!-- 播放器主体区域 -->
    <div class="play_wrap" id="player">
      <div class="search_bar">
        <img src="images/player_title.png" alt="" />
        <!-- 搜索歌曲 -->
        <input type="text" autocomplete="off" v-model="query" @keyup.enter="searchMusic" placeholder="输入关键字"/>
      </div>
      <div class="center_con">
        <!-- 搜索歌曲列表 -->
        <div class='song_wrapper'>
          <ul class="song_list">
           <!-- <li><a href="javascript:;"></a> <b>你好</b> </li> -->
           <li v-for="(v, i) in queryResArray"><a href="javascript:;" @click="startPlay(v)"></a><b>{{v.name}}-{{v.artists[0].name}}</b> <span v-if="v.mvid!=0"><i  @click="openVideo(v)"></i></span></li>
          </ul>
          <img src="images/line.png" class="switch_btn" alt="">
        </div>
        <!-- 歌曲信息容器 -->
        <div class="player_con" :class="{playing:isPlaying}">
          <img src="images/player_bar.png" class="play_bar" />
          <!-- 黑胶碟片 -->
          <img src="images/disc.png" class="disc autoRotate" />
          <img :src="songCover" class="cover autoRotate" />
        </div>
        <!-- 评论容器 -->
        <div class="comment_wrapper">
          <h5 class='title'>热门留言</h5>
          <div class='comment_list'>
            <dl v-for="v in hotComments">
              <dt><img :src="v.user.avatarUrl" alt=""></dt>
              <dd class="name">{{ v.user.nickname}}</dd>
              <dd class="detail">
                {{ v.content }}
              </dd>
            </dl>
          </div>
          <img src="images/line.png" class="right_line">
        </div>
      </div>
      <!-- 播放条 -->
      <div class="audio_con">
        <audio @play="play" @pause="pause" ref='audio'  :src="songSrc" controls autoplay loop class="myaudio"></audio>
      </div>
      <!-- 遮罩层播放器 -->
      <div class="video_con" v-show="videoPlay" style="display: none;">
        <video  :src="videoSrc" controls="controls"></video>
        <div class="mask" @click="quitVedio"></div>
      </div>
    </div>
  </div>
  <!-- 开发环境版本，包含了有帮助的命令行警告 -->
  <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
  <!-- 官网提供的 axios 在线地址 -->
  <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
  <script src="js/listen_main.js"></script>
</body>

</html>
```

```js
var app = new Vue({
    el: "#player",
    data: {
        query: "",
        queryResArray: [],
        songSrc: "",
        songCover: "",
        commentArray: [],
        hotComments: [],
        isPlaying: false,
        videoSrc: "",
        videoPlay: false

    },
    methods:{
        searchMusic(){
           var that = this
           axios.get("https://autumnfish.cn/search" + "?keywords=" + that.query)
           .then(function(res){
                that.queryResArray = res.data.result.songs
               console.log(res.data.result.songs)
           },function(err){
               alert(err);
           })
        },
        startPlay(song){
            var that = this
            axios.get("https://autumnfish.cn/song/url" + "?id=" + song.id)
            .then(function(res){
                that.songSrc = res.data.data[0].url
                if(that.songSrc == null){
                    alert("暂无资源！")
                }
                console.log(res)
            }, function(err){
                console.log(err)
            }),
            axios.get("https://autumnfish.cn/song/detail" + "?ids=" + song.id)
            .then(function(res){
                that.songCover = res.data.songs[0].al.picUrl
                console.log(res)  
            }, function(err){
                alert(err)
            }),
            axios.get("https://autumnfish.cn/comment/hot?type=0&id=" + song.id)
            .then(function(res){
                that.hotComments = res.data.hotComments
                console.log(res)
            }, function(err){
                console.log(err)
            })
        },
        play(){
           this.isPlaying = true
           console.log("play")
        },
        pause(){
            this.isPlaying = false
            console.log("pause")
        },
        openVideo(song){
            var that = this
            axios.get("https://autumnfish.cn/mv/url?id=" + song.mvid)
            .then(res=>{
                that.videoSrc = res.data.data.url
                that.videoPlay = true
                console.log(res)
            }, err=>{
                console.log(err)
            })
        },
        quitVedio(){
            this.videoPlay = false
        }

    }
})
```

