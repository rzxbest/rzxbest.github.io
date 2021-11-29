
---
title:  LJ7取近似值
tags: [面试题]
categories: [华为面试题]
top: 7
date: 2021-11-29 16:33:00
---
# 取近似值
## 描述

写出一个程序，接受一个正浮点数值，输出该数值的近似整数值。如果小数点后数值大于等于 0.5 ,向上取整；小于 0.5 ，则向下取整。

数据范围：保证输入的数字在 32 位浮点数范围内
## 输入描述：

输入一个正浮点数值
## 输出描述：

输出该数值的近似整数值

## 代码实现
```
import java.util.*;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        double input = scanner.nextDouble();
        System.out.println(Math.round(input));

    }
}


```