```bash
---
layout:     post                    # 使用的布局（不需要改）
title:      一个简单的位运算及其复杂度分析(Sample)               # 标题 
subtitle:   sample #副标题
date:       2019-09-03              # 时间
author:     BY                      # 作者
header-img: img/bg.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 编程
---
```

﻿#### 问题：

对于任意非负整数,统计其二进制展开中数位1的总数。
#### 方法：
将n与1位与，若结果为1则将计数器+1，之后将n向右移一位，重复上述过程直至n缩减为0；
例如：
>5转换为二进制：0000 0000 0000 0000 0000 0000 0000 0101
3转换为二进制：0000 0000 0000 0000 0000 0000 0000 0011
1转换为二进制：0000 0000 0000 0000 0000 0000 0000 0001
故 5 & 3 = 1
（位与：第一个操作数的的第n位于第二个操作数的第n位如果都是1，那么结果的第n为也为1，否则为0）

#### 代码实现：
```cpp
#include <iostream>

using namespace std;

int countOnes(unsigned int n) {
	int ones = 0;
	while(n) {
		ones += n & 1;
		n >>= 1;
	}
	return ones;
}
int main() {
	int n;
	while (cin >> n)
	cout << countOnes(n) << endl;
	return 0;
}
```
#### 时间复杂度：
总的循环次数为n展开为二进制后的位数，即$$ 1 + log_2n$$无论是该循环体之前、之内还是之后,均只涉及常数次(逻辑判断、位与运算、加法、右移
等)基本操作。因此,countOnes()算法的执行时间主要由循环的次数决定,亦即:$$ O(1+[log_2n])=O([log_2n])=O(log_2n) $$由大O记号定义,在用函数log r n界定渐进复杂度时,常底数r的具体取值无所谓，故通常不予专门标出而笼统地记作logn，比如,尽管此处底数为常数2,却可直接记作O(logn)。此类算法称作具有“对数时间复杂度”(logarithmic-time algorithm)。
