# SpringBoot

## 自定义Tomcat

```java
package com.example.common.config;

import org.springframework.boot.autoconfigure.web.ServerProperties;
import org.springframework.boot.web.embedded.tomcat.ConfigurableTomcatWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.ConfigurableEnvironment;
import javax.annotation.Resource;

@Configuration
public class TomcatCustomization implements WebServerFactoryCustomizer<ConfigurableTomcatWebServerFactory> {
    @Resource
    private ServerProperties serverProperties;
    @Resource
    private ConfigurableEnvironment environment;

    @Override
    public void customize(ConfigurableTomcatWebServerFactory factory) {
        factory.setBaseDirectory(new File("/usr/local/tomcat/" + environment.getProperty("spring.application.name") + File.separator + serverProperties.getPort() + File.separator));
    }
}

```

