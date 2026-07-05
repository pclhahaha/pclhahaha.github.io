---
title: HTTPS
date: 2019-12-01 00:00:00
updated: 2019-12-01 00:00:00
tags:
  - HTTPS
  - TLS
  - HTTP
  - 网络协议
categories:
  - 网络
---

# HTTPS(HTTP over SSL)

HTTPS 实际上是HTTP over SSL，即在原来的HTTP协议层与TCP协议层之间加入SSL协议层，负责对数据通道进行加密。

TLS/SSL产生的背景是HTTP明文传输带来的几个问题：

（1） 窃听风险（eavesdropping）：第三方可以获知通信内容。

（2） 篡改风险（tampering）：第三方可以修改通信内容。

（3） 冒充风险（pretending）：第三方可以冒充他人身份参与通信。

因此要达成的目标是：加密传输（防窃听）、数字签名校验（防篡改）、身份证书（防冒充），基于这三个目标来了解TLS/SSL的工作原理就比较清晰了，能明确每一个部分设计的原因。

PS : HTTPS 默认端口为443，而HTTP默认端口为80

## TLS/SSL

先来了解一下TLS/SSL的前世今生。

SSL(Secure Socket Layer)，由网景公司开发，3.0版本开始标准化为TLS。

TLS(Transport Layer Security)是SSL的标准化后的产物，有1.0 1.1 1.2三个版本，默认使用1.0

TLS是SSL的标准化后的产物，现在实际使用的是TLS，由于习惯原因仍经常称之为SSL。

### 协议工作流程

安全传输层协议（TLS）用于在两个通信应用程序之间提供保密性和数据完整性。该协议由两层组成： TLS 记录协议（TLS Record）和 TLS 握手协议（TLS Handshake），协议栈如下：

![img](/2019/12/01/HTTP/ssl protocal)

其中记录协议负责数据的分片、压缩、验证、加密等，握手协议负责客户端与服务端的握手，秘钥交换，告警等。整体流程如下图所示。

![img](/2019/12/01/HTTP/ssl handshake)

或者如下图所示：

![img](/2019/12/01/HTTP/ssl handshake 1)

简单的SSL握手连接过程(仅server端交换证书给client)，详情可见参考文献[3]：

- 1.client发送ClientHello，指定版本，随机数(RN)，所有支持的密码套件(CipherSuites) ClientHello附带的数据随机数据RN，会在生成session key时使用，Cipher suite列出了client支持的所有加密算法组合，每一组包含3种算法，一个是非对称算法，如RSA，一个是对称算法如DES，3DES，RC4等，一个是Hash算法，如MD5，SHA等，server会从这些算法组合中选取一组，作为本次SSL连接中使用。
- 2.server回应ServerHello，指定版本，RN，选择CipherSuites，会话ID(Session ID) ServerHello包含一个session id，如果SSL连接断开，再次连接时，可以使用该属性重新建立连接，在双方都有缓存的情况下可以省略握手的步骤。server端也会生成随机的RN，用于生成session key使用。server会从client发送的Cipher suite列表中挑出一个，例如RSA+RC4+MD5。

**client接到ServerHello并处理后，双发状态是已经协商好了一组算法，包括对称加密、非对称加密、摘要，以及后续计算对称秘钥用的随机数**。

- 3.server发送Certificate server的证书信息，只包含public key，server保存有public key对应的private key，用于证明server是该证书的实际拥有者。 那么如何验证呢？原理很简单：client随机生成一串数，用server的public key加密(显然是RSA算法)，发给server，server用private key解密后返回给client，client与原文比较，如果一致，则说明server拥有private key，也就说明与client通信的正是证书的拥有者。 利用这个原理，就可以实现session key的交换，加密前的那串随机数就可用作session key，因为除了client和server，没有第三方能获得该数据了。实际上session key的交换也是这么做的。 原理很简单，实际使用时会复杂很多，数据经过多次hash，伪随机等的运算，前面提到的client和server端得RN都会参与计算。这个原理用于下面client发送ClientKeyExchange前进行session key的计算。

