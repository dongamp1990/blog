---
title: Dockerfile语法结构
tags: [docker]
---
Dockerfile语法结构
## Dockerfile 指令

Dockerfile 有以下指令选项:
### FROM  
FROM指定构建镜像的基础源镜像，如果本地没有指定的镜像，则会自动从 Docker 的	公共库 pull 镜像下来。
```
FROM <image>:<tag> 
```

### MAINTAINER
指定创建镜像的用户
```
MAINTAINER <name>
```

### RUN
有两种使用方式
```
RUN "executable", "param1", "param2"
```
每条RUN指令将在当前镜像基础上执行指定命令，并提交为新的镜像，后续的RUN都	在之前RUN提交后的镜像为基础，镜像是分层的，可以通过一个镜像的任何一个历史	提交点来创建，类似源码的版本控制。

exec 方式会被解析为一个 JSON 数组，所以必须使用双引号而不是单引号。exec 方式	不会调用一个命令 shell，所以也就不会继承相应的变量，如：
```
RUN [ "echo", "$HOME" ]
```

这种方式是不会达到输出 HOME 变量的，正确的方式应该是这样的
```
RUN [ "sh", "-c", "echo", "$HOME" ]
```

### CMD
CMD有三种使用方式:
```
CMD "executable","param1","param2"
CMD "param1","param2"
CMD command param1 param2 (shell form)
```

### EXPOSE
告诉 Docker 服务端容器对外映射的本地端口，需要在 docker run 的时候使用-p或	-P选项生效。
```
EXPOSE <port> [<port>...]  
```

### ENV
指定一个环节变量，会被后续RUN指令使用，并在容器运行时保留。
```
ENV <key> <value>       # 只能设置一个变量
```

### ADD
```
ADD <src> <dest>
ADD tomcat7.sh /etc/init.d/tomcat7
```
所有拷贝到container中的文件和文件夹权限为0755，uid和gid为0；如果是一个目录，	那么会将该目录下的所有文件添加到container中，不包括目录；如果文件是可识别的	压缩格式，则docker会帮忙解压缩（注意压缩格式）；如果<src>是文件且<dest>中不使	用斜杠结束，则会将<dest>视为文件，<src>的内容会写入<dest>；如果<src>是文件且	<dest>中使用斜杠结束，则会<src>文件拷贝到<dest>目录下。

ADD复制本地主机文件、目录或者远程文件 URLS 从 并且添加到容器指定路径中 。
支持通过 GO 的正则模糊匹配，具体规则可参见 Go filepath.Match
```
ADD hom* /mydir/        # adds all files starting with "hom"
ADD hom?.txt /mydir/    # ? is replaced with any single character
```
路径必须是绝对路径，如果 不存在，会自动创建对应目录
路径必须是 Dockerfile 所在路径的相对路径
如果是一个目录，只会复制目录下的内容，而目录本身则不会被复制

### COPY
```
COPY <src> <dest>
COPY tomcat7.sh /etc/init.d/tomcat7
```
COPY复制新文件或者目录从 并且添加到容器指定路径中 。用法同ADD，唯一的不同	是不能指定远程文件 URLS。

### ENTRYPOINT
设置指令，指定容器启动时执行的命令，可以多次设置，但是只有最后一个有效。
两种格式:
```
ENTRYPOINT ["executable", "param1", "param2"] #(like an exec, the preferred form)  
ENTRYPOINT command param1 param2 #(as a shell) 
```
该指令的使用分为两种情况，一种是独自使用，另一种和CMD指令配合使用。
当独自使用时，如果你还使用了CMD命令且CMD是一个完整的可执行的命令，那么	CMD指令和ENTRYPOINT会互相覆盖只有最后一个CMD或者ENTRYPOINT有效。

#### CMD指令将不会被执行，只有ENTRYPOINT指令被执行  
```
CMD echo “Hello, World!”  
ENTRYPOINT ls -l  
```
另一种用法和CMD指令配合使用来指定ENTRYPOINT的默认参数，这时CMD指令不是	一个完整的可执行命令，仅仅是参数部分；ENTRYPOINT指令只能使用JSON方式指定执	行命令，而不能指定参数。
```
FROM ubuntu  
CMD ["-l"]  
ENTRYPOINT ["/usr/bin/ls"]
```

### VOLUME
使容器中的一个目录具有持久化存储数据的功能，该目录可以被容器本身使用，也可以	共享给其他容器使用。我们知道容器使用的是AUFS，这种文件系统不能持久化数据，	当容器关闭后，所有的更改都会丢失。当容器中的应用有持久化数据的需求时可以在	Dockerfile中使用该指令。
```
VOLUME ["/tmp/data"]
```
运行通过该Dockerfile生成image的容器，/tmp/data目录中的数据在容器关闭后，里面	的数据还存在。例如另一个容器也有持久化数据的需求，且想使用上面容器共享的	/tmp/data目录，那么可以运行下面的命令启动一个容器：
```
docker run -t -i -rm -volumes-from container1 image2 bash 
```
container1为第一个容器的ID，image2为第二个容器运行image的名字。

### USER 
容器运行用户
```
USER root
```

### WORKDIR
可以多次切换(相当于cd命令)，对RUN,CMD,ENTRYPOINT生效。
```
WORKDIR /path/to/workdir  
```

## Dockerfile例子

```
# Pull base image  
FROM ubuntu:13.10  
  
MAINTAINER xx "email"  
  
# all command in one
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe"> /etc/apt/sources.list && \ 
   apt-get update && \
   apt-get -y install curl && \
   cd /tmp &&  curl -L 'http://download.oracle.com/otn-pub/java/jdk/7u65-b17/jdk-7u65-linux-x64.tar.gz' -H 'Cookie: oraclelicense=accept-securebackup-cookie; gpw_e24=Dockerfile' | tar -xz  && \
   mkdir -p /usr/lib/jvm && \
   mv /tmp/jdk1.7.0_65/ /usr/lib/jvm/java-7-oracle/  && \
   pdate-alternatives --install /usr/bin/java java /usr/lib/jvm/java-7-oracle/bin/java 300 && \
   update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/java-7-oracle/bin/javac 300 && \
   cd /tmp && curl -L 'http://archive.apache.org/dist/tomcat/tomcat-7/v7.0.8/bin/apache-tomcat-7.0.8.tar.gz' | tar -xz && \
   mv /tmp/apache-tomcat-7.0.8/ /opt/tomcat7/ && \
   chmod 755 /etc/init.d/tomcat7 
 
ENV JAVA_HOME /usr/lib/jvm/java-7-oracle/  
ENV CATALINA_HOME /opt/tomcat7  
ENV PATH $PATH:$CATALINA_HOME/bin  

ADD tomcat7.sh /etc/init.d/tomcat7  

# Expose ports.  
EXPOSE 8080  
  
# Define default command.  
ENTRYPOINT service tomcat7 start && tail -f /opt/tomcat7/logs/catalina.out  
```