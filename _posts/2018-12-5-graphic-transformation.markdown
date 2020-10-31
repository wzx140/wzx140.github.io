---
layout: post
title:  "VTK简单图形处理"
date:   2018-12-5
categories: VTK
keywords: VTK
mathjax: false
author: wzx
---

这里介绍通过vtk库实现的简单图形处理




## 将三维问题转化为二维
将三维数据所在的平面**通过旋转与xy平面平行**，在进行二维计算，最后再旋转回去，这里以由空间两点坐标以及两点坐标的向量生成圆弧为例
```c++
// 添加转换的点和向量
auto points = vtkSmartPointer<vtkPoints>::New();
auto vectors = vtkSmartPointer<vtkDoubleArray>::New();
auto data = vtkSmartPointer<vtkPolyData>::New();
points->InsertNextPoint(stPoint.data());
points->InsertNextPoint(endPoint.data());
data->SetPoints(points);
vectors->SetNumberOfComponents(3);
vectors->InsertNextTuple(stVector.data());
vectors->InsertNextTuple(endVector.data());
data->GetPointData()->SetVectors(vectors);

auto transform = vtkSmartPointer<vtkTransform>::New();
auto filter = vtkSmartPointer<vtkTransformPolyDataFilter>::New();

// 旋转至xy平面
array<double, 3> nAxis = {0};
vtkMath::Cross(stVector.data(), endVector.data(), nAxis.data());
VectorUtil::regularize(nAxis);
double theta = vtkMath::DegreesFromRadians(acos(vtkMath::Dot(nAxis.data(), zAxis)));
array<double, 3> axis = {0};
vtkMath::Cross(nAxis.data(), zAxis, axis.data());
transform->RotateWXYZ(theta, axis.data());
filter->SetTransform(transform);
filter->SetInputData(data);
filter->Update();

// 这里储存转换后的点和向量
array<double, 3> stPointZ{};
array<double, 3> stVectorZ{};
array<double, 3> endPointZ{};
array<double, 3> endVectorZ{};
filter->GetOutput()->GetPoint(0, stPointZ.data());
filter->GetOutput()->GetPoint(1, endPointZ.data());
filter->GetOutput()->GetPointData()->GetVectors()->GetTuple(0, stVectorZ.data());
filter->GetOutput()->GetPointData()->GetVectors()->GetTuple(1, endVectorZ.data());
//        if (stPointZ[2] != endPointZ[2]) {
//            cout << "error" << endl;
//        }
//这是所有点的z坐标应该是相同的
double pointZ = stPointZ[2];


// 进行二维操作
auto verStVectors = VectorUtil::getVerVector(stVectorZ);
auto verEndVectors = VectorUtil::getVerVector(endVectorZ);
auto center = LineUtil::intersection(stPointZ, verStVectors[0], endPointZ, verEndVectors[0]);
center[2] = pointZ;
.........
auto arc = vtkSmartPointer<vtkArcSource>::New();
arc->SetCenter(center.data());
arc->SetPoint1(stPointZ.data());
arc->SetPoint2(endPointZ.data());
arc->SetResolution(resolution);
arc->Update();

// 旋转回去
filter->SetTransform(transform->GetInverse());
filter->SetInputData(arc->GetOutput());
filter->Update();
```


## 两直线交点
由一个点和其方向向量确定一条直线，输入两个点和两个向量来确定交点。先将两直线所在的平面旋转到xy-z平面，在进行二维操作。这里需要注意**考虑斜率不存在的情况**
```c++
array<double, 3> intersection(array<double, 3> &point1, array<double, 3> &vector1, array<double, 3> &point2,
                                  array<double, 3> &vector2) {
        array<double, 3> SecPoint = {0};

        double k1 = vector1[1] / vector1[0];
        double k2 = vector2[1] / vector2[0];

        if (vector1[0] == 0) {
            SecPoint[0] = point1[0];
            SecPoint[1] = k2 * (SecPoint[0] - point2[0]) + point2[1];
            return SecPoint;

        } else if (vector2[0] == 0) {
            SecPoint[0] = point2[0];
            SecPoint[1] = k1 * (SecPoint[0] - point1[0]) + point1[1];
            return SecPoint;
        }

        SecPoint[0] = (point2[1] - point1[1] + k1 * point1[0] - k2 * point2[0]) / (k1 - k2);
        SecPoint[1] = k1 * (SecPoint[0] - point1[0]) + point1[1];

        return SecPoint;

    }
```

## 对圆进行点采样
```c++
for (int i = 0; i < resolution; ++i) {
            double theta = vtkMath::RadiansFromDegrees(360. * i / double(resolution));
            double x = radius * cos(theta);
            double y = radius * sin(theta);
            points->InsertNextPoint(x, y, pointZ);
        }
```

## 求两向量夹角
```c++
double getAngle(const array<double, 3> &vector1, const array<double, 3> &vector2) {
        array<double, 3> tempCross = {0};
        vtkMath::Cross(vector1.data(), vector2.data(), tempCross.data());
        double rad = atan2(vtkMath::Norm(tempCross.data()), vtkMath::Dot(vector1.data(), vector2.data()));
        double deg = vtkMath::DegreesFromRadians(rad);
        return abs(deg);
    }
```
