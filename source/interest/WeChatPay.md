# 微信支付
>个人无法开通微信支付   
>
> [微信支付](https://pay.weixin.qq.com/wiki/doc/api/index.html)   
> [WeChat Open Platform](https://open.weixin.qq.com/)   
>[【主要API列表】](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_1)   


<!-- TOC -->

- [微信支付](#%E5%BE%AE%E4%BF%A1%E6%94%AF%E4%BB%98)
  - [申请开通微信支付参数](#%E7%94%B3%E8%AF%B7%E5%BC%80%E9%80%9A%E5%BE%AE%E4%BF%A1%E6%94%AF%E4%BB%98%E5%8F%82%E6%95%B0)
  - [规范](#%E8%A7%84%E8%8C%83)
  - [[参数规定](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=4_2)](#%E5%8F%82%E6%95%B0%E8%A7%84%E5%AE%9Ahttpspayweixinqqcomwikidocapiwxawxa_apiphpchapter4_2)
  - [安全规范](#%E5%AE%89%E5%85%A8%E8%A7%84%E8%8C%83)
    - [签名算法](#%E7%AD%BE%E5%90%8D%E7%AE%97%E6%B3%95)
      - [Example](#example)
    - [生成随机数算法](#%E7%94%9F%E6%88%90%E9%9A%8F%E6%9C%BA%E6%95%B0%E7%AE%97%E6%B3%95)
    - [API证书](#api%E8%AF%81%E4%B9%A6)
    - [商户回调API安全](#%E5%95%86%E6%88%B7%E5%9B%9E%E8%B0%83api%E5%AE%89%E5%85%A8)
  - [业务流程](#%E4%B8%9A%E5%8A%A1%E6%B5%81%E7%A8%8B)
    - [业务流程时序图](#%E4%B8%9A%E5%8A%A1%E6%B5%81%E7%A8%8B%E6%97%B6%E5%BA%8F%E5%9B%BE)
  - [开发步骤](#%E5%BC%80%E5%8F%91%E6%AD%A5%E9%AA%A4)
    - [openid: code2session() 获取（登录api--getSession）](#openid-code2session-%E8%8E%B7%E5%8F%96%E7%99%BB%E5%BD%95api--getsession)
    - [调起数据签名](#%E8%B0%83%E8%B5%B7%E6%95%B0%E6%8D%AE%E7%AD%BE%E5%90%8D)
  - [ **小程序调起支付API** ](#%E5%B0%8F%E7%A8%8B%E5%BA%8F%E8%B0%83%E8%B5%B7%E6%94%AF%E4%BB%98api)
    - [ **小程序调起支付数据签名字段列表** ](#%E5%B0%8F%E7%A8%8B%E5%BA%8F%E8%B0%83%E8%B5%B7%E6%94%AF%E4%BB%98%E6%95%B0%E6%8D%AE%E7%AD%BE%E5%90%8D%E5%AD%97%E6%AE%B5%E5%88%97%E8%A1%A8)
    - [ **发起支付** ](#%E5%8F%91%E8%B5%B7%E6%94%AF%E4%BB%98)
      - [回调结果：](#%E5%9B%9E%E8%B0%83%E7%BB%93%E6%9E%9C)
      - [示例代码：](#%E7%A4%BA%E4%BE%8B%E4%BB%A3%E7%A0%81)
  - [最佳实践](#%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)
- [HTTPS服务器配置](#https%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%85%8D%E7%BD%AE)
  - [一、SSL证书申请](#%E4%B8%80ssl%E8%AF%81%E4%B9%A6%E7%94%B3%E8%AF%B7)
  - [二、HTTPS服务器配置](#%E4%BA%8Chttps%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%85%8D%E7%BD%AE)

<!-- /TOC -->
## 申请开通微信支付参数
邮件（申请微信支付开通）中的账户参数与接口API参数对应关系见表：
账户参数说明

|邮件中参数	|API参数名	|详细说明|
| --- | --- |
| APPID |	appid |	appid是微信小程序后台APP的唯一标识，在小程序后台申请小程序账号后，微信会自动分配对应的appid，用于标识该应用。可在小程序--&gt;设置--&gt;开发设置中查看。|
| 微信支付商户号	| mch_id	|商户申请微信支付后，由微信支付分配的商户收款账号。|
| API密钥	| key	| **交易过程生成签名的密钥，仅保留在商户系统和微信支付后台，不会在网络中传播。商户妥善保管该Key，切勿在网络中传输，不能在其他客户端中存储，保证key不会被泄漏。商户可根据邮件提示登录微信商户平台进行设置。也可按一下路径设置：微信商户平台(pay.weixin.qq.com)--&gt;账户设置--&gt;API安全--&gt;密钥设置** |
| Appsecret |	secret	| AppSecret是APPID对应的接口密码，用于获取接口调用凭证时使用。|

## 规范
商户接入微信支付，调用API必须遵循以下规则：

| method | desp |
| --- | --- |
| 传输方式	| 为保证交易安全性，采用HTTPS传输 |
| 提交方式	| 采用POST方法提交 |
| 数据格式	| 提交和返回数据都为XML格式，根节点名为xml |
| 字符编码	| 统一采用UTF-8字符编码 |
| 签名算法	| MD5，后续会兼容SHA1、SHA256、HMAC等。 |
| 签名要求	| 请求和接收数据均需要校验签名，详细方法请参考[安全规范-签名算法]() |
| 证书要求	| **调用申请退款、撤销订单接口需要商户证书** |
| 判断逻辑	| **先判断协议字段返回，再判断业务返回，最后判断交易状态** |

## [参数规定](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=4_2)  

## 安全规范

### 签名算法
签名生成的通用步骤如下：

第一步，设所有发送或者接收到的数据为集合M，将集合M内非空参数值的参数按照参数名ASCII码从小到大排序（字典序），使用URL键值对的格式（即`key1=value1&key2=value2…`）拼接成字符串`stringA`。    
**特别注意以下重要规则：**  
```
◆ 参数名ASCII码从小到大排序（字典序）；  
◆ 如果参数的值为空不参与签名；   
◆ 参数名区分大小写；   
◆ 验证调用返回或微信主动通知签名时，传送的sign参数不参与签名，将生成的签名与该sign值作校验。   
◆ 微信接口可能增加字段，验证签名时必须支持增加的扩展字段   
```
第二步，在`stringA`最后拼接上`key`得到`stringSignTemp`字符串，并对`stringSignTemp`进行`MD5`运算，再将得到的`字符串所有字符转换为大写`，得到`sign`值`signValue`。
```
◆ key设置路径：微信商户平台(pay.weixin.qq.com)-->账户设置-->API安全-->密钥设置
```

即：
```
stringA="param1=value1&param2=value2...";
stringSignTemp= stringA +"&key="+value(key);
signValue=upperCase(MD5(stringSignTemp))
```

#### Example
假设传送的参数如下：
```
appid：	wxd930ea5d5a258f4f
mch_id：	10000100                        //商户id
device_info：	1000                      //
body：	test
nonce_str：	ibuaiVcKdpRxkhJA            //随机数
```
第一步：对参数按照key=value的格式，并按照参数名ASCII字典序排序如下：   
```
stringA="appid=wxd930ea5d5a258f4f&body=test&device_info=1000&mch_id=10000100&nonce_str=ibuaiVcKdpRxkhJA";
```
第二步：拼接API密钥：  
```
stringSignTemp=stringA+"&key=192006250b4c09247ec02edce69f6a2d" //注：key为商户平台设置的密钥key
sign=MD5(stringSignTemp).toUpperCase()="9A0A8659F005D6984697E2CA0A9CF3B7" //注：MD5签名方式
//////////////////////新的签名方式：HMAC-SHA256签名方式
sign=hash_hmac("sha256",stringSignTemp,key).toUpperCase()="6A9AE1657590FD6257D693A078E1C3E4BB6BA4DC30B23E0EE2496E54170DACD6" //注：HMAC-SHA256签名方式
```
最终得到最终发送的数据：   
```xml
<xml>
  <appid>wxd930ea5d5a258f4f</appid>
  <mch_id>10000100</mch_id>
  <device_info>1000</device_info>
  <body>test</body>
  <nonce_str>ibuaiVcKdpRxkhJA</nonce_str>
  <sign>9A0A8659F005D6984697E2CA0A9CF3B7</sign>
</xml>
```

### 生成随机数算法
微信支付API接口协议中包含字段nonce_str，主要保证签名不可预测。我们推荐生成随机数算法如下：调用随机数函数生成，将得到的值转换为字符串。    

### API证书
（1）获取API证书(SSL... 校验身份[什么是api证书？如何升级？](http://kf.qq.com/faq/180824JvUZ3i180824YvMNJj.html))
微信支付接口中，涉及资金回滚的接口会使用到API证书，包括退款、撤销接口。商家在申请微信支付成功后，收到的相应邮件后，可以按照指引下载API证书，也可以按照以下路径下载：微信商户平台(pay.weixin.qq.com)-->账户中心-->账户设置-->API安全 。证书文件说明如下：  
![证书文件](./wechat_pay_cer.jpg)   

（2）使用API证书
```
◆ apiclient_cert.p12是商户证书文件，除PHP外的开发均使用此证书文件。
◆ 商户如果使用.NET环境开发，请确认Framework版本大于2.0，必须在操作系统上双击安装证书apiclient_cert.p12后才能被正常调用。
◆ API证书调用或安装需要使用到密码，该密码的值为微信商户号（mch_id）
```
（3）API证书安全   
1. 证书文件不能放在web服务器虚拟目录，应放在有访问权限控制的目录中，防止被他人下载；
2. 建议将证书文件名改为复杂且不容易猜测的文件名；
3. 商户服务器要做好病毒和木马防护工作，不被非法侵入者窃取证书文件。

### 商户回调API安全
在普通的网络环境下，HTTP请求存在DNS劫持、运营商插入广告、数据被窃取，正常数据被修改等安全风险。商户回调接口使用HTTPS协议可以保证数据传输的安全性。所以微信支付建议商户提供给微信支付的各种回调采用HTTPS协议。请参考：HTTPS搭建指南。


## 业务流程
### 业务流程时序图
小程序支付的交互图如下：
![业务流程时序图](https://pay.weixin.qq.com/wiki/doc/api/img/wxa-7-2.jpg "业务流程时序图")
商户系统和微信支付系统主要交互：    
1、小程序内调用登录接口，获取到用户的openid,api参见公共api[【小程序登录API】](https://mp.weixin.qq.com/debug/wxadoc/dev/api/api-login.html?t=20161122)
2、商户server调用支付统一下单，api参见公共api[【统一下单API】](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_1&index=1)
3、商户server调用再次签名，api参见公共api[【再次签名】](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=7_7&index=3)
4、商户server接收支付通知，api参见公共api[【支付结果通知API】](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_7)
5、商户server查询支付结果，api参见公共api[【查询订单API】](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_2)   


## 开发步骤
如果开发者已做过JSAPI或JSSDK调起微信支付，接入小程序支付非常相似，以下是三种接入方式的对比：

| 对比栏目| JSAPI |	JSSDK|	小程序|   
|--- | ---|
|统一下单|	都需要先获取到Openid，调用相同的API|||
|调起数据签名	|五个字段参与签名(区分大小写)：appId,nonceStr,package,signType,timeStamp |||
|调起支付页面协议	| HTTP或HTTPS	|HTTP或HTTPS|	HTTPS|
|支付目录	|有	|有	|无|
|授权域名	|有	|有	|无|
|回调函数	|有	|success回调	|complete、fail、success回调函数|

程序访问商户服务都是通过HTTPS,开发部署的时候需要安装[HTTPS服务器]()


### openid: code2session() 获取（登录api--getSession）
```json
{
	"data": {
		"session_key": "OpCABq/5O1T9MctGd/fq9Q==",
		"openid": "ofAqB4n1m4IKBnvrixY5qshNZrTg"
	},
	"header": {
		"Content-Type": "text/plain;charset=utf-8"
	},
	"statusCode": 200,
	"cookies": [],
	"errMsg": "request:ok"
}
```
### 调起数据签名
详解[小程序调起支付API-小程序调起支付数据签名字段列表]()    
appid,nonceStr(随机数),package(...),signType(MD5/HMAC-SHA256),timeStamp   

## **小程序调起支付API**
### **小程序调起支付数据签名字段列表**

|字段名	|变量名	|必填	|类型	|示例值	|描述|
|---|---|
|小程序ID	|appId	|是	|String	|wxd678efh567hg6787	|微信分配的小程序ID|
|时间戳	|timeStamp	|是	|String	|1490840662	|时间戳从1970年1月1日00:00:00至今的**秒数**,即当前的时间|
|随机串	|nonceStr	|是	|String	|5K8264ILTKCH16CQ2502SI8ZNMTM67VS	|随机字符串，**不长于32位**。推荐随机数生成算法|
|数据包	|package	|是	|String|	prepay_id=wx2017033010242291fcfe0db70013231072	|统一下单接口返回的prepay_id 参数值，提交格式如：prepay_id=wx2017033010242291fcfe0db70013231072|
|签名方式	|signType	|是	|String	|MD5	|签名类型，默认为MD5，支持HMAC-SHA256和MD5。**注意此处需与统一下单的签名类型一致** |
举例如下:   
```
paySign = MD5(appId=wxd678efh567hg6787&nonceStr=5K8264ILTKCH16CQ2502SI8ZNMTM67VS
              &package=prepay_id=wx2017033010242291fcfe0db70013231072
              &signType=MD5&timeStamp=1490840662&key=qazwsxedcrfvtgbyhnujmikolp111111) = 22D9B4E54AB1950F51E0649E8810ACD6
```
### **发起支付**
调用wx.requestPayment(`OBJECT`)发起微信支付    
`Object`参数说明：

| 参数	| 类型	| 必填	| 说明|
|---|---|
| timeStamp	| String	| 是	| 时间戳从1970年1月1日00:00:00至今的秒数,即当前的时间 |
| nonceStr	| String	| 是	| 随机字符串，长度为32个字符以下。 |
| package	| String	|是	| 统一下单接口返回的 prepay_id 参数值，提交格式如：prepay_id=* |
| signType	| String	| 是	| 签名类型，默认为MD5，支持HMAC-SHA256和MD5。注意此处需与统一下单的签名类型一致 |
| paySign	| String	| 是	| 签名,**ok 上一步结果** |
| success	| Function	| 否	| 接口调用成功的回调函数 |
| fail	| Function	| 否	| 接口调用失败的回调函数 |
| complete	| Function	| 否	| 接口调用结束的回调函数（调用成功、失败都会执行） |

#### 回调结果：

| 回调类型	| errMsg	| 说明 |
|----|----|
| success	| requestPayment:ok	| 调用支付成功 |
| fail	| requestPayment:fail cancel	 | 用户取消支付 |
| fail	 | requestPayment:fail (detail message)	 | 调用支付失败，其中 detail message 为后台返回的详细失败原因 |
#### 示例代码：
```js
wx.requestPayment(
  {
  'timeStamp': '',
  'nonceStr': '',
  'package': '',
  'signType': 'MD5',
  'paySign': '',
  'success':function(res){},
  'fail':function(res){},
  'complete':function(res){}
  });
```


## 最佳实践
[网络设置指引](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=23_2&index=1)
[最佳安全实践](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=23_3&index=2)   

---
# HTTPS服务器配置

## 一、SSL证书申请
1. 确认需要申请证书的域名
2. 生成私钥和csr文件    
在linux机器上执行以下命令生成私钥
```shell
#openssl genrsa -out server.key 2048
```
在linux机器上执行以下命令生成csr文件
```shell
#openssl req -new -key server.key -out certreq.csr
```
以下黑色标识文字仅供参考，请根据商户自己实际情况进行填写
```
Country Name： CN                      //您所在国家的ISO标准代号，中国为CN
State or Province Name：guandong       //您单位所在地省/自治区/直辖市
Locality Name：shenzhen                 //您单位所在地的市/县/区
Organization Name： Tencent Technology (Shenzhen) Company Limited                 //您单位/机构/企业合法的名称
Organizational Unit Name： R&D         //部门名称
Common Name： www.example.com     //通用名，例如：www.itrus.com.cn。此项必须与您访问提供SSL服务的服务器时所应用的域名完全匹配。
Email Address：                          //您的邮件地址，不必输入，直接回车跳过
"extra"attributes                        //以下信息不必输入，回车跳过直到命令执行完毕。
```
执行上面的命令后，在当前目录下即可生成私钥文件server.key和certreq.csr csr文件    

3. 将生成的csr文件提交给第三方证书颁发机构申请对应域名的服务器证书，同时将私钥文件保存好，以免丢失。

4. 证书申请后，证书颁发机构会提供服务器证书内容和两张中级CA证书，请按证书颁发机器说明生成服务器证书，此处假设服务器证书文件名称为server.pem

5. 将生成的私钥文件server.key和服务器证书server.pem拷贝至服务器指定的目录即可进行HTTPS服务器配置

## 二、HTTPS服务器配置
1. Nginx配置
```
server {
listen       443;   #指定ssl监听端口
server_name  www.example.com;
ssl on;    #开启ssl支持
ssl_certificate      /etc/nginx/server.pem;    #指定服务器证书路径
ssl_certificate_key  /etc/nginx/server.key;    #指定私钥证书路径
ssl_session_timeout  5m;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;     #指定SSL服务器端支持的协议版本
ssl_ciphers  ALL：!ADH：!EXPORT56：RC4+RSA：+HIGH：+MEDIUM：+LOW：+SSLv2：+EXP;    #指定加密算法
ssl_prefer_server_ciphers   on;    #在使用SSLv3和TLS协议时指定服务器的加密算法要优先于客户端的加密算法
#以下内容请按域名需要进行配置，此处仅供参考
location / {
return 444;
}
}
```
2. 其它web服务器配置
请参考文档：[《服务器证书配置指南》](http://www.itrus.cn/html/fuwuyuzhichi/fuwuqizhengshuanzhuangpeizhizhinan)

三、相关事项
1. 证书颁发机构
推荐天威诚信，具体请见：[链接](http://www.itrus.com.cn)

2. 参考文档    
[《ngx_http_ssl_module》](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_prefer_server_ciphers)   
[《nginx配置HTTPS服务器》](http://nginx.org/cn/docs/http/configuring_https_servers.html)   
[《服务器证书配置指南》](http://www.itrus.cn/html/fuwuyuzhichi/fuwuqizhengshuanzhuangpeizhizhinan)

3. 常见问题   
  1. 证书受信任的问题
  部分国内签发的SSL证书，在Android上不受信任，推荐GeoTrust；
  2. 如果页面有动静分离，静态资源使用独立域名的话，也需要为该域名申请证书；
  3. android低版本不支持SNI扩展，受此限制，一台服务器只能部署一个数字证书；
