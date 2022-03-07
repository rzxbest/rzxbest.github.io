---
title: 学MQ所得
tags: [消息队列]
categories: [消息队列]
top: 30
date: 2022-02-28 22:22:22
---

### 梳理出来自己负责的系统的核心业务流程，核心功能模块，跟其他系统是如何交互的，数据是如何存储，当前已经使用了哪些中间件技术。



### 系统与系统之间的耦合性体现
来思考一个问题，假设促销系统现在有一个接口，专门是让你调用了以后派发优惠券的，现在这个接口接收的参数有 5个，你要是调用这个接口，就必须给他传递5个参数过去，这个是没的说的。
现在问题来了，负责促销系统的工程师某一天突然有一个新的想法，他希望改一改这个接口，在接口调用的时候需要传递7个参数！
一旦他的这个新接口上线了，你还是给他传5个参数，那么他那里就会报错，这个派发优惠券的行为就会失败！
那在这样的一个情况下应该怎么办？
    很简单，你作为订单系统的负责人，必须要配合促销系统去修改代码，既然他要7个参数，那么你就必须得在代码里调用他的接口的时候传递7个参数。
并且你还得配合他的新接口去进行测试以及部署上线，你必须得围绕着他转，配合他。
在这种情况下，就说明你的订单系统跟促销系统是强耦合的

### 订单系统存在的问题

![img.png](aimg.png)

### 消息中间件
异步化提升性能，降低系统耦合，流量削峰

- 解耦异步
    引入MQ之前,a系统调用b系统
    ![img_2.png](aimg_2.png)
    引入MQ之后
    ![img_3.png](aimg_3.png)
- 削峰
    MQ这个技术抗高并发的能力远远高于数据库，同样的机器配置下，如果数据库可以抗每秒6000请求，MQ至少可以抗每秒几万请求。
    ![img_1.png](aimg_1.png)
    MQ进行流量削峰的效果，系统A发送过来的每秒1万请求是一个流量洪峰，然后MQ直接给扛下来了，都存储自己本地磁盘，这个过程就是流量削峰的过程，瞬间把一个洪峰给削下来了，让系统B后续慢慢获取消息来处理。

### kafka rabbitmq rocketmq
- kafka优点
    Kafka的吞吐量几乎是行业里最优秀的，在常规的机器配置下，一台机器 可以达到每秒十几万的QPS
    Kafka性能也很高，发送消息给Kafka都是毫秒级的性能。
    可用性也很高，Kafka是可以支持集群部署的，其中部分机器宕机是可以继续运行的。
- kafka缺点
    丢数据方面的问题，Kafka收到消息之后会写入一个磁盘缓冲区里，并没有直接落地到物理磁盘上去，要是机器本身故障，可能会导致磁盘缓冲区里的数据丢失。
    功能非常的单一，主要是支持发送消息给他，然后从里面消费消息
- kafka使用场景
    Kafka用在用户行为日志的采集和传输上，比如大数据团队要收集APP上用户的一些行为日志，这种日志就是用Kafka来收集和传输的。
    因为那种日志适当丢失数据是没有关系的，而且一般量特别大，要求吞吐量要高，一般就是收发消息，不需要太多的高级功能，所以
Kafka是非常适合这种场景的。

- RabbitMQ优点
    保证数据不丢失
    保证高可用性，集群部署的时候部分机器宕机可以继续运行
    支持部分高级功 能，比如说死信队列，消息重试之类的
- RabbitMQ缺点
    RabbitMQ的吞吐量是比较低的，一般就是每秒几万的级别
    集群扩展的时候（也就是加机器部署），还比较麻烦。
    开发语言是erlang，国内很少有精通erlang语言的工程师，没办法去阅读他的源代码

- RocketMQ 优点
    RocketMQ是阿里开源的消息中间件，几乎同时解决了Kafka和RabbitMQ的缺陷。
    RocketMQ的吞吐量很高，单机可以达到10万QPS以上
    保证高可用性，性能很高，而且支持通过配置保证数据绝对不丢失，部署大规模的集群
    支持各种高级的功能，比如说延迟消息、事务消息、消息回溯、死信队列、消息积压
    RocketMQ是基于Java开发的，符合国内大多数公司的技术栈，可以阅读源码或者修改源码。
  
- RocketMQ 缺点
    RocketMQ的官方文档相对简单，但是Kafka和RabbitMQ的官方文档就非常的全面和详细，这可能是RocketMQ目前唯一的缺点。

- 活跃的社区和广泛的运用
    基本上Kafka、RabbitMQ和RocketMQ的社区都还算活跃，更新频率都还可以，而且基本运用都非常的广泛。
    目前Kafka几乎是国内大数据领域日志采集传输的标准,RabbitMQ在各种中小公司里运用极为广泛,RocketMQ也是开始在一些大公司和其他公司里快速推行中。
  
