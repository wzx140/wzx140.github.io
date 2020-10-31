---
layout: post
title:  "VTK之体绘制的渲染对象"
date:   2018-11-12
categories: VTK
tags: VTK C++ volume
mathjax: false
author: wzx
---

- 目录
{:toc}

vtkVolume类似几何渲染的vtkActor，需要传入 [*vtkVolumeMapper*]({% post_url 2018-11-12-vtk-volume-mapper %}) 和 *vtkVolumeProperty*






## 不透明度传输函数
不透明度传输函数是一个**分段线性标量**映射函数，利用该函数可将光线投射过程中的采样点灰度值映射为不同的不透明度值，以决定最终的颜色值，通过 *vtkPiecewiseFunction* 来实现  
通过`vtkVolumeProperty::SetScalarOpacity(vtkPiecewiseFunction *function);` 来设置不透明度传输函数

### 断点
断点即规定了该线性函数的转折点

`int 	AddPoint (double x, double y)`
- 返回该断点的索引值，若失败返回-1
- x为自变量，即图像的属性值（一般为灰度值）
- y为映射值，即不透明度

`void 	AddSegment (double x1, double y1, double x2, double y2)`
- 添加两个断点，组成一条线段
- 如果该线段内已存在断点，则该断点会被清除

`void 	RemoveAllPoints ()`
- 清除所有断点

`int 	RemovePoint (double x)`
- 清除自变量为x的断点，返回该断点的索引值，若失败返回-1


### Clamping
`void ClampingOn	()`	`void ClampingOff ()`

- 当Clamping标志为真时（默认），对于小于所有断点的最小映射值时，其映射值为断点的最小映射值；对于大于所有断点的最大映射值时，其映射值为断点的最大映射值。

- 当Clamping标志为假时，断点映射范围之外的，其映射值都为0

## 梯度不透明度函数
梯度不透明度函数将梯度值映射为一个不透明度乘子，从而增强过渡区域的显示效果。通过 *vtkPiecewiseFunction* 来实现。**不同材料的临界区域梯度值比较大，而材料内部的梯度值则比较小**  
用法和不透明度传输函数类似
```c++
//设置梯度不透明度
auto volumeGradientOpacity = vtkSmartPointer<vtkPiecewiseFunction>::New();
volumeGradientOpacity->AddPoint(10, 0.0);
volumeGradientOpacity->AddPoint(90, 0.5);
volumeGradientOpacity->AddPoint(100, 1.0);
volumeProperty->SetGradientOpacity(volumeGradientOpacity);
```
## 颜色传输函数
颜色传输函数将一个标量值映射为一个颜色值。这个颜色值即可以是RGB值，也可以是HSV值，采用 *vtkColorTransferFunction* 来实现  
用法和不透明度传输函数类似
```c++
auto color = vtkSmartPointer<vtkColorTransferFunction>::New();
color->AddRGBPoint(0.000, 0.00, 0.00, 0.00);
color->AddRGBPoint(64.00, 1.00, 0.52, 0.30);
color->AddRGBPoint(190.0, 1.00, 1.00, 1.00);
color->AddRGBPoint(220.0, 0.20, 0.20, 0.20);
// color->AddHSVPoint(190.0, 1.00, 1.00, 1.00);
volumeProperty->SetColor(color);
```
> 如果想要映射颜色为灰度值，可以设置颜色传输函数为 *vtkPiecewiseFunction* 类   
`volumeProperty->SetColor(vtkPiecewiseFunction *function);`

### 获取
因为存在两种形式的传输函数，故在获取传输函数的时候需要获取传输函数类型
```c++
int channel = volumeProperty->GetColorChannels();
if (channel==1) {
    volumeProperty->GetGrayTransferFunction();
} else if (channel == 3) {
    volumeProperty->GetRGBTransferFunction();
}
```

## 多元数据的处理

- 如果多元数据的各个组分是相互独立的，则可以为每个组分单独设置相应的不透明度、梯度不透明度和颜色传输函数。
```c++
void SetColor (int index, vtkPiecewiseFunction *function);
void SetColor (int index, vtkColorTransferFunction *function);
void SetScalarOpacity (int index, vtkPiecewiseFunction *function)
void SetGradientOpacity (int index, vtkPiecewiseFunction *function)
```

- 如果各个组分不独立，若为二元数据，第一元用于定义颜色，第二元用于定义不透明度以及梯度不透明度

- 如果各个组分不独立，若为四元数据，前三元作为RBG颜色，第四元用于定义不透明度以及梯度不透明度

## 光照与阴影
阴影效果主要受 *Ambient* 环境光系数，*Diffuse* 散射光系数，*Specular* 反射光系数，*Specular Power* 高光强度系数 这四个参数影响，一般情况下前三个系数之和为1，除非需要很高的亮度
```c++
volumeProperty->ShadeOn();
volumeProperty->SetAmbient(0.4);
volumeProperty->SetDiffuse(0.6);
volumeProperty->SetSpecular(0.2);
```
- 环境光系数占主导时，阴影效果不明显
- 散射光系数占主导时，显示效果比较粗糙
- 反射光系数占主导时，显示效果比较平滑
- 阴影效果关闭等同于环境光系数为1，其余两个都为0
- 开启阴影效果对 *vtkUnstructuredGrid* 数据无效，对最大密度投影也无效，多元数据也无效

## vtkLODProp3D
对于大数据来说，体绘制很耗时，用户体验很不好。*vtkLODProp3D* 能够支持多个 Mapper，Property和Texture按需渲染  
`int AddLOD(vtkMapper *m, vtkProperty *p, double time)`
- 返回该LOD的ID值，用于修改或删除该LOD

- time值为你估计这个Mapper渲染所需的时间

`void SetAllocatedRenderTime (double t, vtkViewport *vp)`
- vp为vtkRenderer及其子类对象

- 设置分配的渲染时间，若存在预估时间小于该时间的Mapper，则取时间最长的Mapper先渲染；若不存在，则取预估时间最少的Mapper先渲染

- **若存在预估时间为0的Mapper，则优先渲染**
