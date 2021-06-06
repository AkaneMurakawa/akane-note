# Google Cache

```
import com.google.common.base.Throwables;
import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import lombok.extern.slf4j.Slf4j;

import java.util.Objects;
import java.util.concurrent.Callable;
import java.util.concurrent.TimeUnit;

/**
 * 缓存管理
 */
@Slf4j
public class ComponentCacheManager<T> {

    private Cache<Object, T> cache;

    private ComponentCacheManager(long cacheSize) {
        this.cache = CacheBuilder.newBuilder().maximumSize(cacheSize)
                .expireAfterWrite(24L, TimeUnit.HOURS)
                .expireAfterAccess(24 * 30L, TimeUnit.HOURS)
                .build();
    }

    private ComponentCacheManager(long cacheSize, long expireAfterWrite, long expireAfterAccess, TimeUnit timeUnit) {
        this.cache = CacheBuilder.newBuilder().maximumSize(cacheSize)
                .expireAfterWrite(expireAfterWrite, timeUnit)
                .expireAfterAccess(expireAfterAccess, timeUnit)
                .build();
    }

    public static <T> ComponentCacheManager<T> create(long cacheSize) {
        return new ComponentCacheManager<>(cacheSize);
    }

    public static <T> ComponentCacheManager<T> create(long cacheSize, long expireAfterWrite, long expireAfterAccess, TimeUnit timeUnit) {
        return new ComponentCacheManager<>(cacheSize, expireAfterWrite, expireAfterAccess, timeUnit);
    }

    /**
     * 根据键值获取缓存
     *
     * @param key  键值
     * @param call 缓存源获取方法
     */
    public T get(String key, Callable<T> call) {
        try {
            return cache.get(key, call);
        } catch (Exception e) {
            log.error("缓存服务异常,key:{},异常:", key, e);
            try {
                T value = call.call();
                if (Objects.nonNull(value)){
                    cache.put(key, value);
                    return value;
                }
            } catch (Exception e1) {
                log.error("", e1);
                Throwables.throwIfUnchecked(e1);
            }
        }
        return null;
    }

    /**
     * 清空缓存
     */
    public void cleanUp() {
        cache.cleanUp();
    }
}
```