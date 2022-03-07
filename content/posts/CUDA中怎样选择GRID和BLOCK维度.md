---
title: "CUDA中怎样选择GRID和BLOCK维度"
date: 2022-03-07T16:10:36+08:00
draft: true
tags:
    - CUDA
---

## 硬件限制

这是容易量化的方面。目前[CUDA编程指南](https://www.nvidia.cn/docs/IO/51635/NVIDIA_CUDA_Programming_Guide_1.1_chs.pdf)的附录F列出了一些硬件限制，这些限制限制了内核启动时每块可以有多少个线程。如果你超过了这些限制，你的内核将无法运行。这些限制可以粗略地概括为：
- 每个区块不能超过 $512$ / $1024$ 个线程（分别是计算能力`1.x`或`2.x`及以后的计算能力
- 每个块的最大尺寸限制在 $[512, 512, 64]$ / $[1024, 1024, 64]$（计算能力`1.x`/`2.x`及以后的计算能力
- 每个块消耗的寄存器总数不能超过 $8k/16k/32k/64k/32k/64k/32k/64k$ （计算能力 $1.0,1.1/1.2,1.3/2.x-3.0/3.2/3.5-5.2/5.3/6-6.1/6.2/7.0$
- 每个块不能消耗超过 $16kb/48kb/96kb$ 的共享内存（计算能力 $1.x/2.x-6.2/7.0$

如果你保持在这些限制之内，任何你能成功编译的内核都会无错误地启动。

## 性能调教

这是需要经验的一部分。在上述的硬件约束条件下，你选择的每块线程数可以而且确实影响到硬件上运行的代码性能。每个代码的表现都是不同的，唯一真正的方法是通过仔细的基准测试和剖析来量化它。但还是那句话，非常粗略地总结一下：

- 每个区块的线程数应该是`wrap`大小的整数倍，在目前所有的硬件上都是 $32$
- `GPU`上的每个流式多处理器必须有足够的`active wraps`来充分隐藏架构的的所有不同内存和指令流水线延迟，以实现最大吞吐量。这里的正确做法是尝试实现最佳的硬件占用率

## CUDA内置函数

上述指出了块的大小是如何影响性能的，并提出了一种基于占用率最大化的通用启发式选择方法。在不想提供选择块大小的标准的情况下，值得一提的是，`CUDA 6.5+`包括几个新的运行时函数来帮助占用率的计算和启动配置[^1]。

[^1]: <https://developer.nvidia.com/blog/cuda-pro-tip-occupancy-api-simplifies-launch-configuration/>

其中一个有用的函数是`cudaOccupancyMaxPotentialBlockSize`，它启发式地计算了一个能达到最佳占用率的块大小。该函数提供的值可以作为手动优化参数的起点。下面是一个例子：

```cpp
/************************/
/* TEST KERNEL FUNCTION */
/************************/
__global__ void MyKernel(int *a, int *b, int *c, int N) 
{ 
    int idx = threadIdx.x + blockIdx.x * blockDim.x; 

    if (idx < N) { c[idx] = a[idx] + b[idx]; } 
} 

/********/
/* MAIN */
/********/
void main() 
{ 
    const int N = 1000000;

    int blockSize;      // The launch configurator returned block size 
    int minGridSize;    // The minimum grid size needed to achieve the maximum occupancy for a full device launch 
    int gridSize;       // The actual grid size needed, based on input size 

    int* h_vec1 = (int*) malloc(N*sizeof(int));
    int* h_vec2 = (int*) malloc(N*sizeof(int));
    int* h_vec3 = (int*) malloc(N*sizeof(int));
    int* h_vec4 = (int*) malloc(N*sizeof(int));

    int* d_vec1; cudaMalloc((void**)&d_vec1, N*sizeof(int));
    int* d_vec2; cudaMalloc((void**)&d_vec2, N*sizeof(int));
    int* d_vec3; cudaMalloc((void**)&d_vec3, N*sizeof(int));

    for (int i=0; i<N; i++) {
        h_vec1[i] = 10;
        h_vec2[i] = 20;
        h_vec4[i] = h_vec1[i] + h_vec2[i];
    }

    cudaMemcpy(d_vec1, h_vec1, N*sizeof(int), cudaMemcpyHostToDevice);
    cudaMemcpy(d_vec2, h_vec2, N*sizeof(int), cudaMemcpyHostToDevice);

    float time;
    cudaEvent_t start, stop;
    cudaEventCreate(&start);
    cudaEventCreate(&stop);
    cudaEventRecord(start, 0);

    cudaOccupancyMaxPotentialBlockSize(&minGridSize, &blockSize, MyKernel, 0, N); 

    // Round up according to array size 
    gridSize = (N + blockSize - 1) / blockSize; 

    cudaEventRecord(stop, 0);
    cudaEventSynchronize(stop);
    cudaEventElapsedTime(&time, start, stop);
    printf("Occupancy calculator elapsed time:  %3.3f ms \n", time);

    cudaEventRecord(start, 0);

    MyKernel<<<gridSize, blockSize>>>(d_vec1, d_vec2, d_vec3, N); 

    cudaEventRecord(stop, 0);
    cudaEventSynchronize(stop);
    cudaEventElapsedTime(&time, start, stop);
    printf("Kernel elapsed time:  %3.3f ms \n", time);

    printf("Blocksize %i\n", blockSize);

    cudaMemcpy(h_vec3, d_vec3, N*sizeof(int), cudaMemcpyDeviceToHost);

    for (int i=0; i<N; i++) {
        if (h_vec3[i] != h_vec4[i]) { printf("Error at i = %i! Host = %i; Device = %i\n", i, h_vec4[i], h_vec3[i]); return; };
    }

    printf("Test passed\n");
}
```

`cudaOccupancyMaxPotentialBlockSize`定义在`cuda_runtime.h`文件中：

```cpp
template<class T>
__inline__ __host__ CUDART_DEVICE cudaError_t cudaOccupancyMaxPotentialBlockSize(
    int    *minGridSize,
    int    *blockSize,
    T       func,
    size_t  dynamicSMemSize = 0,
    int     blockSizeLimit = 0)
{
    return cudaOccupancyMaxPotentialBlockSizeVariableSMem(minGridSize, blockSize, func, __cudaOccupancyB2DHelper(dynamicSMemSize), blockSizeLimit);
}
```

这些参数的含义如下：

- `minGridSize`：返回实现最佳潜在占用率所需的最小网格大小
- `blockSize`：建议的块大小，以实现最大的占用率
- `func`：内核函数
- `dynamicSMemSize`：每块动态共享内存的预定使用量，以字节为单位
- `blockSizeLimit`：最大区块大小，0意味着没有限制。

## References

- <https://stackoverflow.com/questions/9985912/how-do-i-choose-grid-and-block-dimensions-for-cuda-kernels>

- <https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__HIGHLEVEL.html#group__CUDART__HIGHLEVEL_1gee5334618ed4bb0871e4559a77643fc1>