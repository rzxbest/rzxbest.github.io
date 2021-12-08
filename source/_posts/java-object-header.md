---
title: JAVA 对象头情况分析 tags: [内存,java]
categories: [java对象]
date: 2021-12-8 16:14:00
---

## 对象头情况分析

- 32位系统
- 64位系统

## 32位系统

<table>
<tr>
<td rowspan="2">锁状态</td> <td colspan="2">25bit</td> <td rowspan="2">4bit</td> <td>1bit</td> <td>1bit</td>
</tr>
<tr>
 <td>23bit</td> <td >2bit</td> <td>是否偏向锁</td> <td>锁标志位</td>
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



