---
title: jm调优
date: 2022-03-08 09:00:00
tags: [JAVA虚拟机]
categories: [JAVA虚拟机]
top: 100
---
# jm调优

## 遇到了什么问题需要调优？
生产环境新上线了一个系统，莫名其妙奔溃，错误日志报异常OOM

## 类加载过程
加载 -> 验证 -> 准备 -> 解析 -> 初始化 -> 使用 -> 卸载

### 什么情况下会去加载一个类呢?
何时将“.class”字节码文件中加载这个类到JVM内存里来，代码中用到这个类的时候。


### 验证
来校验加载进来的“.class”文件中的内容，是否符合指定的jvm规范。

### 准备
类的类变量(static修饰的变量)分配内存空间，来一个默认的初始值，并非执行赋值

### 解析阶段
符号引用替换为直接引用

### 初始化
执行类变量的赋值语句，执行静态代码块

## 类加载器和双亲委派机制
### 类加载器
- 启动类加载器
Bootstrap ClassLoader，负责加载Java安装目录下的“lib”目录中的核心类库
- 扩展类加载器
Extension ClassLoader，负责加载Java安装目录“lib\ext”目录里面有一些类
- 应用程序类加载器
Application ClassLoader，负责去加载“ClassPath”环境变量所指定的路径中的类 理解为去加载自己写好的Java代码
- 自定义类加载器 除了上面那几种之外，还可以自定义类加载器，去根据自己的需求加载自己的类。
### 双亲委派机制
- JVM的类加载器是有亲子层级结构的，启动类加载器是最上层的，扩展类加载器在第二层，第三层是应用程序类加载器，最后一层是自定义类加载器。
- 双亲委派模型:先找父亲去加载，不行的话再由自己来加载。

## 内存划分
### 方法区（1.8之前） 元空间
将“.class”文件里加载进来的类
### 程序计数器
写好的Java代码会被翻译成字节码，对应各种字节码指令，JVM加载类信息到内存之后，会使用自己的字节码执行引擎执行字节码指令，执行时，JVM里就需要一个特殊的内存区域，那就是“程序计数器”，这个程序计数器就是用来记录当前执行的字节码指令的位置的
### Java虚拟机栈
- JVM必须有一块区域是来保存每个方法内的局部变量等数据的，这个区域就是Java虚拟机栈 
- 每个线程都有自己的Java虚拟机栈
- 如果线程执行了一个方法，就会对这个方法调用创建对应的一个栈帧，栈帧里就有这个方法的局部变量表 、操作数栈、动态链接、方法出口

### Java堆内存
存放代码中创建的各种对象的

## jvm
### jvm参数

- -Xms:Java堆内存的大小 、-Xmx:Java堆内存的最大大小 
- -Xms和-Xmx，分别用于设置Java堆内存的刚开始的大小，以及允许扩张到的最大大小。
- -Xmn:Java堆内存中的新生代大小，扣除新生代剩下的就是老年代的内存大小,默认不设置新生代大小时，大概老年代占2/3，新生代占1/3，eden和2个survivor 8:1:1  jvm参数"- XX:SurvivorRatio=8"
- -XX:PermSize:永久代大小、-XX:MaxPermSize:永久代最大大小
- jdk1.8 用这两个-XX:MetaspaceSize和-XX:MaxMetaspaceSize替换永久代大小
- -Xss:每个线程的栈内存大小

### 如何合理设置永久代大小?
一般你设置个几百MB，大体上都是够用的 因为里面主要就是存放一些类的信息
### 如何合理设置栈内存大小
一般默认就是比如512KB到1MB

### 如何确定对象是垃圾？可达性分析算法GCRoot根由那些对象组成？
对象无引用时会被确定为垃圾对象，方法的局部变量、类的静态变量都被看为GCRoot根
### 垃圾对象一定会被回收吗？
不一定，如果对象重写了Object类中的finialize()方法，并且在该方法建立起与静态变量的引用关系就可以不被回收
### 强引用、软引用、弱引用、虚引用
强引用：绝对不能回收的对象
软引用：有的对象可有可无，如果内存实在不够了，可以回收他。
弱引用：与没引用是类似的，如果发生垃圾回收，就会把这个对象回收掉
虚引用：暂时忽略，很少用。

### 垃圾回收算法
复制算法：内存区域划分为两块，对象就就会分配在其中一块内存空间，垃圾回收时，存活对象复制到另一块内存空间，同时原分配对象空间清除所有对象
缺点：内存使用率太低 优点：不会产生内存碎片，不浪费内存空间

