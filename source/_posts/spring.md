---
title: Spring
date: 2022-03-31 14:59:29
tags: [JAVA,Spring]
top : 200
categories: [Spring]
---

# Spring

- 给容器中注册组件；
   - 1 包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
   - 2 @Bean[导入的第三方包里面的组件]
   - 3 @Import[快速给容器中导入一个组件]
             1）、@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
             2）、ImportSelector接口:返回需要导入的组件的全类名数组；
             3）、ImportBeanDefinitionRegistrar接口:手动注册bean到容器中

spring 扩展点
BeanPostProcessor
首先，我们来看下官方的注释，解释得非常到位

BeanPostProcessor 这是一个勾子，允许对 bean 的实例进行个些自定义的个性，比如检查标记接口、使用代理包装 bean 实例。spring 可以自动检测容器中定义的 BeanPostProcessor，后续创建的 bean 便会被该 BeanPostProcessor 处理。此外，BeanFactory 还允许通过编码的方式注册 BeanPostProcessor

BeanPostProcessor 接口定义如下所示：
————————————————
版权声明：本文为CSDN博主「黄小厮」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/Dwade_mia/article/details/79372768

public interface BeanPostProcessor {

    // 初始化之后被调用，已完成注入，但是尚未执行 InitializingBean#afterPropertiesSet() 方法，或者自定义的 init 方法
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    // 初始化之后被调用
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}


由于 BeanPostProcessor 可以修改 bean 的实例对象，因此在 spring 框架中得到了非常广泛的应用，比如注解注入、AOP 等等。这是一个非常重要的扩展点，就拿注解注入来说吧，spring 根据 BeanDefinition 对 bean 进行实例化之后，会遍历容器内部注册的 InstantiationAwareBeanPostProcessor(BeanPostProcessor的子类)进行属性填充，其中就包括 AutowiredAnnotationBeanPostProcessor，它会处理 @Autowired、@Value 注解，从而完成注入的功能，而 CommonAnnotationBeanPostProcessor 也是如此，只是处理不同的注解而已
pring 有很多内置的 BeanPostProcessor，我们来看个简单的例子，帮助我们理解这个接口的作用。

我们知道 spring 提供了很多 Aware 帮助我们注入 spring 内置的 bean，比如 ApplicationContextAware、ResourceLoaderAware，那么 spring 又是如何实现该功能的呢？非常的简单，只要写一个 BeanPostProcessor，判断下这个 bean 是否实现了该 Aware 接口，如果是则调用 setXXX() 方法进行赋值即可。这个类在 spring-context 模块，叫做 ApplicationContextAwareProcessor，我们来看下源代码


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


通过这个简单的例子，我们可以感受到 BeanPostProcessor 的强大，不过，这仅仅是 spring 的冰山一角，还有很多更高级的应用，比如 AnnotationAwareAspectJAutoProxyCreator 在 bean 的实例化、或者初始化之后创建代理，从而实现 aop 功能。再比如，spring 对注解注入的检查机制是由 RequiredAnnotationBeanPostProcessor 完成的，也是一个 InstantiationAwareBeanPostProcessor 实现，只不过它执行的优先级较低(通过 Ordered 控制)，因为需要等待所有的 InstantiationAwareBeanPostProcessor 完成注入之后才能进行注入检查，否则会有逻辑上的问题




BeanFactoryPostProcessor
BeanFactoryPostProcessor 允许对 Bean 的定义进行修改，还可以往 ConfigurableListableBeanFactory 中注册一些 spring 的组件，比如添加 BeanPostProcessor、ApplicationListener、类型转换器等等，接口定义如下：

public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}

「BeanFactoryPostProcessor#postProcessBeanFactory」

有时候整个项目工程中bean的数量有上百个，而大部分单测依赖都是整个工程的xml，导致单测执行时需要很长时间（大部分时间耗费在xml中数百个单例非懒加载的bean的实例化及初始化过程）

解决方法：利用Spring提供的扩展点将xml中的bean设置为懒加载模式，省去了Bean的实例化与初始化时间

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

spring 利用 BeanFactoryPostProcessor 来处理占位符 ${...}，关键的实现类是 PropertySourcesPlaceholderConfigurer，它会遍历所有的 BeanDefinition，如果 PropertyValue 存在这样的占位符，则会进行解析替换，下面我们来简单的分析下实现原理。

