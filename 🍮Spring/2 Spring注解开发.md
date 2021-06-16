# Spring注解开发

本章是对Spring常用注解的整理，推荐阅读[尚硅谷Spring注解驱动教程](https://www.bilibili.com/video/BV1gW411W7wy)，阅读之前最好有一定的Spring源码基础，如知道Spring的BeanFactory、BeanDefinition、Aware、PostProcessor、Bean的生命周期等概念。



## 概念

### DI依赖注入

Dependency Injection 依赖注入，**把底层类作为参数传递给上层类，实现上层对下层的控制**



**依赖注入方式**

- Setter
- Interface
- Constructor
- Annotation

![image-20201130003338936](images\image-20201130003338936.png)

### DL依赖查找

Dependency Lookup 依赖查找

## 注入

```java
@Bean
给容器中注册一个Bean，类型为返回值的类型，id默认是方法名作为id

@Configuration
配置类 === 配置文件(以前，我们是用xml去配置的)

//
获取容器，基于xml
BeanFactory beanFactory = new ClassPathXmlApplicationContext("classpath:/META-INF/bean-definition-context.xml");

// 获取容器，基于注解
// MyConfig为配置类
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MyConfig.class);
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   this();
   register(componentClasses);
   refresh();
}
```



## 依赖

```
@DependsOn
```



## 扫描

```
@ComponentScan(basePackages = {"com.example.*"})
包扫描，被@Controller、@Service、@Repository、@Conponent注解的

ComponentScans
可以指定多个ComponentScan
```



## 作用域

```java
@Scope("prototype")
/*
* @see ConfigurableBeanFactory#SCOPE_PROTOTYPE 多实例，原型。 IOC容器启动时并不是直接创建对象放进容器里，而是在获取Bean的时候才会创建放进容器
* @see ConfigurableBeanFactory#SCOPE_SINGLETON 单例(默认) IOC启动时就直接调用创建对象
*/

@RefreshScope
// 刷新Bean
```



## 优先级、顺序

```
@Order
```





## 懒加载

懒加载：容器启动时不创建对象，而是在第一次(获取)Bean创建对象，并初始化。

```java
@Lazy
延迟加载
```



## 条件

```java
@Conditional({WindowsCondition.class})
虚拟机环境变量设置：-Dos.name=linux

// SpringBoot
@ConditionalOnBean
@ConditionalOnMissingBean
@ConditionalOnClass({Feign.class})
```



## 导入

```java
之前是：@Configuration + @Bean + @ComponentScan
但如果是第三方的包没有这些注解，可以使用@Import
@Import(AnnotationBeanDefinitionDemo.Config.class)
导入组件，容器中就会自动注册这个组件，id默认是类的全类名。

org.springframework.context.annotation.ImportSelector 返回需要导入的组件的全类名数组
org.springframework.context.annotation.ImportBeanDefinitionRegistrar 手动注册Bean到容器中

BeanFactory 和FactoryBean
1. FactoryBean默认获取的是工厂Bean调用getObject创建的对象
2. 要获取工厂Bean本身，需要在id前面加一个&，表明想获取的FActoryBean
3. 补充： BeanFactory是底层容器，FactoryBean是用于创建Bean的工厂
public class MyFactoryBean implements org.springframework.beans.factory.FactoryBean<T>{}

public interface BeanFactory {

   /**
    * Used to dereference a {@link FactoryBean} instance and distinguish it from
    * beans <i>created</i> by the FactoryBean. For example, if the bean named
    * {@code myJndiObject} is a FactoryBean, getting {@code &myJndiObject}
    * will return the factory, not the instance returned by the factory.
    */
   String FACTORY_BEAN_PREFIX = "&";
```



## Bean的生命周期

执行顺序@PostConstruct > InitializingBean > initMethod，关于原理在Spring Core一章笔记中有说明

```java
指定初始化(构造方法调用后，调用初始化方法)和销毁方法：

方法一
@Bean(initMethod = "", destroyMethod = "")
初始化：对象创建完成，并赋值好(调用构造方法)，调用初始化方法
销毁：单实例容器关闭的时候，多实例(scope = prototype)容器不会管理这个Bean，容器不会调用销毁方法

方法二
通过Bean实现org.springframework.beans.factory.InitializingBean接口，进行初始化
通过Bean实现org.springframework.beans.factory.DisposableBean接口，进行销毁
构造方法 -> InitializingBean -> DisposableBean

方法三
JSR250
@PostConstruct: 在Bean创建完成并且属性赋值完成，来执行初始化方法
@PreDestroy：在容器销毁Bean之前，通知我们进行清理工作
构造方法 -> @PostConstruct -> @PreDestroy

```

- 初始化
- 实现接口org.springframework.beans.factory.config.BeanPostProcessor： Bean的后置处理器
- org.springframework.beans.factory.config.BeanPostProcessor#postProcessBeforeInitialization：@PostConstruct
- InitializingBean    
- init-method
- org.springframework.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization



## Spring底层对BeanPostProcessor使用

Bean的**后置处理器**，当然Spring当中的后置处理器还有很多，例如BeanFactoryPostProcessor

bean的赋值，注入其他组件，@Autowire，生命周期注解功能，@Async等都是通过BeanPostProcessor实现



刷新容器

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      try {
         // Instantiate all remaining (non-lazy-init) singletons.
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }
}
    
