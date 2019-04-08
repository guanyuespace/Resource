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
- jdk自带的aes加密只支持到128位，更高的256位的加密，需要到oracle官网下载jce包，替换java自带的加密包。

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

### JavaScript实现
>[cryptojs](git@github.com:gwjjeff/cryptojs.git)   
在微信小程序中npm构建出错，解决：将代码复制到本地即可使用
npm install cryptojs 可以在本地使用（node test.js）    
>[crypto-js](git@github.com:brix/crypto-js.git)   
npm install crypto-js

### cryptojs测试demo
```js
var Crypto = require('./cryptojs/cryptojs.js').Crypto;//ok
//解析运动数据
var decryptData = function(data, session, iv,that) {
  var dataBytes = Crypto.util.base64ToBytes(data);
  var sessionBytes = Crypto.util.base64ToBytes(session);
  var ivBytes = Crypto.util.base64ToBytes(iv);


  var mode = new Crypto.mode.CBC(Crypto.pad.pkcs7);
  var decryptData = Crypto.AES.decrypt(data, sessionBytes, {
    mode: mode,
    iv: ivBytes,
    asBytes: true //false: bytesToBase64
  });
  console.log("decrypt: " + Crypto.charenc.UTF8.bytesToString(decryptData));
}
```

#### demo(NodeJs)
```js
var crypto = require('crypto')
function WXBizDataCrypt(sessionKey) {
  this.sessionKey = sessionKey
}

WXBizDataCrypt.prototype.decryptData = function (encryptedData, iv) {
  // base64 decode
  var sessionKey = Buffer.from(this.sessionKey, 'base64')//base64解码
  encryptedData = Buffer.from(encryptedData, 'base64')
  iv = Buffer.from(iv, 'base64')
  try {
     // 解密
    var decipher = crypto.createDecipheriv('aes-128-cbc', sessionKey, iv)
    // 设置自动 padding 为 true，删除填充补位
    decipher.setAutoPadding(true)
    //cipher.update(data, [input_encoding], [output_encoding])
    var decoded = decipher.update(encryptedData, 'binary', 'utf8')//解密
    decoded += decipher.final('utf8')
    decoded = JSON.parse(decoded)
  } catch (err) {
    throw new Error('Illegal Buffer')
  }
  return decoded
}
module.exports = WXBizDataCrypt
```
### Java实现
```java
package test;

import org.bouncycastle.crypto.Digest;
import org.bouncycastle.jce.provider.BouncyCastleProvider;

import javax.crypto.*;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.io.UnsupportedEncodingException;
import java.security.*;
import java.util.Base64;

import static javax.crypto.Cipher.getInstance;

/**
 * 介于java 不支持PKCS7Padding，只支持PKCS5Padding 但是PKCS7Padding 和 PKCS5Padding 没有什么区别
 * 要实现在java端用PKCS7Padding填充，需要用到bouncycastle组件来实现
 */
public class Util {

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


    public static void checkSignature(String session, String rawData, String signature) throws NoSuchAlgorithmException, UnsupportedEncodingException {
        String data = rawData + session;
        System.out.println(data);
        MessageDigest messageDigest = MessageDigest.getInstance("SHA-1");
        byte[] bs = messageDigest.digest(data.getBytes("utf-8"));
        byte2Hex(bs);
        System.out.println(signature);
    }

    private static void byte2Hex(byte[] bs) {
        byte2Hex(bs, 0, bs.length);
    }

    public static void byte2Hex(byte[] bs, int start, int offset) {
        StringBuilder stringBuilder = new StringBuilder(2 * offset);
        for (int i = start; i < start + offset; i++) {
            byte high = (byte) ((byte) (bs[i] >> 4) & 0x0f);
            stringBuilder.append("0123456789abcdef".charAt(high));
            byte low = (byte) ((bs[i] & 0x0f));
            stringBuilder.append("0123456789abcdef".charAt(low));
        }
        System.out.println(stringBuilder.toString());
    }

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
