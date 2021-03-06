---
title: JAVA 对象头情况分析 
tags: [内存,java]
categories: [java对象]
date: 2021-12-8 16:14:00
---

## 对象头情况分析

- 32位系统
- 64位系统

## 32位系统

<table>
<tr>
<th rowspan="2">锁状态</th> <th colspan="2">25bit</th> <th rowspan="2">4bit</th> <th>1bit</th> <th>2bit</th>
</tr>
<tr>
 <th>23bit</th> <th >2bit</th> <th>是否偏向锁</th> <th>锁标志位</th>
</tr>
<tr>
 <td>无锁</td>  <td colspan="2">对象的hashcode</td> <td>分代年龄</td> <td>0</td> <td>01</td>
</tr>
<tr>
 <td>偏向锁</td>  <td>线程ID</td> <td>Epoch</td> <td>分代年龄</td>  <td>1</td> <td>01</td>
</tr>
<tr>
 <td>轻量级锁</td>  <td colspan="4">指向栈中的锁记录的指针</td> <td>00</td>
</tr>
<tr>
 <td>重量级锁</td>  <td colspan="4">指向重量级锁的指针</td> <td>10</td>
</tr>
<tr>
 <td>GC标记</td>  <td colspan="4">空</td> <td>11</td>
</tr>
</table>


## 64位系统

<table>
<tr>
<th >锁状态</th> <th colspan="4">61bit</th> <th >是否偏向锁 1bit</th> <th>锁标志 2bit</th>
</tr>
<tr>
 <td>无锁</td> <td>unused 25bit</td> <td >对象的hashcode 31bit</td> <td>unused 1bit </td> <td>分代年龄 4bit</td> <td>0</td> <td>01</td>
</tr>
<tr>
 <td>偏向锁</td>  <td>线程ID 54bit</td> <td>Epoch 2bit </td> <td>unused 1bit </td> <td>分代年龄 4bit</td>  <td>1</td> <td>01</td>
</tr>
<tr>
 <td>轻量级锁</td>  <td colspan="5">指向栈中的锁记录的指针</td> <td>00</td>
</tr>
<tr>
 <td>重量级锁</td>  <td colspan="5">指向重量级锁的指针</td> <td>10</td>
</tr>
<tr>
 <td>GC标记</td>  <td colspan="5">空</td> <td>11</td>
</tr>
</table>

## jvm是如何使用锁和markword?
 
1. 当对象没有被加锁，一个普通的对象，MarkWord记录对象的hashcode，锁标志位01，是否偏向锁0
2. 当对象被当做同步锁并有一个线程A抢到锁时，锁标志位 01 偏向锁改成1，前23bit记录抢到锁的线程id,表示进入偏向状态
3. 当线程A再次试图来获得锁时，JVM发现同步锁对象的标志位是01，是否偏向锁是1，也就是偏向状态，Mark Word中记录的线程id就是线程A自己的id，表示线程A已经获得了这个偏向锁，可以执行同步锁的代码
4. 当线程B试图获得这个锁时，JVM发现同步锁处于偏向状态，但是Mark Word中的线程id记录的不是B，那么线程B会先用CAS操作试图获得锁，这里的获得锁操作是有可能成功的，因为线程A一般不会自动释放偏向锁。如果抢锁成功，就把Mark Word里的线程id改为线程B的id，代表线程B获得了这个偏向锁，可以执行同步锁代码。如果抢锁失败，则继续执行步骤5。
5. 偏向锁状态抢锁失败，代表当前锁有一定的竞争，偏向锁将升级为轻量级锁。JVM会在当前线程的线程栈中开辟一块单独的空间，里面保存指向对象锁Mark Word的指针，同时在对象锁Mark Word中保存指向这片空间的指针。上述两个保存操作都是CAS操作，如果保存成功，代表线程抢到了同步锁，就把Mark Word中的锁标志位改成00，可以执行同步锁代码。如果保存失败，表示抢锁失败，竞争太激烈，继续执行步骤6。
6. 轻量级锁抢锁失败，JVM会使用自旋锁，自旋锁不是一个锁状态，只是代表不断的重试，尝试抢锁。从JDK1.7开始，自旋锁默认启用，自旋次数由JVM决定。如果抢锁成功则执行同步锁代码，如果失败则继续执行步骤7。
7.自旋锁重试之后如果抢锁依然失败，同步锁会升级至重量级锁，锁标志位改为10。在这个状态下，未抢到锁的线程都会被阻塞。





