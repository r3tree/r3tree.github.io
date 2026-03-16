---
layout: post
author: r3tree
title: "安卓模拟器安装Magisk、xposed、LSPosed和SSL pinning单双向绕过"
date: 2023-12-18
music-id: 
permalink: /archives/2023-12-18/1
description: "golang"
categories: "网络安全"
tags: [xposed, SSL pinning]
---

## burp安装证书到系统而非用户
因为安卓7.1不认用户证书
系统证书路径：/system/etc/security/cacerts/
> 参考文章 [burp安装证书到系统](https://www.cnblogs.com/xpro/p/15540426.html)

## Mumu模拟器12安装Magisk面具和LSPosed
[安装教程](https://github.com/cr4n5/XiaoYuanKouSuan/blob/main/README_EMULATOR.md)

## 雷电模拟器9.22安装Magisk面具和LSPosed
1. 雷电模拟器->设置->其他设置->Root权限->开启
2. 雷电模拟器->设置->性能设置->System.vmdk可写入
3. 安装Magisk-18a0b94a-delta(25205).apk，运行并给予root权限
4. 选择右上角安装弹出“允许“magisk Delta”访问您设备上的照片、媒体内容和文件吗？“选择允许，然后结束magisk进程。
5. 从新运行magisk选择右上角安装-选择修补boot镜像中的vbmeta和安装到Recovery
6. 选择安装至系统分区出现All done！字样，然后点雷电模拟器右上角选择重启雷电模拟器
```
如果弹出检测到不属于magisk的su文件，请删除其他超级用户程序。处理方法如下：
    安装R.E文件管理器.APK并打开给予root权限和文件访问权限
    用R.E文件管理器进入/system/xbin目录找到SU文件改名为SU.BAK或删除
    然后重启雷电模拟器解决完毕！
```
7. 运行magisk模块选项安装Shamiko-v0.6(126).zip和Zygisk_-_LSPosed-v1.8.5(6649).zip重启雷电模拟器
8. 重启完进入magisk发现两个模块都没有启用，进入设置
9. 勾选Zygisk，并安装LSPosed 1.8.5.apk（不装会没有LSPosed图标）然后重启雷电模拟器

> 参考文章 [雷电模拟器9.22安装Magisk面具和edxposed ](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1718014)

## 单向验证 绕过SSL pinning
下载SSLUnpinnning: [https://github.com/ac-pm/SSLUnpinning_Xposed/blob/master/mobi.acpm.sslunpinning_latest.apk](https://github.com/ac-pm/SSLUnpinning_Xposed/blob/master/mobi.acpm.sslunpinning_latest.apk)
之后xp框架勾选 绕过单向严重

## 双向验证
雷电用不来就换了Nox，xp框架更方便。  
先查看安卓模拟器adb的端口号，我的是16384  
```bash
adb connect 127.0.0.1:16384

查看设备并连接
adb devices
adb -s 127.0.0.1:16384 shell

如果有问题可以：关闭服务并重新打开服务
adb kill-server
adb start-server
```
### 方案1: Firda+r0capture+WireShaTk
```bash
pip3 install frida
pip3 install frida-tools
```
查看frida版本为16.5.6  
模拟器里执行`getprop ro.product.cpu.abi`  
下载[https://github.com/frida/frida/releases/tag/16.5.6](https://github.com/frida/frida/releases/tag/16.5.6)   
解压后的文件`frida-server-16.5.6-android-x86_64`cp到雷神根目录  
```bash
adb devices
adb shell
.\adb.exe push .\frida-server-16.5.6-android-x86_64 /data/local/frida
```
```bash
如果出现couldn't create file: Permission denied`没有权限导致的
adb root
adb remount #重新加载文件系统
再次尝试：adb push
```
之后安卓模拟器开启监听
```bash
.\adb.exe shell
cd /data/local
chmod +x frida
./frida
```
再开个cmd窗口执行：`PS C:\APP\Nox\bin> frida-ps -R`检查是否和安卓模拟器通信  
`Failed to enumerate processes: unable to connect to remote frida-server`  
这个时候就需要进行转发，在`C:\APP\Nox\bin`目录下运行：`.\adb forward tcp:27042 tcp:27042`  

`PS C:\APP\Nox\bin> frida-ps -R`列出安卓模拟器的app代表成功  
执行命令绕过并抓包
`python3 ./r0capture.py  -U -f com.p1.mobile.putong -p tantan.pcap`
### 方案2: Firda+HOOK-JS+BurpSuite
`frida -U -f com.p1.mobile.putong -l D:\FridaScripts\SSLUnpinning.js`

