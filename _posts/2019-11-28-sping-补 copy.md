---
layout: post
title: Spring 使用问题（补）
categories: Spring
tags : Spring
author: 彭浩
---

下面为碰到的一些问题
## 问题大全

（1）使用aop时出现，cvc-complex-type.2.4.c: 通配符的匹配很全面, 但无法找到元素 'aop:config' 的声明，但idea已经能够提示  
  该问题可以直接百度搜索，然后复制正确的aop的schema文件即可

（2）报错：Caused by: java.lang.ClassNotFoundException: org.aspectj.weaver.reflect ReflectionWorld$ReflectionWorldException  
该问题需要导入对应版本的aspectjweaver.jar包，找对应的jar包下载，然后在FILE->Project Structor中引入(+)即可