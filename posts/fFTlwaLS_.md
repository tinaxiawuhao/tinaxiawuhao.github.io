---
title: 'RestTemplate'
date: 2021-06-07 13:34:32
tags: [springBoot]
published: true
hideInList: false
feature: /post-images/fFTlwaLS_.png
isTop: false
---
### spring环境下

首先导入springboot 的 web 包

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

在启动类同包下创建RestTemplate.java类

```java
import org.apache.http.client.HttpClient;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.impl.client.DefaultHttpRequestRetryHandler;
import org.apache.http.impl.client.HttpClientBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.ClientHttpRequestFactory;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTempleConfig {

    @Bean
    public RestTemplate restTemplate() {

        //生成一个设置了连接超时时间、请求超时时间
        RequestConfig config = RequestConfig.custom()
                .setConnectionRequestTimeout(10000)
                .setConnectTimeout(10000)
                .setSocketTimeout(30000).build();

        // 设置异常重试
        HttpClientBuilder builder = HttpClientBuilder.create()
                .setDefaultRequestConfig(config)
                .setRetryHandler(new DefaultHttpRequestRetryHandler(3, false));
        HttpClient httpClient = builder.build();

        ClientHttpRequestFactory requestFactory =
                new HttpComponentsClientHttpRequestFactory(httpClient);
        RestTemplate restTemplate = new RestTemplate(requestFactory);
        // 日志拦截
        //restTemplate.setInterceptors(Collections.singletonList(new RestTemplateConsumerLogger()));

        return restTemplate;
    }
}
```

### 非spring环境下

导入相关依赖包(注意版本相适应)

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>5.2.16.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.2</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.12.1</version>
    <scope>compile</scope>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.12.1</version>
    <scope>compile</scope>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
    <version>2.12.1</version>
    <scope>compile</scope>
