[TOC]



# Docker 安装与卸载



## 1. 官网

docker官网：http://www.docker.com

Docker Hub官网: https://hub.docker.com/



## 2. 安装

官方文档： https://docs.docker.com/engine/install/centos/



### 2.1 前置条件

要求系统为64位，Linux系统内核版本为3.8以上

查看自己的内核

```shell
# 看centos 版本
cat /etc/redhat-release

# 看内核版本
uname -r
```



### 2.2 开始安装

```shell
# 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2


# 配置yum源（⽐较慢,不⽤）
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 配置yum源 使⽤国内的
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 查看版本
yum list docker-ce --showduplicates | sort -r

# 1. 安装docker
yum -y install docker-ce-20.10.10-3.el7

# 2. 查看docker版本
docker -v
docker version  # 可以分别看到 docker 引擎的 Client版本和Server版本

# 3. 启动docker  运⾏Docker守护进程
systemctl start docker

# 4. 查看docker 启动状态
systemctl status docker

# 开机自启
systemctl enable docker.service

修改镜像仓库
vim /etc/docker/daemon.json
# 改为下⾯内容，然后重启docker
{
"debug":true,"experimental":true,
"registry-mirrors":
	["https://hub-mirror.c.163.com",
	"https://docker.mirrors.ustc.edu.cn"]
}

# 查看信息
docker info
```

```shell
# 启动使⽤Docker
systemctl stop docker #停⽌Docker守护进程
systemctl restart docker #重启Docker守护进程
```

### 2.3 配置命令自动补全

需要使用 bash-completion

```shell
# 检查有 /usr/share/bash-completion/bash_completion 这个文件
ls /usr/share/bash-completion/bash_completion

# 如果有/usr/share/bash-completion目录，
# 但是没有/usr/share/bash-completion/bash_completion文件（centos6为/etc/bash_completion文件），
# 则需要安装bash-completion
yum -y install bash-completion

# 检查/usr/share/bash-completion/completions文件夹下是否有docker开头的自动补全
# docker安装完后会在该文件夹下生成自动补全文件docker
# 如果安装了docker-compose，则该文件夹下还会有 docker-compose文件
ll /usr/share/bash-completion/completions/docker*

# 如果已经安装了docker自动补全，使用source命令使其生效 
source /usr/share/bash-completion/completions/docker

# 如果有报错，且报错中提示_get_comp_words_by_ref: command not found。说明bash-completion的配置文件没有生效，需要source一下 
# 对于centos7，bash-completion安装的是2.x版本，配置文件为/usr/share/bash-completion/bash_completion
source /usr/share/bash-completion/bash_completion

# 如果是centos6，自动安装的bash-completion最新版为1.x版本，配置文件为/etc/bash_completion
# bash /etc/bash_completion
```



## 3. 卸载docker

```shell
systemctl stop docker
yum remove docker-ce docker-ce-cli containerd.io
rm -rf /var/lib/docker
rm -rf /var/lib/containerd
```



### 4. Docker帮助命令

```shell
# 查看全部命令
docker help
# 查看单个命令的详细使用
docker 【命令】 --help
```



