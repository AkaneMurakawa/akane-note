# Redis分布式锁



## 代码示例

```java
分布式锁
/**
 * 分布式锁的服务提供者。此类是线程安全的。
 * 在使用前比如先调用 <code>setJedisCluster</code>
 * 进行初始化。
 * 在使用过程中可以随时调用 <code>setEnabled</code>方法来启用或者禁用，如果设置不启用，那么该方法将不调用
 * redis 的锁。
 *
 **/

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.data.redis.connection.RedisStringCommands;
import org.springframework.data.redis.core.RedisCallback;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.types.Expiration;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.Random;

@Service("distributedLockManager")
public class DistributedLockManager {

    /**
     * 如果是已经取消锁，那么返回此值
     */
    private static final String LOCK_DISABLED_VALUE = "lock_disabled";
    private final Log logger = LogFactory.getLog(getClass());
    /**
     * 所有缓存的 key 加上此前缀
     */
    private final String KEY_PREFIX = "redis:lock.";
    /**
     * 是否启用
     */
    private volatile boolean enabled = true;
    /**
     * 默认超时时间，单位为秒
     */
    private volatile long timeout = 5;
    /**
     * 注入RedisTemplate实例
     */
    @Resource
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 尝试获取指定锁，并设置锁超时时间。
     * 此方法立即返回无论是否能否成功获取锁。
     *
     * @param lockKey     锁的键
     * @param lockTimeout 锁的超时时间，单位为秒
     * @return 如果成功返回一个随机的字符串，解锁的时候需要传入此值来解锁，获取锁失败时立即返回并返回 null
     */
    public String tryLock(final String lockKey, final long lockTimeout) {
        checkLockKey(lockKey);
        checkTimeout(lockTimeout);
        final String key = KEY_PREFIX + lockKey;
        if (enabled && redisTemplate != null) {
            final String lockValue = randomString();
            return redisTemplate.execute((RedisCallback<String>) redisConnection -> {
                redisConnection.set(key.getBytes(), lockValue.getBytes(), Expiration.seconds(lockTimeout), RedisStringCommands.SetOption.SET_IF_ABSENT);
                RedisSerializer serializer = redisTemplate.getStringSerializer();
                Object result = serializer.deserialize(redisConnection.get(key.getBytes()));
                return null == result || !lockValue.equals(result.toString()) ?  "" : result.toString() ;
            });
        }
        return LOCK_DISABLED_VALUE;
    }

    /**
     * 进行解锁。
     *
     * @param lockKey   锁定时设定的键
     * @param lockValue 获取锁时取得的锁值
     * @return 返回true表示成功解锁，返回false表示锁已失效或不存在此锁。
     */
    public boolean unlock(String lockKey, final String lockValue) {
        checkLockKey(lockKey);
        if (enabled && redisTemplate != null) {
            final String key = KEY_PREFIX + lockKey;
            return redisTemplate.execute((RedisCallback<Boolean>) redisConnection -> {
                RedisSerializer serializer = redisTemplate.getStringSerializer();
                Object valueStr = serializer.deserialize(redisConnection.get(key.getBytes()));
                return valueStr != null && (valueStr.toString().equals(lockValue)) && redisConnection.del(key
                        .getBytes()) > 0;
            });
        }
        return false;
    }

    /**
     * 尝试获取指定锁，并设置锁超时时间。
     * 此方法立即返回无论是否能否成功获取锁。
     *
     * @param lockKey 锁的键
     * @return 如果成功返回一个随机的字符串，解锁的时候需要传入此值来解锁，获取锁失败时立即返回并返回 null
     */
    public String tryLock(String lockKey) {
        return tryLock(lockKey, timeout);
    }


    private String randomString() {
        Random random = new Random();
        return System.currentTimeMillis() + "" + random.nextInt(100000);
    }

    private void checkLockKey(String key) {
        if (key == null || key.isEmpty()) {
            throw new RuntimeException("锁的键不能为空");
        }
    }

    private void checkTimeout(long timeout) {
        if (timeout < 0) {
            throw new RuntimeException("锁的超时时间必须不小于0");
        }
    }
}


使用
String method = Thread.currentThread().getStackTrace()[1].getMethodName();
String lockKey = method + 需加锁的id;
String lockValue = distributedLockManager.tryLock(lockKey, 60L);
if (StringUtils.isBlank(lockValue)) {
    log.error("获取锁失败{}", lockKey);
    throw new BusinessException();
}
try {

} finally {
    // 解锁
    if (StringUtils.isNotBlank(lockValue)) {
        // 需通过lockValue来解锁
        distributedLockManager.unlock(lockKey, lockValue);
    }
}
```



## 原理

setnx

```java
public boolean setIfAbsent(String key, Object value, long time) {
    return redisTemplate.execute((RedisCallback<Boolean>) redisConnection -> BooleanUtils.isTrue(redisConnection.set(key.getBytes(), value.toString().getBytes(), Expiration.seconds(time), RedisStringCommands.SetOption.SET_IF_ABSENT)));
}
```