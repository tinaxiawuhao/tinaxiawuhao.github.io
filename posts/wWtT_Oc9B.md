---
title: 'JWT介绍'
date: 2021-06-05 16:45:38
tags: [spring,jwt]
published: true
hideInList: false
feature: /post-images/wWtT_Oc9B.png
isTop: false
---
## JWT介绍

在介绍JWT之前先看一下传统校验令牌的方法，如下图：

![](https://tinaxiawuhao.github.io/post-images/1623401311518.png)

资源服务器授权流程如上图，客户端先去授权服务器申请令牌，申请令牌后，携带令牌访问资源服务器，资源服务器访问授权服务校验令牌的合法性，授权服务会返回校验结果，如果校验成功会返回用户信息给资源服务器，资源服务器如果接收到的校验结果通过了，则返回资源给客户端

**问题：**

传统授权方法的问题是用户每次请求资源服务，资源服务都需要携带令牌访问认证服务去校验令牌的合法性，并根据令牌获取用户的相关信息，性能低下。

**解决：**

使用JWT的思路是，用户认证通过会得到一个JWT令牌，JWT令牌中已经包括了用户相关的信息，客户端只需要携带JWT访问资源服务，资源服务根据事先约定的算法自行完成令牌校验，无需每次都请求认证服务完成授权。

JWT令牌授权过程如下图：

![](https://tinaxiawuhao.github.io/post-images/1623401320289.png)

### 什么是JWT？

JSON Web Token（JWT）是一个开放的行业标准（RFC 7519），它定义了一种简介的、自包含的协议格式，用于在通信双方传递json对象，传递的信息经过数字签名可以被验证和信任。JWT可以使用HMAC算法或使用RSA的公钥/私钥对来签名，防止被篡改。

> 官网：https://jwt.io/
>
> 标准：https://tools.ietf.org/html/rfc7519

#### 优点：

1. jwt基于json，非常方便解析。

2. 可以在令牌中自定义丰富的内容，易扩展。

3. 通过非对称加密算法及数字签名技术，JWT防止篡改，安全性高。

4. 资源服务使用JWT可不依赖认证服务即可完成授权。

#### 缺点：

1. JWT令牌较长，占存储空间比较大。

#### 令牌结构

通过学习JWT令牌结构为自定义jwt令牌打好基础。

JWT令牌由三部分组成，每部分中间使用点（.）分隔，比如：xxxxx.yyyyy.zzzzz

**Header**

头部包括令牌的类型（即JWT）及使用的哈希算法（如HMAC SHA256或RSA）

一个例子如下：

下边是Header部分的内容

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

将上边的内容使用Base64Url编码，得到一个字符串就是JWT令牌的第一部分。

**Payload**

第二部分是负载，内容也是一个json对象，它是存放有效信息的地方，它可以存放jwt提供的现成字段，比如：iss（签发者）,exp（过期时间戳）, sub（面向的用户）等，也可自定义字段。

此部分不建议存放敏感信息，因为此部分可以解码还原原始内容。

最后将第二部分负载使用Base64Url编码，得到一个字符串就是JWT令牌的第二部分。

```json
 {
    "sub": "1234567890",
    "name": "456",
    "admin": true
}
```

**Signature**

第三部分是签名，此部分用于防止jwt内容被篡改。这个部分使用base64url将前两部分进行编码，编码后使用点（.）连接组成字符串，最后使用header中声明签名算法进行签名。

```java
HMACSHA256(base64UrlEncode(header) + "." +base64UrlEncode(payload),secret)
```

`base64UrlEncode(header)`：jwt令牌的第一部分。

`base64UrlEncode(payload)`：jwt令牌的第二部分。

`secret`：签名所使用的密钥。

### JWT入门

Spring Security 提供对JWT的支持，本节我们使用Spring Security 提供的JwtHelper来创建JWT令牌，校验JWT令牌等操作。

### 生成私钥和公钥

JWT令牌生成采用非对称加密算法

1. 生成密钥证书

   下边命令生成密钥证书，采用RSA 算法每个证书包含公钥和私钥

   ```java
   keytool -genkeypair -alias xckey -keyalg RSA -keypass xuecheng -keystore xc.keystore -storepass xuechengkeystore
   ```

    > Keytool 是一个java提供的证书管理工具
    >
    > -alias：密钥的别名
    >
    > -keyalg：使用的hash算法
    >
    > -keypass：密钥的访问密码
    >
    > -keystore：密钥库文件名，xc.keystore保存了生成的证书
    >
    > -storepass：密钥库的访问密码

   **查询证书信息**

   ```java
   keytool -list -keystore xc.keystore
   ```

   **删除别名**

   ```java
   keytool -delete -alias xckey -keystore xc.keystore
   ```


2. 导出公钥

   openssl是一个加解密工具包，这里使用openssl来导出公钥信息。安装 openssl：http://slproweb.com/products/Win32OpenSSL.html

   配置openssl的path环境变量，本文配置在`D:\OpenSSL-Win64\bin`  `cmd`进入`xc.keystore`文件所在目录执行如下命令：

   ```java
   keytool ‐list ‐rfc ‐‐keystore xc.keystore | openssl x509 ‐inform pem ‐pubkey
   ```

   输入密钥库密码：

   ![](https://tinaxiawuhao.github.io/post-images/1623908661423.png)

   下边这一段就是公钥内容：

   ```java
   -----BEGIN PUBLIC KEY----- MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAijyxMdq4S6L1Af1rtB8SjCZHNgsQG8JTfGy55eYvzG0B/E4AudR2prSRBvF7NYPL47scRCNPgLnvbQczBHbBug6uOr78qnWsYxHlW6Aa5dI5NsmOD4DLtSw8eX0hFyK5Fj6ScYOSFBz9cd1nNTvx2+oIv0lJDcpQdQhsfgsEr1ntvWterZt/8r7xNN83gHYuZ6TM5MYvjQNBc5qC7Krs9wM7UoQuL+s0X6RlOib7/mcLn/lFLsLDdYQAZkSDx/6+t+1oHdMarChIPYT1sx9Dwj2j2mvFNDTKKKKAq0cv14Vrhz67Vjmz2yMJePDqUi0JYS2r0iIo7n8vN7s83v5uOQIDAQAB 
   -----END PUBLIC KEY-----
   ```

   将上边的公钥拷贝到文本文件中，合并为一行。

### 生成jwt令牌

在认证工程创建测试类，测试jwt令牌的生成与验证。

```java
@Test
public void testCreateJwt(){
//证书文件
String key_location = "xc.keystore";
//密钥库密码
String keystore_password = "xuechengkeystore";
//访问证书路径
ClassPathResource resource = new ClassPathResource(key_location);
//密钥工厂
KeyStoreKeyFactory keyStoreKeyFactory = new KeyStoreKeyFactory(resource,
keystore_password.toCharArray());
//密钥的密码，此密码和别名要匹配

String keypassword = "xuecheng";
//密钥别名
String alias = "xckey";
//密钥对（密钥和公钥）
KeyPair keyPair = keyStoreKeyFactory.getKeyPair(alias,keypassword.toCharArray());
//私钥
RSAPrivateKey aPrivate = (RSAPrivateKey) keyPair.getPrivate();
//定义payload信息
Map<String, Object> tokenMap = new HashMap<>();
tokenMap.put("id", "123");
tokenMap.put("name", "mrt");
tokenMap.put("roles", "r01,r02");
tokenMap.put("ext", "1");
//生成jwt令牌
Jwt jwt = JwtHelper.encode(JSON.toJSONString(tokenMap), new RsaSigner(aPrivate));
//取出jwt令牌
String token = jwt.getEncoded();
System.out.println("token="+token);
} //资源服务使用公钥验证jwt的合法性，并对jwt解码
```

### 验证jwt令牌

```java
@Test
public void testVerify(){
//jwt令牌
String token
="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHQiOiIxIiwicm9sZXMiOiJyMDEscjAyIiwibmFtZSI6Im1ydCIsImlkIjoiMTIzIn0.KK7_67N5d1Dthd1PgDHMsbi0UlmjGRcm_XJUUwseJ2eZyJJWoPP2IcEZgAU3tUaaKEHUf9wSRwaDgwhrwfyIcSHbs8oy3zOQEL8j5AOjzBBs7vnRmB7DbSaQD7eJiQVJOXO1QpdmEFgjhc_IBCVTJCVWgZw60IEW1_Lg5tqaLvCiIl26K48pJB5fle2zgYMzqR1L2LyTFkq39rG57VOqqSCi3dapsZQd4ctq95SJCXgGdrUDWtD52rp5o6_0uq‐mrbRdRxkrQfsa1j8C5IW2‐T4eUmiN3f9wF9JxUK1__XC1OQkOn‐ZTBCdqwWIygDFbU7sf6KzfHJTm5vfjp6NIA";
//公钥
String publickey = "‐‐‐‐‐BEGIN PUBLIC KEY‐‐‐‐‐
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAijyxMdq4S6L1Af1rtB8SjCZHNgsQG8JTfGy55eYvzG0B/E4AudR2prSRBvF7NYPL47scRCNPgLnvbQczBHbBug6uOr78qnWsYxHlW6Aa5dI5NsmOD4DLtSw8eX0hFyK5Fj6ScYOSFBz9cd1nNTvx2+oIv0lJDcpQdQhsfgsEr1ntvWterZt/8r7xNN83gHYuZ6TM5MYvjQNBc5qC7Krs9wM7UoQuL+s0X6RlOib7/mcLn/lFLsLDdYQAZkSDx/6+t+1oHdMarChIPYT1sx9Dwj2j2mvFNDTKKKKAq0cv14Vrhz67Vjmz2yMJePDqUi0JYS2r0iIo7n8vN7s83v5u
OQIDAQAB
‐‐‐‐‐END PUBLIC KEY‐‐‐‐‐";

//校验jwt
Jwt jwt = JwtHelper.decodeAndVerify(token, new RsaVerifier(publickey));
//获取jwt原始内容
String claims = jwt.getClaims();
//jwt令牌
String encoded = jwt.getEncoded();
System.out.println(encoded);
}
```