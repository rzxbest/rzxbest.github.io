---
title:  LJ6质数因子
tags: [面试题]
categories: [华为面试题]
date: 2021-11-28 21:33:00
---
# 质数因子

## 描述

功能:输入一个正整数，按照从小到大的顺序输出它的所有质因子（重复的也要列举）（如180的质因子为2 2 3 3 5 ）

## 输入描述：

输入一个整数
## 输出描述：

按照从小到大的顺序输出它的所有质数的因子，以空格隔开。最后一个数后面也要有空格。
## 代码实现
```
import java.util.*;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        int input = scanner.nextInt();
        for (int i = 2; i <= Math.sqrt(input); i++) {
            while (input % i == 0){
                System.out.print(i + " ");
                input = input / i;
            }
        }
        if (input > 1 ){
            System.out.print(input + " ");
        }
    }


}



```
