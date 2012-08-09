---
layout: post
title: "Jafka Broker源码阅读之LogManager"
description: ""
category: 
tags: []
---
{% include JB/setup %}

本文主要讲解Jafka Broker中LogManager类，介绍Jafka是如何组织其数据文件，来实现在O(1)时间复杂度内完成消息写和快速读取数据的。

##Logmanager启动代码

在前面讲解Jafka [broker](/2012/08/01/jafka-src/)代码中，涉及Logmanager的代码如下：

{% highlight java linenos %}

 //初始化消息数据管理类LogManager，并将所有的消息数据按照一定格式读入内存（非数据内容本身）
this.logManager = new LogManager(config,//
        scheduler,//
        1000L * 60 * config.getLogCleanupIntervalMinutes(),//
        1000L * 60 * 60 * config.getLogRetentionHours(),//
        needRecovery);
this.logManager.setRollingStategy(config.getRollingStrategy());
logManager.load();

...

//如果开启了zookeeper连接，则将该broker信息注册到zookeeper中，并开启定时flush消息数据的线程
logManager.startup();

{% endhighlight %}

上面代码主要涉及了LogManager的三个函数：构造函数、load函数和startup函数，下面我们一起来看下这几个函数。

##LogManager的构造函数

{% highlight java linenos %}
public class LogManager implements PartitionChooser, Closeable {

    final ServerConfig config;
	//清理数据文件的定时器
    private final Scheduler scheduler;

    final long logCleanupIntervalMs;

    final long logCleanupDefaultAgeMs;

    final boolean needRecovery;


    final int numPartitions;

    final File logDir;

    final int flushInterval;

    private final Object logCreationLock = new Object();

    final Random random = new Random();

    final CountDownLatch startupLatch;

	//以<topic,<partitionNum,Log>>的形式存储所有的jafka数据文件
    private final Pool<String, Pool<Integer, Log>> logs = new Pool<String, Pool<Integer, Log>>();

	//flush jafka文件的定时器
    private final Scheduler logFlusherScheduler = new Scheduler(1, "jafka-logflusher-", false);
	
    private final LinkedBlockingQueue<TopicTask> topicRegisterTasks = new LinkedBlockingQueue<TopicTask>();

    private volatile boolean stopTopicRegisterTasks = false;

    final Map<String, Integer> logFlushIntervalMap;

    final Map<String, Long> logRetentionMSMap;

    final int logRetentionSize;
	//负责见该broker注册到zookeeper的对象
    private ServerRegister serverRegister;

	//<topic,partitionTotalNumber>的配置信息
    private final Map<String, Integer> topicPartitionsMap;

    private RollingStrategy rollingStategy;

public LogManager(ServerConfig config,Scheduler scheduler,long logCleanupIntervalMs,long logCleanupDefaultAgeMs,boolean needRecovery) {
        super();
        //传入配置参数
        this.config = config;
        //传入执行日志清除工作的定时器
        this.scheduler = scheduler;
        //        this.time = time;
        //各种参数配置
        this.logCleanupIntervalMs = logCleanupIntervalMs;
        this.logCleanupDefaultAgeMs = logCleanupDefaultAgeMs;
        this.needRecovery = needRecovery;
        //
        this.logDir = Utils.getCanonicalFile(new File(config.getLogDir()));
        this.numPartitions = config.getNumPartitions();
        this.flushInterval = config.getFlushInterval();
        this.topicPartitionsMap = config.getTopicPartitionsMap();
        this.startupLatch = config.getEnableZookeeper() ? new CountDownLatch(1) : null;
        this.logFlushIntervalMap = config.getFlushIntervalMap();
        this.logRetentionSize = config.getLogRetentionSize();
        this.logRetentionMSMap = getLogRetentionMSMap(config.getLogRetentionHoursMap());
        //
    }

}
{% endhighlight %}

LogManager的成员变量中logs负责组织所有的jafka文件，组织方式也简单，map的数据结构，最终形成`<topic,partition>`对应一个`Log`对象的形式，该Log对象其实是一批jafka文件。构造函数主要工作便是初始化配置参数，参数意义可以参见之前讲解broker使用的[文章](/2012/07/26/jafka-broker/)。


##LogManager.load

{% highlight java linenos %}

