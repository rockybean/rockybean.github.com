---
layout: post
title: "Google Store扩展无法下载解决办法"
description: ""
category: 
tags: []
---
{% include JB/setup %}

今天下载chrome扩展，网站加载有问题，图片文字都加载不出来，即便使用了智能代理也不行。网上搜了以下，有什么改hosts文件、新建download文件夹之类的方法，都不行。看了下其发送的请求，其中有一个url`googleusercontent.com`，将其加入代理后一切正常了。原因肯定是被误伤关在墙外了……
