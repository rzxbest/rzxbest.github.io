---
title: redis
tags: [redis]
categories: [redis]
top: 199
date: 2022-03-31 9:10:00
---
# redis

## aof重写
### aof重写原因
- AOF 持久化是通过保存被执行的写命令来记录数据库状态的，所以AOF文件的大小随着时间的流逝一定会越来越大；影响包括但不限于：对于Redis服务器，计算机的存储压力；AOF还原出数据库状态的时间增加；
- 为了解决AOF文件体积膨胀的问题，Redis提供了AOF重写功能：Redis服务器可以创建一个新的AOF文件来替代现有的AOF文件，新旧两个文件所保存的数据库状态是相同的，但是新的AOF文件不会包含任何浪费空间的冗余命令，通常体积会较旧AOF文件小很多。

### AOF重写功能的实现原理
AOF重写功能的实现原理：从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录该键值对的多个命令;


```
def AOF_REWRITE(tmp_tile_name):

  f = create(tmp_tile_name)

  # 遍历所有数据库
  for db in redisServer.db:

    # 如果数据库为空，那么跳过这个数据库
    if db.is_empty(): continue

    # 写入 SELECT 命令，用于切换数据库
    f.write_command("SELECT " + db.number)

    # 遍历所有键
    for key in db:

      # 如果键带有过期时间，并且已经过期，那么跳过这个键
      if key.have_expire_time() and key.is_expired(): continue

      if key.type == String:

        # 用 SET key value 命令来保存字符串键

        value = get_value_from_string(key)

        f.write_command("SET " + key + value)

      elif key.type == List:

        # 用 RPUSH key item1 item2 ... itemN 命令来保存列表键

        item1, item2, ..., itemN = get_item_from_list(key)

        f.write_command("RPUSH " + key + item1 + item2 + ... + itemN)

      elif key.type == Set:

        # 用 SADD key member1 member2 ... memberN 命令来保存集合键

        member1, member2, ..., memberN = get_member_from_set(key)

        f.write_command("SADD " + key + member1 + member2 + ... + memberN)

      elif key.type == Hash:

        # 用 HMSET key field1 value1 field2 value2 ... fieldN valueN 命令来保存哈希键

        field1, value1, field2, value2, ..., fieldN, valueN =\
        get_field_and_value_from_hash(key)

        f.write_command("HMSET " + key + field1 + value1 + field2 + value2 +\
                        ... + fieldN + valueN)

      elif key.type == SortedSet:

        # 用 ZADD key score1 member1 score2 member2 ... scoreN memberN
        # 命令来保存有序集键

        score1, member1, score2, member2, ..., scoreN, memberN = \
        get_score_and_member_from_sorted_set(key)

        f.write_command("ZADD " + key + score1 + member1 + score2 + member2 +\
                        ... + scoreN + memberN)

      else:

        raise_type_error()

      # 如果键带有过期时间，那么用 EXPIREAT key time 命令来保存键的过期时间
      if key.have_expire_time():
        f.write_command("EXPIREAT " + key + key.expire_time_in_unix_timestamp())

    # 关闭文件
    f.close()

```
### aof子线程重写
- aof子线程重写原因
    - aof_rewrite函数创建新文件并进行大量写入操作，调用这个函数的线程将被长时间的阻塞
    - Redis服务器使用单线程来处理命令请求，如果直接是服务器进程调用AOF_REWRITE函数的话，重写AOF期间，服务器将无法处理客户端发送来的命令请求
    - Redis不希望AOF重写会造成服务器无法处理请求，所以Redis决定将AOF重写程序放到子进程（后台）里执行。

- 使用子进程进行AOF重写的问题
    - 子进程在进行AOF重写期间，服务器进程还要继续处理命令请求，而新的命令可能对现有的数据进行修改，这会让当前数据库的数据和重写后的AOF文件中的数据不一致。

- 解决数据不一致问题
    - Redis增加了一个AOF重写缓存，这个缓存在fork出子进程之后开始启用
    - Redis服务器主进程在执行完写命令之后，会同时将这个写命令追加到AOF缓冲区和AOF重写缓冲区
    - 当子进程完成对AOF文件重写之后，会向父进程发送一个完成信号，父进程接到该完成信号之后，会调用一个信号处理函数
        - 将AOF重写缓存中的内容全部写入到新的AOF文件中；这个时候新的AOF文件所保存的数据库状态和服务器当前的数据库状态一致；
        - 新的AOF文件进行改名，原子的覆盖原有的AOF文件；完成新旧两个AOF文件的替换。

