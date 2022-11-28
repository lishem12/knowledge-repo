

[TOC]

# Redis 缓存工具类封装



基于StringRedisTemplate封装一个缓存工具类，满足下列需求：

存：

- 方法1：将任意Java对象序列化为json并存储在string类型的key中，并且可以设置TTL过期时间
- 方法2：将任意Java对象序列化为json并存储在string类型的key中，并且可以设置逻辑过期时间，用于处理缓存击穿问题

取：

- 方法3：根据指定的key查询缓存，并反序列化为指定类型，利用缓存空值的方式解决缓存穿透问题
- 方法4：根据指定的key查询缓存，并反序列化为指定类型，需要利用逻辑过期解决缓存击穿问题



```java
package com.hmdp.utils;

import cn.hutool.core.util.BooleanUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import com.hmdp.entity.Shop;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;

@Slf4j
@Component
public class CacheClient {

    private final StringRedisTemplate stringRedisTemplate;

    public CacheClient(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    /**
     * 写值，并且设置TTL
     *
     * @param key
     * @param value
     * @param time
     * @param unit
     */
    public void set(String key, Object value, Long time, TimeUnit unit) {
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value), time, unit);
    }

    /**
     * 写值，并设置逻辑过期时间
     *
     * @param key
     * @param value
     * @param time
     * @param unit
     */
    public void setWithLogicalExpire(String key, Object value, Long time, TimeUnit unit) {
        // 设置逻辑过期
        RedisData redisData = new RedisData();
        redisData.setData(value);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
        // 写入Redis
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
    }

    /**
     * 根据指定的key查询缓存，并反序列化为指定类型，利用缓存空值的方式解决缓存穿透问题
     *
     * @param keyPrefix  redis前缀
     * @param id         查询的id
     * @param type       查询对象的类型
     * @param dbFallBack 调用查数据库的方法
     * @param time       时间
     * @param unit       单位
     * @param <R>        返回值类型
     * @param <ID>       id类型
     * @return
     */
    public <R, ID> R queryWithPassThrough(String keyPrefix, ID id, Class<R> type,
                                          Function<ID, R> dbFallBack, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        // 1. 从redis查询商铺缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        // 2. 判断是否存在
        if (StrUtil.isNotBlank(json)) {
            // 3. 存在，直接返回
            return JSONUtil.toBean(json, type);
        }
        // 判断命中是否是空
        if (json != null) {
            // 返回错误信息
            return null;
        }
        // 4. 不存在，根据id查询缓存
        R r = dbFallBack.apply(id);
        // 5. 不存在，返回错误
        if (r == null) {
            // 将空值写入redis
            stringRedisTemplate.opsForValue().set(key, "", RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
            return null;
        }
        // 6. 存在，写入Redis
        this.set(key, r, time, unit);
        // 7. 返回
        return r;
    }

    private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

    /**
     * 根据指定的key查询缓存，并反序列化为指定类型，利用逻辑过期解决缓存击穿问题
     *
     * @param keyPrefix  redis key前缀
     * @param id         查询id
     * @param type       查询得到的类型
     * @param dbFallBack 查询数据库函数
     * @param time       时间
     * @param unit       单位
     * @param <R>        返回类型
     * @param <ID>       id类型
     * @return
     */
    public <R, ID> R queryWithLogicalExpire(String keyPrefix, ID id, Class<R> type,
                                            Function<ID, R> dbFallBack, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        // 1. 从redis查询商铺缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        // 2. 判断是否存在
        if (StrUtil.isBlank(json)) {
            // 3. 不存在，直接返回null
            return null;
        }
        // 4. 命中需要先把Json反序列化为对象
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        JSONObject data = (JSONObject) redisData.getData();
        R r = JSONUtil.toBean(data, type);
        LocalDateTime expireTime = redisData.getExpireTime();
        // 5. 判断是否过期
        if (expireTime.isAfter(LocalDateTime.now())) {
            // 5.1 未过期，直接返回店铺信息
            return r;
        }
        // 5.2 已过期缓存重建
        // 6. 缓存重建
        // 6.1 获取互斥锁
        String localKey = RedisConstants.LOCK_SHOP_KEY + id;
        boolean isLock = tryLock(localKey);
        // 6.2 判断是否获取锁成功
        if (isLock) {
            // 6.3 成功，开启独立线程实现缓存重建
            CACHE_REBUILD_EXECUTOR.submit(() -> {
                try {
                    // 重建缓存
                    // 查数据库
                    R r1 = dbFallBack.apply(id);
                    // 存入redis
                    this.setWithLogicalExpire(key, r1, time, unit);
                } finally {
                    // 释放锁
                    unlock(localKey);
                }
            });
        }
        // 6.4 返回过期商铺信息
        return r;
    }

    private boolean tryLock(String key) {
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);
    }

    private void unlock(String key) {
        stringRedisTemplate.delete(key);
    }
}
```





调用：

```java
// 利用缓存空值的方式解决缓存穿透问题
Shop shop = cacheClient.queryWithPassThrough(RedisConstants.CACHE_SHOP_KEY,
                id, Shop.class, this::getById,
                RedisConstants.CACHE_SHOP_TTL, TimeUnit.MINUTES);


// 利用逻辑过期解决缓存击穿问题
Shop shop = cacheClient.queryWithLogicalExpire(RedisConstants.CACHE_SHOP_KEY,
                id, Shop.class, this::getById, RedisConstants.CACHE_SHOP_TTL, TimeUnit.MINUTES);
```

