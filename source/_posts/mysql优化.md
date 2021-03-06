---
title: mysql优化
date: 2022-02-28 14:59:29
tags: [Mysql]
categories: [Mysql]
---
# mysql优化
## 聚集索引和非聚集索引
- 聚集索引一个表只能有一个，而非聚集索引一个表可以存在多个 
- 聚集索引存储记录是物理上连续存在，而非聚集索引是逻辑上的连续，物理存储并不连续
- 聚集索引:物理存储按照索引排序；聚集索引是一种索引组织形式，索引的键值逻辑顺序决定了表数据行的物理存储顺序。
  非聚集索引:物理存储不按照索引排序；非聚集索引则就是普通索引，仅仅只是对数据列创建相应的索引，不影响整个表的物理存储顺序。
- 聚簇索引：索引的叶节点就是数据节点。而非聚簇索引的叶节点仍然是索引节点，只不过有一个指针指向对应的数据块。

## mysql的myisam和innodb区别
- 索引结构
    InnoDB的数据文件本身就是主索引文件
    MyISAM的主索引和数据是分开的
    InnoDB的辅助索引data域存储相应记录主键的值而不是地址
    MyISAM的辅助索引和主索引没有多大区别
    InnoDB是聚簇索引，数据挂在主键索引之下
- 锁
    myisam 只支持表锁
    innodb 支持行锁
- 事务
    myisam 没有事务和mvcc
    innodb 支持事务和mvcc
- 主键
    MyISAM允许没有任何索引和主键的表存在，索引都是保存行的地址
    InnoDB如果没有设定主键或非空唯一索引，就会自动生成一个6字节的主键，数据是主索引的一部分，附加索引保存的是主索引的值
- 外键
    MyISAM不支持，InnoDB支持
  
## b-tree和b+tree

一棵m阶的B-Tree有如下特性：
    1. 每个节点最多有m个孩子。
    2. 除了根节点和叶子节点外，其它每个节点至少有Ceil(m/2)个孩子。
    3. 若根节点不是叶子节点，则至少有2个孩子
    4. 所有叶子节点都在同一层，且不包含其它关键字信息
    5. 每个非终端节点包含n个关键字信息（P0,P1,…Pn, k1,…kn）
    6. 关键字的个数n满足：ceil(m/2)-1 <= n <= m-1
    7. ki(i=1,…n)为关键字，且关键字升序排序。
    8. Pi(i=1,…n)为指向子树根节点的指针。P(i-1)指向的子树的所有节点关键字均小于ki，但都大于k(i-1)

![img.png](img.png)
每个节点占用一个盘块的磁盘空间，一个节点上有两个升序排序的关键字和三个指向子树根节点的指针，指针存储的是子节点所在磁盘块的地址。两个关键词划分成的三个范围域对应三个指针指向的子树的数据的范围域。以根节点为例，关键字为17和35，P1指针指向的子树的数据范围为小于17，P2指针指向的子树的数据范围为17~35，P3指针指向的子树的数据范围为大于35。

模拟查找关键字29的过程：
    1. 根据根节点找到磁盘块1，读入内存。【磁盘I/O操作第1次】
    2. 比较关键字29在区间（17,35），找到磁盘块1的指针P2。
    3. 根据P2指针找到磁盘块3，读入内存。【磁盘I/O操作第2次】
    4. 比较关键字29在区间（26,30），找到磁盘块3的指针P2。
    5. 根据P2指针找到磁盘块8，读入内存。【磁盘I/O操作第3次】
    6. 在磁盘块8中的关键字列表中找到关键字29。

B+Tree是在B-Tree基础上的一种优化，使其更适合实现外存储索引结构，InnoDB存储引擎就是用B+Tree实现其索引结构。
B+Tree相对于B-Tree有几点不同：
    1. 非叶子节点只存储键值信息。
    2. 所有叶子节点之间都有一个链指针。
    3. 数据记录都存放在叶子节点中。“”
![img_1.png](img_1.png)
推算：
    InnoDB存储引擎中页的大小为16KB，一般表的主键类型为INT（占用4个字节）或BIGINT（占用8个字节），指针类型也一般为4或8个字节，也就是说一个页（B+Tree中的一个节点）中大概存储16KB/(8B+8B)=1K个键值（因为是估值，为方便计算，这里的K取值为〖10〗^3）。
    也就是说一个深度为3的B+Tree索引可以维护10^3 * 10^3 * （16KB/每条数据的大小+键值=100B）= 1600万 条记录。
    实际情况中每个节点可能不能填充满，因此在数据库中，B+Tree的高度一般都在2-4层。
    mysql的InnoDB存储引擎在设计时是将根节点常驻内存的，也就是说查找某一键值的行记录时最多只需要1~3次磁盘I/O操作。