public void load() throws IOException {
        if (this.rollingStategy == null) {
            this.rollingStategy = new FixedSizeRollingStrategy(config.getLogFileSize());
        }

        //检查log.dir配置的文件夹是否存在，不存在的话创建
        if (!logDir.exists()) {
            logger.info("No log directory found, creating '" + logDir.getAbsolutePath() + "'");
            logDir.mkdirs();
        }
        if (!logDir.isDirectory() || !logDir.canRead()) {
            throw new IllegalArgumentException(logDir.getAbsolutePath() + " is not a readable log directory.");
        }
        File[] subDirs = logDir.listFiles();
        //遍历其下的子文件夹，命名方式为topic-partition
        if (subDirs != null) {
            for (File dir : subDirs) {
                if (!dir.isDirectory()) {
                    logger.warn("Skipping unexplainable file '" + dir.getAbsolutePath() + "'--should it be there?");
                } else {
                    logger.info("Loading log from " + dir.getAbsolutePath());
                    final String topicNameAndPartition = dir.getName();
                    //检测是否符合topic-partition的格式
                    if(-1 == topicNameAndPartition.indexOf('-')) {
                        throw new IllegalArgumentException("error topic directory: "+dir.getAbsolutePath());
                    }
                    //从文件夹名称中获取topic和partition数
                    final KV<String, Integer> topicPartion = Utils.getTopicPartition(topicNameAndPartition);
                    final String topic = topicPartion.k;
                    final int partition = topicPartion.v;
                    //新建一个Log对象
                    Log log = new Log(dir, partition, this.rollingStategy, flushInterval, needRecovery);

                    //将该topic-partition文件夹对应的log对象放入到logs中，建立topic-partition的映射关系
                    logs.putIfNotExists(topic, new Pool<Integer, Log>());
                    Pool<Integer, Log> parts = logs.get(topic);
                    
                    parts.put(partition, log);
                    int configPartition = getPartition(topic);
                    if(configPartition < partition) {
                        topicPartitionsMap.put(topic, partition);
                    }
                }
            }
        }

        /* Schedule the cleanup task to delete old logs */
        //启动定时清除日志的线程
        if (this.scheduler != null) {
            logger.info("starting log cleaner every " + logCleanupIntervalMs + " ms");
            this.scheduler.scheduleWithRate(new Runnable() {

                public void run() {
                    try {
                        cleanupLogs();
                    } catch (IOException e) {
                        logger.error("cleanup log failed.", e);
                    }
                }

            }, 60 * 1000, logCleanupIntervalMs);
        }
        //
        if (config.getEnableZookeeper()) {
            //将该broker信息注册到zookeeper上
            this.serverRegister = new ServerRegister(config, this);
            //建立到zookeeper的连接
            serverRegister.startup();
            //启动一个注册topic的线程，以阻塞方式从topicRegisterTasks中获取，当有新的topic注册时，立即向zk中注册
            TopicRegisterTask task = new TopicRegisterTask();
            task.setName("jafka.topicregister");
            task.setDaemon(true);
            task.start();
        }
    }

{% endhighlight %}

load函数代码意义见注释，其主要完成的工作便是遍历`log.dir`下的所有文件夹，这些文件夹名按照topic-paritition命名，这些文件夹下是jafka文件。load依据topic parition建立一个Log对象，该对象中含有了其下所有jafka文件的句柄。然后将该log文件与其topic partition建立起映射关系，存入logs变量中。之后启动了定时清除日志的线程和注册topic到zk的线程。

按照<topic,<partition,Log>>的形式组织jafka数据文件的原因是显而易见的，因为producer consumer的请求都是按照topic partition来做的。关于Log类，我们来简单看下下面这幅图。

![LogManager](/assets/images/logmanager.png)

由上到下是依次包含关系，我们从下面向上看。最底层的是FileMessageSet，这个类在前一篇通信协议的文章中已经有了详细的讲解，我们直到它通过FileChannel打开了jafka文件，可以对其进行读写操作，是最底层的类。接下来我们看其上的LogSegment类，其部分源码如下：

{% highlight java linenos %}
public class LogSegment implements Range, Comparable<LogSegment> {
	//对应的jafka文件
    private final File file;

    private final FileMessageSet messageSet;
	//通过jafka文件名获取的该文件起始偏移量
    private final long start;
	//标记该文件是否可删除
    private volatile boolean deleted;

    public LogSegment(File file, FileMessageSet messageSet, long start) {
        super();
        this.file = file;
        this.messageSet = messageSet;
        this.start = start;
        this.deleted = false;
    }

	//value是传入的offset值，该方法可以判断本jafka文件是否包含该value
    public boolean contains(long value) {
        long size = size();
        long start = start();
        return ((size == 0 && value == start) //
        || (size > 0 && value >= start && value <= start + size - 1));
    }
    }

{% endhighlight %}

由源码可知，LogSegment是一个简单的封装类，包含FileMessageSet和一些判断信息。接下来是SegmentList，从名字上就可知它包含一个LogSegment的列表，其关键源码如下：

