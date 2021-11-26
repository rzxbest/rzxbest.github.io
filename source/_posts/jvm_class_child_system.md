---
title: JVM类加载子系统
date: 2021-11-25 14:59:29
tags: [JAVA,虚拟机]
categories: [JAVA]
---

## 类加载流程

1. 加载：从硬盘读取class文件

2. 验证：验证字节码准确性

3. 准备：类静态变量赋值，默认值基础类型0，false等等，对象类型null

4. 解析：将符号引用替换为直接引用

   1. 静态链接：main方法替换为指向数据所存内存区域的指针
   2. 动态链接：运行期间将符号引用替换为指向内存区域的指针

5. 初始化：类的静态变量初始化为制定的值，执行静态代码块

   

## 类加载器与双亲委派机制

- 启动类加载器：加载jre/lib下的核心类库
- 扩展类加载器：加载lib/ext 下的jar包
- 应用程序加载器：加载自己写的类
- 自定义类加载器：加载自定义路径的类

**打印 启动类加载器 打印出null？原因：c语言实现，java里找不到打印null**

### 自定义类加载器

1. 重写loadClass方法
2. 重写findClass方法


ClassLoader 抽象类 未实现findClass方法

1. ​	loadClass方法
   - 先从父加载器加载
   - 若父加载器加载不到 ,用bootstrap加载器加载（c语言实现，java调用native方法）
   - 若经过上述两步还加载不到则调用自己findClass方法
2. defineClass方法：将二进制字节码文件加载成class对象

### 双亲委派机制

![双亲委派模型](shuangqin.png)

#### 为什么使用双亲委派机制？

- 沙箱安全机制，例如自己写java.lang.String 不会被应用程序加载器（自定义类加载器）加载，防止核心类被篡改
- 避免类的重复加载

#### 如何打印程序加载类信息？

 -verbose:class

#### A a = null，请问是否加载字节码？

不加载字节码

#### 打破双亲委派？

tomcat类加载打破

- 一个容器部署多个应用程序，不同的程序依赖同一类库的版本不同
- 同类库相同可以共存
- tomcat容器有自己的依赖类库，不能与应用程序类库混淆
- 支持jsp的修改

![tomcat](tomcat.png)

注意：

- 每个war包都有一个webAppClassLoader加载
-    早期tomcat有common、server、 shared目录 分别对应CommonClassLoader,CatalinaClassLoader,SharedClassLoader

### 类加载器如何为parent赋值？

程序启动时调用jvm.dll加载启动类加载器，由启动类加载其他类加载器并对parent进行赋值