## 执行计划
### explain两个变种
- explain extended:会在 explain 的基础上额外提供一些查询优化的信息，紧随其后通过 show warnings 命令可以得到优化后的查询语句，从而看出优化器优化了什么。
    - 例如：explain extended select * from film where id = 1;
    - sshow warnings;
- explain partitions:相比 explain 多了个 partitions 字段，如果查询是基于分区表的 话，会显示查询将访问的分区。

### explain中的列
- id列
    - select语句的编号，越大越先执行
    - 相同编号，从上往下执行
- select_type列
    - simple 简单查询
    - primary 复杂查询最外层的查询
    - subquery 子查询但是不包括from后的子查询
    - derived from 后的子查询
    - union union后的select
- table列 查询的表
    - 当from子句中有子查询时，table列是 <derivenN> 格式，表示当前查询依赖 id=N 的查 询，于是先执行 id=N 的查询
    - 当有 union 时，UNION RESULT 的 table 列的值为<union1,2>，1和2表示参与 union 的 select 行id
- type列：查询类型
    - system > const > eq_ref > ref > range > index > ALL 一般来说，得保证查询达到range级别，最好达到ref
    - const, system:mysql能对查询的某部分进行优化并将其转化成一个常量
    - eq_ref:primary key 或 unique key 索引的所有部分被连接使用 ，最多只会返回一条符合条件的记录。
    - ref: 简单 select 查询，name是普通索引(非唯一索引)；关联表查询，使用联合索引连接
    - range：范围扫描通常出现在 in(), between ,> ,<, >= 等操作中。使用一个索引来检索给定范围的行。
    - index:扫描全表索引
    - ALL:全表扫描，
- possible_keys列
    可能使用哪些索引来查找
- key列
    mysql实际采用哪个索引来优化对该表的访问
- key_len列
    - mysql在索引里使用的字节数，算出具体使用了索引中的哪些列。
    - key_len计算规则如下: 字符串:char(n):n字节长度;varchar(n):2字节存储字符串长度，如果是utf-8，则长度 3n +2。数值类型:tinyint:1字节;smallint:2字节;int:4字节 ;bigint:8字节。时间类型:date:3字节;timestamp:4字节;datetime:8字节
    - 如果字段允许为 NULL，需要1字节记录是否为 NULL
    - 索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。
- ref列 key列记录的索引中，表查找值所用到的列或常量，常见的有:const(常量)，字段名
- rows列 mysql估计要读取并检测的行数，注意不是结果集里的行数。
- Extra列:额外信息
    - 1)Using index:查询的列被索引覆盖，并且where筛选条件是索引的是前导列
    - 2)Using where:查询的列未被索引覆盖，where筛选条件非索引的前导列
    - 3)Using index condition:查询的列不全在索引中，where条件中是一个前导列的范围；查询列不完全被索引覆盖，查询条件完全可以使用到索引
    - 4)Using temporary:mysql需要创建一张临时表来处理查询
    - 5)Using filesort:将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序
    - 6)Select tables optimized away:使用某些聚合函数(比如 max、min)来访问存在索引的某个字段
    - 7) Using where Using index 查询的列被索引覆盖，并且where筛选条件是索引列之一但是不是索引的不是前导列
    - 8) NULL 查询的列未被索引覆盖，并且where筛选条件是索引的前导列，需要回表

## 调优
### 索引实践
- 查询条件全值匹配
- 最左前缀法则，联合索引，要遵守最左前缀法则。指的是查询从索引的最左前列开始并且不跳过索引中的列。
- 不在索引列上做任何操作(计算、函数、(自动or手动)类型转换)，会导致索引失效而转向全表扫描
- 存储引擎不能使用联合索引（A，B）中（A）范围条件右边的列（B）
- 尽量使用覆盖索引(只访问索引的查询(索引列包含查询列))，减少select *语句
- mysql在使用不等于(!=或者<>)的时候无法使用索引会导致全表扫描
- is null,is not null 也无法使用索引
- like以通配符开头('%abc...')mysql索引失效会变成全表扫描操作
    - 解决like'%字符串%'索引不被使用的方法? 
        - a)使用覆盖索引，查询字段必须是建立覆盖索引字段
        - b)如果不能使用覆盖索引则可能需要借助搜索引擎
