[TOC]



# Redis主从





## 1. 搭建主从架构

单节点Redis的并发能力是有上限的，要进一步提高Redis的并发能力，就需要搭建主从集群，实现读写分离。

![image-20210725152037611](29_无docker搭建redis集群/image-20210725152037611.png)

共包含三个节点，一个主节点，两个从节点。

这里我们会在同一台虚拟机中开启3个redis实例，模拟主从集群，信息如下：

|       IP        | PORT |  角色  |
| :-------------: | :--: | :----: |
| 192.168.174.201 | 7001 | master |
| 192.168.174.201 | 7002 | slave  |
| 192.168.174.201 | 7003 | slave  |

### 1.1. 准备实例和配置

创建三个文件夹，名字分别叫7001、7002、7003：

```shell
# 进入/tmp目录
cd /tmp
# 创建目录
mkdir 7001 7002 7003
```

修改redis-6.2.4/redis.conf文件，将其中的持久化模式改为默认的RDB模式，AOF保持关闭状态。

```shell
# 开启RDB
# save ""
# save 3600 1
# save 300 100
# save 60 10000

# 关闭AOF
appendonly no

# 允许访问ip打开
bind 0.0.0.0
```

然后将redis-6.2.4/redis.conf文件拷贝到三个目录中（在/tmp目录执行下列命令）：

```shell
# 方式一：逐个拷贝
cp redis-6.2.4/redis.conf 7001
cp redis-6.2.4/redis.conf 7002
cp redis-6.2.4/redis.conf 7003

# 方式二：管道组合命令，一键拷贝
echo 7001 7002 7003 | xargs -t -n 1 cp redis-6.2.4/redis.conf
```

修改每个文件夹内的配置文件，将端口分别修改为7001、7002、7003，将rdb文件保存位置都修改为自己所在目录（在/tmp目录执行下列命令）：

```shell
sed -i -e 's/6379/7001/g' -e 's/dir .\//dir \/tmp\/7001\//g' 7001/redis.conf
sed -i -e 's/6379/7002/g' -e 's/dir .\//dir \/tmp\/7002\//g' 7002/redis.conf
sed -i -e 's/6379/7003/g' -e 's/dir .\//dir \/tmp\/7003\//g' 7003/redis.conf
```

虚拟机本身有多个IP，为了避免将来混乱，我们需要在redis.conf文件中指定每一个实例的绑定ip信息，格式如下：

```shell
# redis实例的声明 IP
replica-announce-ip 192.168.174.201

# 每个目录都要改，我们一键完成修改（在/tmp目录执行下列命令）：
# 逐一执行
sed -i '1a replica-announce-ip 192.168.174.201' 7001/redis.conf
sed -i '1a replica-announce-ip 192.168.174.201' 7002/redis.conf
sed -i '1a replica-announce-ip 192.168.174.201' 7003/redis.conf

# 或者一键修改
printf '%s\n' 7001 7002 7003 | xargs -I{} -t sed -i '1a replica-announce-ip 192.168.174.201' {}/redis.conf
```



### 1.2 启动

```shell
# 第1个
redis-server 7001/redis.conf
# 第2个
redis-server 7002/redis.conf
# 第3个
redis-server 7003/redis.conf
```

如果想要一键停止：

```shell
printf '%s\n' 7001 7002 7003 | xargs -I{} -t redis-cli -p {} shutdown
```



### 1.3 开启主从关系

现在三个实例还没有任何关系，要配置主从可以使用replicaof 或者slaveof（5.0以前）命令。

有临时和永久两种模式：

- 修改配置文件（永久生效）

  - 在redis.conf中添加一行配置：```slaveof <masterip> <masterport>``` 即`slaveof 主节点ip 主节点端口`

- 使用redis-cli客户端连接到redis服务，执行slaveof命令（重启后失效）：

  ```sh
  slaveof <masterip> <masterport>
  ```

<strong><font color='red'>注意</font></strong>：在5.0以后新增命令replicaof，与salveof效果一致。

通过redis-cli命令连接7002，执行下面命令：

```shell
# 连接 7002
redis-cli -p 7002
# 执行slaveof
slaveof 192.168.174.201 7001
```

通过redis-cli命令连接7003，执行下面命令：

```shell
# 连接 7003
redis-cli -p 7003
# 执行slaveof
slaveof 192.168.174.201 7001
```

然后连接 7001节点，查看集群状态：

```shell
# 连接 7001
redis-cli -p 7001
# 查看状态
info replication

### 可以看出 集群已经搭建完成
127.0.0.1:7001> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=192.168.174.201,port=7002,state=online,offset=140,lag=1
slave1:ip=192.168.174.201,port=7003,state=online,offset=140,lag=1
master_failover_state:no-failover
master_replid:8fcd1fdf5fc0af74627101cf773ae048b661e832
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:140
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:140
```



### 1.4 测试

执行下列操作以测试：

- 利用redis-cli连接7001，执行```set num 123```

- 利用redis-cli连接7002，执行```get num```，再执行```set num 666```

- 利用redis-cli连接7003，执行```get num```，再执行```set num 888```

可以发现，只有在7001这个master节点上可以执行写操作，7002和7003这两个slave节点只能执行读操作。

读写已经分离。

### 1.5 小结

