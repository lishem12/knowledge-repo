

[toc]



# Redis的BitMap实现签到功能





## BitMap理论

Redis中是利用string类型数据结构实现BitMap，因此最大上限是512M，转换为bit则是 2^32个bit位。

BitMap的操作命令有：

```shell
SETBIT：向指定位置（offset）存入一个0或1
GETBIT ：获取指定位置（offset）的bit值
BITCOUNT ：统计BitMap中值为1的bit位的数量
BITFIELD ：操作（查询、修改、自增）BitMap中bit数组中的指定位置（offset）的值
BITFIELD_RO ：获取BitMap中bit数组，并以十进制形式返回
BITOP ：将多个BitMap的结果做位运算（与 、或、异或）
BITPOS ：查找bit数组中指定范围内第一个0或1出现的位置
```





## 案例：实现签到接口，将当前用户当天签到信息保存到Redis中

因为BitMap底层是基于String数据结构，因此其操作也都封装在字符串相关操作中了。

```java
// Controller
@PostMapping("/sign")
public Result sign(){
    return userService.sign();
}

// Service
@Override
public Result sign() {
    // 1.获取当前登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.获取日期
    LocalDateTime now = LocalDateTime.now();
    // 3.拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
    String key = USER_SIGN_KEY + userId + keySuffix;
    // 4.获取今天是本月的第几天
    int dayOfMonth = now.getDayOfMonth();
    // 5.写入Redis SETBIT key offset 1
    stringRedisTemplate.opsForValue().setBit(key, dayOfMonth - 1, true);
    return Result.ok();
}
```

## 案例：统计连续签到

从最后一次签到开始向前统计，直到遇到第一次未签到为止，计算总的签到次数，就是连续签到天数。

```java
// Controller
@GetMapping("/sign/count")
public Result signCount(){
    return userService.signCount();
}

// Service
@Override
public Result signCount() {
    // 1.获取当前登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.获取日期
    LocalDateTime now = LocalDateTime.now();
    // 3.拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
    String key = USER_SIGN_KEY + userId + keySuffix;
    // 4.获取今天是本月的第几天
    int dayOfMonth = now.getDayOfMonth();
    // 5.获取本月截止今天为止的所有的签到记录，返回的是一个十进制的数字 
    // BITFIELD sign:5:202203 GET u14 0
    List<Long> result = stringRedisTemplate.opsForValue().bitField(
        key,
        BitFieldSubCommands.create()
        .get(BitFieldSubCommands.BitFieldType.unsigned(dayOfMonth)).valueAt(0)
    );
    if (result == null || result.isEmpty()) {
        // 没有任何签到结果
        return Result.ok(0);
    }
    Long num = result.get(0);
    if (num == null || num == 0) {
        return Result.ok(0);
    }
    // 6.循环遍历
    int count = 0;
    while (true) {
        // 6.1.让这个数字与1做与运算，得到数字的最后一个bit位  // 判断这个bit位是否为0
        if ((num & 1) == 0) {
            // 如果为0，说明未签到，结束
            break;
        }else {
            // 如果不为0，说明已签到，计数器+1
            count++;
        }
        // 把数字右移一位，抛弃最后一个bit位，继续下一个bit位
        num >>>= 1;
    }
    return Result.ok(count);
}
```





