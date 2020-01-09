---
layout: post
title: Windos下使用Vagrant构建Linux开发环境
categories: 测试
tags : Vagrant
author: 彭浩
---
# 下载安装
 VirtualBox 虚拟器 : https://www.virtualbox.org/  
 Vagrant : http://www.vagrantup.com/  
 box：http://www.vagrantbox.es/  


# win10家庭版如何安装Hyper-v
* 注意是win10家庭版，因为win10家庭版是没有内置Hyper-v的，需要自己去安装
* 复制如下代码到一个文本，将其命名为Hyper-V.cmd，直接点击等待最后选择Y即可。安装完就可以在Window管理工具（在左下角的程序中找到）就可以找到Hyper-V管理器，同时也能在控制面板->卸载和更改程序->启用和关闭Windows功能中找到Hyper相关