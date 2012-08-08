---
layout: post
title: "snowflake简介及核心源码解读"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Snowflake是twitter开源的一款提供产生UID的网络服务软件，简介请猛击[这里](https://github.com/twitter/snowflake)，另外笔者会另外写文章介绍其特性与使用，本文旨在描述snowflake是如何实现UID生成的。

##Snowflake ID组成

Snowflake ID有64bits长，由以下三部分组成：

* time---42bits,精确到ms，那就意味着其可以表示长达(2^42-1)/(1000*3600*24*365)=139.5年，另外使用者可以自己定义一个开始纪元（epoch)，然后用(当前时间-开始纪元）算出time，这表示在time这个部分在140年的时间里是不会重复的，官方文档在这里写成了41bits，应该是写错了。另外，这里用time还有一个很重要的原因，就是可以直接更具time进行排序，对于twitter这种更新频繁的应用，时间排序就显得尤为重要了。

* machine id---10bits,该部分其实由datacenterId和workerId两部分组成，这两部分是在配置文件中指明的。
	
	* datacenterId的作用(个人看法)	

		1.方便搭建多个生成uid的service，并保证uid不重复，比如在datacenter0将机器0，1，2组成了一个生成uid的service，而datacenter1此时也需要一个生成uid的service，从本中心获取uid显然是最快最方便的，那么它可以在自己中心搭建，只要保证datacenterId唯一。如果没有datacenterId，即用10bits，那么在搭建一个新的service前必须知道目前已经在用的id，否则不能保证生成的id唯一，比如搭建的两个uid service中都有machine id为100的机器，如果其server时间相同，那么产生相同id的情况不可避免。

		2.加快server启动速度。启动一台uid server时，会去检查zk同workerId目录中其他机器的情况，如其在zk上注册的id和向它请求返回的work_id是否相同，是否处同一个datacenter下，另外还会检查该server的时间与目前已有机器的平均时间误差是否在10s范围内等，这些检查是会耗费一定时间的。将一个datacenter下的机器数限制在32台(5bits)以内，在一定程度上也保证了server的启动速度。

	* workerId是实际server机器的代号，最大到32，同一个datacenter下的workerId是不能重复的。它会被注册到zookeeper上，确保workerId未被其他机器占用，并将host:port值存入，注册成功后就可以对外提供服务了。

* sequence id ---12bits,该id可以表示4096个数字，它是在time相同的情况下，递增该值直到为0，即一个循环结束，此时便只能等到下一个ms到来，一般情况下4096/ms的请求是不太可能出现的，所以足够使用了。

Snowflake ID便是通过这三部分实现了UID的产生，策略也并不复杂。下面我们来看看它的一些关键源码。

##IdWorker.scala(com.twitter.service.snowflake)

UID生成的核心代码都在这个文件里，我们先来看看其中的一些常量定义。

###常量

{% highlight scala linenos %}

//自定义的开始纪元，这里貌似是tweet纪元，但我算出的结果是Nov 04 09:42:54 CST 2010，不大对
  val twepoch = 1288834974657L
  //workerID的字节数
  private[this] val workerIdBits = 5L
  //datacenterId的字节数
  private[this] val datacenterIdBits = 5L
  //workerId和datacenterId的最大表示值
  private[this] val maxWorkerId = -1L ^ (-1L << workerIdBits)//2^5-1
  private[this] val maxDatacenterId = -1L ^ (-1L << datacenterIdBits)//2^5-1
  //sequenceId的字节数
  private[this] val sequenceBits = 12L
  private[this] val sequenceMask = -1L ^ (-1L << sequenceBits)
  //各个id对应的偏移值
  private[this] val workerIdShift = sequenceBits//12
  private[this] val datacenterIdShift = sequenceBits + workerIdBits//12+5=17
  private[this] val timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits//12+5+5=22

  private[this] var lastTimestamp = -1L


{% endhighlight %}

各个变量的含义请看注释，workerId和datacenterId都是配置文件定义好的，没什么可说的，下面我们看下time和sequence id的产生源码。

{% highlight scala linenos %}

protected[snowflake] def nextId(): Long = synchronized {
    //获取当前时间,ms
    var timestamp = timeGen()

    //lastTimestamp中记录着上一次产生id的时间戳
    if (timestamp < lastTimestamp) {
    	//小于，机器时间出问题了
      exceptionCounter.incr(1)
      log.error("clock is moving backwards.  Rejecting requests until %d.", lastTimestamp);
      throw new InvalidSystemClock("Clock moved backwards.  Refusing to generate id for %d milliseconds".format(
        lastTimestamp - timestamp))
    }
	//相等，递增sequence id
    if (lastTimestamp == timestamp) {
      sequence = (sequence + 1) & sequenceMask
      //递增过程中sequence为0了，表明sequence 值用尽了，等待下一个ms的到来。
      if (sequence == 0) {
        timestamp = tilNextMillis(lastTimestamp)
      }
    } else {
    //大于，将sequence设为0，从头递增
      sequence = 0
    }

	//记录此次产生id的时间戳
    lastTimestamp = timestamp
    //通过shift组装返回id
    ((timestamp - twepoch) << timestampLeftShift) |
      (datacenterId << datacenterIdShift) |
      (workerId << workerIdShift) | 
      sequence
  }

{% endhighlight %}

通过上面的注释相信大家已经明白time和sequence id产生的过程了，是不是特别简单？！优雅的设计带来的也往往是简洁的代码。


##小结

本文主要阐述了snowflake uid的组成和产生策略，希望对大家有所帮助。