**client拿到了server的public key。**

- 4.Server发送ServerHelloDone
- 5.client发送ClientKeyExchange，用于与server交换session key client随机生成48字节的pre-master secret，padding后用public key加密得到130字节的数据发送给server，server解密也能得到pre-master secret。

双方使用pre-master secret、”master secret”常量字节流、前期交换的server端RN和client的RN作为参数，使用一个伪随机函数PRF，其实就是hash之后再hash，最后得到48字节的master secret。master secret再与”key expansion”常量，双方RN经过伪随机函数运算得到key_block，PRF伪随机函数可以循环输出数据，因此我们想得到多少字节都可以，就如Random伪随机函数，给它一个种子，后续用hash计算能得到无数个随机数，如果每次种子相同，得到的序列是一样的，但是这里的输入时48字节的master secret，2个28字节的RN和一个字符串常量，碰撞的可能性是很小的。得到key block后，算法就从中取出session key，IV(对称算法中使用的初始化向量)等。client和server使用的session key是不一样的，但只要双方都知道对方使用的是什么就行了。这里会取出4个：client/server加密正文的key，client/server计算handshake数据hash的key。

**注意双方只交换了pre-master key，后续的计算都是独立完成的，由于算法和种子都一样，所以得到的session key也一致。**

**server接到ClientKeyExchange处理后，双方状态是已经协商好了对称秘钥，后续再利用对称秘钥进行一次验证**。

- 6.client发送ChangeCipherSpec，指示Server从现在开始发送的消息都是加密过的
- 7.client发送Finishd，包含了前面所有握手消息的hash，可以让server验证握手过程是否被第三方篡改 Finishd：client发送的加密数据，这个消息非常关键，一是能证明握手数据没有被篡改过，二能证明自己确实是session key的拥有者(这里是单边验证，只有server有certificate，server发送的Finished能证明自己含有private key，原理是一样的)。 client将之前发送的所有握手消息存入handshake messages缓存，进行MD5和SHA-1两种hash运算，再与前面的master secret和一串常量”client finished”进行PRF伪随机运算得到12字节的verify data，还要经过改进的MD5计算得到加密信息。为什么能证明上述两点呢，前面说了只有密钥的拥有者才能解密得到pre-master key，master key，最后得到key block后，进行hash运算得到的结果才与发送方的一致。
- 8.server发送ChangeCipherSpec，指示Client从现在开始发送的消息都是加密过的
- 9.server发送Finishd，包含了前面所有握手消息的hash，可以让client验证握手过程是否被第三方篡改，并且证明自己是Certificate密钥的拥有者（因为如果不是，那么无法得到pre-master key，也就无法计算得到session key），即证明自己的身份。

握手完成后，客户端和服务端完成了对称秘钥session key的交换，数据通过对称秘钥加密后进行传输，实现了加密的数据传输通道。

### CA证书

![img](/2019/12/01/HTTP/ca cert)

上面的握手流程讲完并没有描述数字证书扮演的角色，以及服务端向客户端返回证书，客户端进行验证的过程，这部分过程见上图。

服务端，即站点先向CA申请证书，申请信息中带有自己生成的公钥信息，CA审核通过后签发证书，证书包含服务端公钥、证书签名，证书签名为将证书明文部分计算数字摘要后，用CA的私钥加密的签名，防止证书被篡改（后面会讲到数字签名这块）。证书申请与签发是由站点先行完成的。

客户端在与服务端握手的过程中拿到证书，由于证书明文部分带有证书的公钥信息，能用CA的公钥对签名进行解密，并通过对比解密后的数字摘要与自己用明文计算的数字摘要来验证信息是否被篡改，从而保证了证书的安全性。后续通过证书中的server公钥来验证该server为证书拥有者，原理见上面的握手流程。

CA证书的安全使得客户端可以验证服务端的身份，防止站点被冒充，即钓鱼网站。

### 对称加密算法

