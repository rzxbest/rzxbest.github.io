---
title: JAVA虚拟机
date: 2021-11-25 19:20:47
tags: [JAVA,JAVA虚拟机]
categories: [JAVA]

---

![image-20211126094454822](image-20211126094454822.png)

## 栈

- 每开启一个线程，就开辟一块栈内存区域
- 栈帧：遇到一方法就新建一块栈帧放入栈中
- 方法执行完，对应栈帧出栈销毁

#### 栈帧

- 局部变量表 ：存局部变量
- 操作数栈 ： 做运算
- 动态链接 ：方法入口地址 符号引用转为入口地址
- 方法出口

![image-20211126094254239](image-20211126094254239.png)

#### 反汇编查看字节码文件

javap -c ***.class

#### 单个栈空间大小设置参数

-Xss128k （默认1m）

栈空间满了会出现StackOverFlowError，调用链过长可能导致此位置，或者递归调用

## 堆

![image-20211126094600649](image-20211126094600649.png)

堆参数设置：

- -Xms  初始堆大小 例：-Xms256m
- -Xmx  最大堆大小 例：-Xms256m
- -Xmn  新生代大小     例：-Xmn1m

## 方法区(元空间)

主要存常量，静态变量，类元信息

jdk1.8类元数据放到了**本地内存**中，将**常量池**和**静态变量**放到了Java**堆**， jdk1.7 及以前存储堆中在持久代

方法区设置参数1.8

- -XX:MetaspaceSize:
- -XX:MetaspaceSize:

## 程序计数器

存储正在执行的语句的行号（地址），线程私有，当语句执行完成，字节码执行引擎对程序计数器进行修改

## GC

#### gc类型

- full gc 回收年轻代、老年代、方法区无引用对象
- minor gc 回收eden survivor区无引用对象

注意：gc回收过程中都会 stop the world (STW) 如果回收后仍没有空间放对象，OutOfMemoryError

minor gc 回收 eden区 from区 将存活对象移入to区

minor gc 回收eden区 to 区，将存活对象移入from区

#### 逃逸分析

##### jvm运行模式（java命令运行java类）

1. 解释模式：执行一行jvm字节码编译一行机器码执行 ，启动快执行慢

2. 编译模式：一次性编译成机器码一次性执行  启动慢 执行快

3. 混合模式：使用解释模式，热点代码使用编译模式

   

##### 逃逸分析参数

开启：-XX:+DoEscapeAnalysis

关闭：-XX:-DoEscapeAnalysis

jdk1.8默认开启

##### 逃逸分析说明

如果一个方法中的对象在方法结束时，这个对象就可以被回收，jvm会将这个对象分配栈帧里，如果栈空间不够，分配在堆里，栈帧弹出就回收，不需要等待内存满触发GC

#### 垃圾回收

1. 对象优先分配eden区

2. 大对象可以直接进入老年代，避免在年轻代来回复制，大对象复制比较耗时

3. minor gc后survivor区放不下的对象直接进入老年代

4. 动态年龄判断 ：年龄1+...年龄n > survivor区大小50% 那么年龄大于n的对象直接进入老年代

5. 老年代空间担保机制 -XX:-HandlePromotionFailure 1.8默认设置

   ![image-20211126100653589](image-20211126100653589.png)

#### 如何判断对象可以被回收？

- 引用计数器：计数为0 可以回收，存在问题就是循环依赖，A对象一属性依赖B B对象一属性依赖A
- 可达性分析算法：GCRoot根，遍历GCroot根

  - 线程栈局部变量
  - 静态变量
  - 本地方法栈变量


#### 如何判断一个类时无用类？

- 类的所有实例被回收
- 加载该类的classloader被回收
- 该类对应的java.lang.class对象任何地方没有被引用，Class..forName()

#### 当对象第一次被标记为垃圾对象，如何拯救对象变为正常对象？

重写finalize方法，重新建立引用关系，第二次标记回调用该对象的finalize方法

#### 垃圾回收算法

1. 标记清除：标记需回收的对象，然后一个一个清理
   - 效率较低
   - 空间利用率低，清除后产生不连续碎片
2. 复制算法：内存分为两块，分配对象使用其中一块，垃圾回收时，将存活对象复制到另一块，剩余一次性清理掉
   - 效率快
   - 空间利用率低
   - 一般来说年轻代使用复制算法
3. 标记整理算法：标记出垃圾对象，将存活对象向一端移动，然后清理端边界以外的内存
4. 分代收集算法：年轻代和老年代使用不同垃圾收集算法

#### 垃圾收集器

##### Serial收集器

|        |      使用参数       |     算法     |
| :----: | :-----------------: | :----------: |
| 年轻代 |  -XX:+UseSerialGC   |   复制算法   |
| 老年代 | -XX:+UseSerialOldGC | 标记整理算法 |

说明：单线程收集，收集时必须暂停应用程序线程 stw

优点：

- 简单高效

缺点：

- 对多核cpu来说 效率不高



##### ParNew收集器

说明：是Serial收集器的多线程版本

XX:+UseParNewGC，打开该开关后，使用ParNew(年轻代)+Serial Old(老年代)组合进行GC。另外，ParNew是CMS收集器的默认年轻代收集器。

老年代使用标记整理算法，年轻代是复制算法

ParNew用于垃圾回收的线程可用参数**-XX:ParallelGCThreads=n**进行配置,建议n与主机逻辑cpu数一致。

#### ParallelScavenge收集器

是内存大于2G，2个cpu下的默认收集器

说明：关注吞吐量，关注用户线程的停顿时间

所谓吞吐量，就是cpu用于执行用户代码时间与cpu消耗时间的比值

|        |       使用参数        |     算法     |
| :----: | :-------------------: | :----------: |
| 年轻代 |  -XX:+UseParallelGC   |   复制算法   |
| 老年代 | -XX:+UseParallelOldGC | 标记整理算法 |

#### CMS收集器

相关参数：

- -XX:+UseConcMarkSweepGC 启动cms
- -XX:ConcGCThreads 并发GC的线程数
- -XX:+UseCMSCompactAtFullCollection fullgc之后做压缩整理
- -XX:+CMSFullGCsBeforeCompaction 几次fullgc做一次压缩，默认是0
- -XX:CMSInitiatingOccupancyFraction 老年代使用达到该比例出发fullgc 默认是92，百分比
- -XX:+UseCMSInitiatingOccupancyOnly 只使用设定的回收阈值(- XX:CMSInitiatingOccupancyFraction设定的值) 如果没有指定这个值，只有第一次使用指定值，后续自动调整
- -XX:+CMSScavengeBeforeRemark  在fullgc之前先minor gc一次，目的在于减少老年代对年轻代的引用，降低cms gc 标记阶段开销，一般cms的gc耗时80%都在remark阶段

cms收集器步骤：

1. 初始标记：STW 记录gcRoot根直接引用的对象 速度很快
2. 并发标记：NO STW 全链路标记gcRoot ，耗时长
3. 重新标记：STW 修正并发标记应用程序运行导致标记产生变动的对象，比初始标记稍微耗时长一点
4. 并发清理：NO STW 清理

cms收集器的缺陷：

- 产生浮动垃圾，并发清理又产生垃圾

- cpu资源抢占

- 使用标记清除算法，产生碎片

- 在执行gc过程中，一边标记/回收，一边产生新的垃圾，可能导致再一次的fullgc ,此时会进入stw,使用serial收集器收集

  





