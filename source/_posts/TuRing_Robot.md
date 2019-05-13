---
title: 图灵机器人接入
date: 2019-05-13 10:49:06
categories:
- WeChat
- Relax
tags:
- WeChat
- Relax
---

# TuRing
>网页版微信登陆扩展     
>[网页版微信测试](https://github.com/guanyuespace/Web_WeChat)   
<!-- more -->
## 使用说明

### Request
UTF-8（调用图灵API的各个环节的编码方式均为UTF-8）      
POST: http://openapi.tuling123.com/openapi/api/v2    
req_str:  
```json
{
	"reqType":0,/*请求的数据类型,0:文本数据;1:图片;2:音频*/
    "perception": {/*	输入信息*/
        "inputText": {/*文本信息*/
            "text": "附近的酒店"/*1-128字符*/
        },
        "inputImage": {/*图片信息*/
            "url": "imageUrl"
        },
        "inputMedia":{/*音频信息*/
             "url": "voiceUrl"
        },
        "selfInfo": {
            "location": {
                "city": "北京",
                "province": "北京",
                "street": "信息路"
            }
        }
    },
    "userInfo": {/*用户参数*/
        "apiKey": "",/*机器人标识32位*/
        "userId": "",/*用户标识1-32位*/
        "groupId":"",/*群聊唯一标识,长度小于等于64位*/
        "userIdName":""/*群内用户昵称,长度小于等于64位*/
    }
}
```

### Response

```json
{
   "intent": {/*请求意图*/
       "code": 10005,/*输出功能code*/
       "intentName": "",/*意图名称*/
       "actionName": "",/*意图动作名称*/
       "parameters": {
           "nearby_place": "酒店"/*功能相关参数*/
       }
   },
   "results": [/*输出结果集*/
       {
         "groupType": 1,/*‘组’编号:0为独立输出，大于0时可能包含同组相关内容 (如：音频与文本为一组时说明内容一致)异常返回码*/
           "resultType": "url",/*输出类型*/
           "values": {/*输出值*/
               "url": "http://m.elong.com/hotel/0101/nlist/#indate=2016-12-10&outdate=2016-12-11&keywords=%E4%BF%A1%E6%81%AF%E8%B7%AF"
           }
       },
       {
         "groupType": 1,
           "resultType": "text",
           "values": {
               "text": "亲，已帮你找到相关酒店信息"
           }
       }
   ]
}
```

#### 异常返回码
异常返回格式
```json
{
  "intent":
  {
    "code":5000
  }
}
```

**异常返回说明**

|异常码	|说明|
|---|---|
|5000	| 无解析结果 |
|6000	| 暂不支持该功能 |
|4000	| 请求参数格式错误 |
|4001	| 加密方式错误 |
|4002	| 无功能权限 |
|4003	| 该apikey没有可用请求次数 |
|4005	| 无功能权限 |
|4007	| apikey不合法 |
|4100	| userid获取失败 |
|4200	| 上传格式错误 |
|4300	| 批量操作超过限制 |
|4400	| 没有上传合法userid |
|4500	| userid申请个数超过限制 |
|4600	| 输入内容为空 |
|4602	| 输入文本内容超长(上限150) |
|7002	| 上传信息失败 |
|8008	| 服务器错误 |
|0	| 上传成功 |

## 使用数据加密
>AES数据加密PCS5Padding,128,iv="00000..."

待加密数据同上，简记为`paramStr`    

### 加密密钥`key`

1. MD5HexStr =  HexStr(MD5(secret+timestamp+apiKey))
2. key = MD5(MD5HexStr.getBytes("utf-8"))  <!-- 骚操作！！！ -->

### 加密
Mode: AES/CBC/PKCS5Padding
Key: new SecretKeySpec(key, "AES");

AES_Encrypt(paramStr)

### 样例

```java
/**
 * 文本信息处理
 *
 * @param paramStr
 */
public TuRingRequestEncrypted(String paramStr) {
    TuRingRequest tuRingRequest = new TuRingRequest(0);
    tuRingRequest.setUserInfo(new UserInfo());
    tuRingRequest.setPerception(new Perception(new Perception.InputText(paramStr), new Perception.SelfInfo(new Perception.SelfInfo.Location("深圳"))));//数据构造
    this.paramStr = JSON.toJSONString(tuRingRequest);//待加密数据
    this.timestamp = "" + System.currentTimeMillis();
    this.md5Key = processKey(secret, timestamp, apiKey);
    this.data = AES_Encrypted(this.paramStr, md5Key);
}

private String AES_Encrypted(String paramStr, String md5Key) {
    Cipher cipher = null;
    try {
        MessageDigest messageDigest = MessageDigest.getInstance("MD5");
        messageDigest.update(md5Key.getBytes("utf-8"));
        byte[] tmp = messageDigest.digest();
        Key key = new SecretKeySpec(tmp, "AES");
        cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
        cipher.init(Cipher.ENCRYPT_MODE, key, new IvParameterSpec(new byte[]{0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}));
        byte[] encryptedData = cipher.doFinal(paramStr.getBytes("utf-8"));
        return Base64.getEncoder().encodeToString(encryptedData);
    } catch (NoSuchAlgorithmException | NoSuchPaddingException | InvalidKeyException | InvalidAlgorithmParameterException | UnsupportedEncodingException | IllegalBlockSizeException | BadPaddingException e) {
        e.printStackTrace();
    }
    return null;
}

private String processKey(String secret, String timestamp, String apiKey) {
    try {
        MessageDigest messageDigest = MessageDigest.getInstance("MD5");
        return byte2HexStr(messageDigest.digest((secret + timestamp + apiKey).getBytes("utf-8")));
    } catch (NoSuchAlgorithmException | UnsupportedEncodingException e) {
        e.printStackTrace();
    }
    return null;
}

private String byte2HexStr(byte[] digest) {
    String str = "0123456789abcdef";
    StringBuilder stringBuilder = new StringBuilder(2 * digest.length);
    for (int i = 0; i < digest.length; i++) {
        byte high = (byte) ((byte) 0x0f & (digest[i] >> 4));
        stringBuilder.append(str.charAt(high));
        byte low = (byte) (digest[i] & 0x0f);
        stringBuilder.append(str.charAt(low));
    }
    return stringBuilder.toString();
}
```
