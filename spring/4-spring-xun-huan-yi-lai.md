# 4 Spring循环依赖问题

视频：子路老师Spring循环依赖讲解[https://www.bilibili.com/video/BV1uE411d7L5?p=6](https://www.bilibili.com/video/BV1uE411d7L5?p=6)

注：在观看该视频的时候，其实前面11章已经将循环依赖问题讲解清除，后续的视频有些重复了。

可能大部分人都知道Spring循环依赖问题的解决是通过三级缓存，但是我一直有一个疑问是，Spring处理循环依赖是在哪里调用，真正的调用地方又是在哪里。现在一起来研究一下。

## 前言

循环依赖，简单来说，就是A依赖B，B依赖A。但是如果是简单对象，循环依赖问题是不存在的，很容易解决。

```java
A a = new A();
B b = new B();
a.b = b;
b.a = a;
```

但是Spring中的Bean并不是简单的对象，而是经历了一系列生命周期的Bean，正是因为Bean的生命周期所以Spring才会出现循环依赖问题。

在Spring，默认是支持循环依赖的

```java
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
/** Whether to automatically try to resolve circular references between beans. */
private boolean allowCircularReferences = true;
```

实际上，Spring只是解决了某种场景下的循环。那Spring是如何解决循环依赖的呢？下面一起来探讨一下。

注： 复制Spring源码中的一些无关紧要的代码会进行删减。

## Spring循环依赖示例

```java
@ComponentScan(basePackages = "com.example.*")
public class CircularReferenceConfiguration {
}

// ----------------------------------------------------------------------------
@Component
public class AService {

    @Autowired
    BService bService;

    public AService(){
        System.out.println("A service constructor");
    }
}

// ----------------------------------------------------------------------------
@Component
public class BService {

    @Autowired
    AService aService;

    public BService(){
        System.out.println("B service constructor");
    }
}
// ----------------------------------------------------------------------------
public class CircularReference {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        applicationContext.register(CircularReferenceConfiguration.class);
        applicationContext.refresh();

    }
}
```

## Spring循环依赖的解决过程

**AService**，首先从容器缓存中中获取，**第一次肯定获取不到，**

**而且不会从二三级缓存中获取** \(原因：isSingletonCurrentlyInCreation\(beanName\)为false\)

```java
// Eagerly check singleton cache for manually registered singletons.
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
   bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
// --------------------------------------------------------------------------------
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        // 第一次，AService的isSingletonCurrentlyInCreation肯定是false
        // 因为根本没添加到Set<String> singletonsCurrentlyInCreation里
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
```

接着Spring就开始创建**AService**

```java
// Create bean instance.
if (mbd.isSingleton()) {
   sharedInstance = getSingleton(beanName, () -> {
      try {
         return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
         // Explicitly remove instance from singleton cache: It might have been put there
         // eagerly by the creation process, to allow for circular reference resolution.
         // Also remove any beans that received a temporary reference to the bean.
         destroySingleton(beanName);
         throw ex;
      }
   });
   bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

在这里，需要注意的一点是getSingleton是如何调用的呢? 这里是Java8的函数式编程，会先调用getSingleton，然后再调用createBean方法 **\(也就是调用singletonObject = singletonFactory.getObject\(\);的时候\)**，所以在调用createBean方法的时候，其实已经设置了beforeSingletonCreation\(beanName\);

**这个时候AService就是isSingletonCurrentlyInCreation\(beanName\)为true**

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(beanName, "Bean name must not be null");
   synchronized (this.singletonObjects) {
      Object singletonObject = this.singletonObjects.get(beanName);
      if (singletonObject == null) {
         // isSingletonCurrentlyInCreation = true
         beforeSingletonCreation(beanName);
         boolean newSingleton = false;
         boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
         if (recordSuppressedExceptions) {
            this.suppressedExceptions = new LinkedHashSet<>();
         }
         try {
            // 调用singletonFactory
            singletonObject = singletonFactory.getObject();
            newSingleton = true;
         }
         finally {
            if (recordSuppressedExceptions) {
               this.suppressedExceptions = null;
            }
            afterSingletonCreation(beanName);
         }
         if (newSingleton) {
            addSingleton(beanName, singletonObject);
         }
      }
      return singletonObject;
   }
}
```

创建AService的实例\(createBeanInstance\)，此时AService里的bService属性为null，因为还没创建

```java
instanceWrapper = createBeanInstance(beanName, mbd, args);
```

然后populateBean填充属性之前，会进行一个判断，**是否需要提前暴露**。在这里如果不修改Spring，earlySingletonExposure的三者条件，由前面可知AServcie此时肯定都是true的。

然后从一级缓存中获取不到，便会放入到三级缓存\(singletonFactories\)中，并移除二级缓存\(**其实这个时候earlySingletonObjects 肯定也是不包含的，这里存疑**\)。

```java
// Eagerly cache singletons to be able to resolve circular references
// even when triggered by lifecycle interfaces like BeanFactoryAware.
// AService时，true
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
   addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}

// --------------------------------------------------------------------------------
    protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(singletonFactory, "Singleton factory must not be null");
        synchronized (this.singletonObjects) {
            // 然后从一级缓存中获取不到，便会放入到三级缓存(singletonFactories)中，并移除二级缓存
            if (!this.singletonObjects.containsKey(beanName)) {
                this.singletonFactories.put(beanName, singletonFactory);
                this.earlySingletonObjects.remove(beanName);
                this.registeredSingletons.add(beanName);
            }
        }
    }
```

接着AService便开始**populateBean填充属性**，**由于我们这里使用的是@Autowired注入，因此当bp是AutowiredAnnotationBeanPostProcessor时，便会开始注入bService**

如果是@Resource，对应的后置处理器是CommonAnnotationBeanPostProcessor

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {

   PropertyDescriptor[] filteredPds = null;
   if (hasInstAwareBpps) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
               if (filteredPds == null) {
                  filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
               }
               pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvsToUse == null) {
                  return;
               }
            }
            pvs = pvsToUse;
         }
      }
   }
}
```

这里简单跟踪一下postProcessProperties

postProcessProperties

——&gt;metadata.inject\(bean, beanName, pvs\);

——&gt;element.inject\(target, beanName, pvs\);

——&gt;value = beanFactory.resolveDependency\(desc, beanName, autowiredBeanNames, typeConverter\);

——&gt;result = doResolveDependency\(\);

——&gt;instanceCandidate = descriptor.resolveCandidate

——&gt;beanFactory.getBean\(beanName\);

postProcessProperties这里便开始是创建bService，可以看到最终调用的是beanFactory.getBean\(beanName\);

也就是调用doGetBean\(name, null, null, false\);

此时，依然是拿不到BService

```java
// Eagerly check singleton cache for manually registered singletons.
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
   bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