- 字符串不加单引号索引失效
- 少用or或in，用它查询时，mysql不一定使用索引，mysql内部优化器会根据检索比例、 表大小等多个因素整体评估是否使用索引，详见范围查询优化
- 范围查询优化，可能是由于单次数据量查询过大导致优化器最终选择不走索引 优化方法:可以讲大的范围拆分成多个小
### 为什么推荐建表时字段为not null?

- 通常情况下最好指定列为NOT NULL，除非真的需要存储NULL值。
    - 如果查询中包含可为NULL的列，对MySql来说更难优化，因为可为NULL的列使得索引、索引统计和值比较都更复杂。
    - 可为NULL的列会使用更多的存储空间，在MySql里也需要特殊处理。当可为NULL的列被索引时，每个索引记录需要一个额外的字节，在MyISAM里甚至还可能导致固定大小的索引（例如只有一个整数列的索引）变成可变大小的索引。
    - 通常把可为NULL的列改为NOT NULL带来的性能提升比较小，所以（调优时）没有必要首先在现有schema中查找并修改掉这种情况，除非确定这会导致问题。
    - 如果计划在列上建索引，就应该尽量避免设计成可为NULL的列。

- mysql的默认值

对于MySql而言，如果不主动设置为NOT NULL的话，那么插入数据的时候默认值就是NULL。NULL和NOT NULL使用的空值代表的含义是不一样，NULL可以认为这一列的值是未知的，空值则可以认为我们知道这个值，只不过他是空的而已。举个例子，一张表中的某一条name字段是NULL，我们可以认为不知道名字是什么，反之如果是空字符串则可以认为我们知道没有名字，他就是一个空值。而对于大多数程序的情况而言，没有什么特殊需要非要字段要NULL的吧，NULL值反而会对程序造成比如空指针的问题。

默认值为NULL带来的问题

- 聚合函数不准确
    对于NULL值的列，使用聚合函数的时候会忽略NULL值。例如：有一张表，name字段默认是NULL，此时对name进行count得出的结果是1，这个是错误的。count(*)是对表中的行数进行统计，count(name)则是对表中非NULL的列进行统计。

- =失效
    对于NULL值的列，是不能使用=表达式进行判断的，下面对name的查询是不成立的，必须使用is NULL。

- 与其他值运算
    NULL和其他任何值进行运算都是NULL，包括表达式的值也是NULL。user表第二条记录age是NULL，所以+1之后还是NULL，name是NULL，进行concat运算之后结果还是NULL。

- distinct、group by、order by
    对于distinct和group by来说，所有的NULL值都会被视为相等，对于order by来说升序NULL会排在最前。

- 索引问题
    - 官方文档：使用is NULL和范围查询都是可以和正常一样使用索引的，只不过在某些场景下，由于mysql的执行策略导致索引失效。
    - sql执行过程中，到优化器阶段，会选择使用什么索引比较合理，sql具体执行方案在这里确定下来，索引列存在NULL就会导致优化器在做索引选择的时候更复杂，更加难以优化。

### 排序优化
- MySQL支持两种方式的排序filesort和index，Using index是指MySQL扫描索引本身完成排序。index效率高，filesort效率低。
    - order by满足两种情况会使用Using index。
        - order by语句使用索引最左前列。
        - 使用where子句与order by子句条件列组合满足索引最左前列。 
    - 尽量在索引列上完成排序，遵循索引建立(索引创建的顺序)时的最左前缀法则。
    - 如果order by的条件不在索引列上，就会产生Using filesort。
    - 能用覆盖索引尽量用覆盖索引
    - group by与order by很类似，其实质是先排序后分组，遵照索引创建顺序的最左前缀法则。对于group by的优化如果不需要排序的可以加上order by null禁止排序。注意，where高于having，能写在where中 的限定条件就不要去having限定了。



- Using filesort文件排序原理详解 filesort文件排序方式
    - 单路排序:是一次性取出满足条件行的所有字段，然后在sort buffer中进行排序;用trace工具可 以看到sort_mode信息里显示< sort_key, additional_fields >或者< sort_key, packed_additional_fields >
   
    - 双路排序(回表排序模式):是首先根据相应的条件取出相应的排序字段和可以直接定位行数据的行 ID，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段;用trace工具 可以看到sort_mode信息里显示< sort_key, rowid >

    - MySQL 通过比较系统变量 max_length_for_sort_data(默认1024字节) 的大小和需要查询的字段总大小来 判断使用哪种排序模式。如果 max_length_for_sort_data 比查询字段的总长度大，那么使用 单路排序模式; 如果 max_length_for_sort_data 比查询字段的总长度小，那么使用 双路排序模式。