复制算法做出如下优化：
- 把新生代内存区域划分为三块:1个Eden区，2个Survivor区，其中Eden区占80%内存空间，每一块Survivor区各占10%内存空间
- 刚开始对象都是分配在Eden区内的，如果Eden区快满了，此时就会触发垃圾回收，把Eden区中的存活对象都一次性转移到一块空着的Survivor区，接着Eden区就会被清空
- 然后再次分配新对象到Eden区里，此时，Eden区和一块Survivor区里是有对象的，其中Survivor区里放的是上一次Minor GC后存活的对象。如果下次再次Eden区满，那么再次触发Minor GC，Eden区和放着上一次Minor GC后存活对象的Survivor区内的存活对象，转移到另外一块Survivor区去。然后清空eden区和上一次的survivor区

### 对象何时从新生代转到老年代？
- 15次minor gc对象还存活。JVM参数“-XX:MaxTenuringThreshold”来设置，默认是15岁
- 动态对象年龄判断：假如说当前放对象的Survivor区域的一些对象大小和（按照年龄排正序）大于了这块Survivor区域的内存大小的50%，那么此时大于等于这批对象年龄的对象，就可以直接进入老年代。年龄1+年龄2+年龄n的多个年龄对象总和超过了Survivor区域的50%，此时就会把年龄n以上的对象都放入老年代。
- 大对象直接进入老年代。JVM参数，“-XX:PretenureSizeThreshold”，单位字节数，比如“1048576”字节，就是1MB。
- Minor GC后的对象太多无法放入Survivor区怎么办? 这个时候就必须得把这些对象直接转移到老年代去
- 老年代空间分配担保机制
    - 在执行任何一次Minor GC之前，JVM会先检查一下老年代可用的可用内存空间，是否大于新生代所有对象的总大小。避免Minor GC后老年代空间放不下存活对象
    - 如果老年代的内存大小是大于新生代所有对象的，此时就可以对新生代发起一次Minor GC了
    - 如果老年代的内存大小是小于新生代所有对象的，JVM“-XX:- HandlePromotionFailure”的参数是否设置,
        - 未设置 发生一次Full GC（对老年代进行垃圾回收，同时一般会对新生代进行垃圾回收）
        - 设置表示开启老年代空间担保，接着判断老年代的内存大小，是否大于之前每一次Minor GC后进入老年代的对象的平均大小，如果大于，则Minor GC,如果小于，FullGC（年轻代+老年代+永久代）
    - 如果要是Full GC过后，老年代还是没有足够的空间存放Minor GC过后的剩余存活对象，就会导致所谓的 “OOM”内存溢出

### 老年代采用的什么垃圾回收算法？
- 标记整理算法：首先标记出来老年代当前存活的对象，接着会让这些存活对象在内存里进行移动，把存活对象尽量都挪动到一边去
- 老年代的垃圾回收算法的速度至少比新生代的垃圾回收算法的速度慢10倍。
- 如果系统频繁出现老年代的Full GC垃圾回收，会导致系统性能被严重影响，出现频繁卡顿的情况。
### 垃圾收集器
- Serial和Serial Old垃圾回收器:分别用来回收新生代和老年代的垃圾对象，工作原理就是单线程运行，STW，一般几乎不用。
- ParNew和CMS垃圾回收器:ParNew现在一般用在新生代的，CMS是用在老年代的，都是多线程并发的机制，性能更好，一般是生产系统的标配组合。
- G1垃圾回收器:统一收集新生代和老年代，采用了更加优秀的算法和设计机制

### stw
直接停止Java系统的所有工作线程，让代码不再运行，让垃圾回收线程可以专心致志的进行垃圾回收的工作
    - 假设我们的Minor GC要运行100ms，那么可能就会导致系统直接停顿100ms不能处理任何请求
    - Full GC是最慢的，有的时候弄不好一次回收要进行几秒钟，甚至几十秒，有的极端场景几分钟都是有可能的。一旦频繁的Full GC，系统每一段时间就卡死个30秒吗?
    - 无论是新生代GC还是老年代GC，都尽量不要让频率过高，避免持续时间过长

### ParNew垃圾收集器
暂停系统程序的工作线程，禁止程序继续运行创建新的对象，用多个垃圾回收线程去进行垃圾回收，用复制算法回收
使用ParNew垃圾回收器 JVM参数 -XX:+UseParNewGC
ParNew垃圾回收器默认情况下的线程数量 垃圾回收线程的数量就是跟CPU的核数是一样。 JVM参数设置 - XX:ParallelGCThreads=n

