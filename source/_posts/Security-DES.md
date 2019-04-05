---
title: Security-DES
date: 2019-02-12 16:29:17
categories:
- java
- security
- encryption
tags:
- encryption
- decryption
---
<!-- TOC -->

- [DES](#des)
  - [输入](#%E8%BE%93%E5%85%A5)
  - [使用`javax.crypto`](#%E4%BD%BF%E7%94%A8javaxcrypto)
    - [密钥生成](#%E5%AF%86%E9%92%A5%E7%94%9F%E6%88%90)
    - [分组加密-加密模式](#%E5%88%86%E7%BB%84%E5%8A%A0%E5%AF%86-%E5%8A%A0%E5%AF%86%E6%A8%A1%E5%BC%8F)

<!-- /TOC -->
# DES
分组加密算法，对称加密算法
## 输入
| param | desp |
| --- | --- |
| key  | 8个字节共64位的工作密钥(56位密钥+8位奇偶校验位) |
| data | 8个字节共64位的需要被加密或被解密的数据 |
| mode | DES工作方式，加密或者解密 |

初始向量(8个字节)：IV
填充方式or自行填充
<!-- more -->
## 使用`javax.crypto`

### 密钥生成
四种密钥生成类： **KeyGenerator，KeyPairGenerator，KeyFactory，SecretKeyFactory**      
对密钥进行加工处理  
- 区别：   
 根据 Oracle 的 Standard Algorithm Name Documentation 提供的说明：  
 **KeyGenerator**和**SecretKeyFactory，**都是javax.crypto包的，
  生成的key主要是提供给AES，DES，3DES，MD5，SHA1等**对称**和**单向**加密算法。

 **KeyPairGenerator**和**KeyFactory**，都是java.security包的， 生成的key主要是提供给DSA，RSA， EC等**非对称**加密算法。


### 分组加密-加密模式
>参考：[对称加密算法常用的五种分组模式](https://blog.csdn.net/weixin_42940826/article/details/83687007)  
[分组加密的四种模式](https://blog.csdn.net/includeiostream123/article/details/51066799)

分组加密的四种模式：ECB、CBC、CFB、OFB  
1. ECB(Electronic Code Book)/电码本模式
![电码本模式-加密](https://img-blog.csdn.net/20160405180712271?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center "电码本模式-加密")    
![电码本模式-解密](https://img-blog.csdn.net/20160405180727459?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center "电码本模式-解密")    
DES ECB（电子密本方式）其实非常简单，就是将数据按照8个字节一段进行DES加密或解密得到一段8个字节的密文或者明文，最后一段不足8个字节，按照需求补足8个字节进行计算，之后按照顺序将计算所得的数据连在一起即可，各段数据之间互不影响。
特点：    
 1. 简单，有利于并行计算，误差不会被传送；  
 2. **不能隐藏明文的模式；**  
 repetitions in message may show in cipher text/在密文中出现明文消息的重复 
 3. **可能对明文进行主动攻击；**    
 加密消息块相互独立成为被攻击的弱点/weakness due to encrypted message blocks being independent

2. CBC(Cipher Block Chaining)/密文分组链接方式(**双方约定初始向量：IV**)
![CBC-加密](https://img-blog.csdn.net/20160405180943506 "CBC-加密")  
![CBC-解密](https://img-blog.csdn.net/20160405180951038 "CBC-解密")
特点：  
 1. 不容易主动攻击,安全性好于ECB,适合传输长度长的报文,是SSL、IPSec的标准。  
 each ciphertext block depends on all message blocks/每个密文块依赖于所有的信息块  
 thus a change in the message affects all ciphertext blocks/明文消息中一个改变会影响所有密文块  
 2. need Initial Vector (IV) known to sender & receiver/发送方和接收方都需要知道初始化向量     
 3. 加密过程是串行的，无法被并行化(在解密时，从两个邻接的密文块中即可得到一个明文块。因此，解密过程可以被并行化)。

---
CFB模式与OFB模式不需要填充

---

3. Cipher Feedback (CFB)/密文反馈模式(**双方约定初始向量：IV---Shift register移位寄存器，每次加密位数：移位寄存器位数**)  
![CFB-加密](https://img-blog.csdn.net/20160405181228492 "CFB-加密")    
![CFB-解密]( "CFB-解密")      
密文反馈（CFB，Cipher feedback）模式类似于CBC，可以将块密码变为自同步的流密码；工作过程亦非常相似，CFB的解密过程几乎就是颠倒的CBC的加密过程。
加密过程：**需要使用一个与块的大小相同的移位寄存器，并用IV将寄存器初始化。**然后，将寄存器内容使用块密码加密，然后将结果的最高x位与明文的x进行异或，以产生密文的x位。下一步将生成的x位密文移入寄存器中，并对下面的x位明文重复这一过程。解密过程与加密过程相似，以IV开始，对寄存器加密，将结果的高x与密文异或，产生x位明文，再将密文的下面x位移入寄存器。
与CBC相似，**明文的改变会影响接下来所有的密文**，因此**加密过程不能并行化；**而同样的，与CBC类似**，解密过程是可以并行化的。**

4. Output Feedback (OFB)/输出反馈模式
![OFB-加密](https://img-blog.csdn.net/20160405181324195 "OFB-加密")   
输出反馈模式（Output feedback, OFB）可以将块密码变成同步的流密码。它产生密钥流的块，然后将其与明文块进行异或，得到密文。与其它流密码一样，密文中一个位的翻转会使明文中同样位置的位也产生翻转。这种特性使得许多错误校正码，例如奇偶校验位，即使在加密前计算而在加密后进行校验也可以得出正确结果。    
每个使用OFB的输出块与其前面所有的输出块相关，因此**不能并行化处理(加密&解密)**。然而，由于明文和密文只在最终的异或过程中使用，因此可以事先对IV进行加密，最后并行的将明文或密文进行并行的异或处理。
可以利用输入全0的CBC模式产生OFB模式的密钥流。这种方法十分实用，因为可以利用快速的CBC硬件实现来加速OFB模式的加密过程。

---

> 第5种加密方式  **CTR - CounTeR, 计数器模式（重点，推荐使用）**   
> 特点: 密文没有规律, 明文分组是和一个数据流进行的按位异或操作, 最终生成了密文  
> **不需要初始化向量 **   
> **不需要填充 **     
> 这里我们有必要给出CTR模式的加密流程，因为CTR模式的解密和加密是一模一样的过程，在程序实现中也是可逆的   
> ![CTR加密](https://img-blog.csdnimg.cn/2018110314132228.png "CTR加密")  
> CTR加密即解密，解密即加密，且各分组之间是独立的，可以并发完成，效率高。  

小节：  
![加密模式比较](https://img-blog.csdnimg.cn/20181103140304845.png "加密模式比较")


### 数据填充
- NoPadding
明文数据必须为8字节的整数倍

- PKCS7Padding（PKCS5Padding）
为.NET和JAVA的默认填充方式，对加密数据字节长度对8取余为r，如r大于0，则补8-r个字节，字节为8-r的值；如果r等于0，则补8个字节8。比如：  
加密字符串为为AA，则补位为AA666666;加密字符串为BBBBB，则补位为BBBBB333；加密字符串为CCCCCCCC，则补位为CCCCCCCC88888888。

- AES算法中不支持PKCS7Padding，只支持PKCS5Padding 但是PKCS7Padding 和 PKCS5Padding 没有什么区别要实现在java端用PKCS7Padding填充，需要用到bouncycastle组件来实现 

---

## Java 示例
```java
密钥生成：
// 第一种，Factory
DESKeySpec keySpec = new DESKeySpec(keyBytes);
SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("DES");
SecretKey key1 = keyFactory.generateSecret(keySpec);

// 第二种, Generator
KeyGenerator keyGenerator = KeyGenerator.getInstance("DES");
keyGenerator.init(56, new SecureRandom(keyBytes));// key为8个字节，实际用了56位； 后面随机数用key作为种子seed生成
SecretKey key2 = keyGenerator.generateKey();

// 第三种， SecretKeySpec
//// 输入必须为8字节，原始密码未作替换！！！
SecretKey key3 = new SecretKeySpec(keyBytes, "DES");// SecretKeySpec类同时实现了Key和KeySpec接口

加密：
DESKeySpec keySpec = new DESKeySpec(keyBytes);
SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("DES");
SecretKey key = keyFactory.generateSecret(keySpec);//生成密钥
//DES CBC（密文分组连接）模式   PKCS5Padding
Cipher cipher = Cipher.getInstance("DES/CBC/PKCS5Padding");
cipher.init(Cipher.ENCRYPT_MODE, key, new IvParameterSpec("01234567"));
byte[] result = cipher.doFinal(content);

解密：
DESedeKeySpec keySpec = new DESedeKeySpec(key.getBytes());
SecretKeyFactory keyFactory = SecretKeyFactory.getInstance("DES");
SecretKey secretKey = keyFactory.generateSecret(keySpec);

Cipher cipher = Cipher.getInstance("DES/CBC/PKCS5Padding");
cipher.init(Cipher.DECRYPT_MODE, secretKey, new IvParameterSpec("01234567"));
byte[] result = cipher.doFinal(res.getBytes());
```

---
later:  
&nbsp;&nbsp;&nbsp;&nbsp;[DES算法原理完整版](https://blog.csdn.net/qq_27570955/article/details/52442092)
