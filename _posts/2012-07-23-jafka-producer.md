---
layout: post
title: "Producer使用示例与配置简要说明"
description: ""
category: 
tags: []
---
{% include JB/setup %}

本文主要讲解Jafka中Producer的详细使用，还会讲解其配置的用处，在看本文前，读者请确认已经对Jafka的使用有了初步的了解，如果还没有，狂点[这里](https://github.com/adyliu/jafka/wiki/quickstart.zh_CN)。另外，本文使用整合了zookeeper的Jafka。
简单来讲，生产者发送消息到服务端包含两个步骤：构造消息和发送消息。
先上段简单的代码：

{% highlight java linenos %}
Properties props = new Properties();
//指明获取发送地点的地址
props.setProperty("zk.connect","localhost:2181");
props.setProperty("serializer.class", StringEncoder.class.getName());
Producer<String,String> producer = new Producer<String, String>(new  ProducerConfig(props));
for(int i=0;i<1000;i++){
	//构造消息并发送
	producer.send(new StringProducerData("hehe","hehe-data"+i));
}
producer.close();
{% endhighlight %}

上面的代码完成的功能如下：
1.构造信息，使用StringProducerData类，该类继承了ProducerData类，该类内容如下：

	class ProducerData<K, V> {
            private String topic;
	    private K key;
	    private List<V> data;
	........
	}

ProducerData类由topic(消息主题名称)、key(消息分配到固定partition的依据，后面会有介绍)、data(数据)组成。用户完全可以继承该类实现自己的消息构造类。

2.发送消息。发送前首先要获取服务端(broker)的地址，该信息是由zookeeper获取的，所以在配置信息中要指明zookeeper的连接地址，然后调用send函数即可将消息发送出去。
如上就是Jafka中Producer的使用，简单明了。实际使用中推荐单例模式。producer是线程安全的，新建producer时会连接zookeeper来获取broker等信息，这个过程比较耗费时间的，使用单例只连接一次即可。

##Producer配置简要说明
Producer的所有配置见[这里](https://github.com/adyliu/jafka/wiki/configuration.zh_CN)，其中有一个配置为producer.type=sync|async，是指Producer有同步和异步两种发送形式，区别在于前者立即发送，而后者延迟发送。显而易见的是，异步发送对于使用producer的程序影响更小，不至于因为发送消息过慢而影响当前程序的响应。这两种发送方式在使用上的区别在于配置文件的使用，下面会对二者的使用进行说明，但我们首先来看下二者使用上相同的配置。


|参数名|默认值|参数意义
|:-------:|:-------:|:-------:|
|serializer.class|com.sohu.jafka.producer.serializer.DefaultEncoders|其实现了com.sohu.jafka.producer.serializer.Encoder/Decoder接口，serializer只要实现Encoder即可，常用的如StringEncoder。将传送的数据封装为Message，其实就是一个序列化的过程。|
|artitioner.class|com.sohu.jafka.producer.DefaultPartitioner|根据用户为消息指定的key来判断消息应该存储在哪一个分区(partition)中。|
{: class="table table-striped table-bordered"}

###Serializer.class

Serializer.class让用户自定义类，可以将要发送的数据封装为Message，只要用户实现com.sohu.jafka.producer.serializer.Encoder接口即可，该接口如下：

{% highlight java linenos %}
public interface Encoder<T> {
    Message toMessage(T event);
}
{% endhighlight %}

T即为消息的类型，Message的构造方法为new Message(byte[]),所以这个方法要做的就是将T对象序列化为byte数组。

###Partitioner.class

Partitioner.class是为发送的消息选择分区，这里首先说明下分区的概念。Jafka以分区(partition)的形式存储每个topic，分区的个数可以在服务端设定，或者后期通过脚本来修改，这些以后会涉及。分区的意义有两个：

* 一是使得consumer可以多线程消费消息，后面讲到consumer的时候有详细讲解；

* 二是用户可以限定同一topic下具有某些特性(key)的消息发送到同一个partition下面，这样consumer可以通过fetch指定partition来获取这一组数据（在consumer章节会详细讲解）。
用户自定义Partitioner，只要实现

{% highlight java linenos %}
public interface Partitioner<T> {
     int partition(T key, int numPartitions);
}
{% endhighlight %}

只要实现partition方法就可以，key是在ProducerData里面指定的如：

	StringProducerData data = new StringProducerData("topic","data");
	 //key如果不设置的话，不会调用partitioner
	 data.setKey("hehe");

key的类型也是可以任意指定的，ProducerData&lt;K, V&gt;中的K就是key的类型，而V是数据的类型。下面的代码简单地实现传送用户自定义bean对象的消息。



##用户自定义代码举例


{% highlight java linenos %}

//用户自定义类
public class Person implements Serializable{
    private String name;
    private int id;
    Person(String name, int id) {
        this.name = name;
        this.id = id;
    }
  // ……
}
//用户自定义bean的serializer类
public class PersonDataEncoder implements Encoder<Person> {
    @Override
    public Message toMessage(Person event) {
        try {
	    //序列化对象
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(baos);
            oos.writeObject(event);
            oos.close();
            byte[] tmpBytes = baos.toByteArray();
            baos.close();
            return new Message(tmpBytes);
        } catch (IOException e) {
            e.printStackTrace();  
        }
        return null;   
   }
}
//Partitioner类
public class PersonDataPartitioner implements Partitioner<String>{
    @Override
    public int partition(String key, int numPartitions) {
        return key.length()%numPartitions; 
    }
}
//构造producer发送消息
Properties props = new Properties();
props.setProperty("zk.connect","localhost:2181");
props.setProperty("serializer.class",PersonDataEncoder.class.getName());
//只有在zookeeper时，可以使用
props.setProperty("partitioner.class",PersonDataPartitioner.class.getName());
Producer<String,Person> producer = new Producer<String, Person>(new ProducerConfig(props));
for(int i=0;i<100000;i++){
PersonProducerData data = new PersonProducerData("person",new Person("name",i));
data.setKey("haha"+i);
    producer.send(data);
 }
 producer.close();
{% endhighlight %}



##同步发送配置

上面的producer代码就是同步发送的示例。同步发送的常用配置如下：

|参数名|默认值|参数意义|
|:-------:|:-------:|:-------:|
|buffer.size|  102400 |   socket通信缓冲区大小，byte|
|connect.timeout.ms |   5000  |  连接broker的超时时间，超过时报错|
|socket.timeout.ms  |  30000  |  socket连接超时时间|
|reconnect.interval |   30000 |   producer的请求数达到该数目后，重新建立到broker的连接。|
|max.message.size   | 1000000 |   producer发送一条消息的最大字节数|
{: class="table table-striped table-bordered"}

##异步发送配置

|参数名|默认值|参数意义|
|:-------:|:-------:|:-------:|
|queue.time |   5000  |  在该段时间内没有新消息到来的话，发送消息到broker|
|queue.size |   10000 |   指定异步发送队列的大小|
|batch.size |   200   | 指定异步发送每次批量发送的消息个数|
|queue.enqueueTimeout.ms  |  0   | 指定入队的方法，0表示队列未满时成功入队，否则丢弃。小于0表示当队列满时，阻塞直到成功入队，大于0表示等待这些ms都无法成功入队，舍弃|
{: class="table table-striped table-bordered"}


异步发送的代码与同步发送差异不大，只需添加一行配置：

>props.setProperty("producer.type","async");

异步发送的实现机制也并不复杂，它维护一个队列(queue)，该队列的长度可以配置(queue.size)，每一次调用producer的send方法，都是将消息封装一下，然后入队，另外会新开一个sendThread线程，不断地从这个queue中拿数据(poll)，当获取的数据达到batch.size时，就批量发送给broker。queue是一个BlockingQueue，具体实现在后续源码阅读章节会给大家详细讲解。

##小结

Producer的使用整体来讲还是很简单的，对于实时性要求较高的信息，采取同步发送的方法好，而对于像日志这种数据，可以采取异步发送的形式，减小对当前程序的压力。