### RocketMQ 相关问题
- RocketMQ是如何集群化部署来承载高并发访问的？
    假设RocketMQ部署在一台机器上，即使这台机器配置很高，一般来说一台机器也就是支撑10万+的并发访问。 RocketMQ是可以集群化部署的，可以部署在多台机器上，假设每台机器都能抗10万并发，然后你只要让几十万请求分散到多 台机器上就可以了，让每台机器承受的QPS不超过10万不就行了
- 如果RocketMQ中要存储海量消息，如何实现分布式存储架构？
    MQ会收到大量的消息，这些消息并不是立马就会被所有的消费方获取过去消费的，所以一般MQ都得把消息在自己本地磁盘存储起来，然后等待消费方获取消息去处理。
- RocketMQ是如何分布式存储海量消息的呢？
    每台机器上部署的RocketMQ进程一般称之为Broker，每个Broker都会收到不同的消息，然后就会把这批消息存储在自己本地的磁盘文件里
- 高可用保障:万一Broker宕机了怎么办？
    Broker主从架构以及多副本策略
    ![img.png](bimg.png)
    Master Broker收到消息后会同步给Slave Broker,这个时候如果任何一个Master Broker出现故障，还有一个Slave Broker上有一份数据副本，可以保证数据不丢失，还能继续对外提供服务，保证了MQ的可靠性和高可用性
- 数据路由:怎么知道访问哪个Broker？
    有一个NameServer的概念，他也是独立部署在几台机器上的，然后所有的Broker都会把自己注册到NameServer上去，NameServer不就知道集群里有哪些Broker了？
  
### RocketMQ NameServer设计原理
- NameServer到底可以部署几台机器？
    NameServer支持部署多台机器的,NameServer可以集群化部署的，
    那为什么NameServer要集群化部署？ 最主要的一个原因，就是高可用性。
    NameServer是集群里非常关键的一个角色，管理Broker信息，别人都要通过他才知道跟哪个Broker通信。
    NameServer多机器部署，集群，高可用，保证任何一台机器宕机,其他机器上的NameServer可以继续对外提供服务 
- Broker在启动的时候是把自己的信息注册到哪个NameServer上去的？
    每个Broker启动都得向所有的NameServer进行注册
    比如一共有10台Broker机器，2个NameServer机器，然后其中5台Broker会把自己的信息注册到1个NameServer上去，另外5台Broker会把自己的信息注册到另外1个NameServer上去。这样搞有一个最大的问题，如果1台NameServer上有5个Broker的信息，另外1个NameServer上有另外5个Broker的信息，那么此时 任何一个NameServer宕机了，不就导致5个Broker的信息就没了吗
- 扮演生产者和消费者的系统们，如何从NameServer那儿获取到集群的Broker信息呢？
  知道集群里有哪些Broker，根据一定的算法挑选一个Broker去发送消息或者获取消息
  第一种办法是这样，NameServer那儿会主动发送请求给所有的系统，告诉他们Broker信息。
    NameServer无法知道要推送Broker信息给哪些系统
  第二种办法是这样的，每个系统自己每隔一段时间，定时发送请求到NameServer去拉取最新的集群Broker信息。
    RocketMQ中的生产者和消费者就是这样，自己主动去NameServer拉取Broker信息
- 如果Broker挂了，NameServer是怎么感知到的？
    Broker跟NameServer之间的心跳机制，Broker会每隔30s给所有的NameServer发送心跳，告诉每个NameServer自己目前还活着。
    NameServer会每隔10s运行一个任务，去检查一下各个Broker的最近一次心跳时间，如果某个Broker超过120s都没发送心跳了，则认为这个Broker挂掉了
- Broker挂了，系统是怎么感知到的？
    可能某个系统就会发送消息到那个已经挂掉的Broker上去，此时是绝对不可能成功发送消息的
    可以考虑不发送消息到那台Broker，改成发到其他Broker上去。
    你必须要发送消息给那台Broker，Slave机器是一个备份，可以继续使用，可以考虑Slave进行通信
    系统又会重新从NameServer拉取最新的路由信息了，此时就会知道有一个Broker已经宕机了。
![img.png](cimg.png)
  
### RocketMQ Broker原理分析
- Master Broker是如何将消息同步给Slave Broker的？
    Master Broker主动推送给Slave Broker？还是Slave Broker发送请求到Master Broker去拉取？
    RocketMQ的Master-Slave模式采取的是Slave Broker不停的发送请求到Master Broker去拉取消息
    RocketMQ自身的Master-Slave模式采取的是Pull模式拉取消息