### 服务器模式与客户端模式
- 启动系统的时候是可以区分服务器模式和客户端模式的，启动系统的时候加入“-server”就是服务器模式，如果加入“-cilent”就是客户端模式。
- 系统部署在比如4核8G的Linux服务器上那么就应该用服务器模式，如果你的系统是运行在比如Windows上的客户端程序，那么就应该是客户端模式。
- 服务器模式通常ParNew来进行垃圾回收，客户端模式使用采用Serial垃圾回收器，单CPU单线程垃圾回收

### CMS
CMS垃圾回收器采取的是垃圾回收线程和系统工作线程尽量同时执行的模式来处理的，使用标记-清理的算法
初始标记 ：stw 很快，仅仅标记出GC Root根
并发标记 ：对老年代所有对象进行GC Roots追踪，其实是最耗时的
重新标记 ：stw 很快,对在第二阶段中被系统程序运行变动过的少数对象进行标记
并发清理 ：很耗时

缺点： 
    - 会消耗CPU资源，在并发标记时，老年代存活对象多，追踪大量对象，耗时较高，并发清理，把垃圾对象从随机的内存位置清理掉，也比较耗时，CMS的垃圾回收线程是比较耗费CPU资源的。CMS默认启动的垃圾回收线程的数量是(CPU核数 + 3)/ 4。
    - 会产生浮动垃圾，并发清理同时随着系统运行让一些对象进入老年代，这种垃圾需要等到下一次GC才能回收掉


-XX:CMSInitiatingOccupancyFaction 用来设置老年代占用多少时触发CMS垃圾回收，JDK 1.6里面默认的值是 92%。
-XX:+UseCMSCompactAtFullCollection 默认开启 Full GC之后再次进行stw，碎片整理
-XX:CMSFullGCsBeforeCompaction 执行多少次Full GC之后执行一次内存碎片整理，默认是0，每次Full GC之后都会进行内存整理。

### 优化方案
- 评估Eden区进行Minor Gc 之后存活的对象大小，Suvivor区空间够不够，建议的是调整新生代和老年代的大小，让短期存活的对象在新生代就被垃圾回收
- 结合系统的运行模型，@Service、@Controller之类的注解那种需要长期存活的核心业务逻辑组件，降低“-XX:MaxTenuringThreshold”参数的值，避免对长期存活的对象在新生代Survivor区来回复制
- 大对象直接进入老年代，-XX:PretenureSizeThreshold=1M，避免对象在新生代Survivor区来回复制
一般启动参数：
-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M - XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=5 -XX:PretenureSizeThreshold=1M -XX:+UseParNewGC - XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFaction=92 -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0

### Concurrent Mode Failure
CMS在垃圾回收的时候，尤其是并发清理期间，系统程序是可以并发运行的，假设此时老年代空闲空间仅剩100MB，此时系统程序还在不停的创建对象，万一这个时候系统运行触发了某个条件，比如说有200MB对象要进入老年代，这个时候就会触发“Concurrent Mode Failure”问题，因为此时老年代没有足够内存来放这200MB对象，此时就会导致立马进入Stop the World，然后切换CMS为Serial Old收集器来单线程收集垃圾 

### G1 垃圾收集器（针对大内存机器通常建议采用G1垃圾回收器）
- G1垃圾回收器把Java堆内存拆分为多个大小相等的Region，G1会有新生代和老年代的概念，但是只是逻辑上的概念
- G1垃圾回收器可以设置一个垃圾回收的预期停顿时间
- G1通过把内存拆分为大量小Region，以及追踪每个Region中可以回收的对象大小和预估时间，在垃圾回收的时候，尽量把垃圾回收对系统造成的影响控制在指定的时间范围内，同时在有限的时间内尽量回收尽可能多的垃圾对象
- 使用“-XX:+UseG1GC”来指定使用G1垃圾回收器，会自动用堆大小除以2048得到每个region的大小，可以使用-XX:G1HeapRegionSize指定region的大小
G1中新生代（Eden、Survivor）、老年代的逻辑概念，-XX:G1NewSizePercent来设置新生代初始占比的，随着对象的不停创建，属于新生代的Region会不断增加，Eden和Survivor对应的Region也会不断增加。一旦新生代达到了设定的占据堆内存的最大大小60%，会触发新生代的GC，G1就会用复制算法来进行垃圾回收，进入一个“Stop the World”状态，然后把Eden对应的Region中的存活对象放入S1对应的Region中，接着回收掉Eden对应的Region中的垃圾对象   
- G1是可以设定目标GC停顿时间的，G1执行GC的时候最多可以让系统停顿多长时间，可以通过“-XX:MaxGCPauseMills”参数来设定，默认值是200ms。
- 年轻代何时进入老年代？跟之前几乎是一样的
    - 对象在新生代gc多次仍然存活 -XX:MaxTenuringThreshold
    - 年龄为1岁，2岁，3岁，n岁的对象的大小总和超过了Survivor的50%，n+1岁进入老年代
    - 在G1中，大对象的判定规则：一个大对象超过了一个Region大小的50%，会被放入大对象专门的Region
