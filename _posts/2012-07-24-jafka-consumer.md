---
layout: post
title: "Consumer使用示例与配置简要说明"
description: ""
category: 
tags: []
---
{% include JB/setup %}

通过上一篇文章的介绍，相信大家都已经掌握了使用Producer发送消息的方式，消息已经有了，那接下来就是如何消费了。在讲解如何消费前，首先要和大家说明一下Jafka对消息的存储格式，因为消费消息时某些方法需要使用者明了消息的存储格式。

上一篇文章中Producer发送消息时要指定topic名称，并且用户可以自定义partition.class来分配自己的消息到固定的分区(partition)中，由此可见Jafka对于消息的存储是以topic-partition为一个单元来处理的，在存储设备中的呈现方式也是如此，一个topic-partition是一个文件夹，比如topic名为test,partition数目为3，那么如果只有一个broker的话，该broker的data文件夹下便有test-0，test-1，test-2三个文件夹，如果是一个broker集群，那么可能在一个broker上是test-0,test-1，在另外的一个broker上是test-0，即所有broker上的topic-partition数目相加等于自定义的partition数目。

在topic-partition文件夹下面便是一个个存储消息的文件了，比如在test-0文件夹下可能有如下的文件：

	00000000000000000000.jafka，
	00000000000000001024.jafka，
	00000000000000002048.jafka……

其中00000000000000000000.jafka表示最开始的文件，起始偏移量为0，00000000000000001024.jafka表明该文件的起始偏移量为1024，而上一个文件的大小为1024-0=1024Byte，00000000000000002048.jafka表明该文件的起始偏移量为2048，而上一个文件的大小为2048-1024=1024Byte。这里有几点要说明，如下：

1.jafka文件名的数字总共有20位，能够表达上EB(1EB=1024PB)级别的数据(offset)，这也就意味着一个topic-partition文件夹可以存储上EB的数据，虽然实际使用中用不到这么大的数据量，但这也算是Jafka扩展性的一个体现吧。

2.偏移量(offset)是在消费消息时会用到的一个概念。生产者发送到服务端的消息数据被顺序存储在\*.jafka文件中，那么要表示一个消息数据最少要有两部分：该消息在该topic-partition中的偏移量(offset)和该消息的实际数据（后面会专门写文章来讲解Jafka消息数据的存储格式），对应的类便是MessageAndOffset。该offset便是指明该消息在这其topic-partition中的起始偏移量。consumer提供了fetch方法可以让用户自主地抓取某个offset开始的消息数据，函数为：

>public ByteBufferMessageSet fetch(FetchRequest request)

FetchRequest中指定offset等，后面会有对应的例子。

3.jafka使用offset作文件名是为了根据offset快速找到其所在的文件，比如用户要抓取从1306位置开始，最多1M大小的消息数据集合(MessageSet)，此时服务端便可以在所有的jafka文件名列表中以二分查找的方式，快速定位1306偏移的消息数据存储在00000000000000001024.jafka文件中，又该文件大小为1024B，所以实际传输的数据大小为(1024+1024)-1306=742B，这742B的数据中可能包含若干条消息。如果抓取时指定最多抓取100B大小的数据，那么实际传输的数据是否就是100B哪？当然不一定，比如1024其实的消息数据为90B，其后的消息数据为30B，这两条消息数据大小为130B，大于100B，对于这种情况只返回前面90B大小的消息，以后再讲解服务端(broker）处理fetch请求源码时再给大家讲解其原理。

大家在明白了消息的存储方式后，再使用消费消息的代码就很容易了。Jafka Consumer消费消息数据的方式也分为同步和异步两种方式，这两种方式的区别在于同步消费使用fetch等low-level的api，单线程地消费数据（因为它必须指明topic-partition，而每一个topic-partition只能被一个线程消费，否则无法保证消息消费的有序性），而异步消费多是以线程池的形式发起多个消费者组成一个消费组(consumer group)的方式来消费消息。一般如果没有特殊的需求，推荐使用异步消费。

##同步消费

代码如下：

{% highlight java linenos %}
SimpleConsumer consumer = new SimpleConsumer("127.0.0.1",9092);
long offset = 0;
//指定fetch请求的参数：topic partition offset maxSize
FetchRequest fetch = new FetchRequest("person",0,offset,1024*1024);
try {
	//获取消息数据集合
	ByteBufferMessageSet messageSet = consumer.fetch(fetch);
	//遍历消息数据集合
	for(MessageAndOffset messageAndOffset:messageSet){
	//从消息中获取代表实际数据的byte数组
	ByteBuffer buffer = messageAndOffset.message.payload();
	byte[] objBytes = new byte[buffer.remaining()];
	buffer.get(objBytes);
	//反序列化字节数组为对象
	ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(objBytes));
	Person tmpPerson = (Person)ois.readObject();
	System.out.println("person:"+tmpPerson.getName()+","+tmpPerson.getId());
	ois.close();
}
} catch (IOException e) {
	e.printStackTrace(); 
} catch (ClassNotFoundException e) {
	e.printStackTrace(); 
}
{% endhighlight %}


上述代码实现功能为抓取person topic第0个partition中从0开始最多1M大小的消息数据集合。fetch后返回的是ByteBufferMessageSet 类对象，遍历它可得MessageAndOffset对象，其中包含了消息数据及其offset，之前producer传递的byte数组便可由message中获得，之后用户按照自己的方式处理该数组即可，上面例子的处理方式即将字节流反序列化为对象。
通过上面的代码，相信大家已经注意到在初始化consumer时，必须要指定broker的地址，抓取消息数据时也必须要详细到topic-partition，甚至offset，所以称其为low-level的api，除非特别需求，一般不会使用。

