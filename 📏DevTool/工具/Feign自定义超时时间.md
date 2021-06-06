# Feign

## 自定义超时时间

```

@Configuration
public class MdataRemoteConfiguration {

    @Bean
    public static Request.Options requestOptions(ConfigurableEnvironment env){
        return new Request.Options(60000, 60000);
    }
}
```

<https://www.jianshu.com/p/191d45210d16>

