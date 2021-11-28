---
title: JAVA 对象占内存情况分析
tags: [内存,java]
categories: [java对象]
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
对象大小测试 java代码
```
import org.openjdk.jol.info.ClassLayout;
public class Main   {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseInstance(new Object()).toPrintable());
    }
}
```
开启指针压缩 -XX:+UseCompressedOops
```
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           28 0f 00 00 (00101000 00001111 00000000 00000000) (3880)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```
由此可见 上述第三个 object header 为kclass指针，占用空间4bytes

关闭指针压缩-XX:-UseCompressedOops 
```
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           00 1c 1c 12 (00000000 00011100 00011100 00010010) (303832064)
     12     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```
由此可见，上述第三个第四个objectheader 为Klass指针，占用空间8bytes

数组对象测试 java代码
```
import org.openjdk.jol.info.ClassLayout;
public class Main   {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseInstance(new Object[10]).toPrintable());
    }
}
```
指针压缩情况下输出
```
[Ljava.lang.Object; object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           60 1a 01 00 (01100000 00011010 00000001 00000000) (72288)
     12     4                    (object header)                           0a 00 00 00 (00001010 00000000 00000000 00000000) (10)
     16    40   java.lang.Object Object;.<elements>                        N/A
Instance size: 56 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```
由此可见 第四个header为数组长度，同时reference类型占用4bytes

关闭指针压缩情况下输出
```
[Ljava.lang.Object; object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           d8 9c 3a 16 (11011000 10011100 00111010 00010110) (372939992)
     12     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
     16     4                    (object header)                           0a 00 00 00 (00001010 00000000 00000000 00000000) (10)
     20     4                    (alignment/padding gap)                  
     24    80   java.lang.Object Object;.<elements>                        N/A
Instance size: 104 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```
由此可见，第五个为header为数组长度，同时reference类型占用8bytes



