# SpringBoot Cache

Spring从3.1开始定义了Cache和CacheManager接口来统一不同的缓存技术，并支持使用Cache(JSR-107)注解简化开发
```text
* SpringBoot默认的缓存配置
* org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration
* 缓存管理器
* org.springframework.cache.concurrent.ConcurrentMapCacheManager
*
* 当引入其他配置的时候，就会用其他，例如：Redis
* org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration
*
* 每一个缓存管理器，可以管理多个缓存Cache
* CacheManager -> Cache 
*              -> Cache
*              -> Cache
*
* cacheNames/value: 指定缓存组件的名字
* KeyGenerator: key的生成策略
```

## 说明
需要在application.properties配置相应的数据源

然后如果你需要缓存，请使用`@Cacheable`注解，具体的测试可以看`com.example.rediscache.controller.UserController`


## 注解说明
* @EnableCaching: 开启缓存
* @CacheConfig(cacheNames = "userInfo"): 抽取缓存的公共配置
* @Cacheable: 对请求结果进行缓存，常用于查询操作。
* @CachePut: 保证方法被调用，又希望结果被缓存。常用于更新操作。
* @CacheEvict: 清空缓存，常用于删除和插入操作。

## 缓存刷新
@Cacheable和@CachePut搭配使用，或者手动删除，示例：
```java
Set<String> keys = redisDao.keys(CACHE_PREFIX + CACHE_KEY + REDIS_ASTERISK);
redisDao.delete(keys);
```

## @EnableCaching

@EnableCaching注解是spring framework中的注解驱动的缓存管理功能，如果你使用了这个注解，那么就不需要配置cache manager了。

当你在配置类(@Configuration)上使用@EnableCaching注解时，会触发一个post processor，这会扫描每一个spring bean，查看是否已经存在注解对应的缓存。如果找到了，就会自动创建一个代理拦截方法调用，使用缓存的bean执行处理。


