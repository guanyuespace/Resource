---
title: SSL&Https Server
date: 2019-04-04 09:27:32  
categories:  
- Network
- SSL
tags:  
- Network
- SSL
---
# SSL  
>Https协议：超文本传输安全协议，简单讲是HTTP的安全版。即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。   

SSL(SecureSocketLayer)是netscape公司提出的主要用于web的安全通信标准.   
TLS(TransportLayerSecurity)是IETF的TLS工作组在SSL3.0基础之上提出的安全通信标准，SSL/TLS提供的安全机制可以保证应用层数据在互联网络传输不被监听,伪造和窜改。   

SSL位与应用层和 TCP/IP之间的一层，数据经过它流出的时候被加密，再往TCP/IP送，而数据从TCP/IP流入之后先进入它这一层被解密，同时它也能够验证网络连接两端的身份

SSL协议包含2个子协议，一个是 *包协议，* 一个是 *握手协议。* 包协议位于握手协议更下一层。

SSL握手过程说简单点就是：**通信双方通过不对称加密算法来协商好一个对称加密算法以及使用的key，然后用这个算法加密以后所有的数据完成应用层协议的数据交换。**  
<!-- more -->

## Https服务器
>利用Nodejs搭建简易服务器

HTTPS 区别于 HTTP，它多了加密(encryption)，认证(verification)，鉴定(identification)。它的安全源自非对称加密以及第三方的 CA 认证。   

```js
var express = require('express');
var https = require('https');
var http = require('http');
var fs = require('fs');
var app = express();

var options = {
  key:fs.readFileSync('./CA/server.key'),
  cert:fs.readFileSync('./CA/server.crt')
};

app.get("/getSession",(req,res)=>{
  res.send("Hello Here");
});

http.createServer(app).listen(80);
https.createServer(options,app).listen(443);
```

![微信小程序服务请求](./https.jpg "微信小程序服务请求")
![服务器返回结果](./https_res.jpg "服务器返回结果")

### 证书创建
>[HTTPS证书生成原理和部署细节](https://www.cnblogs.com/liyulong1982/p/6106129.html)   


### Https运作
![](http://www.barretlee.com/blogimgs/2015/10/20151001_b347f684.jpg )    
如上图所示，简述如下：   
- 客户端生成一个随机数 random-client，传到服务器端（Say Hello)
- 服务器端生成一个随机数 random-server，和着公钥，一起回馈给客户端（I got it)
- 客户端收到的东西原封不动，加上 premaster secret（通过 random-client、random-server 经过一定算法生成的东西），再一次送给服务器端，这次传过去的东西会使用公钥加密
- 服务器端先使用私钥解密，拿到 premaster secret，此时客户端和服务器端都拥有了三个要素：random-client、random-server 和 premaster secret
- 此时安全通道已经建立，以后的交流都会校检上面的三个要素通过算法算出的 session key


#### CA 数字证书认证中心    
如果网站只靠上图运作，可能会被中间人攻击，试想一下，在客户端和服务端中间有一个中间人，两者之间的传输对中间人来说是透明的，那么中间人完全可以获取两端之间的任何数据，然后将数据原封不动的转发给两端，由于中间人也拿到了三要素和公钥，它照样可以解密传输内容，并且还可以篡改内容。  

为了确保我们的数据安全，我们还需要一个 CA 数字证书。HTTPS的传输采用的是非对称加密，一组非对称加密密钥包含公钥和私钥，通过公钥加密的内容只有私钥能够解密。上面我们看到，整个传输过程，服务器端是没有透露私钥的。而 CA 数字认证涉及到私钥，整个过程比较复杂，我也没有很深入的了解，后续有详细了解之后再补充下。   

CA 认证分为三类：DV ( domain validation)，OV ( organization validation)，EV ( extended validation)，证书申请难度从前往后递增，貌似 EV 这种不仅仅是有钱就可以申请的。    

对于一般的小型网站尤其是博客，可以使用自签名证书来构建安全网络，所谓自签名证书，就是自己扮演 CA 机构，自己给自己的服务器颁发证书。    

#### 生成密钥、证书

1. 为服务器端和客户端准备公钥、私钥

 ```shell
# 生成服务器端私钥
openssl genrsa -out server.key 1024
# 生成服务器端公钥
openssl rsa -in server.key -pubout -out server.pem
# 生成客户端私钥
openssl genrsa -out client.key 1024
# 生成客户端公钥
openssl rsa -in client.key -pubout -out client.pem
 ```


2. 生成 CA 证书   

 ```shell
# 生成 CA 私钥
openssl genrsa -out ca.key 1024
# X.509 Certificate Signing Request (CSR) Management.
openssl req -new -key ca.key -out ca.csr
# X.509 Certificate Data Management.
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
 ```
在执行第二步时会出现：
 ```shell
➜ keys openssl req -new -key ca.key -out ca.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Zhejiang
Locality Name (eg, city) []:Hangzhou
Organization Name (eg, company) [Internet Widgits Pty Ltd]:My CA
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:localhost
Email Address []:
 ```
 注意，这里的 *Organization Name* (eg, company) [Internet Widgits Pty Ltd]: 后面生成客户端和服务器端证书的时候也需要填写，**不要写成一样的！！！** 可以随意写如：My CA, My Server, My Client。   

 然后 Common Name (e.g. server FQDN or YOUR name) []: 这一项，是最后可以访问的域名，我这里为了方便测试，写成 localhost，如果是为了给我的网站生成证书，需要写成 barretlee.com。    

3. 生成服务器端证书和客户端证书
 ```shell
# 服务器端需要向 CA 机构申请签名证书，在申请签名证书之前依然是创建自己的 CSR 文件
openssl req -new -key server.key -out server.csr
# 向自己的 CA 机构申请证书，签名过程需要 CA 的证书和私钥参与，最终颁发一个带有 CA 签名的证书
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -out server.crt
# client 端
openssl req -new -key client.key -out client.csr
# client 端到 CA 签名
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in client.csr -out client.crt
 ```
此时，ok...

#### HTTPS本地测试
服务器代码：
```js
// file https-server.js
var https = require('https');
var fs = require('fs');
var options = {
    key: fs.readFileSync('./keys/server.key'),
    cert: fs.readFileSync('./keys/server.crt')
};
https.createServer(options, function(req, res) {
res.writeHead(200);
res.end('hello world');
}).listen(8000);
```
https 服务器，options 将私钥和证书带上。

客户端代码
```js
// file https-client.js
var https = require('https');
var fs = require('fs');

var options = {
    hostname: "localhost",
    port: 8000,
    path: '/',
    methed: 'GET',
    key: fs.readFileSync('./keys/client.key'),
    cert: fs.readFileSync('./keys/client.crt'),
    ca: [fs.readFileSync('./keys/ca.crt')]
};

options.agent = new https.Agent(options);

var req = https.request(options, function(res) {
    res.setEncoding('utf-8');
    res.on('data', function(d) {
        console.log(d);
    });
});
req.end();

req.on('error', function(e) {
console.log(e);
});
```
