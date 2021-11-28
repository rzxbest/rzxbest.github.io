---
title:  LJ4字符串分割
tags: [面试题]
categories: [华为面试题]
top: 4
date: 2021-11-28 10:33:00
---
# 字符串分割

## 描述

•连续输入字符串，请按长度为8拆分每个输入字符串并进行输出； 
•长度不是8整数倍的字符串请在后面补数字0，空字符串不处理。
## 输入描述：

连续输入字符串(输入多次,每个字符串长度小于等于100)
## 输出描述：

依次输出所有分割后的长度为8的新字符串
## 代码实现
```
import java.util.*;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        List<String> strInput = new ArrayList<>();
        while (scanner.hasNext()) {
            String s = scanner.nextLine();
            strInput.addAll(strInput(s));
        }
        for (String str : strInput) {
            System.out.println(str);
        }
    }

    public static List<String> strInput(String input) {
        List<String> result = new ArrayList<>();
        while (input.length() >= 8) {
            result.add(input.substring(0, 8));
            input = input.substring(8);
        }
        int len = input.length();
        if (len != 0) {
            input += "00000000".substring(0, 8 - len);
            result.add(input);
        }
        return result;
    }

}



```