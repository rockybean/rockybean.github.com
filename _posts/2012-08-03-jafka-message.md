---
layout: post
title: "Jafka 源码阅读之通信协议及相关类解析"
description: ""
category: 
tags: []
---
{% include JB/setup %}

本文讲解Jafka的通信协议，其实就是传输数据的约定格式。

Jafka的通信发生在producer和broker、consumer和broker之间，目前producer和broker之间通信是单向的(producer->broker),consumer和broker之间通信是双向的(consumer<->broker)。主要涉及到的类有Message MessageSet Request Send，下面分别介绍这些类。

##Message(com.sohu.jafka.message)
Message类是具体的消息数据，在传递的过程中，其字节序列的组成及含义如下:

|字节大小(Byte)|含义|
|:-------:|:-------:|
|4|length,该条消息总长度|
|1|version(magic byte),消息版本，适应后面更改通信协议，目前只有一种协议|
|1|attribute,目前主要用于指明压缩算法：0--NoCompression 1--GzipCompression|
|4|crc32，消息完整性校验|
|x|实际数据，x=length-1-1-4|
{: class="table table-striped table-bordered"}

Message类即对这个字节序列作了封装，其主要的属性和方法如下：

{% highlight java linenos %}
//消息总长度length
private final int messageSize;
public int getSizeInBytes() {
        return messageSize;
}


//存储字节序列length之后的数据
final ByteBuffer buffer;

//自定义version类型，目前只有这一个
private static final byte MAGIC_VERSION2 = 1;
//当前version值
public static final byte CurrentMagicValue = 1;
//version值的偏移和长度
public static final byte MAGIC_OFFSET = 0;
public static final byte MAGIC_LENGTH = 1;
public byte magic() {
        return buffer.get(MAGIC_OFFSET);
}


//属性值的偏移和长度
public static final byte ATTRIBUTE_OFFSET = MAGIC_OFFSET + MAGIC_LENGTH;
public static final byte ATTRIBUT_ELENGTH = 1;
public byte attributes() {
        return buffer.get(ATTRIBUTE_OFFSET);
}


//属性中只用最后两位指明压缩算法
public static final int CompressionCodeMask = 0x03; //
//0表明不使用压缩算法
public static final int NoCompression = 0;

//crc长度
public static final byte CrcLength = 4;

//计算crc的偏移
public static int crcOffset(byte magic) {
        switch (magic) {
            case MAGIC_VERSION2:
                return ATTRIBUTE_OFFSET + ATTRIBUT_ELENGTH;
        }
        throw new UnknownMagicByteException(format("Magic byte value of %d is unknown", magic));
    }


//消息数据的偏移
public static int payloadOffset(byte magic) {
        return crcOffset(magic) + CrcLength;
}
//获取实际的消息数据
public ByteBuffer payload() {
        ByteBuffer payload = buffer.duplicate();
        payload.position(headerSize(magic()));
        payload = payload.slice();
        payload.limit(payloadSize());
        payload.rewind();
        return payload;
    }
{% endhighlight %}

从上面的注释中我们可以看到，Message类就是对定义的字节序列格式进行了一个封装，对外提供了方便的调用函数，其主要的一个构造函数如下：

{% highlight java linenos %}
//这里的bytes为实际的消息数据，该构造函数会根据传入的参数，自动生成消息的header数据
public Message(long checksum, byte[] bytes, CompressionCodec compressionCodec) {
	//初始化buffer的大小=headerSize+messageLength
        this(ByteBuffer.allocate(Message.headerSize(Message.CurrentMagicValue) + bytes.length));
        //存储version,1Byte
        buffer.put(CurrentMagicValue);
        byte attributes = 0;
        if (compressionCodec.codec > 0) {
            attributes = (byte) (attributes | (CompressionCodeMask & compressionCodec.codec));
        }
        //存储attribute，1Byte
        buffer.put(attributes);
        //存储crc32
        Utils.putUnsignedInt(buffer, checksum);
        //存储消息数据
        buffer.put(bytes);
        //定位buffer到头部，以备buffer被读取使用
        buffer.rewind();
    }

{% endhighlight %}