</dependency>
```

### RestTemplate

RestTemplate定义了36个与REST资源交互的方法，大多数都对应于HTTP的方法。 其中只有11个独立的方法，其中有十个有三种重载形式，而第十一个则重载了六次，这样一共形成了36个方法。

1. getForEntity() 发送一个HTTP GET请求，返回的ResponseEntity包含了响应体所映射成的对象

2. getForObject() 发送一个HTTP GET请求，返回的请求体将映射为一个对象

3. postForEntity() POST 数据到一个URL，返回包含一个对象的ResponseEntity，这个对象是从响应体中映射得到的

4. postForObject() POST 数据到一个URL，返回根据响应体匹配形成的对象

5. exchange() 在URL上执行特定的HTTP方法，返回包含对象的ResponseEntity，这个对象是从响应体中映射得到的

6. execute() 在URL上执行特定的HTTP方法，返回一个从响应体映射得到的对象

7. delete() 在特定的URL上对资源执行HTTP DELETE操作

8. headForHeaders() 发送HTTP HEAD请求，返回包含特定资源URL的HTTP头

9. optionsForAllow() 发送HTTP OPTIONS请求，返回对特定URL的Allow头信息

10. postForLocation() POST 数据到一个URL，返回新创建资源的URL

11. put() PUT 资源到特定的URL

#### getForEntity

get请求就和正常在浏览器url上发送请求一样

```java
RestTemplate restTemplate = new RestTemplate();
ResponseEntity<String> responseEntity=restTemplate.getForEntity(url+"?name={1}", String.class, "username");
String body = responseEntity.getBody();
```

```java
RestTemplate restTemplate = new RestTemplate();
ResponseEntity<TokenBeen> responseEntity =restTemplate.getForEntity(url+"?name={1}", TokenBeen.class, "username");
if(responseEntity!=null){
    TokenBeen body = responseEntity.getBody();
}
```
```java
#注意map的key要和参数中占位符相同
RestTemplate restTemplate = new RestTemplate();
Map<String, String> params = new HashMap<>();
params.put("name", "username");
ResponseEntity<String> responseEntity = restTemplate.getForEntity(url+"?name={name}", String.class, params);
String body = responseEntity.getBody();
```
```java
RestTemplate restTemplate = new RestTemplate();
UriComponents uriConponents = UriComponentsBuilder.fromUriString(url+"?name={name}").build().expand("username").encode();
URI uri = uriConponents.toUri();
ResponseEntity<String> responseEntity = restTemplate.getForEntity(uri, String.class);
String body = responseEntity.getBody();
```

```java
@GetMapping("getForEntity/{id}")
public User getById(@PathVariable(name = "id") String id) {
    ResponseEntity<User> response = restTemplate.getForEntity("http://localhost/get/{id}", User.class, id);
    User user = response.getBody();
    return user;
}
```

#### getForObject

getForObject 和 getForEntity 用法几乎相同,getForObject函数可以看作是对getForEntity进一步封装,指示返回值返回的是响应体,省去了我们 再去 getBody() 

```java
RestTemplate restTemplate = new RestTemplate();
//注意参数中是uri
UriComponents uriConponents = UriComponentsBuilder.fromUriString(url+"?name={name}").build().expand("username").encode();
URI uri = uriConponents.toUri();
String body = restTemplate.getForObject(uri, String.class);
```

```java
RestTemplate restTemplate = new RestTemplate();
//注意参数中是uri
UriComponents uriConponents = UriComponentsBuilder.fromUriString(url+"?name={name}").build().expand("username").encode();
URI uri = uriConponents.toUri();
TokenBeen body = restTemplate.getForObject(uri, TokenBeen.class);
```

```java
@GetMapping("getForObject/{id}")
public User getById(@PathVariable(name = "id") String id) {
    User user = restTemplate.getForObject("http://localhost/get/{id}", User.class, id);
    return user;
}
```

#### postForEntity

```java
public <T> T postForObject(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables)throws RestClientException {} 
public <T> T postForObject(String url, @Nullable Object request, Class<T> responseType, Map<String, ?> uriVariables) throws RestClientException {} 
public <T> T postForObject(URI url, @Nullable Object request, Class<T> responseType) throws RestClientException {}
```
```java
RestTemplate restTemplate = new RestTemplate();
HttpHeaders headers = new HttpHeaders();
//header参数
MediaType type = MediaType.parseMediaType("application/json; charset=UTF-8");
headers.setContentType(type);
headers.add("Accept", MediaType.APPLICATION_JSON.toString());
//body参数
JSONObject param = new JSONObject();
param.put("username", "123");
HttpEntity<JSONObject> formEntity = new HttpEntity<>(param, headers);
//发送请求
String result = restTemplate.postForObject(url, formEntity, String.class);
```

```java
@RequestMapping("saveUser")
public String save(User user) {
    ResponseEntity<String> response = restTemplate.postForEntity("http://localhost/save", user, String.class);
    String body = response.getBody();
    return body;
}
```

#### postForObject

用法与 getForObject 一样

```java
@RequestMapping("saveUser")
public String save(User user) {
    String body = restTemplate.postForObject("http://localhost/save", user, String.class);
    return body;
}
```

#### exchange

```java
@PostMapping("demo")
public void demo(Integer id, String name){
 
    HttpHeaders headers = new HttpHeaders();//header参数
    headers.add("authorization",Auth);
    headers.setContentType(MediaType.APPLICATION_JSON);

    JSONObject content = new JSONObject();//放入body中的json参数
    content.put("userId", id);
    content.put("name", name);

    //post发送
    HttpEntity<JSONObject> request = new HttpEntity<>(content,headers); //组装
    ResponseEntity<String> response = template.exchange("http://localhost:8080/demo",HttpMethod.POST,request,String.class);
    //返回指定对象
    ParameterizedTypeReference<User> responseBodyType = new ParameterizedTypeReference<RestBean<String>>() {};
    User user = template.exchange("http://localhost:8080/demo",HttpMethod.POST,request,responseBodyType);
    
    
    //get发送
    HttpEntity<String> request = new HttpEntity<>("parameters",headers); //组装
    ResponseEntity<String> response = template.exchange("http://localhost:8080/demo",HttpMethod.GET,request,String.class);
    //返回指定对象
    ResponseEntity<User> response = template.exchange("http://localhost:8080/demo",HttpMethod.GET,request,User.class);
}
```



![](https://tinaxiawuhao.github.io/post-images/1623908192461.png)