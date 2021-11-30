
---
title: LJ15求int型正整数在内存中存储时1的个数
tags: [面试题]
categories: [华为面试题]
top: 15
date: 2021-11-30 11:42:00
---
# 求int型正整数在内存中存储时1的个数

## 描述

输入一个 int 型的正整数，计算出该 int 型数据在内存中存储时 1 的个数。

数据范围：保证在 32 位整型数字范围内
## 输入描述：

 输入一个整数（int类型）
## 输出描述：

 这个数转换成2进制后，输出1的个数

 ## 代码实现
```
 import java.util.*;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        int n = scanner.nextInt();
        scanner.nextLine();
        String s = Integer.toString(n, 2);
        int length = s.replaceAll("[^1]", "").length();
        System.out.println(length);
    }
}
```