---
title: 412.FizzBuzz
tags:
  - java
categories:
  - technical
  - leetcode
toc: true
date: 2020-06-16 22:50:01
---

## 题目描述

写一个程序，输出从 1 到 n 数字的字符串表示。

    1. 如果 n 是3的倍数，输出“Fizz”；

    2. 如果 n 是5的倍数，输出“Buzz”；

    3. 如果 n 同时是3和5的倍数，输出 “FizzBuzz”。

示例：

n = 15,

返回:
```
[
    "1",
    "2",
    "Fizz",
    "4",
    "Buzz",
    "Fizz",
    "7",
    "8",
    "Fizz",
    "Buzz",
    "11",
    "Fizz",
    "13",
    "14",
    "FizzBuzz"
]
```

>来源：力扣（LeetCode）

>链接：https://leetcode-cn.com/problems/fizz-buzz

>著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

<!-- more -->

## 解题思路

时间复杂度上无法更好的优化，但是在代码的编写上可以优化，为以后的修改采用更加“优雅”的方式

## 代码

```java
// 直观的思路： 求余数  时间复杂度O(N) 空间复杂度O(1)
class Solution {
    public List<String> fizzBuzz(int n) {
        if(n < 1) return null;
        List<String> list = new ArrayList<>();
        for(int i=1; i<=n; i++){
            if(i % 3 == 0 && i % 5 == 0)
                list.add("FizzBuzz");
            else if(i % 3 == 0)
                list.add("Fizz");
            else if(i % 5 == 0)
                list.add("Buzz");
            else
                list.add(String.valueOf(i));
        }
        
        return list;
    }
}
```

```java
// 拼接字符串  时间复杂度O(N) 空间复杂度O(1)  但是条件多情况下可以减少编程代码复杂度
// 对于多条件的合适解法
// 例如：3 ---> "Fizz" , 5 ---> "Buzz", 7 ---> "Jazz"
// 那么判断的条件就会有很多种很多的if/else，那么拼接字符串的方式会更适合
class Solution {
    public List<String> fizzBuzz(int n) {
        if(n < 1) return null;
        List<String> list = new ArrayList<>();
        for(int i=1; i<=n; i++){
            String resString = "";
            if(i % 3 == 0) resString += "Fizz";
            if(i % 5 == 0) resString += "Buzz";
            // 如果到这还为空，那么说明不能被3或5整除
            if("".equals(resString)) resString += String.valueOf(i);
            list.add(resString);
        }
        return list;
    }
}
```
 
```java 
// 使用散列表保存每个映射关系，方便以后添加/修改    时间复杂度O(N) 空间复杂度O(1)
class Solution {
    public List<String> fizzBuzz(int n) {
        if(n < 1) return null;
        List<String> list = new ArrayList<>();
        // 如果对于FizzBuzz的顺序有要求的话，要用LinkedHashMap
        Map<Integer, String> map = new LinkedHashMap<>();
        map.put(3, "Fizz");
        map.put(5, "Buzz");
        for(int i=1; i<=n; i++){
            String resString = "";
            for(Integer j : map.keySet()){
                if(i % j == 0)
                    resString += map.get(j);
            }
            if("".equals(resString)) resString += String.valueOf(i);
            list.add(resString);
        }
        return list;
    }
}
```

> 作者：gumptlu

> 链接：https://leetcode-cn.com/problems/fizz-buzz/solution/bu-nan-dan-shi-you-liang-chong-hen-hao-de-si-lu-by/

>来源：力扣（LeetCode）

>著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。