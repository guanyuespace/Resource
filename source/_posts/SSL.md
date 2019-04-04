---
title: SSL
date: 2019-04-04 09:27:32  
categories:  
- Network
- SSL
tags:  
- Network
- SSL
---

SSL
>Https协议：超文本传输安全协议，简单讲是HTTP的安全版。即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。   

SSL(SecureSocketLayer)是netscape公司提出的主要用于web的安全通信标准.   
TLS(TransportLayerSecurity)是IETF的TLS工作组在SSL3.0基础之上提出的安全通信标准，SSL/TLS提供的安全机制可以保证应用层数据在互联网络传输不被监听,伪造和窜改。   

SSL位与应用层和 TCP/IP之间的一层，数据经过它流出的时候被加密，再往TCP/IP送，而数据从TCP/IP流入之后先进入它这一层被解密，同时它也能够验证网络连接两端的身份

SSL协议包含2个子协议，一个是 *包协议，* 一个是 *握手协议。* 包协议位于握手协议更下一层。

SSL握手过程说简单点就是：**通信双方通过不对称加密算法来协商好一个对称加密算法以及使用的key，然后用这个算法加密以后所有的数据完成应用层协议的数据交换。**  
