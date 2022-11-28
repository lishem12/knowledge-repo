

[TOC]





# Dockerfile



官网： https://docs.docker.com/engine/reference/builder/



## 1. Dockerfile概述

Dockerfile是用来构建Docker镜像的文本文件，是由一条条构建镜像所需的指令和参数构成的脚本。

构件三步骤：

- 编写Dockerfile文件
- `docker build`命令构建镜像
- `docker run` 依镜像运行容器实例

### Dockerfile构件过程解析

1. 每条**保留字**指令都必须为**大写字母**且后面要跟随至少一个参数
2. 指令按照从上到下，顺序执行
3. `#`表示注释
4. 每条指令都会创建一个新的镜像层并对镜像进行提交

### Docker 执行Dockerfile的流程

1. docker从基础镜像运行一个容器
2. 执行一条指令并对容器作出修改
3. 执行类似docker commit的操作提交一个新的镜像层
4. docker再基于刚提交的镜像运行一个新容器
5. 执行dockerfile中的下一条指令直到所有指令都执行完成



## 2. Dockerfile 保留字

```dockerfile
FROM # 基础镜像，当前新镜像是基于哪个镜像的，指定一个已经存在的镜像作为模板，第一条必须是FROM
MAINTAINER  # 镜像维护者的姓名和邮箱地址
RUN # 容器构建时需要运行的命令
# 有 shell和 exec 两种格式 RUN是docker build 时运行的
# run <命令行命令>  等同于在终端操作的shell命令 如  RUN yum -y install vim
# RUN ["可执行文件","参数1","参数2"]  
#  如 RUN ["./test.php","dev","offline"]  等价于 RUN ./test.php dev offline


EXPOSE # 当前容器对外暴露出的端口
WORKDIR # 指定在创建容器后，终端默认登陆的进来工作目录，一个落脚点
USER  # 指定该镜像以什么样的用户去执行，如果都不指定，默认是root
ENV  # 用来在构建镜像过程中设置环境变量
# 如： ENV MY_PATH /usr/mytest
# 这个环境变量可以在后续的任何RUN指令中使用，这就如同在命令前面指定了环境变量前缀一样；也可以在其它指令中直接使用这些环境变量，比如：WORKDIR $MY_PATH

VOLUME # 容器数据卷，用于数据保存和持久化工作

ADD # 将宿主机目录下的文件拷贝进镜像且会自动处理URL和解压tar压缩包
COPY # 类似ADD，拷贝文件和目录到镜像中。
# 将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置
# 如 COPY src dest
# COPY ["src", "dest"]
# <src源路径>：源文件或者源目录
# <dest目标路径>：容器内的指定路径，该路径不用事先建好，路径不存在的话，会自动创建。


CMD  # 指定容器启动后的要干的事情
# 需要区分 RUN 和 CMD ， RUN 运行在 镜像构建过程，CMD运行在启动容器
# 其使用格式和 RUN 相似，也是支持空格 分割和 [] 写参数
# 一个Dockerfile中可以有多个CMD指令，但只有最后一个生效，
# CMD的参数会被docker run 之后的参数替换
# CMD的参数会被docker run 之后的参数替换
# CMD的参数会被docker run 之后的参数替换


ENTRYPOINT
# 也是用来指定一个容器启动时要运行的命令
# 类似于 CMD 指令，但是ENTRYPOINT不会被docker run后面的命令覆盖， 而且这些命令行参数会被当作参数送给 ENTRYPOINT 指令指定的程序

# ENTRYPOINT可以和CMD一起用，一般是变参才会使用CMD, 这里的CMD等于是在给ENTRYPOINT传参。
# 当指定了ENTRYPOINT后，CMD的含义就发生了变化，不再是直接运行其命令而是将CMD的内容作为参数传递给ENTRYPOINT指令，他两个组合会变成 <ENTRYPOINT> "<CMD>" 
```

ENTRYPOINT 使用案例

编写Dockerfile构建`nginx:test` 镜像

```dockerfile
FROM nginx

ENTRYPOINT ["nginx", " -c"]  # 定参
CMD ["/etc/nginx/nginx.conf"] # 变参


## 执行这个Dockerfile构建的镜像
# 如果运行
docker run nginx:test      
# 实际相当于运行
nginx -c /etc/nginx/nginx.conf

# 如果运行
docker run nginx:test -c /etc/nginx/new.conf
# 实际相当于运行
nginx -c /etc/nginx/new.conf
```



## 3. 案例

### 3.1  在Centos7 镜像上配置vim/ifconfig/jdk8

创建 Dockerfile文件

```shell
cd /root/dockerfile
touch Dockerfile
```

编写Dockerfile文件

```dockerfile
FROM centos:centos7.9.2009
MAINTAINER lishem<xxx.qq.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH

#安装vim编辑器
RUN yum -y install vim
# 安装ifconfig命令查看网络IP
RUN yum -y install net-tools
# 安装java8及lib库
RUN yum -y install glibc.i686
RUN mkdir /usr/local/java
# ADD 是相对路径jar,把jdk-8u212-linux-x64.tar.gz添加到容器中,安装包必须要和Dockerfile文件在同一位置
ADD jdk-8u212-linux-x64.tar.gz /usr/local/java/
# 配置java环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_212
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
ENV PATH $JAVA_HOME/bin:$PATH

EXPOSE 80

CMD echo $MYPATH
CMD echo "success--------------ok"
CMD /bin/bash
```

构建镜像

```shell
# docker build -t 新镜像名字:TAG . 注意后个点，必须进入Dockerfile同级目录执行这个
docker build -t my_centos7:1.0.1 .
```

### 3.2 在ubuntu 镜像上安装网络工具

```dockerfile
FROM ubuntu
MAINTAINER lishem<xxx.qq.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH

RUN apt-get update
RUN apt-get install net-tools
#RUN apt-get install -y iproute2
#RUN apt-get install -y inetutils-ping

EXPOSE 80

CMD echo $MYPATH
CMD echo "install inconfig cmd into ubuntu success--------------ok"
CMD /bin/bash
```









