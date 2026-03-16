---
layout: post
author: r3tree
title: "学习go笔记"
date: 2024-10-01
music-id: 
permalink: /archives/2024-10-01/1
description: "golang"
categories: "网络安全"
tags: [golang, study]
---

## 变量
`a := "aaa"` 冒号为定义相当于var，只能在局部变量使用。
## 常量与iota
`iota`只能够配合`const()` 一起使用，`iota`只有在`const`进行累加效果。
```go
const(
	a, a1= iota,   iota+2  //默认为0，之后依次累加+1
	b, b1	// 1，3
	c, c1	//  2，4
)
```
## 函数
```go
func foo(a string, b int) (r1 int, r2 int) {
	fmt.Println("a =", a)
	fmt.PrintIn("b = ", b)
	//r1 r2 属于foo的形参初始化默认的值是0
	//r1 r2 作用域空间 是foo 整个函数体的{}空间
	fmt.PrintIn("r1 = ", r1)
	fmt.Println("r2 = ", r2)
	//给有名称的返回值变量赋值
	r1 = 1000
	r2 = 2000
	return
}
```
## 导包特性
在import导其他包时，其他包里API对外公开 首字母大写代表public，小写私有。

在import导包时，匿名导包 `_ "lib1"` 代表只调用lib1的`init()`

```go
import (
	mylib "GolangStudy/lib1" //起别名mylib
	. "GolangStudy/lib2" //.代表直接导入到当前包 不建议
)
func main() {
	mylib.Lib1Test() //调用别名mylib
	Lib2Test() //.代表直接导入到当前包 直接调用
}
```
## 指针
```go
func main() {
	var a int = 10
	var p *int // *int 中的*表示指针类型

	p=&a // &表示传的地址
	fmt.Println(&a) //0x123
	fmt.Println(p) //0x123

	var pp **int // 二级指针
	pp = &p
	fmt.Println(&p) //0x456
	fmt.Println(pp) //0x456
```
持续更新中...