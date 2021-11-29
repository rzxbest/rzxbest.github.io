---
title: LJ1提取不重复的数
tags: [面试题]
categories: [华为面试题]
top: 1
date: 2021-11-28 9:10:00
---
# 提取不重复的数
## 描述

输入一个 int 型整数，按照从右向左的阅读顺序，返回一个不含重复数字的新的整数。
保证输入的整数最后一位不是 0 。

数据范围： 
## 输入描述：

输入一个int型整数
## 输出描述：

按照从右向左的阅读顺序，返回一个不含重复数字的新的整数
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
        String reversenum = new StringBuilder(num).reverse().toString();
        char[] chars = reversenum.toCharArray();
        String print = "";
        ArrayList<Character> characters = new ArrayList<>();
        for (char c : chars) {
            if (!characters.contains(c)) {
                characters.add(c);
                print += c;
            }
        }
        System.out.println(print);
    }
}
```