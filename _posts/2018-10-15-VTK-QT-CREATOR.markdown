---
layout: post
title:  "VTK + QT Creator开发环境搭建"
date:   2018-10-15
categories: VTK
keywords: QT VTK CMake QT-Creator
mathjax: true
author: wzx
---

经历了各种报错后，历经千辛万苦终于搭建好了**VTK8.1+QT5.1+QT Creator4.5**的开发环境，先上图。
![]({{ site.url }}/assets/img/2018-10-15-1.png){:height="300" width="500"}





## 准备工作
> 由于在VTK官网上只提供源码，所以则需要我们自己编译，安装  

**注意文中所有目录请对照自己的安装目录修改**

鉴于vs非常庞大，QT Creator是轻量级的，杀鸡焉用宰牛刀呢，而且vs只是通过插件实现了qt designer的功能，还不如直接使用QT Creator。
1. 下载[VTK源码](https://www.vtk.org/download/)
2. 安装[CMAKE](https://cmake.org/download/)，下载exe文件，一路next，注意安装目录不要有中文
3. 安装[qt及qt creator](http://download.qt.io/official_releases/qt/)，安装过程参见[QT开发环境搭建]({% post_url 2018-10-10-QT-start %})

## 开始编译

### 准备工作
> 没错开始编译前还有准备工作

1. 添加**MinGW**环境变量，将 *path-to-mingw\bin*
 添加到环境变量中，不会添加环境变量的自行百度。
2. 在你要安装的磁盘下生成以下目录
```
.
├── vtk_bin           #存放二进制文件
├── vtk_release       #存放编译好的文件
├── vtk_res           #临时存放
└── vtk_src           #存放源文件，vtk源码
```
3. 将解压后的vtk文件复制到**vtk_src**下

### CMAKE
如下图所示操作：  
![]({{ site.url }}/assets/img/2018-10-15-2.png){:height="500" width="800"}
点击**Finish**  
之后会出现红色的条目，我们依次修改就行，按照下图所示修改。  
**注：若没有图所示变量，则再次点击Configure后操作。目录值要以你安装的目录为准，我的仅供参考**

#### build
![]({{ site.url }}/assets/img/2018-10-15-5.png)

#### type设置为Release
![]({{ site.url }}/assets/img/2018-10-15-6.png)

#### 修改Qt5_DIR
在这之前需要勾选`VTK_Group_Qt`。若出现以下报错不要紧张，按要求修改即可。Configure之后还会出现红色的Qt5Core_DIR之类的，你会发现值已经根据你的填的Qt5_DIR的值自动修改好了，这时再Configure一下就行了。  
![]({{ site.url }}/assets/img/2018-10-15-8.png)

-------

![]({{ site.url }}/assets/img/2018-10-15-9.png)
#### Generate
修改编译目录，默认是放在*C:\Program Files (x86)\VTK*，我们修改在vtk_release文件夹里  
点击**Generate**，若出现**Generating done**，则二进制文件生成在*vtk_bin*里  

![]({{ site.url }}/assets/img/2018-10-15-12.png)  
> 至此为止你可能出现一些其他错误，注意看报错信息，可能是目录写错了或者目录的分隔符打错了导致的，其他错误可以自行百度

### make，make install
在开始菜单中打开cmd,切换目录到vtk_bin（不会的，自行百度），执行`mingw32-make`，然后漫长等待，完成后执行`mingw32-make install`，此时*vtk_release*目录下就是我们编译好的文件

## 在QT Creator中使用QVTKOpenGLWidget
> 特别提示：QVTKWdige将在8.1.1版本后弃用，建议使用QVTKOpenGLWidget

* 复制*vtkDir\vtk_release\plugins\designer*下的**libQVTKWidgetPlugin.dll**
至*G:\qt5\5.10.0\mingw53_32\plugins\designer*（根据你qt安装的位置而定，注意别找错了）

* 在QT Creator中新建工程，通过[自定义控件的方法（promote to）]({% post_url 2018-10-15-QT-promote-to %})，在窗口中加入**QVTKOpenGLWidget**继承自**QOpenGLWidget**。注意其头文件为**QVTKOpenGLWidget.h**
![]({{ site.url }}/assets/img/2018-10-15-13.png)

* 以下使用QMake构建工程，若要使用CMake，参考[CLion+QT+VTK开发环境搭建]({% post_url 2018-10-19-vtk-clion-qt %})， [QT Creator与CMake开发环境搭建]({% post_url 2018-10-17-qt-creator-cmake %})

* 修改pro文件，添加以下代码

```
CONFIG +=USE_VTK

INCLUDEPATH += G:\VTK\include\vtk-8.1
win32:LIBS +=G:\VTK\lib\libvtkalglib-8.1.dll.a
//后面还有好多，就是把G:\VTK\lib\下的文件都加载进来
```
我写了一个Python脚本，修改dir变量为对应的你的目录，注意** \\ 的转义**，将控制台打印出来的东西复制上去就行。
```python
import os

if __name__ == '__main__':
    dir = 'G:\\VTK\\lib\\'
    files = os.listdir(dir)
    lib = ''
    for file in files:
        if file.endswith('.a'):
            lib += 'win32:LIBS +=' +dir+ file + '\n'
    print(lib)
```
> python语言是不是很简洁，所以说**生命苦短，我用Python**

* 在窗口类的头文件里加入

```c++
#ifndef INITIAL_OPENGL
#define INITIAL_OPENGL
#include <vtkAutoInit.h>
VTK_MODULE_INIT(vtkRenderingOpenGL2)
VTK_MODULE_INIT(vtkInteractionStyle)
#endif
```

* 在QT Creator中修改左下方的编译类型为`Release`,下面即可运行程序。请开始你的表演。**啥？什么都不会，那就用下面这个demo测试一下吧。**

## Hellow VTK
**这里补充一下需要将 G:\VTK\bin 添加环境变量，不然不能运行**  
**还要将 tiff的bin 目录添加环境变量，不然不能读图片**
```c++
#include "mainwindow.h"
#include "ui_mainwindow.h"

#include <vtkCylinderSource.h>
#include <vtkGenericOpenGLRenderWindow.h>
#include <vtkPolyDataMapper.h>
#include <vtkRenderer.h>
#include <vtkSmartPointer.h>

MainWindow::MainWindow(QWidget* parent)
    : QMainWindow(parent), ui(new Ui::MainWindow) {
  ui->setupUi(this);
  auto cylinderSource = vtkSmartPointer<vtkCylinderSource>::New();
  cylinderSource->SetCenter(0.0, 0.0, 0.0);
  cylinderSource->SetRadius(5.0);
  cylinderSource->SetHeight(7.0);
  cylinderSource->SetResolution(100);

  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(cylinderSource->GetOutputPort());

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);

  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->AddActor(actor);

  auto window = vtkSmartPointer<vtkGenericOpenGLRenderWindow>::New();
  window->AddRenderer(renderer);

  ui->openGLWidget->SetRenderWindow(window);
}

MainWindow::~MainWindow() {
  delete ui;
}
```
结果就是生成我一开始展示的那个圆柱体，可以用鼠标操作

## 学习资源
1. [官方文档](https://www.vtk.org/doc/nightly/html/index.html)
2. [VTK User’s Guide 中文版](https://blog.csdn.net/column/details/vtk-users-guide.html)
3. **[问答网站](http://vtk.1045678.n5.nabble.com/)，强烈推荐里面有很多大神**
