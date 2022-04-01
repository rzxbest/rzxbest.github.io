---
title: Spring
date: 2022-03-31 14:59:29
tags: [JAVA,Spring]
top : 200
categories: [Spring]
---

# Spring
## 注册组件方法  
- 1.包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
- 2.@Bean[导入的第三方包里面的组件]
- 3.@Import[快速给容器中导入一个组件]
        - 1.@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
        - 2.ImportSelector接口:返回需要导入的组件的全类名数组；
        - 3.ImportBeanDefinitionRegistrar接口:手动注册bean到容器中

## spring 扩展点
### spring启动流程图
![b](b.jpeg)

### BeanFactoryPostProcessor
允许对Bean的定义进行修改，向ConfigurableListableBeanFactory里注册spring 的组件，比如添加 BeanPostProcessor、ApplicationListener、类型转换器等等
```
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```
- 应用
    - 单元测试懒加载
        - 有时候整个项目工程中bean的数量有上百个，而大部分单测依赖都是整个工程的xml，导致单测执行时需要很长时间（大部分时间耗费在xml中数百个单例非懒加载的bean的实例化及初始化过程）
        - 解决方法：利用Spring提供的扩展点将xml中的bean设置为懒加载模式，省去了Bean的实例化与初始化时间
    - BeanFactoryPostProcessor 来处理占位符 ${...}，关键的实现类是 PropertySourcesPlaceholderConfigurer，遍历所有的 BeanDefinition，如果 PropertyValue 存在这样的占位符，则会进行解析替换。

```
public class LazyBeanFactoryProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        DefaultListableBeanFactory fac = (DefaultListableBeanFactory) beanFactory;
        Map<String, AbstractBeanDefinition> map = (Map<String, AbstractBeanDefinition>) ReflectionTestUtils.getField(fac, "beanDefinitionMap");
        for (Map.Entry<String, AbstractBeanDefinition> entry : map.entrySet()) {
            //设置为懒加载
            entry.getValue().setLazyInit(true);
        }
    }
}

public class PropertySourcesPlaceholderConfigurer extends PlaceholderConfigurerSupport implements EnvironmentAware {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 省略对 PropertySources 的处理逻辑......
        processProperties(beanFactory, new PropertySourcesPropertyResolver(this.propertySources));
        this.appliedPropertySources = this.propertySources;
    }

    protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
            final ConfigurablePropertyResolver propertyResolver) throws BeansException {

        propertyResolver.setPlaceholderPrefix(this.placeholderPrefix);
        propertyResolver.setPlaceholderSuffix(this.placeholderSuffix);
        propertyResolver.setValueSeparator(this.valueSeparator);
        StringValueResolver valueResolver = new StringValueResolver() {
            @Override
            public String resolveStringValue(String strVal) {
                String resolved = (ignoreUnresolvablePlaceholders ?
                        propertyResolver.resolvePlaceholders(strVal) :
                        propertyResolver.resolveRequiredPlaceholders(strVal));
                if (trimValues) {
                    resolved = resolved.trim();
                }
                return (resolved.equals(nullValue) ? null : resolved);
            }
        };

        // 调用父类 PlaceholderConfigurerSupport 进行 properties 处理
        // 遍历每一个 BeanDefinition，并将其交给 BeanDefinitionVisitor 修改内部的属性
        doProcessProperties(beanFactoryToProcess, valueResolver);
    }
}

```