{% highlight java linenos %}

public class SegmentList {

	//topic-partition文件夹下的所有jafka文件
    private final AtomicReference<List<LogSegment>> contents;
	
    private final String name;

    /**
     * create the messages segments
     * 
     * @param name the message topic name
     * @param segments exist segments
     */
    public SegmentList(final String name, List<LogSegment> segments) {
        this.name = name;
        contents = new AtomicReference<List<LogSegment>>(segments);
    }

     //添加一个新的jafka文件到list后面，注意此处使用了类似CopyOnWrite的方法来避免读写冲突
    public void append(LogSegment segment) {
        while (true) {
            List<LogSegment> curr = contents.get();
            List<LogSegment> updated = new ArrayList<LogSegment>(curr);
            updated.add(segment);
            if (contents.compareAndSet(curr, updated)) {
                return;
            }
        }
    }

    //截取某个offset之前的所有LogSegment，供删除用
    public List<LogSegment> trunc(int newStart) {
        if (newStart < 0) {
            throw new IllegalArgumentException("Starting index must be positive.");
        }
        while (true) {
            List<LogSegment> curr = contents.get();
            int newLength = Math.max(curr.size() - newStart, 0);
            List<LogSegment> updatedList = new ArrayList<LogSegment>(curr.subList(Math.min(newStart, curr.size() - 1),
                    curr.size()));
            if (contents.compareAndSet(curr, updatedList)) {
                return curr.subList(0, curr.size() - newLength);
            }
        }
    }

	//获取最后一个LogSegment，该segment是可写的
    public LogSegment getLastView() {
        List<LogSegment> views = getView();
        return views.get(views.size() - 1);
    }

}

{% endhighlight %}

上述代码中使用了AtomicReference+while(true)的形式采用CopyOnWrite的思想来实现线程安全，这是非常值得学习的地方。SegmentList包含了一个LogSegment的链表，并且提供了add get trunc等操作方法。好了，终于到了Log类了，这是一个很重要的类，首先来看下它的构造函数。

{% highlight java linenos %}
public Log(File dir, //
            int partition,//
            RollingStrategy rollingStategy,//
            int flushInterval, //
            boolean needRecovery) throws IOException {
        super();
        //一堆配置
        this.dir = dir;
        this.partition = partition;
        this.rollingStategy = rollingStategy;
        this.flushInterval = flushInterval;
        this.needRecovery = needRecovery;
        this.name = dir.getName();
        this.logStats.setMbeanName("jafka:type=jafka.logs." + name);
        Utils.registerMBean(logStats);
        //载入所有的jafka文件
        segments = loadSegments();
    }

private SegmentList loadSegments() throws IOException {
        List<LogSegment> accum = new ArrayList<LogSegment>();
        File[] ls = dir.listFiles(new FileFilter() {

            public boolean accept(File f) {
                return f.isFile() && f.getName().endsWith(FileSuffix);
            }
        });
        logger.info("loadSegments files from [" + dir.getAbsolutePath() + "]: " + ls.length);
        int n = 0;
        //遍历该文件夹下的所有jafka文件
        for (File f : ls) {
            n++;
            String filename = f.getName();
            //获取起始的offset值
            long start = Long.parseLong(filename.substring(0, filename.length() - FileSuffix.length()));
            final String logFormat = "LOADING_LOG_FILE[%2d], start(offset)=%d, size=%d, path=%s";
            logger.info(String.format(logFormat, n, start, f.length(), f.getAbsolutePath()));
            //建立FileMessageSet对象，即打开了文件,false代表以只读形式打开
            FileMessageSet messageSet = new FileMessageSet(f, false);
            accum.add(new LogSegment(f, messageSet, start));
        }
        if (accum.size() == 0) {
            //如果没有jafka文件，则以可读写形式新建一个文件
            File newFile = new File(dir, Log.nameFromOffset(0));
            FileMessageSet fileMessageSet = new FileMessageSet(newFile, true);
            accum.add(new LogSegment(newFile, fileMessageSet, 0));
        } else {
            //将accum中的jafka文件按照start值排序
            Collections.sort(accum);
            //检测日志数据完整性
            validateSegments(accum);
        }
        //以读写方式打开最后一个文件，以供新消息数据写
        LogSegment last = accum.remove(accum.size() - 1);
        last.getMessageSet().close();
        logger.info("Loading the last segment " + last.getFile().getAbsolutePath() + " in mutable mode, recovery " + needRecovery);
        LogSegment mutable = new LogSegment(last.getFile(), new FileMessageSet(last.getFile(), true, new AtomicBoolean(
                needRecovery)), last.start());
        accum.add(mutable);
        return new SegmentList(name, accum);
    }
{% endhighlight %}

