---
layout: post
title:  "VTK的数据存储与表达"
date:   2018-10-23
categories: VTK
tags: VTK
mathjax: false
author: wzx
---

- 目录
{:toc}

vtkDataArray是一个即可定义为动态也可定义为静态的数组，每一个元素是一个元组（Tuple），与Python里的元组不同这里的元组可以改变元组里的元素，但不可以改变元组的大小。  
vtkDataArray只是一个抽象类，具体使用应是其子类。以vtkPolyData为例，里面的很多数据都是以vtkDataArray的子类形式存在的。





## vtkDataArray
### 动态数组
```c++
auto array = vtkSmartPointer<vtkFloatArray>::New();
// 设定每个tumple的大小
array->SetNumberOfComponents(2);

// 最好使用这种方法插值
array->InsertNextTuple2(1.0,2.0);
array->InsertNextTuple2(1.0,2.0);;

//改变array,第1个tumple中第二个值为0.5
array->SetComponent(0, 1, 0.5);
//改变array,第2个tumple的值为(0.5,0.5)
array->SetTuple2(1, 0.5, 0.5);

// 取值，这个方法取到的是tumple的指针(如果NumberOfComponents大于1)
array->GetTuple2(0);
auto p = array->GetTuple2(0);
for (int i = 0; i < 2; ++i) {
    // 遍历操作
    p++;
}
// 取值，这个方法取到的是第1个tumple里的第二个值
array->GetComponent(0, 1);

// maxid目前为3,你们想想是为什么？没有插值时，值为-1
array->GetMaxId();
```

### 静态数组
```c++
auto array = vtkSmartPointer<vtkFloatArray>::New();

array->SetNumberOfComponents(2);
//设定tuple的个数
array->SetNumberOfTuples(2);

//改变或初始化数据,array中的数默认是随机生成的
array->SetComponent(0, 1, 0.5);
array->SetTuple2(1, 0.5, 0.5);

// 取值与动态数组一样

// maxid目前为19,你们知道为什么了吗？
cout<<array->GetMaxId();
```

## vtkFieldData
vtkFieldData是一个数组的数组，数组里每个元素都是一个数组，数组的类型、长度、元组的大小和名称都可以各不相同。一般用来存储数据对象的属性数据