## 主从复制

### 全量同步
- 从服务器连接主服务器，发送SYNC命令； 
- 主服务器接收到SYNC命名后，开始执行BGSAVE命令子线程生成RDB文件并使用缓冲区记录此后执行的所有写命令； 
- 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令； 
- 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照； 
- 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令； 
- 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令； 

### 增量同步
- Redis增量复制是指Slave初始化后开始正常工作时主服务器发生的写操作同步到从服务器的过程。 
- 增量复制的过程主要是主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令

### Redis主从同步策略
主从刚刚连接的时候，进行全量同步；全同步结束后，进行增量同步。当然，如果有需要，slave 在任何时候都可以发起全量同步。redis策略是，无论如何，首先会尝试进行增量同步，如不成功，要求从机进行全量同步。


### 主从同步存在的问题
- 数据不一致，网络传输延迟，读数据读取从库
- 读取过期时间的key,从库读取,3.2之前的不检查过期, 会返回数据, 3.2之后虽然不删除过期数据, 但是返回空值


## 主从切换
### 切换方式 
- 手动切换
    - 手动将从节点设置成主节点。命令：redis-cli -h <从节点ip> -p <从节点端口号> slaveof no one
- 哨兵模式
    - Sentinel实例可以自动的将主节点下的其中一个从节点升级为新的主节点
    - Sentinel实例会不断检测主从节点是否正常运
    - 当某个节点出现异常宕机时，Sentinel实例会向管理员或者其他应用发送提醒
    - 当主节点宕机时，Sentinel实例会将该主节点下的其中一个从节点升级为新的主节点，并且原先其他从节点重新发起socket请求成为新的主节点的从节点
    - 向客户端返回新主节点的地址，就可以正常上使用新的主节点来处理请求
    
### 哨兵模式
只开启一个Sentinel实例进行监视，容易出现问题，一般情况下会开启多个Sentinel实例进行监控，一般情况下至少需要3个Sentinel实例。
主节点宕机状态
    - 只有一个哨兵认为这个主节点宕机了，则成为主观宕机。
    - 如果达到一定数量的节点认为该主节点宕机，则成为客观宕机。

为什么至少需要3个Sentinel实例？
- 当指定时间内一定哨兵数量（哨兵数量 / 2 + 1）都认为主节点宕机则称为客观宕机，如果数量为2，出现一个哨兵宕机的情况，在需要主从切换的时候因为无法达到认为主节点宕机的哨兵数量为2，所以在主节点出现宕机时无法进行主从切换。所以说部署哨兵至少需要3个Sentinel实例来保证健壮性。

### 哨兵模式引发数据丢失问题
哨兵模式 + Redis主从复制这种部署结构，无法保证数据不会出现丢失。
哨兵模式下数据丢失主要有两种情况：
因为主从复制是异步操作，可能主从复制还没成功，主节点宕机了。这时候还没成功复制的数据就会丢失了。
如果主节点无法与其他从节点连接，但是实际上还在运行。这时候哨兵会将一个从节点切换成新的主节点，但是在这个过程中实际上主节点还在运行，所以继续向这个主节点写入的数据会被丢失。

### 解决数据丢失方案
使用命令：
min-slaves-to-write 1
min-slaves-max-lag 10
使用这组命令可以设置至少有一个从节点数据复制延迟不能超过10S，也就是说如果一个直接点下所有从节点数据复制延迟都超过10S，则停止主节点继续接收处理新的请求。这样可以保证数据丢失最多只会丢失10S内的数据。

## redis 持久化方案

数据库的恢复
服务器启动时，如果没有开启AOF持久化功能，则会自动载入RDB文件，期间会阻塞主进程。如果开启了AOF持久化功能，服务器则会优先使用AOF文件来还原数据库状态，因为AOF文件的更新频率通常比RDB文件的更新频率高，保存的数据更完整。

