# 5 Mybatis和SpringBoot

阅读源码的意义，学习其实现的架构和思想，其次懂得扩展。

本篇文章研究Mybatis自动装配过程

首先，引入pom

```markup
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>
```

查看自动配置类，spring.factories

```text
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisLanguageDriverAutoConfiguration,\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```

## MybatisAutoConfiguration

1. 首先查看MybatisAutoConfiguration结构，实现了InitializingBean，这个可以忽略。
2. 主要的两个方法：sqlSessionTemplate\(\)和sqlSessionFactory\(\)，这两个方法都加上了@Bean和@ConditionalOnMissingBean注解，如果我们未定义则会注入默认的Bean。
3. 另外在这个类中，还包含了两个内部类：AutoConfiguredMapperScannerRegistrar，MapperScannerRegistrarNotFoundConfiguration。可以看到MapperScannerRegistrarNotFoundConfiguration通过@Import导入了AutoConfiguredMapperScannerRegistrar，所以自动装配的过程其实是在AutoConfiguredMapperScannerRegistrar。

![mybatis](../.gitbook/assets/mybatis14.png)

## AutoConfiguredMapperScannerRegistrar

该类继承了ImportBeanDefinitionRegistrar，这里的ImportBeanDefinitionRegistrar用于将配置类上的BeanDefinition注册到容器中。

> 通过Import注解可以注册bean，虽然它也支持同时注册多个bean，但是不方便，特别是当我们想通过实现一些复杂逻辑来注册bean的话，仅仅通过Import注解是不方便的，这时就可以使用ImportBeanDefinitionRegistrar这个接口来动态的注册bean了，我这里说的注册bean指的是：**通过生成BeanDefinition，并且把BeanDefinition放入BeanFactory中。**

这里主要分两个过程：

1. 首先获取包的扫描路径
2. 扫描mapper

```java
  public static class AutoConfiguredMapperScannerRegistrar
      implements BeanFactoryAware, ImportBeanDefinitionRegistrar, ResourceLoaderAware {

    private BeanFactory beanFactory;

    private ResourceLoader resourceLoader;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

      if (!AutoConfigurationPackages.has(this.beanFactory)) {
        logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.");
        return;
      }

      logger.debug("Searching for mappers annotated with @Mapper");
        // 扫描包路径
      List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
      if (logger.isDebugEnabled()) {
        for (String pkg : packages) {
          logger.debug("Using auto-configuration base package '{}'", pkg);
        }
      }

      ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
      if (this.resourceLoader != null) {
        scanner.setResourceLoader(this.resourceLoader);
      }
      scanner.setAnnotationClass(Mapper.class);
      scanner.registerFilters();
      // mybatis-spring 扫描mapper
      scanner.doScan(StringUtils.toStringArray(packages));

    }
```

### doScan

在这里扫描包，并生成了BeanDefinition，但是为什么还要再处理一次呢？

这里我们可以思考一下，首先我们在代码里写的是一个接口，如果是单纯的Mybatis，没有结合Spring，我们的Mapper对象是通过SqlSession.getMapper获取的，然后SqlSession又从Configuration中获取，而Configuration是在一开始解析的时候就生成的Mapper对象。

而如果是Mybatis-Spring，我们需要将Mapper放到Spring容器中，那如何才能让Spring帮我们管理Bean呢，答案就是：生成一个FactoryBean。

```java
/**
 * Calls the parent search that will search and register all the candidates.
 * Then the registered objects are post processed to set them as
 * MapperFactoryBeans
 */
@Override
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
   //  扫描并生成BeanDefinition
  Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

  if (beanDefinitions.isEmpty()) {
    logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
  } else {
    // 处理BeanDefinitions
    processBeanDefinitions(beanDefinitions);
  }

  return beanDefinitions;
}
```

### processBeanDefinitions

definition.setBeanClass\(this.mapperFactoryBean.getClass\(\)\);在这一步设置BeanClass为FactoryBean类型

```java
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();

      if (logger.isDebugEnabled()) {
        logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
          + "' and '" + definition.getBeanClassName() + "' mapperInterface");
      }

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
      // 设置BeanClass为FactoryBean类型
      definition.setBeanClass(this.mapperFactoryBean.getClass());

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        if (logger.isDebugEnabled()) {
          logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        }
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
```

### MapperFactoryBean

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
```

MapperFactoryBean继承了FactoryBean，我们来看一下getObject\(\)方法，该方法用于返回工厂返回的Bean

```text
  /**
   * {@inheritDoc}
   */
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }

  // -----------------------------------------------------------------——>
  /**
   * {@inheritDoc}
   */
  @Override
  public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
  }

  // -----------------------------------------------------------------——>
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```

### MapperRegistry

1. 获取Mapper代理工厂
2. 实例化

```java
public class MapperRegistry {

  private final Configuration config;
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();

  public MapperRegistry(Configuration config) {
    this.config = config;
  }

  @SuppressWarnings("unchecked")
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 获取Mapper代理工厂
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      // 实例化  
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

### MapperProxyFactory.newInstance

1. MapperProxy实现了InvocationHandler，底层会自动调用invoke，这便是JDK动态代理
2. invoke中执行SQL，mapperMethod.execute\(sqlSession, args\);

```java
/**
 * @author Lasse Voss
 */
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    // 动态代理
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}


// -------------------------------------------------------------------------------------------------
public class MapperProxy<T> implements InvocationHandler, Serializable {
   @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    // 执行SQL
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
}
```

## 小结

1. Mybatis的自动配置依赖于MybatisAutoConfiguration
2. 然后通过ClassPathMapperScanner扫描包，解析注入到BeanDefinition中。
3. 设置BeanClass为FactoryBean类型，MapperFactoryBean通过动态代理生成

