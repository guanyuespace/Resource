---
title: node-crypto
date: 2019-04-04 09:37:32  
categories:  
- Node
- Crypto
tags:  
- Node
- Crypto
---
# NodeJs
>加解密工具
const crypto=require("crypto")    
[浅谈nodejs中的Crypto模块](https://blog.csdn.net/XSFD_DFSX/article/details/81737130)   
[\[转载\]加密算法库Crypto——nodejs中间件系列](https://www.cnblogs.com/sunws/p/4783358.html)



## 哈希&散列

```js
const crypto=require("crypto");

var md5 = crypto.createHash('md5');
md5.update('foo');
md5.digest();
md5.digest('hex');// Error [ERR_CRYPTO_HASH_FINALIZED]: Digest already called
```
一旦md5.digest();这个方法被调用了，hash 对象就被清空了是不能被重用的。
```js
var sha1 = crypto.createHash('sha1');
sha1.update('foo');
sha1.update('bar');
sha1.digest('hex'); //8843d7f92416211de9ebb963ff4ce28125932878
var sha1 = crypto.createHash('sha1');
sha1.update('foobar');
sha1.digest('hex') //8843d7f92416211de9ebb963ff4ce28125932878
```
这2次sha1加密结果是一样的，也就是说hash.update()方法就是将字符串相加，然后在hash.digest()将字符串加密返回

### HMAC   
HMAC全名是 *keyed-Hash Message Authentication Code* ，中文直译就是密钥相关的哈希运算消息认证码，**HMAC运算利用哈希算法，以一个密钥和一个消息为输入，生成一个加密串作为输出。**    
***HMAC可以有效防止一些类似md5的彩虹表等攻击，比如一些常见的密码直接MD5存入数据库的，可能被反向破解。***
crypto.createHmac(algorithm, key)
这个方法返回和createHash一样，返回一个HMAC的实例，有update和digest方法。
```js
var crypto = require('crypto');
var fs = require('fs');

var pem = fs.readFileSync('key.pem');
var key = pem.toString('ascii');

var hmac = crypto.createHmac('sha1', key);

hmac.update('foo');
{}
hmac.digest('hex');
'7b058f2f33ca28da3ff3c6506c978825718c7d42'
```
先通过 fs.readFileSync 方法读取了key.pem密钥，然后将它转为ascii码，最后通过 createHmac('sha1', key) 方法获得HMAC实例，然后执行update和digest，生成一串密钥字符串。
由于key的不同，所以同样的字符串'foo'经过hmac加密后生成的16进制字符串也是不同的，从而更加保障了数据的安全性。  
key.pem: 利用opensll命令来创建一个key.pem
```
# openssl genrsa  -out server.pem 1024
<!-- Generating RSA private key, 1024 bit long modulus -->
```

## 加解密算法
```js
var crypto = require('crypto');
var fs = require('fs');

var pem = fs.readFileSync('key.pem');
var key = pem.toString('ascii');

var cipher = crypto.createCipher('blowfish', key);

cipher.update(new Buffer(4), 'binary', 'hex');
”
cipher.update(new Buffer(4), 'binary', 'hex');
'ff57e5f742689c85'
cipher.update(new Buffer(4), 'binary', 'hex');
”
cipher.final('hex')
'96576b47fe130547'
```
读取之前的key，然后利用 blowfish 加密算法生成 cipher 实例，接着update内容到cipher实例，最后通过cipher.final()方法输出加密串。     
其中有几个方法我们要看下api的解释    
```js
crypto.createCipher(algorithm, password)    
crypto.createCipheriv(algorithm, key, iv)   
```     
上面这2个方法都返回cipher实例，第一个参数 algorithm 表示用何种加密算法，可以利用 openssl list-cipher-algorithms 命令来查看你的系统支持哪些加密算法。password和key, iv表示密钥，即利用何种密钥加密，password是用来派生key和iv的，key的话是算法原生的key，iv表示初始化向量。    
```js
cipher.update(data, [input_encoding], [output_encoding])    
```
往cipher实例中添加数据，第一个参数是填充的数据，第二个参数表示传入数据的格式，可以是'utf8', 'ascii' 或 'binary'，默认是 'binary'。第三个参数是返回block的数据格式。     
注意:这里我们update了 new Buffer(4),表示通过随机内存中的4byte字节的内容填充进去。  
*为什么第一次update没有block返回呢?* 因为4byte不够生成一个block，所以这点我们要注意下。
最后我们通过final方法和之前digest方法一样，生成加密过后的串。
```js
decipher.setAutoPadding(auto_padding=true)
```
如果这些加密块不是使用标准的填充块的话，你可以把自动填充关闭。   
这么做是为了防止执行 decipher.final()的时候监察和去除标准填充块，从而可能出错，一般这个方法不会去用。必须在执行update之前执行它。   
***demo***
```js
var crypto = require('crypto');
var cipher = crypto.createCipher('aes-256-cbc','InmbuvP6Z8')
var text = "123|123123123123123";
var crypted = cipher.update(text,'utf8','hex')
crypted += cipher.final('hex')
var decipher = crypto.createDecipher('aes-256-cbc','InmbuvP6Z8')
var dec = decipher.update(crypted,'hex','utf8')
dec += decipher.final('utf8')
```
ok。。。

```js
var crypto = require('crypto');
var fs = require('fs');

var pem = fs.readFileSync('key.pem');
var key = pem.toString('ascii');

var plaintext = Buffer.from('abcdefghijklmnopqrstuv');
var encrypted = “”;
var cipher = crypto.createCipher('blowfish', key);
..
encrypted += cipher.update(plaintext, 'binary', 'hex');
encrypted += cipher.final('hex');

var decrypted = “”;
var decipher = crypto.createDecipher('blowfish', key);
decrypted += decipher.update(encrypted, 'hex', 'binary');
decrypted += decipher.final('binary');

var output = new Buffer(decrypted);

output

plaintext
```
最后我们看下签名和验证 Class: Signer 和 Class: Verify

先通过openssl命令生成公钥：
```shell
# openssl req -key server.pem -new -x509 -out cert.pem
```
还记得我们之前说的不对称加密算法么，这里我们就利用私钥和公钥来做个简单的例子
```js
var crypto = require('crypto');
var fs = require('fs');

var privatePem = fs.readFileSync('server.pem');
var publicPem = fs.readFileSync('cert.pem');
var key = privatePem.toString();
var pubkey = publicPem.toString();

var data = “abcdef”

var sign = crypto.createSign('RSA-SHA256');
sign.update(data);
{}
var sig = sign.sign(key, 'hex');

var verify = crypto.createVerify('RSA-SHA256');
verify.update(data);
{}
verify.update(data);
{}
verify.verify(pubkey, sig, 'hex');
1
```
首先通过，crypto.createVerify(algorithm)和crypto.createSign(algorithm)方法生成实例，然后利用update方法更新数据，最后利用key（私钥）生成签名，同样的验证也是如此，最后通过 verify.verify(pubkey, sig, 'hex'); 函数签名。
