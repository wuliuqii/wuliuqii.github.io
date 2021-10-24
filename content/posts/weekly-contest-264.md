---
title: "Weekly Contest 264"
date: 2021-10-24T16:01:03+08:00
categories: [leetcode]
tags: [golang, contest]
draft: true
---

# 第 264 场周赛

https://leetcode-cn.com/contest/weekly-contest-264?utm_campaign=weekly_contest_2021_cider_264&utm_medium=leetcode_message&utm_source=message&gio_link_id=Y9JBn239

## 句子中的有效单词

https://leetcode-cn.com/problems/number-of-valid-words-in-a-sentence/

### 思路

直接模拟

### 代码

```go
func countValidWords(sentence string) int {
	res := 0
	tokens := strings.Fields(sentence) // 按照空格划分
	for _, token := range tokens {
		if isValid(token) {
			res++
		}
	}
	return res
}

func isValid(token string) bool {
	if strings.ContainsAny(token, "0123456789") { // 不能有数字
		return false
	}

	n := len(token)
	i := strings.IndexByte(token, '-')
	if i >= 0 && (strings.Contains(token[i+1:], "-") || // 最多只能有一个连字符, 且左右都为小写字母
		i == 0 || i == n-1 || !unicode.IsLetter(rune(token[i-1])) || !unicode.IsLetter(rune(token[i+1]))) {
		return false
	}

	flag := false // 是否有多个符号
	for _, s := range "!.," {
		if i := strings.IndexRune(token, s); i >= 0 {
			if flag || i != n-1 {
				return false
			}
			flag = true
		}
	}

	return true
}
```

## 下一个更大的数值平衡数

https://leetcode-cn.com/problems/next-greater-numerically-balanced-number/

### 思路

暴力求解，直接枚举。

凡是能暴力求解的，应该首先尝试暴力再想办法优化。

### 代码

```go
func nextBeautifulNumber(n int) int {
	for n++; ; n++ {
		if isBeautifulNumber(n) {
			return n
		}
	}
}

func isBeautifulNumber(n int) bool {
	cnt := make([]int, 10)
	for x := n; x > 0; x /= 10 {
		cnt[x%10]++
	}
	for x := n; x > 0; x /= 10 {
		if cnt[x%10] != x%10 {
			return false
		}
	}
	return true
}
```

## 统计最高分的节点数目

https://leetcode-cn.com/problems/count-nodes-with-the-highest-score/

### 思路

对于一个节点 v， 删除与 v 相连的边后，剩余两部分：

1.   以 v 的子节点为根的子树
2.   去除以 v 为根的子树的剩余部分

### 代码

```go
func countHighestScoreNodes(parents []int) int {
	n := len(parents)
	g := make([][]int, n)
	for w := 1; w < n; w++ { // 重建二叉树
		v := parents[w]
		g[v] = append(g[v], w)
	}

	maxScore := 0
	ans := 0
	var dfs func(int) int // 计算以当前节点为根的子树的大小
	dfs = func(v int) int {
		size, score := 1, 1
		for _, w := range g[v] { // 左右子树大小
			sz := dfs(w)
			size += sz
			score *= sz
		}
		if v > 0 { // 去除以当前节点为根的子树后的剩余部分
			score *= n - size
		}
		if score > maxScore {
			maxScore = score
			ans = 1
		} else if score == maxScore {
			ans++
		}
		return size
	}

	dfs(0)
	return ans
}
```

## 并行课程 III

https://leetcode-cn.com/problems/parallel-courses-iii/

### 思路

拓扑排序 + 动态规划
$$
f[i]=\textit{time}[i] + \max f[j]
$$

#### 拓扑排序

