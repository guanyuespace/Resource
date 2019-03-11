---
title: Security-Base64
date: 2019-02-13 10:53:54
categories:
- java
- security
- encryption
tags:
- encryption
- decryption
---
# Base64  
Base64是网络上最常见的用于传输8Bit字节码的编码方式之一，Base64就是一种基于64个可打印字符来表示二进制数据的方法。
<!-- more -->
## 缘起
**什么情况下需要使用到Base64**   Base64一般用于在HTTP协议下传输二进制数据，由于HTTP协议是文本协议，所以在HTTP协议下传输二进制数据需要将二进制数据转换为字符数据。  
***然而直接转换是不行的。因为网络传输只能传输可打印字符。***  
**什么是可打印字符？**  在ASCII码中规定，0~31、127这33个字符属于控制字符，32~126这95个字符属于可打印字符，也就是说网络传输只能传输这95个字符，不在这个范围内的字符无法传输。
**那么该怎么才能传输其他字符呢？**其中一种方式就是使用Base64。   


## 转码过程
```
3*8=4*6  
内存1个字节占8位  
转前： s 1 3  
先转成ascii：对应 115 49 51  
2进制： 01110011 00110001 00110011  
6个一组（4组） 011100110011000100110011  
然后才有后面的 011100 110011 000100 110011  
然后计算机是8位8位的存数 6不够，自动就补两个高位0了  
所有有了 高位补0  
科学计算器输入 00011100 00110011 00000100 00110011  
得到 28 51 4 51  
查对下照表 c z E z  
```

## 使用`java.util.Base64`
```java
对于标准的Base64：
加密为字符串使用Base64.getEncoder().encodeToString();
加密为字节数组使用Base64.getEncoder().encode();
解密使用Base64.getDecoder().decode();
对于URL安全或MIME的Base64，只需将上述getEncoder()getDecoder()更换为getUrlEncoder()getUrlDecoder()
或getMimeEncoder()和getMimeDecoder()即可。
```