例如：select * from a_table where b = '111' order by c; 注：字段b建立索引，字段c无索引
- 单路排序:
    1. 索引b找到第一个满足 b = '111' 条件的主键id 
    2. 根据主键 id 取出整行，取出所有字段的值，存入 sort_buffer 中 
    3. 从索引b找到下一个满足 b = '111' 条件的主键 id 
    4. 重复步骤 2、3 直到不满足  b = '111' 
    5. 对 sort_buffer 中的数据按照字段 c 进行排序
    6. 返回结果给客户端
- 双路排序: set max_length_for_sort_data = 10; ‐‐设置表所有字段长度总和肯定大于10字节
    1. 从索引 b 找到第一个满足 b = '111' 的主键id
    2. 根据主键 id 取出整行，把排序字段 c 和主键 id 这两个字段放到 sort buffer 中 
    3. 从索引 b 取下一个满足 b = '111' 记录的主键 id
    4. 重复 3、4 直到不满足 b = ‘111’
    5. 对 sort_buffer 中的字段 c 和主键 id 按照字段 c 进行排序
    6. 遍历排序好的 id 和字段 c，按照 id 的值回到原表中取出 所有字段的值返回给客户端

如果全部使用sort_buffer内存排序一般情况下效率会高于磁盘文件排序，但不能因为这个就随便增 大sort_buffer(默认1M)，mysql很多参数设置都是做过优化的，不要轻易调整。

### 分页查询优化
例如：select * from employees limit 10000,10 
从表 employees 中取出从 10001 行开始的 10 行记录。看似只查询了 10 条记录，实际这条 SQL 是先读取 10010 条记录，然后抛弃前 10000 条记录，然后读到后面 10 条想要的数据。因此要查询一张大表比较靠后的数据，执行效率 是非常低的。

- 按照主键排序并limit：条件：主键自增且连续，结果是按照主键排序的
    select * from employees where id > 90000 limit 5; 没单独加 order by，表示通过主键排序
- 根据非主键字段排序的分页查询:连接子查询，子查询使用覆盖索引
    select * from employees ORDER BY name limit 90000,5; name字段有索引，但是查询计划显示没有走索引，使用索引比全表扫描成本高
    优化方案：select * from employees e inner join (select id from employees order by name limit 90000,5) eq on eq.id = e.id

### join关联优化
- mysql的表关联常见有两种算法
    - Nested-Loop Join 算法
    - Block Nested-Loop Join 算法
- 嵌套循环连接 Nested-Loop Join(NLJ)算法 ：优化器一般会优先选择小表做驱动表，执行计划 Extra 中未出现 Using join buff
    - 循环地从第一张表(称为驱动表)中读取行，在这行数据中取到关联字段，根据关联字段在另一张表(被驱动表)里取出满足条件的行，然后取出两张表的结果合集。
    - select*from t1 inner join t2 on t1.a= t2.a; 这里假设t2小表（100条） t1大表 （10000）
    - 上面sql的大致流程如下:
        1. 从表 t2 中读取一行数据;
        2. 从第 1 步的数据中，取出关联字段 a，到表 t1 中查找;
        3. 取出表 t1 中满足条件的行，跟 t2 中获取到的结果合并，作为结果返回给客户端; 
        4. 重复上面 3 步。
    - 整个过程会读取 t2 表的所有数据(扫描100行)，然后遍历这每行数据中字段 a 的值，根据 t2 表中 a 的值索引扫描 t1 表 中的对应行(扫描100次 t1 表的索引，1次扫描可以认为最终只扫描 t1 表一行完整数据，也就是总共 t1 表也扫描了100 行)。因此整个过程扫描了 200 行。
    - 如果被驱动表的关联字段没索引，使用NLJ算法性能会比较低(下面有详细解释)，mysql会选择Block Nested-Loop Join 算法。
- 基于块的嵌套循环连接 Block Nested-Loop Join(BNL)算法：被驱动表连接字段无索引
    - 把驱动表的数据读入到 join_buffer 中，然后扫描被驱动表，把被驱动表每一行取出来跟 join_buffer 中的数据做对比。 
    - select*from t1 inner join t2 on t1.b= t2.b; 这里假设t2小表（100条） t1大表 （10000）
        1. 把 t2 的所有数据放入到 join_buffer 中
        2. 把表 t1 中每一行取出来，跟 join_buffer 中的数据做对比 3. 返回满足 join 条件的数据
    - 整个过程对表 t1 和 t2 都做了一次全表扫描，因此扫描的总行数为10000(表 t1 的数据总量) + 100(表 t2 的数据总量) = 10100。并且 join_buffer 里的数据是无序的，因此对表 t1 中的每一行，都要做 100 次判断，所以内存中的判断次数是 100 * 10000= 100 万次。

