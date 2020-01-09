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

# vagrant up执行时出现“未安装VirtualBox”但实际上已安装的情况解决方式
* 原因是Vagrant需要使用VBoxManager，会在path中扫描VirtualBox的文件目录，详情如下
```
Vagrant (or VBoxManage.exe, for that matter) does not need to be in your PATH environment variable. The Virtual Box installer automatically sets the VBOX_INSTALL_PATH or VBOX_MSI_INSTALL_PATH environment variable which is what Vagrant uses to look it up, but Vagrant cannot run it unless it's elevated.
```
所以解决方式为
```
将
D:\HashiCorp\Vagrant\embedded\gems\2.2.4\gems\vagrant-2.2.4\plugins\providers\virtualbox\driver
中base.rb修改，将所有VBOX_INSTALL_PATH都替换为VBOX_MSI_INSTALL_PATH即可
```

# win10家庭版如何安装Hyper-v
* 注意是win10家庭版，因为win10家庭版是没有内置Hyper-v的，需要自己去安装
* 复制如下代码到一个文本，将其命名为Hyper-V.cmd，直接点击等待最后选择Y即可。安装完就可以在Window管理工具（在左下角的程序中找到）就可以找到Hyper-V管理器，同时也能在控制面板->卸载和更改程序->启用和关闭Windows功能中找到Hyper相关