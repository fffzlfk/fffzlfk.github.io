---
title: "OpenCV Canny Detector"
date: "2022-01-17T11:35:18+08:00"
tags:
    - OpenCV
---

## 理论

**Canny边缘检测**是由John F. Canny在1986年开发的。许多人也将其称为最佳检测器，Canny算法旨在满足三个主要标准。

- **好的检测**：算法能够尽可能多地标识出图像中的实际边缘。
- **好的定位**：标识出的边缘要与实际图像中的实际边缘尽可能接近。
- **最小响应**：图像中的边缘只能标识一次，并且可能存在的图像雜訊不应标识为边缘。

### 步骤

1. **降噪**：使用高斯滤波来达到，下面是一个大小为 $5$ 的高斯核的例子：
$$
K = \frac{1}{159}\begin{bmatrix}
2 & 4 & 5 & 4 & 2\\\\
4 & 9 & 12 & 9 & 4\\\\
5 & 12 & 15 & 12 & 5\\\\
4 & 9 & 12 & 9 & 4\\\\
2 & 4 & 5 & 4 & 2
\end{bmatrix}
$$

2. **找到图像的亮度梯度**：为此，我们遵循一个类似于`Sobel`的程序：
    1. 应用一对卷积`masks`（在 $x$ 和 $y$ 方向上）：
    $$
    G_x = \begin{bmatrix}
    -1 & 0 & +1\\\\
    -2 & 0 & +2\\\\
    -1 & 0 & +1
    \end{bmatrix}, G_y=\begin{bmatrix}
    -1 & -2 & -1\\\\
    0 & 0 & 0\\\\
    +1 & +2 & +1
    \end{bmatrix}
    $$
    2. 寻找梯度强度和方向：
    $$
    G=\sqrt{G_x^2+G_y^2}\\\\
    \theta = \arctan(\frac{G_y}{G_x})
    $$
    方向被四舍五入为四个可能的角度之一（即 $0\degree$ 、 $45\degree$  、 $90\degree$ 或 $135\degree$ ）。

3. **过滤非最大值**：在高斯滤波过程中，边缘有可能被放大了。这个步骤使用一个规则来过滤不是边缘的点，使边缘的宽度尽可能为1个像素点：如果一个像素点属于边缘，那么这个像素点在梯度方向上的梯度值是最大的。否则不是边缘，将灰度值设为 $0$ 。
$$
M_T(m, n) = \begin{cases}
M(m, n)& \text{if } M(m, n) \lt T\\\\
0 & \text{otherwise}
\end{cases}
$$

4. **滞后**：这是最后一步，Canny确实使用了两个阈值（上限和下限）。
    1. 如果一个像素的梯度值高于上限阈值，则该像素被接受为边缘。
    2. 如果一个像素的梯度值低于下限阈值，那么它将被拒绝。
    3. 如果像素梯度在两个阈值之间，那么只有当它与高于上层阈值的像素相连时，才会被接受。
    Canny建议上下限的比值在 $2:1$ 和 $3:1$ 之间。

## Code

[代码链接](https://github.com/fffzlfk/opencv_learning/blob/main/src/transformations/canny.cpp)

### Explanation

#### 变量定义

```cpp
Mat src, src_gray;
Mat dst, detected_edges;
int lowThreshold = 0;
const int max_lowThreshold = 100;
const int ratio = 3;
const int kernel_size = 3;
const char* window_name = "Edge Map";
```

- 我们建立了一个 $3:1$ 的下限:上限阈值的比值（用可变比值）。
- 我们设定内核大小为 $3$ （用于`Canny`函数内部进行的`Sobel`运算）
- 我们将下限阈值的最大值设定为 $100$ 。

#### 加载图像

```cpp
CommandLineParser parser( argc, argv, "{@input | fruits.jpg | input image}" );
src = imread( samples::findFile( parser.get<String>( "@input" ) ), IMREAD_COLOR ); // Load an image
if( src.empty() )
{
  std::cout << "Could not open or find the image!\n" << std::endl;
  std::cout << "Usage: " << argv[0] << " <Input image>" << std::endl;
  return -1;
}
```

#### 创建一个与src相同类型和大小的矩阵（dst）。

```cpp
dst.create( src.size(), src.type() );
```

#### 灰度化

```cpp
cvtColor( src, src_gray, COLOR_BGR2GRAY );
```

#### 创建一个窗口来显示结果

```cpp
namedWindow( window_name, WINDOW_AUTOSIZE );
```

#### 创建一个Trackbar，让用户为我们的Canny检测器输入低阈值

```cpp
createTrackbar( "Min Threshold:", window_name, &lowThreshold, max_lowThreshold, CannyThreshold );
```

#### 让我们一步一步地观察CannyThreshold函数

- 首先，我们用内核大小为 $3$ 的滤波器对图像进行模糊处理。

    ```cpp
    blur( src_gray, detected_edges, Size(3,3) );
    ```

- 接着，我们应用OpenCV函数`cv::Canny`

    ```cpp
    Canny( detected_edges, detected_edges, lowThreshold, lowThreshold*ratio, kernel_size );
    ```
    - `detected_edges`：源图像（灰度图）
    - `detected_edges`：输出图像（可与输入相同）
    - `lowThreshold`：用户移动`Trackbar`时输入的值
    - `highThreshold`：在程序中设置为`lowThreshold`的 $3$ 倍（按照Canny的建议）
    - `kernel_size`：我们将其定义为 $3$ （内部使用的`Sobel`内核的大小）。

#### 我们用零来填充dst图像（意味着图像是完全黑色的）

```cpp
dst = Scalar::all(0);
```

#### 显示

最后，我们将使用函数`cv::Mat::copyTo`只映射图像中被识别为边缘的区域（在黑色背景上）。`cv::Mat::copyTo`将`src`图像复制到`dst`上。注意，它只会复制像素的非零值的位置。因为Canny检测器的输出是黑色背景上的边缘轮廓，所以除了检测到的边缘，生成的`dst`将是黑色的。
>若没有这一步，那么我们得到的图像是边缘的灰度图

## 结果

![](https://i.imgur.com/4M66qk0.png)

## References

<https://docs.opencv.org/4.5.5/da/d5c/tutorial_canny_detector.html>
