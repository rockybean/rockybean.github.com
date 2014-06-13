---
layout: post
title: "网络知识随记"
description: ""
category: 
tags: []
---
{% include JB/setup %}

写作思路如下：

1. TCP协议栈-应用层 传输层IP层 物理层

每一层的作用？重点介绍IP协议，其不保证数据到达对方。引出TCP UDP，这两者的区别以及为什么要叫UDP User Datagram，数据报这个名字不怪么？

ICMP协议？ping和traceroute的实现 linode禁止ping的实现方式？

2. UDP介绍

介绍UDP的特征，比如不保证可达性等。

3. TCP

详细介绍 三次握手连接 发送消息 四次握手关闭，结合图层讲清楚。每一个状态码都要讲。

上个大图[TCP-Ip.gif],比较全面了，看起来挺晕，做个备份。


讲解网络配置中各个参数的意义

      

** TCP/IP协议栈
TCP/IP协议分为4层：应用层 传输层 网络层 网络接口层。简单介绍如下：

应用层：定义了一些上层应用可以直接使用的协议，如http ftp等协议。

传输层：定义了一些控制数据传输的协议，用以保证数据的可靠性和顺序到达性等，如tcp udp协议。

网络层：定义了一些网络间相关的协议，如IP协议用于网际路由，ICMP协议用于检测网络的通畅性，ARP协议用于获取设备mac地址等等。

网络接口层：定义了一些网络介质上传输的协议，如Ethernet协议、802.3协议等等，主要包含在os的网卡驱动程序中。

网络协议从上层到下层的封装如图[network-protocol-all.png]

??? TCP传给IP的数据单元称作TCP报文段(segment。
??? IP传给链路层的数据单元称作IP数据报(IP datagram)
??? 通过以太网传输的比特流称作帧（Frame)


* 网络接口层

tcp/ip的网络接口层对应OSI模型中的链路层和物理层，其传输的数据单位是帧(frame)。在上层要发送的数据包的首部和尾部添加相关数据后封装成帧然后发送出去， 其主要有如下三个作用：
1）为网络层接收和发送ip数据包。(IP datagram)
2）为arp发送请求和接收数据。
3）为rarp发送请求和接收数据。

网络接口层协议有Ethernet协议、802.3等等，如图[network-link-protocol.png]展示了这两种协议的组成部分。

该层首部通常包含目的和源地址，就是设备的mac地址，尾部是一个CRC校验码，用于保证数据的准确性。

细心的同学应该观察到中间传输的数据域是有长度限制的，如802.3是从38到1492，ethernet协议是从46到1500。这里数据长度的限制是由传输介质的物理特性决定的，如果传输的数据长度不在该范围内，则该帧会被丢弃。这里要引出MTU的概念，MTU(Maximal Transmission Unit)，最大传输单位值，即可以一次性传输数据的最大值。在以太网协议中，链路层的MTU值为1500.一个帧的构成为7字节前导同步吗＋1字节帧开始定界符＋6字节的目的MAC＋6字节的源MAC＋2字节的帧类型＋1500＋4字节的FCS,最大值为1526，但通过抓包工具获取的一个帧最大却是1514，原因在于数据帧到达网卡后，在物理层网卡会先去掉前导同步码和帧开始定界符，之后根据FCS进行验证，如果失败则丢弃该帧，否则就检查目的MAC地址是否符合自己的接收条件（MAC地址和自己的匹配、广播地址等等），如果符合，将该帧交设备驱动程序做进一步的处理，这个时候抓包工具才能抓到包，此时的帧也被去除了校验码，所以最终抓到的帧大小为6+6+2+1500=1514。有最大值就有最小值，当上层传输的数据小于最小值时(ethernet时46)，比如tcp三次握手时的ack返回仅有20（tcp头部）+20(ip头部)=40字节，小于最小值46字节。对于此种情况，网卡驱动程序会进行自动填充，但如果抓包工具如果先于驱动程序抓到该帧，那么其大小就要取决于抓包工具本身的显示，wireshark只是显示原大小。

