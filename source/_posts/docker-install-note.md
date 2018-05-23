---
title: 安装Docker日志
tags: [docker]
date: 2018-01-11
---
安装Docker日志
## 安装Docker

### 系统要求
查看官方文档 https://docs.docker.com/engine/installation/
文内使用店是centos7


### 删除旧的版本
```
$ sudo yum remove docker \
 docker-common \
 docker-selinux \
 docker-engine
```

### 设置Docker存储库
安装必要的一些系统工具
```
$ sudo yum install -y yum-utils \
	device-mapper-persistent-data \
	lvm2
```

添加软件源信息
```
$ sudo yum-config-manager \
 --add-repo \
 https://download.docker.com/linux/centos/docker-ce.repo
```
 
### 安装docker-ce
安装docker稳定版
```
$ sudo yum install docker-ce-stable
```
显示可用的版本

```
$ yum list docker-ce --showduplicates | sort -r
docker-ce.x86_64   17.03.0.ce-1.el7.centos   docker-ce-stable
docker-ce.x86_64   17.03.1.ce-1.el7.centos   docker-ce-stable
docker-ce.x86_64   17.03.2.ce-1.el7.centos   docker-ce-stable
docker-ce.x86_64   17.06.0.ce-1.el7.centos   docker-ce-stable
docker-ce.x86_64   17.06.1.ce-1.el7.centos   docker-ce-stable
docker-ce.x86_64   17.06.2.ce-1.el7.centos   docker-ce-stable
docker-ce.x86_64   17.09.0.ce-1.el7.centos   docker-ce-stable
```

指定安装版本
```
$ yum install -y docker-ce-17.06.2.ce
```

### 增加阿里云docker镜像，加快镜像pull速度
```
#如果没有该文件可以创建一个
$ vi /etc/docker/daemon.json 
{
 "registry-mirrors": ["https://zln0jqua.mirror.aliyuncs.com"]
}
```

### 启动docker
```
$ sudo systemctl start docker
```

### 运行hello-word验证安装是否正确
```
$ sudo docker run hello-world
```

### 如果启动不了，可以查看日志文件
```
$ cat /var/log/upstart/docker.log
```

### 更多内容可查阅官方的安装文档
https://docs.docker.com/engine/installation/linux/docker-ce/centos/


## 安装问题汇总
安装后启动不了，查看详细日志
```
$ ail /var/log/upstart/docker.log
```

### 启动docker报no available network。
<font color=red>Error starting daemon: Error initializing network controller: list bridge addresses failed: no available network。</font><br>
手工添加bridge虚拟网络即可解决
```
$ sudo ip link add name docker0 type bridge
$ sudo ip addr add dev docker0 172.17.0.1/16
```

### 集群部署，woker节点报错
<font color=red>starting container failed: error creating external connectivity network: could not find an available, non-overlapping IPv4 address pool among the defaults to assign to the network</font><br>
可能是docker0的桥接网络网段冲突了。
```
查看网络ip
$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:16:3e:00:01:f0 brd ff:ff:ff:ff:ff:ff
    inet 10.170.122.155/21 brd 10.170.127.255 scope global eth0
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:16:3e:00:0e:c8 brd ff:ff:ff:ff:ff:ff
    inet 120.25.105.218/22 brd 120.25.107.255 scope global eth1
       valid_lft forever preferred_lft forever
57: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default 
    link/ether 66:f6:dc:19:a1:96 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.1/20 scope global docker0
       valid_lft forever preferred_lft forever


#查看路由表
$ ip route 
default via 120.25.107.247 dev eth1 
10.0.0.0/8 via 10.170.127.247 dev eth0 
10.170.120.0/21 dev eth0  proto kernel  scope link  src 10.170.122.155 
11.0.0.0/8 via 10.170.127.247 dev eth0 
100.64.0.0/10 via 10.170.127.247 dev eth0 
120.25.104.0/22 dev eth1  proto kernel  scope link  src 120.25.105.218 
172.16.0.0/12 via 10.170.127.247 dev eth0 
192.168.0.0/20 dev docker0  proto kernel  scope link  src 192.168.0.1 

#把路由表的192.168.0.0/20删掉，解决问题。

```

### 配置了内存限制，容器启动报task: non-zero exit (137)
查看日志/var/log/upstart/docker.log 发现<br>
<font color=red>Your kernel does not support swap limit capabilities,or the cgroup is not mounted. Memory limited without swap.</font><br>

```
#修改/etc/default/grub 增加
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
保存
#更新GRUB
$ sudo update-grub
#重启系统
$ reboot
```

## 构建docker镜像
构建一个带了jdk的ubuntu镜像