- RocketMQ 实现读写分离了吗？
    Master Broker主要是接收系统的消息写入，然后会同步给Slave Broker
    作为消费者的系统在获取消息的时候，是从Master Broker获取的？还是从Slave Broker获取的？
    有可能从Master Broker获取消息，也有可能从Slave Broker获取消息
    Master Broker在返回消息给消费者系统的时候，会根据当时Master Broker的负载情况和Slave Broker的同步情况，向消费者系 统建议下一次拉取消息的时候是从Master Broker拉取还是从Slave Broker拉取。
    举个例子，Master Broker负载很重, 所以此时Master Broker就会建议你从Slave Broker去拉取消息。
    举另外一个例子，本身这个时候Master Broker上都已经写入了100万条数据了，结果Slave Broke同步的特别慢， 才同步了96万条数据，落后了整整4万条消息的同步，这个时候你作为消费者系统可能都获取到96万条数据了，那么下次还是只能从Master Broker去拉取消息。 
- 如果Slave Broke挂掉了有什么影响？
    有一点影响，但是影响不太大
    消息写入全部是发送到Master Broker的，消息获取也可以走Master Broker，只不过有一些消息获取可能是从Slave Broker 去走的。所以如果Slave Broker挂了，那么此时无论消息写入还是消息拉取，还是可以继续从Master Broke去走，对整体运行不影响。
    只不过少了Slave Broker，会导致所有读写压力都集中在Master Broker上。 
- 如果Master Broker挂掉了该怎么办？
    对消息的写入和获取都有一定的影响了。但是其实本质上而言，Slave Broker也是跟Master Broker一样有一份数据在的，
    只不过Slave Broker上的数据可能有部分没来得及从Master Broker同步。
    此时RocketMQ可以实现直接自动将Slave Broker切换为Master Broker吗？不能
    Master-Slave模式不是彻底的高可用模式，他没法实现自动把Slave切换为Master
- 基于Dledger实现RocketMQ高可用自动切换
  RocketMQ 4.5之后，RocketMQ支持了一种基于Raft协议实现的一个机制，叫做Dledger,,Dledger融入RocketMQ之后，可以让一个Master Broker对应多个Slave Broker，也就是说一份数据可以有多份副本，比如一个Master Broker对应两个Slave Broker。
  此时一旦Master Broker宕机了，就可以在多个副本，也就是多个Slave中，通过Dledger技术和Raft协议算法进行leader选举，直接将一个Slave Broker选举为新的Master Broker，然后这个新的Master Broker就可以对外提供服务了。整个过程也许只要10秒或者几十秒的时间就可以完成，这样的话，就可以实现Master Broker挂掉之后，自动从多个Slave Broker中选举出来一个新的Master Broker，继续对外服务，一切都是自动的。
![img.png](dimg.png)

### 高可用部署方案
- NameServer集群化部署，保证高可用，建议3台机器
- 基于Dledger的Broker主从架构
    - Dledger技术是要求至少得是一个Master带两个Slave，这样有三个Broke组成一个Group，也就是作为一个分组来运行。一旦 Master宕机，他就可以从剩余的两个Slave中选举出来一个新的Master对外提供服务
- Broker是如何跟NameServer进行通信的?
    - Broker会跟每个NameServer都建立一个TCP长连接，然后定时通过TCP长连接发送心跳请求过去，30s心跳一次，120秒检查一次是否一直没有心跳包
- 使用MQ的系统都要多机器集群部署
    - 生产者 消费者
- MQ的核心数据模型:Topic到底是什么?
    - Topic不能直译，表达的意思就是一个数据集合的意思，不同类型的数据你得放不同的Topic
- Topic作为一个数据集合是怎么在Broker集群里存储的?
    - 分布式存储 多个master
    - 创建Topic的时候指定数据分散存储在多台Broker机器
     ![img.png](fimg.png)
- 生产者系统是如何将消息发送给Broker的?
    - 在发送消息之前，得先有一个Topic，然后在发送消息的时候你得指定你要发送到哪个Topic里面去。
    - 生产者跟nameserver建立长连接，拉取路由信息找到要投递消息的Topic分布在哪几台Broker上，根据负载均衡算法，从里面选择一台Broke机器出来
    - 选择一台Broker，跟这个Broker建立一个TCP长连接，通过长连接向Broker发送消息
- 消费者是如何从Broker上拉取消息的?
    - 消费者跟nameserver建立长连接，拉取路由信息,接着找到自己要获取消息的 Topic在哪几台Broker上，就可以跟Broker建立长连接，从里面拉取消息

### rocket mq可视化界面管理工具
- git clone https://github.com/apache/rocketmq-externals.git 
- 进入rocketmq-console的目录
    cd rocketmq-externals/rocketmq-console 
- 执行以下命令对rocketmq-cosole进行打包，把他做成一个jar包: 
    mvn package -DskipTests