### RDB持久化
- RDB持久化即通过创建快照（压缩的二进制文件）的方式进行持久化，保存某个时间点的全量数据。RDB持久化是Redis默认的持久化方式。
- 触发方式
    - 手动触发
        - save， 在命令行执行save命令，将以同步的方式创建rdb文件保存快照，会阻塞服务器的主进程，生产环境中不要用    
        - bgsave, 在命令行执行bgsave命令，将通过fork一个子进程以异步的方式创建rdb文件保存快照，除了fork时有阻塞，子进程在创建rdb文件时，主进程可继续处理请求
    - 自动触发
        - 在redis.conf中配置 save m n 定时触发，如 save 900 1表示在900s内至少存在一次更新就触发
        - 主从复制时，如果从节点执行全量复制操作，主节点自动执行bgsave生成RDB文件并发送给从节点
        - 执行debug reload命令重新加载Redis时
        - 执行shutdown且没有开启AOF持久化
        - redis.conf中RDB持久化配置
```
# 只要满足下列条件之一，则会执行bgsave命令
save 900 1 # 在900s内存在至少一次写操作
save 300 10
save 60 10000
# 禁用RBD持久化，可在最后加 save ""

# 当备份进程出错时主进程是否停止写入操作
stop-writes-on-bgsave-error yes  
# 是否压缩rdb文件 推荐no 相对于硬盘成本cpu资源更贵
rdbcompression no
```
### AOF持久化
AOF（Append-Only-File）持久化即记录所有变更数据库状态的指令，以append的形式追加保存到AOF文件中。在服务器下次启动时，通过载入和执行AOF文件中保存的命令，来还原服务器关闭前的数据库状态。

redis.conf中AOF持久化配置如下
```
# 默认关闭AOF，若要开启将no改为yes
appendonly no

# append文件的名字
appendfilename "appendonly.aof"

# 每隔一秒将缓存区内容写入文件 默认开启的写入方式
appendfsync everysec 

# 当AOF文件大小的增长率大于该配置项时自动开启重写（这里指超过原大小的100%）。
auto-aof-rewrite-percentage 100

# 当AOF文件大小大于该配置项时自动开启重写
auto-aof-rewrite-min-size 64mb

```
AOF持久化的实现包括3个步骤:

命令追加：将命令追加到AOF缓冲区
文件写入：缓冲区内容写到AOF文件
文件保存：AOF文件保存到磁盘
其中后两步的频率通过appendf sync来配置，appendfsync的选项包括

always， 每执行一个命令就保存一次，安全性最高，最多只丢失一个命令的数据，但是性能也最低（频繁的磁盘IO）
everysec，每一秒保存一次，推荐使用，在安全性与性能之间折中，最多丢失一秒的数据
no， 依赖操作系统来执行（一般大概30s一次的样子），安全性最低，性能最高，丢失操作系统最后一次对AOF文件触发SAVE操作之后的数据

### RDB vs AOF
RDB与AOF两种方式各有优缺点。

RDB的优点：与AOF相比，RDB文件相对较小，恢复数据比较快（原因见数据恢复部分）
RDB的缺点：服务器宕机，RBD方式会丢失掉上一次RDB持久化后的数据；使用bgsave fork子进程时会耗费内存。

AOF的优点： AOF只是追加文件，对服务器性能影响较小，速度比RDB快，消耗内存也少，同时可读性高。
AOF的缺点：生成的文件相对较大，即使通过AOF重写，仍然会比较大；恢复数据的速度比RDB慢。


### 数据恢复
![qidong.png](qidong.png)

### RDB、AOF混合持久化
Redis从4.0版开始支持RDB与AOF的混合持久化方案。首先由RDB定期完成内存快照的备份，然后再由AOF完成两次RDB之间的数据备份，由这两部分共同构成持久化文件。该方案的优点是充分利用了RDB加载快、备份文件小及AOF尽可能不丢数据的特性。缺点是兼容性差，一旦开启了混合持久化，在4.0之前的版本都不识别该持久化文件，同时由于前部分是RDB格式，阅读性较低。

- 开启混合持久化 aof-use-rdb-preamble yes
- 数据恢复加载过程就是先按照RDB进行加载，然后把AOF命令追加写入。

### 持久化方案的建议
Redis只用来做缓存服务器，比如数据库查询数据后缓存，可以不用考虑持久化，因为缓存服务失效还能再从数据库获取恢复。
提供很高的数据保障性，建议你同时使用两种持久化方式。如果可以接受灾难带来的几分钟的数据丢失，那么可以仅使用RDB。
通常的设计思路是利用主从复制机制来弥补持久化时性能上的影响。即Master上RDB、AOF都不做，保证Master的读写性能，而Slave上则同时开启RDB和AOF（或4.0以上版本的混合持久化方式）来进行持久化，保证数据的安全性。
