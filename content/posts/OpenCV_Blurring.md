---
title: "OpenCV Blurring"
date: "2022-01-09T07:37:09+08:00"
tags:
    - OpenCV
---

## 理论


- `Smoothing`也叫`blurring`（模糊化），是一个简单而常用的图像处理操作。

- `Smoothing`有很多原因。在本教程中，我们将重点讨论平滑操作，以减少噪音。

- 为了进行平滑操作，我们将对我们的图像应用一个`filter`。最常见的`filter`是线性的，其中输出像素的值（即 $g(i,j)$ ）为输入像素值的加权和（即 $f(i+k,j+l)$ ）。

$$
g(i, j) = \sum_{k, l}{f(i+k, j+l)h(k, l)}
$$

- $h(k,l)$ 被称为`kernel`，它只不过是`filter`的系数。这有助于把滤波器想象成一个在图像上滑动的系数窗口。

- 滤波器有很多种类，这里我们将提到最常用的几种。

## Normalized Box Filter（归一化块滤波器）

- 这个滤波器是最简单的，每个输出像素都是其内核邻居的平均值（所有的像素都有相同的权重）。

$$
K = \frac{1}{K_{width} \cdot k_{height}}
\begin{bmatrix}
1 & 1 & 1 & ...& 1\\\\
1 & 1 & 1 & ...& 1\\\\
. & . & . & ...& 1\\\\
1 & 1 & 1 & ...& 1
\end{bmatrix}
$$

## Gaussian Filter（高斯滤波器）

- 可能是最有用的滤波器（尽管不是最快的）。高斯滤波是通过用高斯内核对输入阵列中的每个点进行卷积，然后将它们全部相加来产生输出阵列的。

- 让我们回顾一下`1D Gaussian kernel`是什么样子：
![](https://docs.opencv.org/4.5.5/Smoothing_Tutorial_theory_gaussian_0.jpg)

- 假设图像是一维的，你可以注意到，位于中间的像素会有最大的权重。它的邻居的权重随着它们与中心像素之间的空间距离增加而减少。

>一个二维高斯可以表示为：
>$$
>G_0(x, y)=Ae^{\frac{-(x-\mu_x)^2}{2\sigma_x^2}+\frac{-(y-\mu_y)^2}{2\sigma_y^2}}
>$$
>其中 $\mu$ 是平均值（峰值）， $\sigma^2$ 代表方差（每一个变量 $x$ 和 $y$ ）。

## Bilateral Filter（双边滤波器）

- 到目前为止，我们已经解释了一些主要目标是平滑输入图像的滤波器。然而，有时这些滤波器不仅能消除噪声，还会使边缘变得平滑。为了避免这种情况（至少在一定程度上），我们可以使用一个双边滤波器。

- 类似于高斯滤波器的方式，双边滤波器也考虑了相邻的像素，并为每个像素分配权重。这些权重有两个组成部分，第一个组成部分与高斯滤波器使用的权重相同。第二部分考虑到了相邻像素和被评估像素之间的强度差异。

## Code

- 这个程序是做什么的？
    - 加载一张图片
    - 应用4种不同的过滤器（在理论中解释过的），并按顺序显示过滤后的图片
- [代码链接](https://github.com/fffzlfk/opencv_learning/blob/main/src/basic/blur.cpp)

### 函数解释

#### Normalized Block Filter

- OpenCV 提供了`blur`函数来用这个滤波器进行平滑处理。
    - `src`：源图像
    - `dst`: 目的图像
    - `ksize: Size(w, h)`: 定义要使用的内核的大小（宽度为 $w$ 像素，高度为 $h$ 像素）。
    - `anchor: Point(-1, -1)`: 表示锚点（被评估的像素）相对于邻域的位置。如果有一个负值，那么内核的中心被认为是锚点。
    

#### Gaussian Filter

- function`GaussianBlur`
    - `src`：源图像
    - `dst`: 目的图像
    - `ksize: Size(w, h)`: 要使用的核的大小（要考虑的邻居）。 $w$ 和 $h$ 必须是奇数和正数，否则大小将用 $\sigma_x$ 和 $\sigma_y$ 参数计算
    - `sigmaX`：x的标准偏差。写成 $0$ 意味着用核大小计算的。
    - `sigmaY`：y的标准差。写 $0$ 意味着是用内核大小计算的。

#### Median Filter（中值滤波器）

- function`medianBlur`
    - `src`：源图像
    - `dst`: 目的图像
    - `ksize`：内核的大小（只有一个数，因为我们使用的是方形窗口），必须是奇数。

#### Bilateral Filter（双边滤波）

- function`bilateralFilter`
    - `src`：源图像
    - `dst`: 目的图像
    - `d`: 每个像素邻域的直径
    - `sigmaColor`：色彩空间的标准偏差
    - `sigmaSpace`：坐标空间中的标准偏差


## 结果

### 原图

![](https://s3.bmp.ovh/imgs/2022/01/f5549e957ee61360.png)

### 归一化块滤波

![](https://s3.bmp.ovh/imgs/2022/01/af6bea85825437db.png)

### 高斯滤波

![](https://s3.bmp.ovh/imgs/2022/01/7c940b8963215a69.png)

### 中值滤波

![](https://s3.bmp.ovh/imgs/2022/01/c7a7643fbe5ac8a1.png)

### 双边滤波

![](https://s3.bmp.ovh/imgs/2022/01/787448c87c53ff10.png)

## References

<https://docs.opencv.org/4.5.5/d7/da8/tutorial_table_of_content_imgproc.html>
