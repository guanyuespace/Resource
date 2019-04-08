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

# Keytool和OpenSSL生成和签发数字证书   
>[Keytool和OpenSSL生成和签发数字证书](https://blog.csdn.net/naioonai/article/details/81045780)

生成数字签名证书具体操作          
<!-- more -->
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
1. 通过使用一下的命令来创建服务器端的密匙库
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
keytool -genkey -dname " CN=hellking-Client, OU=tsinghua, O=tsinghua, L=BEIJING, S=BEIJING, C=CN" -storepass changeit -keystore client.keystore -keyalg RSA -keypass changeit  
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
