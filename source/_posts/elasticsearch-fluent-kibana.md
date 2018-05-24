---
title: Elasticsearch+Fluentd+Kibana使用
tags: [Elasticsearch, Docker, Fluentd, Kibana]
date: 2018-05-24
---

## 前言
分布式应用，部署在Docker中，日志分散在各个服务器上，为了方便查看日志，
在应用中使用Fluentd把日志收集起来，存放到Elasticsearch中，使用Kibana来查看日志，分析等

## 编译配置Fluentd
Fluentd配合elasticsearch使用需要安装fluent-plugin-elasticsearch
下面来构建Dokerfile来编译一个带elasticsearch插件的Fluentd

```
FROM fluent/fluentd:v0.12-debian
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.2"]

```
编译镜像
```
# docker build -t fluent-plugin-elastic:1.9.2 ./
```
准备好fluent配置文件
fluentd/conf/fluent.conf
```
# fluentd/conf/fluent.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<match *.**>
  @type copy
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix fluentd
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name access_log
    tag_key @log_name
    flush_interval 1s
  </store>
  <store>
    @type stdout
  </store>
</match>
```
详细配置文件语法：https://docs.fluentd.org/v1.0/articles/config-file

## 部署efk(elasticsearch fluent kibana)
docker-compose.yaml
```
version: '3'
services:
  fluentd:
    image: fluent-plugin-elastic:1.9.2
    volumes:
      #挂载fluent配置文件
      - ./fluent/conf:/fluentd/etc
    environment:
      #配置elasticsearch主机端口
      - "ELASTICSEARCH_URL=http://172.19.0.1:9200"
    ports:
      - "24224:24224"
      - "24224:24224/udp"

  elasticsearch:
    image: elasticsearch:5.6.9-alpine
    volumes:
      - ./elasticsearch:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:	
    image: kibana:5.6.9
    environment:
      - "ELASTICSEARCH_URL=http://172.19.0.1:9200"
    ports:
      - "5601:5601"
    networks: 
      - efk_default
networks:
  efk_default:
    external: true

```
注意：ELASTICSEARCH_URL环境变量参数，可以配置成宿主机内网ip或者其他elasticsearch主机端口

部署
```
# docker stack deploy -c docker-compose.yaml efk
```

