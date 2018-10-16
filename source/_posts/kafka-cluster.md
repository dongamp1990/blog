---
title: 使用Docker快速搭建Kafka集群
tags: [docker,kafka]
date: 2018-06-26
---

## 前言
工作项目中使用到了MeessageQueue，原来是使用阿里云的MQ，是按使用量收费的，用了一段时间感觉数据量还是蛮大的，所以打算自己搭建消息队列服务器，先前测试过RabbitMQ，使用rabbitmq:3.7-management在docker环境下，消费者关掉后，容器CPU一直占用很高，不知什么情况，最后选择了Kafka，下面来讲下怎么快速搭建Kafka集群。

## 准备工作
两台服务器，
服务器A(192.168.10.1)：部署zookeeper集群、kafka
服务器B(192.168.10.2)：部署kafka

各服务器创建一个kafka网络
```
$ docker network create kafka
```

[下载docker-compose](https://github.com/docker/compose/releases/download/1.21.2/docker-compose-Linux-x86_64), 复制到/usr/bin/下
```
$ wget -O docker-compose https://github.com/docker/compose/releases/download/1.21.2/docker-compose-Linux-x86_64
$ mv ./docker-compose /usr/bin
$ chmod +x /usr/bin/docker-compose
$ docker-compose version
docker-compose version 1.21.2, build a133471
docker-py version: 3.3.0
CPython version: 3.6.5
OpenSSL version: OpenSSL 1.0.1t  3 May 2016

```

## 服务器A
该服务器部署zookeeper，和kafka，为了方便zookeeper和kafka通信，指定了同一个docker network，
同时kafka开启了JMX，方便使用kafka-manager做监控。

zookeeper镜像使用说明：
ZOO_MY_ID参数指定zooid，每个zooid不能重复
ZOO_SERVERS指定其他zookeeper服务器地址
更多说明请参阅https://hub.docker.com/_/zookeeper/

kafka镜像使用说明：
KAFKA_ADVERTISED_HOST_NAME 监听地址，指向服务器外网ip地址
KAFKA_ZOOKEEPER_CONNECT zookeeper服务器ip:port多个使用,分隔
KAFKA_JMX_OPTS jmx参数
KAFKA_BROKER_ID brokerid,不填写会自动生成
KAFKA_ADVERTISED_PORT 监听端口默认是9092，可以按需求修改
java.rmi.server.hostname 需要指向服务器的ip
更多说明请参阅https://hub.docker.com/r/wurstmeister/kafka/

<font color='red'>注意: zookeeper版本和kafka zookeeper的版本要一致，之前试过使用zookeeper 3.5启动kafka 1.1.0会出现kafka will not attempt to authenticate using sasl错误</font>
kafka-zookeeper.yaml文件
```
version: '3.1'

services:
  zoo1:
    image: zookeeper:3.4
    restart: always
    hostname: zoo1
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    networks:
      - kafka
    restart: always
  zoo2:
    image: zookeeper:3.4
    restart: always
    hostname: zoo2
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=0.0.0.0:2888:3888 server.3=zoo3:2888:3888
    networks:
      - kafka
    restart: always  
  zoo3:
    image: zookeeper:3.4
    restart: always
    hostname: zoo3
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=0.0.0.0:2888:3888
    networks:
      - kafka
    restart: always
  kafka:
    image: wurstmeister/kafka:1.1.0
    ports:
      - "9092:9092"
      - "1099:1099"
    environment:
      #指向服务器ip地址
      KAFKA_ADVERTISED_HOST_NAME: 192.168.10.1
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2182,zoo3:2183
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=192.168.10.1 -Dcom.sun.management.jmxremote.rmi.port=1099"
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_BROKER_ID: 1
      JMX_PORT: 1099
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/localtime:/etc/localtime
    networks:
    - kafka
    restart: always
networks:
  kafka:
    external: true
```
### 部署
```
$ docker-compose -f kafka-zookeeper.yaml up -d
Creating kafka_zoo2_1 ... done
Creating kafka_zoo2_2 ... done
Creating kafka_zoo2_3 ... done
Creating kafka_kafka_1 ... done
```

## 服务器B

kafka.yaml
```
version: '3.1'

services:
  kafka2:
    image: wurstmeister/kafka:1.1.0
    ports:
      - "9092:9092"
      - "1099:1099"
    environment:
      #指向服务器ip地址
      KAFKA_ADVERTISED_HOST_NAME: 192.168.10.2
      KAFKA_ZOOKEEPER_CONNECT: 192.168.10.2:2181,192.168.10.2:2182,192.168.10.2:2183
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=192.168.10.2 -Dcom.sun.management.jmxremote.rmi.port=1099"
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_BROKER_ID: 2
      JMX_PORT: 1099
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/localtime:/etc/localtime
    networks:
    - kafka
    restart: always
  kafka3:
    image: wurstmeister/kafka:1.1.0
    ports:
      - "9093:9092"
      - "2099:2099"
    environment:
      #指向服务器ip地址
      KAFKA_ADVERTISED_HOST_NAME: 192.168.10.2
      KAFKA_ZOOKEEPER_CONNECT: 192.168.10.2:2181,192.168.10.2:2182,192.168.10.2:2183
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=192.168.10.2 -Dcom.sun.management.jmxremote.rmi.port=2099"
      KAFKA_ADVERTISED_PORT: 9093
      KAFKA_BROKER_ID: 3
      JMX_PORT: 2099
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/localtime:/etc/localtime
    networks:
    - kafka
    restart: always    
networks:
  kafka:
    external: true
```

### 部署
```
$ docker-compose -f kafka.yaml up -d
Creating kafka_kafka3_1 ... done
Creating kafka_kafka2_1 ... done
```

## 部署kafka-manager
yahoo开源的kafka监控管理工具https://github.com/yahoo/kafka-manager

在服务器A上部署kafka-manager
```
docker run -d \
     --name kafka-manager \
     -p 9000:9000  \
     --network kafka \
     -e ZK_HOSTS="zoo1:2181,zoo2:2181,zoo3:2183" \
     hlebalbau/kafka-manager:latest \
     -Dpidfile.path=/dev/null
```

访问http://192.168.10.1:9000 就可以访问啦，怎么玩，稍微看看就知道了。

## 项目集成kafka

spring-boot版本:1.5.10.RELEASE
spring-kafka版本:1.3.5.RELEASE

https://docs.spring.io/spring-kafka/docs/1.3.5.RELEASE/reference/html/