---
layout: post
title: "一场由Mavericks乱改curl引起的https请求失败的疑案"
description: ""
category: 
tags: []
---
{% include JB/setup %}


## 背景
公司最近成功申请了微信支付，我负责相关接入工作。接入过程可谓一波三折，也令我对微信的技术能力不再盲目称赞，这里暂时不说，后面专门写文章讲。在接入退款接口时，根据demo里面提供的代码进行测试。

```php
$curl = curl_init();

curl_setopt( $curl , CURLOPT_TIMEOUT , 5 );
curl_setopt( $curl , CURLOPT_RETURNTRANSFER , 1 );

curl_setopt( $curl , CURLOPT_POST , 1 );
curl_setopt( $curl , CURLOPT_URL ,     self::REFUND_URL );
curl_setopt( $curl , CURLOPT_POSTFIELDS , array() );

// 设置公钥证书
curl_setopt( $curl , CURLOPT_SSL_VERIFYHOST , 1 );
curl_setopt( $curl , CURLOPT_SSL_VERIFYPEER , 1 );
curl_setopt( $curl , CURLOPT_CAINFO , $this->getCacertDir() . '/tenpayCacert.pem' );

// 设置客户端证书信息
curl_setopt( $curl , CURLOPT_SSLCERT , $this->getCacertDir() . '/tenpay.pem' );
curl_setopt( $curl , CURLOPT_SSLCERTPASSWD , '1111' );
curl_setopt( $curl , CURLOPT_SSLCERTTYPE , 'PEM' );

$res = curl_exec( $curl );
$responseCode = curl_getinfo( $curl , CURLINFO_HTTP_CODE ) ;

```php