被驱动表的关联字段没索引为什么要选择使用 BNL 算法而不使用 Nested-Loop Join 呢? 
如果上面第二条sql使用 Nested-Loop Join，那么扫描行数为 100 * 10000 = 100万次，这个是磁盘扫描。
很显然，用BNL磁盘扫描次数少很多，相比于磁盘扫描，BNL的内存计算会快得多。

### in和exsits优化 ：小表驱动大表
- A表与B表的ID字段应建立索引
- in:当B表的数据集小于A表的数据集时，in优于exists
```
    select * from A where id in(select id from B)
    等价于 
    for  bid in select * from B
        select * from A where id = bid  //先扫描索引，然后在根据索引查数据

```
- exists:当A表的数据集小于B表的数据集时，exists优于in
```
select * from A where exists(select 1 from B where B.id=A.id)
 等价于
 for(select * from A){
 select * from B where B.id = A.id
}
```
### count（*）优化
id为主键，name普通索引
EXPLAIN select count(1) from employees; 
EXPLAIN select count(id) from employees;
EXPLAIN select count(name) from employees; 
EXPLAIN select count(*) from employees;
四个sql的执行计划一样，说明这四个sql执行效率应该差不多，区别在于根据某个字段count不会统计字段为null值的数据行
为什么mysql最终选择辅助索引而不是主键聚集索引?因为二级索引相对主键索引存储数据更少，检索性能应该更高

常见优化方法
1、查询mysql自己维护的总行数 对于myisam存储引擎的表做不带where条件的count查询性能是很高的，因为myisam存储引擎的表的总行数会被 mysql存储在磁盘上，查询不需要计算
对于innodb存储引擎的表mysql不会存储表的总记录行数，查询count需要实时计算
2、show table status 如果只需要知道表总行数的估计值可以用如下sql查询，性能很高
3、将总数维护到Redis里 插入或删除表数据行的时候同时维护redis里的表总行数key的计数值(用incr或decr命令)，但是这种方式可能不准，很难 保证表操作和redis操作的事务一致性
4、增加计数表 插入或删除表数据行的时候同时维护计数表，让他们在同一个事务里操作

### 锁
- 锁分类
    - 性能上分为乐观锁和悲观锁
    - 对数据库的操作上分为读锁(共享锁：多个读不影响)和写锁(排他锁：阻断其他读写锁)，都属于悲观锁
    - 从数据的操作粒度上分表锁和行锁
- 表锁
    - 每次操作锁住整张表。开销小，加锁快;不会出现死锁;锁定粒度大，发生锁冲突的概率最高，并发度最低
    - 手动增加表锁 lock table 表名称 read(write),表名称2 read(write);
    - 查看表上加过的锁show open tables;
    - 删除表锁 unlock tables;
    - MyISAM在执行查询语句(SELECT)前,会自动给涉及的所有表加读锁,在执行增删改操作前,会自动给涉及的表加写锁。
    - 读锁会阻塞写，不会阻塞读。写锁则会把读和写都阻塞


- 行锁 
    - 每次操作锁住一行数据。开销大，加锁慢;会出现死锁;锁定粒度最小，发生锁，冲突的概率最低，并发度最高。

- InnoDB与MYISAM的最大不同有两点:
    支持事务(TRANSACTION) 支持行级锁

### 面试过程中如何回答mysql相关的优化？
- 从建表语句
    - 主键尽量使用整形自增
        - 占用空间小
        - 主键自增，新增的数据永远在后面
        - 排序效率高
        - 范围查找
    - 表字段使用not null,null值很难查询且占用额外空间
    - 单表字段不要太多
    - 建立查询字段索引
- 从一条查询语句
    - 不用select*
    - 尽可能使用覆盖索引，避免回表
    - join 字段建立索引
    -  大表 in 小表  ，小表 exist 大表
    - 查询条件不用函数
    - 避免使用不等于
    - 对于连续数值，使用BETWEEN不用IN
    - 避免like '%xxx'
    - 不做列运算
    - limit 查询用连接子查询优化，子查询使用覆盖索引

- 拆分大的delete或insert语句
- 可通过开启慢查询日志来找出较慢的SQL