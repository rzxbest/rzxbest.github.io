---
title: JAVA 对象占内存情况分析
tags: []
categories: []
date: 2021-11-24 16:14:00
---
## 对象组成结构

- 对象头
- 对象实例数据

## 对象头

|对象头  |  32bit系统   | 64bit系统  |
|  :--:  |  :--:  | :--:  |
| _mark    | 4byte  | 8byte |
| _klass    | 4byte  | 8byte(64位指针压缩4byte) |
| _length   | 4byte  | 4byte |

64位系统在内存小于32G情况下默认开启指针压缩；在开启指针压缩时，64位系统 _klass 占用4byte

- 注意：
    - 对象头中_length 只有数组才有此字段
    - klass类指针，指向类元数据的指针

## 对象实例数据

|   数据类型   | 占用空间大小 32位系统 |  占用空间大小 64位系统   |
| :----------: | :-------------------: | :----------------------: |
| long/double  |         8byte         |          8byte           |
|  int/float   |         4byte         |          4byte           |
|  short/char  |         2byte         |          2byte           |
| boolean/byte |         1byte         |          1byte           |
|  reference   |         4byte         | 8byte(64位指针压缩4byte) |

对象实例数据对齐，对象首地址必须为该对象占用空间的整数倍 
- 例如 int类型实例的首地址 %4 =0 如果不满足，则需要对齐填空padding 使其满足
- 对象实例数据在内存排序规则： long/double，int/float,short/char,boolean/byte,reference

**对象对齐**：(对象尾 + padding) % 8 = 0 ; 0 < padding < 8

## 查看普通java对象的内部布局工具

```
<dependency>
	<groupId>org.openjdk.jol</groupId>
	<artifactId>jol-core</artifactId>
	<version>0.9</version>
</dependency>
```


