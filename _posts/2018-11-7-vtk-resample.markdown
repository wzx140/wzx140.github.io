---
layout: post
title:  "VTK之重采样"
date:   2018-11-7
categories: VTK
tags: VTK C++ image
mathjax: false
author: wzx
---

- 目录
{:toc}

图像重采样指对数字图像按所需的像素位置或像素间距重新采样，以构成几何变换后的新图像





## 原理
将输入的离散数字图像重建代表原始图像的连续函数，再按新的像素间距和像素位置进行采样

## 实现
*vtkImageShrink3D* 和 *vtkImageMagnify* 用于升采样和降采样。这里介绍 vtkImageResample 既可以升采样也可以降采样。

```c++
    auto resample = vtkSmartPointer<vtkImageResample>::New();
    resample->SetInputConnection(reader->GetOutputPort());

//    设置图像维度，若为2，则会设置第三个axis的采样率为1
    resample->SetOutputDimensionality(2);
//    设置各个维度的采样率
    resample->SetAxisMagnificationFactor(0, 5);
    resample->SetAxisMagnificationFactor(1, 5);
//    设置插值方式
    resample->SetInterpolationModeToLinear();
```
