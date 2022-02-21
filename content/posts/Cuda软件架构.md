---
title: "Cuda软件架构"
date: "2022-01-08T06:27:04+08:00"
tags:
    - Cuda
---

## 硬件

- `SP(Streaming Processor)`：流处理器，是GPU最基本的处理单元，在`fermi`架构开始被叫做`CUDA core`。 

- `SM(Streaming MultiProcessor)`：一个SM由多个`CUDA core`组成。

比如说，如果一个GPU有 $4$ 个`SM`，并且每个`SM`有 $768$ 个`SP`(Aka `CUDA core`)；那么在某一时刻，真正并行运行的线程数不会超过 $4 \times 768$ 个。

## 软件

`threads`被组织成`blocks`。一个`block`的线程可以用`1Dimension(x)`, `2Dimensions(x, y)`或者`3Dim indexs(x, y, z)` 索引，

显然，如果你需要 $4 \times 768$ 个以上的`threads`的话你需要 $4$ 个以上的`blocks`。`blocks`也可以使用`1D`, `2D`或`3D`索引，这些`blocks`被放在等待队列上进入GPU执行。

## Wrap

当一个`kernel`被执行时，`grid`中的线程块被分配到`SM`上。一个`CUDA core`可以执行一个`thread`，一个`SM`的`CUDA core`会分成几个`wrap`，由`wrap scheduler`负责调度。

一个`wrap`中的线程在同一个`block`中，如果`block`所含线程数不是`wrap`的大小的整数倍，那么多出来的那些`thread`所在的`wrap`中，会剩余一些`inactive`的`thread`。

## 一个简单的case

处理一张 $512 \times 512$ 的图片。

假设我们希望一个线程处理一个像素`pixel(i, j)`。

我们可以使用每 $64$ 个线程的区块。所以我们需要 $\frac{512 \times 512 }{64} = 4096$ 个区块（为了拥有 $512 \times 512 $ 个线程 ）。

通常情况下，我们将线程组织在`2D`区块中（为了更容易索引图像像素）。`blockDim`= $8 * 8$ ，我更喜欢叫它`threadsPerBlock`。

```cpp
dim3 threadsPerBlock(8, 8);
```

还有`2D`的`gridDim`= $64 \times 64$ （需要 $4096$ 个区块）。我更喜欢叫它`numBlocks`。

```cpp
dim3 numBlocks(imageWidth/threadPerBlock.x, imageHeight/threadPerBlock.y);
```

这个`kernel`是这样启动的：

```cpp
myKernel <<<numBlocks, threadsPerBlock>>>(/* params for the kernel function */)
```

最后，会有一个类似 $4096$ 个区块的队列的东西，其中一个块正在等待被分配到GPU的一个多处理单元中，以获得其 $64$ 个线程的执行。

在`kernel`中，一个线程要处理的像素是这样计算的：

```cpp
uint i = (blockIdx.x + blockDim.x) + threadIdx.x;
uint j = (blockIdx.y + blockDim.y) + threadIdx.y;
```

![]("https://face2ai.com/CUDA-F-2-0-CUDA%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B%E6%A6%82%E8%BF%B01/4.png" )

## Reference

- [Understanding CUDA grid dimensions, block dimensions and threads organization (simple explanation) [closed]](https://stackoverflow.com/questions/2392250/understanding-cuda-grid-dimensions-block-dimensions-and-threads-organization-s)

- [CUDA的thread,block,grid和warp](https://zhuanlan.zhihu.com/p/123170285)