```
$ vi Dockerfile
#基础镜像名称
FROM ubuntu
#用来指定镜像创建者信息
MAINTAINER dong “dongamp1990@gmail.com”
#工作目录 
WORKDIR /opt/
#复制在Dockerfile目录下 复制jdk1.8.0_101.tar.gz 到 /opt目录下, 如果是docker可以识别的压缩包，目的目录不写/会自	动解压
ADD jdk1.8.0_101.tar.gz /opt
#执行命令
RUN mv /opt/jdk1.8.0_101 /opt/jdk8
#设置环境
ENV JAVA_HOME /opt/jdk8 
ENV PATH $PATH:$JAVA_HOME/bin
```

打包镜像
```
# docker build [OPTIONS] PATH | URL |
$ docker build -t ubuntu-jdk8 .
```

查看镜像
```
$ docker images
```

运行镜像
```
$ docker run -it ubuntu-jdk8 javac
```

## Docker使用

### 查看当前镜像
```
$ docker images 
```

### 拉取镜像
```
$ docker pull ubuntu #拉取latest tag的镜像
$ docker pull ubuntu:13.10 #拉取指定tag的镜像
```

### 运行容器
```
$ docker run -d -v /opt/eureka/logs/:/opt/logs/ --name eureka -p 8000:8761 springcloud/eureka
 -d 后台运行 
 -v 文件挂载 把容器内的/opt/logs映射到/opt/eureka/logs目录下
 --name 指定容器名称，如果不知道docker会随机生成应用名
-p 端口映射 8000是外部端口 8761是容器内的端口
```

### 查看当前运行的容器
```
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS  PORTS  NAMES
容器ID         镜像      命令        创建时间   状态    端口   容器名称	

docker ps -a 查看所有容器
```

### 停止容器
```
$ docker stop CONTAINERID
```

### 启动容器
```
$ docker start CONTAINERID
```

### 删除镜像 
```
$ docker rmi -f imagename[:TAG]
```

### 容器资源使用统计信息
官方文档地址：https://docs.docker.com/engine/reference/commandline/stats/
```
$ docker stats

CONTAINER           CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
221f321b1324        0.12%               467.3MiB / 3.858GiB   11.83%              1.08MB / 1.16MB     754kB / 0B          30
fac3348f831b        0.11%               322.7MiB / 512MiB     63.02%              467MB / 497MB       61.4kB / 0B         52
d960f8146f1d        1.29%               328.7MiB / 512MiB     64.19%              15.5GB / 18.3GB     20.5kB / 0B         66
f1441134f6d4        0.87%               367.3MiB / 512MiB     71.73%              5.31GB / 3.81GB     610kB / 0B          71
9007528c7621        0.00%               6.395MiB / 3.858GiB   0.16%               11.2MB / 51.8MB     5.41MB / 254kB      7


使用--format自定义显示数据内容

$ docker stats --format "table {{.Name}}\t{{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

NAME                                              CONTAINER           CPU %               MEM USAGE / LIMIT     NET I/O             BLOCK I/O
brave_kare                                        221f321b1324        0.11%               467.3MiB / 3.858GiB   1.08MB / 1.16MB     754kB / 0B
1_maxfunPayNewAdmin.1.639izogqf896v2jd4qmyl9c77   fac3348f831b        0.12%               322.7MiB / 512MiB     467MB / 497MB       61.4kB / 0B
1_maxfunGatewayZuul.1.0gn8x5yd229bjkj35ejjnyi7o   d960f8146f1d        1.41%               328.7MiB / 512MiB     15.5GB / 18.3GB     20.5kB / 0B
1_maxfunEureka.1.0q690ptcca0cxy7div3o1z92p        f1441134f6d4        0.92%               367.3MiB / 512MiB     5.31GB / 3.81GB     610kB / 0B
portainer                                         9007528c7621        0.00%               6.395MiB / 3.858GiB   11.2MB / 51.8MB     5.41MB / 254kB
```

## Docker私服Registry搭建

### 拉取registry v2镜像
```
$ docker pull registry:2
```

### 启动仓库
```
$ docker run -d -p 5000:5000 --restart=always --name registry \ 
  -v /opt/registry/:/var/lib/registry \ 
  -v /opt/registry/:/tmp/docker-registry.db registry:2

# -v 把registry容器内的/var/lib/registry目录，挂载到/opt/registry目录下，
  可以把镜像内容也挂载存储到磁盘上，防止registry停止后丢失数据

```

### 标记镜像
```
$ docker tag registry:2 192.168.42.132:5000/registry:v2
```

### 推送镜像到仓库
```
$ docker push 192.168.42.132:5000/registry:v2
```
如果出现<br>
<font color="red">Error response from daemon: Get https://192.168.42.132:5000/v2/users/: http: server gave HTTP response to HTTPS client</font><br>
在/etc/docker/daemon.json里面增加以下配置
```
"insecure-registries":["192.168.42.132:5000"]	
```

### 重启 docker
```
$ systemctl restart docker
```

### 重新推送tag到仓库
```
$ docker push 192.168.42.132:5000/registry:v2
```
### 查看仓库
```
$ curl http://192.168.42.132:5000/v2/_catalog 
# 如果没错，会返回如下内容。
{"repositories":["registry"]}
```