### BeanDefinitionRegistryPostProcessor
BeanDefinitionRegistryPostProcessor 是 BeanFactoryPostProcessor 的子类，可以通过编码的方式，改变、新增类的定义，甚至删除某些 bean 的定义
```
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}

在 spring 初始化容器之后，解析Class 为一个BeanDefinition 之前的时候去执行， spring 中有一个开天辟地的类 ConfigurationClassPostProcessor 就是实现了该接口， ConfigurationClassPostProcessor 通过重写 这个方法

第一步：去扫描加了 @Configuration 注解的这些类，判断是否是AnnotatedBeanDefinition类型的，将这些类去打上 Lite/Full 的属性，以便后续使用 。
第二步：去实现扫描basePackage下的加了 @Service @Controller @Component 的这些类 将这些类变成为一个BeanDefinition 注册到 Spring 的BeanDefinitionMap中去 。
第三步： 去解析 @import，实现 ImportSelector接口的，实现 ImportBeanDefinitionRegistrar 接口的 这些类的方法，将对应的bean 解析为BeanDefinition，注册到BeanDefinitionMap 中
注意： 在新的Mybatis-spring 的框架中，也使用了到了这个接口 MapperScannerConfigurer 这个扫描配置类 也实现了这个接口！ 重写了postProcessBeanDefinitionRegistry() 用于创建扫描器对象，去扫描mybatis 里的那些接口转化为BeanDefinition到容器中去
作用： 有了参数 registry 的对象，就能手动向spring容器添加 自定义的BeanDefinition registry 提供了添加的 API，但是一般我们不调用 因为在spring里的执行顺序太早了


```
应用：
- spring 集成 mybatis，使用 spring 提供的扫描功能，为我们的Dao接口生成实现类而不需要编写具体的实现类，简化了大量的冗余代码          
    - mybatis-spring 框架就是利用 BeanDefinitionRegistryPostProcessor通过编码的方式往spring容器中添加 bean。
    - MapperScannerConfigurer 重写了 postProcessBeanDefinitionRegistry方法，扫描Dao接口的BeanDefinition，并将BeanDefinition注册到 spring容器中。


```
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, 
    InitializingBean, ApplicationContextAware, BeanNameAware {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        if (this.processPropertyPlaceHolders) {
            processPropertyPlaceHolders();
        }

        // ClassPathMapperScanner 持有 BeanDefinitionRegistry 引用，可以添加 BeanDefinition
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

        // 省略部分代码……，设置 ClassPathMapperScanner 各种属性

        // 扫描并注册 Dao 接口的 BeanDefinition，当然是通过动态代理实现
        scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
    }
}

```

### InstantiationAwareBeanPostProcessor接口

```
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
    Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;
    boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;
    PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException;

}

```
- BeanPostProcessor的子接口，用于在实例化之后，但在设置显式属性或自动装配之前，设置实例化之前的回调函数。通常用于抑制特定目标bean的默认实例化，例如，创建具有特殊TargetSources（池化目标，延迟初始化目标等）的代理，或者实现其他注入策略，例如字段注入。注意：这个接口是一个专用接口，主要用于框架内的内部使用。 建议尽可能实现简单的BeanPostProcessor接口，或者从InstantiationAwareBeanPostProcessorAdapter派生，以便屏蔽此接口的扩展。
- postProcessBeforeInstantiation方法，在目标bean实例化之前创建bean，如果在这里创建了bean，则不会走默认的实例化过程，通常用来创建代理。注意工厂方法生成的bean不会走这个方法。
- postProcessAfterInstantiation方法，在目标bean实例化后，但是没有进行属性填充前执行的方法。
- postProcessPropertyValues方法，在将给定属性值设置到到给定的bean后，对其进行后处理。 允许检查所有的依赖关系是否被满足，例如基于bean属性设置器上的“Required”注解。还允许替换要应用的属性值，通常通过创建基于原始PropertyValues的新MutablePropertyValues实例，添加或删除特定值。

接口应用
- spring不建议用户直接实现，如果必须在这些扩展点应用自己的回调函数，spring建议继承InstantiationAwareBeanPostProcessorAdapter，重写相应的方法即可。
- org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator，基于beanName创建代理，就是应用了这个接口，在生成bean前生成代理bean，从而替代默认的实例化。


### SmartInstantiationAwareBeanPostProcessor

```
public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {
 
// 预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null
    Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException;
// 选择合适的构造器，比如目标对象有多个构造器，在这里可以进行一些定制化，选择合适的构造器
// beanClass参数表示目标实例的类型，beanName是目标实例在Spring容器中的name
// 返回值是个构造器数组，如果返回null，会执行下一个PostProcessor的determineCandidateConstructors方法；否则选取该PostProcessor选择的构造器
    Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException;
// 获得提前暴露的bean引用。主要用于解决循环引用的问题
// 只有单例对象才会调用此方法
    Object getEarlyBeanReference(Object bean, String beanName) throws BeansException;
}
```

