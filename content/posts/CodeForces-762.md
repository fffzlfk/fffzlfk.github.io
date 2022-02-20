---
title: "CodeForces 762"
date: "2021-12-26T17:18:09+08:00"
toc: true
draft: false
summary: "Codeforces Round #762 (Div. 3)"
tags:
    - Rust
    - CodeForces
---

## A - Square String?

### 思路

判断$s[0.. \frac{n}{2}]$ 和 $s[\frac{n}{2}..n]$是否相等

### 代码

```rust
use std::io::stdin;

#[derive(Default)]
struct Scanner {
    buffer: Vec<String>,
}
impl Scanner {
    fn next<T: std::str::FromStr>(&mut self) -> T {
        loop {
            if let Some(token) = self.buffer.pop() {
                return token.parse().ok().expect("Failed parse");
            }
            let mut input = String::new();
            stdin().read_line(&mut input).expect("Failed read");
            self.buffer = input.split_whitespace().rev().map(String::from).collect();
        }
    }
}

fn solve(scan: &mut Scanner) {
    let t: usize = scan.next();
    for _ in 0..t {
        let s: String = scan.next();
        let n = s.len();
        if s[..n / 2].to_owned() == s[n / 2..].to_owned() {
            println!("YES");
        } else {
            println!("NO");
        }
    }
}

fn main() {
    let mut scan = Scanner::default();
    solve(&mut scan);
}
```

## B - Squares and Cubes

### 思路

使用`HashSet`就能过

### 代码

```rust
use std::io::stdin;

#[derive(Default)]
struct Scanner {
    buffer: Vec<String>,
}
impl Scanner {
    fn next<T: std::str::FromStr>(&mut self) -> T {
        loop {
            if let Some(token) = self.buffer.pop() {
                return token.parse().ok().expect("Failed parse");
            }
            let mut input = String::new();
            stdin().read_line(&mut input).expect("Failed read");
            self.buffer = input.split_whitespace().rev().map(String::from).collect();
        }
    }
}

use std::collections::HashSet;
fn solve(scan: &mut Scanner) {
    let t: usize = scan.next();
    for _ in 0..t {
        let n: usize = scan.next();
        let mut st = HashSet::new();
        let c = f64::cbrt(n as f64) as usize;
        let s = f64::sqrt(n as f64) as usize;
        for i in 1..=c {
            st.insert(i * i);
            st.insert(i * i * i);
        }
        for i in c..=s {
            st.insert(i * i);
        }
        println!("{}", st.len());
    }
}

fn main() {
    let mut scan = Scanner::default();
    solve(&mut scan);
}
```

## C - Wrong Addition

### 思路

模拟

### 代码

```rust
use std::io::stdin;

#[derive(Default)]
struct Scanner {
    buffer: Vec<String>,
}
impl Scanner {
    fn next<T: std::str::FromStr>(&mut self) -> T {
        loop {
            if let Some(token) = self.buffer.pop() {
                return token.parse().ok().expect("Failed parse");
            }
            let mut input = String::new();
            stdin().read_line(&mut input).expect("Failed read");
            self.buffer = input.split_whitespace().rev().map(String::from).collect();
        }
    }
}

fn solve(scan: &mut Scanner) {
    let t: usize = scan.next();
    'outer: for _ in 0..t {
        let mut a: usize = scan.next();
        let mut s: usize = scan.next();

        let mut b: Vec<u8> = vec![];

        while s != 0 {
            let mut y = s % 10;
            let x = a % 10;
            if y >= x {
                b.push((y - x) as u8 + b'0');
            } else {
                s /= 10;
                y += 10 * (s % 10);
                if y > x && y >= 10 && y <= 19 {
                    b.push((y - x) as u8 + b'0');
                } else {
                    println!("-1");
                    continue 'outer;
                }
            }
            a /= 10;
            s /= 10;
        }

        if a != 0 {
            println!("-1");
            continue;
        }
        b.reverse();
        let b: usize = String::from_utf8_lossy(&b).parse().unwrap();
        println!("{}", b);
    }
}

fn main() {
    let mut scan = Scanner::default();
    solve(&mut scan);
}
```

## D - New Year's Problem

### 思路

二分答案，注意最多只能visit $n-1$ 个转换为存在一个商店买两个人的礼物。

### 代码

```rust
use std::io::stdin;

#[derive(Default)]
struct Scanner {
    buffer: Vec<String>,
}
impl Scanner {
    fn next<T: std::str::FromStr>(&mut self) -> T {
        loop {
            if let Some(token) = self.buffer.pop() {
                return token.parse().ok().expect("Failed parse");
            }
            let mut input = String::new();
            stdin().read_line(&mut input).expect("Failed read");
            self.buffer = input.split_whitespace().rev().map(String::from).collect();
        }
    }
}

fn solve(scan: &mut Scanner) {
    let n: usize = scan.next();
    let m: usize = scan.next();
    let mut p: Vec<Vec<u32>> = vec![vec![0; m]; n];
    for row in p.iter_mut() {
        for col in row.iter_mut() {
            *col = scan.next::<u32>();
        }
    }

    let (mut l, mut r) = (0, 1e9 as u32);
    while l < r {
        let mid = (l + r + 1) >> 1;
        if check(&p, mid) {
            l = mid;
        } else {
            r = mid-1;
        }
    }

    println!("{}", l);
}

fn check(p: &Vec<Vec<u32>>, mid: u32) -> bool {
    let mut ok = vec![false; p[0].len()];
    let mut has_pair = false;
    for i in 0..p.len() {
        let mut ok_cols = 0;
        for j in 0..p[0].len() {
            if p[i][j] >= mid {
                ok[j] = true;
                ok_cols += 1;
            }
        }
        has_pair = has_pair || ok_cols >= 2;
    }
    has_pair && ok.iter().all(|&x| x)
}

fn main() {
    let mut scan = Scanner::default();
    let t: usize = scan.next();
    for _ in 0..t {
        solve(&mut scan);
    }
}
```
