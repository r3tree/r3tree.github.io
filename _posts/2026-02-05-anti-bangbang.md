---
layout: post
author: r3tree
title: "APP脱壳：梆梆企业版"
date: 2026-02-05
music-id: 
permalink: /archives/2026-02-05/1
description: "APP梆梆企业版加固"
categories: "网络安全"
tags: [bangbang, anti]
---

## 查壳

![image-20260316215959353](/assets/pic/bangbang2026/image-20260316215959353.png)

## 启动frida

![image-20260316220140512](/assets/pic/bangbang2026/image-20260316220140512.png)

## 查看启动app闪退点

`frida -U -f com.xxx.xxx -l .\1hook_dlopen.js` 

![image-20260316220602281](/assets/pic/bangbang2026/image-20260316220602281.png)

libDexHelper.so 加载后frida退出

## 找闪退地址

`frida -U -f com.xxx.xxx -l .\3hook_remove_findKillAddr.js`

发现0x31ec0时frida退出

![image-20260316221411616](/assets/pic/bangbang2026/image-20260316221411616.png)

提取libDexHelper.so到IDA里分析，打开发现已混淆

![image-20260316221647175](/assets/pic/bangbang2026/image-20260316221647175.png)

## 内存dump so解混淆

打开app并执行`python3 dump_so.py libDexHelper.so`

![image-20260316221844582](/assets/pic/bangbang2026/image-20260316221844582.png)

使用IDA打开解混淆的so文件，跳转地址`0x31EC0`

![image-20260316222050436](/assets/pic/bangbang2026/image-20260316222050436.png)

F5

![image-20260316222144246](/assets/pic/bangbang2026/image-20260316222144246.png)

很明显frida关键字，进行nop掉

![image-20260316222308041](/assets/pic/bangbang2026/image-20260316222308041.png)

成功绕过frida检测

## 内存dump dex拿源码

`frida -UF -l dexCache_dump.js`

![image-20260316222414378](/assets/pic/bangbang2026/image-20260316222414378.png)

拿到源码，jdax打开愉快的找加解密

![image-20260316222649468](/assets/pic/bangbang2026/image-20260316222649468.png)
