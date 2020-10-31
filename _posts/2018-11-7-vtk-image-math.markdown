---
layout: post
title:  "VTK之图像的计算"
date:   2018-11-7
categories: VTK
tags: VTK C++ image
mathjax: false
author: wzx
---

- 目录
{:toc}

VTK里提供了许多对一个或多个图像的像素进行计算的类




## 数学计算
通过 *vtkImageMathematics* 类来对图像进行一元或二元计算
### 一元计算

```c++
auto imageMath = vtkSmartPointer<vtkImageMathematics>::New();
imageMath->SetInputConnection(reader->GetOutputPort());
imageMath->SetOperationToCos();
```
#### 更多
- SetOperationToInvert()  
图像像素值取倒数运算

- SetOperationToSin()  
图像像素值正弦运算

- SetOperationToCos()  
图像像素值余弦运算

- SetOperationToExp()  
图像像素值自然指数运算

- SetOperationToLog()  
图像像素值自然对数运算

- SetOperationToAbsoluteValue()  
图像像素值取绝对值

- SetOperationToSquare()  
图像像素值平方运算

-	SetOperationToSquareRoot()  
图像像素值平方根运算

-	SetOperationToATAN()  
图像像素值反正切运算

-	SetOperationToATAN2()  
图像像素值二元反正切运算

- SetOperationToMultiplyByK()  
图像像素值乘以常数K，需要先调用SetConstantK()设置K值

- SetOperationToAddConstant()  
图像像素值加上常数K，需要先调用SetConstantK()设置K值

-	SetOperationToReplaceCByK()  
将图像中像素为C的像素值替换为K,需要先调用 SetConstantK() 和 SetConstantC() 设置K和C值

### 二元计算
二元数学计算要求两个输入图像具有相同的像素数据类型和颜色组分。当两个图像大小不同时，输出图像的范围为两个输入图像范围的并集，并且原点和像素间隔与第一个输入图像保持一致
```c++
auto imageMath = vtkSmartPointer<vtkImageMathematics>::New();
imageMath->SetInputConnection(0, reader1->GetOutputPort());
imageMath->SetInputConnection(1, reader2->GetOutputPort());
imageMath->SetOperationToAdd();
```
#### 更多
- SetOperationToAdd()  
两个图像对应像素加法运算

- SetOperationToSubtract()  
两个图像对应像素减法运算

- SetOperationToMultiply()  
两个图像对应像素相乘运算

- SetOperationToDivide()  
两个图像对应像素相除运算

- SetOperationToConjugate()  
将两个标量图像对应像素组合为共辄复数

- SetOperationToComplexMultiply()  
两个图像对应像素复数乘法运算

- SetOperationToMin()  
取两个图像对应像素中的较小值

- SetOperationToMax()  
取两个图像对应像素中的较大值

## 逻辑运算
与数学运算差不多
```c++
auto imageLogic = vtkSmartPointer<vtkImageLogic>::New();
imageLogic->SetInputConnection(0, imageSource1->GetOutputPort());
imageLogic->SetInputConnection(1, imageSource2->GetOutputPort());
//  异或运算
imageLogic->SetOperationToXor();
// 设置结果为True的像素值
imageLogic->SetOutputTrueValue(200);
```
### 更多
- SetOperationToAnd()  
逻辑与

- SetOperationToOr()   
逻辑或

- SetOperationToXor()  
逻辑异或

- SetOperationToNand()  
逻辑与非

- SetOperationToNor()  
逻辑或非

- SetOperationToNot()  
逻辑非

## 图像二值化
二值图像的每个像素只有两个可能的取值，即 0 和 255。而二值化就是设置一个灰度阀值，将阀值以上的像素值设置为 255，其余设置为 0.具体再在VTK中可以通过 *vtkImageThreshold* 类实现
```c++
auto threshold = vtkSmartPointer<vtkImageThreshold>::New();
threshold->SetInputConnection(reader->GetOutputPort());

// 方法一：大于此阀值为 in value
threshold->ThresholdByUpper(128);

// 方法二：小于此阀值为 in value
threshold->ThresholdByLower(128);

// 方法三：在区间内为 in value
threshold->ThresholdBetween(10, 200);

threshold->SetInValue(255);
threshold->SetOutValue(0);
```