加密和解密使用相同的密钥，计算较快，安全性较差。在SSL中常用的对称加密算法有RC4,AES,3DES,Camellia等。

### 非对称加密算法

加密和解密使用不同的密钥，数据在一端用公钥加密后，在另一端用私钥解密，安全性较高。由于性能问题，非对称加密一般用于数字签名（前面提到的证书的签名）和秘钥（对称加密算法的秘钥）的交换。常见的算法有RSA、DSA、ECC。

### 数字签名

数字签名就是“非对称加密+摘要算法”，其目的不是为了加密，而是用来防止他人篡改数据。
其核心思想是：比如A要给B发送数据，A先用摘要算法得到数据的指纹，然后用A的私钥加密指纹，加密后的指纹就是A的签名，B收到数据和A的签名后，也用同样的摘要算法计算指纹，然后用A公开的公钥解密签名，比较两个指纹，如果相同，说明数据没有被篡改，确实是A发过来的数据。假设C想改A发给B的数据来欺骗B，因为篡改数据后指纹会变，要想跟A的签名里面的指纹一致，就得改签名，但由于没有A的私钥，所以改不了，如果C用自己的私钥生成一个新的签名，B收到数据后用A的公钥根本就解不开。

至于为什么要使用摘要算法是因为非对称加密算法对原文长度有要求，所以先通过摘要算法生成一段较短的指纹，再进行非对称加密

#### 摘要算法

摘要算法不是用来加密的，其输出长度固定，相当于计算数据的指纹，主要用来做数据校验，验证数据的完整性和正确性。常见的算法有CRC、MD5、SHA1、SHA256。
CRC32:CRC本身是“冗余校验码”的意思，CRC32则表示会产生一个32bit（8位十六进制数）的校验值。由于CRC32产生校验值时源数据块的每一个bit（位）都参与了计算，所以数据块中即使只有一位发生了变化，也会得到不同的CRC32值。

# HTTP 短连接 vs 长连接

下面来聊一下短连接与长连接。

HTTP是一个无状态的面向连接的协议，无状态不代表HTTP不能保持TCP连接，更不能代表HTTP使用的是UDP协议（无连接）。HTTP的短连接、长连接实际上指的是TCP协议的短连接、长连接，长连接即是指多个HTTP请求复用一个TCP连接。

在HTTP/1.0中，默认使用的是短连接。也就是说，浏览器和服务器每进行一次HTTP操作，就建立一次连接，但任务结束就中断连接。如果客户端浏览器访问的某个HTML或其他类型的 Web页中包含有其他的Web资源，如JavaScript文件、图像文件、CSS文件等，当浏览器每遇到这样一个Web资源，就会建立一个HTTP会话。但从 HTTP/1.1起，默认使用长连接，用以保持连接特性。使用长连接的HTTP协议，会在响应头加入这行代码：

Connection:keep-alive 服务器和客户端都要设置

## TCP 协议

### 三次握手与四次挥手

TCP连接的建立与释放分别对应的是三次握手与四次挥手，具体如下图：

![img](/2019/12/01/HTTP/tcp handshake)
![img](/2019/12/01/HTTP/tcp fin)

# HTTP/2 多路复用

详细可参考此文：[https://blog.wangriyu.wang/2018/05-HTTP2.html，作者写的非常详细：）](https://blog.wangriyu.wang/2018/05-HTTP2.html，作者写的非常详细：）)

###### 参考文献：

[1] [http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)

[2] [https://blog.csdn.net/ustccw/article/details/76691248](https://blog.csdn.net/ustccw/article/details/76691248)

[3] [https://www.cnblogs.com/piyeyong/archive/2010/07/02/1770208.html](https://www.cnblogs.com/piyeyong/archive/2010/07/02/1770208.html)

[4] [https://blog.csdn.net/huangyuhuangyu/article/details/78220005](https://blog.csdn.net/huangyuhuangyu/article/details/78220005)

[5] [https://blog.wangriyu.wang/2018/05-HTTP2.html](https://blog.wangriyu.wang/2018/05-HTTP2.html)
