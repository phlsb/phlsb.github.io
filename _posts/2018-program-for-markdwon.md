---
layout: post
title:  "MarkDown的一些使用技巧"
categories: 工具
tags:  hexo JavaScript
author: 彭浩
---

* content
{:toc}

## 插入图片

（1）将图片存入markdown文件

基本格式：!\[Alt text](图片链接 "optional title")
  
高级用法（用base64转码工具将图片转换为一串字符串，随后将字符串填充在基础格式的那个链接中）：  
![avatar][base64str]  
[base64str]:data:image/png;base64,iVBORw0......

base64的图片的编码可以使用python来转化

（1）将图片转化为base64的字符串

```python

import base64
f=open('723.png','rb') #二进制方式打开图文件
ls_f=base64.b64encode(f.read()) #读取文件内容，转换为base64编码
f.close()
print(ls_f)

```

（2）将base64字符串转化为图片

```python

import base64
bs='iVBORw0KGgoAAAANSUhEUg....' # 太长了省略
imgdata=base64.b64decode(bs)
file=open('2.jpg','wb')
file.write(imgdata)
file.close()

```