我们可以看到 PropertySourcesPlaceholderConfigurer 重写了 BeanFactoryPostProcessor 的 postProcessBeanFactory 方法，遍历了 BeanFactory 内部注册的 BeanDefinition，并且使用 BeanDefinitionVisitor 修改其属性值，从而完成占位符的替换。如果，我们要对某些配置进行加解密处理，也可以使用这种方式进行处理
PropertySourcesPlaceholderConfigurer.java

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


BeanDefinitionRegistryPostProcessor
BeanDefinitionRegistryPostProcessor 是 BeanFactoryPostProcessor 的子类，可以通过编码的方式，改变、新增类的定义，甚至删除某些 bean 的定义

public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}

这也是一个非常有用的扩展点，比如 spring 集成 mybatis，我们可以使用 spring 提供的扫描功能，为我们的 Dao 接口生成实现类而不需要编写具体的实现类，简化了大量的冗余代码，mybatis-spring 框架就是利用 BeanDefinitionRegistryPostProcessor 通过编码的方式往 spring 容器中添加 bean。

关键代码如下所示，MapperScannerConfigurer 重写了 postProcessBeanDefinitionRegistry 方法，扫描 Dao 接口的 BeanDefinition，并将 BeanDefinition 注册到 spring 容器中。我们也可以借鉴这种思路，通过具体的应用场景，从 spring 容器中删除或者新增 BeanDefinition


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

ApplicationListener
用于接收spring的事件通知，比如常用的 ContextRefreshedEvent 事件，spring 在成功完成 refresh 动作之后便会发出该事件，代表 spring 容器已经完成初始化了，可以做一些额外的处理了，比如开启 spring 定时任务、拉取 MQ 消息，等等。

下面列出了 spring 处理 @Scheduled 注解的部分实现，在收到 Refresh 事件之后对 ScheduledTaskRegistrar 进行额外的设置，并开启定时任务

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


「InstantiationAwareBeanPostProcessor#postProcessPropertyValues」

非常规的配置项比如

<context:component-scan base-package="com.zhou" />
Spring提供了与之对应的特殊解析器

正是通过这些特殊的解析器才使得对应的配置项能够生效

而针对这个特殊配置的解析器为 ComponentScanBeanDefinitionParser

在这个解析器的解析方法中，注册了很多特殊的Bean

public BeanDefinition parse(Element element, ParserContext parserContext) {
  //...
  registerComponents(parserContext.getReaderContext(), beanDefinitions, element);
    //...
  return null;
}

public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
   BeanDefinitionRegistry registry, Object source) {

  Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(4);
  //...
    //@Autowire
  if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
   def.setSource(source);
   beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
   //@Resource
  if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      //特殊的Bean
   RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
   def.setSource(source);
   beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
  }
  //...
  return beanDefs;
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


「invokeAware」

实现BeanFactoryAware接口的类，会由容器执行setBeanFactory方法将当前的容器BeanFactory注入到类中

@Bean
class BeanFactoryHolder implements BeanFactoryAware{
   
    private static BeanFactory beanFactory;
    
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}
「BeanPostProcessor#postProcessBeforeInitialization」

实现ApplicationContextAware接口的类，会由容器执行setApplicationContext方法将当前的容器applicationContext注入到类中

@Bean
class ApplicationContextAwareProcessor implements BeanPostProcessor {

    private final ConfigurableApplicationContext applicationContext;

    public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
      this.applicationContext = applicationContext;
    }

    @Override
    public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
      //...
      invokeAwareInterfaces(bean);
      return bean;
    }

    private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof ApplicationContextAware) {
          ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
    }
}
我们看到是在BeanPostProcessor的postProcessBeforeInitialization中进行了setApplicationContext方法的调用

class ApplicationContextHolder implements ApplicationContextAware{
   