getEarlyBeanReference调用时机

```
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
        Object exposedObject = bean;
        if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                    SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                    exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
                    if (exposedObject == null) {
                        return null;
                    }
                }
            }
        }
        return exposedObject;
    }
```
determineCandidateConstructors调用时机

检测Bean的构造器，可以检测出多个候选构造器，再有相应的策略决定使用哪一个，如AutowiredAnnotationBeanPostProcessor实现将自动扫描通过@Autowired/@Value注解的构造器从而可以完成构造器注入




### BeanPostProcessor
BeanPostProcessor允许对bean的实例进行个些自定义的个性，比如检查标记接口、使用代理包装bean实例。spring可以自动检测容器中定义的 BeanPostProcessor，后续创建的bean便会被该BeanPostProcessor处理。

```
public interface BeanPostProcessor {

    // 初始化之后被调用，已完成注入，但是尚未执行 InitializingBean#afterPropertiesSet() 方法，或者自定义的 init 方法
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    // 初始化之后被调用
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```

应用：
- 注解注入、AOP 
    - AnnotationAwareAspectJAutoProxyCreator 在bean的实例化、或者初始化之后创建代理，从而实现 aop 功能。
        
    - spring 根据 BeanDefinition 对bean进行实例化之后，会遍历容器内部注册的 InstantiationAwareBeanPostProcessor(BeanPostProcessor的子类)进行属性填充。
    - AutowiredAnnotationBeanPostProcessor，它会处理 @Autowired、@Value 注解，从而完成注入的功能
    - CommonAnnotationBeanPostProcessor 也是如此，只是处理不同的注解而已
    - 注解注入的检查机制RequiredAnnotationBeanPostProcessor，执行的优先级较低(通过 Ordered 控制)
    - ApplicationContextAwareProcessor

```

class AspectJAutoProxyBeanDefinitionParser implements BeanDefinitionParser {
 @Override
 public BeanDefinition parse(Element element, ParserContext parserContext) {
    //注册特殊的bean
  AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
  extendBeanDefinition(element, parserContext);
  return null;
    }
}

public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {
  public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean != null) {
          Object cacheKey = getCacheKey(bean.getClass(), beanName);
          if (!this.earlyProxyReferences.containsKey(cacheKey)) {
            //如果该类需要被代理，返回动态代理对象；反之，返回原对象
            return wrapIfNecessary(bean, beanName, cacheKey);
          }
        }
        return bean;
 }
}
```


```
class ApplicationContextAwareProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
        // 省略 AccessControlContext 处理的代码
        invokeAwareInterfaces(bean);
        return bean;
    }

    private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof ApplicationEventPublisherAware) {
                ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
            }
            // 省略部分代码……
        }
    }
}

以@Resource为例，看看这个特殊的bean做了什么

public class CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor
  implements InstantiationAwareBeanPostProcessor, BeanFactoryAware, Serializable {
     
      public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, 
      Object bean, String beanName) throws BeansException {
          InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass());
          try {
            //属性注入
            metadata.inject(bean, beanName, pvs);
          }
          catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
          }
          return pvs;
    }
    
}
```

### invokeAware
实现BeanFactoryAware接口的类，会由容器执行setBeanFactory方法将当前的容器BeanFactory注入到类中
```
@Bean
class BeanFactoryHolder implements BeanFactoryAware{
   
    private static BeanFactory beanFactory;
    
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}
```


### afterPropertySet()和init-method
目前很多Java中间件都是基本Spring Framework搭建的，而这些中间件经常把入口放到afterPropertySet或者自定义的init中

### destroy()和destroy-method
bean生命周期的最后一个扩展点，该方法用于执行一些bean销毁前的准备工作，比如将当前bean持有的一些资源释放掉


