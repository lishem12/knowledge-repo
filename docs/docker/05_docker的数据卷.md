[TOC]



# Docker的数据卷





## 1. 容器卷概述

卷就是目录或文件，存在于一个或多个容器中，由docker挂载到容器，但不属于联合文件系统，因此能够绕过Union File System提供一些用于持续存储或共享数据的特性：

卷的设计目的就是数据的持久化，完全独立于容器的生存周期，因此Docker不会在容器删除时删除其挂载的数据卷。



特点：

1. 数据卷可在容器之间共享或重用数据
2. 卷中的更改可以直接实时生效
3. 数据卷中的更改不会包含在镜像的更新中
4. 数据卷的生命周期一直持续到没有容器使用它为止



## 2. 带容器卷的容器操作

### 2.1 运行带容器卷的容器

```shell
docker run --privileged=true -v /宿主机绝对路径目录:/容器内目录  镜像名

# 如,宿主机如果目录不存在，会自动创建
docker run -it --privileged=true -v /tmp/host_data:/docker_data  ubuntu
```



注意：如果 Docker挂载主机目录访问如果出现`cannot open directory .: Permission denied`时，可以尝试在命令中加入`--privileged=true`

解释：CentOS7安全模块会比之前系统版本加强，不安全的会先禁止，所以目录挂载的情况被默认为不安全的行为，在SELinux里面挂载目录被禁止掉了，如果要开启，我们一般使用--privileged=true命令，扩大容器的权限解决挂载目录没有权限的问题，也即：

- **使用该参数，container内的root拥有真正的root权限，否则，container内的root只是外部的一个普通用户权限。**



### 2.2 查看已挂在容器卷的容器

```shell
docker inspect 容器id

# 在显示的JSON里，有一个 Mounts：
"Mounts": [
    {
        "Type": "bind",
        "Source": "/tmp/host_data",
        "Destination": "/docker_data",
        "Mode": "",
        "RW": true,
        "Propagation": "rprivate"
    }
]
```



## 3. 容器卷权限ro与rw

默认情况挂载的容器卷，对宿主机和容器卷来说，挂载的目录是 `rw`的，也就是宿主机和容器卷对挂载目录都具有读写权限

默认挂载相当于：

```shell
docker run --privileged=true -v /宿主机绝对路径目录:/容器内目录:rw  镜像名
```

如果需要容器对数据卷只有读权限，主机对容器内具有读写权限，则需要添加`ro`参数，即`read only`

```shell
docker run --privileged=true -v /宿主机绝对路径目录:/容器内目录:ro  镜像名
```

用这种方式可以将宿主机的内容同步到容器内，以免容器内修改



## 4. 容器卷的继承

```shell
docker run -it  --privileged=true -v /mydocker/u:/tmp --name u1 ubuntu

docker run -it  --privileged=true --volumes-from u1  --name u2 ubuntu
```

此时 u2 容器会继承 u1 容器的容器卷配置，上面的案例中， u1  u2 host 同时同步 `/tmp ` 下的内容

