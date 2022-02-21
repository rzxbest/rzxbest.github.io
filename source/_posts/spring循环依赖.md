---
title: Spring循环依赖
date: 2022-02-18 14:59:29
tags: [JAVA,Spring]
categories: [Spring]
---

# Spring循环依赖

## 循环依赖图
![spring循环依赖](spring循环依赖问题.png)

## aop创建proxy对象时机
- 常规一点的就是在BeanFactory.initializeBean()方法里面调用初始化方法之后，调用applyBeanPostProcessorsAfterInitialization() --> AbstractAutoProxyCreator.wrapIfNecessary()生成proxy对象。
- 还有就是在存在循环依赖的时候，在doCreateBean() --> addSingletonFactory(beanName, () --> getEarlyBeanReference(beanName, mbd, bean)); --> 然后在getSingleton()的时候从三级缓存中取对象的时候，会调用getEarlyBeanReference() --> AbstractAutoProxyCreator.wrapIfNecessary()生成proxy对象。

## aop创建案例  
- 第一种情况，Student是代理对象，但是getBean("teacher")。
1. 创建Teacher bean
2. 开始populateBean("student")
3. 开始创建Student的bean实例
4. 在创建完bean实例的之后也开始populateBean("teacher")
5. 开始创建Teacher的bean实例 
6. 这时候doGetBean("teacher")-->getSingleton("teacher")的时候会从三级缓存中获取bean对象，直接返回。
7. Student继续下面步骤，即initializeBean()-->applyBeanPostProcessorsAfterInitialization()开始为Student创建Proxy对象(因为Teacher是正常的singleton bean)
8. 再返回到Teacher的创建，Teacher设置的就是Student的Proxy对象
9. 然后Teacher继续调用initializeBean()完成初始化。

- 第二种情况，Student是代理对象，同时getBean("student")。
1. 这种情况下就先开始创建Student对象的bean，
2. 同样的开始populateBean("teacher")，
3. 开始创建Teacher的bean，
4. 创建完Teacher bean之后，接着开始populateBean("student")，
5. 循环依赖开始创建student的bean， 
6. 那么这个时候这时候doGetBean("student")-->getSingleton("student")的时候会从三级缓存中获取bean对象，
7. 跟第一种情况不一样的是，Student是需要被AOP代理的，那么这时候从三级缓存中取出的是getEarlyBeanReference()方法返回的代理对象Proxy，
8. 然后将这个proxy对象放到二级缓存earlySingletonObjects中。
9. 后面Teacher populateBean("student")就拿到了Student的代理Proxy，完成Teacher的initializeBean()完成初始化， 
10. 然后回到创建student的bean创建，populateBean("teacher")设置好Teacher之后，
11. 开始完成student的initializeBean()初始化，
12. 这里applyBeanPostProcessorsAfterInitialization()-->wrapIfNecessary()不会再创建代理对象，
13. 因为缓存earlyProxyReferences里面已经存在student的proxy对象，这也就导致exposedObject = initializeBean(beanName, exposedObject, mbd)这个exposedObject不是代理对象proxy，而是真实对象，如果这样返回就有问题了，所以这里二级缓存的作用就体现出来了，在initializeBean()之后会再次调用getSingleton()去检查二级缓存，此时拿到earlySingletonReference是student对象的proxy对象，那么此时就将exposedObject从真实对象换成proxy代理对象，然后返回。
```
if (earlySingletonExposure) {
    //再次检查二级缓存的值
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
         if (exposedObject == bean) {
        //将真实对象转换成代理Proxy对象
        exposedObject = earlySingletonReference;
        }
        。。。省略代码
    }
}
```
## processon地址
https://www.processon.com/diagraming/620f41cbe0b34d5aaa947aec






