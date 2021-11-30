
---
title: LJ14字符串排序
tags: [面试题]
categories: [华为面试题]
top: 14
date: 2021-11-30 11:41:00
---
# 字符串排序
## 描述
给定 n 个字符串，请对 n 个字符串按照字典序排列。

## 输入描述：

输入第一行为一个正整数n(1≤n≤1000),下面n行为n个字符串(字符串长度≤100),字符串中只含有大小写字母。
## 输出描述：

数据输出n行，输出结果为按照字典序排列的字符串。

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
        List<String> list = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            String s = scanner.nextLine();
            list.add(s);
        }
        list.sort(new Comparator<String>() {
            @Override
            public int compare(String o1, String o2) {
                return o1.compareTo(o2);
            }
        });
        for (String c : list) {
            System.out.println(c);
        }
    }
}
```