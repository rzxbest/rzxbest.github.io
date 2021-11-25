title: JAVA 对象占内存情况分析
date: 2021-11-24 16:14:37
tags:
---
对象组成结构分为
- 对象头
- 对象实例数据

对象头

|对象头  |  32bit系统   | 64bit系统  |
|  ----  |  ----  | ----  |
| _mark    | 4byte  | 8byte |
| _klass    | 4byte  | 8byte |
| _length   | 4byte  | 4byte |

64位系统在内存小于32G情况下默认开启指针压缩；在开启指针压缩时，64位系统 _klass 占用4byte

- 注意：
    - 对象头中_length 只有数组才有此字段
    - klass类指针，指向类元数据的指针

对象实例数据
- long/double 8byte
- int/float 4byte
- short/char 2byte
- boolean/byte 1byte
- reference  32位4byte 64位8byte 64位指针压缩4byte 

对象实例数据对齐，对象首地址必须为该对象占用空间的整数倍 
- 例如 int类型实例的首地址 %4 =0 如果不满足，则需要对齐填空padding 使其满足
- 对象实例数据在内存排序规则： long/double，int/float,short/char,boolean/byte,reference

对象对齐：(对象尾 + padding) % 8 = 0 ;0 < padding < 8
  


