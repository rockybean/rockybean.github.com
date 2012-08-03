---
layout: post
title: "Jafka Broker源码阅读之SocketServer解析"
description: ""
category: 
tags: []
---
{% include JB/setup %}

本文将讲解Jafka SocketServer的实现，从前一篇总览的文章中，我们知道SocketServer的主要调用方法是startUp，那么我们就来研究下这个方法做了什么，不过在此之前我们还是先看下它的构造函数。

{% highlight java linenos %}
   public SocketServer(RequestHandlerFactory handlerFactory,ServerConfig serverConfig) {
        super();
        //broker配置信息
        this.serverConfig = serverConfig;
        //消息数据的处理类
        this.handlerFactory = handlerFactory;
	//每个请求包的最大值，对应max.socket.request.bytes配置
        this.maxRequestSize = serverConfig.getMaxSocketRequestSize();
        //worker线程组，负责处理具体的socket读写请求
        this.processors = new Processor[serverConfig.getNumThreads()];
        //broker信息监控类
        this.stats = new SocketServerStats(1000L * 1000L * 1000L * serverConfig.getMonitoringPeriodSecs());
        //acceptor处理连接请求类
        this.acceptor = new Acceptor(serverConfig.getPort(), //
                processors, //
                serverConfig.getSocketSendBuffer(), //
                serverConfig.getSocketReceiveBuffer());
   }
{% endhighlight %}

构造函数所做的事情已经在注释中有所说明，这里面都是一些初始化的操作，虽然没有实质性的线程启动等，但相信聪明读者已经注意到了acceptor processor handleractory这几个名词，并且从它们的名字上也基本可以大略地猜测其作用。

##SocketServer.startup

下面就让我们在startup方法中揭开这些对象的面纱。

{% highlight java linenos %}

   public void startup() throws InterruptedException {
        //每个processor可以处理的最大连接数
        final int maxCacheConnectionPerThread = serverConfig.getMaxConnections() / processors.length;
        logger.info("start " + processors.length + " Processor threads");
        //初始化并启动所有的Processor线程，其数目默认为cpu个数，可以通过num.threads来配置
        for (int i = 0; i < processors.length; i++) {
            processors[i] = new Processor(handlerFactory, stats, maxRequestSize, maxCacheConnectionPerThread);
            Utils.newThread("jafka-processor-" + i, processors[i], false).start();
        }
        //初始化并启动acceptor线程
        Utils.newThread("jafka-acceptor", acceptor, false).start();
        acceptor.awaitStartup();
    }

{% endhighlight %}

startup方法做的事情很简单，启动一个acceptor线程和多个processor线程，下图是Acceptor和Processor的类图。

![acceptor类图](/assets/images/threadClzDiagram.png)


##AbstractServerThread

我们先来看看Acceptor和Processor的父类AbstractServerThread，这个抽象类实现了Runnable和Closable接口，前者是实现run方法，传入线程执行，后者实现close方法，为了统一对象关闭调用。父类的意义在于抽取子类中有相同作用的代码，AbstractServerThread的实现也的确是这样的，源码如下：


{% highlight java linenos %}

public abstract class AbstractServerThread implements Runnable,Closeable {

    private Selector selector;
    //启动和停止的闭锁
    protected final CountDownLatch startupLatch = new CountDownLatch(1);
    protected final CountDownLatch shutdownLatch = new CountDownLatch(1);
    //线程状态布尔值
    protected final AtomicBoolean alive = new AtomicBoolean(false);
    
