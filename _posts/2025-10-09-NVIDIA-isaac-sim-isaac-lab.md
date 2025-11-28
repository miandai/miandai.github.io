---
layout: post
keywords: blog
description: 心血来潮想玩下机器人仿真 NVIDIA Isaac Sim，想来英伟达出品，必定 so easy，结果光安装就把我干吐血了，灰头土脸。
title: "具身智能仿真踩坑记"
categories: [Robot]
tags: [Robot]
excerpt: 心血来潮想玩下机器人仿真 NVIDIA Isaac Sim，想来英伟达出品，必定 so easy，结果光安装就把我干吐血了，灰头土脸。
location: 北京
author: 增益
---

# Isaac Sim

心血来潮想玩下机器人仿真 NVIDIA Isaac Sim，想来英伟达出品，必定 so easy，结果光安装就把我干吐血了，灰头土脸。





一开始就选择了 CS 部署架构，把 Isaac Sim 安装到公司共用的GPU服务器上，我工作 MAC 作为客户端 UI，正好 Isaac Sim 也支持。往服务器上安装时起初用 Workstation 安装方式，配置各种环境，然后就发现 ubuntu 版本太低，Isaac Sim 不允许使用低版本，个人没权限升级，于是果断放弃这种方式，改用 Docker Container 安装。Docker Container 很快就遇到 GPU Driver Version 过低问题（报错日志模糊不清，各种查才知道原因），尝试各种方法都不行，GPU Driver 同样没法动。折腾半天，对共用服务器彻底死心了。

既然开始了，就不想放弃，死活得跑通一把。共用服务器难搞，就找独享服务器。买一台当然没必要，GPU 价格不是闹着玩的。于是看上了 GPU 云服务器，按量计费，用完就释放，简单划算。依次尝试了阿里云、autodl、腾讯云，还是用 Docker Container 安装方式。

首先支持集团阿里云，折腾了半天，不知是什么原因，安装 NVIDIA Container Toolkit  时老访问 网页链接 失败，容器里跑 Isaac Sim 就老报 CUDA error，玩不转，不得不放弃。然后尝试 autodl（autodl 的产品界面比阿里云强多了，阿里云界面乱成鬼），选配置、付费，进入系统，然后很快发现个严重问题，这服务器本身就是个容器，容器内不能允许再跑容器，此路不通，赶紧释放跑路。

最后抱着死马当活马医的心态尝试了腾讯云，安装环节意外地顺利，没有遇到阿里云上的诸多问题，网络通畅，isaac-sim 跑起来了，挺激动。但我的 MAC 客户端 Isaac Sim WebRTC Streaming Client 依然无法访问，于是先定位了服务器端口被腾讯云安全规则屏蔽访问，在腾讯云控制台的安全组规则加了白名单，外部可访问该端口了。但 MAC 客户端依然一团漆黑，也没任何报错，想到可能是这个 MAC 本的芯片 Apple M2 跟 Isaac Sim 提供的 macOS arm64、macOS x86_64 软件版本不兼容，于是用手头的另外一台 intel 芯片的 MAC 装了个 Client 试了下，瞬间就展示出华丽的界面了。算是对自己有交代了。

Apple M2 是我工作用电脑，性能最好用得最多，这么高配置竟然不能玩，有点不甘心，不过遍寻良方依然无果，只能认了，尽力了。有台能跑通的可以了。

系统跑通，看似微不足道，但在我看来，已经算是万里长征迈出了第一步，必须庆贺下。


<center><img src="/image/issac_sim/issac-sim.png"></center>


# Isaac Lab

安装完 NVIDIA Isaac Sim，发现 Isaac Lab 才是集大成者，于是开始安装 Isaac Lab。这次使用官方提供的 Alibaba Cloud Deployment，只要付费充足，Docker 安装包会调用 SDK 自动连接阿里云申请服务器资源，并配置环境。

<center><img src="/image/issac_sim/system.jpg"></center>

安装文档看似简单，但实操起来困难重重。最大的挡路虎就是网络，包括公司VPN、加速VPN、阿里云网络。前两个网络跟本机有关，公司VPN连接必须关闭，加速VPN有时必须开启，有时必须关闭，开关时机不对就是不行。最让人头疼的是阿里云网络。

阿里云服务器没有加速，有些海外安装包不能访问。有时就尝试本机下载后传到服务器上，这种方式还需修改安装代码，通过URL获取改为本地文件读取，巨麻烦，阿里云客服提了个方案，建议 ping 到 ip 地址后加入 /etc/hosts，实验可行，但对有些访问无效，只能网上费力搜解决方案。为摆脱网络问题困扰，一度尝试使用阿里云香港服务器，但不知为何资源不可用，客服也没说明白，无奈只能继续专攻北京服务器。

之前安装  Isaac Sim 在腾讯云服务器，就没这种鸟网络问题，阿里云这问题是只有我遇到吗？

搞定在阿里云服务器遇到的网络问题后，快看到成功希望了，最后报了个Request omniverse-launcher-linux.AppImage failed. Because NVIDIA has deprecated the Omniverse Launcher on October 1st, 2025. 安装包 omniverse-launcher-linux.AppImage 已从英伟达网站彻底消失，搜都搜不到，这被弃用也才刚过十天。

折腾心累了。

<center><img src="/image/issac_sim/install_error.jpg"></center>

<center><img src="/image/issac_sim/omniverse.png"></center>

等等吧，这玩意儿还得频繁更新。

