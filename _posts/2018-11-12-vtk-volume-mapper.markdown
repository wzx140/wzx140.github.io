---
layout: post
title:  "VTK volume mapper"
date:   2018-11-12
categories: VTK
tags: VTK C++ volume
mathjax: false
author: wzx
---

- 目录
{:toc}

所有 体绘制Mapper 都继承自 *vtkVolumeMapper*
![]({{ site.url }}/assets/img/2018-11-12-1.png)





## vtkFixedPointVolumeRayCastMapper
> ~~*vtkVolumeRayCastMapper*~~ 已经在 vtk8 之后被删除，可用 *vtkFixedPointVolumeRayCastMapper* 替代

*光线投射法*是最常用的体绘制方法。它是一种基于图像序列的直接体绘制方法，其基本
原理是从投影图像平面（通常为平面）的每个像素沿着视线方向发射一条穿过体数据的射线，
然后在射线上按照一定的步长进行等距采样，对每个采样点采用插值技术来计算其体素值，
根据颜色传输函数和不透明度传输函数来获取相应的颜色值和不透明度，最后利用光线吸收
模型将颜色值进行累加直至光线穿过体数据，即可得到当前平面像素的渲染颜色，生成最终
的显示图像。光线投射法的优点是能够比较精确地模拟原始体数据，但计算量比较大，实时
体绘制对计算机硬件的要求比较高。  如果你还是一脸懵逼可以阅读 **[光线投射法](https://blog.csdn.net/liu_lin_xm/article/details/4850609)**  
*vtkFixedPointVolumeRayCastMapper* 可以设置多线程，但是如果线程数大于1, 结果可能不一致
### 混合模式
#### Composite
`SetBlendModeToComposite ()`  
该方式通过Alpha合成技术生成每个像素的颜色值。对于每条光线在穿过体数据时，先根据设置的采样步长进行采样，通过插值技术来计算每个采样点的像素值；然后根据vtkVolumeProperty中设置颜色传输函数和不透明度传输函数来计算采样点的颜色和不透明度。最后，对所有采样点采用Alpha合成方法计算最终的颜色
#### Maximum and minimum intensity
`SetBlendModeToMaximumIntensity ()` `SetBlendModeToMinimumIntensity ()`  
主要用于对体数据中高灰度值得结构进行可视化。当光线穿过体数据时，在光线上进行等距采样。取采样点中属性最大(小)值为该条光线的输出。然后根据vtkVolumeProperty中设置颜色传输函数和不透明度传输函数来计算采样点的颜色和不透明度,**该方法常用于显示血管的三维结构**
#### Additive
`SetBlendModeToAdditive ()`  
每个值传递到不透明度传递函数，然后将值与其不透明度的乘积相加来累积标量值。换句话说，使用不透明度传递函数对标量值进行缩放，并求和以得出最终颜色。请注意，生成的图像始终为灰度，即聚合值不会通过颜色传递函数传递。这是因为最终值是派生值，而不是沿采样光线的实际数据值。

### 采样步长与距离
- 通过 `SetSampleDistance (float)` 来设置每条射线的采样步长，默认值为1，单位为世界坐标系单位，**数值越小越精细**

- 通过 `SetImageSampleDistance (float)` 来设置图像采样距离，默认值为1，即投射光线的间隔，**数值越小越精细**，**调用前需通过`SetAutoAdjustSampleDistances (int)` 关闭自动设置距离**

- 通过 `SetMinimumImageSampleDistance (float)` 和 `SetMaximumImageSampleDistance (float)` 来设置自动调节采样距离时，最大与最小采样距离

## vtkOpenGLGPUVolumeRayCastMapper
由于光线投射体绘制算法计算量非常大，不利于实时渲染，所以 *vtkOpenGLGPUVolumeRayCastMapper* 使用了图形硬件利用纹理映射来加速，其他的和 *vtkFixedPointVolumeRayCastMapper* 差不多

### 混合模式
拥有 *vtkFixedPointVolumeRayCastMapper* 的所有混合模式

#### Average intensity
`SetBlendModeToAverageIntensity ()`  
工作方式和additive类似，其中标量值乘以不透明度传递函数计算的不透明度，然后相加。这里的附加步骤是将总和除以通过体积采集的样本数。与additive的情况一样，最终图像将始终为灰度，即聚合值不通过颜色传递函数。这是因为结果值是导出值而不是沿采样射线的实际数据值

## vtkSmartVolumeMapper
可以根据渲染参数以及硬件信息，自适应选择特定的 体绘制Mapper

### 模式
通过 `SetRequestedRenderMode(int)` 来指定模式

#### vtkSmartVolumeMapper::DefaultRenderMode
允许 *vtkSmartVolumeMapper* 根据渲染参数和硬件支持选择最佳映射器。如果支持GPU光线投射，则此Mapper将用于所有渲染。如果没有，但支持3D纹理映射，那它将用于交互式渲染，并且 *vtkFixedPointRayCastMapper* 将用于静态渲染。如果不支持3D纹理映射，则将独占使用 *vtkFixedPointRayCastMapper*。这是默认请求的渲染模式，通常是最佳选择

#### vtkSmartVolumeMapper::RayCastAndTextureRenderMode
使用3D纹理贴图进行交互式渲染，使用 *vtkFixedPointVolumeRayCastMapper* 进行静态渲染。如果不支持3D纹理贴图，则将专门使用 *vtkFixedPointVolumeRayCastMapper*

#### vtkSmartVolumeMapper::RayCastRenderMode
使用 *vtkFixedPointVolumeRayCastMapper* 进行交互式渲染和静态渲染

#### vtkSmartVolumeMapper::TextureRenderMode
使用3D纹理贴图进行所有渲染，如果不支持则不会渲染任何图像

#### vtkSmartVolumeMapper::GPURenderMode
使用 *vtkGPUVolumeRayCastMapper* 进行所有渲染，如果不支持则不会渲染任何图像
