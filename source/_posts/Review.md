---
title: Review
date: 2019-04-15 09:36:50
categories:
- Life
- Study
tag:
- thinking
---

<div style="background-image:url(http://5b0988e595225.cdn.sohucs.com/images/20180305/fa45432d378e45168ea620718209f14e.gif); background-position:left top; background-repeat: no-repeat; filter: grayscale(1) invert(1); padding:25px;"><!--mix-blend-mode:difference; -->
  <font size="+2">2019.02.11-2019.04.15---&gt; 63天<br/>anxious, worrisome</font><br/>
  <ul>
    <li>茫然无措</li>
    <li>注目人生路口，斩立决断</li>
    <li>当断不断，必受其乱</li>
    <li>断？？？</li>
  </ul>
</div>

<!-- more -->
# Review

## Java SourceCode

### Crypt&Decrypt
#### DES
#### AES
#### RSA
### MessageDigest  
#### MD5
#### SHA-1
### encode/decode(Base64) 

### List&Map
### String&StringBuilder
### Usages about UnSafe

## 微信小程序
### HttpsServer

---
# TODO
## Thread
## ConcurrentMap
## JNI(Native Method)

---
# Java位操作
数据表示： 补码（原码取反加一）   
补码： 使用补码可以将符号位和其它位统一处理；同时，减法也可按照加法来处理。另外，两个用补码表示的数相加时，如果最高位（符号位）有进位，则进位被舍弃。

## 操作

- 左移<<   
所有的数字向左移动对应的位数，**高位移出（舍弃），低位的空位补零。**   

- 右移
  - 有符号>>
  **低位移出（舍弃），高位的空位补符号位。** (即正数补零，负数补1。)   
  - 无符号>>>
  **低位移出（舍弃），高位的空位补零。** (正数运算结果与带符号右移相同，对于负数来说则不同。)
