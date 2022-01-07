---
title: java反射机制
tags:
  - java
categories:
  - technical
  - java
toc: true
declare: true
date: 2020-08-21 11:36:38
---

# 反射：框架设计的灵魂

- 框架：半成品软件。可以再框架的基础上进行软件开发，简化代码。
- 反射：将类的各个组成部分封装为其他对象，这就是反射机制。

![](http://xwjpics.gumptlu.work/qiniu_picGo/20200821114808.png)

反射的好处：
  1. 在程序的运行中，操作这些对象
  2. 解耦，提高程序的可拓展性

<!-- more -->

# 获取Class对象的方式

* Class.forName("全类名"):将字节码文件加载到内存，返回Class对象
  * 多用于配置文件，将类名定义在配置文件中，读取文件，加载类 
* 类名.class:通过类名的属性class获取
  * 多用于参数的传递
* 对象.getClass(): getClass()在Object类中定义着
  * 多用于对象的获取字节码的方式
    
![](http://xwjpics.gumptlu.work/qiniu_picGo/20200821120200.png)

测试代码
```java
package work.gumptlu.javaAdvanced.reflect;

import work.gumptlu.javaAdvanced.domain.Person;

/**
 * 获取Class对象的方式：
 *   1. Class.forName("全类名"):将字节码文件加载到内存，返回Class对象
 *   2. 类名.class:通过类名的属性class获取
 *   3. 对象.getClass(): getClass()在Object类中定义着
 */

public class ReflectDemo01 {

    public static void main(String[] args) throws ClassNotFoundException {
        fun();
    }

    public static  void fun() throws ClassNotFoundException {
        //1. Class.forName("全类名"):将字节码文件加载到内存，返回Class对象
        Class cls1 = Class.forName("work.gumptlu.javaAdvanced.domain.Person");
        System.out.println(cls1);
        //2. 类名.class:通过类名的属性class获取
        Class cls2 = Person.class;
        System.out.println(cls2);
        //3. 对象.getClass(): getClass()在Object类中定义着
        Person p = new Person();
        Class cls3 = p.getClass();
        System.out.println(cls3);

        //比较三者
        System.out.println(cls1 == cls2);
        System.out.println(cls2 == cls3);
        System.out.println(cls1 == cls3);

    }

}
```

结论：**同一个字节码文件（*.class）在一次程序运行过程中，只会被加载一次，不论是通过哪一种方式获取的class对象都是同一个**


# Class对象功能

## 获取功能：
* 获取成员变量们
	* Field[] getFields() ：获取所有public修饰的成员变量
	* Field getField(String name)   获取指定名称的 public修饰的成员变量

	* Field[] getDeclaredFields()  获取所有的成员变量，不考虑修饰符
	* Field getDeclaredField(String name)  
* 获取构造方法们
	* Constructor<?>[] getConstructors()  
	* Constructor<T> getConstructor(类<?>... parameterTypes)  
  
	* Constructor<T> getDeclaredConstructor(类<?>... parameterTypes)  
	* Constructor<?>[] getDeclaredConstructors()  
* 获取成员方法们：
	* Method[] getMethods()  
	* Method getMethod(String name, 类<?>... parameterTypes)  

	* Method[] getDeclaredMethods()  
	* Method getDeclaredMethod(String name, 类<?>... parameterTypes)  
* 获取全类名	
	* String getName() 

ps:带Declare的都可以获取（可以强制），不带的只能获取public

## Field：成员变量
- 操作：
	1. 设置值
		* void set(Object obj, Object value)  
	2. 获取值
		* get(Object obj) 
  3. 忽略访问权限修饰符的安全检查
		* setAccessible(true):暴力反射
    
## Constructor:构造方法
- 创建对象：
	* T newInstance(Object... initargs)  

	* 如果使用空参数构造方法创建对象，操作可以简化：Class对象的newInstance方法

## Method：方法对象
- 执行方法：
	* Object invoke(Object obj, Object... args)  

	* 获取方法名称：
	  * String getName:获取方法名