// --------------------------------------------------------------------------------
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
```

然后便后创建BService实例\(createBeanInstance\)，接着填充属性populateBean，然后发现需要AService，最终又会由populateBean -&gt; postProcessProperties这里调用到beanFactory.getBean\(beanName\);

此时是AService第二次调用，这里AService和BService的isSingletonCurrentlyInCreation=true，于是从一级缓存中获取不到，便会从二级缓存中获取，获取不到便会从三级缓存中获取。此时发现已经可以获取了，并调用singletonFactory.getObject\(\)返回对象，**放到二级缓存中。**

注意，这时候返回的对象是singletonFactory.getObject\(\)获取的Bean，BService便会将拿到的对象赋值给自己的属性bService，然后BService的populateBean结束，继续完成接下来的生命周期initializeBean

```text
// Eagerly check singleton cache for manually registered singletons.
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
   bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
// --------------------------------------------------------------------------------
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
```

最后完成BService的创建，然后调用getSingleton里afterSingletonCreation\(beanName\);进行清除，标记不在创建中。最后将BService放入到一级缓存中。

```java
// Create bean instance.
if (mbd.isSingleton()) {
   sharedInstance = getSingleton(beanName, () -> {
      try {
         return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
         // Explicitly remove instance from singleton cache: It might have been put there
         // eagerly by the creation process, to allow for circular reference resolution.
         // Also remove any beans that received a temporary reference to the bean.
         destroySingleton(beanName);
         throw ex;
      }
   });
   bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}

// -------------------------------------------------------------------------------
if (newSingleton) {
    // 放入到一级缓存中
    addSingleton(beanName, singletonObject);
}
```

然后将创建的BService返回给AService，回到AService的populateBean完成，继续完成接下来的生命周期initializeBean。最后创建结束，放入到一级缓存中。

## 二级缓存的作用

减少工厂singletonFactory.getObject\(\)的重复创建 ，例如AService和CService都用到了BService，那么这两个BService应该是同一个的。如果没有二级缓存，获取的时候需要再调用一次singletonFactory.getObject\(\)。

```java
@Component
public class AService {
    @Autowired
    BService bService;

    public AService(){
        System.out.println("A service constructor");
    }
}

@Component
public class CService {
    @Autowired
    BService bService;

    public CService(){
        System.out.println("A service constructor");
    }
}
```

## 三级缓存

俗称的三级缓存指的是对Bean的缓存，其实在前面我们也可以看到其实Spring中用了很多Map做为缓存，如：beanDefinitionMap、beanDefinitionNames、dependentBeanMap等

* **一级缓存singletonObjects**: 缓存单例Bean，也就是经历了完整的**生命周期**的Bean
* **二级缓存earlySingletonObjects**:  缓存早期\(early\)的Bean，也就是**未完成生命周期**的Bean
* **三级缓存singletonFactories**: 缓存ObjectFactory，对象的工厂

earlySingletonObjects，如我们上面提到的AService，当BService创建的时候，会在三级缓存中获取到AService，然后调用singletonFactory.getObject\(\)。此时AService并没有完成生命周期，而是等到BService创建完毕后，AService执行populateBean完成，继续完成接下来的生命周期initializeBean。最后创建完成放入到一级缓存中。

## AOP

如果使用了AOP，getBean获取的是代理对象。

## 总结

Spring解决循环依赖，主要是通过BeanPostProcessors和三级缓存singletonFactory解决。当调用populateBean时候，通过BeanPostProcessors进行填充属性。