##MessageSet(com.sohu.jafka.message)

从MessageSet的类名中，大家也可以猜到这个类是多个Message的集合，它有两个子类：ByteBufferMessageSet和FileMessageSet，前者供producer和consumer使用，后者供broker使用。首先我们来看下MessageSet的源码。

{% highlight java linenos %}
public abstract class MessageSet implements Iterable<MessageAndOffset> {

    public static final MessageSet Empty = new ByteBufferMessageSet(ByteBuffer.allocate(0));
    //代表消息长度占用字节数：4个
    public static final int LogOverhead = 4;

	//将多条message封装到一个ByteBuffer中
    public static ByteBuffer createByteBuffer(CompressionCodec compressionCodec, Message... messages) {
        if (compressionCodec == CompressionCodec.NoCompressionCodec) {
            ByteBuffer buffer = ByteBuffer.allocate(messageSetSize(messages));
            for (Message message : messages) {
                message.serializeTo(buffer);
            }
            buffer.rewind();
            return buffer;
        }
        //
        if (messages.length == 0) {
            ByteBuffer buffer = ByteBuffer.allocate(messageSetSize(messages));
            buffer.rewind();
            return buffer;
        }
        //
        Message message = CompressionUtils.compress(messages, compressionCodec);
        ByteBuffer buffer = ByteBuffer.allocate(message.serializedSize());
        message.serializeTo(buffer);
        buffer.rewind();
        return buffer;
    }

	//每条消息的长度，就是上面讲到Message的字节序列，包含length
    public static int entrySize(Message message) {
        return LogOverhead + message.getSizeInBytes();
    }

    public static int messageSetSize(Iterable<Message> messages) {
        int size = 0;
        for (Message message : messages) {
            size += entrySize(message);
        }
        return size;
    }

    public static int messageSetSize(Message... messages) {
        int size = 0;
        for (Message message : messages) {
            size += entrySize(message);
        }
        return size;
    }

    public abstract long getSizeInBytes();

    public void validate() {
        for (MessageAndOffset messageAndOffset : this)
            if (!messageAndOffset.message.isValid()) {
                throw new InvalidMessageException();
            }
    }
	//将消息数据写入channel中
    public abstract long writeTo(GatheringByteChannel channel, long offset, long maxSize) throws IOException;
}
{% endhighlight %}

由上面的代码可以知道，MessageSet封装产生的ByteBuffer就是多个Message首尾相连构造而成。这里要注意一点：这些Message并不一定是单条消息数据，还可能是多条消息数据经过压缩后组成的一条Message，将该Message解压后得到的其实也是一个MessageSet。这一点大家在阅读ByteBufferMessageAndOffset遍历部分代码的时候会看到。
该类可以看作是message的工具类，负责消息数据的批量读和写。另外此类实现了Iterable<MessageAndOffset>，可以遍历MessageAndOffset对象，该对象封装了message数据和下一条message的offset信息。在[consumer](/2012/07/24/jafka-consumer/)里，我们可以使用如下代码遍历获取的消息。

{% highlight java linenos %}
//获取消息数据集合
ByteBufferMessageSet messageSet = consumer.fetch(fetch);
//遍历消息数据集合
for(MessageAndOffset messageAndOffset:messageSet){
...
}
{% endhighlight %}
MessageSet的两个子类从名字上可以看出它们封装消息的来源：一个来自ByteBuffer,一个来自File(即jafka文件)。下面分别介绍下这两个类。

###ByteBufferMessageSet
该类主要被Producer和Consumer使用，前者将要传送的Message封装成ByteBufferMessageSet,然后传送到broker；后者将fetch到的Message封装成ByteBufferMessageSet，然后遍历消费。那么我们先来看下其提供的封装入口：

