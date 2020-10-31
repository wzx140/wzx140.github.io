---
layout: post
title:  "VTK之体绘制渲染管线"
date:   2018-11-11
categories: VTK
tags: VTK C++ volume
mathjax: false
author: wzx
---

- 目录
{:toc}

VTK中体绘制与几何渲染相似，有体绘制管线和渲染引擎
![]({{ site.url }}/assets/img/2018-11-11-1.png)




## 渲染引擎
数据传入 *vtkRenderer* 中，经由 *vtkRenderWindow*，*vtkRenderWindowInteractor* 渲染

## 体绘制管线
1. 由 *vtkDataSet* 为起点，数据向后传输
2. 将 DataSet 经 *vtkVolumeMapper* 处理
3. 将 *vtkVolumeProperty* 和 *vtkVolumeMapper* 的数据传入 *vtkVolume*

[mummy.128.vtk]({{ site.url }}/assets/vtk_res/mummy.128.vtk)
```c++
 auto reader = vtkSmartPointer<vtkStructuredPointsReader>::New();
    reader->SetFileName("../mummy.128.vtk");

//    Adaptive volume mapper
    auto volumeMapper = vtkSmartPointer<vtkSmartVolumeMapper>::New();
    volumeMapper->SetInputConnection(reader->GetOutputPort());

    auto volumeProperty = vtkSmartPointer<vtkVolumeProperty>::New();
//    设置差值方式
    volumeProperty->SetInterpolationTypeToLinear();
//    打开阴影
    volumeProperty->ShadeOn();
    volumeProperty->SetAmbient(0.4);
    volumeProperty->SetDiffuse(0.6);
    volumeProperty->SetSpecular(0.2);

    auto compositeOpacity = vtkSmartPointer<vtkPiecewiseFunction>::New();
    compositeOpacity->AddPoint(70, 0.00);
    compositeOpacity->AddPoint(90, 0.40);
    compositeOpacity->AddPoint(180, 0.60);
//    不透明度传输函数
    volumeProperty->SetScalarOpacity(compositeOpacity);

//    设置梯度不透明度
//    auto volumeGradientOpacity = vtkSmartPointer<vtkPiecewiseFunction>::New();
//    volumeGradientOpacity->AddPoint(10, 0.0);
//    volumeGradientOpacity->AddPoint(90, 0.5);
//    volumeGradientOpacity->AddPoint(100, 1.0);
//    volumeProperty->SetGradientOpacity(volumeGradientOpacity);

    auto color = vtkSmartPointer<vtkColorTransferFunction>::New();
    color->AddRGBPoint(0.000, 0.00, 0.00, 0.00);
    color->AddRGBPoint(64.00, 1.00, 0.52, 0.30);
    color->AddRGBPoint(190.0, 1.00, 1.00, 1.00);
    color->AddRGBPoint(220.0, 0.20, 0.20, 0.20);
//    颜色传输函数
    volumeProperty->SetColor(color);

    auto volume = vtkSmartPointer<vtkVolume>::New();
    volume->SetMapper(volumeMapper);
    volume->SetProperty(volumeProperty);

    auto ren = vtkSmartPointer<vtkRenderer>::New();
    ren->AddVolume(volume);

    auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
    renWin->AddRenderer(ren);

    auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
    auto iren = vtkSmartPointer<vtkRenderWindowInteractor>::New();
    iren->SetRenderWindow(renWin);
    iren->SetInteractorStyle(style);

    iren->Initialize();
    iren->Start();
```
