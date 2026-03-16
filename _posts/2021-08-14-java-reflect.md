---
layout: post
author: r3tree
title: "什么是java反射?"
date: 2021-08-14
music-id: 
permalink: /archives/2021-08-14/1
description: "对java反射的学习"
categories: "网络安全"
tags: [java, reflect]
---

## 什么是反射?
允许对封装类的字段，方法和构造函数的信息进行编程访问。  
允许对成员变量，成员方法和构造方法的信息进行编程访问。  
就是从类里面拿东西。  

idea里的自动提示就是用的反射

## 获取class对象的三种方式
1. Class.forName("全类名");
2. 类名.class
3. 对象.getclass(); 

例：
```java
// 1.第一种方式 源码阶段
// 全类名：包名+名
// 最为常用的
Class clazz1 = Class.forName("com.test.myreflect.Student");

// 2.第二种方式 加载阶段
// 一般更多的是当做参数进行传递
Class clazz2 = Student.class;

// 3.第三种方式 运行阶段
// 当我们已经有了这个类的对象时，才可以使用。
Student s = new Student();
Class clazz3 = s.getClass();
```
## 构造方法

```java
// Class类中用于获取构造方法的方法
Constructor<?>[] getDeclaredConstructors() // 返回所有构造方法对象的数组
Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes) // 返回单个构造方法对象

// Constructor类中用于创建对象的方法
T newlnstance(Object... initargs) // 根据指定的构造方法创建对象
setAccessible(boolean flag) // 设置为true,表示取消访问检查
```
例：
```java
Class clazz = Class.forName("com.test.myreflect.Student");
Constructor con = clazz.getDeclaredConstructor(String.class,int.class);
// 表示临时取消权限校验
con.setAccessible(true);
Student stu = (Student)con.newInstance("disbb",18);
```

## 字段(成员变量)
```java
// Class类中用于获取成员变量的方法
Field[] getDeclaredFields() // 返回所有成员变量对象的数组
Field getDeclaredField(String name) // 返回单个成员变量对象 

// Field类中用于创建对象的方法
void set(Object obj, Object value) // 赋值
Object get(Object obj) // 获取值
```
例：
```java
Class clazz = Class.forName("com.test.myreflect.Student");
Field name = clazz.getDeclaredField("name");
// 获取成员变量记录的值
Student s = new Student("disbb", 18, "男");
s.setAccessible(true);
Object value = name.get(s); //输出值为：disbb，获取当前s的name值。
```
## 成员方法
```java
// Class类中用于获取成员方法的方法
Method[] getDeclaredMethods() // 返回所有成员方法对象的数组,不包括继承的 
Method getDeclaredMethod(String name, Class<?>... parameterTypes) // 返回单个成员方法对象

// Method类中用于创建对象的方法
Object invoke(Object obj, Object.. args) // 运行方法
参数一: 用obj对象调用该方法
参数二: 调用方法的传递的参数(如果没有就不写)
返回值: 方法的返回值(如果没有就不写)
```
```java
// 获取指定的单一方法
Class clazz = Class.forName("com.test.myreflect.Student");
Method method = clazz.getDeclaredMethod("方法名字", String.class);

Student s = new Student();
method.setAccessible(true);
// 参数— s ：表示方法的调用者
// 参数二"disbb"：表示在调用方法的时候传递的实际参数
method.invoke(s, "disbb");
```