---
layout:     post
title:      一道简单题
subtitle:   (Easy_Problem_from_Rujia_Liu?UVa_11991)
date:       2019-09-22
author:     fffzlfk
header-img: img/girl.jpg
catalog: true
tags:
    - 抽象数据类型(ADT)
    - C++
---

### 题目描述
给出一个包含n个整数的数组，你需要回答若干询问。每次询问两个整数k和v，输出从左到右第k个v的下标(数组从1～n编号)。

### 输入格式
多组数据。每组数据第一行为n和m(1 <= n, m <= 100000)，第二行为n个整数(n不超过1e6)；以下m行每行为k和v。

### 输出格式
对于每个查询，输出查询结果。若不存在，输出0。

### 代码实现
```cpp
#include <cstdio>
#include <vector>
#include <map>

using namespace std;

map<int, vector<int>> a;
int main(int argc, char const *argv[])
{
	int n, m, x, y;
	while (scanf("%d%d", &n, &m) == 2) {
		for (int i = 0; i < n; i++) {
			scanf("%d", &x);
			if (!a.count(x))
				a[x] = vector<int>();
			a[x].push_back(i + 1);
		}
		while (m--) {
			scanf("%d%d", &x, &y);
			if (a.count(y) || (int)a[y].size() >= x)
				printf("%d\n", a[y][x-1]);
			else
				printf("0\n");
		}
	}
	return 0;
}
```

### 复杂度分析

预处理每个元素时间复杂度为O(logn)(map查找)，总时间复杂度为O(nlogn)。
