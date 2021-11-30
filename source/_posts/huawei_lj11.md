
---
title: LJ11数字颠倒
tags: [面试题]
categories: [华为面试题]
top: 11
date: 2021-11-30 11:20:00
---
# 数字颠倒
## 描述

输入一个整数，将这个整数以字符串的形式逆序输出
程序不考虑负数的情况，若数字含有0，则逆序形式也含有0，如输入为100，则输出为001

## 输入描述：

输入一个int整数
## 输出描述：

将这个整数以字符串的形式逆序输出

## 代码实现
```
import java.util.*;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        String num = scanner.nextLine();
        System.out.println(new StringBuilder(num).reverse().toString());
    }
}

```