---
layout: post
title:  "VTK之SmartPointer"
date:   2018-11-1
categories: VTK
keywords: VTK, SmartPointer
mathjax: false
author: wzx
---

VTK 的对象在引用完了之后需要手动 *delete*，这样很不方便并且可能导致内存泄漏。*SmartPointer* 就是为了解决这一问题的，如果对象超出作用域，并且不会在其他地方是用来，那么他将会被自动 *delete*






### 用SmartPointer创建对象
```c++
vtkSmartPointer<vtkObject> MyObject = vtkSmartPointer<vtkObject>::New();
```

### 把现有对象放入SmartPointer
#### 使用TakeReference()
```c++
vtkPolyData *NewMesh()
{
  vtkPolyData *pd = vtkPolyData::New();
  // build the mesh
  // ...
  return pd;
}

vtkSmartPointer<vtkPolyData> MyObject;
MyObject.TakeReference(NewMesh());
```

#### 使用Take()
```c++
auto MyObject = vtkSmartPointer<vtkPolyData>::Take(NewMesh());
```

### 通过SmartPointer获取对象
```c++
auto Reader = vtkSmartPointer<vtkXMLPolyDataReader>::New();
//reader超出作用域且不被引用时，pd会被释放内存
vtkPolyData* pd = Reader->GetOutput();

//当reader和pd同时超出作用域时，pd才会被释放内存
vtkSmartPointer<vtkPolyData> pd = Reader->GetOutput();
```

### 作为函数返回值使用Smart Pointer
```c++
//注意返回类型不要写成普通指针
vtkSmartPointer<vtkPolyData> MyFunction()
{
  auto myObject = vtkSmartPointer<vtkPolyData>::New();
  return myObject;
}
```
### 作为类成员使用Smart Pointer
```c++
class MyClass
{
  vtkSmartPointer<vtkFloatArray> Distances;
};

//初始化方法一
MyClass::MyClass()
: Distances(vtkSmartPointer<vtkFloatArray>::New())
{}

//初始化方法二
MyClass::MyClass()
{
  Distances = vtkSmartPointer<vtkFloatArray>::New();
}
```
参考文献：[SmartPointers](https://www.vtk.org/Wiki/VTK/Tutorials/SmartPointers)
