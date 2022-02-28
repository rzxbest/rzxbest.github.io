---
title: 学MQ所得
date: 2022-02-28 14:59:29
tags: [MQ]
categories: [MQ]
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
