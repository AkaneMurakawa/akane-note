# Mybatis-Plus

## 文档

https://baomidou.com/guide/

## 配置

### 日志

```yml
# 方式一
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl 
    
# 方式二 application.yml 中增加配置，指定 mapper 文件所在的包
logging:
  level:
    com.baomidou.example.mapper: debug
```

### id

```
@TableId(value="id", type=IdType.AUTO)
```

### @TableField

https://baomidou.com/guide/auto-fill-metainfo.html

```
@TableField(fill=FieldFill.INSERT)
```

### 乐观锁

```
@Version
```

```java
// Spring Boot 方式
@Configuration
@MapperScan("按需修改")
public class MybatisPlusConfig {    
    /**
     * 新版
     */
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor mybatisPlusInterceptor = new MybatisPlusInterceptor();
        mybatisPlusInterceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        return mybatisPlusInterceptor;
    }
}
```

### 分页

https://baomidou.com/guide/page.html

```
 // 最新版
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.H2));
        return interceptor;
    }
    
```

### 逻辑删除

```
@TableLogic
private Integer deleted;
```

