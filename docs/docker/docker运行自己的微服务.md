



# Docker运行自己的微服务





## 1. 打包成jar包，并上传至linux服务器

略



## 2. 编写Dockerfile

```shell
# 基础镜像使用java
FROM java:8
# 作者
MAINTAINER lishem
# VOLUME 指定临时文件目录为/tmp，在主机/var/lib/docker目录下创建了一个临时文件并链接到容器的/tmp
VOLUME /tmp
# 将jar包添加到容器中并更名为xxxxx.jar
ADD docker_boot-0.0.1-SNAPSHOT.jar zzyy_docker.jar
# 运行jar包
RUN bash -c 'touch /xxxxx.jar'
ENTRYPOINT ["java","-jar","/xxxxx.jar"]
# 暴露6001端口作为微服务
EXPOSE 6001
```

## 3.  构建镜像

```shell
docker build -t 仓库名:版本名 .
```

## 4. 运行

```shell
docker run 仓库名:版本名
```

