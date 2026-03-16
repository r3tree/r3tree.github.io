---
layout: post
author: r3tree
title: "开源SOAR：OctoMation的安装"
date: 2024-10-28
music-id: 
permalink: /archives/2024-10-28/1
description: "SOAR编排OctoMation的安装"
categories: "网络安全"
tags: [SOAR, OctoMation]
---

OctoMation是接收外部事件信息并开展自动化剧本响应的配置过程

主要逻辑：SIEM产品通过Kafka发了一条告警-->OctoMation接收这个外部事件->根据接入规则识别出特定事件类型->自动触发对应的事件处置剧本
# 安装OctoMation
## 操作系统及软件要求
CPU：4核+  
操作系统：22.04.1-Ubuntu（切记不要用18.04、20.04这是血的教训）    
Docker版本：不低于20.10.12  
Docker-compose版本：1.29.2  
硬盘：200GB+  
`swap`分区空间不少于8GB  
系统防火墙`firewalld`需要处于运行状态  
系统`umask`必须为022

### umask查询
```
root@123:~/桌面# umask
0022
```

### 查看docker-compose版本
`docker-compose --version`

### 检查防火墙firewalld的运行状态
`firewall-cmd --state`输出为running即可

如果需要安装firewalld
`apt install firewalld`

### 确认 systemctl 是否存在
`ls /usr/bin/systemctl`

### 检查Swap空间
查看`free -h`

Swap空间需8G，不够的话执行下面命令
```
sudo dd if=/dev/zero of=/swapfile count=8192 bs=1MiB
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
swapon --show
```
修改或者添加vm.swappiness = 1 到 /etc/sysctl.conf
`vm.swappiness = 1`
修改完成后，执行命令sysctl -p查看结果。

### 安装我用的大文件离线安装，防止不必要的出错
```
# 以root身份执行离线安装脚本
chmod +x octomation_community_docker_install_offline_<VERSION>.sh
./octomation_community_docker_install_offline_<VERSION>.sh
```

接下来就是漫长的安装等待...

### 如果安装时docker报错
```
Error response from daemon: Failed to Setup IP tables: Unable to enable SKIP DNAT rule:  (iptables failed: iptables --wait -t nat -I DOCKER -i br-970a9be1b195 -j RETURN: iptables: No chain/target/match by that name.
(exit status 1))
shakespeare install error,exit
```
重启 Docker 服务：`sudo systemctl restart docker`  
检查 iptables 是否正常工作：`sudo iptables -L -n -v`  
重建 Docker 网络 `docker network prune`

因为老笔记的原因，一开始直接之前虚拟机搭好的ubuntu18，一堆报错后又用了ubuntu20，还是如此。

又下载了ubuntu22桌面版，笔记本老矣，尚能饭否？答案是运行脚本直接崩，又又又下载了server版。

终于！搞了半天的安装，出现久违的安装成功，如释重负。
![installsuccess](/assets/pic/soar/soar1Install.png)
## 登录web管理界面
使用浏览器访问https://x.x.x.x 账号密码admin/octomation进行登录改密
![web](/assets/pic/soar/soar2web.png)

登录后天塌了，需要申请license，此时已经12点陪伴我的蚊子也要开始性生活了，果断看看别人的文章睡觉。
![license](/assets/pic/soar/soar3license.png)

---
申请到license了，安装安装！

剧本中配置动作详情
![配置动作详情](/assets/pic/soar/soar4配置动作详情.png)

安装app
![安装app](/assets/pic/soar/soar5安装app.png)

导入剧本
![导入剧本](/assets/pic/soar/soar5导入剧本.png)

钉钉配置发送消息
![钉钉配置发送消息](/assets/pic/soar/soar6钉钉配置发送消息.png)


> 参考文章1 [cyberzone.cloud开源SOAR的初探](https://cyberzone.cloud/2024/04/11/%E5%BC%80%E6%BA%90SOAR%E7%9A%84%E5%88%9D%E6%8E%A2/#1-%E5%89%8D%E8%A8%80)  
> 参考文章2 [我的第一个OctoMation-事件接入]( https://github-wiki-see.page/m/flagify-com/OctoMation/wiki/%E6%88%91%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AAOctoMation-%E4%BA%8B%E4%BB%B6%E6%8E%A5%E5%85%A5)  
> 参考文章3 [我的第一个OctoMation Playbook]( https://github-wiki-see.page/m/flagify-com/OctoMation/wiki/%E6%88%91%E7%9A%84%E7%AC%AC%E4%B8%80%E4%B8%AAOctoMation-Playbook)