    private static ApplicationContext applicationContext;
    
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
「afterPropertySet()和init-method」

目前很多Java中间件都是基本Spring Framework搭建的，而这些中间件经常把入口放到afterPropertySet或者自定义的init中

「BeanPostProcessor#postProcessAfterInitialization」

熟悉aop的同学应该知道，aop底层是通过动态代理实现的

当配置了<aop:aspectj-autoproxy/>时候，默认开启aop功能，相应地调用方需要被aop织入的对象也需要替换为动态代理对象

不知道大家有没有思考过动态代理是如何「在调用方无感知情况下替换原始对象」的？

❝ 根据上文的讲解，我们知道：
❞
<aop:aspectj-autoproxy/>
Spring也提供了特殊的解析器，和其他的解析器类似，在核心的parse方法中注册了特殊的bean

这里是一个BeanPostProcessor类型的bean

class AspectJAutoProxyBeanDefinitionParser implements BeanDefinitionParser {
 @Override
 public BeanDefinition parse(Element element, ParserContext parserContext) {
    //注册特殊的bean
  AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
  extendBeanDefinition(element, parserContext);
  return null;
    }
}
将于当前bean对应的动态代理对象返回即可，该过程对调用方全部透明

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
正是利用Spring的这个扩展点实现了动态代理对象的替换

「destroy()和destroy-method」

bean生命周期的最后一个扩展点，该方法用于执行一些bean销毁前的准备工作，比如将当前bean持有的一些资源释放掉


BeanDefinition与BeanFactory扩展
在Spring生成bean的过程这篇文章中，我们了解了spring在生成bean前会先生成bean的定义，然后注册到BeanFactory中，再之后才能生成bean。那么对于从xml配置好的BeanDefinition，如果想要增加删除修改该怎么办呢？

BeanDefinitionRegistryPostProcessor接口
BeanDefinitionRegistryPostProcessor接口继承了BeanFactoryPostProcessor接口，BeanFactoryPostProcessor接口随后我们也会讲到这个接口。

/**
 * Extension to the standard {@link BeanFactoryPostProcessor} SPI, allowing for
 * the registration of further bean definitions <i>before</i> regular
 * BeanFactoryPostProcessor detection kicks in. In particular,
 * BeanDefinitionRegistryPostProcessor may register further bean definitions
 * which in turn define BeanFactoryPostProcessor instances.
 *
 * @author Juergen Hoeller
 * @since 3.0.1
 * @see org.springframework.context.annotation.ConfigurationClassPostProcessor
 */
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

    /**
     * Modify the application context's internal bean definition registry after its
     * standard initialization. All regular bean definitions will have been loaded,
     * but no beans will have been instantiated yet. This allows for adding further
     * bean definitions before the next post-processing phase kicks in.
     * @param registry the bean definition registry used by the application context
     * @throws org.springframework.beans.BeansException in case of errors
     */
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}


这个接口扩展了标准的BeanFactoryPostProcessor 接口，允许在普通的BeanFactoryPostProcessor接口实现类执行之前注册更多的BeanDefinition。特别地是，BeanDefinitionRegistryPostProcessor可以注册BeanFactoryPostProcessor的BeanDefinition。

postProcessBeanDefinitionRegistry方法可以修改在BeanDefinitionRegistry接口实现类中注册的任意BeanDefinition，也可以增加和删除BeanDefinition。原因是这个方法执行前所有常规的BeanDefinition已经被加载到BeanDefinitionRegistry接口实现类中，但还没有bean被实例化。

接口应用
我们仅仅需要写一个类实现接口，然后将这个类配置到spring的xml配置中。以下是我写的一个简单的实现类:

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;

public class DefaultBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    private Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        logger.info("BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法,在这里可以增加修改删除bean的定义");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        logger.info("BeanDefinitionRegistryPostProcessor的postProcessBeanFactory方法,在这里可以对beanFactory做一些操作");
    }
}

实际上，Mybatis中org.mybatis.spring.mapper.MapperScannerConfigurer就实现了该方法，在只有接口没有实现类的情况下找到接口方法与sql之间的联系从而生成BeanDefinition并注册。而Spring的org.springframework.context.annotation.ConfigurationClassPostProcessor也是用来将注解@Configuration中的相关生成bean的方法所对应的BeanDefinition进行注册。



Bean实例化中的扩展
前一小节关注的是BeanDefinition和BeanFactory，那么在bean的实例化过程中会调用一些特定的接口实现类，这些接口都有哪些？