- G1有“-XX:InitiatingHeapOccupancyPercent”，默认值是45%：老年代占据了堆内存的45%的Region的时候，尝试触发一个新生代+老年代一起回收的混合回收阶段。
- G1垃圾收集几个阶段（混合回收可以执行多次）
    - 初始标记 stw 标记一下GC Roots直接能引用的对象 很快
    - 并发标记，允许系统程序的运行，同时进行GC Roots追踪，从GC Roots开始追踪所有的存活，同时记录并发标记系统运行导致哪些对象的改变
    - 最终标记阶段，stw 对并发标记阶段记录的对象进行标记，最终标记一下有哪些存活对象，有哪些是垃圾对象
    - 混合回收，从新生代和老年代里都回收一些Region，先停止工作，执行一次混合回收回收掉一些Region，接着恢复系统运行，然后再次停止系统运行，再执行一次混合回收回收掉一些Region。-XX:G1MixedGCCountTarget  混合回收的过程中，最后一个阶段执行几次混合 回收，默认值是8次

### G1调优
- -Xms4096M -Xmx4096M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -XX:+UseG1GC
- -XX:G1NewSizePercent参数是用来设置新生代初始占比的，不用设置，维持默认值为5%即可
- -XX:G1MaxNewSizePercent参数是用来设置新生代最大占比的，也不用设置，维持默认值为60%即可
- -XX:MaxGCPauseMills 默认值是200毫秒 垃圾回收时最大停顿时间 
- G1里是很动态灵活的，根据你设定的gc停顿时间给你的新生代不停分配更多Region，到一定程度，就会触发新生代gc，保证新生代gc的时候导致的系统停顿时间在预设范围内。
- 新生代的GC优化：合理设置“-XX:MaxGCPauseMills”参数，设置小了，Gc频繁，设置大了，GC停顿时间长
- mixed gc优化：默认老年代占据了堆内存的45%的Region的时候，发生Gc,如果-XX:MaxGCPauseMills设置过大，就会导致新生代GC后更多的对象进入老年代，加速mixed gc 发生

### 日处理上亿数据案例
- 分布式系统，4核心8G，JVM给了4G，新生代和老年代各1.5G（1.2G Eden区Survivor区150M）,每台机器每分钟计算100条数据，每秒取10000条数据，需要10秒计算完成
    - 预估每条数据20个字段，每个字段4B，大约1条数据100B,10000条数据1M，系统运行产生其他对象*10，10M
    - 每次运行产生10M，一分钟产生1G，大概1分钟就要产生一次年轻代垃圾
    - 每次年轻代垃圾回收，会有大概17～20次还在计算中，这些数据都是存活的对象，占用空间170M～200M 
    - 每次年轻代垃圾回收，Survivor区无法放入存活的对象，放老年代，在7～9次年轻代垃圾回收后，老年代就满了，触发Fullgc 耗时
    - 将年轻代分配2G eden1.6G surviro 200m

### BI系统案例
- 10台机器来部署BI系统，4核8G的配置，堆内存中的新生代分配的内存都在1.5G左右，Eden区大概也就1G左右的空间，几万商家作为你的系统用户，很可能同一时间打开那个实时报表的商家就有几千个，然后每个商家打开实时报表之后，前端页面都会每隔几秒钟发送请求到后台来加载最新数据BI系统部署的每台机器每秒的请求会达到几百个，假设5000个商家同时在线，每秒刷新一次，每秒5000个请求，每台机器分摊500个请求
    - 预算每个请求大概需要加载出来10kb的数据进行计算，每秒需要加载出来5MB的数据到内存中进行计算
    - 200s 就会触发年轻代GC，存活的对象估计几十m
    - 估计50次年轻代gc 会发生老年代Gc
- 每秒十万并发，那么就要部署上百台机器来，扩大机器内存，使用G1收集器，采用10台机器

## 实战
### 打印gc日志
- -XX:+PrintGCDetils:打印详细的gc日志 
- -XX:+PrintGCTimeStamps:这个参数可以打印出来每次GC发生的时间 
- -Xloggc:gc.log:这个参数可以设置将gc日志写入一个磁盘文件


