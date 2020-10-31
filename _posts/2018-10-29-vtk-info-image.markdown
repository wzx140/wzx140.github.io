---
layout: post
title:  "VTK之图像信息的读取与修改"
date:   2018-10-28
categories: VTK
tags: VTK C++ image
mathjax: false
author: wzx
---

- 目录
{:toc}

读取图像的头信息以及像素值，并进行修改。以及有关颜色映射的操作





## 图像信息的访问与修改
```c++
//先读取图像，别忘了update

//获取图像维度，二维的dims[2]值为1
int dims[3];
reader->GetOutput()->GetDimensions(dims);

//获取图像原点
double origin[3];
reader->GetOutput()->GetOrigin(origin);

//获取像素间隔
double spaceing[3];
reader->GetOutput()->GetSpacing(spaceing);

auto changer =vtkSmartPointer<vtkImageChangeInformation>::New();
//这里只是传递数据，并没有管道连接，故需update
changer->SetInput(reader->GetOutput());

//修改图像原点
changer->SetOutputOrigin(100, 100, 0);

//修改像素间隔
changer->SetOutputSpacing(5,5,1);

//将原点修改为图像中点，会覆盖SetOutputOrigin()
changer->SetCenterImage(1);

changer->Update();

//之后获取信息就要从changer里获取了，因为数据已经传到changer里了
changer->GetOutput();
```

## 像素的访问与修改
### 直接访问
```c++
//千万别忘了update

//获取维度，用于迭代
int dims[3];
reader->GetOutput()->GetDimensions(dims);

//获取像素的元组组分数(判断是灰度图,梯形图,RGB图)
int nbOfComp;
nbOfComp = reader->GetOutput()->GetNumberOfScalarComponents();

for (int k = 0; k < dims[2]; k++) {
	for (int j = 0; j < dims[1]; j++) {
		for (int i = 0; i < dims[0]; i++) {
			//返回的是void指针,需要根据实际属性类型进行强制转换
			unsigned char *pixel = static_cast<unsigned char *>(reader->GetOutput()->GetScalarPointer(i, j, k));

			//对元组内的元素进行操作
			*pixel;
			*(pixel + 1);
			*(pixel + 2);
		}
	}
}
```
### 迭代器访问
```c++
//定义一个子区域，用于指定要迭代的区域
int subRegion[6] = {0, 300, 0, 300, 0, 0};
vtkImageIterator<unsigned char> it(reader->GetOutput(), subRegion);

while (!it.IsAtEnd()) {

    //获取tuple的指针，分别指向开始和结尾处
    unsigned char *inSI = it.BeginSpan();
    unsigned char *inSIEnd = it.EndSpan();

    while (inSI != inSIEnd) {

        //操作这个像素中的某个属性值
        ++inSI;
    }
    it.NextSpan();
}
```

## 数据类型的转化
如某些属性数据类型为float或double的图像，但通常在显示时需要unsigned char类型的属性数据，这时就需要转化，不推荐用 *vtkImageCast*
```c++
auto shift = vtkSmartPointer<vtkImageShiftScale>::New();
shift->SetInputData((vtkDataObject *)reader->GetOutput());
shift->SetOutputScalarTypeToUnsignedChar();
//设置偏移量
shift->SetShift(1);
//设置缩放值
shift->SetScale(127.5);
shift->Update();
```
> 这里解释一下 *shift* 和 *scale* ，输入数据-1映射为 *(-1+1)x127.5=0*，而+1则映射为 *(+1+1)x127.5=255*

## 颜色映射
### RGB转灰度
```c++
auto luminanceFilter = vtkSmartPointer<vtkImageLuminance>::New();
luminanceFilter->SetInputData(reader->GetOutput());
luminanceFilter->Update();
```
### 提取颜色组分
提取RGB中某一颜色组分，提取的数据既是灰度图，目前并不清楚这么做有什么意义，可能在分析医学图像上有用吧
```c++
auto extractRedFilter = vtkSmartPointer<vtkImageExtractComponents>::New();
extractRedFilter->SetInputConnection(reader->GetOutputPort());
//0 1 2分别对应R G B
extractRedFilter->SetComponents(0);
extractRedFilter->Update();
```
### 彩色映射
vtkImageMapToColors接收数据属性为标量(scala)的图像数据(只会处理第一个组分的值)  
vtkLookupTable是一个颜色映射表
```c++
auto colorTable = vtkSmartPointer<vtkLookupTable>::New();
//要映射的标量数据范围
colorTable->SetRange(0.0, 255.0);
//可以用SetTableValue指定特定颜色，也可以像下面一样指定范围
colorTable->SetHueRange(0.1, 0.5);
colorTable->SetValueRange(0.6, 1.0);
colorTable->Build();

auto colorMap = vtkSmartPointer<vtkImageMapToColors>::New();
colorMap->SetInputConnection(reader->GetOutputPort());
colorMap->SetLookupTable(colorTable);
colorMap->Update();
```
> HSV是一种颜色表示方法，[Hue, Saturation, Value](https://baike.baidu.com/item/HSV/547122)

### 彩色合成
```c++
//就是融合三个灰度图，三个灰度值分别对应RGB
auto appendFilter = vtkSmartPointer<vtkImageAppendComponents>::New();
appendFilter->AddInputConnection(reader1->GetOutputPort());
appendFilter->AddInputConnection(reader2->GetOutputPort());
appendFilter->AddInputConnection(reader3->GetOutputPort());
appendFilter->Update();
```