- 进入target目录下，可以看到一个jar包，接着执行下面的命令启动工作台:
    java -jar rocketmq-console-ng-1.0.1.jar --server.port=8080 --rocketmq.config.namesrvAddr=127.0.0.1:9876

### 中间件系统，对os内核参数调整

vm.overcommit_memory 有三个值可以选择，0、1、2
- 0，在中间件系统申请内存的时候，os内核会检查可用内存是否足够，如果足够的话就分配内存给你，如果感觉剩余内存不是太够了，干脆就拒绝你的申请，导致你申请内存失败，进而导致中间件系统异常出错。
- 1，把所有可用的物理内存都允许分配给你，只要有内存就给你来用，可以避免申请内存失败的问题。
因此一般需要将这个参数的值调整为1
比如我们曾经线上环境部署的Redis就因为这个参数是0，导致在save数据快照到磁盘文件的时候，需要申请大内存的时候被拒绝了，进而导致了异常报错。
用如下命令修改:echo 'vm.overcommit_memory=1' >> /etc/sysctl.conf。

vm.max_map_count 默认值是65536
- 会影响中间件系统可以开启的线程的数量
- 默认值有时候是不够的，比如我们大数据团队的生产环境部署的Kafka集群曾经有一次就报出过这个异常，说无法开启足够多的线程，直接导致Kafka宕机了。
建议可以把这个参数调大10倍，比如655360这样的值，保证中间件可以开启足够多的线程。
用如下命令修改:echo 'vm.max_map_count=655360' >> /etc/sysctl.conf。

vm.swappiness
用来控制进程的swap行为，os会把一部分磁盘空间作为swap区域，有的进程现在可能不是太活跃，就会被操作系统把进程调整为睡眠状态，把进程中的数据放入磁盘上的swap区域，然后让这个进程把原来占用的内存空间腾出来，交给其他活跃运行的进程来使用。
- 如果这个参数的值设置为0，意思就是尽量别把任何一个进程放到磁盘swap区域去，尽量大家都用物理内存。
- 如果这个参数的值是100，那么意思就是尽量把一些进程给放到磁盘swap区域去，内存腾出来给活跃的进程使用。 
- 默认这个参数的值是60，有点偏高了，可能会导致我们的中间件运行不活跃的时候被迫腾出内存空间然后放磁盘swap区域去。
- 生产环境建议把这个参数调整小一些，比如设置为10，尽量用物理内存，别放磁盘swap区域去。
 用如下命令修改:echo 'vm.swappiness=10' >> /etc/sysctl.conf。

ulimit
用来控制linux上的最大文件链接数的，默认值可能是1024，一般肯定是不够的，因为你在大量频繁的读写磁盘文件的时候，或者是进行网络通信的时候，都会跟这个参数有关系
- 对于一个中间件系统而言肯定是不能使用默认值的，如果你采用默认值，很可能在线上会出现如下错误:error: too many open files。
因此通常建议用如下命令修改这个值:echo 'ulimit -n 1000000' >> /etc/profile。

### 中间件系统，对jvm参数调整
- -server:服务器模式启动
- -Xms8g -Xmx8g -Xmn4g:默认的堆大小是8g内存，新生代是4g内存
- -XX:+UseG1GC -XX:G1HeapRegionSize=16m:用了G1垃圾回收器来做分代回收，对新生代和老年代都是用G1来回收，G1的region大小设置为了16m，这个因-为机器内存比较多，所以region大小可以调大一些给到16m，不然用2m的region，会导致致region数量过多的
- -XX:G1ReservePercent=25:G1管理的老年代里预留25%的空闲内存，保证新生代对象晋升到老年代的时候有足够空间，避免老年代内存都满了，新生代有对象要进入老年代没有充足内存了，默认值是10%，偏少，调大
- -XX:InitiatingHeapOccupancyPercent=30:当堆内存的使用率达到30%之后就会自动启动G1的并发垃圾回收，开始尝试回收一些垃圾对象
默认值是45%，调低，提高了GC的频率，避免了垃圾对象过多，一次垃圾回收耗时过长的问题
- -XX:SoftRefLRUPolicyMSPerMB=0:建议这个参数不要设置为0，避免频繁回收一些软引用的Class对象，调整为比如1000
- -verbose:gc -Xloggc:/dev/shm/mq_gc_%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps - XX:+PrintGCApplicationStoppedTime -XX:+PrintAdaptiveSizePolicy -XX:+UseGCLogFileRotation - XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m:控制GC日志打印输出的，gc日志文件的地址，打印哪些详细信息，每个gc日志文件的大小是30m，最多保留5个gc日志文件。
- -XX:-OmitStackTraceInFastThrow:有时候JVM会抛弃一些异常堆栈信息，这个参数设置之后，禁用这个特性，要把完整的异常堆栈信息打印出来
- -XX:+AlwaysPreTouch:刚开始指定JVM用多少内存，不会真正分配给他，会在实际需要使用的时候再分配给他，使用这个参数之后，就是强制让JVM启动的时候直接分配我们指定的内存，不要等到使用内存的时候再分配
- -XX:MaxDirectMemorySize=15g:RocketMQ里大量用了NIO中的direct buffer，限定了direct buffer最多申请多少， 如果你机器内存比较大，可以适当调大这个值
- -XX:-UseLargePages -XX:-UseBiasedLocking:这两个参数的意思是禁用大内存页和偏向锁
RocketMQ默认的JVM参数是采用了G1垃圾回收器，默认堆内存大小是8G 可以根据机器内存来调整，增大一些也是没有问题的，然后就是一些G1的垃圾回收的行为参数做了调整，
### 对RocketMQ核心参数进行调整
sendMessageThreadPoolNums=16 RocketMQ内部用来发送消息的线程池的线程数量，默认是16
- 根据你的机器的CPU核数进行适当增加，比如机器CPU是24核的，可以增加这个线程数量到24或者30