{% highlight java linenos %}
    public ByteBufferMessageSet(ByteBuffer buffer) {
        this(buffer,0L,ErrorMapping.NoError);
    }
    public ByteBufferMessageSet(ByteBuffer buffer,long initialOffset,ErrorMapping errorCode) {
        this.buffer = buffer;
        this.initialOffset = initialOffset;
        this.errorCode = errorCode;
        this.validBytes = shallowValidBytes();
    }

    public ByteBufferMessageSet(CompressionCodec compressionCodec,Message...messages) {
        this(MessageSet.createByteBuffer(compressionCodec, messages),0L,ErrorMapping.NoError);
    }
    public ByteBufferMessageSet(Message...messages) {
        this(CompressionCodec.NoCompressionCodec,messages);
    }
{% endhighlight %}

由传入参数可知，前两个构造函数为consumer使用，后两个为producer使用。在讲解producer的使用时，有讲到一个配置参数serializer.class，它的作用是将ProducerData<K,V>中的K类对象转化为Message，也就是这里构造函数的传入参数Message。

ByteBufferMessageSet的一个重要接口是遍历消息数据，即其iterator()方法，其实现这里不详细讲了，原理简单和大家说一下。前面提到过一个Message可能是多条消息数据缩后构成的，所以在遍历的时候便存在一个是否要遍历压缩的Message中每条消息数据的问题，其由isShallow参数决定：true不遍历，false遍历。ByteBufferMessageSet的iterator方法是调用的是`return internalIterator(false);`,是会遍历包括压缩Message中的所有消息数据的。实现方式是通过topIter遍历一级Message，当遇到压缩的Message时，将其解压缩并且用innerIter记录其遍历情况，当遍历结束后，回到topIter继续遍历。

ByteBufferMessageSet的writeTo(Channel)的方法代码如下，将数据写入指定的channel。

{% highlight java linenos %}
public long writeTo(GatheringByteChannel channel, long offset, long maxSize) throws IOException {
        buffer.mark();
        int written = channel.write(buffer);
        buffer.reset();
        return written;
    }
{% endhighlight %}


###FileMessageSet

该类主要由broker使用，我们来看下它的构造函数：

{% highlight java linenos %}
public FileMessageSet(FileChannel channel, long offset, long limit, //
            boolean mutable, AtomicBoolean needRecover) throws IOException {
        super();
        this.channel = channel;
        this.offset = offset;
        this.mutable = mutable;
        this.needRecover = needRecover;
        if (mutable) {
            if (limit < Long.MAX_VALUE || offset > 0) throw new IllegalArgumentException(
                    "Attempt to open a mutable message set with a view or offset, which is not allowed.");

            if (needRecover.get()) {
                // set the file position to the end of the file for appending messages
                long startMs = System.currentTimeMillis();
                long truncated = recover();
                logger.info("Recovery succeeded in " + (System.currentTimeMillis() - startMs) / 1000 + " seconds. " + truncated + " bytes truncated.");
            } else {
                setSize.set(channel.size());
                setHighWaterMark.set(getSizeInBytes());
                channel.position(channel.size());
            }
        } else {
            setSize.set(Math.min(channel.size(), limit) - offset);
            setHighWaterMark.set(getSizeInBytes());
        }
    }

    public FileMessageSet(FileChannel channel, boolean mutable) throws IOException {
        this(channel, 0, Long.MAX_VALUE, mutable, new AtomicBoolean(false));
    }

    public FileMessageSet(File file, boolean mutable) throws IOException {
        this(Utils.openChannel(file, mutable), mutable);
    }

    public FileMessageSet(FileChannel channel, boolean mutable, AtomicBoolean needRecover) throws IOException {
        this(channel, 0, Long.MAX_VALUE, mutable, needRecover);
    }

    public FileMessageSet(File file, boolean mutable, AtomicBoolean needRecover) throws IOException {
        this(Utils.openChannel(file, mutable), mutable, needRecover);
    }
{% endhighlight %}

由第一个构造函数可知，构造一个FileMessageSet需要以下几点：

* FileChannel,即打开一个文件，并且指明是否是mutable（可写）。
* offset,读入文件的起始位置
* limit，读入文件的大小
* mutable,是否可写

fileChannel打开的文件即为jafka文件，其中存储着message，存储的格式与MessageSet是相同的，也是message首尾相连存储。FileMessageSet的遍历比较简单，顺序从channel中读取出来组装成MessageAndOffset即可，这里没有考虑Message是否压缩，原因应该是没有使用的需求。代码就不贴了，大家可以自己去阅读iterator方法。

