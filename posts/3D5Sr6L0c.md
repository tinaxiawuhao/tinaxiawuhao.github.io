---
title: 'docker安装ELK'
date: 2021-05-27 15:39:14
tags: [docker,ELK]
published: true
hideInList: false
feature: /post-images/3D5Sr6L0c.png
isTop: false
---

## Docker部署ElasticSearch

### 搜索ElasticSearch镜像

```java
docker search elasticsearch
```

### 拉取镜像

拉取镜像的时候，可以指定版本，如果不指定，默认使用latest。

```java
docker pull elasticsearch:7.12.0
```

### 查看镜像

```java
docker images
```

### docker 启动 elasticsearch

```java
# --name : 为 elasticsearch 容器起个别名
# -e : 指定为单节点集群模式
# -i：表示运行容器
# -t：表示容器启动后会进入其命令行。加入这两个参数后，容器创建就能登录进去。即分配一个伪终端。
# -v：表示目录映射关系（前者是宿主机目录，后者是映射到宿主机上的目录），可以使用多个－v做多个目录或文件映射。注意：最好做目录映射，在宿主机上做修改，然后共享到容器上。
# -d：在run后面加上-d参数,则会创建一个守护式容器在后台运行（这样创建容器后不会自动登录容器，如果只加-i -t两个参数，创建后就会自动进去容器）。
# -p：表示端口映射，前者是宿主机端口，后者是容器内的映射端口。可以使用多个-p做多个端口映射
docker run -di --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.12.0 
```

### elasticsearch配置

#### 创建elasticsearch滚动策略

```json
# 定义审计日志管理策略
curl -X PUT "${host}/_ilm/policy/audit_policy" -H 'Content-Type: application/json' -d'
{
  "policy": {                       
    "phases": {
      "hot": {                      
        "actions": {
          "rollover": {             
            "max_size": "30GB",
            "max_age": "180d"
          }
        }
      },
      "delete": {
        "min_age": "180d",           
        "actions": {
          "delete": {}              
        }
      }
    }
  }
}
```

热数据最大30G,最多180天，数据最少保持180天后删除

#### 创建索引模板

```json
# 导出日志索引模板
curl -X PUT "${host}/_template/export_log_index_template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": [
    "export_log_index*"
  ],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "audit_policy",
    "index.lifecycle.rollover_alias": "export_log_index",
    "index.max_result_window": "100000"
  },
  "mappings": {
    "properties": {
      "applicationSide": {
        "type": "integer"
      },
      "exportComment": {
        "type": "keyword"
      },
      "exportFileSize": {
        "type": "integer"
      },
      "exportType": {
        "type": "integer"
      },
      "id": {
        "type": "keyword"
      },
      "operationTime": {
        "type": "date",
        "store": true,
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "operationUser": {
        "type": "keyword"
      },
      "operationUserName": {
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "remarks": {
        "type": "keyword"
      },
      "successful": {
        "type": "boolean"
      }
    }
  }
}
```

#### 创建索引

```json
# 创建导出日志索引，按日期命名
curl -X PUT "${host}/%3Cexport_log_index-%7Bnow%2Fd%7D-1%3E" -H 'Content-Type: application/json' -d'
{
  "aliases": {
    "export_log_index": {
      "is_write_index": true
    }
  }
}
```

## docker 安装 kibana

### 拉取镜像

拉取镜像的时候，需要注意的是, **kibana 的版本最好与 elasticsearch 保持一致**, 避免发生不必要的错误

```java
docker pull kibana:7.12.0
```

### 查看镜像

```java
docker images
```

### docker 启动 kibana

```java
# -e : 指定环境变量配置, 提供汉化
# --like : 建立两个容器之间的关联, kibana 关联到 es
docker run -di --name kibana --link elasticsearch:elasticsearch -e "I18N_LOCALE=zh-CN" -p 5601:5601 kibana:7.12.0
# kibana 的汉化我感觉做的并不好
# 如果不习惯汉化, 可以把条件去除
docker run -di --name kibana --link elasticsearch:elasticsearch -p 5601:5601 kibana:7.12.0
```

## Docker 安装 Logstash

### 拉取镜像

拉取镜像的时候，需要注意的是, **Logstash 的版本最好与 elasticsearch 保持一致**, 避免发生不必要的错误

```java
docker pull logstash:7.12.0
```

### 查看镜像

```java
docker images
```

### 文件映射

在本机建立配置文件和目录,用来存放所有配置的映射

```java
/usr/local/logstash/config/logstash.yml
/usr/local/logstash/conf.d/
```

logstash.yml (文件内容)

```java
path.config: /usr/share/logstash/conf.d/*.conf
path.logs: /var/log/logstash
```

conf.d/configuration.conf (文件内容)

**Filebeat和elasticsearch的交互**

```json
input {
    beats {
    port => 5044
    codec => "json"
}
}

output {
  elasticsearch { 
    hosts => ["elasticsearch:9200"]，
	index => "export_log_index"
  }
  stdout { codec => rubydebug }
}
```

**mysql和elasticsearch的交互**

```json
input {
  jdbc {
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/db_example"
    jdbc_user => root
    jdbc_password => ymruan123
    #启用追踪，如果为true，则需要指定tracking_column
    use_column_value => true
    #指定追踪的字段，
    tracking_column => "last_updated"
    #追踪字段的类型，目前只有数字(numeric)和时间类型(timestamp)，默认是数字类型
    tracking_column_type => "numeric"
    #记录最后一次运行的结果
    record_last_run => true
    #上面运行结果的保存位置
    last_run_metadata_path => "jdbc-position.txt"
    statement => "SELECT * FROM user where last_updated >:sql_last_value;"
    schedule => " * * * * * *"
  }
}
output {
  elasticsearch {
    document_id => "%{id}"
    document_type => "_doc"
    index => "export_log_index"
    hosts => ["http://localhost:9200"]
  }
  stdout{
    codec => rubydebug
  }
}
```

