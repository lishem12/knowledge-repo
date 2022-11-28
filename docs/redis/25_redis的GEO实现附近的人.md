



# Redis的GEO数据类型实现附近的人



## 1. 理论

GEO就是Geolocation的简写形式，代表地理坐标。Redis在3.2版本中加入了对GEO的支持，允许存储地理坐标信息，帮助我们根据经纬度来检索数据。常见的命令有：

```shell
# GEOADD：添加一个地理空间信息，包含：经度（longitude）、纬度（latitude）、值（member）
# GEODIST：计算指定的两个点之间的距离并返回
# GEOHASH：将指定member的坐标转为hash字符串形式并返回
# GEOPOS：返回指定member的坐标
# GEORADIUS：指定圆心、半径，找到该圆内包含的所有member，并按照与圆心之间的距离排序后返回。6.2以后已废弃
# GEOSEARCH：在指定范围内搜索member，并按照与指定点之间的距离排序后返回。范围可以是圆形或矩形。6.2.新功能
# GEOSEARCHSTORE：与GEOSEARCH功能一致，不过可以把结果存储到一个指定的key。 6.2.新功能
```



举个栗子：

- 添加下面几条数据：
  - 北京南站（ 116.378248 39.865275 ）
  - 北京站（ 116.42803 39.903738 ）
  - 北京西站（ 116.322287 39.893729 ）
- 计算北京西站到北京站的距离
- 搜索天安门（ 116.397904 39.909005 ）附近10km内的所有火车站，并按照距离升序排序

```shell
# 添加 三个站的数据
GEOADD g1 116.378248 39.865275 bjn 116.42803 39.903738 bjz 116.322287 39.893729 bjx

# 计算北京西到北京站距离 默认单位是米
GEODIST g1 bjx bjz

# 搜索天安门（ 116.397904 39.909005 ）附近10km内的所有火车站，并按照距离升序排序
GEOSEARCH g1 FROMLONLAT 116.397904 39.909005 BYRADIUS 10 km WITHDIST
```





## 2. 基于GEO数据类型实现附近的商铺功能

### 2.1 将商铺的地理信息数据导入到redis

```java
@Test
void loadShopData() {
    // 1. 查询店铺信息
    List<Shop> list = shopService.list();
    // 2. 把店铺分组，按照typeId分组，id一致的放到一个集合
    Map<Long, List<Shop>> map = list.stream().collect(Collectors.groupingBy(Shop::getTypeId));
    // 3. 分批完成写入Redis
    for (Map.Entry<Long, List<Shop>> entry : map.entrySet()) {
        // 3.1 获取类型id
        Long typeId = entry.getKey();
        String key = SHOP_GEO_KEY + typeId;
        // 3.2 获取同类型店铺的集合
        List<Shop> value = entry.getValue();
        List<RedisGeoCommands.GeoLocation<String>> locations = new ArrayList<>(value.size());
        // 3.3.写入redis GEOADD key 经度 纬度 member
        for (Shop shop : value) {
            // stringRedisTemplate.opsForGeo().add(key, new Point(shop.getX(), shop.getY()), shop.getId().toString());
            locations.add(new RedisGeoCommands.GeoLocation<>(
                shop.getId().toString(),
                new Point(shop.getX(), shop.getY())
            ));
        }
        stringRedisTemplate.opsForGeo().add(key, locations);
    }
}
```



### 2.2 更新项目依赖

SpringDataRedis的2.3.9版本并不支持Redis 6.2提供的GEOSEARCH命令，因此我们需要提示其版本，修改自己的POM文件，内容如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-redis</artifactId>
        </exclusion>
        <exclusion>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>2.6.2</version>
</dependency>
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>6.1.6.RELEASE</version>
</dependency>
```



### 2.3 实现查询附近商户功能

```java
// Controller
/**
 * 根据商铺类型分页查询商铺信息
 *
 * @param typeId  商铺类型
 * @param current 页码
 * @param x       精度
 * @param y       维度
 * @return 商铺列表
 */
@GetMapping("/of/type")
public Result queryShopByType(
		@RequestParam("typeId") Integer typeId,
		@RequestParam(value = "current", defaultValue = "1") Integer current,
		@RequestParam(value = "x", required = false) Double x,
		@RequestParam(value = "y", required = false) Double y
) {
	return shopService.queryShopByType(typeId, current, x, y);
}


// service
@Override
public Result queryShopByType(Integer typeId, Integer current, Double x, Double y) {
	// 1.判断是否需要根据坐标查询
	if (x == null || y == null) {
		// 不需要坐标查询，按数据库查询
		Page<Shop> page = query()
				.eq("type_id", typeId)
				.page(new Page<>(current, SystemConstants.DEFAULT_PAGE_SIZE));
		// 返回数据
		return Result.ok(page.getRecords());
	}

	// 2.计算分页参数
	int from = (current - 1) * SystemConstants.DEFAULT_PAGE_SIZE;
	int end = current * SystemConstants.DEFAULT_PAGE_SIZE;

	// 3.查询redis、按照距离排序、分页。结果：shopId、distance
	String key = SHOP_GEO_KEY + typeId;
	// GEOSEARCH key BYLONLAT x y BYRADIUS 10 WITHDISTANCE
	GeoResults<RedisGeoCommands.GeoLocation<String>> results = stringRedisTemplate.opsForGeo()
			.search(
					key,
					GeoReference.fromCoordinate(x, y),
					new Distance(5000),
					RedisGeoCommands.GeoSearchCommandArgs.newGeoSearchArgs().includeDistance().limit(end)
			);
	// 4.解析出id
	if (results == null) {
		return Result.ok(Collections.emptyList());
	}
	List<GeoResult<RedisGeoCommands.GeoLocation<String>>> list = results.getContent();
	if (list.size() <= from) {
		// 没有下一页了，结束
		return Result.ok(Collections.emptyList());
	}
	// 4.1.截取 from ~ end的部分
	List<Long> ids = new ArrayList<>(list.size());
	Map<String, Distance> distanceMap = new HashMap<>(list.size());
	list.stream().skip(from).forEach(result -> {
		// 4.2.获取店铺id
		String shopIdStr = result.getContent().getName();
		ids.add(Long.valueOf(shopIdStr));
		// 4.3.获取距离
		Distance distance = result.getDistance();
		distanceMap.put(shopIdStr, distance);
	});
	// 5.根据id查询Shop
	String idStr = StrUtil.join(",", ids);
	List<Shop> shops = query().in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list();
	for (Shop shop : shops) {
		shop.setDistance(distanceMap.get(shop.getId().toString()).getValue());
	}
	// 6.返回
	return Result.ok(shops);
}
```