### Jvm 命令
- jps 查看jvm进程ID
- jstat -gc jvm进程ID 查看jvm进程内存使用情况
    - S0C:From Survivor区的大小
    - S1C:To Survivor区的大小
    - S0U:From Survivor区当前使用的内存大小 
    - S1U:To Survivor区当前使用的内存大小 
    - EC:Eden区的大小
    - EU:Eden区当前使用的内存大小
    - OC:老年代的大小
    - OU:老年代当前使用的内存大小 
    - MC:方法区(永久代、元数据区)的大小 
    - MU:方法区(永久代、元数据区)的当前使用的内存大小 
    - YGC:Young GC次数 
    - YGCT:Young GC的耗时 
    - FGC:Full GC次数
    - FGCT:Full GC的耗时
    - GCT:所有GC的总耗时
- jstat -gc jvm进程ID 1000 10  1秒输出一次 输出10次 用于观察环境上内存变化
- jmap -heap jvm进程ID 查看堆内存信息
- jmap -histo:live jvm进程ID 查看内存对象信息
- jmap -dump:live,file=/filename.hprof jvm进程ID  dump出内存信息
- jhat -port 7000 filename.hprof 将dump文件加载出来以网页方式呈现

## 调优
### 调优方案
尽量让每次Young GC后的存活对象小于Survivor区域的50%，都留存在年轻代里。
尽量别让对象进入老年代。
尽量减少Full GC的频率，避免频繁Full GC对JVM性能的影响。
### 社交app，QPS 十万
- 老年代频繁Gc 压缩碎片，避免每次GC间隔越来越短 -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5
### 垂直电商app 每小时总会有卡顿
- 垂直电商APP的各个系统通过jstat分析JVM GC之后发现，基本上高峰期的时候，Full GC每小时都会发生好几次
- 通用JVM参数 -Xms4096M -Xmx4096M -Xmn3072M -Xss1M -XX:PermSize=256M -XX:MaxPermSize=256M -XX:+UseParNewGC - XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFaction=92 -XX:+UseCMSCompactAtFullCollection - XX:CMSFullGCsBeforeCompaction=0
- 优化fullgc的效率
    - -XX:+CMSParallelInitialMarkEnabled CMS垃圾回收器的“初始标记”阶段开启多线程并发执行
    - -XX:+CMSScavengeBeforeRemark 会在CMS的重新标记阶段之前，尽量执行一次Young GC。

### 频繁的fullgc 观察后发现元空间的满了
- 代码里写了大量反射代码，JVM会动态的去生成一些类放入Metaspace区域里的，JVM自己创建的奇怪的类Class对象是SoftReference
- 软引用，只有内存不够才会回收，怎么判断要不要回收？clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB，clock - timestamp代表了一个软引用对象他有多久没被访问过了，freespace代表JVM中的空闲内存空间，SoftRefLRUPolicyMSPerMB代表每一MB空闲内存空间可以允许SoftReference对象存活多久
- SoftRefLRUPolicyMSPerMB 设置为0 导致
- 在有大量反射代码的场景下 -XX:SoftRefLRUPolicyMSPerMB=0 可以设置个1000，2000，3000，或者5000毫秒 默认1000

### 线上fullgc频繁
- 机器配置:2核4G，JVM堆内存大小:2G，系统运行时间:6天，系统运行6天内发生的Full GC次数和耗时:250次，70多秒，系统运行6天内发生的Young GC次数和耗时:2.6万次，1400秒，每天会发生4000多次Young GC，每分钟会发生3次，每次Young GC在50毫秒左右
- 未优化前 -Xms1536M -Xmx1536M -Xmn512M -Xss256K -XX:SurvivorRatio=5 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC - XX:CMSInitiatingOccupancyFraction=68 -XX:+CMSParallelRemarkEnabled -XX:+UseCMSInitiatingOccupancyOnly - XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC
- 通过jstat的观察，每次Young GC过后升入老年代里的对象很少 ， 每次Young GC过后大概就存活几十MB而已，那么Survivor区域因为就70MB，所以经常会触发动态年龄判断规则，导致偶尔一次Young GC过后有几十MB对象进入老年代
- 大对象 jstat工具观察系统，发现老年代里突然进入了几百MB的大对象,就是特殊场景全表查了数据库，量比较大
- 优化方案
    - 去除查大数据量的bug
    - 年轻代变大，避免触发动态年龄判断，部分垃圾对象进入老年代
    - -Xms1536M -Xmx1536M -Xmn1024M -Xss256K -XX:SurvivorRatio=5 -XX:PermSize=256M -XX:MaxPermSize=256M - XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=92 - XX:+CMSParallelRemarkEnabled -XX:+UseCMSInitiatingOccupancyOnly -XX:+PrintGCDetails -XX:+PrintGCTimeStamps - XX:+PrintHeapAtGC