## 在项目中使用fluentd收集日志
以Java springboot项目为例子
添加maven依赖
```
        <dependency>
            <groupId>org.fluentd</groupId>
            <artifactId>fluent-logger</artifactId>
            <version>0.3.3</version>
        </dependency>

        <dependency>
            <groupId>com.sndyuk</groupId>
            <artifactId>logback-more-appenders</artifactId>
            <version>1.4.3</version>
        </dependency>
```
增加logback-spring.xml配置文件
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds" debug="true"
	xmlns="http://ch.qos.logback/xml/ns/logback" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://ch.qos.logback/xml/ns/logback http://ch.qos.logback/xml/ns/logback/logback.xsd">
	<property name="fileName" value="maxfunSpider-console" />
    
    <!-- ${FLUENTD_HOST: -xx.xx.xx.xx 意思是从配置文件获取FLUENTD_HOST，如果没有就用默认值-->
    <property name="FLUENTD_HOST" value="${FLUENTD_HOST:-服务器IP}" />
    <property name="FLUENTD_PORT" value="${FLUENTD_PORT:-24224}" />
    
	<include resource="org/springframework/boot/logging/logback/base.xml" />
  
	<appender name="FILE"
		class="ch.qos.logback.core.rolling.RollingFileAppender">
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<fileNamePattern>./log/${fileName}.%d{yyyy-MM-dd}.log
			</fileNamePattern>
			<maxHistory>30</maxHistory>
		</rollingPolicy>

		<encoder>
			<pattern>${CONSOLE_LOG_PATTERN}</pattern>
		</encoder>
	</appender>    
    
    <springProfile name="dev">
        <!-- 测试环境日志级别 -->
        <property name="loggerLevel" value="INFO" />
        <appender name="FLUENT" class="ch.qos.logback.more.appenders.DataFluentAppender">
            <tag>maxfunCrawler</tag>
            <label>dev</label>
            <remoteHost>${FLUENTD_HOST}</remoteHost>
            <port>${FLUENTD_PORT}</port>
        </appender>
    </springProfile>
    
    <springProfile name="prod">
        <!-- 生成环境日志级别 -->
        <property name="loggerLevel" value="ERROR" />
        <appender name="FLUENT" class="ch.qos.logback.more.appenders.DataFluentAppender">
            <tag>maxfunCrawler</tag>
            <label>prod</label>
            <remoteHost>${FLUENTD_HOST}</remoteHost>
            <port>${FLUENTD_PORT}</port>
        </appender>
    </springProfile>
    
    <root level="${loggerLevel}">
		<appender-ref ref="FILE" />
		<appender-ref ref="FLUENT" />
	</root>
</configuration>
```

启动springboot项目，不出意外的话，在Kibana就可以看到收集到的日志了

## 使用
部署成功后，访问ip:5601浏览Kibana页面了，第一次打开是如下界面。
![avatar](http://7xp9ph.com1.z0.glb.clouddn.com/blog/images/kibana.png)

把Index pattern改成fluentd-*，点Create。如果elasticsearch已经有日志的话，会显示出来，没有的就是这样子的
![avatar](http://7xp9ph.com1.z0.glb.clouddn.com/blog/images/kibana-noresult.png)

收集到的日志
![avatar](http://7xp9ph.com1.z0.glb.clouddn.com/blog/images/kibana-log.png)

## 增加http验证
到处就可以用Kibana来分析收集到的日志了，目前任何人都访问kibana，没有设置登录验证，不安全，可以配合treafik做个简单的http auth验证。

### 配置部署trarfik
配置
trarfik.toml来自https://raw.githubusercontent.com/containous/traefik/master/traefik.sample.toml
把entryPoints改成下面这样。
```
[entryPoints]
    [entryPoints.http]
    address = ":80"
    [entryPoints.http.auth.basic]
    #可以使用在线的htpasswd生成
    users = ["test:$apr1$CMMXhsE1$ltlr0YMcFYV16nKB.L9Ah0"]

```

traefik.yaml
```
ersion: "3"
services:
  traefik:
    image: traefik
    #--docker.domain可以根据自己需求修改
    command: --docker --docker.swarmmode  --docker.domain=kevin.co --docker.watch  --web --api --logLevel=DEBUG
    ports:
      #这台机器的80端口被占用了，所以用81端口。
      - "81:80"
      #traefik dashboard端口
      - "8080:8080"
      - "443:443"
    deploy:
      placement:
        constraints:
        - node.role == manager
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      #挂载配置文件
      - $PWD/traefik.toml:/etc/traefik/traefik.toml
    networks:
    #使用跟kibana同样的网络
    - efk_default
networks:
  efk_default:
    external: true
```


部署traefik
```
# docker stack deploy -c traefik.yaml efk
```

把kibana的docker-compose稍微改一下，增加deploy属性，加入相关traefik label。 
```
  kibana:	
    image: kibana:5.6.9
    #links:
    #  - "elasticsearch"
    environment:
      - "ELASTICSEARCH_URL=http://172.19.0.1:9200"
    #ports:
    #  - "5601:5601"
    deploy:
      labels:
        - "traefik.port=5601"
        - "traefik.docker.network=efk_default"
        - "traefik.backend=kibana"
        - "traefik.frontend.rule=Host:kibana.kevin.co"
    networks: 
      - efk_default
```
### 重新部署kibana
```
# docker service rm efk_kibana
# docker stack deploy -c docker-compose.yaml efk
```
kibana.kevin.co域名是没有经过dns解析的的所以要修改hosts文件指向kibana.kevin.co域名
修改完后，就可以使用域名加端口访问Kinaba了，这时候就需要输入配置好的用户密码登录才能正常访问，否则会报401错误

完。