// --------------------------------------------------------------------------------
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   // Instantiate all remaining (non-lazy-init) singletons.
   beanFactory.preInstantiateSingletons();
}
// --------------------------------------------------------------------------------
    @Override
public void preInstantiateSingletons() throws BeansException {
       getBean(beanName);
}
```



```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean // 为属性赋值        
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {

       
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization             
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
      throws BeansException {
```



```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
       // 前置处理
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
       // InitializingBean、init-method
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
       // 后置处理
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```



## ApplicationContextAwareProcessor

```
实现org.springframework.context.support.ApplicationContextAwareProcessor：可以注入容器

public class ApplicationContextUtil implements ApplicationContextAware {

    private ApplicationContext applicationContext;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```



## 赋值

```
@Value("") 可以用在参数中，例如set(@Value("${db.username}")))
@Value("#{20-2}") 可以写SpEL
@Value("${}") 可以写${}

读取外部配置文件的k/v保存到运行的环境变量中;加载完外部的配置文件后使用${}取出配置文件中的值
@PropertySource(value = {"classpath:/xxx.properties"})

获取环境变量，或者
Environment environment = applicationContext.getEnvironment();
environment.getProperty("person.nickname");
```



## 自动注入

```java
@Autowired 优先按照类型，如果找到多个，再将属性中的名称作为组件的id去容器查找
例如：
@Autowired
private BookDao bookDao;  // bookDao就会作为组件的id去容器里查

@Qualifier 指定需要装配的组件的id

@Primary 让Spring自动注入的时候，优先选择
----------------------------------------------------------------------------------------------------------------------
@Autowired可以用在方法，构造器，属性上，都是从容器中获取
@Autowired
public void setCar(Car car){
    this.car = car
}

@Autowired
public Boss(Car car){
    this.car = car
}

@Autowired
private BookDao bookDao; 

@Autowired + @Bean 参数从容器中获取，默认是不用写@Autowired
@Bean
public Car getCar(Test test){ // 参数从容器中获取
    Car car =  new Car();
    car.setName=  test.name;
    return car;
}
```



```java
Java规范的注解
@Resource (JSR250) 默认是按照组件的名称注入

一个是spring的，一个是java的
org.springframework.beans.factory.annotation.Autowired
javax.annotation.Resource

区别：
@Autowired(required = false)，而@Resource不支持@Primary和required = false

@Inject (JSR330) @Named
区别：和@Autowire功能一样，需要导入javax.inject的包，但不支持@Primary和required = false
```



## Aware

提供一种**回调函数**的机制

```java
BeanNameAware
org.springframework.context.ApplicationContextAware 
org.springframework.context.EmbeddedValueResolverAware 解析字符串

class Red implements BeanNameAware, ApplicationContextAware, EmbeddedValueResolverAware{}
```



```java
xxxAware功能是使用xxxProcessor实现的， 
例如：
org.springframework.context.ApplicationContextAware 
org.springframework.context.support.ApplicationContextAwareProcessor


class ApplicationContextAwareProcessor implements BeanPostProcessor {
    @Override
   @Nullable
   public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
            bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
            bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
         return bean;
      }
```



## Profile

Spring为我们提供的可以根据当前环境，动态的激活和切换一系列组件的功能

```java
获取properties内容
第一步，先导入配置文件
@PropertySource(value = {"classpath:/xxx.properties"})
方法一
@Value("") 可以用在参数中，例如set(@Value("${db.username}")))

方法二
实现EmbeddedValueResolverAware 

StringValueResolver resolver;
@Override
public void setEmbeddedValueResolver(StringValueResolver resolver) {
    this.resolver = resolver;
}

private void getProperties(){
    String s = resolver.resolveStringValue("${db.username}");

}
```



```java
@Profile("test")
@Profile("default") 默认

设置环境
方法一
设置JVM参数
-Dspring.profiles.active=test

方法二
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
// 先设置环境
applicationContext.getEnvironment().setActiveProfiles("test");
applicationContext.register(AnnotationBeanDefinitionDemo.class);
// 启动引用上下文
applicationContext.refresh();
```



## AOP

```java
1.通知方法：
@Before 前置通知
@After 后置通知
@AfterReturning 正常返回后
@AfterThrowing 执行出现异常后
@Around 环绕通知，手动执行方法

2.切入点
@Pointcut

3.切面类
@Aspect 

4.加如到容器中
@Componet

5.启用Aspect
@EnableAspectJAutoProxy
SpringBoot中不需要

// 获取签名
Signature signature = joinPoint.getSignature();
// 方法返回的类型
Class returnType = ((MethodSignature) signature).getReturnType();
// 获得方法名称
String methodName = signature.getName();
Class<?> targetClass = joinPoint.getTarget().getClass();
// 获得请求的参数
Object[] args = joinPoint.getArgs();

原理：(看容器中注册了什么组件，功能是什么)
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
利用AspectJAutoProxyRegistrar自定义给容器中注册Bean，
给容器中注册一个org.springframework.aop.config.internalAutoProxyCreator
```

##  

## 声明式事务

```java
@Transactional

@EnableTransactionManagement

其它
JdbcTemplate
```



## Servlet3.0

```java
@WebServlet

@ServletContainerInitializer

Servlet Filter Listener

@EnableWebMvc
WebMvcConfigurerAdapter

异步
@WebServlet(value="/async", asyncSupported=true)
req.startAsync()

@RequestMapping("")
Callable<String> async()

new DeferredResult()
```



## SpringMVC

```java
@Controller
@RestController
@RequestMapping
@GetMapping
@PostMapping
@xxxMapping

Modle
ModleMap
@RequestParam
@RequestBody
@PathVariable
@ModelAttribute

@ResponseBody
```



更多注解，可以在IDEA中输入@然后查看


## @Retryable
```
<dependency>
   <groupId>org.springframework.retry</groupId>
   <artifactId>spring-retry</artifactId>
</dependency>
        
@EnableRetry
@Retryable(value = Exception.class, maxAttempts = 6, backoff = @Backoff(delay = 3000L, multiplier = 60))

value:指定发生的异常进行重试
include:和value一样，默认空，当exclude也为空时，所有异常都重试
exclude:指定异常不重试，默认空，当include也为空时，所有异常都重试
maxAttemps:重试次数，默认3
backoff:重试补偿机制，默认没有。delay = 3000L, multiplier = 60表示第一次3s，第二次隔3*60s，第三次为隔3*60*60s，以此类推
```