### 压测
在RocketMQ的TPS和机器的资源使用率和负载之间取得一个平衡。
例如：两个Producer不停的往RocketMQ集群发送消息，每个Producer所在机器启动了80个线程，相当于每台机器有80个线程并发的往RocketMQ集群写入消息。RocketMQ集群是1主2从组成的一个dledger模式的高可用集群，只有一个Master Broker会接收消息的写入，有2个Cosumer不停的从RocketMQ集群消费数据。
每条数据的大小是500个字节。
- cpu负载情况 top
- 内存使用率 free 
- JVM GC频率 gc日志 
- 磁盘IO负载 top
- 网卡流量 服务器使用的是千兆网卡，千兆网卡的理论上限是每秒传输128M数据，但是一般实际最大值是每秒传输100M数据 
    sar -n DEV 1 2  
    RocketMQ处理到每秒7万消息的时候，每条消息500字节左右的大小的情况下，每秒网卡传输数据量已经达到100M了，就是已经达到了网卡的一个极限值

### 订单系统改造
![img.png](gimg.png)

### 发消息的几种方式（生产者）
- 同步发送消息到MQ
    通过代码发送消息到MQ去，SendResult sendResult = producer.send(msg)，然后等待mq返回一个结果继续往下执行

- 异步发送消息到MQ
    消息发送出去，不会等待MQ返回结果，当MQ返回结果给你的时候，Producer会回调你的SendCallback里的函数，发送成功回调onSuccess函数，发送失败了回调onExceptino函数。

- 发送单向消息到RocketMQ
    发送一个消息给MQ，不会等待mq返回结果，不会关注MQ有没有返回结果，不管发送的消息是成功还是失败
### 收消息的几种方式（消费者）
- push模式
    Broker会主动把消息发送给消费者，消费者是被动的接收Broker推送给过来的消息，然后进行处理
- pull模式
    消费者主动发送请求到Broker去拉取消息并处理

### 对其他团队要获取你们核心数据的问题，应该如何解决?
- 定时同步 统计数据按天，周可以定时同步
- 实时同步
    - 发mq，其他团队消费，容许延迟
    - 调接口 告知 
- 其他团队拉数据
    - 提供接口给他们

### 秒杀场景下？
高并发读,抢购商品界面 
- cdn + nginx + redis  提前将商品数据放入redis, 使用nginx + lua 本地缓存和redis缓存加载商品页面
高并发写 
- 独立于订单系统，新建秒杀订单系统，避免秒杀时影响普通订单的下订单
- 客户端页面需要新增答题功能，防止作弊
- nginx + lua 判断库存为0拦截下单接口，避免更多的请求直接打到下单接口
- 调用下单接口，商品库存存redis,扣减库存
- 下单接口 采用将数据写入mq，从而避免瞬间高并发数据库，使数据库宕机,订单系统拉取mq数据进行下单

### Topic、MessageQueue和Broker之间的关系是什么?
- 在创建Topic的时候需要指定一个很关键的参数，就是MessageQueue,指定你的这个Topic对应了多少个队列，也就是多少个MessageQueue
- 每个MessageQueue中会平均分配Topic的数据,每个Broker机器上都存储一些MessageQueue，MessageQueue是RocketMQ中非常关键的一个数据分片机制，通过这个方法，就可以实现Topic数据的分布式存储!
- 生产者会跟NameServer进行通信获取Topic的路由数据，生产者从NameServer中就知道一个Topic有几个MessageQueue，哪些MessageQueue在哪台Broker机器上，基于写入MessageQueue的策略，从而使消息分散在多个broker上
- Master Broker挂了，此时正在等待的其他Slave Broker自动热切换为Master Broker，那么这个时候对这一组Broker就没有Master Broker可以写入了
按照之前的策略来均匀把数据写入各个Broker上的MessageQueue，那么会导致你在一段时间内，每次访问到这个挂掉的 Master Broker都会访问失败，在Producer中开启一个开关，sendLatencyFaultEnable 一个自动容错机制，某次访问一个Broker发现网络延迟有500ms，无法访问，自动回避访问这个Broker一段时间

