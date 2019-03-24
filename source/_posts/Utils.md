---
title: Utils
date: 2019-02-01 11:31:46
categories:
- Java
- Utils
tags:
- Java
- Utils
---
<!-- TOC -->

- [byte&String](#bytestring)
  - [byte2HexString](#byte2hexstring)
  - [hexString2Byte](#hexstring2byte)

<!-- /TOC -->
# byte&String
## byte2HexString
```java
public void byte2HexString(byte[] bytes){
    if (bytes == null) return null;
    StringBuilder ret = new StringBuilder(2*bytes.length);
    for (int i = 0 ; i < bytes.length ; i++) {
        int b;
        b = 0x0f & (bytes[i] >> 4);
        ret.append("0123456789abcdef".charAt(b));
        b = 0x0f & bytes[i];
        ret.append("0123456789abcdef".charAt(b));
    }
    return ret.toString();
}
```

## hexString2Byte
```java
public void hexString2Byte(String str){
  if (str==null||str.length()&0x1==1) return null;
  str=str.toLowerCase();
  byte[] ret=new byte[str.length()/2];
  for(int i=0;i<str.length;i++){
    byte high = (byte) "0123456789abcdef".indexOf(str.charAt(i));
    byte low = (byte) "0123456789abcdef".indexOf(str.charAt(++i));
    ret[i/2] = (byte) ((byte)(high<<4) | low );
  }
  return ret;
}
```
# base64&Img
> just base64 encode/decode

## Base64toImg
```java
public static void parseImg(String path, String imgStr) {
    Base64.Decoder decoder = Base64.getDecoder();
    byte[] retbytes = decoder.decode(imgStr);
    try {
        Files.write(Paths.get(path), retbytes);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