上面对于链路层中的包长度进行了简单地说明，感兴趣的同学可以去查看[这篇博文](http://blog.csdn.net/naturebe/article/details/6712153)。


下面这篇文章对ip长度研究较深。
http://blog.csdn.net/naturebe/article/details/6712153



把帧的详细组成再详细讲下吧？？参考 http://lbzxy.blog.51cto.com/497155/125527

* 网络层

网络层也常叫IP层，因为这一层最重要的协议就是IP协议，用于网际路由，提供不可靠无连接的数据报传送服务，该层的数据传送单位为IP数据报(IP datagram)。

其协议格式如图[network-ip-protocol.png].

这里不再详细讲解每一部分的作用，简单介绍几个部分。首部长度(4bit)是指头部长度有多少个32bit，所以其表示的最大值为60byte。总长度(16bit)是指整个ip数据报的长度，表示最大值为64K，也就是说IP报是有最大传输限定值的。标识字段用于唯一的表示一次数据传输，标志位(3bit)用于表示是否可以分片或者是否有其他分片，片偏移(13bit)用于切片时的片偏移。

这里详细讲解下IP分片。IP数据报下一层要通过数据链路层封装成帧发送出去，但帧大小是由限制的，即MTU，不同的网络拓扑和设备的MTU也不同，如果IP数据报大于MTU，那么它必须分片才能实现数据的传送，分片发生在各个网络设备上，在目的主机参照标识字段、标志位和片偏移来实现重组。值得注意的是，ip分片传输后，每一个标识字段都被复制到每一个分片上，而总长度也变为该片的长度。优点是IP数据报可以穿过复杂多变的网络环境，缺点是一个分片丢失，该数据报就发送失败，增大了丢包的概率。

网络层另一个重要的协议是ICMP(Internet Control Message Protocol)协议，其提供了一套查找网络故障的机制，有差错报文和控制报文，可用于检查主机不可达、中间路由出问题、网络拥塞等问题，不同的问题会返回不同的错误类型。ICMP的功能只是报告问题而不能纠正错误。ping命令使用控制报文中的回显请求和应答来实现的，traceroute是使用差错报文中的ttl超时报文和目标不可达报文实现的。

网络层还有ARP和RARP协议用于在IP和mac地址之间转换，这里就不详细讲解了。

另外[这篇文章](http://blog.csdn.net/zhaqiwen/article/details/7727031)详细地讲述了数据报是如何在局域网和广域网中传递的，感兴趣的可以去详细看下。

* 传输层

该层主要对应用层的数据进行分割和重装，提供端到端的服务。主要由UDP和TCP两种协议，下面分别讲解这两种协议。

* UDP
UDP即User Datagram，用户数据报协议，从名字上会联系到IP Datagram，即Ip数据报，两者也确实有关联。在UDP被并入TCP/UDP协议簇之前，是作为IP协议的上层抽象存在的，它的名字也是源于IP Datagram，前面加上User便有了端到端的意味。图[network-udp-protocol.png]展示了UDP的包结构。

UDP的首部只占用8个字节，顶部的12个字节的伪首部是用于计算UDP首部校验和的。源端口号和目的端口号是传输层协议用以区分应用程序的标志。

UDP是面向报文的，没有可靠性控制，没有拥塞控制，无连接，所以其开销小，但网络环境差或者发送数据过大导致ip分片过多时，会导致发送率降低，影响程序的使用。为了提高UDP的发送率，应该尽量使得UDP可以使用一个IP数据报就能发送出去，这里就涉及到前文提到的MTU。UDP推荐传送的数据大小为1500-20(ip数据报首部)-8(udp首部)=1472字节。

* TCP
TCP(Transportation Control Protocol)，传输控制协议，提供了一种可靠的面向连接的字节流传输层服务。TCP协议相对复杂，主要知识点有三次握手建立连接、四次握手关闭连接、滑动窗口协议、拥塞控制策略、Nagle算法等等，我们这里只做简单介绍，基本一带而过，感兴趣的可以去详细研究。图[network-tcp-protocol.png]展示了tcp报文段(segment)的格式。

源端口号和目的端口号不必多讲了，序号和确认号是用于握手的凭证。4bit的首位长度表明tcp首部最大为15*4Byte=60byte。后面的标志位有几个要讲解下：

ACK：ACK=1表示该报文段中有确认号需要处理。
PSH：PSH=1表示该报文段中有数据需要处理。
RST：RST=1表示到目的端的连接出问题了，需要上层做出处理，如重新建立连接等。
SYN：SYN=1 ACK=0表明是建立连接请求报文段，SYN=1 ACK=1表明同意建立连接报文。
FIN: FIN=1表示对端的数据已经发送完毕，要求释放连接。

窗口大小是用来实现滑动窗口协议的，用来加快tcp数据的传输。

下面我们来看看建立连接的示意图。[network-tcp-connection.png]

三次握手的详细过程上图已经表现地很详细了，这里不再赘述。这里说一下第三次握手的必要性，即发送方在收到接收方的ack后又主动发送了一次ack给接收方。原因是为了避免一种异常情况：在网络不稳定的情况下，发送方发出的一个连接请求经过在某个网络中间节点滞留，等其到达接收端时正常的通信早已结束，但接收方不知道，所以它会立刻发送一个ack给发送方，如果此时没有第三次握手的确认，那么服务端会认为该连接有效，造成资源的浪费。

下面再看一下释放链接的示意图。[network-tcp-fin.png]

这里主要说一下TIME_WAIT这个状态，为什么会有一个2MSL的时间存在？MSL即Max Segment LifeTime，一个报文段的最长生存时间。2MSL便是用来保证A发送的最后一个ack可以到达B，如果没有到达B，B会超时重发ACK和FIN报文，此时A也可以收到该报文，然后重新发送ack。

TCP的状态迁移见下图[network-tcp-status.png]

上面这张图基本阐述了每个状态的迁移时机，这是一个有限状态机表现方式。


* 应用层

该层主要是面向应用的协议，如http ftp等等，这些协议都相对复杂，这里就不展开讲。 


前文简要介绍了tcp/ip的组成与每部分的包结构，而这些都是为了解决下面的几个问题，各位请看。

常见问题解答：
   
      * MTU是什么？MSS是什么？UDP和TCP是如何分段和分片的？
      MTU是Maximal Transmission Unit，最大传输单元，即链路层传输数据段的最大字节数，如在链路层以太网协议中，MTU是1500，当然这个值是可以自己修改的。
      MSS是Maximal Segment Size，最大段长度，是每个TCP报文段中数据段的最大长度，默认是536byte。该值存在的意义是防止传输数据过小造成资源浪费，数据过大造成ip包分片增大丢包率。如果MSS为1byte，那么为了传输这1byte的数据，需要额外传输ip头部20byte和tcp头部20byte，即要多传输40个byte，这造成了资源的浪费，传输效率也大打折扣；另一方面如果MSS过大，超过了MTU值，那么势必会在ip层发生分片，而接收方在处理分片时也会消耗更多地资源和时间，如果分片在传输过程中发生了重传，也会进一步增大网络开销。所以MSS的合理值应为保证在网络传输中不产生IP分片的最大值，从而最大化利用网络资源。
      图[network-mtu-mss.png]描述了MTU和MSS的关系，即MTU=IP Header + TCP Header + MSS

      图[network-tcp-mss.png]描述了TCP是如何协商mss大小的。注意mss只出现在syn包中。

      * 什么是半连接?Syn flood是怎么一回事？
      半连接是指在服务端接收到客户端的SYN数据包后，会生成一个不完全连接对象存储在半连接队列中，当收到客户端的ack包后会将该对象由半连接队列(syn queue)转移到已连接队列(accept queue)等待accept系统调用处理。这里有两个配置参数要讲，一个是/proc/sys/net/ipv4/tcp_max_syn_backlog，这个指定了半连接队列的最大长度，另一个是sk_max_ack_backlog，该参数指定了全连接队列的最大长度，即完成了三次握手但还没有被accept系统调用取走的连接，一般是listen函数的第二个参数（listen(int sockfd,int backlog)(对应java编程中的new ServerSocket(port,backlog)？)）。
      Syn flood攻击便是利用了半连接队列来实现的，通过伪造大量的tcp SYN数据包，使得对方资源耗尽（CPU满负载或者内存耗尽）。当收到SYN Flood攻击时，服务端的半连接队列会被迅速占满，正常连接会由于半连接队列已满而被丢弃，此时会发现服务端大量连接处于SYN_RECV状态，服务端会不停重试发送ack包给并不存在的客户端，导致cpu满负载，从而达到攻击效果。一般可以通过修改net.ipv4.tcp_synack_retries 来减少重试发送ack的次数、开启 net.ipv4.tcp_syncookies、调大net.ipv4.tcp_max_syn_backlog来进行防御。
      下面2篇文章对半连接都进行了很详细的讲解：[文章一](http://blog.chinaunix.net/uid-20357359-id-1963498.html)和[文章二](http://www.piao2010.com/linux%E8%AF%A1%E5%BC%82%E7%9A%84%E5%8D%8A%E8%BF%9E%E6%8E%A5syn_recv%E9%98%9F%E5%88%97%E9%95%BF%E5%BA%A6%E4%B8%80)
      阿里云在2014年春节期间也遭遇过SYN Flood攻击，[文章链接](http://c.blog.sina.com.cn/profile.php?blogid=e8e60bc089000xcq)

      * 什么是粘包？怎么解决？封包和拆包
      粘包出现的原因是TCP数据的“流”特性。所谓“流”是指tcp传输的数据并不会在逻辑上进行划分，比如客户端给服务端发送了A和B两份数据，tcp发送的时候并不是分两次先发A数据，再发B数据，而是可能出现下面的情况：
      1. 先发送A和B的一部分，再发送B剩余部分
      2. 先发送A的一部分，再发送A剩余部分和B

      可能有人会问，我们在编程的时候不是直接调用了send(byte[])了吗？为什么数据会出现这种混乱的发送情况，原因有两个：
      1. 如果开启了Nagle算法，即tcp_no_delay是false，那么几遍调用了send方法，tcp也不一定会立即将数据发送出去，而是可能会等待其他数据一同发送，这样做是为了最大化利用网络资源。
      2. 如果发送的数据过大，超过了MSS(见第一个问题)的大小，tcp会对数据进行分段发送，这是也不可能一次性将数据发送完毕。

      这个时候服务端需要能够把A和B区分开，如果只是单纯地将一次接收的数据作为完整数据处理，很容易出错。这里就引出了封包和拆包的概念，所谓封包，就是将要传送的数据按照一个可辨识的结构传输，比如客户端和服务端约定传递的数据都遵从 字节长度(4Byte)+实际数据 的规则，那么服务端在解析数据(即拆包)的时候，就可以先读取4Byte，获取到一个完整数据的实际长度，然后再读取该长度的数据，之后再交由上层处理。
      [这篇文章](http://blog.csdn.net/fengge8ylf/article/details/793808)对粘包 封包和拆包讲解的比较清楚。


      * socket的close和shutdown有区别吗？
      这个涉及到tcp关闭连接的方式，主要由两种：一种正常的通过4次握手关闭，这是优雅的方式，可以保证双方的数据都被接受了，另一种方式是一方发送RST，则对方会立刻关闭链接，这是暴力的方式。
      close会直接关闭socket的文件描述符，包括输入和输出流，但shutdown只是在输入输出流上做了处理，比如丢弃了输入流中的数据，读到EOF结束符，向输出流写入东西时报错等等，但并不会关闭socket，即对方还是正常读写，要关闭还是要调用close。另外close只会关闭本进程的socket 文件描述符，如果其还被其他进程调用，则实际不会被关闭。但shutdown却会影响所有使用了该socket的进程。无特殊情况，使用close关闭socket即可。
      在shutdownOutput后，再写入时会报broken pipe的错误。对执行了close的socket读入时会报错：socket closed
      [参考资料](http://hi.baidu.com/yoshubom/item/758f025d98df733e33e0a9ad)

      * broken Pipe和connnection reset的出现原因是什么？
      1.broken pipe是指socket已经不能再写数据，原因有二：一是对方已经没有进程在读数据，一般是异常中止，二是自己关闭了输出，调用shutdownOutput方法。
      试验：客户端连接到服务端，服务端每1s向客户端发送一个数据，然后手动将客户端kill掉，此时服务端向客户端发送数据时会报broken pipe的错误。tcpdump抓包如下：

11:51:40.927214 IP localhost.64909 > localhost.44444: Flags [S], seq 803735780, win 65535, options [mss 16344,nop,wscale 4,nop,nop,TS val 444822151 ecr 0,sackOK,eol], length 0
11:51:40.927349 IP localhost.44444 > localhost.64909: Flags [S.], seq 4280413071, ack 803735781, win 65535, options [mss 16344,nop,wscale 4,nop,nop,TS val 444822151 ecr 444822151,sackOK,eol], length 0
11:51:40.927366 IP localhost.64909 > localhost.44444: Flags [.], ack 1, win 9186, options [nop,nop,TS val 444822151 ecr 444822151], length 0
11:51:40.927379 IP localhost.44444 > localhost.64909: Flags [.], ack 1, win 9186, options [nop,nop,TS val 444822151 ecr 444822151], length 0

11:51:40.934675 IP localhost.44444 > localhost.64909: Flags [P.], seq 1:2, ack 1, win 9186, options [nop,nop,TS val 444822158 ecr 444822151], length 1
11:51:40.934710 IP localhost.64909 > localhost.44444: Flags [.], ack 2, win 9186, options [nop,nop,TS val 444822158 ecr 444822158], length 0

11:51:42.936147 IP localhost.44444 > localhost.64909: Flags [P.], seq 3:4, ack 3, win 9186, options [nop,nop,TS val 444824059 ecr 444823099], length 1
11:51:42.936213 IP localhost.64909 > localhost.44444: Flags [.], ack 4, win 9186, options [nop,nop,TS val 444824059 ecr 444824059], length 0


11:51:43.705731 IP localhost.64909 > localhost.44444: Flags [F.], seq 4, ack 4, win 9186, options [nop,nop,TS val 444824794 ecr 444824060], length 0
11:51:43.705769 IP localhost.44444 > localhost.64909: Flags [.], ack 5, win 9186, options [nop,nop,TS val 444824794 ecr 444824794], length 0
11:51:43.705780 IP localhost.64909 > localhost.44444: Flags [.], ack 4, win 9186, options [nop,nop,TS val 444824794 ecr 444824794], length 0

11:51:43.937395 IP localhost.44444 > localhost.64909: Flags [P.], seq 4:5, ack 5, win 9186, options [nop,nop,TS val 444825014 ecr 444824794], length 1
11:51:43.937443 IP localhost.64909 > localhost.44444: Flags [R], seq 803735785, win 0, length 0


44444是服务端进程，前3行是3次握手建立连接，第4行服务端又向客户端发送了一个ack包，这里貌似是拥塞控制用于调整窗口的包。后面服务端向客户端连着发送了两个P包，用来发送数据。之后客户端被kill掉，可以看到客户端主动向服务端发送了一个FIN包，即关闭socket连接。之后当服务端再次发送PUSH包时，客户端返回RESET包，这时服务端进程就抛出broken pipe的异常了。

从这里可以看出RESET包是报错的关键，我们可以先来看下RESET包出现的时机：
1. 当尝试和未开放的服务端端口建立连接时，对方会直接返回RESET包，客户端会抛出Connection refused的异常。
抓包如下：
11:51:27.901582 IP localhost.64889 > localhost.44444: Flags [S], seq 1844850030, win 65535, options [mss 16344,nop,wscale 4,nop,nop,TS val 444809686 ecr 0,sackOK,eol], length 0
11:51:27.901613 IP localhost.44444 > localhost.64889: Flags [R.], seq 0, ack 1844850031, win 0, length 0
2. 双方建立了连接，当一方因为异常中止时，当另一方尝试写时会发回RESET包,即上面的情况。

3. ack报文丢失，并且超出一定重传次数或时间后，会主动向对方发送RESET包。

4. 当收到的TCP报文不是当前tcp列表可处理时，发送RESET包。

5. 当close一个socket时，如果其缓存中还有未处理的待读数据，也会向对方发送RESET包，而不是正常的FIN包。注意，这种情况只在部分系统中出现，参见论文(http://cs.baylor.edu/~donahoo/practical/CSockets/TCPRST.pdf);

 todo：这篇文章讲了写的时候第一次不报错，第二次报错的情况，最后有很好的说明
 http://wushank.blog.51cto.com/3489095/1135060
 

后面两个是参考的其他文章，没有实际测试。

2. connection reset by peer错误
 用阻塞式没有捕获到该异常，都是broken pipe，使用nio试一下吧
 http://weixiaolu.iteye.com/blog/1479656

 还没有重现，网上看了下，貌似是在大并发连接的情况出现了这种错误

 http://stackoverflow.com/questions/2692319/java-clearing-up-the-confusion-on-what-causes-a-connection-resett 

 http://lzy.iteye.com/blog/383884

 http://blog.163.com/tyw_andy/blog/static/1167902120097131113149/


发现一个现象，当客户端因异常关闭时，服务端的selector会收到一个可读的SelectionKey，此时一读就会报错

为什么？？

broken pipe从字面可以理解为继续向已关闭socket中写数据，这其中涉及到的细节是当一方异常关闭时，会主动close掉自己的socket，并且不等4次握手，直接发送Reset报文给对方后便退出。另一方收到Reset报文后，上层捕获后应该
同步的时候会触发读事件吗》？？？测试下

connection reset是在什么情况出现的？收到了EPIPE等信号吗？



      

      * flush有必要吗？
      我们在使用文件流的时候，都会强调一下flush函数，有时如果不主动调用，会导致写入的数据没有到磁盘上。那么socket编程中是否要注意flush呢？世界上当我们socketOutputStream.write(byte[])的时候，数据已经发送出去，与flush无关。当然如果你再outputstream的外层套一层buffer，这个时候flush就有必要了，因为buffer相当于在jvm层做了一层缓存，直到调用flush函数，数据才会被写出。

      * 进程在意外停止的时候，其关联的socket会关闭吗？
      在linux系统中，的确是这样的。当进程意外中止或者被kill的时候，系统会直接调用相关socket的close函数，单方面关闭socket链接并且不等待对方关闭。即进程关闭了，socket也会被关闭。
      [参考链接](http://blog.csdn.net/dog250/article/details/13760985)

      * 后台TIME_WAIT状态的连接很多是什么原因？怎么解决？TIME_WAIT重用
      TIME_WAIT存在的原因是tcp为了安全的完成四次握手而关闭连接。四次握手大致分为以下三个步骤：
      第一，主动关闭方发送FIN包，处于FIN_WAIT1状态，被动方返回ACK包，处于CLOSE_WAIT状态。
      第二，被动方发送FIN包，处于LAST_ACK状态，主动方返回ACK包，处于TIME_WAIT状态。
      第三，被动方接收ACK包，处于CLOSED状态。主动方等2MSL一过变为CLOSED状态。

      MSL是Max Segment LifeTime是报文最大生存时间，是指任何报文在地球上存在的最长时间，超过该时间的报文会被丢弃，也就是说在火星上这个时间会变化。RFC规定该时间为2分钟，实际使用一般是30s。
      假设没有TIME_WAIT这个状态主动方直接CLOSED会出现什么问题呢？
      由于网络堵塞等原因，主动方在第二个步骤返回的ACK包没有到达被动方，那么被动方会再次发送FIN包，如果这个时候主动关闭方与被动方又重新建立了连接，此时收到FIN包，新连接就会进入4次握手关闭步骤，异常发生。所以TIME_WAIT的存在是有意义，但该状态要持续多久呢？由于tcp规定对于纯ACK包是不能做出反应的，也就是说主动方不能靠再接受ack包来结束TIME_WAIT状态，那就只能设计一个安全时间了，这个时间就是2MSL。如果在该段时间内，主动方还没有收到被动方的FIN包，就可以认定对方已经收到ACK包，之后就可以安心的转为CLOSED状态了。
      TIME_WAIT虽然确保了连接的安全关闭，但却极大地浪费了端口资源，很多时候处于TIME_WAIT的连接都是可以被立即关闭的。那么如何解决这个问题呢？有两种方法：快速回收和重用。

      1. 快速回收是指在主动关闭方无需等待2个MSL的时间，只需要等待一个重传时间随即释放，通常时间很短。（可参考此处的源码分析http://blog.sina.com.cn/s/blog_781b0c850100znjd.html）
      快速回收需要配置两个参数，
      net.tcp_tw_recycle=1 
      net.tcp_timestamp=1
      只有当这两者都开启时，快速回收才会起作用。原因是虽然连接被快速回收了，但是有潜在的风险：
      新建立的连接可能会被重传的FIN包关闭，造成异常。
      解决该问题的方案是引入ip层的缓存记录。ip层可以缓存连接方的ip和最后tcp通信的时间戳。但下一次同一个ip发送连接请求时(SYN包)，如果满足以下三个条件，则被视为老的重复数据包，被直接丢弃掉。
      1) 来自同一个ip的tcp请求，且带有时间戳
      2) 在MSL之内有同一个ip的tcp数据到过
      3) 新连接的时间戳小于上次最后tcp通信的时间戳，且差值大于重放窗口戳。

      由于判断条件使用了时间戳，所以tcp_timestamp的参数必须打开。 
      然而当通过NAT网络连接服务端时，会非常容易触发上面三个条件，可以参照七牛的一个分享(http://blog.lidaobing.com/2013/09/14/no-gamble-on-bug.html)。他们就碰到了类似的问题，虽然分享者认为是开了tcp.reuse的问题，但我认为其实应该是开了tcp.recyle导致的。
      一般不推荐使用该方法解决TIME_WAIT过多的问题，无论是客户端的NAT网络还是服务端的NAT网络，都容易出现时间戳紊乱问题导致SYN被拒绝的情况发生，出现超时等现象。

      2. 重用是指当客户端连接满足一定条件时，服务端可以将处于TIME_WAIT的原连接(四元组)重用。
      只要满足一下任意一个条件即可发生连接重用，当然前提是四元组信息不能变，也就是要求客户端发起连接的ip和端口都不变。

      1) 新连接的SYN序列号比TIME_WAIT老连接的最后一个序列号要大

      2) 如果开启了net_timestamp，那么新连接的时间戳大于老连接的时间戳

      从上面可以看到tcp_reuse是可以独立使用的，只不过开启了net_timestamp会加大重用的概率。

      还有常被提及的参数tcp_max_tw_buckets，用于控制timewait的连接总数，超过了限制的timewait连接会被删除。简单粗暴，但并不是一个推荐的做法，是有一定风险性的。

      这里还有一个解决方法，就是不要让服务端去主动关闭连接。我们知道TIME_WAIT是出现在主动关闭一方的，所以如果客户端可以主动关闭连接的，那么服务端就可以大量减少TIME_WAIT的连接数了。比如上面七牛碰到的问题最终便是通过开启nginx的keep alive参数来解决的。当然他们tw多得原因是nginx反向代理连接过多导致的，如何配置？？！！！所以要再upstream里面配，注意这里不是配置真正客户端开启keepalive参数。



      参考 http://blog.csdn.net/dog250/article/details/13760985
      参考二 http://huoding.com/2013/12/31/316
      修改参数

      net.ipv4.tcp_tw_reuse tcp重用

      tcp重用和快速回收
      都可以开启timestamp
      快速回收一定要开启timestamp，而tcp重用不一定 
      


      tcp_tw_recycle打开导致的问题
      http://www.pagefault.info/?p=416

      七牛出国由于设置了tcp_tw-reuse出现了问题
      
      为什么都是12s延时？

       

      * Tcp_No_Delay参数有什么用？什么情况下进行设置？
      tcp_no_delay是指是否使用Nagle算法，true为不使用，反之使用。当希望调用write函数后，数据立马发送，即对实时性要求高时，要将该参数设为true。

      * tcp最大连接数怎么提高？
      除了ulimit -n
专业一点的，应该是这样：
http://www.cyberciti.biz/faq/linux-increase-the-maximum-number-of-open-files/

系统级别的用fs.file-max

[217 gryphon]# cat /proc/sys/fs/file-max
2000000

      * RST复位标志在何时发出？

        RST: 复位字段被用于当一个报文发送到某个socket接口而出现错误时，TCP则会发出复位报文段。常见出现的情况有以下几种：  www.2cto.com  
        发送到不存在的端口的连接请求：此时对方的目的端口并没有侦听，对于UDP，将会发出ICMP不可达的   错误信息，而对于TCP，将会发出设置RST复位标志位的数据报。异常终止一个连接：正常情况下，通过发送FIN去正常关闭一个TCP连接，但也有可能通过发送一个复位   报文段去中途释放掉一个连接。在socketAPI中通过设置socket选 项SO_LINGER去关闭这种异常关闭的情况。

        当调用close方法时，程序会先讲发送缓冲区中的数据发送完毕，然后进行四次握手关闭，可以通过SO_LINGER来控制该特性。SO_LINGER打开可以在close方法被调用时，能够在设定的范围内尽量将发送缓冲区内的数据发送，如果数据完全发送那么进行四次握手关闭连接，否则直接以RST形式关闭连接，就跟程序异常终止一样。


linux常见网络参数意义及优化





网络编程常见的参数
http://blog.csdn.net/huang_xw/article/category/1095933