### broker如何存数据？
- commitlog 顺序写文件
    当生产者的消息发送到一个Broker上的时候，broker接收到了一条消息，将这个消息直接写入磁盘上的一个日志文件，叫做CommitLog，直接顺序写入这个文件，CommitLog是很多磁盘文件，每个文件限定最多1GB，Broker收到消息之后就直接追加写入这个文件的末尾，如果一个CommitLog写满了1GB，就会创建一个新的CommitLog文件。
- MessageQueue在数据存储中是体现在哪里呢?
    - 在Broker中，对Topic下的每个MessageQueue都会有一系列的ConsumeQueue文件。
    - 在Broker的磁盘上，这种格式的一系列文件: $HOME/store/consumequeue/{topic}/{queueId}/{fileName}
        {topic}指代的就是某个Topic，{queueId}指代的就是某个MessageQueue，有很多的ConsumeQueue文件，ConsumeQueue文件里存储的是一条消息对应在CommitLog文件中的offset偏移量，还包含了消息的长度，以及tag hashcode，一条数据是20个字节
    ![img.png](abc.png)
- 如何让消息写入CommitLog文件近乎内存写性能的?
    - Broker是基于OS操作系统的PageCache和顺序写两个机制
    - 数据写入CommitLog文件的时候，不是直接写入底层的物理磁盘文件，先写入OS的PageCache内存缓存中，然后由OS的后台线程选一个时间，异步化的将OS PageCache内存缓冲中的数据刷入底层的磁盘文件。
    - 磁盘文件顺序写+OS PageCache写入+OS异步刷盘的策略，基本上可以让消息写入CommitLog的性能 跟你直接写入内存里是差不多的
    - 异步刷盘模式，消息写入吞吐量非常高，可能会有数据丢失的风险 （Broker将消息写入OS PageCache中，就直接返回ACK，尚未刷入磁盘，宕机丢数据）
    - 同步刷盘模式 必须强制把消息刷入底层的物理磁盘文件中，然后才会返回ack给producer

### 基于Dledger的高可用broker如何运行？
- 基于DLedger技术管理CommitLog 
    - 用DLedger先替换掉原来Broker 自己管理的CommitLog，由DLedger来管理CommitLog
    - Broker基于DLedger管理的CommitLog 去构建出来机器上的各个ConsumeQueue磁盘文件

- Broker集群启动时，基于DLedger技术和Raft协议完成Leader选举
    - 每个节点都投票给自己
    - 选举失败后，每个节点随机休眠，先醒的机器投票给自己并发送投票信息给其他节点，其他节点醒了收到先醒机器的投票，会投票给先醒的节点
    - 选票 大于等于（机器数/2+1）,选为master节点

- Leader Broker写入之后，基于DLedger技术和Raft协议同步给Follower Broker 
    - 数据同步会分为两个阶段，一个是uncommitted阶段，一个是commited阶段
    - 首先Leader Broker上的DLedger收到一条数据之后，会标记为uncommitted状态，通过自己的DLedgerServer组件把这个uncommitted数据发送给Follower Broker的DLedgerServer。Follower Broker的DLedgerServer收到uncommitted消息之后，必须返回一个ack给Leader Broker的DLedgerServer，然后如果Leader Broker收到超过半数的Follower Broker返回ack之后，就会将消息标记为committed状态。Leader Broker上的DLedgerServer就会发送commited消息给Follower Broker机器的DLedgerServer，让他们也把消息标记为 comitted状态。

- 如果Leader Broker崩溃，则基于DLedger和Raft协议重新选举Leader
    基于DLedger还是采用Raft协议的算法，去选举出来一个新的Leader Broker继续对外提供服务，而且会对没有完成的数据同步进行一些恢复性的操作，保证数据不会丢失。

### 消费者是如何获取消息并处理？
- 消费者组概念
    一组消费者的简称，集群模式下，一个消费组获取到一条消息，只会交给组内的一台机器去处理，广播模式下，消费组获取到的一条消息，组内每台机器都可以获取到这条消息
- 重温MessageQueue、CommitLog、ConsumeQueue之间的关系
    MessageQueue会分散在多个Broker上，在每个Broker机器上，一 个MessageQueue就对应了一个ConsumeQueue，在物理磁盘上其实是对应了多个ConsumeQueue文件的，
- MessageQueue与消费者的关系
    均匀的将MessageQueue分配给消费组的多台机器来消费

