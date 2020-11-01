---
layout: post
title:  "VTK之区域提取"
date:   2018-10-30
categories: VTK
keywords: VTK
mathjax: false
author: wzx
---

vtk如何提取图片中的某个区域




## VOI 提取
*VOI* 即 *Volume of Interest*，感兴趣区域
```c++
// 获取维数
int dims[3];
reader->GetOutput()->GetDimensions(dims);

auto extractVOI = vtkSmartPointer<vtkExtractVOI>::New();
extractVOI->SetInputConnection(reader->GetOutputPort());

// 参数提取区域三个维度坐标的最大值和最小值
extractVOI->SetVOI(dims[0] / 4, dims[0] * 3 / 4., dims[1] / 4, dims[1] * 3 / 4, 0, 0);
extractVOI->Update();
```

## 三维切片提取
```c++
// 将原点(0,0,0)设置为图像中心
auto changer = vtkSmartPointer<vtkImageChangeInformation>::New();
changer->SetInputConnection(reader->GetOutputPort());
changer->SetCenterImage(1);


//切面的变换矩阵
static double axialElements[16] = {
       1, 0, 0, 0,
        0, 1, 0, 0,
       0, 0, 1, 0,
       0, 0, 0, 1
};

auto resliceAxes = vtkSmartPointer<vtkMatrix4x4>::New();
// 将数组变成4x4矩阵
resliceAxes->DeepCopy(axialElements);

// 将(0,3)位置设置为0
//resliceAxes->SetElement(0, 3, 0);


auto reslice = vtkSmartPointer<vtkImageReslice>::New();
reslice->SetInputConnection(changer->GetOutputPort());
// 设置输出数据为二维
reslice->SetOutputDimensionality(2);
// 传入变换矩阵
reslice->SetResliceAxes(resliceAxes);
// 采用线性插值法
reslice->SetInterpolationModeToLinear();
```
### 变换矩阵
**变换矩阵的规则**：矩阵的第一列指定x轴向量（第四个元素必须设置为0），第二列指定y轴，第三列指定z轴。 第四列是轴的原点（第四个元素必须设置为1）。
#### 常见变换矩阵

```c++
// 平行于XY平面的切片
{
  1, 0, 0, 0,
  0, 1, 0, 0,
  0, 0, 1, 0,
  0, 0, 0, 1};

// 平行于XZ平面的切片
{
  1, 0, 0, 0,
  0, 0, 1, 0,
  0,-1, 0, 0,
  0, 0, 0, 1};  

// 平行于YZ平面的切片
{
  0, 0,-1, 0,
  1, 0, 0, 0,
  0,-1, 0, 0,
  0, 0, 0, 1};
```
### 插值方式
```c++
// 采用线性插值法
reslice->SetInterpolationModeToLinear();

// 最近邻插值
reslice->SetInterpolationModeToNearestNeighbor();

// 三次线性插值
reslice->SetInterpolationModeToCubic();
```
