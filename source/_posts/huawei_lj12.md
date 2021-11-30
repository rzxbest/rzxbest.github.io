---
title: LJ12字符串反转
tags: [面试题]
categories: [华为面试题]
top: 12
date: 2021-11-30 11:30:00
---
# 字符串反转
## 描述

接受一个只包含小写字母的字符串，然后输出该字符串反转后的字符串。（字符串长度不超过1000）
## 输入描述：

输入一行，为一个只包含小写字母的字符串。
## 输出描述：

输出该字符串反转后的字符串。

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