InstantiationAwareBeanPostProcessor接口
/**
 * Subinterface of {@link BeanPostProcessor} that adds a before-instantiation callback,
 * and a callback after instantiation but before explicit properties are set or
 * autowiring occurs.
 *
 * <p>Typically used to suppress default instantiation for specific target beans,
 * for example to create proxies with special TargetSources (pooling targets,
 * lazily initializing targets, etc), or to implement additional injection strategies
 * such as field injection.
 *
 * <p><b>NOTE:</b> This interface is a special purpose interface, mainly for
 * internal use within the framework. It is recommended to implement the plain
 * {@link BeanPostProcessor} interface as far as possible, or to derive from
 * {@link InstantiationAwareBeanPostProcessorAdapter} in order to be shielded
 * from extensions to this interface.
 *
 * @author Juergen Hoeller
 * @author Rod Johnson
 * @since 1.2
 * @see org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#setCustomTargetSourceCreators
 * @see org.springframework.aop.framework.autoproxy.target.LazyInitTargetSourceCreator
 */
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

    /**
     * Apply this BeanPostProcessor <i>before the target bean gets instantiated</i>.
     * The returned bean object may be a proxy to use instead of the target bean,
     * effectively suppressing default instantiation of the target bean.
     * <p>If a non-null object is returned by this method, the bean creation process
     * will be short-circuited. The only further processing applied is the
     * {@link #postProcessAfterInitialization} callback from the configured
     * {@link BeanPostProcessor BeanPostProcessors}.
     * <p>This callback will only be applied to bean definitions with a bean class.
     * In particular, it will not be applied to beans with a "factory-method".
     * <p>Post-processors may implement the extended
     * {@link SmartInstantiationAwareBeanPostProcessor} interface in order
     * to predict the type of the bean object that they are going to return here.
     * @param beanClass the class of the bean to be instantiated
     * @param beanName the name of the bean
     * @return the bean object to expose instead of a default instance of the target bean,
     * or {@code null} to proceed with default instantiation
     * @throws org.springframework.beans.BeansException in case of errors
     * @see org.springframework.beans.factory.support.AbstractBeanDefinition#hasBeanClass
     * @see org.springframework.beans.factory.support.AbstractBeanDefinition#getFactoryMethodName
     */
    Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

    /**
     * Perform operations after the bean has been instantiated, via a constructor or factory method,
     * but before Spring property population (from explicit properties or autowiring) occurs.
     * <p>This is the ideal callback for performing custom field injection on the given bean
     * instance, right before Spring's autowiring kicks in.
     * @param bean the bean instance created, with properties not having been set yet
     * @param beanName the name of the bean
     * @return {@code true} if properties should be set on the bean; {@code false}
     * if property population should be skipped. Normal implementations should return {@code true}.
     * Returning {@code false} will also prevent any subsequent InstantiationAwareBeanPostProcessor
     * instances being invoked on this bean instance.
     * @throws org.springframework.beans.BeansException in case of errors
     */
    boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

    /**
     * Post-process the given property values before the factory applies them
     * to the given bean. Allows for checking whether all dependencies have been
     * satisfied, for example based on a "Required" annotation on bean property setters.
     * <p>Also allows for replacing the property values to apply, typically through
     * creating a new MutablePropertyValues instance based on the original PropertyValues,
     * adding or removing specific values.
     * @param pvs the property values that the factory is about to apply (never {@code null})
     * @param pds the relevant property descriptors for the target bean (with ignored
     * dependency types - which the factory handles specifically - already filtered out)
     * @param bean the bean instance created, but whose properties have not yet been set
     * @param beanName the name of the bean
     * @return the actual property values to apply to the given bean
     * (can be the passed-in PropertyValues instance), or {@code null}
     * to skip property population
     * @throws org.springframework.beans.BeansException in case of errors
     * @see org.springframework.beans.MutablePropertyValues
     */
    PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException;

}


这个接口是BeanPostProcessor的子接口，用于在实例化之后，但在设置显式属性或自动装配之前，设置实例化之前的回调函数。通常用于抑制特定目标bean的默认实例化，例如，创建具有特殊TargetSources（池化目标，延迟初始化目标等）的代理，或者实现其他注入策略，例如字段注入。注意：这个接口是一个专用接口，主要用于框架内的内部使用。 建议尽可能实现简单的BeanPostProcessor接口，或者从InstantiationAwareBeanPostProcessorAdapter派生，以便屏蔽此接口的扩展。

postProcessBeforeInstantiation方法，在目标bean实例化之前创建bean，如果在这里创建了bean，则不会走默认的实例化过程，通常用来创建代理。注意工厂方法生成的bean不会走这个方法。

postProcessAfterInstantiation方法，在目标bean实例化后，但是没有进行属性填充前执行的方法。

postProcessPropertyValues方法，在将给定属性值设置到到给定的bean后，对其进行后处理。 允许检查所有的依赖关系是否被满足，例如基于bean属性设置器上的“Required”注解。还允许替换要应用的属性值，通常通过创建基于原始PropertyValues的新MutablePropertyValues实例，添加或删除特定值。

接口应用
这个接口spring不建议用户直接实现，如果必须在这些扩展点应用自己的回调函数，spring建议继承InstantiationAwareBeanPostProcessorAdapter，重写相应的方法即可。

org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator，基于beanName创建代理，就是应用了这个接口，在生成bean前生成代理bean，从而替代默认的实例化。