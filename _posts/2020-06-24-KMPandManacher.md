---
layout:     post
title:      KMP & Manacher
subtitle:   模式匹配and最长回文子串
date:       2020-06-24
author:     fffzlfk
header-img: img/2020-06-24.png
catalog: true
tags:
    - C++
    - KMP
    - Manacher
---

### P3375 【模板】KMP字符串匹配
<h4><a href="https://www.luogu.com.cn/problem/P3375">题目链接</a></h4>

#### AC代码
```cpp
#include <bits/stdc++.h>
using namespace std;
const int maxn = 1e6 + 5;
char s1[maxn];
char s2[maxn];
int _next[maxn];

void get_next(int len) {
    int j = 0;
    for (int i = 2; i <= len; i++) {
        while (j && s2[j + 1] != s2[i])
            j = _next[j];
        if (s2[j + 1] == s2[i])
            j++;
        _next[i] = j;
    }
}

void find(int len1, int len2) {
    int j = 0;
    for (int i = 1; i <= len1; i++) {
        while (j && s2[j + 1] != s1[i])
            j = _next[j];
        if (s2[j + 1] == s1[i])
            j++;
        if (j == len2) {
            printf("%d\n", i - len2 + 1);
            j = _next[j];
        }
    }
}

int main() {
    cin >> s1 + 1;
    cin >> s2 + 1;
    int len1 = strlen(s1 + 1);
    int len2 = strlen(s2 + 1);
    get_next(len2);
    find(len1, len2);
    for (int i = 1; i <= len2; i++)
        printf("%d ", _next[i]);
    return 0;
}
```

### P3805 【模板】manacher算法
<h4><a href="https://www.luogu.com.cn/problem/P3805">题目链接</a></h4>

#### AC代码
```cpp
#include <bits/stdc++.h>
using namespace std;
const int maxn = 1.1e7 + 5;
char s[maxn];
char ns[maxn * 2];
int p[maxn * 2];
void init(int len) {
    ns[0] = '$';
    ns[1] = '#';
    int k = 2;
    for (int i = 0; i < len; i++) {
        ns[k++] = s[i];
        ns[k++] = '#';
    }
    ns[k] = '\0';
}

int manacher(int len) {
    int idx;
    int max_len = -1;
    int mx = 0;
    for (int i = 1; i < len; i++) {
        if (i < mx)
            p[i] = min(p[2 * idx - i], mx - i);
        else
            p[i] = 1;
        while (ns[i - p[i]] == ns[i + p[i]])
            p[i]++;
        if (mx < i + p[i]) {
            idx = i;
            mx = i + p[i];
        }
        max_len = max(max_len, p[i] - 1);
    }
    return max_len;
}

int main() {
    cin >> s;
    int len = strlen(s);
    init(len);
    cout << manacher(strlen(ns)) << '\n';
    return 0;
}
```
