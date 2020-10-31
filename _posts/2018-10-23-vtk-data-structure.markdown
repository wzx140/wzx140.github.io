---
layout: post
title:  "VTK的数据结构"
date:   2018-10-23
categories: VTK
keywords: VTK
mathjax: false
author: wzx
---

![]({{ site.url }}/assets/img/2018-10-23-1.jpg)





## DataSet数据集

将数据对象组织成一种结构并且赋予相应的属性值，就是数据集
![]({{ site.url }}/assets/img/2018-10-23-8.png)
![]({{ site.url }}/assets/img/2018-10-23-9.png)
![]({{ site.url }}/assets/img/2018-10-23-2.png)

### vtkPolyData
> concrete dataset represents vertices（零维的点，即顶点）, lines, polygons, and triangle strips  

*vtkPolyData* 数据是**不规则结构**，如（e）所示。其组成的单元也有多种类型

```c++
//创建点数据 Geometry
vtkSmartPointer<vtkPoints> points = vtkSmartPointer<vtkPoints>::New();
points->InsertNextPoint ( 1.0, 0.0, 0.0 );
points->InsertNextPoint ( 0.0, 0.0, 0.0 );
points->InsertNextPoint ( 0.0, 1.0, 0.0 );

//将创建的点数据加入到vtkPolyData数据里
vtkSmartPointer<vtkPolyData> polydata = vtkSmartPointer<vtkPolyData>::New();
polydata->SetPoints ( points );
```
> 这里的数据集里只有几何结构（Geometry），没有拓扑结构和属性数据

### vtkImageData
*vtkImageData* 类型的数据是**规则**排列在矩形方格中的点和单元的集合，如（a）所示。如果其排列在平面（二维）上，则称此数据集为**像素映射，位图或图像**；如果排列在层叠面（三维）上，则称为**体**。其单元是由一维的线 *(vtkLine)* 、二维的像素 *(vtkPixel)* 或三维的体素 *(vtkVoxel)* 组成的

### vtkRectilinearGrid
*vtkRectilinearGrid* 类型的数据是**排列（不是规则排列欧）**在矩形方格中的点和单元的集合，如（b）所示，**拓扑结构是规则的，但几何结构只有部分是规则的**。其单元由像素或体素组成

### vtkStructuredGrid
*vtkStructuredGrid* 是结构化网络数据，具有**规则的拓扑结构和不规则的几何结构**，其单元没有重叠或交叉，如（c）所示。其单元由四边形 *(vtkQuad)* 或六面体 *(vtkHexahedron)* 组成，常用于有限元分析

### vtkUnstructuredGrid
*vtkUnstructuredGrid* 是非结构化网络，如（f）所示，**其拓扑结构和几何结构都是不规则的**，所有单元类型都可以组成任意组合。在VTK中任意数据集都可用非结构化网格来表达

### vtkUnstructuredPoints
*vtkUnstructuredPoints* 是非结构化点集，如（d）所示，**其几何结构是不规则的，并且没有拓扑结构**

## Topology拓扑结构

拓扑结构由一个个的 *vtkCell*（应具体到其子类）组成，许多的 *vtkCell* 构成的 *vtkCellArray* 就构成了拓扑结构
```c++
//创建三个坐标点
auto points = vtkSmartPointer<vtkPoints>::New();
points->InsertNextPoint ( 1.0, 0.0, 0.0 ); //返回第一个点的ID：0
points->InsertNextPoint ( 0.0, 0.0, 1.0 ); //返回第二个点的ID：1
points->InsertNextPoint ( 0.0, 0.0, 0.0 ); //返回第三个点的ID：2

//每两个坐标点之间分别创建一条线
//SetId()的第一个参数是线段的端点ID，第二个参数是连接的点的ID
auto line0 = vtkSmartPointer<vtkLine>::New();
line0->GetPointIds()->SetId ( 0,0 );
line0->GetPointIds()->SetId ( 1,1 );

auto line1 = vtkSmartPointer<vtkLine>::New();
line1->GetPointIds()->SetId ( 0,1 );
line1->GetPointIds()->SetId ( 1,2 );

auto line2 = vtkSmartPointer<vtkLine>::New();
line2->GetPointIds()->SetId ( 0,2 );
line2->GetPointIds()->SetId ( 1,0 );

//创建Cell数组，用于存储以上创建的线段
auto lines = vtkSmartPointer<vtkCellArray>::New();
lines->InsertNextCell ( line0 );
lines->InsertNextCell ( line1 );
lines->InsertNextCell ( line2 );

auto polydata = vtkSmartPointer<vtkPolyData>::New();

//将点和线加入到数据集中，前者指定数据集的几何，后者指定其拓扑
polydata->SetPoints ( points );
polydata->SetLines ( lines );
```

### Cell单元

一系列有序的点按照指定类型连接所定义的结构就是单元。点连接的顺序定义了单元的拓扑结构，而点的坐标定义了单元的几何结构。

#### 线性单元
![]({{ site.url }}/assets/img/2018-10-23-3.png){:height="500" width="1000"}
![]({{ site.url }}/assets/img/2018-10-23-4.png)

*vtkCell* 的子类即描述了上图中的各种线性结构，图形的名字前加上前缀vtk。
> 如 *vtkPolygon* ,表示多边形，二维的基本类型，是由共面的三个或三个以上的点按逆时针方向的顺序连接定义的。

#### 非线性单元
![]({{ site.url }}/assets/img/2018-10-23-5.png){:height="500" width="1000"}
![]({{ site.url }}/assets/img/2018-10-23-6.png)
*vtkNonLinearCell* 的子类即描述了上图中的各种非线性结构，图形的名字前加上前缀vtk。

## Attribute Data属性数据

属性数据是与数据集的组织结构相关联的信息，主要用于描述数据集的属性特征，对数据集的可视化实质上就是对属性数据的可视化，例如 [利用MC算法提取等值面]({% post_url 2018-10-22-vtk-marching-cubes %}) 就是对CT值的可视化。在VTK中，用 *vtkDataSetAttributes * 的子类 *vtkPointData*,*vtkCellData* 来分别表达几何结构和拓扑结构的属性
![]({{ site.url }}/assets/img/2018-10-23-7.png)

### Scalar标量

标量是数据集里的每个位置具有的单值的数据，只表示数据的大小
```c++
//设置标量
//准备加入点数据的标量值，两个标量值分别为1和2。
auto weights = vtkSmartPointer<vtkDoubleArray>::New();
weights->SetNumberOfValues(2);
weights->SetValue(0, 1);
weights->SetValue(1, 2);
//设置该点数据的标量属性值。
aDataSet->GetPointData()->SetScalars(weights);

//获取对应点的标量
aDataSet->GetPointData()->GetScalars()->GetScalar(id);
```

### Vector矢量
指既有大小也有方向的量，三维方向上用三元组表示为 *(u,v,w)*

### Texture coordinate纹理坐标
为了使物体看起来更加真实，纹理坐标可以将点从笛卡尔坐标空间映射到一维二维三维的纹理控空间中。

### Tensor张量
可以理解成为一个矩阵，零阶的张量是标量是标量，一阶的张量是矢量，二阶的张量是纹理坐标，三阶的张量是三维矩阵。**VTK只支持3x3的对称矩阵对称向量**
