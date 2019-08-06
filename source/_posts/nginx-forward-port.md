---
title: Nginx端口转发
tags: [Nginx]
date: 2019-07-25
---
## 前言
使用Nginx做端口转发，解决Elasticsearch、Kibana端口对外网暴露问题。

## 安装Nginx
nginx可以使用各平台的默认包来安装，本文是介绍使用源码编译安装，包括具体的编译参数信息。
正式开始前，编译环境gcc g++ 开发库之类的需要提前装好，这里默认你已经装好。
ububtu平台编译环境可以使用以下指令
```
apt-get install build-essential
apt-get install libtool
```
centos平台编译环境使用如下指令
安装make：
```
yum -y install gcc automake autoconf libtool make
```
安装g++:
```
yum install gcc gcc-c++
```

下面正式开始
一般我们都需要先装pcre, zlib，前者为了重写rewrite，后者为了gzip压缩。
### 安装pcre

```
cd /usr/local/src
wget https://ftp.pcre.org/pub/pcre/pcre-8.43.tar.gz
tar -zxf pcre-8.43.tar.gz
cd pcre-8.43
./configure
make
make install
```
### zlib库
```
cd /usr/local/src
wget http://zlib.net/zlib-1.2.11.tar.gz
tar -zxvf zlib-1.2.11.tar.gz
cd zlib-1.2.11
./configure
make
make install
```
### 安装ssl(某些vps默认没装ssl)
```
cd /usr/local/src
wget https://www.openssl.org/source/openssl-1.0.1t.tar.gz
tar -zxvf openssl-1.0.1t.tar.gz
```
### 安装nginx
Nginx 一般有两个版本，分别是稳定版和开发版，您可以根据您的目的来选择这两个版本的其中一个，下面是把 Nginx 安装到 /usr/local/nginx 目录下的详细步骤：
```
cd /usr/local/src
wget http://nginx.org/download/nginx-1.14.2.tar.gz
tar -zxf nginx-1.14.2.tar.gz
cd nginx-1.14.2
./configure --sbin-path=/usr/local/nginx/nginx \
--conf-path=/usr/local/nginx/nginx.conf \
--pid-path=/usr/local/nginx/nginx.pid \
--with-http_ssl_module \
--with-pcre=/usr/local/src/pcre-8.43 \
--with-zlib=/usr/local/src/zlib-1.2.11 \
--with-openssl=/usr/local/src/openssl-1.0.1t
 
make
make install
```
--with-pcre=/usr/src/pcre-8.43 指的是pcre-8.43 的源码路径。
--with-zlib=/usr/src/zlib-1.2.11 指的是zlib-1.2.11 的源码路径。
--with-openssl=/usr/local/src/openssl-1.0.1t 指的是openssl-1.0.1t 的源码路径。
安装成功后，路径是在/usr/local/nginx

### 启动
确保系统的 80 端口没被其他程序占用，运行/usr/local/nginx/nginx 命令来启动 Nginx
```
netstat -ano|grep 80
```
如果查不到结果后执行，有结果则忽略此步骤（ubuntu下必须用sudo启动，不然只能在前台运行）
```
sudo /usr/local/nginx/nginx
```
打开浏览器访问此机器的 IP，如果浏览器出现 Welcome to nginx! 则表示 Nginx 已经安装并运行成功。
到这里nginx就安装完成了。

## 端口转发
### 转发Elasticsearch端口
监听11880端口，转发到es集群机器的9200端口，监听5601端口，转发到Kibana机器的5601端口
编辑nginx.conf
```
vi /usr/local/nginx/nginx.conf
```
在http {}里面增加配置内容
```
	#配置负载均衡池
    upstream es_pool {
       server 172.18.131.1:9200;
       server 172.18.132.2:9200;
       server 172.18.133.3:9200;
    }

    server {
        listen       11880;
        
		#将所有请求转发给es_pool池的应用处理
        location / {
            #root   html;
            #index  index.html index.htm;
            proxy_pass http://es_pool;
        }
    }
	
    server {
        listen       5601;
        auth_basic   "Authorization";
        auth_basic_user_file /usr/local/nginx/pass_file;

        autoindex on;
        autoindex_exact_size on;
        autoindex_localtime on;
        location / {
            #root   html;
            #index  index.html index.htm;
            proxy_pass http://172.18.132.103:5601;
        }
    }
```

### 增加Basic Authorization认证
创建授权用户和密码文件
使用openssl生成创建密码
```
printf "ttlsa:$(openssl passwd -crypt 123456)\n" >> /usr/local/nginx/pass_file
cat /usr/local/nginx/pass_file
ttlsa:XVrmhIOW5ujrw
```

在server {}内增加以下内容
```
        auth_basic   "Authorization";
        auth_basic_user_file /usr/local/nginx/pass_file;

        autoindex on;
        autoindex_exact_size on;
        autoindex_localtime on;
```

完整的nginx.conf
```
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
    server {
        listen       80;
        location / {
            root   html;
            index  index.html index.htm;
        }
    }

    upstream es_pool {
       server 172.18.132.101:9200;
       server 172.18.132.103:9200;
       server 172.18.132.103:9200;
    }

    server {
        listen       11880;
        auth_basic   "Authorization";
        auth_basic_user_file /usr/local/nginx/pass_file;

        autoindex on;
        autoindex_exact_size on;
        autoindex_localtime on;
        location / {
            #root   html;
            #index  index.html index.htm;
            proxy_pass http://es_pool;
        }
    }
	
	server {
        listen       5601;
        auth_basic   "Authorization";
        auth_basic_user_file /usr/local/nginx/pass_file;

        autoindex on;
        autoindex_exact_size on;
        autoindex_localtime on;
        location / {
            #root   html;
            #index  index.html index.htm;
            proxy_pass http://172.18.132.103:5601;
        }
    }
}

```
重新加载nginx配置文件
```
/usr/local/nginx/nginx -s reload
```
访问http://IP:5601 输入用户密码即可以调整到目的主机的5601端口了，其他同理。

资料参考：http://www.nginx.cn/doc/