Log的初始化函数完成系列配置后，调用loadSegments方法载入所有的jafka文件，并将文件按照其offset值由大到小排序，这样做的目的是为了通过二分查找可以快速定位某offset所在的文件。另外最后一个jafka文件要以读写方式打开，其他文件以只读方式打开，从而做到了顺序读写，也就可以在O(1)的时间复杂度内完成消息数据写操作。Log类提供了读写消息的方法，读方法如下：

{% highlight java linenos %}
//读取自offset始，最长length的所有消息
public MessageSet read(long offset, int length) throws IOException {
        List<LogSegment> views = segments.getView();
        //二分查找符合条件的log文件爱你
        LogSegment found = findRange(views, offset, views.size());
        if (found == null) {
            if (logger.isTraceEnabled()) {
                logger.trace(format("NOT FOUND MessageSet from Log[%s], offset=%d, length=%d", name, offset, length));
            }
            return MessageSet.Empty;
        }
        //调用FileMessageSet的read方法，读取消息数据
        return found.getMessageSet().read(offset - found.start(), length);
    }

public static <T extends Range> T findRange(List<T> ranges, long value, int arraySize) {
        if (ranges.size() < 1) return null;
        T first = ranges.get(0);
        T last = ranges.get(arraySize - 1);
        // check out of bounds
        if (value < first.start() || value > last.start() + last.size()) {
            throw new OffsetOutOfRangeException("offset " + value + " is out of range");
        }

        // check at the end
        if (value == last.start() + last.size()) return null;

	//二分查找的代码
        int low = 0;
        int high = arraySize - 1;
        while (low <= high) {
            int mid = (high + low) / 2;
            T found = ranges.get(mid);
            
            if (found.contains(value)) {
                return found;
            } else if (value < found.start()) {
                high = mid - 1;
            } else {
                low = mid + 1;
            }
        }
        return null;
    }


{% endhighlight %}

上面便是读消息的相关代码，相信读者结合注释很容易便能读懂。二分查找是一个亮点，另外在FileMessageSet的read方法还是有很多细节要注意的，比如如果length指定的位置不是一条消息的结尾时如何处理等等，感兴趣的读者可以自己去看下源码是如何解决这些问题的。

下面来看下写消息的代码。

{% highlight java linenos %}
public List<Long> append(ByteBufferMessageSet messages) {
        //validate the messages
        int numberOfMessages = 0;
        for (MessageAndOffset messageAndOffset : messages) {
            if (!messageAndOffset.message.isValid()) {
                throw new InvalidMessageException();
            }
            numberOfMessages += 1;
        }

        ByteBuffer validByteBuffer = messages.getBuffer().duplicate();
        long messageSetValidBytes = messages.getValidBytes();
        if (messageSetValidBytes > Integer.MAX_VALUE || messageSetValidBytes < 0) throw new InvalidMessageSizeException(
                "Illegal length of message set " + messageSetValidBytes + " Message set cannot be appended to log. Possible causes are corrupted produce requests");

        validByteBuffer.limit((int) messageSetValidBytes);
        ByteBufferMessageSet validMessages = new ByteBufferMessageSet(validByteBuffer);

        // they are valid, insert them in the log
        synchronized (lock) {
            try {
            	//获取最后一个logsegment对象，append数据即可
                LogSegment lastSegment = segments.getLastView();
                long[] writtenAndOffset = lastSegment.getMessageSet().append(validMessages);
                if (logger.isTraceEnabled()) {
                    logger.trace(String.format("[%s,%s] save %d messages, bytes %d", name, lastSegment.getName(),
                            numberOfMessages, writtenAndOffset[0]));
                }
                //检查是否要flush数据到磁盘
                maybeFlush(numberOfMessages);
                //检测该文件是否达到定义的文件大小，如果达到了，要新建一个文件
                maybeRoll(lastSegment);

            } catch (IOException e) {
                logger.fatal("Halting due to unrecoverable I/O error while handling producer request", e);
                Runtime.getRuntime().halt(1);
            } catch (RuntimeException re) {
                throw re;
            }
        }
        return (List<Long>) null;
    }

{% endhighlight %}

写文件的代码也很简单，获取最后一个LogSegment，然后append就可以了，之后检查一下是否要flush和roll就好了。

另外该类也提供了markDeletedWhile和getOffsetsBefore方法，分别用于标记jafka文件是否可删除和在某时间之前的offset值，这里就不展开讲了，感兴趣的读者可以自行去阅读。

##小结

本文主要讲述了LogManager相关的类，jafka数据文件的组织方式等，希望对大家理解这部分源码有所帮助。
