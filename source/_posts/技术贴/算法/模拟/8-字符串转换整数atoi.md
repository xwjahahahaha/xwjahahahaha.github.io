---
title: 8.字符串转换整数atoi
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-01-14 22:21:16
---

## 题目描述

[8. 字符串转换整数 (atoi)](https://leetcode-cn.com/problems/string-to-integer-atoi/)

难度中等

请你来实现一个 `myAtoi(string s)` 函数，使其能将字符串转换成一个 32 位有符号整数（类似 C/C++ 中的 `atoi` 函数）。

函数 `myAtoi(string s)` 的算法如下：

- 读入字符串并丢弃无用的前导空格
- 检查下一个字符（假设还未到字符末尾）为正还是负号，读取该字符（如果有）。 确定最终结果是负数还是正数。 如果两者都不存在，则假定结果为正。
- 读入下一个字符，直到到达下一个非数字字符或到达输入的结尾。字符串的其余部分将被忽略。
- 将前面步骤读入的这些数字转换为整数（即，"123" -> 123， "0032" -> 32）。如果没有读入数字，则整数为 `0` 。必要时更改符号（从步骤 2 开始）。
- 如果整数数超过 32 位有符号整数范围 `[−231, 231 − 1]` ，需要截断这个整数，使其保持在这个范围内。具体来说，小于 `−231` 的整数应该被固定为 `−231` ，大于 `231 − 1` 的整数应该被固定为 `231 − 1` 。
- 返回整数作为最终结果。

**注意：**

- 本题中的空白字符只包括空格字符 `' '` 。
- 除前导空格或数字后的其余字符串外，**请勿忽略** 任何其他字符。

 <!-- more -->

**示例 1：**

```
输入：s = "42"
输出：42
解释：加粗的字符串为已经读入的字符，插入符号是当前读取的字符。
第 1 步："42"（当前没有读入字符，因为没有前导空格）
         ^
第 2 步："42"（当前没有读入字符，因为这里不存在 '-' 或者 '+'）
         ^
第 3 步："42"（读入 "42"）
           ^
解析得到整数 42 。
由于 "42" 在范围 [-231, 231 - 1] 内，最终结果为 42 。
```

**示例 2：**

```
输入：s = "   -42"
输出：-42
解释：
第 1 步："   -42"（读入前导空格，但忽视掉）
            ^
第 2 步："   -42"（读入 '-' 字符，所以结果应该是负数）
             ^
第 3 步："   -42"（读入 "42"）
               ^
解析得到整数 -42 。
由于 "-42" 在范围 [-231, 231 - 1] 内，最终结果为 -42 。
```

**示例 3：**

```
输入：s = "4193 with words"
输出：4193
解释：
第 1 步："4193 with words"（当前没有读入字符，因为没有前导空格）
         ^
第 2 步："4193 with words"（当前没有读入字符，因为这里不存在 '-' 或者 '+'）
         ^
第 3 步："4193 with words"（读入 "4193"；由于下一个字符不是一个数字，所以读入停止）
             ^
解析得到整数 4193 。
由于 "4193" 在范围 [-231, 231 - 1] 内，最终结果为 4193 。
```

**示例 4：**

```
输入：s = "words and 987"
输出：0
解释：
第 1 步："words and 987"（当前没有读入字符，因为没有前导空格）
         ^
第 2 步："words and 987"（当前没有读入字符，因为这里不存在 '-' 或者 '+'）
         ^
第 3 步："words and 987"（由于当前字符 'w' 不是一个数字，所以读入停止）
         ^
解析得到整数 0 ，因为没有读入任何数字。
由于 0 在范围 [-231, 231 - 1] 内，最终结果为 0 。
```

**示例 5：**

```
输入：s = "-91283472332"
输出：-2147483648
解释：
第 1 步："-91283472332"（当前没有读入字符，因为没有前导空格）
         ^
第 2 步："-91283472332"（读入 '-' 字符，所以结果应该是负数）
          ^
第 3 步："-91283472332"（读入 "91283472332"）
                     ^
解析得到整数 -91283472332 。
由于 -91283472332 小于范围 [-231, 231 - 1] 的下界，最终结果被截断为 -231 = -2147483648 。
```

 

**提示：**

- `0 <= s.length <= 200`
- `s` 由英文字母（大写和小写）、数字（`0-9`）、`' '`、`'+'`、`'-'` 和 `'.'` 组成

通过次数388,975

提交次数1,787,680

## 解题思路及代码

![image-20220818164555637](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220818164555637.png)

```go
// 状态机
type stateMachine struct {
    ans int
    sign int
    table [4][4]int
    state int
}

func NewStateMachine() *stateMachine {
    // 横坐标：0: start, 1: signed, 2: in_number, 3: end
    // 纵坐标：0: 空格, 1: 符号, 2: 数字, 3: 其他
    table := [4][4]int{
        [4]int{0, 1, 2, 3}, 
        [4]int{3, 3, 2, 3},
        [4]int{3, 3, 2, 3},
        [4]int{3, 3, 3, 3},
    }
    return &stateMachine{0, 1, table, 0}
}

// 根据当前状态转换到下一个状态, 顺便计算结果值
func (s *stateMachine) getNextState(c byte) {
    // 根据操作转换为纵坐标
    var actionIndex int
    switch{
        case c == ' ':
            actionIndex = 0
        case c == '+' || c == '-':
            actionIndex = 1
        case c >= '0' && c <= '9':
            actionIndex = 2
        default:
            actionIndex = 3
    }
    // 转换状态
    s.state = s.table[s.state][actionIndex]
    // 计算需要的结果
    if s.state == 2 {
        // 状态是数字
        s.ans = s.ans * 10 + int(c -'0')
        // 考虑溢出
        if s.sign == 1 && s.ans > math.MaxInt32 { s.ans = math.MaxInt32 }
        // 注意负数溢出的判断
        if s.sign == -1 && s.ans > -1 * math.MinInt32 { s.ans = -1 * math.MinInt32 }
    }else if s.state == 1 {
        // 状态是符号(并且只可能从start状态转换过来即最开始)
        if c == '-' {
            s.sign = -1         // +号不用变
        }
    }
}

func myAtoi(s string) int {
    // 创建自动状态机
    autoMachine := NewStateMachine()
    for _, c := range s {
        autoMachine.getNextState(byte(c))
    }
    return autoMachine.sign * autoMachine.ans
}

```

二刷

```go
func myAtoi(s string) int {
    newAM := NewStateMachine()
    for i := range s {
        newAM.readNewByte(s[i])
    }

    return newAM.sign * newAM.number
}

type autoMechine struct {
    state int
    table [4][4]int
    sign int
    number int
}

func NewStateMachine() *autoMechine {
    // 横坐标：0: start, 1: signed, 2: in_number, 3: end
    // 纵坐标：0: 空格, 1: 符号, 2: 数字, 3: 其他
    table := [4][4]int{
        [4]int{0, 1, 2, 3}, 
        [4]int{3, 3, 2, 3},
        [4]int{3, 3, 2, 3},
        [4]int{3, 3, 3, 3},
    }
    return &autoMechine{0, table, 1, 0}
}

func (am *autoMechine) readNewByte(c byte) {
    // 判读操作
    var opt int
    switch {
        case c == ' ':
            opt = 0
        case c == '+' || c == '-':
            opt = 1
        case c >= '0' && c <= '9': 
            opt = 2
        default:
            opt = 3
    }
    
    // 转换状态
    am.state = am.table[am.state][opt]

    // 注意要状态转变完之后再计算结果，因为状态的转换有先后序关系
    if am.state == 2 {
        // 计算数
        am.number = am.number * 10 + int(c-'0')
        // 处理溢出
        if am.sign == 1 && am.number > math.MaxInt32 {
            am.number = math.MaxInt32
        }
        if am.sign == -1 && (-1 * am.number) < math.MinInt32 {   
            am.number = -1 * math.MinInt32  // 注意这里要*-1
        }
    }else if am.state == 1 && c == '-' {
        am.sign = -1
    }
}
```

三刷：

```go
var stateTable = [][]int {
    []int{0, 1, 2, 3},
    []int{3, 3, 2, 3},
    []int{3, 3, 2, 3},
    []int{3, 3, 3, 3},
}

// 状态: 0, 1, 2, 3: start, signed, in_number, end
// opt: 0, 1, 2, 3: 空格, +/-, 数字字符, others
func myAtoi(s string) int {
    res := 0
    state, signed := 0, 1
    for _, opt := range s {
        if opt == ' ' {
            if state = stateTable[state][0]; state == 3 {       // 判断状态, 如果是结束状态就立即结束循环
                break
            }   
        }else if opt >= '0' && opt <= '9' {
            if state = stateTable[state][2]; state == 3 {
                break
            }
            num := int(opt-'0')
            if res == 0 {
                res += num
            }else {
                res = res*10 + num 
            }
            // 考虑整数溢出问题
            if signed == 1 && res > math.MaxInt32 {
                res = math.MaxInt32
            }
            if signed == -1 && res > -math.MinInt32 {
                res = -math.MinInt32
            }
        }else if opt == '+' || opt == '-' {
            if state = stateTable[state][1]; state == 3 {
                break
            }
            if opt == '-' {
                signed *= -1
            }
        }else {
            break
        }
    }

    return signed*res
} 
```

