---
title: Security-RSA
date: 2019-02-13 13:25:10
categories:
- java
- security
- encryption
tags:
- encryption
- decryption
---
# RSA
分组加密算法，非对称加密算法
>参考    
[RSA - 原理、特点（加解密及签名验签）及公钥和私钥的生成](https://blog.csdn.net/kikajack/article/details/80703894)
## 基于--大数N难分解  `e*d≡1mod(r)--->e*d≡1mod(φ(N))`

<!-- more -->

## 公钥私钥获取
随意选择两个大的质数p和q，p不等于q，计算N=pq。  
根据欧拉函数，求得r=φ(N)=φ(p)φ(q)=(p−1)(q−1)  
选择一个小于r的整数e，使e与r互质。并求得e关于r的模反元素，命名为d（求d令ed≡1(modr)）。（模反元素存在，当且仅当e与r互质;**将p和q的记录销毁。**         
(N,e)是公钥，(N,d)是私钥。公钥发送给所有的通信对象（对服务器来说就是所有的客户端），私钥则必须保管好，防止泄露。  

## 数学原理  
1. 欧拉函数：求小于N的正整数中与N互质的数的数目。φ(N) :φ(11)=10; φ(4)=2[1,3]; φ(8)=4[1,3,5,7]      
2. 欧拉定理证明当a,n为两个互素的正整数时，则有a^φ(n)≡1(modn),其中 φ(n)为欧拉函数（小于等于n且与n互素的正整数个数）。

### 加密过程
明文：m   公钥（N,e）
密文：c≡m^e(mod N)

### 解密过程
密文：c    私钥（N,d）
明文： c^d(mod N)
```
c^d(mod N)≡ m^de(mod N)

由于ed≡1(modr)  -->   ed≡1(mod φ(N))

c^d(mod N)≡ m^de(mod N)  ≡ m^ (Kφ(N)+1) (mod N) ≡m* (m^ φ(N))^K  (mod N) //K 为倍系数  倍数

N=p*q (p,q为两不相同大质数)，m与N互质

m^φ(N) ≡ 1 mod N

c^d(mod N)≡ m^de(mod N)  ≡ m^ (Kφ(N)+1) (mod N) ≡m* (m^ φ(N))^K  (mod N)

≡ m *1^K (mod N)≡m (mod N)
```
## 使用
```java
方式一：
BigInteger rs = new BigInteger(String.format("%x", new BigInteger(1, text.getBytes())), 16)
					.modPow(new BigInteger(pubKey, 16), new BigInteger(modulus, 16));//(pubKey=e,modulus=N)
String result = rs.toString(16);

方式二：
RSAPublicKeySpec keySpec = new RSAPublicKeySpec(modulus, pubkey);//(modulus=N,pubKey=e)
KeyFactory keyFactory = KeyFactory.getInstance("RSA");
PublicKey publicKey = keyFactory.generatePublic(keySpec);

//ECB（无IV） 无填充
Cipher cipher = Cipher.getInstance("RSA/ECB/NoPadding");// PKCS1Padding
cipher.init(Cipher.ENCRYPT_MODE, publicKey);

randomStr = new StringBuilder(randomStr).reverse().toString();//待加密字符串
byte[] data = randomStr.getBytes("utf-8");
int inputLen = data.length;
ByteArrayOutputStream out = new ByteArrayOutputStream();
int offSet = 0;
byte[] cache;
int i = 0;

// 对数据分段加密
while (inputLen - offSet > 0) {
	if (inputLen - offSet > MAX_ENCRYPT_BLOCK) {
		cache = cipher.doFinal(data, offSet, MAX_ENCRYPT_BLOCK);
	} else {
		cache = cipher.doFinal(data, offSet, inputLen - offSet);
	}
	out.write(cache, 0, cache.length);
	i++;
	offSet = i * MAX_ENCRYPT_BLOCK;
}
byte[] encryptedData = out.toByteArray();
out.close();
String result = byte2HexString(encryptedData);
```

---
# DSA(Digital Signature Algorithm)  
DSA是基于整数有限域离散对数难题的，其安全性与RSA相比差不多。DSA的一个重要特点是两个素数公开，这样，当使用别人的p和q时，即使不知道私钥，你也能确认它们是否是随机产生的，还是作了手脚。RSA算法却做不到。<!--？？？ 理解 ？？？-->   

```
算法中应用了下述参数：
p：L bits长的素数。L是64的倍数，范围是512到1024；
q：p - 1的160bits的素因子；
g：g = h^((p-1)/q) mod p，h满足h < p - 1, h^((p-1)/q) mod p > 1；
x：x < q，x为私钥 ；
y：y = g^x mod p ，( p, q, g, y )为公钥；
H( x )：One-Way Hash函数。DSS（FIPS186-4）中选用SHA-1或者SHA-2( Secure Hash Algorithm系列中的2个较新版本，其中SHA-2有4个，SHA-224，SHA-256，SHA-384，SHA-512，最原始的SHA已经不再被使用)。
p, q, g可由一组用户共享，但在实际应用中，使用公共模数可能会带来一定的威胁。签名及验证协议如下：
1. P产生随机数k，k < q；
2. P计算 r = ( g^k mod p ) mod q
s = ( k^(-1) (H(m) + xr)) mod q
签名结果是( m, r, s )。
3. 验证时计算 w = s^(-1)mod q
u1 = ( H( m ) * w ) mod q
u2 = ( r * w ) mod q
v = (( g^u1 * y^u2 ) mod p ) mod q
若v = r，则认为签名有效。
```


## 使用
```java
import org.apache.commons.codec.binary.Base64;
/**
 * 常用数字签名算法DSA
 * 数字签名
 * @author kongqz
 * */
public class DSACoder {
	//数字签名，密钥算法
	public static final String KEY_ALGORITHM="DSA";

	/**
	 * 数字签名
	 * 签名/验证算法
	 * */
	public static final String SIGNATURE_ALGORITHM="SHA1withDSA";

	/**
	 * DSA密钥长度，RSA算法的默认密钥长度是1024
	 * 密钥长度必须是64的倍数，在512到1024位之间
	 * */
	private static final int KEY_SIZE=1024;
	//公钥
	private static final String PUBLIC_KEY="DSAPublicKey";

	//私钥
	private static final String PRIVATE_KEY="DSAPrivateKey";

	/**
	 * 初始化密钥对
	 * @return Map 甲方密钥的Map
	 * */
	public static Map<String,Object> initKey() throws Exception{
		//实例化密钥生成器
		KeyPairGenerator keyPairGenerator=KeyPairGenerator.getInstance(KEY_ALGORITHM);
		//初始化密钥生成器
		keyPairGenerator.initialize(KEY_SIZE,new SecureRandom());
		//生成密钥对
		KeyPair keyPair=keyPairGenerator.generateKeyPair();
		//甲方公钥
		DSAPublicKey publicKey=(DSAPublicKey) keyPair.getPublic();
		//甲方私钥
		DSAPrivateKey privateKey=(DSAPrivateKey) keyPair.getPrivate();
		//将密钥存储在map中
		Map<String,Object> keyMap=new HashMap<String,Object>();
		keyMap.put(PUBLIC_KEY, publicKey);
		keyMap.put(PRIVATE_KEY, privateKey);
		return keyMap;

	}


	/**
	 * 签名
	 * @param data待签名数据
	 * @param privateKey 密钥
	 * @return byte[] 数字签名
	 * */
	public static byte[] sign(byte[] data,byte[] privateKey) throws Exception{

		//取得私钥
		PKCS8EncodedKeySpec pkcs8KeySpec=new PKCS8EncodedKeySpec(privateKey);
		KeyFactory keyFactory=KeyFactory.getInstance(KEY_ALGORITHM);
		//生成私钥
		PrivateKey priKey=keyFactory.generatePrivate(pkcs8KeySpec);
		//实例化Signature
		Signature signature = Signature.getInstance(SIGNATURE_ALGORITHM);
		//初始化Signature
		signature.initSign(priKey);
		//更新
		signature.update(data);
		return signature.sign();
	}
	/**
	 * 校验数字签名
	 * @param data 待校验数据
	 * @param publicKey 公钥
	 * @param sign 数字签名
	 * @return boolean 校验成功返回true，失败返回false
	 * */
	public static boolean verify(byte[] data,byte[] publicKey,byte[] sign) throws Exception{
		//转换公钥材料
		//实例化密钥工厂
		KeyFactory keyFactory=KeyFactory.getInstance(KEY_ALGORITHM);
		//初始化公钥
		//密钥材料转换
		X509EncodedKeySpec x509KeySpec=new X509EncodedKeySpec(publicKey);
		//产生公钥
		PublicKey pubKey=keyFactory.generatePublic(x509KeySpec);
		//实例化Signature
		Signature signature = Signature.getInstance(SIGNATURE_ALGORITHM);
		//初始化Signature
		signature.initVerify(pubKey);
		//更新
		signature.update(data);
		//验证
		return signature.verify(sign);
	}
	/**
	 * 取得私钥
	 * @param keyMap 密钥map
	 * @return byte[] 私钥
	 * */
	public static byte[] getPrivateKey(Map<String,Object> keyMap){
		Key key=(Key)keyMap.get(PRIVATE_KEY);
		return key.getEncoded();
	}
	/**
	 * 取得公钥
	 * @param keyMap 密钥map
	 * @return byte[] 公钥
	 * */
	public static byte[] getPublicKey(Map<String,Object> keyMap) throws Exception{
		Key key=(Key) keyMap.get(PUBLIC_KEY);
		return key.getEncoded();
	}
	/**
	 * @param args
	 * @throws Exception
	 */
	public static void main(String[] args) throws Exception {
		//初始化密钥
		//生成密钥对
		Map<String,Object> keyMap=DSACoder.initKey();
		//公钥
		byte[] publicKey=DSACoder.getPublicKey(keyMap);

		//私钥
		byte[] privateKey=DSACoder.getPrivateKey(keyMap);
		System.out.println("公钥：/n"+Base64.encodeBase64String(publicKey));
		System.out.println("私钥：/n"+Base64.encodeBase64String(privateKey));

		System.out.println("================密钥对构造完毕,甲方将公钥公布给乙方，开始进行加密数据的传输=============");
		String str="DSA数字签名算法";
		System.out.println("原文:"+str);
		//甲方进行数据的加密
		byte[] sign=DSACoder.sign(str.getBytes(), privateKey);
		System.out.println("产生签名："+Base64.encodeBase64String(sign));
		//验证签名
		boolean status=DSACoder.verify(str.getBytes(), publicKey, sign);
		System.out.println("状态："+status+"/n/n");


	}
}
控制台输出：
公钥：
MIIBtzCCASwGByqGSM44BAEwggEfAoGBAP1/U4EddRIpUt9KnC7s5Of2EbdSPO9EAMMeP4C2USZp
RV1AIlH7WT2NWPq/xfW6MPbLm1Vs14E7gB00b/JmYLdrmVClpJ+f6AR7ECLCT7up1/63xhv4O1fn
xqimFQ8E+4P208UewwI1VBNaFpEy9nXzrith1yrv8iIDGZ3RSAHHAhUAl2BQjxUjC8yykrmCouuE
C/BYHPUCgYEA9+GghdabPd7LvKtcNrhXuXmUr7v6OuqC+VdMCz0HgmdRWVeOutRZT+ZxBxCBgLRJ
FnEj6EwoFhO3zwkyjMim4TwWeotUfI0o4KOuHiuzpnWRbqN/C/ohNWLx+2J6ASQ7zKTxvqhRkImo
g9/hWuWfBpKLZl6Ae1UlZAFMO/7PSSoDgYQAAoGAdLUOPbXqOQi4MFUm5tgBs2zQO20P7P1iPCC9
pslWvixp13NX9dTwdddkwqQtwKAfm/Ao2Gqe7VGq48kTIPr0wz01LKlCLbbw6VLikuFcgVNN+sVx
mwVTm54aFpiOaenS575Qtyek3zjVV+eMtRNKn91rMMWpsP6pucqku6xO5uY=
私钥：
MIIBSwIBADCCASwGByqGSM44BAEwggEfAoGBAP1/U4EddRIpUt9KnC7s5Of2EbdSPO9EAMMeP4C2
USZpRV1AIlH7WT2NWPq/xfW6MPbLm1Vs14E7gB00b/JmYLdrmVClpJ+f6AR7ECLCT7up1/63xhv4
O1fnxqimFQ8E+4P208UewwI1VBNaFpEy9nXzrith1yrv8iIDGZ3RSAHHAhUAl2BQjxUjC8yykrmC
ouuEC/BYHPUCgYEA9+GghdabPd7LvKtcNrhXuXmUr7v6OuqC+VdMCz0HgmdRWVeOutRZT+ZxBxCB
gLRJFnEj6EwoFhO3zwkyjMim4TwWeotUfI0o4KOuHiuzpnWRbqN/C/ohNWLx+2J6ASQ7zKTxvqhR
kImog9/hWuWfBpKLZl6Ae1UlZAFMO/7PSSoEFgIUQhtJHLosUo0sxPKmCxC8NFVjD9c=
================密钥对构造完毕,甲方将公钥公布给乙方，开始进行加密数据的传输=============
原文:DSA数字签名算法
产生签名：MCwCFAsJPC7xUfrvGYXtsxiWcS6GHAe1AhRnZLZ1CJe8I41vaScvJ7UZ8yk0oA==
状态：true
```

## 总结
1. DSA的公钥长度略长于私钥，这个和RSA有较大差别。可以通过两个算法的控制台输出公钥私钥长度比对
2. DSA算法的签名长度与密钥长度无关。可能和待签名数据有联系...