**kafka和elasticsearch的交互**

```json
input {
 kafka{
    topics => "topic_export"    #kafka中topic名称，记得创建该topic
    group_id => "group_export"     #默认为“logstash”
    codec => "json"    #与Shipper端output配置项一致
    consumer_threads => 1    #消费线程数，集群中所有logstash相加最好等于 topic 分区数
    bootstrap_servers => "kafka:9092"
    decorate_events => true    #在输出消息的时候回输出自身的信息，包括：消费消息的大小、topic来源以及consumer的group信息。
    type => "topic_export"  
    tags => ["canal"] # 标签，额外使用该参数可以在elastci中创建不同索引
  }
  
}
 
 
filter {
  # 把默认的data字段重命名为message字段，方便在elastic中显示
  mutate {
    rename => ["data", "message"]
  }
  # 还可以使用其他的处理方式，在此就不再列出来了
}
 
output {
  elasticsearch {
    hosts => ["http://172.17.107.187:9203", "http://172.17.107.187:9201","http://172.17.107.187:9202"]
    index => "export_log_index" # decorate_events=true的作用，可以使用metadata中的数据
    #user => "elastic"
    #password => "escluter123456"
  }
 }
```

**logback和和elasticsearch的交互**

引入依赖

```xml
<dependency>
      <groupId>net.logstash.logback</groupId>
      <artifactId>logstash-logback-encoder</artifactId>
     <version>4.10</version>
</dependency>
```

参数配置

```json
input {
  # 应用日志
  tcp{
    type => "app"
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json
  }
}

output {
   elasticsearch {
      hosts => "http://127.0.0.1:9200"
      index => "export_log_index"
    }
}

```

应用日志入口端口为4560，需要配置java客户端logstash入口

```xml
<!-- 这个是控制台日志输出格式 方便调试对比-->
<appender name="console" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss} %contextName %-5level %logger{50} -%msg%n</pattern>
    </encoder>
</appender>
<!--开启tcp格式的logstash传输，通过TCP协议连接Logstash-->
<appender name="STASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>127.0.0.1:9600</destination>

    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
        <!--中文序列化-->
        <jsonFactoryDecorator class="net.logstash.logback.decorate.CharacterEscapesJsonFactoryDecorator">
            <escape>
                <targetCharacterCode>10</targetCharacterCode>
                <escapeSequence>\u2028</escapeSequence>
            </escape>
        </jsonFactoryDecorator>
        <providers>
            <pattern>
                <pattern>
                    <!--{
                    "timestamp":"%date{ISO8601}",
                    "user":"test",
                    "message":"[%d{yyyy-MM-dd HH:mm:ss.SSS}][%p][%t][%l{80}|%L]%m"}%n
                    }-->
                    {
                    "timestamp": "%date{\"yyyy-MM-dd' 'HH:mm:ss,SSSZ\"}",
                    "level": "%level",
                    "thread": "%thread",
                    "class_name": "%class",
                    "line_number": "%line",
                    "message": "%message",
                    "stack_trace": "%exception{5}",
                    "req_id": "%X{reqId}",
                    "elapsed_time": "#asLong{%X{elapsedTime}}"
                    }
                </pattern>
            </pattern>
        </providers>
        <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->


    </encoder>
    <keepAliveDuration>5 minutes</keepAliveDuration>
</appender>

<root level="INFO">
    <appender-ref ref="STASH"/>
    <appender-ref ref="console"/>
</root>
```



### docker 启动 Logstash

```java
#docker容器互访 运行容器的时候加上参数link
docker run -id -p 5044:5044 --name logstash --link elasticsearch --link beats -v /usr/local/logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml -v /usr/local/logstash/conf.d/:/usr/share/logstash/conf.d/ logstash:7.4.1
```

## Docker 安装 Filebeat

### 拉取镜像

拉取镜像的时候，需要注意的是, **Filebeat 的版本最好与 elasticsearch 保持一致**, 避免发生不必要的错误

```java
docker pull store/elastic/filebeat:7.12.0
```

### 查看镜像

```java
docker images
```

### 文件映射

**下载默认官方配置文件**

```http
wget https://raw.githubusercontent.com/elastic/beats/7.12/deploy/docker/filebeat.docker.yml
```

注意：文件放在宿主机/data/elk/filebeat目录下

**打开配置文件**
`vim filebeat.docker.yml`，内容如下：

```xml
# 日志输入配置
filebeat.inputs:
- type: log
  enabled: true
  paths:
  # 需要收集的日志所在的位置，可使用通配符进行配置
  #- /data/elk/*.log
  - /logs/*/*.log

#日志输出配置(采用 logstash 收集日志，5044为logstash端口)
output.logstash:
  hosts: ['192.168.12.183:5044']
```

### docker运行Filebeat

```java
docker run --name filebeat --user=root -d --net somenetwork --volume="/usr/local/filebeat/log/nginx/:/var/log/nginx/" --volume="/data/elk/filebeat/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml" --volume="/var/lib/docker/containers:/var/lib/docker/containers:ro" --volume="/var/run/docker.sock:/var/run/docker.sock:ro" store/elastic/filebeat:7.12.0
```