### ApplicationListener
用于接收spring的事件通知，比如常用的ContextRefreshedEvent事件，spring 在成功完成refresh动作之后便会发出该事件，代表spring容器已经完成初始化了，可以做一些额外的处理了，比如开启 spring 定时任务、拉取 MQ 消息，等等。
spring 处理 @Scheduled 注解的部分实现，在收到 Refresh 事件之后对 ScheduledTaskRegistrar 进行额外的设置，并开启定时任务
```
public class ScheduledAnnotationBeanPostProcessor
        implements MergedBeanDefinitionPostProcessor, DestructionAwareBeanPostProcessor,
        Ordered, EmbeddedValueResolverAware, BeanNameAware, BeanFactoryAware, ApplicationContextAware,
        SmartInitializingSingleton, ApplicationListener<ContextRefreshedEvent>, DisposableBean {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext() == this.applicationContext) {
            finishRegistration();   // 对 ScheduledTaskRegistrar 进行额外的设置，并开启定时任务
        }
    }

    // 其它代码......
}
```

### FactoryBean 扩展点
FactoryBean主要用来定制化Bean的创建逻辑。

FactoryBean 接口提供了三个方法：
- Object getObject()：返回这个 FactoryBean 所创建的对象。
- boolean isSingleton()：返回 FactoryBean 所创建的对象是否为单例，默认返回 true。
- Class getObjectType()：返回这个 FactoryBean 所创建的对象的类型，如果我们能确认返回对象的类型的话，我们应该正常对这个方法做出实现，而不是返回 null。

假设定义了一个FactoryBean，名为MyFactoryBean，当我们调用 getBean("MyFactoryBean")方法时返回的并不是这个FactoryBean，而是这个FactoryBean所创建的 Bean，如果我们想获取到这个FactoryBean 需要在名字前面拼接"&"，例如这种形式：getBean("&MyFactoryBean")

```
public static boolean isFactoryDereference(@Nullable String name) {
    return name != null && name.startsWith("&");
}

protected Object getObjectForBeanInstance(Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
    //判断beanName是不是&开头
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        } else if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        } else {
            if (mbd != null) {
                mbd.isFactoryBean = true;
            }

            return beanInstance;
        }
    }//判断beanInstance 不是factory bean的子类直接返回 
    else if (!(beanInstance instanceof FactoryBean)) {
        return beanInstance;
    } 
    // beanname非&开头，且factory bean的子类
    else {
        Object object = null;
        if (mbd != null) {
            mbd.isFactoryBean = true;
        } else {
            //从缓存里取对象
            object = this.getCachedObjectForFactoryBean(beanName);
        }
        //缓存没有取到
        if (object == null) {
            FactoryBean<?> factory = (FactoryBean)beanInstance;
            if (mbd == null && this.containsBeanDefinition(beanName)) {
                mbd = this.getMergedLocalBeanDefinition(beanName);
            }

            boolean synthetic = mbd != null && mbd.isSynthetic();
            //调用factorybean getobject方法
            object = this.getObjectFromFactoryBean(factory, beanName, !synthetic);
        }

        return object;
    }
}


protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        if (factory.isSingleton() && this.containsSingleton(beanName)) {
            synchronized(this.getSingletonMutex()) {
                Object object = this.factoryBeanObjectCache.get(beanName);
                if (object == null) {
                    //调用factorybean getobject方法
                    object = this.doGetObjectFromFactoryBean(factory, beanName);
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                
                if (alreadyThere != null) {
                        object = alreadyThere;
                    } else {
                        if (shouldPostProcess) {
                            if (this.isSingletonCurrentlyInCreation(beanName)) {
                                return object;
                            }

                            this.beforeSingletonCreation(beanName);

                            try {
                                object = this.postProcessObjectFromFactoryBean(object, beanName);
                            } catch (Throwable var14) {
                                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's singleton object failed", var14);
                            } finally {
                                this.afterSingletonCreation(beanName);
                            }
                        }
                        //判断是否单例，存缓存里
                        if (this.containsSingleton(beanName)) {
                            this.factoryBeanObjectCache.put(beanName, object);
                        }
                    }    
                }
            }
        }
}

```

### 扩展点之间的调用顺序
![a](a.png)