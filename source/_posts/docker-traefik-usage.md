---
title: Docker Swarm集群使用Traefik
tags: [docker]
date: 2018-04-02
---

## 前言
Træfɪk 是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。 它支持多种后台 (Docker, Swarm, Kubernetes, Marathon, Mesos, Consul, Etcd, Zookeeper, BoltDB, Rest API, file…) 来自动化、动态的应用它的配置文件设置。
官方文档里面介绍了很多场景的使用，详情点击<a href='https://docs.traefik.cn/'>官方文档</a>

## 部署Traefik
### swarm网络

swarm集群里面使用，需要使用overlay网络, 需先创建一个overlay网络，或者使用已有的overlay网络
<font color='red'>（不要使用名字为ingress的那个网络，一开始我使用了，怎么弄都不成功，踩过得坑。谨记）</font>

```
$ docker network create traefik-net --driver overlay

```
查看网络
```
$ docker network ls

p86srgdn2zne        traefik-net         overlay             swarm
```

traefik.yaml
```
version: "3"
services:
  traefik:
    image: traefik
    command: --docker --docker.swarmmode  --docker.domain=maxfun.co --docker.watch  --web --api --logLevel=DEBUG
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
    deploy:
      placement:
        constraints:
        - node.role == manager 
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - $PWD/traefik.toml:/etc/traefik/traefik.toml
    networks:
    - traefik-net
networks:
  traefik-net:
    external: true
```
treafik配置文件:https://raw.githubusercontent.com/containous/traefik/master/traefik.sample.toml
启动traefik服务
```
$ docker stack deploy -c traefik.yaml traefik
```
或者

```
docker service create \
    --name traefik \
    --constraint=node.role==manager \
    --publish 80:80 \
    --publish 8080:8080 \
    --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
    --mount type=bind,source=/opt/traefik/traefik.toml,target=/etc/traefik/traefik.toml \
    --network traefik-net \
    traefik \
    --docker \
    --docker.swarmmode \
    --docker.domain=maxfun.co \
    --docker.watch \
    --web \
    --api
```


###  部署jenkins应用
在master节点部署jenkins，不使用用端口来访问jenkins

run-jenkins.sh
```
mkdir -p /opt/jenkins-blueocean-data
MVN_REPO_PATH=/home/coder/developtools/mavenRepository
docker service create \
  --name jenkins \
  --constraint=node.role==manager \
  -u root \
  -e JAVA_OPTS=-Duser.timezone=GMT+08 \
  -e JENKINS_OPTS=--httpPort=8080 \
  --publish 8089:8080 \
  --mount type=bind,source=/opt/jenkins-blueocean-data,target=/var/jenkins_home \
  --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
  --mount type=bind,source=$MVN_REPO_PATH,target=/root/.m2/repository \
  --mount type=bind,source=/opt/jenkins-keys,target=/root/.ssh \
  --hostname jenkins \
  --network traefik-net \
  --label traefik.port=8080 \
  --label traefik.docker.network=traefik-net \
  --label traefik.frontend.rule=Host:jenkins.maxfun.co \
  --label traefik.backend=jenkins-backend
 jenkinsci/blueocean:1.3.5
```

重点参数是下面几个
```
  --network traefik-net \
  --label traefik.port=8080 \
  --label traefik.docker.network=traefik-net \
  --label traefik.frontend.rule=Host:jenkins.maxfun.co \
  
  --network 
   指定已创建好的overlay网络
  --label traefik.port=8080
   注册使用这个端口。当容器暴露出多个端口时非常有效。
  --label traefik.docker.network=traefik-net 
   设置连接到这个容器的docker网络。 如果容易被链接到多个网络，
   一定要设置合适的网络名称（你可以使用docker检查<container_id>）否则它将自动选择一个（取决于docker如何返回它们）。
  --label traefik.frontend.rule=Host:jenkins.maxfun.co 
   覆盖默认前端规则（默认：Host:{containerName}.{domain}）
  --label traefik.backend=jenkins-backend
   将容器指向 enkins-backend 后端
  --traefik.enable=false
   可以使用docker service update servicename --label-add traefik.enable=false 调整该service不可用，从而达到把应用从负载均衡列表剔除
  JENKINS_OPTS=--httpPort=8080
   jenkins参数 --httpPort自定义jenkins端口 
```

更多traefik容器覆盖默认表现方式的Label：https://docs.traefik.cn/toml#docker-backend

启动jenkins服务
```
$ sh run-jenkins.sh
```

测试
```
$ curl -H Host:jenkins.maxfun.co http://127.0.0.1
```
会返回以下内容，证明成功了。
```
<html><head><meta http-equiv='refresh' content='1;url=/login?from=%2F'/>
<script>window.location.replace('/login?from=%2F');</script></head>
<body style='background-color:white; color:white;'>
Authentication required
<!--
You are authenticated as: anonymous
Groups that you are in:
  
Permission you need to have (but didn't): hudson.model.Hudson.Read
 ... which is implied by: hudson.security.Permission.GenericRead
 ... which is implied by: hudson.model.Hudson.Administer
-->
</body></html>
```
就可以愉快的使用域名来访问jenkins啦。

### 官方文档的例子

whoami.yaml
```
version: "3"
services:
  web:
    image: emilevauge/whoami
    networks: 
      - traefik-net
    deploy:
      labels:
        - "traefik.backend=whoami-backend"
        - "traefik.port=80"
        - "traefik.docker.network=traefik-net"
        - "traefik.frontend.rule=Host:whoamistack.traefik"
networks:
  traefik-net:
    external: true
```
<font color='red'>注意： docker-compose文件 labels必须在deploy里面，结构跟上面所示</font>


#### 启动第一个服务
```
$ docker stack deploy -c whoami.yaml whoami1
```

测试
```
$ curl -H Host:whoamistack.traefik http://127.0.0.1
```

```
Hostname: 9e402666ec66
IP: 127.0.0.1
IP: 10.0.1.9
IP: 10.0.1.8
IP: 192.168.16.10
GET / HTTP/1.1
Host: whoamistack.traefik
User-Agent: curl/7.35.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.255.0.2
X-Forwarded-Host: whoamistack.traefik
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: 361dbc4b6660
X-Real-Ip: 10.255.0.2
```
#### 启动第二个服务
```
$ docker stack deploy -c whoami.yaml whoami12
```

多次执行
```
$ curl -H Host:whoamistack.traefik http://127.0.0.1
```
Hostname: 来回变化，这表明负载均衡功能正常工作了


资料参考：https://docs.traefik.cn/toml#docker-backend