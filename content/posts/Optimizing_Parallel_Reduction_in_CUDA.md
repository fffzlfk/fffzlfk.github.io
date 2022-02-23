---
title: "Optimizing Parallel Reduction in CUDA"
date: 2022-02-21T23:06:33+08:00
tags:
    - CUDA
draft: true
---

## Parallel Reduction

- 在每个线程块中使用基于树的方法
![](https://i.imgur.com/TErxqgm.png)
- 需要能够使用多个线程块
    - 去处理非常大的数组
    - 为了让`GPU`上所有多处理器保持忙碌
    - 每个线程块都`reduce`了数组的一部分
- 但是我们如何在线程块之间交流部分结果呢？

## 问题: 全局同步

- 如果我们能够在所有的线程块中进行同步，就可以很容易地`reduce`非常大的数组，对吗？
    - 在每个线程块产生结果后进行全局同步
    - 一旦所有线程块达到同步，继续递归
- 但是`CUDA`没有全局同步，为什么？
    - 为很多处理器的`GPU`建立硬件的成本很高
    - 将迫使程序员运行较少数量的块以避免死锁，这可能会降低整体效率
- 解决方法：分解成多个内核
    - 内核启动作为一个全局同步点
    - 内核启动的硬件开销可忽略不计，软件开销较低

## 解决方法：内核分解

- 通过分解计算来避免全局同步
![](https://i.imgur.com/yTUW816.png)
- 在`reduce`中，所有级别的代码是相同的
- 递归的内核调用

## 我们的优化目标是什么

- 我们应该努力达到`GPU`的峰值性能
- 选择正确的衡量标准
    - `GFLOP/s`：用于计算型内核
    - 带宽：用于内存型内核
- `Reduction`的算术强度非常低
    - 每个元素占一个`FLOP`（带宽最佳）
- 因此我们需要力争达到峰值带宽

## Reduction #1: 交错式寻址

```cpp
__global__ void reduce0(int *g_idata, int *g_odata) {
    extern __shared__ int sdata[];

    // each thread loads one element from global to shared mem
    unsigned int tid = threadIdx.x;
    unsigned int i = blockIdx.x*blockDim.x + threadIdx.x;
    sdata[tid] = g_idata[i];
    __syncthreads();

    // do reduction in shared mem
    for(unsigned int s=1; s < blockDim.x; s *= 2) {
        if (tid % (2*s) == 0) {
            sdata[tid] += sdata[tid + s];
        }
        __syncthreads();
    }

    // write result for this block to global mem
    if (tid == 0) g_odata[blockIdx.x] = sdata[0];
}
```

![](https://i.imgur.com/XngFi37.png)

![](https://i.imgur.com/1xDoxAA.png)