`redis` 集群搭建就是在 从节点上执行 `slaveof` 命令



## 2. 主从同步原理

### 2.1 全量同步

主从第一次建立连接时，会执行**全量同步**，将master节点的所有数据都拷贝给slave节点，流程：

![image-20210725152222497](29_redis主从/image-20210725152222497.png)

这里有一个问题，master如何得知salve是第一次来连接呢？？

有几个概念，可以作为判断依据：

- **Replication Id**：简称replid，是数据集的标记，id一致则说明是同一数据集。每一个master都有唯一的replid，slave则会继承master节点的replid
- **offset**：偏移量，随着记录在repl_baklog中的数据增多而逐渐增大。slave完成同步时也会记录当前同步的offset。如果slave的offset小于master的offset，说明slave数据落后于master，需要更新。

因此slave做数据同步，必须向master声明自己的`replication id` 和`offset`，master才可以判断到底需要同步哪些数据。

因为slave原本也是一个master，有自己的replid和offset，当第一次变成slave，与master建立连接时，发送的replid和offset是自己的replid和offset。

master判断发现slave发送来的replid与自己的不一致，说明这是一个全新的slave，就知道要做全量同步了。

master会将自己的replid和offset都发送给这个slave，slave保存这些信息。以后slave的replid就与master一致了。

因此，**master判断一个节点是否是第一次同步的依据，就是看replid是否一致**。

如图：

![image-20210725152700914](29_redis主从/image-20210725152700914.png)

完整流程描述：

- slave节点请求增量同步
- master节点判断replid，发现不一致，拒绝增量同步
- master将完整内存数据生成RDB，发送RDB到slave
- slave清空本地数据，加载master的RDB
- master将RDB期间的命令记录在repl_baklog，并持续将log中的命令发送给slave
- slave执行接收到的命令，保持与master之间的同步



### 2.2 增量同步

全量同步需要先做RDB，然后将RDB文件通过网络传输个slave，成本太高了。因此除了第一次做全量同步，其它大多数时候slave与master都是做**增量同步**。

什么是增量同步？就是只更新slave与master存在差异的部分数据。如图：

![image-20210725153201086](29_redis主从/image-20210725153201086.png)

那么master怎么知道slave与自己的数据差异在哪里呢?



### 2.3 repl_backlog原理

master怎么知道slave与自己的数据差异在哪里呢?

这就要说到全量同步时的repl_baklog文件了。

这个文件是一个固定大小的数组，只不过数组是环形，也就是说**角标到达数组末尾后，会再次从0开始读写**，这样数组头部的数据就会被覆盖。

repl_baklog中会记录Redis处理过的命令日志及offset，包括master当前的offset，和slave已经拷贝到的offset：

![image-20210725153359022](29_redis主从/image-20210725153359022.png)

slave与master的offset之间的差异，就是salve需要增量拷贝的数据了。

随着不断有数据写入，master的offset逐渐变大，slave也不断的拷贝，追赶master的offset：

![image-20210725153524190](29_redis主从/image-20210725153524190.png)

直到数组被填满：

![image-20210725153715910](29_redis主从/image-20210725153715910.png)

此时，如果有新的数据写入，就会覆盖数组中的旧数据。不过，旧的数据只要是绿色的，说明是已经被同步到slave的数据，即便被覆盖了也没什么影响。因为未同步的仅仅是红色部分。

但是，如果slave出现网络阻塞，导致master的offset远远超过了slave的offset： 

![image-20210725153937031](29_redis主从/image-20210725153937031.png)

如果master继续写入新数据，其offset就会覆盖旧的数据，直到将slave现在的offset也覆盖：

![image-20210725154155984](29_redis主从/image-20210725154155984.png)

棕色框中的红色部分，就是尚未同步，但是却已经被覆盖的数据。此时如果slave恢复，需要同步，却发现自己的offset都没有了，无法完成增量同步了。只能做全量同步。

> 注意：
>
> repl_baklog大小有上限，写满后会覆盖最早的数据。如果slave断开时间过久，导致尚未备份的数据被覆盖，则无法基于log做增量同步，只能再次全量同步。



### 2.4 主从同步优化

主从同步可以保证主从数据的一致性，非常重要。

可以从以下几个方面来优化Redis主从就集群：

- 在master中配置`repl-diskless-sync yes`启用无磁盘复制，避免全量同步时的磁盘IO。
- Redis单节点上的内存占用不要太大，减少RDB导致的过多磁盘IO
- 适当提高`repl_baklog`的大小，发现slave宕机时尽快实现故障恢复，尽可能避免全量同步
- 限制一个master上的slave节点数量，如果实在是太多slave，则可以采用主-从-从链式结构，减少master压力

主从从架构图：

![image-20210725154405899](29_redis主从/image-20210725154405899.png)



### 2.5 小结

简述全量同步和增量同步区别？

- 全量同步：master将完整内存数据生成RDB，发送RDB到slave。后续命令则记录在repl_baklog，逐个发送给slave。
- 增量同步：slave提交自己的offset到master，master获取repl_baklog中从offset之后的命令给slave

什么时候执行全量同步？

- slave节点第一次连接master节点时
- slave节点断开时间太久，repl_baklog中的offset已经被覆盖时

什么时候执行增量同步？

- slave节点断开又恢复，并且在repl_baklog中能找到offset时

