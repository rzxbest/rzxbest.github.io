---
title:  LJ8合并表记录
tags: [面试题]
categories: [华为面试题]
top: 8
date: 2021-11-29 16:35:00
---
# 合并表记录
## 描述

数据表记录包含表索引和数值（int范围的正整数），请对表索引相同的记录进行合并，即将相同索引的数值进行求和运算，输出按照key值升序进行输出。


提示:
0 <= index <= 11111111
1 <= value <= 100000

## 输入描述：

先输入键值对的个数n（1 <= n <= 500）
然后输入成对的index和value值，以空格隔开
## 输出描述：

输出合并后的键值对（多行）

## 代码实现
```

import java.util.*;

/**
 * author renzhengxiao
 */
public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        int num = scanner.nextInt();
        scanner.nextLine();
        List<Integer > keys = new ArrayList<>();
        Map<Integer,Integer> datamap = new HashMap<>();
        for (int i = 0; i < num; i++) {
            String s = scanner.nextLine();
            String[] s1 = s.split(" ");
            Integer key = Integer.parseInt(s1[0]);
            int value =Integer.parseInt(s1[1]);
            if (datamap.get(key)!=null){
                datamap.put(key,datamap.get(key) + value);
            }else{
                datamap.put(key,value);
                keys.add(key);
            }
        }
        keys.sort(new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
                return o1.compareTo(o2);
            }
        });
        for (Integer key : keys) {
            System.out.println(key + " " +datamap.get(key));
        }
    }
}

```