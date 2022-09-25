---
title: 'springBoot,mybatis,druid整合'
date: 2021-05-26 17:02:37
tags: [springBoot,druid,mybatis]
published: true
hideInList: false
feature: /post-images/Ari4Sf8A0.png
isTop: false
---
## springBoot,mybatis,druid单数据源整合
### 项目框架

![](https://tinaxiawuhao.github.io/post-images/1621415131656.png)

#### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>datasources</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>datasources</name>
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.45</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.10</version>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.2.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

#### application.yml

```yaml
spring:
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql://localhost:3306/datatest?characterEncoding=utf-8&useSSl=false
    driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      initialSize: 10 # 初始化时建立物理连接的个数。初始化发生在显示调用init方法，或者第一次getConnection时
      minIdle: 10 # 最小连接池数量
      maxActive: 200 # 最大连接池数量
      maxWait: 60000 # 获取连接时最大等待时间，单位毫秒。配置了maxWait之后，缺省启用公平锁，并发效率会有所下降，如果需要可以通过配置
      timeBetweenEvictionRunsMillis: 60000 # 关闭空闲连接的检测时间间隔.Destroy线程会检测连接的间隔时间，如果连接空闲时间大于等于minEvictableIdleTimeMillis则关闭物理连接。
      minEvictableIdleTimeMillis: 300000 # 连接的最小生存时间.连接保持空闲而不被驱逐的最小时间
      validationQuery: SELECT 1 FROM DUAL # 验证数据库服务可用性的sql.用来检测连接是否有效的sql 因数据库方言而差, 例如 oracle 应该写成 SELECT 1 FROM DUAL
      testWhileIdle: true # 申请连接时检测空闲时间，根据空闲时间再检测连接是否有效.建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRun
      testOnBorrow: false # 申请连接时直接检测连接是否有效.申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
      testOnReturn: false # 归还连接时检测连接是否有效.归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
      poolPreparedStatements: true # 开启PSCache
      maxPoolPreparedStatementPerConnectionSize: 20 #设置PSCache值
      connectionErrorRetryAttempts: 3 # 连接出错后再尝试连接三次
      breakAfterAcquireFailure: true # 数据库服务宕机自动重连机制
      timeBetweenConnectErrorMillis: 300000 # 连接出错后重试时间间隔
      asyncInit: true # 异步初始化策略
      remove-abandoned: true # 是否自动回收超时连接
      remove-abandoned-timeout: 1800 # 超时时间(以秒数为单位)
      transaction-query-timeout: 6000 # 事务超时时间
      filters: stat,wall,log4j2
      connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=500


#mybatis是独立节点，需要单独配置
mybatis-plus:
  mapper-locations: classpath*:mapper/*.xml
  type-aliases-package: com.example.datasources.entity
  configuration:
    map-underscore-to-camel-case: true
```

#### DruidConfig

```java
@Configuration
public class DruidConfig {

    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean<StatViewServlet> bean = new ServletRegistrationBean<StatViewServlet>( new StatViewServlet(), "/druid/*" );

        Map<String,String> iniParms=new HashMap<>(  );

        iniParms.put( "loginUsername","admin" );//登录druid的用户名
        iniParms.put( "loginPassword","123456" );//登录druid的密码
        iniParms.put("allow","");//默认允许所有
        //iniParms.put( "deny","192.168.***.***" );//拒绝的ip地址
        bean.setInitParameters( iniParms );
        return bean;

    }

    @Bean
    public FilterRegistrationBean webStatFilter(){
        FilterRegistrationBean bean= new FilterRegistrationBean();
        bean.setFilter(new WebStatFilter());
        Map<String,String> iniParms=new HashMap<>();
        iniParms.put( "excliusions", "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*");//使静态文件访问，还有/druid/* 的访问不被拦截
        bean.setInitParameters( iniParms );
        bean.setUrlPatterns( Arrays.asList("/*"));
        return bean;
    }

}
```

#### 访问Druid

[http://localhost:8080/druid/login.html](http://localhost:8080/druid/login.html)

![](https://tinaxiawuhao.github.io/post-images/1621415196405.png)

![](https://tinaxiawuhao.github.io/post-images/1621415233878.png)

## springBoot,mybatis,druid多数据源整合

### 项目架构

![](https://tinaxiawuhao.github.io/post-images/1621415260516.png)

#### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>datasources</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>datasources</name>
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.45</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.10</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.2.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

#### application.yml

```yaml
spring:
  datasource:
    #使用druid连接池
    type: com.alibaba.druid.pool.DruidDataSource

# 自定义的主数据源配置信息
primary:
  datasource:
    #druid相关配置
    druid:
      #监控统计拦截的filters
      filters: stat
      driverClassName: com.mysql.jdbc.Driver
      #配置基本属性
      url: jdbc:mysql://127.0.0.1:3306/primary_database?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true&autoReconnect=true&useSSL=false
      username: root
      password: 123456
      #配置初始化大小/最小/最大
      initialSize: 1
      minIdle: 1
      maxActive: 20
      #获取连接等待超时时间
      maxWait: 60000
      #间隔多久进行一次检测，检测需要关闭的空闲连接
      timeBetweenEvictionRunsMillis: 60000
      #一个连接在池中最小生存的时间
      minEvictableIdleTimeMillis: 300000
      validationQuery: SELECT 'x'
      testWhileIdle: true
      testOnBorrow: false
      testOnReturn: false
      #打开PSCache，并指定每个连接上PSCache的大小。oracle设为true，mysql设为false。分库分表较多推荐设置为false
      poolPreparedStatements: false
      maxPoolPreparedStatementPerConnectionSize: 20

# 自定义的从数据源配置信息
back:
  datasource:
    #druid相关配置
    druid:
      #监控统计拦截的filters
      filters: stat
      driverClassName: com.mysql.jdbc.Driver
      #配置基本属性
      url: jdbc:mysql://127.0.0.1:3306/back_database?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true&autoReconnect=true&useSSL=false
      username: root
      password: 123456
      #配置初始化大小/最小/最大
      initialSize: 1
      minIdle: 1
      maxActive: 20
      #获取连接等待超时时间
      maxWait: 60000
      #间隔多久进行一次检测，检测需要关闭的空闲连接
      timeBetweenEvictionRunsMillis: 60000
      #一个连接在池中最小生存的时间
      minEvictableIdleTimeMillis: 300000
      validationQuery: SELECT 'x'
      testWhileIdle: true
      testOnBorrow: false
      testOnReturn: false
      #打开PSCache，并指定每个连接上PSCache的大小。oracle设为true，mysql设为false。分库分表较多推荐设置为false
      poolPreparedStatements: false
      maxPoolPreparedStatementPerConnectionSize: 20
```

#### DruidConfig

```java
@Configuration
public class DruidConfig {

    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean<StatViewServlet> bean = new ServletRegistrationBean<StatViewServlet>( new StatViewServlet(), "/druid/*" );

        Map<String,String> iniParms=new HashMap<>(  );

        iniParms.put( "loginUsername","admin" );//登录druid的用户名
        iniParms.put( "loginPassword","123456" );//登录druid的密码
        iniParms.put("allow","");//默认允许所有
        //iniParms.put( "deny","192.168.***.***" );//拒绝的ip地址
        bean.setInitParameters( iniParms );
        return bean;

    }

    @Bean
    public FilterRegistrationBean webStatFilter(){
        FilterRegistrationBean bean= new FilterRegistrationBean();
        bean.setFilter(new WebStatFilter());
        Map<String,String> iniParms=new HashMap<>();
        iniParms.put( "excliusions", "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*");//使静态文件访问，还有/druid/* 的访问不被拦截
        bean.setInitParameters( iniParms );
        bean.setUrlPatterns( Arrays.asList("/*"));
        return bean;
    }

}
```

#### PrimaryDataBaseConfig

```java
/**
 * @Description: 主数据源配置类
 */
@Data
@Configuration
// 前缀为primary.datasource.druid的配置信息
@ConfigurationProperties(prefix = "primary.datasource.druid")
@MapperScan(basePackages = PrimaryDataBaseConfig.PACKAGE, sqlSessionFactoryRef = "primarySqlSessionFactory")
public class PrimaryDataBaseConfig {
 
    /**
     * dao层的包路径
     */
    static final String PACKAGE = "com.example.datasources.dao.primary";
 
    /**
     * mapper文件的相对路径
     */
    private static final String MAPPER_LOCATION = "classpath:mapper/primary/*.xml";
 
    private String filters;
    private String url;
    private String username;
    private String password;
    private String driverClassName;
    private int initialSize;
    private int minIdle;
    private int maxActive;
    private long maxWait;
    private long timeBetweenEvictionRunsMillis;
    private long minEvictableIdleTimeMillis;
    private String validationQuery;
    private boolean testWhileIdle;
    private boolean testOnBorrow;
    private boolean testOnReturn;
    private boolean poolPreparedStatements;
    private int maxPoolPreparedStatementPerConnectionSize;
 
    // 主数据源使用@Primary注解进行标识
    @Primary
    @Bean(name = "primaryDataSource")
    public DataSource primaryDataSource() throws SQLException {
        DruidDataSource druid = new DruidDataSource();
        // 监控统计拦截的filters
        druid.setFilters(filters);
 
        // 配置基本属性
        druid.setDriverClassName(driverClassName);
        druid.setUsername(username);
        druid.setPassword(password);
        druid.setUrl(url);
 
        //初始化时建立物理连接的个数
        druid.setInitialSize(initialSize);
        //最大连接池数量
        druid.setMaxActive(maxActive);
        //最小连接池数量
        druid.setMinIdle(minIdle);
        //获取连接时最大等待时间，单位毫秒。
        druid.setMaxWait(maxWait);
        //间隔多久进行一次检测，检测需要关闭的空闲连接
        druid.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
        //一个连接在池中最小生存的时间
        druid.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
        //用来检测连接是否有效的sql
        druid.setValidationQuery(validationQuery);
        //建议配置为true，不影响性能，并且保证安全性。
        druid.setTestWhileIdle(testWhileIdle);
        //申请连接时执行validationQuery检测连接是否有效
        druid.setTestOnBorrow(testOnBorrow);
        druid.setTestOnReturn(testOnReturn);
        //是否缓存preparedStatement，也就是PSCache，oracle设为true，mysql设为false。分库分表较多推荐设置为false
        druid.setPoolPreparedStatements(poolPreparedStatements);
        // 打开PSCache时，指定每个连接上PSCache的大小
        druid.setMaxPoolPreparedStatementPerConnectionSize(maxPoolPreparedStatementPerConnectionSize);
 
        return druid;
    }
 
    // 创建该数据源的事务管理
    @Primary
    @Bean(name = "primaryTransactionManager")
    public DataSourceTransactionManager primaryTransactionManager() throws SQLException {
        return new DataSourceTransactionManager(primaryDataSource());
    }
 
    // 创建Mybatis的连接会话工厂实例
    @Primary
    @Bean(name = "primarySqlSessionFactory")
    public SqlSessionFactory primarySqlSessionFactory(@Qualifier("primaryDataSource") DataSource primaryDataSource) throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(primaryDataSource);  // 设置数据源bean
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources(PrimaryDataBaseConfig.MAPPER_LOCATION));  // 设置mapper文件路径
 
        return sessionFactory.getObject();
    }
}
```

#### BackDataBaseConfig

```java
/**
 * @Description: 从数据源配置类
 */
@Data
@Configuration
@ConfigurationProperties(prefix = "back.datasource.druid")
@MapperScan(basePackages = BackDataBaseConfig.PACKAGE, sqlSessionFactoryRef = "backSqlSessionFactory")
public class BackDataBaseConfig {
 
    /**
     * dao层的包路径
     */
    static final String PACKAGE = "com.example.datasources.dao.back";
 
    /**
     * mapper文件的相对路径
     */
    private static final String MAPPER_LOCATION = "classpath:mapper/back/*.xml";
 
    private String filters;
    private String url;
    private String username;
    private String password;
    private String driverClassName;
    private int initialSize;
    private int minIdle;
    private int maxActive;
    private long maxWait;
    private long timeBetweenEvictionRunsMillis;
    private long minEvictableIdleTimeMillis;
    private String validationQuery;
    private boolean testWhileIdle;
    private boolean testOnBorrow;
    private boolean testOnReturn;
    private boolean poolPreparedStatements;
    private int maxPoolPreparedStatementPerConnectionSize;
 
    @Bean(name = "backDataSource")
    public DataSource backDataSource() throws SQLException {
        DruidDataSource druid = new DruidDataSource();
        // 监控统计拦截的filters
        druid.setFilters(filters);
 
        // 配置基本属性
        druid.setDriverClassName(driverClassName);
        druid.setUsername(username);
        druid.setPassword(password);
        druid.setUrl(url);
 
        //初始化时建立物理连接的个数
        druid.setInitialSize(initialSize);
        //最大连接池数量
        druid.setMaxActive(maxActive);
        //最小连接池数量
        druid.setMinIdle(minIdle);
        //获取连接时最大等待时间，单位毫秒。
        druid.setMaxWait(maxWait);
        //间隔多久进行一次检测，检测需要关闭的空闲连接
        druid.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
        //一个连接在池中最小生存的时间
        druid.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
        //用来检测连接是否有效的sql
        druid.setValidationQuery(validationQuery);
        //建议配置为true，不影响性能，并且保证安全性。
        druid.setTestWhileIdle(testWhileIdle);
        //申请连接时执行validationQuery检测连接是否有效
        druid.setTestOnBorrow(testOnBorrow);
        druid.setTestOnReturn(testOnReturn);
        //是否缓存preparedStatement，也就是PSCache，oracle设为true，mysql设为false。分库分表较多推荐设置为false
        druid.setPoolPreparedStatements(poolPreparedStatements);
        // 打开PSCache时，指定每个连接上PSCache的大小
        druid.setMaxPoolPreparedStatementPerConnectionSize(maxPoolPreparedStatementPerConnectionSize);
 
        return druid;
    }
 
    @Bean(name = "backTransactionManager")
    public DataSourceTransactionManager backTransactionManager() throws SQLException {
        return new DataSourceTransactionManager(backDataSource());
    }
 
    @Bean(name = "backSqlSessionFactory")
    public SqlSessionFactory backSqlSessionFactory(@Qualifier("backDataSource") DataSource backDataSource) throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(backDataSource);
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources(BackDataBaseConfig.MAPPER_LOCATION));
 
        return sessionFactory.getObject();
    }
}
```

#### 访问Druid

[http://localhost:8080/druid/login.html](http://localhost:8080/druid/login.html)

![](https://tinaxiawuhao.github.io/post-images/1621415300791.png)

![](https://tinaxiawuhao.github.io/post-images/1621415336496.png)

## springBoot,mybatis,druid主从备份

### 项目架构

![](https://tinaxiawuhao.github.io/post-images/1621481203900.png)

#### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>datasources</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>datasources</name>
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.45</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.10</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.2.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>

```

#### application.yml

```yaml
spring:
  datasource:
    #使用druid连接池
    type: com.alibaba.druid.pool.DruidDataSource

# 自定义的主数据源配置信息
primary:
  datasource:
    #druid相关配置
    druid:
      #监控统计拦截的filters
      filters: stat
      driverClassName: com.mysql.jdbc.Driver
      #配置基本属性
      url: jdbc:mysql://127.0.0.1:3306/primary_database?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true&autoReconnect=true&useSSL=false
      username: root
      password: 123456
      #配置初始化大小/最小/最大
      initialSize: 1
      minIdle: 1
      maxActive: 20
      #获取连接等待超时时间
      maxWait: 60000
      #间隔多久进行一次检测，检测需要关闭的空闲连接
      timeBetweenEvictionRunsMillis: 60000
      #一个连接在池中最小生存的时间
      minEvictableIdleTimeMillis: 300000
      validationQuery: SELECT 'x'
      testWhileIdle: true
      testOnBorrow: false
      testOnReturn: false
      #打开PSCache，并指定每个连接上PSCache的大小。oracle设为true，mysql设为false。分库分表较多推荐设置为false
      poolPreparedStatements: false
      maxPoolPreparedStatementPerConnectionSize: 20

# 自定义的从数据源配置信息
back:
  datasource:
    #druid相关配置
    druid:
      #监控统计拦截的filters
      filters: stat
      driverClassName: com.mysql.jdbc.Driver
      #配置基本属性
      url: jdbc:mysql://127.0.0.1:3306/back_database?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true&autoReconnect=true&useSSL=false
      username: root
      password: 123456
      #配置初始化大小/最小/最大
      initialSize: 1
      minIdle: 1
      maxActive: 20
      #获取连接等待超时时间
      maxWait: 60000
      #间隔多久进行一次检测，检测需要关闭的空闲连接
      timeBetweenEvictionRunsMillis: 60000
      #一个连接在池中最小生存的时间
      minEvictableIdleTimeMillis: 300000
      validationQuery: SELECT 'x'
      testWhileIdle: true
      testOnBorrow: false
      testOnReturn: false
      #打开PSCache，并指定每个连接上PSCache的大小。oracle设为true，mysql设为false。分库分表较多推荐设置为false
      poolPreparedStatements: false
      maxPoolPreparedStatementPerConnectionSize: 20
```

#### DruidConfig

```java
@Configuration
public class DruidConfig {
 
    public final static String MAPPER_XML_PATH = "classpath:mapper/*.xml";
 
    @ConfigurationProperties(prefix = "master.datasource.druid")
    @Bean(name = "masterDataSource")
    public DataSource masterDataSource() {
        return new DruidDataSource();
    }
 
 
    @Bean
    public PlatformTransactionManager txManager(DataSource dynamicDataSource) {
        return new DataSourceTransactionManager(dynamicDataSource);
    }
 
 
    @ConfigurationProperties(prefix = "slave.datasource.druid")
    @Bean
    public DataSource slaveDataSource(){
        return  new DruidDataSource();
    }
 
 
    @Bean
    public DynamicDataSource dynamicDataSource(){
        DynamicDataSource dynamicDataSource=new DynamicDataSource();
        Map<Object,Object> map=new HashMap<>();
        map.put(DbUtil.master,masterDataSource());
        map.put(DbUtil.slave,slaveDataSource());
        dynamicDataSource.setDefaultTargetDataSource(masterDataSource());
        dynamicDataSource.setTargetDataSources(map);
        return dynamicDataSource;
    }
 
    @Bean
    public SqlSessionFactoryBean sqlSessionFactoryBean(DataSource dynamicDataSource) throws IOException {
        SqlSessionFactoryBean sqlSessionFactory = new SqlSessionFactoryBean();
        sqlSessionFactory.setDataSource(dynamicDataSource);
        sqlSessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(MAPPER_XML_PATH));
        return sqlSessionFactory;
    }
 
    @Bean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactoryBean sqlSessionFactoryBean) throws Exception {
        SqlSessionTemplate sqlSessionTemplate = new SqlSessionTemplate(sqlSessionFactoryBean.getObject());
        return sqlSessionTemplate;
    }

    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean<StatViewServlet> bean = new ServletRegistrationBean<StatViewServlet>( new StatViewServlet(), "/druid/*" );

        Map<String,String> iniParms=new HashMap<>(  );

        iniParms.put( "loginUsername","admin" );//登录druid的用户名
        iniParms.put( "loginPassword","123456" );//登录druid的密码
        iniParms.put("allow","");//默认允许所有
        //iniParms.put( "deny","192.168.***.***" );//拒绝的ip地址
        bean.setInitParameters( iniParms );
        return bean;

    }

    @Bean
    public FilterRegistrationBean webStatFilter(){
        FilterRegistrationBean bean= new FilterRegistrationBean();
        bean.setFilter(new WebStatFilter());
        Map<String,String> iniParms=new HashMap<>();
        iniParms.put( "excliusions", "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*");//使静态文件访问，还有/druid/* 的访问不被拦截
        bean.setInitParameters( iniParms );
        bean.setUrlPatterns( Arrays.asList("/*"));
        return bean;
    }
}@Configuration
public class DruidConfig {

    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean<StatViewServlet> bean = new ServletRegistrationBean<StatViewServlet>( new StatViewServlet(), "/druid/*" );

        Map<String,String> iniParms=new HashMap<>(  );

        iniParms.put( "loginUsername","admin" );//登录druid的用户名
        iniParms.put( "loginPassword","123456" );//登录druid的密码
        iniParms.put("allow","");//默认允许所有
        //iniParms.put( "deny","192.168.***.***" );//拒绝的ip地址
        bean.setInitParameters( iniParms );
        return bean;

    }

    @Bean
    public FilterRegistrationBean webStatFilter(){
        FilterRegistrationBean bean= new FilterRegistrationBean();
        bean.setFilter(new WebStatFilter());
        Map<String,String> iniParms=new HashMap<>();
        iniParms.put( "excliusions", "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*");//使静态文件访问，还有/druid/* 的访问不被拦截
        bean.setInitParameters( iniParms );
        bean.setUrlPatterns( Arrays.asList("/*"));
        return bean;
    }

}
```

#### MasterDataSource

```java
/**
 * 自定义主数据库注解
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MasterDataSource {
    String value() default "";
}
```

#### DatabaseAOP

```java
/**
 * aop从dao层判断使用哪个数据库
 * @MasterDataSource标识使用主数据库
 * 不标识使用从数据库
 */
@Aspect
@Component
public class DatabaseAOP {
    @Pointcut(value = "execution(* com.example.datasources.dao..*.*(..))")
    public void pointCut() {
 
    }
 
    @Before("pointCut()")
    public void before(JoinPoint joinPoint) {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        boolean isExist = method.isAnnotationPresent(MasterDataSource.class);
        if (!isExist) {
            DbUtil.setDb(DbUtil.slave);
            return;
        }
        DbUtil.setDb(DbUtil.master);
    }
}
```

#### DbUtil

```java
/**
 * 存储当前线程使用数据库标识
 */
public class DbUtil {
    public static String master="master";
    public static String slave="slave";
 
    private static final ThreadLocal<String> threadLocal=new ThreadLocal();
 
 
    public static void setDb(String db){
        threadLocal.set(db);
    }
 
    public static String getDb(){
        return threadLocal.get();
    }
 
}
```

#### DynamicDataSource

```java
/**
 * spring的jdbc提供了动态数据源的入口
 * 继承AbstractRoutingDataSource覆盖determineCurrentLookupKey()方法 返回当前使用数据库
 */
@Slf4j
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        log.info("当前使用数据库：{}", DbUtil.getDb());
        return DbUtil.getDb();
    }
}
```

#### CityMapper

```java
/**
  * springBoot启动入口DatasourcesApplication注解@MapperScan(basePackages = "com.example.datasources.dao")
  * 起到和@Mapper一样的作用
  * 添加了@Mapper注解之后这个接口在编译时会生成相应的实现类，不再需要写mapper映射文件，可以按下方@Select("select * from city where city_name like CONCAT('%', #{cityName},'%')")实现功能，@Autowired注解也需要通过@Mapper注解实现接口的动态代理实现类
  */
@Mapper
@Component
public interface CityMapper {
	/**
     * @MasterDataSource指定使用主数据库新增
     * 不指定默认使用从数据库
     */
    @MasterDataSource
    void insertCity(City city);
    /**
     * 根据城市名称，查询城市信息
     *
     * @param cityName 城市名
     * 默认使用从数据库
     */
    //@Select("select * from city where city_name like CONCAT('%', #{cityName},'%')")
    List<City> selectByName(@Param("cityName") String cityName);
}
```

#### CityMapper.xml

```java
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.datasources.dao.CityMapper">
    <resultMap id="BaseResultMap" type="com.example.datasources.entity.City">
        <id column="id" jdbcType="INTEGER" property="id"/>
        <result column="city_name" jdbcType="VARCHAR" property="cityName"/>
    </resultMap>
    <sql id="Base_Column_List">
        id,city_name
    </sql>
    <insert id="insertCity" parameterType="com.example.datasources.entity.City">
        INSERT into city (id,city_name) VALUES (#{id,jdbcType=INTEGER}, #{cityName,jdbcType=VARCHAR});
    </insert>
    <select id="selectByName" resultMap="BaseResultMap">
        select
        <include refid="Base_Column_List"/>
        from city
        where city_name like CONCAT('%', #{cityName},'%')
    </select>
</mapper>
```