---
title: 'SpringBoot  JWT实现'
date: 2021-06-06 16:50:36
tags: [spring,jwt]
published: true
hideInList: false
feature: /post-images/ZsckGVdlz.png
isTop: false
---
### SpringBoot  JWT实现

（只是实现了jwt,没有生成证书,安全性得不到保障,证书安全验证查看[Spring security JWT](https://tinaxiawuhao.github.io/post/wWtT_Oc9B/)）

```java
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.6.0</version>
</dependency>
```

```java
import io.jsonwebtoken.JwtBuilder;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import java.util.Date;

public class CreateJwtTest3 {
    public static void main(String[] args) {
        //为了方便测试，我们将过期时间设置为1分钟
        long now = System.currentTimeMillis();//当前时间
        long exp = now + 1000 * 60;//过期时间为1分钟
        JwtBuilder builder = Jwts.builder().setId("888")
                .setSubject("小白")
                .setIssuedAt(new Date())
                .signWith(SignatureAlgorithm.HS256, "itcast")
                .setExpiration(new Date(exp))
                .claim("roles", "admin") //自定义claims存储数据
                .claim("logo", "logo.png");
        System.out.println(builder.compact());
    }
}
```

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import org.apache.commons.lang3.time.DateFormatUtils;

import java.util.Date;

public class ParseJwtTest {
    public static void main(String[] args) {
        String token = "eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiI4ODgiLCJzdWIiOiLlsI_nmb0iLCJpYXQiOjE1NjExMDE3MzIsImV4cCI6MTU2MTEwMTc5MSwicm9sZXMiOiJhZG1pbiIsImxvZ28iOiJsb2dvLnBuZyJ9.5iVVdTw747L3ScHeCqle-bwj3cezK8NnE7VilQWOr8Y";
        Claims claims = Jwts.parser().setSigningKey("itcast").parseClaimsJws(token).getBody();
        System.out.println("id:" + claims.getId());
        System.out.println("subject:" + claims.getSubject());
        System.out.println("roles:" + claims.get("roles"));
        System.out.println("logo:" + claims.get("logo"));
        System.out.println("签发时间:"+ DateFormatUtils.format(claims.getIssuedAt(),"yyyy-MM-dd hh:mm:ss"));
        System.out.println("过期时间:"+DateFormatUtils.format(claims.getExpiration(),"yyyy-MM-dd hh:mm:ss"));
        System.out.println("当前时间:"+DateFormatUtils.format(new Date(),"yyyy-MM-dd hh:mm:ss"));
    }
}
```