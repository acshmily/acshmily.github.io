---
title: longest-substring-without-repeating-characters
date: 2019-03-27 00:15:08
tags:
	-	Algorithms
	-	Golang
	-	ARTS
categories:	算法
---

## [无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)

> 给定一个字符串，请你找出其中不含有重复字符的 **最长子串** 的长度。
>
> **示例 1:**
>
> ```
> 输入: "abcabcbb"
> 输出: 3 
> 解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
> ```
>
> **示例 2:**
>
> ```
> 输入: "bbbbb"
> 输出: 1
> 解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
> ```
>
> **示例 3:**
>
> ```
> 输入: "pwwkew"
> 输出: 3
> 解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
>   请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
> ```

### 思路

以示例1为例，可以主观的判断非重复的最长子串应该是 [abc]abcbb，那我们是怎么主观判断比较的呢？

从示例3我们在从人类的角度观察一下，结果应该是pw[wke]w，经过示例1以及示例3我们可以得出，如果源字符串内仅存一个单一的非重复子串，则子串的第一个位置应该与子串结束后源字符串后一个字符串相同，才截断为一个子串

所以我们需要一个变量来统计源字符串最长的子串长度；需要一个Map来储存非重复子串的字符串以及在源字符串中的位置(key:每一个字符 value:在源字符串中出现的位置)；还需要一个游标变量来记录当前子串在源字符串中开始的位置

伪代码大概是这样的

```go
func funcname(s string)int{
  maxlength := 0 // 保存当前非重复子串的最大长度
  start := 0 //游标
  charmap := make(map[string]int) // 声明一个map来存储每一个字符出现的位置索引
  for i:=0;i<len(s);i++{ // 循环每一个字符
    if inx,ok := charmap[s[i]];ok && start <= inx{ // start <= inx 表示当子串开始位置小于等于当前字符在源字符串的位置，换句话说是 游标位置在 字符索引之前或当前
      // 当这个字符已经在map中找的到，也就意味着曾经出现过了，此时子串就应该截断了
      start = i + 1 // 记录下一个子串是从哪儿开始判断
    }
    if i - start + 1 > maxlength { // 比较这个子串是否比之前储存的子串长度要大
      maxlength = i - start + 1 //如果成立则将这个子串长度赋值给maxlength
    }
    charmap[s[i]] = i //更新这个字符串出现的位置
  }
  return maxlength //返回最大长度
}
```

其中 start <= inx  是为了判断游标位置是否在当前字符索引位置之前或当前，用示例1来举例：

abcabcbb，当i =  3的时候 应该是讯轮到abc[a]bcbb在map中找到a曾经出现的索引位置是 0，并且游标之前还在0位置，那游标判断下一个子串开始的位置应该为 [a]的索引位置 - 游标索引位置 + 1 

同时还要判断最大子串长度是否符合，i - start +  1即为截取子串的长度 i 为在当前循环中最后一次出现的相同字符索引位置 - 之前游标标定的开始位置 + 1(因为是计算长度) 

最后更新这个字符串出现的最后位置

[代码实现]: https://github.com/acshmily/ARTS/blob/master/LeetCode/Algorithms/longest-substring-without-repeating-characters/main.go	"基于Golang实现的代码"

