

[TOC]

# Redisson的主从一致锁



又称为Redisson联锁



## 原理

就是将单节点拓展为多节点，锁加入到每个Redis中，如果一个Redis宕机，另外几个节点也能保证锁的高可用

![image-20221112213701520](20_Redisson的主从一致锁/image-20221112213701520.png)





## 实现

### 配置

需要配置多个Redisson

```java
package com.hmdp.config;

import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        // 配置
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.2.181:6379").setPassword("123456");
        // 创建RedissonClient对象
        return Redisson.create(config);
    }

    @Bean
    public RedissonClient redissonClient2() {
        // 配置
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.2.181:6380").setPassword("123456");
        // 创建RedissonClient对象
        return Redisson.create(config);
    }

    @Bean
    public RedissonClient redissonClient3() {
        // 配置
        Config config = new Config();
        // 添加redis地址，也可以使用config.useClusterServers();添加集群地址
        config.useSingleServer().setAddress("redis://192.168.2.181:6381").setPassword("123456");
        // 创建RedissonClient对象
        return Redisson.create(config);
    }
}
```

### 使用联锁

```java
package com.hmdp;


import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
import java.util.concurrent.TimeUnit;

@Slf4j
@SpringBootTest
public class RedissonTest {

    @Resource
    private RedissonClient redissonClient;

    @Resource
    private RedissonClient redissonClient2;

    @Resource
    private RedissonClient redissonClient3;

    private RLock lock;

    @BeforeEach
    void setUp() {
        RLock lock1 = redissonClient.getLock("order");
        RLock lock2 = redissonClient2.getLock("order");
        RLock lock3 = redissonClient3.getLock("order");

        // 创建联锁
        lock = redissonClient.getMultiLock(lock1, lock2, lock3);
    }

    @Test
    void method1() throws InterruptedException {
        // 尝试获取锁
        boolean isLock = lock.tryLock(1L, TimeUnit.SECONDS);
        if (!isLock) {
            log.error("获取锁失败 .... 1");
            return;
        }
        try {
            log.info("获取锁成功 .... 1");
            method2();
            log.info("开始执行业务 ... 1");
        } finally {
            log.warn("准备释放锁 .... 1");
            lock.unlock();
        }
    }

    void method2() {
        // 尝试获取锁
        boolean isLock = lock.tryLock();
        if (!isLock) {
            log.error("获取锁失败 .... 2");
            return;
        }
        try {
            log.info("获取锁成功 .... 2");
            log.info("开始执行业务 ... 2");
        } finally {
            log.warn("准备释放锁 .... 2");
            lock.unlock();
        }
    }
}
```



## 总结

1）不可重入Redis分布式锁：

- 原理：利用setnx的互斥性；利用ex避免死锁；释放锁时判断线程标示
- 缺陷：不可重入、无法重试、锁超时失效

2）可重入的Redis分布式锁：

- 原理：利用hash结构，记录线程标示和重入次数；利用watchDog延续锁时间；利用信号量控制锁重试等待
- 缺陷：redis宕机引起锁失效问题

3）Redisson的multiLock：

- 原理：多个独立的Redis节点，必须在所有节点都获取重入锁，才算获取锁成功
- 缺陷：运维成本高、实现复杂