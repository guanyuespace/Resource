---
title: Security-AES
date: 2019-02-13 10:27:32  
categories:  
- java
- security
- encryption

tags:  
- encryption
- decryption
---
# AES
分组加密算法，对称加密算法
AES的区块长度固定为128 比特，密钥长度则可以是128，192或256比特
## 输入
| param | desp |
| --- | --- |
| key  | 128，192或256比特 |
| data | 16个字节共128位的需要被加密或被解密的数据 |
| mode | DES工作方式，加密或者解密 |

初始向量(16个字节)：IV
填充方式or自行填充  
<!-- more -->
```java
KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
keyGenerator.init(128, new SecureRandom(keyBytes));
SecretKey key = keyGenerator.generateKey(); // key长可设为128，192，256位，这里只能设为128

Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
cipher.init(Cipher.ENCRYPT_MODE, key, new IvParameterSpec(iv));//iv 16字节初始向量
byte[] result = cipher.doFinal(content);


Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
cipher.init(Cipher.DECRYPT_MODE, key, new IvParameterSpec(iv));
byte[] result = cipher.doFinal(content);
```
