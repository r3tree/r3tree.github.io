---
layout: post
author: r3tree
title: "[复现] 利用x64dbg破解注册Sublime Text 4"
date: 2024-09-11
music-id: 
permalink: /archives/2024-09-11/1
description: "利用x64dbg破解Sublime Text 4的关于界面是否注册成功"
categories: "二进制安全"
tags: [Sublime Text 4, crack]
---

在了解完二进制的时候看到一篇文章 自己动手试了下

## 使用DIE查下软件
![64bit](/assets/pic/SublimetextCrack/1查64位.jpg)

打开x64dbg

把Sublime Text.exe直接拖进x64dbg里
## 点击帮助->关于 显示未注册
![Unregistered](/assets/pic/SublimetextCrack/1.2关于未注册.jpg)
## 搜索当前模块的字符串 "Unregistered"
![search](/assets/pic/SublimetextCrack/2当前模块搜索字符串.jpg)

![search2](/assets/pic/SublimetextCrack/3搜索字符串Unregistered.jpg)

双击此地址
## 发现一个jmp点
可以看到下面就显示"Unregistered"字符串

![jmp](/assets/pic/SublimetextCrack/4jmp点.jpg)

## 关键点 move ecx,220
可以看到字符串上方有个jmp跳过它，那很明显jmp下面的`move ecx,220`就是关于界面未注册赋值的入口

对`move ecx,220`这个地址ctrl+r找下引用
![ecx220](/assets/pic/SublimetextCrack/5对ecx220找引用.jpg)

## 双击je地址
![ikun](/assets/pic/SublimetextCrack/6对ecx220找引用后双击.jpg)

## cmp为关键比较 je为关键跳转
所以对cmp处下断点运行 再次点击关于 断下来到断点处 FPU中RSI寄存器值指向了地址xxxxxxxxxC8
![cmp](/assets/pic/SublimetextCrack/7cmp断点.jpg)

`cmp byte ptr ds:[rsi+5],0`
可以看到0和rsi+5比较 所以我们需要改的地址是RSI+第五个值
## RSI右键->在内存中转到，查看内存
很明显这地方为0的时候标识未注册
![RSI+5](/assets/pic/SublimetextCrack/8RSI内存中转到是和RSI+5比较.jpg)

把值改为1，运行之验证下结果。
![RSI+5](/assets/pic/SublimetextCrack/9RSI内存中转到修改为1.jpg)
## 关于界面 成功注册
![success](/assets/pic/SublimetextCrack/10关于界面成功显示注册.jpg)

后面的文章我也测试成功了 但是我懒得写咯

> [本文作者地址](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1468154&extra=&highlight=sublime&page=1)