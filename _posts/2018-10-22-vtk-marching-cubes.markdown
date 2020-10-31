---
layout: post
title:  "VTK之Marching Cubes算法"
date:   2018-10-22
categories: VTK
tags: C++ VTK
mathjax: false
author: wzx
---

- 目录
{:toc}

**Marching Cubes**算法用于在三维离散数据场中提取等值面。






## 算法简介

1. 读入三维离散数据场<sup>[1]</sup>，设定等值面的值C0<sup>[2]</sup>。设定若干个小立方体包裹住目标区域，称为体元。  

2. 我们假定某体元一个角点的函数值大于 C0 , 则将该角点赋值为 1, 并称该角点位于等值面之内。如果该角点的函数值小于等值面的值C0 , 则将该角点赋值为零, 并称该角点位于等值面之外。显然, 如果某体元中一条边的一个角点在等值面之内, 而另一个角点在等值面之外, 那么, 该边必然与所求等值面相交。根据这一原理就可以判断所求等值面将与哪些体元相交, 或者说将穿过哪些体元。  

3. 由于每个顶点都有0和1这两个可能的值，所以一个体元中可能的等值面情况是 2<sup>8</sup>=256 种。但是可以用两种对称性将 256 种情况降到 15种 情况。第一，若将体元中值等于1的顶点的值和值等于0的顶点的值进行对换，等值面的情况不变，这样可能的情况变成原来的一半。第二，立方体是中心对称的，根据这个特性情况再次减少，降为15种。如下图所示。
![]({{ site.url }}/assets/img/2018-10-22-1.gif)  

4. 将所有体元立方体生成的三角形拼在一起，就是要的等值面

> [1] 用于医疗诊断的断层扫描仪( CT ) 及核磁共振仪( MRI ) 等产生的图象均属于这一类型  
[2] 该值表示一种阈值，例如在CT中，改变阈值可以过滤出不同的组织

## 代码实现
[head.vtk]({{ site.url }}/assets/vtk_res/head.vtk)
```c++
#include <vtkSmartPointer.h>
#include <vtkStructuredPointsReader.h>
#include <vtkRenderer.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkMarchingCubes.h>
#include <vtkPolyDataMapper.h>
#include <vtkActor.h>
#include <vtkInteractorStyleTrackballCamera.h>

int main(int argc, char *argv[]) {

    //读入Structured_Points类型的vtk文件。
    auto reader = vtkSmartPointer<vtkStructuredPointsReader>::New();
    reader->SetFileName("../res/head.vtk");

    // 用移动立方体法提取等值面。
    auto marchingCubes = vtkSmartPointer<vtkMarchingCubes>::New();
    marchingCubes->SetInputConnection(reader->GetOutputPort());
    marchingCubes->SetValue(0, 500);

    // 将生成的等值面数据进行Mapper
    auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
    mapper->SetInputConnection(marchingCubes->GetOutputPort());

    // 把Mapper的输出送入渲染引擎进行显示
    auto actor = vtkSmartPointer<vtkActor>::New();
    actor->SetMapper(mapper);

    // 渲染
    auto renderer = vtkSmartPointer<vtkRenderer>::New();
    renderer->AddActor(actor);
    renderer->SetBackground(1.0, 1.0, 1.0);

    // 设置渲染窗口
    auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
    renWin->AddRenderer(renderer);

    // 添加交互样式
    auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
    interactor->SetRenderWindow(renWin);
    renWin->Render();
    auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
    interactor->SetInteractorStyle(style);

    interactor->Initialize();
    interactor->Start();

    return 0;
}
```
