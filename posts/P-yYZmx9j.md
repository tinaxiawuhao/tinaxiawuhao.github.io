---
title: 'MyBatis 缓存详解'
date: 2021-06-01 16:51:37
tags: [mybatis]
published: true
hideInList: false
feature: /post-images/P-yYZmx9j.png
isTop: false
---
　　缓存是一般的ORM 框架都会提供的功能，目的就是提升查询的效率和减少数据库的压力。跟Hibernate 一样，MyBatis 也有一级缓存和二级缓存，并且预留了集成第三方缓存的接口。

缓存体系结构：

![](https://tinaxiawuhao.github.io/post-images/1622193533861.png)

　　MyBatis 跟缓存相关的类都在cache 包里面，其中有一个Cache 接口，只有一个默认的实现类 `PerpetualCache`，它是用`HashMap` 实现的。

所有的缓存实现类总体上可分为三类：基本缓存、淘汰算法缓存、装饰器缓存。

| 缓存实现类 | 描述| 作用| 装饰条件 |
| :--------: | :--------------------: | :----------------------------------------------------------: | :------: |
| 基本缓存   | 缓存基本实现类 | 默认是PerpetualCache，也可以自定义比如RedisCache等，具备基本功能的缓存类 | 无       |
|LruCache	|LRU策略的缓存	|当缓存达到上限时，删除最近最少使用的缓存|	eviction=“LRU” （默认）|
|FifoCache	|FIFO策略的缓存	|当缓存达到上限时，删除最先入队的缓存	|eviction=“FIFO”|
|SoftCache/WeakCache|	带清理策略的缓存|	通过JVM的软引用和弱引用来实现缓存，当JVM内存不足时，会自动清理掉这些缓存	|eviction=“SOFT”/eviction=“WEAK”|
|LoggingCache	|带日志功能的缓存	|比如输出缓存命中率|	基本|
|SynchronizedCache	|同步缓存|	基于Synchronized关键字实现，解决并发问题|	基本|
|BlockingCache|	阻塞缓存	|通过在get/put方式中加锁，保证只有一个线程操作缓存，基于Java重入锁实现|	blocking=true|
|SerializedCache|	支持序列化的缓存|	将对象序列化以后存到缓存中，取出是反序列化|	readOnly=false(默认)|
|ScheduledCache|	定时调度的缓存|	在进行 get/put/remove/getSize 等操作前，判断 缓存时间是否超过了设置的最长缓存时间（默认是 一小时），如果是则清空缓存–即每隔一段时间清 空一次缓存|	flushInterval不为空|
|TransactionalCache|	事务缓存|	在二级缓存中使用，可以一次存入多个缓存，删除多个缓存	|在TransactionalCacheManager中用Map维护对应关系|


### 一级缓存（本地缓存）

　　一级缓存也叫本地缓存，MyBatis 的一级缓存是在会话（SqlSession）层面进行缓存的。MyBatis 的一级缓存是默认开启的，不需要任何的配置。首先我们必须去弄清楚一个问题，在MyBatis 执行的流程里面，涉及到这么多的对象，那么缓存`PerpetualCache` 应该放在哪个对象里面去维护？如果要在同一个会话里面共享一级缓存，这个对象肯定是在SqlSession 里面创建的，作为SqlSession 的一个属性。

　　`DefaultSqlSession` 里面只有两个属性，Configuration 是全局的，所以缓存只可能放在Executor 里面维护——`SimpleExecutor/ReuseExecutor/BatchExecutor` 的父类`BaseExecutor `的构造函数中持有了`PerpetualCache`。在同一个会话里面，多次执行相同的SQL 语句，会直接从内存取到缓存的结果，不会再发送SQL 到数据库。但是不同的会话里面，即使执行的SQL 一模一样（通过一个Mapper 的同一个方法的相同参数调用），也不能使用到一级缓存。

　　如下图所示，MyBatis会在一次会话的表示----一个SqlSession对象中创建一个本地缓存(local cache)，对于每一次查询，都会尝试根据查询的条件去本地缓存中查找是否在缓存中，如果在缓存中，就直接从缓存中取出，然后返回给用户；否则，从数据库读取数据，将查询结果存入缓存并返回给用户。

![](https://tinaxiawuhao.github.io/post-images/1622193556324.png)

一级缓存的生命周期有多长？

1. MyBatis在开启一个数据库会话时，会 创建一个新的`SqlSession`对象，`SqlSession`对象中会有一个新的Executor对象，Executor对象中持有一个新的`PerpetualCache`对象；当会话结束时，`SqlSession`对象及其内部的Executor对象还有`PerpetualCache`对象也一并释放掉。
2. 如果`SqlSession`调用了close()方法，会释放掉一级缓存`PerpetualCache`对象，一级缓存将不可用；
3. 如果`SqlSession`调用了`clearCache()`，会清空`PerpetualCache`对象中的数据，但是该对象仍可使用；
4. `SqlSession`中执行了任何一个update操作(update()、delete()、insert()) ，都会清空`PerpetualCache`对象的数据，但是该对象可以继续使用；

SqlSession 一级缓存的工作流程：

1. 对于某个查询，根据`statementId`,`params`,`rowBounds`来构建一个key值，根据这个key值去缓存Cache中取出对应的key值存储的缓存结果
2. 判断从Cache中根据特定的key值取的数据数据是否为空，即是否命中；
3. 如果命中，则直接将缓存结果返回；
4. 如果没命中：

- 1. 去数据库中查询数据，得到查询结果；
  2. 将key和查询到的结果分别作为key,value对存储到Cache中；
  3. 将查询结果返回；

　　接下来我们来验证一下，MyBatis 的一级缓存到底是不是只能在一个会话里面共享，以及跨会话（不同session）操作相同的数据会产生什么问题。判断是否命中缓存：如果再次发送SQL 到数据库执行，说明没有命中缓存；如果直接打印对象，说明是从内存缓存中取到了结果。

1. 在同一个session 中共享（不同session 不能共享）

2. 同一个会话中，update（包括delete）会导致一级缓存被清空

3. 其他会话更新了数据，导致读取到脏数据（一级缓存不能跨会话共享）

一级缓存的不足：

　　使用一级缓存的时候，因为缓存不能跨会话共享，不同的会话之间对于相同的数据可能有不一样的缓存。在有多个会话或者分布式环境下，会存在脏数据的问题。如果要解决这个问题，就要用到二级缓存。MyBatis 一级缓存（MyBaits 称其为 Local Cache）无法关闭，但是有两种级别可选：

1. session 级别的缓存，在同一个 sqlSession 内，对同样的查询将不再查询数据库，直接从缓存中。
2. statement 级别的缓存，避坑： 为了避免这个问题，可以将一级缓存的级别设为 statement 级别的，这样每次查询结束都会清掉一级缓存。

### 二级缓存

　　二级缓存是用来解决一级缓存不能跨会话共享的问题的，范围是`namespace` 级别的，可以被多个`SqlSession` 共享（只要是同一个接口里面的相同方法，都可以共享），生命周期和应用同步。如果你的MyBatis使用了二级缓存，并且你的Mapper和select语句也配置使用了二级缓存，那么在执行select查询的时候，MyBatis会先从二级缓存中取输入，其次才是一级缓存，即MyBatis查询数据的顺序是：二级缓存  —> 一级缓存 —> 数据库。

　　作为一个作用范围更广的缓存，它肯定是在SqlSession 的外层，否则不可能被多个SqlSession 共享。而一级缓存是在SqlSession 内部的，所以第一个问题，肯定是工作在一级缓存之前，也就是只有取不到二级缓存的情况下才到一个会话中去取一级缓存。第二个问题，二级缓存放在哪个对象中维护呢？ 要跨会话共享的话，SqlSession 本身和它里面的BaseExecutor 已经满足不了需求了，那我们应该在BaseExecutor 之外创建一个对象。

　　实际上MyBatis 用了一个装饰器的类来维护，就是`CachingExecutor`。如果启用了二级缓存，MyBatis 在创建Executor 对象的时候会对Executor 进行装饰。`CachingExecutor` 对于查询请求，会判断二级缓存是否有缓存结果，如果有就直接返回，如果没有委派交给真正的查询器Executor 实现类，比如`SimpleExecutor` 来执行查询，再走到一级缓存的流程。最后会把结果缓存起来，并且返回给用户。

![](https://tinaxiawuhao.github.io/post-images/1622193575153.png)

　　开启二级缓存的方法

第一步：配置 `mybatis.configuration.cache-enabled=true`，只要没有显式地设置`cacheEnabled=false`，都会用`CachingExecutor` 装饰基本的执行器。

第二步：在Mapper.xml 中配置<cache/>标签：

```xml
<cache type="org.apache.ibatis.cache.impl.PerpetualCache"
    size="1024"
eviction="LRU"
flushInterval="120000"
readOnly="false"/>
```

基本上就是这样。这个简单语句的效果如下:

- 映射语句文件中的所有 select 语句的结果将会被缓存。
- 映射语句文件中的所有 insert、update 和 delete 语句会刷新缓存。
- 缓存会使用最近最少使用算法（LRU, Least Recently Used）算法来清除不需要的缓存。
- 缓存不会定时进行刷新（也就是说，没有刷新间隔）。
- 缓存会保存列表或对象（无论查询方法返回哪种）的 1024 个引用。
- 缓存会被视为读/写缓存，这意味着获取到的对象并不是共享的，可以安全地被调用者修改，而不干扰其他调用者或线程所做的潜在修改。

这个更高级的配置创建了一个 FIFO 缓存，每隔 60 秒刷新，最多可以存储结果对象或列表的 512 个引用，而且返回的对象被认为是只读的，因此对它们进行修改可能会在不同线程中的调用者产生冲突。可用的清除策略有：

- LRU – 最近最少使用：移除最长时间不被使用的对象。
- FIFO – 先进先出：按对象进入缓存的顺序来移除它们。
- SOFT – 软引用：基于垃圾回收器状态和软引用规则移除对象。
- WEAK – 弱引用：更积极地基于垃圾收集器状态和弱引用规则移除对象。

默认的清除策略是 LRU。

**`flushInterval`**（刷新间隔）属性可以被设置为任意的正整数，设置的值应该是一个以毫秒为单位的合理时间量。 默认情况是不设置，也就是没有刷新间隔，缓存仅仅会在调用语句时刷新。

**`size`**（引用数目）属性可以被设置为任意正整数，要注意欲缓存对象的大小和运行环境中可用的内存资源。默认值是 1024。

**`readOnly`**（只读）属性可以被设置为 true 或 false。只读的缓存会给所有调用者返回缓存对象的相同实例。 因此这些对象不能被修改。这就提供了可观的性能提升。而可读写的缓存会（通过序列化）返回缓存对象的拷贝。 速度上会慢一些，但是更安全，因此默认值是 false。

　　注：二级缓存是事务性的。这意味着，当 SqlSession 完成并提交时，或是完成并回滚，但没有执行 `flushCache=true `的 `insert/delete/update` 语句时，缓存会获得更新。

　　`Mapper.xml` 配置了<cache>之后，select()会被缓存。update()、delete()、insert()会刷新缓存。：如果`cacheEnabled=true`，`Mapper.xml` 没有配置标签，还有二级缓存吗？（没有）还会出现CachingExecutor 包装对象吗？（会）

　　只要`cacheEnabled=true` 基本执行器就会被装饰。有没有配置<cache>，决定了在启动的时候会不会创建这个mapper 的Cache 对象，只是最终会影响到`CachingExecutorquery` 方法里面的判断。如果某些查询方法对数据的实时性要求很高，不需要二级缓存，怎么办？我们可以在单个Statement ID 上显式关闭二级缓存（默认是true）：

```xml
<select id="selectBlog" resultMap="BaseResultMap" useCache="false">
```

#### 第三方缓存做二级缓存

　　除了MyBatis 自带的二级缓存之外，我们也可以通过实现Cache 接口来自定义二级缓存。MyBatis 官方提供了一些第三方缓存集成方式，比如ehcache 和redis：https://github.com/mybatis/redis-cache ,这里就不过多介绍了。当然，我们也可以使用独立的缓存服务，不使用MyBatis 自带的二级缓存。

pom 文件引入依赖：

```java
<dependency>
	<groupId>org.mybatis.caches</groupId>
	<artifactId>mybatis-redis</artifactId>
	<version>1.0.0-beta2</version>
</dependency>
```

MybatisRedisCache

```java

import com.xxx.util.JsonUtils;
import org.apache.ibatis.cache.Cache;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.data.redis.core.RedisTemplate;
 
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;
 
/**
 * Mybatis - redis二级缓存
 *
 */
public final class MybatisRedisCache implements Cache {
    /**
     * 日志工具类
     */
    private static final Logger logger = LogManager.getLogger(MybatisRedisCache.class);
 
    /**
     * 读写锁
     */
    private final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    /**
     * ID
     */
    private String id;
 
    /**
     * 集成redisTemplate
     */
    private static RedisTemplate redisTemplate;
 
    public MybatisRedisCache() {
    }
 
    public MybatisRedisCache(String id) {
        if (id == null) {
            throw new IllegalArgumentException("Cache instances require an ID");
        } else {
            logger.debug("MybatisRedisCache.id={}", id);
            this.id = id;
        }
    }
 
 
    @Override
    public String getId() {
        return this.id;
    }
 
    @Override
    public int getSize() {
        try {
            Long size = redisTemplate.opsForHash().size(this.id.toString());
            logger.debug("MybatisRedisCache.getSize: {}->{}", id, size);
            return size.intValue();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return 0;
    }
 
    @Override
    public void putObject(final Object key, final Object value) {
        try {
            logger.debug("MybatisRedisCache.putObject: {}->{}->{}", id, key, JsonUtils.toJson(value));
            redisTemplate.opsForHash().put(this.id.toString(), key.toString(), value);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
 
    @Override
    public Object getObject(final Object key) {
        try {
            Object hashVal = redisTemplate.opsForHash().get(this.id.toString(), key.toString());
            logger.debug("MybatisRedisCache.getObject: {}->{}->{}", id, key, JsonUtils.toJson(hashVal));
            return hashVal;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
 
    @Override
    public Object removeObject(final Object key) {
        try {
            redisTemplate.opsForHash().delete(this.id.toString(), key.toString());
            logger.debug("MybatisRedisCache.removeObject: {}->{}->{}", id, key);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
 
    @Override
    public void clear() {
        try {
            redisTemplate.delete(this.id.toString());
            logger.debug("MybatisRedisCache.clear: {}", id);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
 
    @Override
    public ReadWriteLock getReadWriteLock() {
        return this.readWriteLock;
    }
 
    @Override
    public String toString() {
        return "MybatisRedisCache {" + this.id + "}";
    }
 
    /**
     * 设置redisTemplate
     *
     * @param redisTemplate
     */
    public void setRedisTemplate(RedisTemplate redisTemplate) {
        MybatisRedisCache.redisTemplate = redisTemplate;
    }
}
```

RedisConfig

```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

/**
 * Redis配置
 *
 */

@Configuration
public class RedisConfig {

    /**
     * 配置RedisTemplate
     *
     * @param factory
     * @return
     */
    @Bean
    public RedisTemplate redisTemplate(RedisConnectionFactory factory, Jackson2JsonRedisSerializer redisJsonSerializer) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        //redis连接工厂
        template.setConnectionFactory(factory);
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        //redis.key序列化器
        template.setKeySerializer(stringRedisSerializer);
        //redis.value序列化器
        template.setValueSerializer(redisJsonSerializer);
        //redis.hash.key序列化器
        template.setHashKeySerializer(stringRedisSerializer);
        //redis.hash.value序列化器
        template.setHashValueSerializer(redisJsonSerializer);
        //调用其他初始化逻辑
        template.afterPropertiesSet();
        //这里设置redis事务一致
        template.setEnableTransactionSupport(true);
        return template;
    }

    /**
     * 配置redis Json序列化器
     *
     * @return
     */
    @Bean
    public Jackson2JsonRedisSerializer redisJsonSerializer() {
        //使用Jackson2JsonRedisSerializer来序列化和反序列化redis的value值（默认使用JDK的序列化方式）
        Jackson2JsonRedisSerializer serializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        serializer.setObjectMapper(mapper);
        return serializer;
    }

}
```

开启Mybatis二级缓存设置

方式1：mybatis-config.xml

```java
<configuration>
    <settings>
        <!-- 开启二级缓存 -->
        <setting name="cacheEnabled" value="true"/>
    </settings>
   ...
</configuration>
```


方式2：Springboot - application.properties

```java
#使全局的映射器启用或禁用缓存。
mybatis.configuration.cache-enabled=true
```

Mapper.xml 配置，type 使用RedisCache：

```java
<cache type="org.mybatis.caches.redis.RedisCache"
eviction="FIFO" flushInterval="60000" size="512" readOnly="true"/>
```


redis.properties 配置：

```java
host=localhost
port=6379
connectionTimeout=5000
soTimeout=5000
database=0
```