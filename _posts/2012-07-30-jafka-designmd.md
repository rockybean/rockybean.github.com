---
layout: post
title: "jafka design.md"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Jafka设计理念浅析
===

本文将从以下几个方面去尝试讲解Jafka的设计理念，主要参考文献在[这里](http://incubator.apache.org/kafka/design.html)：  

* Jafka设计背景及原因

* Jafka设计的几个特色  
    * 消息存储到磁盘
    * 强调吞吐率   
    * 消费状态由消费者自己维护
    * 分布式

## Jafka设计背景及原因



	



