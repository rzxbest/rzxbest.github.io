
---
title: LJ13句子逆序
tags: [面试题]
categories: [华为面试题]
top: 13
date: 2021-11-30 11:40:00
---
# 句子逆序
## 描述

将一个英文语句以单词为单位逆序排放。例如“I am a boy”，逆序排放后为“boy a am I”
所有单词之间用一个空格隔开，语句中除了英文字母外，不再包含其他字符

数据范围：输入的字符串长度满足 

注意本题有多组输入
## 输入描述：

输入一个英文语句，每个单词用空格隔开。保证输入只包含空格和字母。
## 输出描述：

得到逆序的句子

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
        String[] s = num.split(" ");
        List<String> list = new ArrayList<>();
        for (String c : s) {
            list.add(c);
        }
        Collections.reverse(list);
        String result = "";
        for (int i = 0; i < list.size(); i++) {
            result += list.get(i);
            if (i + 1 != list.size()) {
                result += " ";
            }
        }
        System.out.println(result);
    }
}
```