##异步消费

代码如下：

{% highlight java linenos %}
Properties props = new Properties();
//指明zookeeper地址
props.setProperty("zk.connect","localhost:2181");
//指明consumer group的名字
props.setProperty("groupid","test_group");
ConsumerConfig config = new ConsumerConfig(props);
//创建了ZookeeperConsumerConnector，连接zookeeper，获取当前topic的数据信息
ConsumerConnector connector = Consumer.create(config);
//指明每一个topic的消费线程数
Map<String,Integer> topicCountMap = new HashMap<String, Integer>();
topicCountMap.put("hehe",2);
topicCountMap.put("hehe3",1);
//创建消费消息流，key为topic，value为MessageStream的list，大小为上面map中指定的大小
Map<String,List<MessageStream<String>>> streams = connector.createMessageStreams(topicCountMap,new StringDecoder());
List<MessageStream<String>> messageStreamList = streams.get("hehe");
messageStreamList.addAll(streams.get("hehe3"));
final AtomicInteger count = new AtomicInteger(0);
final AtomicInteger streamCount = new AtomicInteger(0);
//创建线程池，该线程池数目必须不小于上面所有的消费线程数
ExecutorService executor = Executors.newFixedThreadPool(3);
//提交消费任务，开始消费消息
for(final MessageStream<String> stream:messageStreamList){
executor.execute(new Runnable() {
	@Override
	public void run() {
		int threadNum = streamCount.incrementAndGet();
		//从stream中获取消息，此处为阻塞式消费，即当没有新消息到来时，阻塞直到新消息到来或者线程结束
		//通过BlockingQueue实现，后续内容会详细讲解
		for(String msg:stream){
		System.out.println("stream#"+threadNum+":msg#"+count.incrementAndGet()+"=>"+msg);
		}
	}
});
}
try {
	executor.awaitTermination(1, TimeUnit.HOURS);
} catch (InterruptedException e) {
	e.printStackTrace();
}

{% endhighlight %}


代码的具体意义，请参照注释。这里创建了一个groupid为test_group的消费组，运行一次这个代码就会生成一个Consumer，每一个consumer都有自己唯一的id，系统自动生成的id格式为host_name-current_time-uuid。该consumer消费hehe和hehe3这两个topic，并且为hehe开设2个stream，可以理解为线程，为hehe3开设一个stream进行消费。可能大家对stream这个概念会有些糊涂，下面笔者尝试给大家说明。

Jafka设计consumer时指明其可以并行消费消息，这里并行消费是以topic-partition为基本单元的，即每一个topic-parition只能给一个线程消费，否则无法保证其消息消费的有序性。对于有多个partition的topic，jafka有自己的负载均衡算法，这里简单举例说明下，在后面文章中再详细阐述其实现原理。比如hehe有5个分区，Jafka以集群形式运行，假设有两个broker，其id分别为0和1，又假设在broker0中hehe有hehe-0,hehe-1两个partition,在broker1中有hehe-0,hehe-1,hehe-2三个partition。运行上面的代码两次，就意味着test_group的消费组中有两个consumer，且每个consumer都开设2个stream消费hehe的消息数据。负载均衡算法是这样进行的，首先它从zookeeper中获取hehe这个topic的所有分区，分区名字为broker-partition，所以hehe的所有分区为0-0，0-1，1-0，1-1，1-2，可以用于消费的stream总共有4个(此处不列出实际的consumerId，用consumer+数字表示):consumer0-0,consumer0-1,consumer1-0,consumer1-1。该算法最后分配的结果会是consumer0-0消费0-0和0-1，consumer0-1消费1-0，consumer1-0消费1-1，consumer1-1消费1-2。即将消费任务均分，具体的实现后续在介绍，毕竟这篇文章主要讲解consumer的使用，大家记住有balance这回事就好了。理解了这个balance的机制，相信大家对stream也不会再有困惑了吧，你完全可以把它理解为一个消费线程。

##消费者部分配置简要说明

|参数名|默认值|参数意义
|:-------:|:-------:|:-------:|
|groupid|  | 消费者分组信息|
|consumerid | |消费者唯一标识|
|socket.timeout.ms|30*1000|socket超时时间，默认30秒|
|socket.buffersize| 64*1024    |  socket接受区缓冲大小,B|
|Fetch.size   | 300 * 1024 |  一次请求抓取的消息大小，对于实时性要求高的此值要设置的小一点。|
|fetcher.backoff.ms | 1000 | 每次抓取之后如果没有抓取到增加抓取时间间隔的毫秒数|
|autocommit.enable | true  |  自动提交consumer的消费offset|
|autocommit.interval.ms |1000| 自动提交消费offset的间隔|
|autooffset.reset |smallest | smallest：从最小offset开始消费数据;largest：从最大offset开始消费，即只消费最新消息|
|consumer.timeout.ms| -1 |  -1指明consumer在没有新消息到来时阻塞，若设为正值，则在阻塞此时间后，还没有消息可消费，抛出异常。|
{: class="table table-striped table-bordered"}

##小结

相信大家看完上面的介绍，对consumer的使用也有一定了解了，那就赶紧动手试试吧！
另外欢迎大家留言讨论！！！
