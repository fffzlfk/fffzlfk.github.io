---
title: "使用Rust实现Helang😅"
date: 2022-08-19T21:44:22+08:00
---

## 引入

最近b站上很多小伙伴对何同学的错误代码进行了一些很有意思的二创，一度登上了Github Trending：

![vsZ7DO.png](https://s1.ax1x.com/2022/08/19/vsZ7DO.png)

---

其中看到有人使用C++中的宏实现了何同学的“~~或运算~~”，于是受到启发，使用Rust中的宏实现了一下。

## Source Code

```rust
fn parse_to_vec(nums: &str) -> Vec<usize> {
    let nums = nums.replace(' ', "");
    nums.split('|').map(|x| x.parse().unwrap()).collect()
}

fn power_con(powers: &mut [u8], nums: &[usize], power: u8) {
    for &num in nums {
        powers[num] = power;
    }
}

#[macro_export]
macro_rules! powerCon {
    ($powers: expr, $nums: expr, $force: expr) => {
        power_con($powers, &parse_to_vec(stringify!($nums)), $force)
    };
}

fn main() {
    let mut powers = [0_u8; 68];
    powerCon!(&mut powers, 0, 100);
    powerCon!(&mut powers, 1 | 2 | 6 | 7 | 11 | 52 | 57 | 58 | 65, 10);
    println!("{:?}", powers)
}
```

## References

- <https://github.com/kifuan/helang/blob/master/helang.cpp>

- <https://www.bilibili.com/video/BV1zW4y1h7YH?share_source=copy_web&vd_source=083fa967106cc2114fabad75979818b9>