FileMessageSet的writeTo方法要特别强调一下，之前我们有提到jafka使用了sendfile这个高级系统调用，大大提升了传输效率，对应代码就在这里。
{% highlight java linenos %}
public long writeTo(GatheringByteChannel destChannel, long writeOffset, long maxSize) throws IOException {
        return channel.transferTo(offset + writeOffset, Math.min(maxSize, getSizeInBytes()), destChannel);
    }
{% endhighlight %}

FileMessageSet还有两个方法与消息数据的持久化有关。

{% highlight java linenos %}
//将messages添加如当前的messageOffset
 public long[] append(MessageSet messages) throws IOException {
        checkMutable();
        long written = 0L;
        while (written < messages.getSizeInBytes())
            written += messages.writeTo(channel, 0, messages.getSizeInBytes());
        long beforeOffset = setSize.getAndAdd(written);
        return new long[] {  written,beforeOffset };
    }

//flush消息到磁盘
    public void flush() throws IOException {
        checkMutable();
        long startTime = System.currentTimeMillis();
        channel.force(true);
        long elapsedTime = System.currentTimeMillis() - startTime;
        LogFlushStats.recordFlushRequest(elapsedTime);
        logger.debug("flush time " + elapsedTime);
        setHighWaterMark.set(getSizeInBytes());
        logger.debug("flush high water mark:" + highWaterMark());
    }
{% endhighlight %}

第一个函数的作用便是将producer传递过来的messages添加到当前messageset对象(channel)中，虽然调用了writeTo方法，但是由于操作系统缓冲的存在，数据可能还没有真正写入磁盘，而flush方法的作用便是强制写磁盘。这两个方法便完成了消息数据持久化到磁盘的操作。
另外FileMessageSet还提供了read方法，可以读取指定offset到offset+limit的所有消息，这里就不贴代码了。


##Request

Request是producer consumer与broker通信的顶级封装类，即最终发送与接受的都是该类，下图是其简单的类图：
![request_clz](/assets/images/request_clz.png)

Request的源码为：

{% highlight java linenos %}
public interface Request extends ICalculable {

	//请求类型，enum
    RequestKeys getRequestKey();
	//将该request写入buffer，从这里可以看到request的协议格式
    void writeTo(ByteBuffer buffer);
}
//request 的所有类型
public enum RequestKeys {
    PRODUCE, //0
    FETCH, //1
    MULTIFETCH, //2
    MULTIPRODUCE, //3
    OFFSETS,//4
    CREATE,//5
    DELETE;//6

    public int value = ordinal();

    final static int size = values().length;

    public static RequestKeys valueOf(int ordinal) {
        if (ordinal < 0 || ordinal >= size) return null;
        return values()[ordinal];
    }
}
{% endhighlight %}

从上面可以看到RequestKeys的类型与类图中Request的子类是一一对应的，Request在传送时都有自己的协议格式，这里以ProducerRequest举例，其协议格式可以在其writeTo方法中获取，另外发送时，ProducerRequest会被封装在一个BoundedByteBufferSend类中，该类会在字节序列中添加消息总长度和request类型这两个基本信息，最终producerRequest的协议格式如下：

|字节数(Byte)|含义|
|:----------:|:--:|
|4|消息长度,length|
|2|Request类型，对应RequestKeys|
|2|topic length|
|x|topic|
|4|partition|
|4|messageset length|
|x|messageset|
{: class="table table-striped table-bordered"}

当request传递到broker时，在上一篇文章中我们分析processor源码时，曾经提到过的`Send handle(SelectionKey key, Receive request)`方法，其中有以下代码：

> final short requestTypeId = request.buffer().getShort(); 

> final RequestKeys requestType = RequestKeys.valueOf(requestTypeId);

其中request封装了客户端传递来的request字节序列，此处先读取2个字节，获取request的类型，然后选取对应的handler来处理，感兴趣的读者可以自行去查看相应的代码。

其他Request的协议类型，大家可以自行去查看其writeTo方法，这里就不逐个列举了。