    final protected Logger logger = Logger.getLogger(getClass());
    /**
     * @return the selector
     */
    public Selector getSelector() {
        if (selector == null) {
            try {
                selector = Selector.open();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
        return selector;
    }
    
    protected void closeSelector() {
        Closer.closeQuietly(selector,logger);
    }

    //线程关闭方法
    public void close() {
        alive.set(false);
        //唤醒调用了selector.select()的方法
        selector.wakeup();
        try {
            //等待其他资源释放
            shutdownLatch.await();
        } catch (InterruptedException e) {
            logger.error(e.getMessage(),e);
        }
    }

    //调用该方法，表明线程已经完全启动
    protected void startupComplete() {
        alive.set(true);
        startupLatch.countDown();
    }

    //调用该放法，表明线程已经完全关闭
    protected void shutdownComplete() {
        shutdownLatch.countDown();
    }

    protected boolean isRunning() {
        return alive.get();
    }

    //阻塞直到线程完全启动，即调用了startupComplete方法
    public void awaitStartup() throws InterruptedException {
        startupLatch.await();
    }
}

{% endhighlight %}

上述代码中设计了java.nio和java.util.concurrent包中的类，关于这两个包中类的使用不是本文的重点，有疑惑的同学可以去搜索相关知识。AbstractServerThread类将线程启动关闭的方法和selector取出来，Selector是java nio中的一个类，未使用过的同学最好先去搜索下nio的相关知识，再继续往下看，否则会无法理解源码的意义。）

##Acceptor
我们来看下Acceptor的代码，从其名称上我们可以看到它主要负责accept工作，即处理socket连接请求，其run方法如下：

{% highlight java linenos %}
   public void run() {
        final ServerSocketChannel serverChannel;
        try {
            //启动socket服务器，并注册连接事件到selector上
            serverChannel = ServerSocketChannel.open();
            serverChannel.configureBlocking(false);
            serverChannel.socket().bind(new InetSocketAddress(port));
            serverChannel.register(getSelector(), SelectionKey.OP_ACCEPT);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        //

        logger.info("Awaiting connection on port "+port);
        startupComplete();
        //
        int currentProcessor = 0;
        //开始等待连接事件
        while(isRunning()) {
            int ready = -1;
            try {
                //阻塞至有连接请求或者500ms超时
                ready = getSelector().select(500L);
            } catch (IOException e) {
                throw new IllegalStateException(e);
            }
            if(ready<=0)continue;
            //遍历所有的连接请求
            Iterator<SelectionKey> iter = getSelector().selectedKeys().iterator();
            while(iter.hasNext() && isRunning())
                try {
                    SelectionKey key = iter.next();
                    iter.remove();
                    //
                    if(key.isAcceptable()) {
                        //处理连接请求，关键方法
                        accept(key,processors[currentProcessor]);
                    }else {
                        throw new IllegalStateException("Unrecognized key state for acceptor thread.");
                    }
                    //以round-robin形式选择processor
                    currentProcessor = (currentProcessor + 1) % processors.length;
                } catch (Throwable t) {
                    logger.error("Error in acceptor",t);
                }
            }
        //run over
        logger.info("Closing server socket and selector.");
        Closer.closeQuietly(serverChannel, logger);
        Closer.closeQuietly(getSelector(), logger);
        shutdownComplete();
   }

{% endhighlight %}
 
<pre>
 友情提醒：
 	如果上述代码看得您一头雾水，请先去补一下java nio的知识，笔者在第一次阅读该代码时也困惑了好一阵。
</pre>

run方法主要做了以下两件事情：

* 启动了一个Socket Server，绑定到port，然后等待连接事件的发生。
* 当连接事件发生时，以round-robin形式选择processor，调用ccept方法，将该连接传入processor处理。

因此，accept方法是处理的关键，代码如下：

{% highlight java linenos %}

private void accept(SelectionKey key, Processor processor) throws IOException{
        ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();
        serverSocketChannel.socket().setReceiveBufferSize(receiveBufferSize);
        //接受连接请求，获取socketChannel
        SocketChannel socketChannel = serverSocketChannel.accept();
        //配置channel为非阻塞方式
        socketChannel.configureBlocking(false);
        socketChannel.socket().setTcpNoDelay(true);
        socketChannel.socket().setSendBufferSize(sendBufferSize);
        //传入processor
        processor.accept(socketChannel);
    }

//processor.accept
public void accept(SocketChannel socketChannel) {
        //将该channel加入newConnections这个blockingQueue中，等待processor在新一轮循环中处理
        newConnections.add(socketChannel);
        //欢迎seletor，以便尽快处理新加入的channel
        getSelector().wakeup();
    }
{% endhighlight %}


accept方法执行逻辑也是很清晰的，首先接受连接请求，获得响应的channel，然后将processor会将该channel加入自己的队列中，等待处理。

Acceptor类的主要作用到这里就讲清楚了，下面我们看看Processor类的实现。

##Processor

Processor类负责实际的读写请求，所以其实现也稍显复杂，run方法如下：

{% highlight java linenos %}

public void run() {
 startupComplete();
 while (isRunning()) {
     try {
         //处理连接队列中新加入的channel，其实就是注册读事件
         configureNewConnections();
         //等待读写请求事件
         final Selector selector = getSelector();
         int ready = selector.select(500);
         if (ready <= 0) continue;
         Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
         while (iter.hasNext() && isRunning()) {
             SelectionKey key = null;
             try {
                 key = iter.next();
                 iter.remove();
                 if (key.isReadable()) {
                     //读请求
                     read(key);
                 } else if (key.isWritable()) {
                     //写请求
                     write(key);
                 } else if (!key.isValid()) {
                     close(key);
                 } else {
                     throw new IllegalStateException("Unrecognized key state for processor thread.");
                 }
             } catch (EOFException eofe) {
                 Socket socket = channelFor(key).socket();
                 logger.debug(format("connection closed by %s:%d.", socket.getInetAddress(), socket.getPort()));
                 close(key);
             } catch (InvalidRequestException ire) {
                 Socket socket = channelFor(key).socket();
                 logger.info(format("Closing socket connection to %s:%d due to invalid request: %s", socket.getInetAddress(), socket.getPort(),
                         ire.getMessage()));
                 close(key);
             } catch (Throwable t) {
                 Socket socket = channelFor(key).socket();
                 final String msg = "Closing socket for %s:%d becaulse of error";
                 if (logger.isDebugEnabled()) {
                     logger.error(format(msg, socket.getInetAddress(), socket.getPort()), t);
                 } else {
                     logger.error(format(msg, socket.getInetAddress(), socket.getPort()));
                 }
                 close(key);
             }
         }
     } catch (IOException e) {
         logger.error(e.getMessage(), e);
     }

 }
 //
 logger.info("Closing selector while shutting down");
 closeSelector();
 shutdownComplete();
}
{% endhighlight %}


Processor的循环处理体中，首先处理连接队列中的新请求，方法为configNewConnection，其源码在下方，可以看到，其所做的就是将在该processor的selector上注册该channel的read事件，之后processor等待读写请求并做出响应的操作。

这里有一个编程细节，希望大家可以注意，就是在上面processor的accept方法中，调用selector.wakeup方法，其作用便是唤醒selector.select(500)，使该线程立即执行，尽快处理新连入的channel。

{% highlight java linenos %}
private void configureNewConnections() throws ClosedChannelException {
  while (newConnections.size() > 0) {
      SocketChannel channel = newConnections.poll();
      if (logger.isDebugEnabled()) {
          logger.debug("Listening to new connection from " + channel.socket().getRemoteSocketAddress());
      }
      channel.register(getSelector(), SelectionKey.OP_READ);
  }
}
{% endhighlight %}

###read请求
下面我们看下read请求的处理方法。

{% highlight java linenos %}
 private void read(SelectionKey key) throws IOException {
        SocketChannel socketChannel = channelFor(key);
        Receive request = null;
        if (key.attachment() == null) {
            //第一次读取数据
            request = new BoundedByteBufferReceive(maxRequestSize);
            key.attach(request);
        } else {
            //多次数据时，直接由key的attachment中获取
            request = (Receive) key.attachment();
        }
        //从channel中读取数据
        int read = request.readFrom(socketChannel);
        stats.recordBytesRead(read);
        if (read < 0) {
            //没有消息数据
            close(key);
        } else if (request.complete()) {
            //成功读取消息数据，传入handle处理
            Send maybeResponse = handle(key, request);
            key.attach(null);
            //如果有返回数据，则注册write事件
            if (maybeResponse != null) {
                key.attach(maybeResponse);
                key.interestOps(SelectionKey.OP_WRITE);
            }
        } else {
            //传递数据多，要分多次读取，所以要再次注册read事件
            key.interestOps(SelectionKey.OP_READ);
            getSelector().wakeup();
            if (logger.isTraceEnabled()) {
                logger.trace("reading request not been done. " + request);
            }
        }
    }

private Send handle(SelectionKey key, Receive request) {
        final short requestTypeId = request.buffer().getShort();
        //获取request类型
        final RequestKeys requestType = RequestKeys.valueOf(requestTypeId);
        //获取对应种类的RequestHandler
        RequestHandler handlerMapping = requesthandlerFactory.mapping(requestType, request);
        if (handlerMapping == null) {
            throw new InvalidRequestException("No handler found for request");
        }
        long start = System.nanoTime();
        //调用handler方法，返回处理结果
        Send maybeSend = handlerMapping.handler(requestType, request);
        stats.recordRequest(requestType, System.nanoTime() - start);
        return maybeSend;
    }
{% endhighlight %}

代码逻辑请参照注释，简单说来就是先获取channel，然后尝试从channel中读取数据，如果没有获取数据，直接close；如果获取的数据不完整需要多次读取，就再注册read事件；如果已经获取所有数据了，那么传入handle方法执行相应的RequestHandler方法，如果返回值不为空，则注册写事件，将结果返回客户端。

> `提示`：Request Send以及RequestHandler会另外写文章分析，本文主要讲解SocketServer的处理逻辑，不再展开讲解。
	

###write请求
下面我们看write请求的处理方法。

{% highlight java linenos %}
private void write(SelectionKey key) throws IOException {
  Send response = (Send) key.attachment();
  SocketChannel socketChannel = channelFor(key);
  //将response写到channel中
  int written = response.writeTo(socketChannel);
  stats.recordBytesWritten(written);
  if (response.complete()) {
      key.attach(null);
      key.interestOps(SelectionKey.OP_READ);
  } else {
      key.interestOps(SelectionKey.OP_WRITE);
      getSelector().wakeup();
  }

{% endhighlight %}

写处理和读处理是类似的，一次写不完的就再注册写事件等待下一次写。


##小结

通过上面的分析，相信大家脑海中已经对SocketServer有了大概的了解，它就是由一个acceptor和多个processor组成，前者负责处理连接请求，后者负责处理读写请求。这个实现简单灵活，processor的数目是可调节的，其性能也有相应的[测试](http://sna-projects.com/blog/2009/08/introducing-the-nio-socketserver-implementation/)。
总的来说，SocketServer使用nio实现了一个高性能的socket服务器，感兴趣的同学可以去关注下目前NIO方面的一些框架[mina](http://mina.apache.org/)、[netty](https://netty.io/)、taobao开源的[gecko](http://code.taobao.org/p/gecko/src/)等等，借助这些框架可以更快更好地实现高性能的socket服务器。虽然jafka在服务端是自己实现了socket服务端，但nio编程中有[许多陷阱和需要注意](http://www.blogjava.net/killme2008/archive/2010/11/22/338420.html)的地方，即便是一些老手都经常在这上面再跟头，所以还是建议大家尽量使用框架。Nio编程中有许多细节要注意，比如关闭的操作，本文列出的代码中有响应关闭的操作，读者可以好好地体会下Jafka是如何在捕获各种异常的情况下来合理关闭资源的。
