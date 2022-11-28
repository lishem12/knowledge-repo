[TOC]





# Redis哨兵



## 1. 哨兵原理

### 1.1 集群结构和作用

哨兵的结构如图：

![image-20210725154528072](30_redis哨兵/image-20210725154528072.png)

哨兵的作用如下：

- **监控**：Sentinel 会不断检查master和slave是否按预期工作
- **自动故障恢复**：如果master故障，Sentinel会将一个slave提升为master。当故障实例恢复后也以新的master为主
- **通知**：Sentinel充当Redis客户端的服务发现来源，当集群发生故障转移时，会将最新信息推送给Redis的客户端



### 1.2 集群监控原理

Sentinel基于心跳机制监测服务状态，每隔1秒向集群的每个实例发送ping命令：

- 主观下线：如果某sentinel节点发现某实例未在规定时间响应，则认为该实例**主观下线**。

- 客观下线：若超过指定数量（quorum）的sentinel都认为该实例主观下线，则该实例**客观下线**。quorum值最好超过Sentinel实例数量的一半。

![image-20210725154632354](30_redis哨兵/image-20210725154632354.png)

### 1.3 集群故障恢复原理

一旦发现master故障，sentinel需要在salve中选择一个作为新的master，选择依据是这样的：

- 首先会判断slave节点与master节点断开时间长短，如果超过指定值（down-after-milliseconds * 10）则会排除该slave节点
- 然后判断slave节点的slave-priority值，越小优先级越高，如果是0则永不参与选举
- 如果slave-prority一样，则判断slave节点的offset值，越大说明数据越新，优先级越高
- 最后是判断slave节点的运行id大小，越小优先级越高。



当选出一个新的master后，该如何实现切换呢？

流程如下：

- sentinel给备选的slave1节点发送slaveof no one命令，让该节点成为master
- sentinel给所有其它slave发送slaveof 192.168.150.101 7002 命令，让这些slave成为新master的从节点，开始从新的master上同步数据。
- 最后，sentinel将故障节点标记为slave，当故障节点恢复后会自动成为新的master的slave节点

![image-20210725154816841](30_redis哨兵/image-20210725154816841.png)

### 1.4 小结

Sentinel的三个作用是什么？

- 监控
- 故障转移
- 通知

Sentinel如何判断一个redis实例是否健康？

- 每隔1秒发送一次ping命令，如果超过一定时间没有相向则认为是主观下线
- 如果大多数sentinel都认为实例主观下线，则判定服务下线

故障转移步骤有哪些？

- 首先选定一个slave作为新的master，执行`slaveof no one`
- 然后让所有节点都执行slaveof 新master
- 修改故障节点配置，添加slaveof 新master



## 2. 搭建哨兵集群

### 2.1 集群结构

这里我们搭建一个三节点形成的Sentinel集群，来监管之前的Redis主从集群。如图：

![image-20210701215227018](30_redis哨兵/image-20210701215227018.png)

三个sentinel实例信息如下：

| 节点 |       IP        | PORT  |
| :--- | :-------------: | :---: |
| s1   | 192.168.174.201 | 27001 |
| s2   | 192.168.174.201 | 27002 |
| s3   | 192.168.174.201 | 27003 |



### 2.2 准备实例和配置

要在同一台虚拟机开启3个实例，必须准备三份不同的配置文件和目录，配置文件所在目录也就是工作目录。

我们创建三个文件夹，名字分别叫s1、s2、s3：

```shell
# 进入/tmp目录
cd /tmp
# 创建目录
mkdir s1 s2 s3
```

在s1目录创建一个`sentinel.conf`文件，添加下面的内容：

```shell
port 27001
sentinel announce-ip 192.168.174.201
sentinel monitor mymaster 192.168.174.201 7001 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
dir "/tmp/s1"
```

解读：

- `port 27001`：是当前sentinel实例的端口
- `sentinel monitor mymaster 192.168.174.201 7001 2`：指定主节点信息
  - `mymaster`：主节点名称，自定义，任意写
  - `192.168.150.101 7001`：主节点的ip和端口
  - `2`：选举master时的quorum值

然后将`s1/sentinel.conf`文件拷贝到s2、s3两个目录中（在/tmp目录执行下列命令）：

```sh
# 方式一：逐个拷贝
cp s1/sentinel.conf s2
cp s1/sentinel.conf s3
# 方式二：管道组合命令，一键拷贝 记得回到 /tmp 目录下
echo s2 s3 | xargs -t -n 1 cp s1/sentinel.conf
```

修改s2、s3两个文件夹内的配置文件，将端口分别修改为27002、27003：

```shell
sed -i -e 's/27001/27002/g' -e 's/s1/s2/g' s2/sentinel.conf
sed -i -e 's/27001/27003/g' -e 's/s1/s3/g' s3/sentinel.conf
```



### 2.3 启动

启动命令：

```shell
# 第1个
redis-sentinel s1/sentinel.conf
# 第2个
redis-sentinel s2/sentinel.conf
# 第3个
redis-sentinel s3/sentinel.conf
```



### 2.4 测试

尝试让master节点7001宕机，查看sentinel日志：

![image-20210701222857997](30_redis哨兵/image-20210701222857997.png)

查看7003的日志：

![image-20210701223025709](30_redis哨兵/image-20210701223025709.png)

查看7002的日志：

![image-20210701223131264](30_redis哨兵/image-20210701223131264.png)



## 3. RedisTemplate 连接哨兵

在`Sentinel`集群监管下的Redis主从集群，其节点会因为自动故障转移而发生变化，Redis的客户端必须感知这种变化，及时更新连接信息。Spring的RedisTemplate底层利用lettuce实现了节点的感知和自动切换。

### 3.1 引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```



### 3.2 配置Redis地址

配置的是`sentinel`哨兵集群地址

```yaml
spring:
  redis:
    sentinel:
      master: mymaster
      nodes:
        - 192.168.174.201:27001
        - 192.168.174.201:27002
        - 192.168.174.201:27003
```



### 3.3 配置读写分离

在项目的启动类中，添加一个新的bean：

```java
@SpringBootApplication
public class RedisDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(RedisDemoApplication.class, args);
    }

    @Bean
    public LettuceClientConfigurationBuilderCustomizer clientConfigurationBuilderCustomizer() {
        return clientConfigurationBuilder -> clientConfigurationBuilder.readFrom(ReadFrom.REPLICA_PREFERRED);
    }
}
```

这个bean中配置的就是读写策略，包括四种：

- MASTER：从主节点读取
- MASTER_PREFERRED：优先从master节点读取，master不可用才读取replica
- REPLICA：从slave（replica）节点读取
- REPLICA _PREFERRED：优先从slave（replica）节点读取，所有的slave都不可用才读取master