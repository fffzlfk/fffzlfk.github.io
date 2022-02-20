---
title: "OpenCV Erosion Dilatation"
date: "2022-01-09T12:25:24+08:00"
---

## 形态学操作

- 简而言之：一套基于形状处理图像的操作。形态学操作将一个结构化元素应用于输入图像，并生成一个输出图像。
- 最基本的形态学操作是。腐蚀和膨胀。它们有广泛的用途，即
    - 去除噪音
    - 隔离单个元素和连接图像中不同的元素
    - 寻找图像中的强度凹凸点或洞
- 我们将以下面的图像为例，简要地解释膨胀和侵蚀。
![](https://docs.opencv.org/4.5.5/Morphology_1_Tutorial_Theory_Original_Image.png)

### Dilation（膨胀）

- 这种操作包括将图像A与一些`kernel B`进行卷积，内核可以有任何形状或大小，通常是一个正方形或圆形。
- `kernel B`有一个定义的锚点(`anchor point`)，通常是核的中心。
- 当`kernel B`在图像上扫描时，我们计算出被B重叠的最大像素值，并用该最大值替换锚点位置的图像像素。正如你可以推断的那样，这种最大化的操作会使图像中的明亮区域 "增长"（因此被称为膨胀）。
- 膨胀操作： $dst(x, y)=max(x^{'}, y^{'})_{:element(x^{'}, y^{'}) \ne 0} src(x+x^{'}, y+y^{'})$
- 以上面的图像为例。应用膨胀的方法，我们可以得到
![](https://docs.opencv.org/4.5.5/Morphology_1_Tutorial_Theory_Dilation.png)

### 腐蚀

- 它在给定内核的区域内计算局部最小值。当内核B在图像上被扫描时，我们计算出被B重叠的最小像素值，并用该最小值替换锚点下的图像像素。
- 腐蚀操作为：$dst(x, y)=min(x^{'}, y^{'})_{:element(x^{'}, y^{'}) \ne 0} src(x+x^{'}, y+y^{'})$
- 与膨胀的例子类似，我们可以对原始图像应用腐蚀算子（如上图）。你可以在下面的结果中看到，图像的亮区变薄了，而暗区变大了。
![](https://docs.opencv.org/4.5.5/Morphology_1_Tutorial_Theory_Erosion.png)

## Code

```cpp
#include "basic/erosion_dilatation.hpp"

namespace basic {
namespace erosion_dilatation {
namespace impl {
Mat src, erosion_dst, dilation_dst;
int erosion_elem = 0;
int erosion_size = 0;
int dilation_elem = 0;
int dilation_size = 0;
int const max_elem = 2;
int const max_kernel_size = 21;

int work(int argc, char **argv) {
    CommandLineParser parser(argc, argv,
                             "{@input | ./images/LinuxLogo.jpg | input image}");
    src = imread(samples::findFile(parser.get<String>("@input")), IMREAD_COLOR);
    if (src.empty()) {
        cout << "Could not open or find the image!\n" << endl;
        cout << "Usage: " << argv[0] << " <Input image>" << endl;
        return -1;
    }
    namedWindow("Erosion Demo", WINDOW_AUTOSIZE);
    namedWindow("Dilation Demo", WINDOW_AUTOSIZE);
    moveWindow("Dilation Demo", src.cols, 0);
    createTrackbar("Element:\n 0: Rect \n 1: Cross \n 2: Ellipse",
                   "Erosion Demo", &erosion_elem, max_elem, Erosion);
    createTrackbar("Kernel size:\n 2n +1", "Erosion Demo", &erosion_size,
                   max_kernel_size, Erosion);
    createTrackbar("Element:\n 0: Rect \n 1: Cross \n 2: Ellipse",
                   "Dilation Demo", &dilation_elem, max_elem, Dilation);
    createTrackbar("Kernel size:\n 2n +1", "Dilation Demo", &dilation_size,
                   max_kernel_size, Dilation);
    Erosion(0, 0);
    Dilation(0, 0);
    waitKey(0);
    return 0;
}
void Erosion(int, void *) {
    int erosion_type = 0;
    if (erosion_elem == 0) {
        erosion_type = MORPH_RECT;
    } else if (erosion_elem == 1) {
        erosion_type = MORPH_CROSS;
    } else if (erosion_elem == 2) {
        erosion_type = MORPH_ELLIPSE;
    }
    Mat element = getStructuringElement(
        erosion_type, Size(2 * erosion_size + 1, 2 * erosion_size + 1),
        Point(erosion_size, erosion_size));
    erode(src, erosion_dst, element);
    imshow("Erosion Demo", erosion_dst);
}
void Dilation(int, void *) {
    int dilation_type = 0;
    if (dilation_elem == 0) {
        dilation_type = MORPH_RECT;
    } else if (dilation_elem == 1) {
        dilation_type = MORPH_CROSS;
    } else if (dilation_elem == 2) {
        dilation_type = MORPH_ELLIPSE;
    }
    Mat element = getStructuringElement(
        dilation_type, Size(2 * dilation_size + 1, 2 * dilation_size + 1),
        Point(dilation_size, dilation_size));
    dilate(src, dilation_dst, element);
    imshow("Dilation Demo", dilation_dst);
}

} // namespace impl
} // namespace erosion_dilatation
} // namespace basic
```

#### 程序功能

- 加载一个图像（可以是BGR或灰度）。
- 创建两个窗口（一个用于膨胀输出，另一个用于侵蚀）
- 为每个操作创建一组两个`TraceBar`
    -  `erosion_elem` 或 `dilation_elem`（矩形、十字、椭圆）
    - 内核大小`kernel size`

## 结果

![](https://i.imgur.com/4ZpgH4P.png)

## References

<https://docs.opencv.org/4.5.5/db/df6/tutorial_erosion_dilatation.html>