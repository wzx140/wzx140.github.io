---
layout: post
title:  "VTK的图像文件读取与显示"
date:   2018-10-28
categories: VTK
tags: VTK C++ image
mathjax: false
author: wzx
---

- 目录
{:toc}

一些简单的图像读取与显示操作





## 读取图像
### 读取一组图像
读取一张图片很简单，直接`reader->SetFileName(xx)`就可以了。但实际中通常要读取一组三维图像的二维切片，就可以用如下三种方法
#### 传入文件名数组
```c++
//生成图像序列的文件名数组
vtkSmartPointer<vtkStringArray> fileArray = vtkSmartPointer<vtkStringArray>::New();
char fileName[128];
for (int i = 1; i < 100; i++) {
    sprintf(fileName, "../data/Head/head%03d.jpg", i);
    vtkStdString fileStr(fileName);
    fileArray->InsertNextValue(fileStr);
}

//读取JPG序列图像
vtkSmartPointer<vtkJPEGReader> reader = vtkSmartPointer<vtkJPEGReader>::New();
reader->SetFileNames(fileArray);
```

#### 设置文件名类型
```c++
vtkSmartPointer<vtkJPEGReader> reader =
        vtkSmartPointer<vtkJPEGReader>::New();
reader->SetFilePrefix("../data/Head/head");
reader->SetFilePattern("%s%03d.jpg");
//100张图片，255*255
reader->SetDataExtent(0, 255, 0, 255, 1, 100);
reader->Update();
```

#### vtkImageAppend进行拼接
```c++
vtkSmartPointer<vtkImageAppend> append = vtkSmartPointer<vtkImageAppend>::New();
//指定z轴为堆叠方向
append->SetAppendAxis(2);

auto reader =vtkSmartPointer<vtkJPEGReader>::New();
char fileName[128];
for (int i = 1; i < 21; i++) {
    sprintf(fileName, "../data/Head/head%03d.jpg", i);
    reader->SetFileName(fileName);
    append->AddInputConnection(reader->GetOutputPort());
}
```
### 常见图像数据类
#### vtkImageData
[]({{ site.url }}/assets/img/2018-10-28-1.png)
> *vtkDicomImageReader* 不支持多帧 *DICOM* 图像读取，也不能进行写操作。可以使用[DCMTK](http://dcmtk.org)，[ITK](http://www.itk.org)

#### vtkPolyData
![]({{ site.url }}/assets/img/2018-10-28-3.png)
![]({{ site.url }}/assets/img/2018-10-28-2.png)
#### vtkRectilinearGrid
![]({{ site.url }}/assets/img/2018-10-28-4.png)
#### vtkStructuredGrid
![]({{ site.url }}/assets/img/2018-10-28-5.png)
#### vtkUnstructuredGrid
![]({{ site.url }}/assets/img/2018-10-28-6.png)

## 图像的显示
### vtkImageViewer2
比较适合三维图像的切片显示
```c++
auto reader = vtkSmartPointer<vtkMetaImageReader>::New();
//注意构建目录
reader->SetFileName("../res/brain.mhd");
//千万要加这一句
reader->Update();

auto imageViewer = vtkSmartPointer<vtkImageViewer2>::New();
imageViewer->SetInputConnection(reader->GetOutputPort());

auto renderWindowInteractor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
imageViewer->SetupInteractor(renderWindowInteractor);

//设定窗位和窗宽
imageViewer->SetColorLevel(500);
imageViewer->SetColorWindow(2000);
imageViewer->SetSlice(40);

//设定方向 2-xy
imageViewer->SetSliceOrientation(2);
imageViewer->Render();

renderWindowInteractor->Start();
```
> 窗位和窗宽是有关医学图像显示的概念，不同的组织适合不同的窗位和窗宽。[窗宽窗位含义](https://blog.csdn.net/chenhuakang/article/details/79164134)

### vtkImageActor
比 *vtkImageViewer2* 更高端一点
```c++
auto reader = vtkSmartPointer<vtkMetaImageReader>::New();
reader->SetFileName("../res/brain.mhd");
reader->Update();

auto imgActor = vtkSmartPointer<vtkImageActor>::New();
imgActor->GetMapper()->SetInputConnection(reader->GetOutputPort());

auto renderer = vtkSmartPointer<vtkRenderer>::New();
renderer->AddActor(imgActor);
renderer->SetBackground(1.0, 1.0, 1.0);

auto renderWindow = vtkSmartPointer<vtkRenderWindow>::New();
renderWindow->AddRenderer(renderer);

auto renderWindowInteractor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
auto style = vtkSmartPointer<vtkInteractorStyleImage>::New();
renderWindowInteractor->SetInteractorStyle(style);
renderWindowInteractor->SetRenderWindow(renderWindow);
renderWindow->Render();

renderWindowInteractor->Initialize();

renderWindowInteractor->Start();
```
> 接受的图像数据 *VTKImageData* 像素类型必须为 *unsigned char*，可以事先转化

### 图像融合
```c++
auto reader = vtkSmartPointer<vtkJPEGReader>::New();
reader->SetFileName("../res/lena-gray.jpg");
//    reader->Update();

auto imageSource = vtkSmartPointer<vtkImageCanvasSource2D>::New();
imageSource->SetNumberOfScalarComponents(1);
imageSource->SetScalarTypeToUnsignedChar();
imageSource->SetExtent(0, 512, 0, 512, 0, 0);
imageSource->SetDrawColor(0.0);
imageSource->FillBox(0, 512, 0, 512);
imageSource->SetDrawColor(255.0);
imageSource->FillBox(100, 400, 100, 400);
//    imageSource->Update();

auto imageBlend = vtkSmartPointer<vtkImageBlend>::New();
//这里需要注意
imageBlend->AddInputConnection(reader->GetOutputPort());
imageBlend->AddInputConnection(imageSource->GetOutputPort());

imageBlend->SetOpacity(0, 0.4);
imageBlend->SetOpacity(1, 0.6);
//    imageBlend->Update();

auto actor = vtkSmartPointer<vtkImageActor>::New();
actor->GetMapper()->SetInputConnection(imageBlend->GetOutputPort());

auto render = vtkSmartPointer<vtkRenderer>::New();
auto window = vtkSmartPointer<vtkRenderWindow>::New();
auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();

render->AddActor(actor);
window->AddRenderer(render);
interactor->SetRenderWindow(window);

window->Render();
interactor->Initialize();
interactor->Start();
```
