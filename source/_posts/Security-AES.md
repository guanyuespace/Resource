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

- AES算法中不支持PKCS7Padding，只支持PKCS5Padding 但是PKCS7Padding 和 PKCS5Padding 没有什么区别要实现在java端用PKCS7Padding填充，需要用到bouncycastle组件来实现

```
// https://mvnrepository.com/artifact/org.bouncycastle/bcprov-jdk16
compile group: 'org.bouncycastle', name: 'bcprov-jdk16', version: '1.46'
```

## 微信小程序数据解密

```java
static {
     Security.addProvider(new BouncyCastleProvider());
 }
 public static String decryptData(String data, String session_key, String iv) throws NoSuchPaddingException, InvalidKeyException, NoSuchAlgorithmException, IllegalBlockSizeException, BadPaddingException, InvalidAlgorithmParameterException, UnsupportedEncodingException {
     byte[] content = base64DecodeData(data);
     byte[] keyBytes = base64DecodeData(session_key);
     byte[] ivBytes = base64DecodeData(iv);
     return new String(AESDecodeData(content, keyBytes, ivBytes), "utf-8");
 }

 private static byte[] AESDecodeData(byte[] content, byte[] keyBytes, byte[] iv) throws NoSuchAlgorithmException, NoSuchPaddingException, BadPaddingException, IllegalBlockSizeException, InvalidAlgorithmParameterException, InvalidKeyException {
     Key key = new SecretKeySpec(keyBytes, "AES");

     Cipher cipher = getInstance("AES/CBC/PKCS7Padding");
     cipher.init(Cipher.DECRYPT_MODE, key, new IvParameterSpec(iv));
     return cipher.doFinal(content);
 }

 private static byte[] base64DecodeData(String s) {
     Base64.Decoder decoder = Base64.getDecoder();
     return decoder.decode(s);
 }
```






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
