---
title:  LJ2计算某字符出现次数
tags: [面试题]
categories: [华为面试题]
date: 2021-11-28 9:14:00
---
# 计算某字符出现次数

## 描述

写出一个程序，接受一个由字母、数字和空格组成的字符串，和一个字符，然后输出输入字符串中该字符的出现次数。（不区分大小写字母）

数据范围： 输入的数据有可能包含大小写字母、数字和空格
## 输入描述：

第一行输入一个由字母和数字以及空格组成的字符串，第二行输入一个字符。
## 输出描述：

输出输入字符串中含有该字符的个数。（不区分大小写字母）
## 代码实现
```
import java.util.Scanner;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        String allstr = scanner.nextLine().toUpperCase();
        String single= scanner.nextLine().toUpperCase();
        System.out.println(allstr.replaceAll("[^"+single + "]","").length());

    }
}

```