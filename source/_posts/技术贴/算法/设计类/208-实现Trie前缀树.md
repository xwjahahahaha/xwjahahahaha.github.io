---
title: 208-实现Trie前缀树
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-05-12 10:03:12
---

## 题目描述

[208. 实现 Trie (前缀树)](https://leetcode-cn.com/problems/implement-trie-prefix-tree/)

难度中等

**[Trie](https://baike.baidu.com/item/字典树/9825209?fr=aladdin)**（发音类似 "try"）或者说 **前缀树** 是一种树形数据结构，用于高效地存储和检索字符串数据集中的键。这一数据结构有相当多的应用情景，例如自动补完和拼写检查。

请你实现 Trie 类：

- `Trie()` 初始化前缀树对象。
- `void insert(String word)` 向前缀树中插入字符串 `word`。
- `boolean search(String word)` 如果字符串 `word` 在前缀树中，返回 `true`（即，在检索之前已经插入）；否则，返回 `false` 。
- `boolean startsWith(String prefix)` 如果之前已经插入的字符串 `word` 的前缀之一为 `prefix` ，返回 `true` ；否则，返回 `false` 。

<!-- more --> 

**示例：**

```
输入
["Trie", "insert", "search", "search", "startsWith", "insert", "search"]
[[], ["apple"], ["apple"], ["app"], ["app"], ["app"], ["app"]]
输出
[null, null, true, false, true, null, true]

解释
Trie trie = new Trie();
trie.insert("apple");
trie.search("apple");   // 返回 True
trie.search("app");     // 返回 False
trie.startsWith("app"); // 返回 True
trie.insert("app");
trie.search("app");     // 返回 True 
```

**提示：**

- `1 <= word.length, prefix.length <= 2000`
- `word` 和 `prefix` 仅由小写英文字母组成
- `insert`、`search` 和 `startsWith` 调用次数 **总计** 不超过 `3 * 104` 次


## 解题思路及代码

https://leetcode-cn.com/problems/implement-trie-prefix-tree/solution/trie-tree-de-shi-xian-gua-he-chu-xue-zhe-by-huwt/

核心比较巧妙的是，使用trie前缀树节点本身没有记录字符，那是如何保存一个字符串的呢？

其使用链接的索引隐式的定义了对应的字符：

![来自算法4](http://xwjpics.gumptlu.work/qinniu_uPic/e3c98484881bd654daa8419bcb0791a2b6f8288b58ef50df70ddaeefc4084f48-file_1575215107950.png)

```go
type Trie struct {
    isEnd bool
    children [26]*Trie
}


/** Initialize your data structure here. */
func Constructor() Trie {
    return Trie{}
}


/** Inserts a word into the trie. */
func (this *Trie) Insert(word string)  {
    node := this
    for _, c := range word {
        i := c - 'a'
        if node.children[i] == nil {
            // 不存在就创建
            node.children[i] = new(Trie)
        }
        // 向下遍历
        node = node.children[i]
    }
    node.isEnd = true
}


/** Returns if the word is in the trie. */
func (this *Trie) Search(word string) bool {
   node := this
   for _, c := range word {
       i := c - 'a'
       if node.children[i] == nil {
           return false
       }
       node = node.children[i]
   }
   // 注意: 返回最后一个单词的结束字符而不是直接返回true
   return node.isEnd
}


/** Returns if there is any word in the trie that starts with the given prefix. */
// 判断 Trie 中是或有以 prefix 为前缀的单词
func (this *Trie) StartsWith(prefix string) bool {
    node := this
    for _, c := range prefix {
        i := c - 'a'
        if node.children[i] == nil {
            return false
        }
        node = node.children[i]
    }
    // 注意: 这里就是直接返回true,因为只需要判断前缀存在即可,后续是否还有不用考虑
    return true
}


/**
 * Your Trie object will be instantiated and called as such:
 * obj := Constructor();
 * obj.Insert(word);
 * param_2 := obj.Search(word);
 * param_3 := obj.StartsWith(prefix);
 */
```

二刷：

```go
type Trie struct {
    isEnd bool                      // 标记是否是叶子结点(最后一个字母)
    childrens [26]*Trie             // 向下26个字母的映射
}


func Constructor() Trie {
    return Trie{false, [26]*Trie{}}
}


func (this *Trie) Insert(word string)  {
    node := this
    for _, c := range word {
        // 如果没有创建这个字母的子节点，就创建
        if node.childrens[c-'a'] == nil {
            node.childrens[c-'a'] = &Trie{false, [26]*Trie{}}
        }
        // 如果已经有了（之前有单词有相同的前缀）,那么就迭代（其实都要迭代）
        node = node.childrens[c-'a']
    }
    // 字母全部创建，此时node在最后一个字符对应的节点上，所以设置为true
    node.isEnd = true
}


func (this *Trie) Search(word string) bool {
    node := this
    for _, c := range word {
        // 当前字符没有索引，返回false
        if node.childrens[c-'a'] == nil {
            return false
        }
        node = node.childrens[c-'a']
    }

    // 如果到达最后一个字符但是节点不是end，那么也返回false
    return node.isEnd
}


func (this *Trie) StartsWith(prefix string) bool {
    node := this
    for _, c := range prefix {
        // 当前字符没有索引，返回false
        if node.childrens[c-'a'] == nil {
            return false
        }
        node = node.childrens[c-'a']
    }

    // 和search差不多，但是不用判断end
    return true
}


/**
 * Your Trie object will be instantiated and called as such:
 * obj := Constructor();
 * obj.Insert(word);
 * param_2 := obj.Search(word);
 * param_3 := obj.StartsWith(prefix);
 */
```

