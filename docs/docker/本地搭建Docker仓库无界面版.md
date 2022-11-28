

# 本地搭建Docker仓库无界面版







## 首先要安装docker



## 安装 docker Registry

```shell
docker pull registry

docker run -d -p 5000:5000 -v /root/docker_registry:/tmp/registry --privileged=true registry
```



## 验证是否安装成功



通过以下请求，可以获取 registry中有哪些镜像

```shell
curl -XGET http://192.168.174.200:5000/v2/_catalog
```

![image-20221104205958188](本地搭建Docker仓库无界面版/image-20221104205958188.png)



## 推送自己的镜像进入私服

### 在推送的机器上配置

docker默认不允许http推送镜像，可以通过配置取消这个限制

```shell
vim /etc/docker/daemon.json


{
  "registry-mirrors": ["https://xxxxxxxx.mirror.aliyuncs.com"],
  "insecure-registries":["192.168.174.200:5000"]
}
```

### 推送

```shell
docker tag 镜像ID host:port/仓库名：tag
docker push host:port/仓库名:tag

# 如
docker tag 95e877c38a2c 192.168.174.200:5000/ubuntu:8.0.0
docker push 192.168.174.200:5000/ubuntu:8.0.0
```



## 从私服库上拉取

```shell
docker pull 192.168.174.200:5000/ubuntu:8.0.0
```

