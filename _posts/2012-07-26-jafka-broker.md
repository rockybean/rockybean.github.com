---
layout: post
title: "Jafka 服务端(Broker)使用示例与配置简要说明"
description: ""
category: 
tags: []
---
{% include JB/setup %}

本文主要讲解Jafka中服务端broker的使用和配置，最后还会说明下jafka服务端文件夹bin下面部分脚本的用处。

  Jafka服务端broker的启动有两种方式：作为单独应用启动和内嵌在其他应用中启动。前者可以通过Jafka服务端bin文件夹下的脚本实现，后者需要用户在自己的应用代码中嵌入Jafka的启动代码。

##作为单独应用启动
Jafka服务端作为独立的应用运行，主要应用场景是搭建Jafka集群，供多个其他应用使用。Jafka服务端bin文件夹下已经提供了很方便的脚本，用户直接使用即可。相关脚本为：

* `server.sh`

  **使用示例为：**
	
		bash bin/server.sh conf/server.properties
	指明配置文件位置即可，关于配置文件可以查看下面的[详细讲解][config]。

* `run.sh`

	1.1.0版本后Jafka使用Java Service Wrapper进一步对服务端启动进行了封装，简化了启动命令，帮助信息如下：


		Usage: ./run.sh [ console | start | stop | restart | condrestart | status | install | remove | dump ]
		Commands:
		  console      Launch in the current console.
		  start        Start in the background as a daemon process.
		  stop         Stop if running as a daemon or in another console.
		  restart      Stop if running and then start.
		  condrestart  Restart only if already running.
		  status       Query the current status.
		  install      Install to start automatically when system boots.
		  remove       Uninstall.
		  dump         Request a Java thread dump if running.


	Jafka对linux提供了32位和64位两个版本，用户使用时注意使用对应版本，修改方式为run.sh中

		#WRAPPER_CMD="./jafka"
		WRAPPER_CMD="./jafka64"

	将符合自己的命令保留即可。更详细的使用文档，请猛击[这里](https://github.com/adyliu/jafka/wiki/install.zh_CN "jafka安装详解")。

##内嵌在其他应用中启动

这种方式的主要应用场景是在一个应用中嵌入jafka的服务端（broker）代码，将jafka作为消息队列使用，一般只启动一个服务端（broker）。使用代码如下：

{% highlight java linenos %}
Properties props = new Properties();
props.setProperty("port","9093");
props.setProperty("log.dir","/home/alfred/jafkaDataDirs/data1");
Jafka broker = new Jafka();
broker.start(props,null,null);
broker.awaitShutdown();
{% endhighlight %}

上述代码运行完毕，即在本机9093端口启动了一个broker，之后便可以使用producer consumer连接该broker进行消息的生产和消费。上面的代码逻辑也很简单，首先配置服务器的相关信息（可以参见下面的配置介绍），之后将配置传入broker即可。这么几行代码即可将jafka嵌入到自己的应用中，并拥有了一个小巧强悍的消息队列。


## broker常用配置说明

###基本配置

| 参数名   | 参数意义  | 默认值/示例   |
|:-------:|:--------:|:-------:|
| brokerid| 该broker的唯一标识 | 无      |
| port    | 该broker绑定端口   | 9092   |
| num.partitions   | topic的默认partition数目，用户可以通过topic.partition.count.map来为每个topic指定自己的partition数目   | 1   |
| topic.partition.count.map   | 为每个topic指定自己的partition数目，覆盖默认配置   | topic1:10,topic2:20   |
{: class="table table-striped table-bordered" text-align="center"}


###消息数据存储的相关配置


| 参数名   | 参数意义  | 默认值/示例   |
|:-------:|:--------:|:-------:|
| log.dir | 消息数据存储目录，即jafka文件的存储地址 | 无      |
| log.file.size    | 配置单个jafka文件的大小，超过该数目后便新建一个jafka文件。单位为B。   | 100MB   |
{: class="table table-striped table-bordered"}


###强制写磁盘（flush操作）配置

Jafka存储消息数据到磁盘上，而操作系统为了提高操作效率，会对写操作进行缓冲，但程序可以调用flush系统调用强制操作系统写磁盘，只有写入磁盘的数据才能被其他程序读取。


| 参数名   | 参数意义  | 默认值/示例   |
|:-------:|:--------:|:-------:|
|log.flush.interval|该配置指明Jafka为每个topic缓冲了该数目的消息后便强制写磁盘。|500|
|log.default.flush.scheduler.interval.ms|Jafka会后台运行一个线程去定期检查所有可写的jafka文件是否已经超过了flush.interval.ms时间（下方的配置），如果超过了，则强制写磁盘。|3000|
|log.default.flush.interval.ms|默认每个topic强制写磁盘的间隔时间，必须在设定了log.default.flush.scheduler.interval.ms后才有意义。可以通过下面的配置为每个topic指定自己flush的时间。| |
|topic.flush.intervals.ms|配置每个topic flush的时间，实时性要求高的可以将此值配置小一点。|topic1:1000,topic2:2000|
{: class="table table-striped table-bordered"}


###日志保留相关配置

日志写入磁盘后不会永久保存，需要一个保留机制，比如超过了一定时间或者超过了一定大小后就将日志删除或者存档打包等。


| 参数名   | 参数意义  | 默认值/示例   |
|:-------:|:--------:|:-------:|
|log.retention.hours|日志在磁盘的保留时间，单位为小时|168，即一周|
|log.retention.size|日志增长到一定大小后进行部分删除操作。|-1，即不限定大小。|
|topic.log.retention.hours|配置每个topic保留的时间|topic1:10,topic2:20|
|log.cleanup.interval.mins|Jafka后台会启动一个cleanup线程，定期检查jafka文件是否可删除，判断依据是在log.retention.hours时间内jafka文件没有改动。|10|
{: class="table table-striped table-bordered"}


###SocketServer相关配置

Jafka使用NIO技术，开启一个acceptor线程处理连接请求，多个processor线程处理具体业务。详细实现后续会写文章介绍。


| 参数名   | 参数意义  | 默认值/示例   |
|:-------:|:--------:|:-------:|
|max.connections|一个broker最多同时处理的连接请求数| |
|socket.receive.buffer|Server Socket的配置SO_RCVBUFF的大小|102400B|
|num.threads|配置处理客户端请求的worker线程的数目|CPU数目|
{: class="table table-striped table-bordered"}


###zookeeper配置

| 参数名   | 参数意义  | 默认值/示例   |
|:-------:|:--------:|:-------:|
|enable.zookeeper|是否启用zookeeper|false|
|zk.connect|zookeeper的连接地址| |
|zk.connectiontimeout.ms|zookeeper连接超时时间|6000ms|
|zk.sessiontimeout.ms|zookeeper 会话超时时间|6000ms|
{: class="table table-striped table-bordered"}

###管理员帐号配置

Jafka添加了简单的权限机制，支持plain、md5、crc32三种格式。

> password=plain:jafka

配置文件示例请参考`conf/server.properties`文件



##bin文件夹下脚本说明

在Jafka服务端bin文件夹下有许多脚本，这里对其中几个的使用作下说明。

###admin-console.sh

该脚本可以创建、删除、修改topic的一些信息。

	Create/Delete Topic
	Usage: -c -h <host> -p <port> -t <topic> -P <partition> [-e]
           -d -h <host> -p <port> -t <topic> --password <password>
	Option                                  Description                            
	------                                  -----------                            
	-P, --partition [Integer: partition]    topic partition (default: 1)           
	-c, --create                            create topic                           
	-d, --delete                            delete topic                           
	-e, --enlarge                           enlarge partition number if exists     
	-h, --host <host>                       server address                         
	-p, --port <Integer: port>              server port                            
	--password [password]                   jafka password                         
	-t, --topic <topic>                     topic name  
####示例：
1.修改hehe的partition数目为3

	>bin/admin-console.sh -c -h localhost -p 9093 -t hehe -P 3 -e
2.删除hehe2的所有文件

	>bin/admin-console.sh -d -h localhost -p 9093 -t hehe2 --password jafka

###dumper.sh

用于查看jafka文件内的信息

	Usage: [options] --file <file>...
	Option                                  Description                            
	------                                  -----------                            
	-c <Integer: count>                     max count mesages. (default: -1)       
	--file <filepath>                       decode file list                       
	--no-message                            do not decode message(utf-8 strings)   
	--no-offset                             do not print message offset            
	--no-size                               do not print message size         

####示例：
* 查看该jafka文件中的3条信息
>bin/dumper.sh -c 3 --file ~/jafkaDataDirs/data1/hehe-0/00000000000000000000.jafka 
结果为：

		offset|size|message  
		0|5|data1  
		15|5|data0  
		30|5|data1  

###getoffset-console.sh

可以查看某段时间内消息的偏移值(offset)

	Option                                  Description                            
	------                                  -----------                            
	--offsets <Integer: count>              number of offsets returned (default: 1)
	--partition <Integer: partition_id>     partition id (default: 0)              
	--server <jafka://ip:port>              REQUIRED: the jafka request uri        
	--time <Long: unix_time>                unix time(ms) of the offsets.          
	                                          -1: lastest; -2: earliest; unix      
	                                          million seconds: offset before this  
	                                          time (default: -1)                   
	--topic <topic>                         REQUIRED: The topic to get offset from.

####示例

显示hehe topic下面分区0内最近时间内3条消息的位移(offset)值

>bin/getoffset-console.sh --offsets 3 --partition 0 --server jafka://localhost:9092 --time -1 --topic hehe

结果为：


	get 3 result
	54700239
	54626449
	53577563

##小结

Jafka服务端的使用就介绍到这里，只说不练假把式，通过笔者关于producer、consumer和broker的介绍，相信读者应该可以较好地使用Jafka了，如果有什么问题或者好的建议，欢迎留言讨论！


[config]: #config