- Push模式 vs Pull模式
    - 两个消费模式本质都是消费者机器主动发送请求到Broker机器去拉取一批消息下来。
    - Push是Broker会尽可能实时的把新消息交给消费者机器来进行处理，消息时效性会更好。
        一般我们使用RocketMQ的时候，消费模式通常都是基于他的Push模式来做的，因为Pull模式的代码写起来更加的复杂和繁琐
    - Push当消费者发送请求到Broker去拉取消息，有新的消息可以消费那么就会立马返回一批消息到消费机器去处理，处理完之后会接着立刻发送请求到Broker机器去拉取下一批消息。
    - Push模式下有一个请求挂起和长轮询的机制
        当消费者的请求发送到Broker，没有新的消息给你处理的时候，就会让请求线程挂起，默认是挂起15秒，后台线程每隔一会儿检查一下是否有的新的消息，在这个挂起过程中，有新的消息到达了会主动唤醒挂起的线程，然后把消息返回给消费者。

- Broker是如何将消息读取出来返回给消费机器的?
    要消费的MessageQueue以及开始消费的位置，去找到对应的ConsumeQueue读取里面对应位置的消息在CommitLog中的物理offset偏移量，然后到CommitLog中根

- 消费者机器如何处理消息、进行ACK以及提交消费进度?
    处理完这批消息之后，消费者机器就会提交我们目前的一个消费进度到Broker上去，然后Broker就会存储我们的消费进度
- 如果消费组中出现机器宕机或者扩容加机器，会怎么处理?
    如果消费组中出现机器宕机或者扩容加机器的情况，他会怎么处理?这个时候其实会进入一个rabalance的环节，也就是说重新给各个消费机器分配他们要处理的MessageQueue。

### 消费者根据什么策略从master还是slave拉取消息？
- ConsumeQueue文件也是基于os cache的
    - os读取一个磁盘文件时，自动把磁盘文件里的数据缓存到os cache中
    - ConsumeQueue文件主要是存放消息的offset，文件小，30万条消息的offset就只有5.72MB。实际上ConsumeQueue文件是不占用多少磁盘空间的，整体数据量很小，几乎可以完全被os缓存在内存cache里。
- 根据你读取到的offset去CommitLog里读取消息的完整数据
    os cache对于CommitLog而言，主要是提升文件写入性能，当你不停的写入的时候，很多最新写入的数据都会先停留在os cache里，比如这可能有10GB~20GB的数据。os会自动把cache里的比较旧的一些数据刷入磁盘里，腾出来空间给更新写入的数据放在os cache里
    - 如果你读取的是那种刚刚写入CommitLog的数据，大概率停留在os cache中，可以直接从os cache里读取CommitLog中的数据，这个就是内存读取，性能是很高的。
    - 读取的是比较早之前写入CommitLog的数据，数据被刷入磁盘，不存在os cache里了，只能从磁盘上的文件里读取，性能是比较差

- Master Broker什么时候会让你从Slave Broker拉取数据?
    对比你当前没有拉取消息的数量和大小，以及最多可以存放在os cache内存里的消息的大小，如果没拉取的消息超过了最大能使用的内存的量，那么说明你后续会频繁从磁盘加载数据，此时就让你从slave broker去加载数据了

### 基于mmap技术实现磁盘高性能读写
- 传统文件读写：
    普通的IO操作的一个弊端，必然涉及到两次数据拷贝操作，对磁盘读写性能是有影响的。 
        - 从磁盘上把数据读取到内核IO缓冲区里，再从内核IO缓存区里读取到用户进程私有空间，才能拿到这个文件里的数据
        - 必须先把数据写入到用户进程私有空间里去，再进入内核IO缓冲区，最后进入磁盘文件里去。
- mmap技术
    建立用户进程私有空间的虚拟内存和文件物理磁盘地址映射，此时并没有任何的数据拷贝操作，JDK NIO包下的MappedByteBuffer的map()函数 将文件映射到内存。mmap技术在进行文件映射的时候，一般有大小限制，在1.5GB~2GB之间
- mmap + page cache
    接下来MappedByteBuffer执行写入操作，写入的时候直接进入PageCache中，过一段时间之后，由os的线程异步刷入磁盘中
    判断要读取的数据是否在PageCache里?如果在的话，直接从PageCache里读取。如果不在PageCache，会从磁盘文件里加载数据到PageCache中，PageCache技术在加载数据的时候，会将你加载的数据块的临近的其他数据块一起加载到PageCache中。

### 丢数据场景
- 生产者往broker发送数据，网络异常没有发送成功
- broker正常接受到数据，写入page cache ，异步刷盘策略，未刷盘机器宕机
- leader故障，slave切换leader过程中，这个过程出现异常
- 磁盘坏了
- 消费者读取消息策略自动提交offset，数据未处理完宕机

