---
title: LJ10字符个数统计
tags: [面试题]
categories: [华为面试题]
top: 10
date: 2021-11-30 11:10:00
---
# 字符个数统计
## 描述

编写一个函数，计算字符串中含有的不同字符的个数。字符在 ASCII 码范围内( 0~127 ，包括 0 和 127 )，换行表示结束符，不算在字符里。不在范围内的不作统计。多个相同的字符只计算一次
例如，对于字符串 abaca 而言，有 a、b、c 三种不同的字符，因此输出 3 。

数据范围： 
## 输入描述：

输入一行没有空格的字符串。
## 输出描述：

输出 输入字符串 中范围在(0~127，包括0和127)字符的种数。
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
        Set<Character> sets = new HashSet<>();
        for (char c : num.toCharArray()) {
            sets.add(c);
        }

        System.out.println(sets.size());
    }
}
```

