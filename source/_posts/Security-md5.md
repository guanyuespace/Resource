---
title: Security-md5
date: 2019-02-13 11:23:42
categories:
- java
- security
- encryption
tags:
- encryption
- decryption
---
# 哈希算法-MD5,SHA  
Hash，就是把任意长度的输入（又叫做预映射pre-image）通过散列算法变换成固定长度的输出，该输出就是散列值。这种转换是一种压缩映射，也就是，散列值的空间通常远小于输入的空间。**简单的说就是一种将任意长度的消息压缩到某一固定长度的消息摘要的函数。**  
<!-- more -->
## 输出
128位（16字节）的散列值（hash value）

## 使用
```java
//MD5,SHA1等
MessageDigest md = MessageDigest.getInstance("MD5");
md.digest(pass.getBytes("utf-8"));

MessageDigest md5 = MessageDigest.getInstance("MD5");
md5.update(pass.getBytes("utf-8"));
md.digest();
```
