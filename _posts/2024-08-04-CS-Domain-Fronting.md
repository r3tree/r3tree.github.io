---
layout: post
author: r3tree
title: "[分享] 域前置C2隐藏"
date: 2024-08-04
music-id: 
permalink: /archives/2024-08-04/1
description: "域前置C2隐藏"
categories: "网络安全"
tags: [Domain Fronting, c2]
---
域前置步骤：  
如果CDN厂商没有验证`c2域名`归属权，那么可以写成任意高信誉域名。
1. `c2域名`的DNS解析填写为`CDN的DNS地址`
2. `CDN`上设置好DNS解析为`c2服务器的ip`
3. host写`高信誉域名`，host头写`c2域名`

解释：
```bash
curl https://www.allow.com -H "Host: www.forbidden.com" -v
curl https://1.1.1.1 -H "Host: www.forbidden.com" -v  ##1.1.1.1为CDN的IP
```
结果是，客户端实际通信的对象是www.forbidden.com，但在流量监控设备看来，客户端是在与www.allow.com通信，即客户端将流量成功伪装成了与www.allow.com通信的流量
![Domain Fronting](/assets/pic/cs_Domain_Fronting.png)
在`curl https://www.allow.com -H "Host: www.forbidden.com" -v`时，用户用合法的域名allow.com向DNS请求CDN的IP，然后向CDN发起请求，这一步自然是没有任何问题的

因为在处理HTTPS请求时，CDN会首先将它解密，并根据HTTP Host_Header的值做请求转发。

所以用户想要访问一个非法网站www.forbidden.com，可以使用一个CDN上的合法的域名www.allow.com作为SNI，然后使用www.forbidden.com作为HTTP Host与CDN进行HTTPS通信

由于HTTP Host只能被转发看到，而审查者是看不到的，故CDN会放过这条请求，并将HTTP请求根据HTTP Host重新封装，发往www.forbidden.com的服务器，所以，审查者是看不见这个forbidden的

## 域前置简介
域前置（Domain Fronting）基于 HTTPS 通用规避技术，也被称为域前端网络攻击技术。 这是一种用来隐藏 Metasploit、Cobalt Strike 等团队控制服务器流量，以此来一定程度绕过检查器或防火墙检测的技术，如 Amazon、Google、Akamai 等大型厂商会提供一些域前端技术服务。

## 域前置技术原理（CDN 分发）
通过 CDN 节点将流量转发到真实的 C2 服务器，其中 CDN 节点 IP 通过识别请求的 HOST 头进行流量转发，利用我们配置域名的高可信度，如我们可以设置一个微软的子域名，可以有效的躲避 DLP、agent 等流量检测。

## 工作原理（也就是 CDN 工作原理）
域前置的核心是 CDN
CDN 工作原理： 同一个 IP 可以被不同的域名进行绑定加速，再通过 HTTP 请求包头里的 HOST 域名来确定访问哪一个。

详情查看：

> 参考地址1 [域前置 0xl4k1d](https://www.cnblogs.com/0xl4k1d/p/15643269.html)  
> 参考地址2 [域前置溯源方法思考](https://www.anquanke.com/post/id/260888)  
> 参考地址3 [CS 域前置加密 (免备案版)](https://webxxe.cn/index.php/archives/238/)  
> 参考地址4 [我对域前置的理解与思考](https://xdym11235.com/archives/52.html)