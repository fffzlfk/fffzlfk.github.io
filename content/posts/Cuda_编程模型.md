---
title: "Cuda 编程模型"
date: "2022-01-19T12:55:19+08:00"
---

## Kernels

`CUDA C++`对`C++`进行了扩展，允许程序员定义`C++`函数，称为内核，当被调用时，由 $N$ 个不同的`CUDA`线程并行执行 $N$ 次，而不是像普通`C++`函数那样只执行一次。

`kernel`是使用`__global__`声明定义的，对于特定的内核调用，执行该内核的`CUDA`线程数量是使用`<<<...>>>`执行配置语法指定的（[C++语言扩展](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#c-language-extensions)）。每个执行内核的线程都有一个唯一的线程`ID`，可以在内核内通过内置变量访问。

作为说明，下面的示例代码，使用内置变量`threadIdx`，将两个大小为 $N$ 的向量 $A$ 和 $B$ 相加，并将结果存入向量 $C$ 。

```cpp
// Kernel definition
__global__ void VecAdd(float* A, float* B, float* C)
{
    int i = threadIdx.x;
    C[i] = A[i] + B[i];
}

int main()
{
    ...
    // Kernel invocation with N threads
    VecAdd<<<1, N>>>(A, B, C);
    ...
}
```

在这里，执行`VecAdd()`的 $N$ 个线程中的每一个都执行了一次加法。

## 线程体系

为方便起见，`threadIdx`是一个 $3$ 分量的向量，因此可以用一维、二维或三维的线程索引来识别线程，形成一个一维、二维或三维的线程块，称为线程块。这提供了一种自然的方式来调用域中的元素进行计算，如矢量、矩阵或体积。

一个例子，下面的代码将两个大小为 $N\times N$ 的矩阵 $A$ 和 $B$ 相加，并将结果存入矩阵 $C$ 。

```cpp
// Kernel definition
__global__ void MatAdd(float A[N][N], float B[N][N],
                       float C[N][N])
{
    int i = threadIdx.x;
    int j = threadIdx.y;
    C[i][j] = A[i][j] + B[i][j];
}

int main()
{
    ...
    // Kernel invocation with one block of N * N * 1 threads
    int numBlocks = 1;
    dim3 threadsPerBlock(N, N);
    MatAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);
    ...
}
```

每个块的线程数量是有限制的，因为一个块的所有线程都要驻留在同一个处理器核心上，并且必须分享该核心的有限内存资源。在目前的`GPU`上，一个线程块最多可以包含 $1024$ 个线程。

然而，一个内核可以由多个形状相同的线程块执行，因此线程的总数等于每个块的线程数乘以块的数量。

块被组织成一个一维、二维或三维的线程块网格，如图所示。网格中的线程块的数量通常由正在处理的数据的大小决定，这通常超过了系统中的处理器数量。

![](https://docs.nvidia.com/cuda/cuda-c-programming-guide/graphics/grid-of-thread-blocks.png)

在`<<<...>>>`中指定的每个块的线程数和每个网格的块数可以是`int`或`dim3`类型。可以像上面的例子那样指定二维块或网格。

网格中的每个块可以通过一个一维、二维或三维的唯一索引来识别，在内核中可以通过内置的`blockIdx`变量访问。线程块的尺寸可以在内核中通过内置的`blockDim`变量访问。

扩展之前的`MatAdd()`例子以处理多个块，代码如下：

```cpp
// Kernel definition
__global__ void MatAdd(float A[N][N], float B[N][N],
float C[N][N])
{
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    int j = blockIdx.y * blockDim.y + threadIdx.y;
    if (i < N && j < N)
        C[i][j] = A[i][j] + B[i][j];
}

int main()
{
    ...
    // Kernel invocation
    dim3 threadsPerBlock(16, 16);
    dim3 numBlocks(N / threadsPerBlock.x, N / threadsPerBlock.y);
    MatAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);
    ...
}
```

线程块大小为 $16 \times 16$ （ $256$ 个线程），尽管在这种情况下是任意的，但却是一种常见的选择。网格是用足够的块创建的，以便像以前一样每个矩阵元素有一个线程。为简单起见，本例假设每个维度的每个网格的线程数被该维度的每个块的线程数平均分割，尽管不一定是这样的。

线程块被要求独立执行。必须能够以任何顺序、平行或串联的方式执行它们。如图所示，这种独立性要求允许线程块在任何数量的内核上以任何顺序进行调度，使程序员能够编写随内核数量扩展的代码。

![](https://docs.nvidia.com/cuda/cuda-c-programming-guide/graphics/automatic-scalability.png)

一个块内的线程可以通过一些**共享内存**来共享数据，并通过同步它们的执行来协调内存访问来进行合作。更确切地说，我们可以通过调用`__syncthreads()`函数来指定内核中的同步点；`__syncthreads()`作为一个障碍，块中的所有线程必须在这个障碍处等待，然后才允许继续进行。共享内存给出了一个使用共享内存的例子。除了`__syncthreads()`之外，合作组`API`还提供了一套丰富的线程同步原语。

共享内存应该是靠近每个处理器核心的低延迟内存（很像`L1`高速缓存），并且`__syncthreads()`应该是轻量级的。

## 内存体系

如图所示，`CUDA`线程在执行过程中可以从多个内存空间访问数据。每个线程都有私有的本地内存。每个线程块都有共享内存，对该块的所有线程都是可见的，并且与该块具有相同的生命周期。所有线程都可以访问相同的**全局**内存。

还有两个额外的只读内存空间可供所有线程访问：**常量**和**纹理**内存空间。全局、常量和纹理内存空间针对不同的内存使用情况进行了优化（见[设备内存访问](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#device-memory-accesses)）。纹理内存还为一些特定的数据格式提供了不同的寻址模式，以及数据过滤（见[纹理和表面内存](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#texture-and-surface-memory)）。

全局、常量和纹理内存空间在同一个应用程序启动内核时是持久的。

![](https://docs.nvidia.com/cuda/cuda-c-programming-guide/graphics/memory-hierarchy.png)

## 异构编程

如图所示，`CUDA`编程模型假设`CUDA`线程在一个物理上独立的设备上执行，该设备作为运行`C++`程序的主机的协处理器运行。例如，当内核在`GPU`上执行，而`C++`程序的其余部分在`CPU`上执行时，就属于这种情况。

![](https://docs.nvidia.com/cuda/cuda-c-programming-guide/graphics/heterogeneous-programming.png)

`CUDA`编程模型还假定主机和设备都在`DRAM`中保持自己独立的内存空间，分别称为主机内存和设备内存。因此，程序通过调用`CUDA`运行时（在[编程接口](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#programming-interface)中描述）来管理内核可见的全局、常量和纹理内存空间。这包括设备内存的分配和撤销，以及主机和设备内存之间的数据传输。

统一内存提供了管理的内存来连接主机和设备内存空间。管理的内存可以从系统中的所有`CPU`和`GPU`访问，作为一个具有共同地址空间的单一、连贯的内存图像。这种能力使设备内存的超额认购成为可能，并且通过消除在主机和设备上显式镜像数据的需要，大大简化了移植应用程序的任务。关于统一内存的介绍，请参见[统一内存编程](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#um-unified-memory-programming-hd)。

## 异步的SIMT编程模型

在`CUDA`编程模型中，线程是进行计算或内存操作的最低级别的抽象概念。从基于`NVIDIA Ampere GPU`架构的设备开始，`CUDA`编程模型通过异步编程模型为内存操作提供加速。异步编程模型定义了与CUDA线程有关的异步操作的行为。

异步编程模型定义了用于CUDA线程之间同步的异步障碍的行为。该模型还解释并定义了`cuda::memcpy_async`如何在GPU中计算时用于从全局内存异步移动数据。

### 异步操作

异步操作被定义为由一个`CUDA`线程发起并由另一个线程异步执行的操作。在一个完善的程序中，一个或多个`CUDA`线程会与异步操作同步。启动异步操作的`CUDA`线程并不需要在同步线程中。

这样的异步线程（`as-if`线程）总是与启动异步操作的`CUDA`线程有关。一个异步操作使用一个同步对象来同步完成操作。这样的同步对象可以由用户显式管理（如`cuda::memcpy_async`）或在库内隐式管理（如`cooperative_groups::memcpy_async`）。

一个同步对象可以是`cuda::barrier`或`cuda::pipeline`。这些对象在[Asynchronous Barrier](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#aw-barrier)和[Asynchronous Data Copies using cuda::pipeline](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#memcpy_async_pipeline)中有详细解释。这些同步对象可以在不同的线程范围内使用。一个范围定义了可以使用同步对象与异步操作同步的线程集合。下表定义了`CUDA C++`中可用的线程作用域以及可以与每个线程同步的线程。

Thread 作用域 | 描述
-- | --
`cuda::thread_scope::thread_scope_thread` | 只有发起异步操作的CUDA线程才会同步
`cuda::thread_scope::thread_scope_block` | 与启动线程在同一线程块内的所有或任何CUDA线程都会同步
`cuda::thread_scope::thread_scope_device` | 与启动线程相同的GPU设备中的所有或任何CUDA线程进行同步
`cuda::thread_scope::thread_scope_system` | 与启动线程在同一系统中的所有或任何CUDA或CPU线程进行同步

这些线程作用域在[CUDA Standard C++](https://nvidia.github.io/libcudacxx/extended_api/thread_scopes.html)库中作为标准C++的扩展来实现。

## References

<https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#programming-model>