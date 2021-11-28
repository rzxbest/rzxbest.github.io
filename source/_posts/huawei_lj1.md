---
title: LJ1计算字符串最后一个单词的长度
tags: [面试题]
categories: [华为面试题]
date: 2021-11-28 9:10:00
---
# 计算字符串最后一个单词的长度

## 描述
计算字符串最后一个单词的长度，单词以空格隔开，字符串长度小于5000。
（注：字符串末尾不以空格为结尾）
## 输入描述：
输入一行，代表要计算的字符串，非空，长度小于5000。
## 输出描述：
输出一个整数，表示输入字符串最后一个单词的长度。
## 代码实现
```
import java.util.Scanner;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        String allwords = scanner.nextLine();
        int index = allwords.lastIndexOf(" ");
        String  word= allwords.substring(index+1,allwords.length());
        System.out.println(word.length());
    }
}

```