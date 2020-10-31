---
layout: post
title:  "VTK的渲染引擎和可视化管线"
date:   2018-10-24
categories: VTK
keywords: VTK
mathjax: false
author: wzx
---

**Rendering Engine**(渲染引擎)负责数据的可视化表达。**Visualization Pipeline**(可视化管线)用于获取或创建数据，处理数据，把数据传递给 *Writer*（写入文件）或 *Rendering Engine* 进行显示




## Visualization Pipeline 可视化管线
*Visualization Pipeline* 通过 **Source——>Filter——>Mapper** 再传递给 *Rendering Engine* 或者是 *Writer* 写入到文件（写入文件时无需在经过 *Mapper* 处理）

### Source
*Source* 的来源有两个，一是用vtk自带的数据（如 *vtkCylinderSource*）[demo]({% post_url 2018-10-15-VTK-QT-CREATOR %}#hellow-vtk)，二是用从外部文件读取的数据（如 *vtkStructuredPointsReader*）[demo]({% post_url 2018-10-22-vtk-marching-cubes %}#%E4%BB%A3%E7%A0%81%E5%AE%9E%E7%8E%B0)。

### Filter
顾名思义，即处理过滤原始数据，数据源既可以是 *Source* ，也可以从经 *Filter* 后的数据。例如 [MC算法]({% post_url 2018-10-22-vtk-marching-cubes %}#%E4%BB%A3%E7%A0%81%E5%AE%9E%E7%8E%B0) 就是一个 *Filter* ，用于从读取的数据中提取等值面

![]({{ site.url }}/assets/img/2018-10-24-3.png)

### Mapper
*Mapper* 是 *Visualization Pipeline* 的终点，同时也是 *Visualization Pipeline* 连接到 *Rendering Engine* 的桥梁

### 三者区别
![]({{ site.url }}/assets/img/2018-10-24-2.png)

### Visualization Pipeline的执行

使用*SetInputConnection()* 和 *GetOutPort()* 连接 *Visualization Pipeline* 后，数据并没有真正的在管线里流动，只有目标的 *Update()* 函数，数据才会流动，例如
```c++
auto reader = vtkSmartPointer<vtkJPEGReader>::New();
reader->SetFileName("test.jpg");
//只有调用了这个函数，reader才会真正读取图像
reader->Update();
```
但是通常，我们不显式调用 *Update()* ,因为在 *Rendering Engine* 最后，当调用 *Render()* 函数时，会一层一层的向下请求 *Update* 。但如果中间需要某些信息，则需显式的调用该方法

![]({{ site.url }}/assets/img/2018-10-24-4.png)  

**如果只是调用了 *SetInputConnection()* 和 *GetOutPort()* 函数，这样只是传递了数据，并没有建立管线，调用 *Render()* 函数时，管线并不会流通，所以需要调用 *Update()* 函数**

## Rendering Engine 渲染引擎
```c++
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
renWin->Render();

// 添加交互样式
auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
interactor->SetRenderWindow(renWin);
auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
interactor->SetInteractorStyle(style);

interactor->Initialize();
interactor->Start();
```
这是一个代码片段，展示了渲染引擎渲染了 *Mapper* 处理过的数据，并显示到窗口中  
**actor——>render——>renderwindow——>interactor**

![]({{ site.url }}/assets/img/2018-10-24-1.png)

> 每个 *vtkRenderWindow* 中可以有多个 *vtkRender*  
*vtkRenderWindowInteractor* (窗口交互器)可以设定交互的样式