### 解决丢数据
- RocketMQ的事务消息机制确定生产者发送的消息到达broker
    - 首先要让生产者去发送一条half消息到MQ去，这个half消息本质就是一个业务执行成功的消息，可以理解这个消息的状态是half状态，这时消费者系统是看不见这个half消息的，生产者等待接收half消息写入成功的响应通知
    - half消息给MQ失败了，报错了，可能mq挂了，可能网络故障了，这时执行回滚操作
    - half消息成功，生产者系统完成自己的任务
    - 生产者系统做自己的任务时发生异常，需要发rollback给mq,让mq删除half消息
    - 生产者系统完成自己的任务，需要commit half消息，发一个commit请求给mq
    - rocketmq 有一个补偿，扫描自己的half消息，如果一直没有commit或rollback 会回调生产者系统的接口，询问这个消息是commit还是rollback
     ![img.png](d.png)
- 异步刷盘改为同步刷盘
    调整broker的配置文件，将其中的flushDiskType配置设置为:SYNC_FLUSH，默认他的值是ASYNC_FLUSH，即默认是异步刷盘的。
- 消费端 改为不自动提交offset
    RocketMQ的消费者中会注册一个监听器，MessageListenerConcurrently，当你的消费者获取到一批消息之后，回调你的这个监听器函数，让消费者系统处理这一批消息。
    默认的Consumer的消费模式，处理完一批消息了，才会返回 ConsumeConcurrentlyStatus.CONSUME_SUCCESS状态标识消息都处理结束，去提交offset到broker去
    ![img.png](f.png)

### 事务机制底层实现原理
- half消息写入到rocketmq内部的“RMQ_SYS_TRANS_HALF_TOPIC”这个Topic对应的一个ConsumeQueue里，此时就会认为half消息写入成功，响应给生产者系统
- 定时任务会去扫描RMQ_SYS_TRANS_HALF_TOPIC中的half消息，超过一定时间还是half消息，回调生产者的接口，让系统判断这个half消息是要rollback还是commit
- 执行rollback操作，RocketMQ就会在OP_TOPIC里写入一条记录，标记half消息已经是rollback状态
- 执行commit操作之后，RocketMQ就会在OP_TOPIC里写入一条记录，标记half消息已经是commit状态，接着需要把放在RMQ_SYS_TRANS_HALF_TOPIC中的half消息给写入到业务topic的ConsumeQueue里去，然后消费者系统可以看见
- 一直没有执行commit/rollback，RocketMQ会回调订单系统的接口去判断half消息的状态，最多就是回调15次，如果15次之后没法告知他half消息的状态，自动把消息标记为rollback。

### 同步发送+反复重试发送的缺陷
- 反复重试耗时，影响接口返回
- 反复重试失败，回滚业务代码不方便，本地事务容易会滚，但是操作中间件的不容易会滚
- 反复重试失败，未会滚业务代码，导致系统之间数据不一致，未收到消息的系统与发送消息的系统数据不一致

### 消息重复消费原因？
- 生产者接口由于意外原因被调用多次
- 反复重试代码有概率重复发送
- 消费者消费未提交offset之前系统重启

### 幂等性，保证数据不会重复消费
- 生产者保证幂等性
    - 利用rocket mq 可查询已发送消息判断
    - 利用redis 存储已发过的消息，存在缺陷，消息已推送mq，未写入redis，系统重启
    - RocketMQ支持你查询某个消息是否存在的，性能也不是太好，会影响接口的性能。基于Redis的消息发送状态的方案，在极端情况下还是没法100%保证幂等性
- 消费者保证幂等性
    - 推荐业务判断法，直接根据你的数据存储中的记录来判断这个消息是否处理过， 如果处理过，那就别再次处理

### 消息消费过程异常怎么办？
- 重试流程？
    - RocketMQ会有一个针对你这个ConsumerGroup的重试队列，如果消费消息返回RECONSUME_LATER状态，就会把这个消息放到这个消费者组的重试队列中，消费组的名称是“VoucherConsumerGroup”，一个消费者组，那么会有一个 “%RETRY%VoucherConsumerGroup”的重试队列
    - 过一段时间，重试队列中的消息会再次给消费者处理，如果再次失败，又返回了RECONSUME_LATER，那么会再过一段时间让我们来进行处理，默认最多是重试16次!
    - 重试时间间隔 messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
- 重试失败了怎么办？
    - 重试了16次还一直没处理成功，就不要继续重试这批消息，同时消息会自动进入死信队列。
    - 死信队列的名字是“%DLQ%VoucherConsumerGroup”
    - 可以开一个后台线程，订阅“%DLQ%VoucherConsumerGroup”死信队列， 对死信队列中的消息，一直不停的处理
