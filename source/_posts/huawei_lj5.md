---
title:  LJ5进制转换
tags: [面试题]
categories: [华为面试题]
date: 2021-11-28 10:33:00
---
# 进制转换

## 描述

写出一个程序，接受一个十六进制的数，输出该数值的十进制表示。

数据范围：保证结果在 

注意本题有多组输入
## 输入描述：

输入一个十六进制的数值字符串。注意：一个用例会同时有多组输入数据，请参考帖子https://www.nowcoder.com/discuss/276处理多组输入的问题。
## 输出描述：

输出该数值的十进制字符串。不同组的测试用例用\n隔开。
## 代码实现
```
import java.util.*;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        List<Integer> strInput = new ArrayList<>();
        while (scanner.hasNext()) {
            String inputStr = scanner.nextLine();
            int i = Integer.parseInt(inputStr.substring(2), 16);
            strInput.add(i);

        }
        for (Integer i : strInput) {
            System.out.println(i);
        }
    }


}

```

## java进制通用转换
### r进制转10进制
Integer.parseInt(str,r);
### 10进制转r进制
Integer.toString(int,r);