通过以上代码测试的时候请求总是失败，由于我对于https的知识不了解，于是去简单地查看了下相关介绍，感兴趣的同学可以去看下这篇文章(http://blog.nklike.com/network/ssl%E5%8D%8F%E8%AE%AE%E5%92%8C%E6%8A%93%E5%8C%85/)。https是用SSL来建立连接传输数据的，而SSL是在TCP之上实现的安全性连接，其安全性的实现就是靠证书，其实就是密钥机制。简单讲，加密有两种方式：非对称加密和对称加密，前者有公钥和私钥，用公钥加密的内容只能用私钥解开，反之亦然；而后者就是我们常见的只有一把密钥，用它来加解密。前者的好处安全性高，因为我只需要把公钥对外发布即可，私钥自己保留，但效率低，因为会涉及到大量的计算；后者的好处是效率高，安全性低，因为密钥容易泄露。SSL的实现就是基于这两种加密方式来实现的，首先用非对称加密的方式在客户端和服务端之间确定一个对称加密的密钥(有客户端随机生成)，之后便用对称加密的方式来传输数据。那么接下来的问题便是非对称加密中的公钥和私钥如何来管理？私钥自然要保存在服务端，那么公钥如何分发给客户端呢？答案是证书，我们常用的证书和区别如下：

* DER Encoded Certificate .cer/.crt用于存放证书，DER二进制编码形式存储,不含私钥

* PEM Encoded Certificate .pem是以Base64的ASCII编码形式存储，只是一种存储形式，可以公钥或私钥。

* PKCS#12 Personal Information Exchange .pfx/.p12用于存放个人证书(可以包含公钥和私钥),二进制形式保存。



其实上面这段代码用到了SSL双向认证，即客户端认证服务端，服务端也要认证客户端，后者可以简单理解为客户端做了一个登录请求，服务端验证通过后，客户端就可以操作自己的数据了，这个过程自然也是通过证书来实现的，原理类似就不深究了。下面简单讲下cur使用中相关参数的含义。


CURLOPT_SSL_VERIFYHOST参数用于指定是否验证对方证书中的域名，1验证证书中是否存在common name(证书的一个内容),2验证该common name是否和请求的hostname一致。

CURLOPT_SSL_VERIFYPEER参数用于指定是否校验对方证书是否合法。true验证，false不验证，配合CURLOPT_CAINFO使用。

CURLOPT_CAINFO参数用于指定验证对方证书合法性的证书，一个或多个。推测其原理是通过对比已获取的有效证书和对方在ssl握手中传回的证书来实现。

上面三个参数是用于验证服务端证书的，如果把CURLOPT_SSL_VERIFYPEER关闭，即不进行证书验证。

CURLOPT_SSLCERT参数用于指定个人证书，该证书一般为pem格式，可以包含公钥和私钥，如果包含私钥，则不必再指定CURLOPT_SSL_KEY参数。

CURLOPT_SSLCERTPASSWD参数用于指定使用上述参数指定证书的密码。

CURLOPT_SSLCERTTYPE参数用于指定证书类型，可以为PEM DER等。

如果CURLOPT_SSLCERT指定的证书不包含私钥，那么还要用到下面的两个参数用于指定私钥来完成ssl认证。

CURLOPT_SSLKEY参数用于指定一个私钥文件，一般为PEM格式。

CURLOPT_SSLKEYPASSWD参数用于指定上述文件的使用密码。

使用上述配置就可以在php下面完成https请求了。同理，直接使用curl命令也是可以完成请求的，方法如下：

curl --cert [certFile:passwd] --cacert [caFile] -v [https://your/request/site]

--cert用于指定个人证书，证书路径和密码间用冒号分离，--cacert用于指定公钥证书，-k可以用于指定跳过验证服务器证书的步骤，相当于前文的CURLOPT_SSL_VERIFYPEER设为false。


一切看来很美好，然而我在开发的时候却总是请求失败，报错信息为
curl: (35) Unknown SSL protocol error in connection to ...

毫无头绪，去网络上搜寻半天，尝试了各种方法依然无解，就这样2天过去了。突然有天想到直接使用curl来测试，首先定位下问题所在。但用curl的时候依然失败，证书重新导了好几遍，依然无效，就这样1天又过去了。新的一天突然想到会不会是curl的问题，于是去服务器上试了一把，立马成功。当时我那个心情啊，无边落草泥马有木有！！比较二者curl的版本发现了不同。

OSX 10.9.2 自带的curl

curl 7.30.0 (x86_64-apple-darwin13.0) libcurl/7.30.0 SecureTransport zlib/1.2.5
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp smtp smtps telnet tftp
Features: AsynchDNS GSS-Negotiate IPv6 Largefile NTLM NTLM_WB SSL libz

centos curl

curl 7.19.7 (x86_64-redhat-linux-gnu) libcurl/7.19.7 NSS/3.14.0.0 zlib/1.2.3 libidn/1.18 libssh2/1.4.2
Protocols: tftp ftp telnet dict ldap ldaps http file https ftps scp sftp
Features: GSS-Negotiate IDN IPv6 Largefile NTLM SSL libz

这两个版本的curl分别使用SecureTransport和NSS做ssl证书的处理，具体可参考链接[http://curl.haxx.se/docs/sslcerts.html]。难道就因为这个导致我请求失败吗？果断用brew安装了一个curl。
brew install curl --with-openssl

注意这里指定使用openssl处理ssl证书，否则依然会使用SecureTransport。再次运行curl命令，成功，涕泪纵横啊！

后来在curl官网找到了一篇文章[http://curl.haxx.se/mail/archive-2013-10/0036.html]，详细说明了OSX 10.9对于curl的更改，简单来讲apple只允许curl使用其keychain中的证书，通过--cacert --cert指定的证书统统无效。好吧，问题的根本原因算是找到了！后面重新用brew编译了php版本，并且让其依赖了openssl版本的curl，代码运行终于正常！

brew install curl —with-openssl
brew install php54 --with-homebrew-curl --with-imap  --with-debug --with-fpm

这个疑案解决花费了我近1周的时间，究其原因是自己对于ssl机制不理解，对于curl也不熟悉，对证书有畏惧心理，导致不敢去尝试，花费了大量的时间去查阅网络上不相关的资料和胡思乱想。
不能有畏难情绪！





