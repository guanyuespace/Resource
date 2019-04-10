---
title: Keytool和OpenSSL生成和签发数字证书
date: 2019-04-08 16:45:30
categories:
- Network
- SSL
- Certificate
tags:
- Certificate
---
# openssl自签证书

<!-- more -->
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

---

# Keytool和OpenSSL生成和签发数字证书   
>[Keytool和OpenSSL生成和签发数字证书](https://blog.csdn.net/naioonai/article/details/81045780)

生成数字签名证书具体操作          

1. 创建CA的自签名证书，做RootCA使用     
```shell
openssl req -new -x509 -keyout test_ca.key -out test_ca.cer -days 3650
```   
![创建CA的自签名证书](https://img-blog.csdn.net/20180714172903542 "创建CA的自签名证书")     

2. 利用keytool生成server端的证书，信息和创建CA时需要保持一致      
```shell
keytool -genkeypair -alias server -keyalg RSA -keystore server.keystore
```
![利用keytool生成server端的证书](https://img-blog.csdn.net/20180714173020277 "利用keytool生成server端的证书")      

3. 生成server端证书签名请求     
```shell
keytool -certreq -alias server -file server.csr -keystore server.keystore
```
<!-- csr: Certificate Signing Request  -->   
![生成server端证书签名请求](https://img-blog.csdn.net/20180714173122456 "生成server端证书签名请求 ")    

4. 使用OpenSSL RootCA 签发server端证书签名请求的证书     
```shell
openssl ca -in server.csr -out server.cer -cert test_ca.cer -keyfile test_ca.key -notext
```
![使用OpenSSL RootCA 签发server端证书签名请求的证书](https://img-blog.csdn.net/20180714173226851 "使用OpenSSL RootCA 签发server端证书签名请求的证书")   
**Question:**
 - **文件notfound**    
参考：[OpenSSL生成证书对](http://blog.sina.com.cn/s/blog_49f8dc400100tznt.html)   
```shell
mkdir ./demoCA
mkdir ./demoCA/newcerts
touch index.txt
vim serial // echo '01\n'>serial
```

 - **编码格式**     
 有时候,CA颁发机构和签名请求方极低可能是同一家公司或组织,即使是同一家公司或组织也可能是不同的部门,比如说,Android 组的朋友使用 Java 自带的 KeyTool 生成了一个签名请求文件(\*.csr)给我来签名,尽管我们所有关键域的值都是一样,但还是签不过,总是报如下问题:     
```
The xxx field needed to be same
```
  ~~为什么 sichuang和 sichuang 不相同呢????~~    
  **原因就在于 Java 的 KeyTool 生成的签名请求文件的编码格式与 OpenSSL 的默认编码格式不一致,Java KeyTool默认使用就是全部 PRINTABLE 而 OpenSSL 既有PRINTABLE 也有 ASN 1.12。**    
 解决方法：修改/etc/pki/tls/openssl.cnf 中如下选项         
 vim /etc/pki/tls/openssl.cnf
![solved_method](https://img-blog.csdn.net/20180714173519603 "solved_method")  
问题解决，成功签名证书      
![solved](https://img-blog.csdn.net/20180714173730839 "solved")     
5. 导入CA根证书到keystore，信任此证书    
```shell
keytool -import -trustcacerts -alias ca_root -file test_ca.cer -keystore server.keystore
```
![导入CA根证书到keystore](https://img-blog.csdn.net/2018071417382917 "导入CA根证书到keystore")     

6. 将RootCA签名后的server.cer 证书添加到server.keystore
```shell
keytool -import -alias serverbyCA -file server.cer -keystore server.keystore
```
![将RootCA签名后的server.cer 证书添加到server.keystore](https://img-blog.csdn.net/20180714173938783 "将RootCA签名后的server.cer 证书添加到server.keystore")   
我们一般将RootCA自签名证书，可信任的证书放到truststore中  
```shell
keytool -importcert -alias ca_root -file test_ca.cer -keystore truststore.jks
```
![](https://img-blog.csdn.net/20180714174013967)

keystore和truststore从其文件格式来看其实是一个东西，只是为了方便管理将其分开    
- keystore中一般保存的是我们的私钥，用来加解密、为别人做签名或者保存签名证书。    
- truststore中保存的是一些可信任的证书，存放的是只包含公钥的数字证书，代表了可以信任的证书，而keystore是包含私钥的，如RootCA自签名证书，主要是java在代码中访问某个https的时候对被访问者进行认证的，以确保其实可信任的。    


---
# KeyTool
1. 通过使用以下的命令来创建服务器端的密匙库
```shell
keytool -genkey -alias hellking -keystore server.keystore -keyalg RSA  
```  
以上命令执行完成后，将获得一个名为server.keystore的密匙库。      
2. 生成客户端的信任库。首先输出RSA证书：    
```shell
keytool -export -file test_axis.cer -storepass changeit -keystore server.keystore     
```
然后把RSA证书输入到一个新的信任库文件中。这个信任库被客户端使用，被用来验证服务器端的身份。
```shell
keytool -import -file test_axis.cer -storepass changeit -keystore client.truststore -alias serverkey -noprompt
```
3. 创建客户端密匙库。重复步骤1，创建客户端的密匙库。也可以使用以下命令来完成：
```shell
keytool -genkey -dname "CN=hellking-Client, OU=tsinghua, O=tsinghua, L=BEIJING, S=BEIJING, C=CN" -storepass changeit -keystore client.keystore -keyalg RSA -keypass changeit  
```
4. 生成服务器端的信任库。    
```shell
keytool -export -file test_axis.cer -storepass changeit -keystore client.keystore  
keytool -import -file test_axis.cer -storepass changeit -keystore server.truststore -alias clientkey -noprompt  
```
生成了密匙库和信任库，我们把服务器端的密匙库（server.keystore）和信任库（server.truststore）拷贝到Tomcat的某个目录。    

5. 为Tomcat配置SSL协议。
```xml
<Connector port="443"   
           maxThreads="150" minSpareThreads="25" maxSpareThreads="75"  
           enableLookups="false" disableUploadTimeout="true"  
           acceptCount="100" debug="0" scheme="https" secure="true"  
           clientAuth="true" keystoreFile="K:\jakarta-tomcat-5.0.16\server.keystore" keystorePass="changeit"  
           truststoreFile="K:\jakarta-tomcat-5.0.16\server.truststore" truststorePass="changeit"  
           sslProtocol="TLS" />  
```
clientAuth参数制定服务器是否要验证客户端证书，如果指定为true，那么客户端必须拥护服务器端可信任的证书后服务器才能响应客户端；如果指定为false，那么服务器不需要验证客户端的证书。
