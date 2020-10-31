---
layout: post
title:  "CLion+QT+VTK开发环境搭建"
date:   2018-10-19
categories: VTK
keywords: QT CMake VTK CLion
mathjax: false
author: wzx
---

既然都不使用*QMake*而使用*CMake*了，那只要编辑*ui，qrc*文件时用*qt creator*就行了。基于本人*jetbrains*脑残粉，和*CLion*的代码提示等功能非常好用。这里介绍如何在*CLion中使用CMake构建QT+VTK工程*。





## 配置QT环境
> 首先，官网上有如何在CMake里配置QT，见[CMake Manual](http://doc.qt.io/qt-5/cmake-manual.html)。但在实际使用中却还有出入。CMake基本语法不会的，[戳这里]({% post_url 2018-10-18-cmake %})

### 目录结构
> 根据自己需要修改

```
.
├── CMakeLists.txt
├── include             //存放头文件
│   ├── mainwindow.h
│   └── ui_mainwindow.h
├── lib                 //存放库文件
├── res                 //存放资源文件
└── src                 //存放源文件
    ├── main.cpp
    ├── mainwindow.cpp
    └── mainwindow.ui

4 directories, 6 files
```
### CMakeLists.txt
> 根据目录结构要做修改

```
# 所需最低版本
cmake_minimum_required(VERSION 3.12)
# c++标准
set(CMAKE_CXX_STANDARD 14)

# 工程名
project(test_qt)

# 从 include/ 下寻找头文件
include_directories(include)

# 从 src/ 下寻找cpp文件，并赋值给 SOURCE_FILES
aux_source_directory(src SOURCE_FILES)

#qt配置

# Find includes in corresponding build directories
set(CMAKE_INCLUDE_CURRENT_DIR ON)

# Instruct CMake to run moc automatically when needed
set(CMAKE_AUTOMOC ON)

# Create code from a list of Qt designer ui files
set(CMAKE_AUTOUIC ON)

# Find the QtWidgets library,注意修改为自己的地址
set(CMAKE_PREFIX_PATH G:\\qt5\\5.10.0\\mingw53_32\\lib\\cmake)
find_package(Qt5 REQUIRED Widgets)

set(QT_FORM_FILES include/mainwindow.h )

# 添加编译文件,WIN32为windows下的应用程序
add_executable(${PROJECT_NAME} WIN32 ${SOURCE_FILES} ${QT_FORM_FILES})

# Use the Widgets module from Qt 5
target_link_libraries(${PROJECT_NAME} Qt5::Widgets)

```
### 添加环境变量
以上配置完成后，在*QT Creator*中可以完美运行，但在*CLion*却出现，窗口一闪而过，控制台打印`exit code -1073741571`。这个值正常情况应该跟你主函数*return*的值一样，这里返回这种奇怪的数肯定其中有错误。  
在网上查找资料后发现，这个代表缺少某些*dll*文件。可能是QT Creator能自动找到自家的库文件，而*CLion*本身不支持*QT*吧。在系统环境变量中添加`G:\qt5\5.10.0\mingw53_32\lib`和`G:\qt5\5.10.0\mingw53_32\bin`。

### 添加External Tools
> 具体执行文件位置，自行修改

1. 添加*Qt Designer*，用于编辑*ui*文件
```
Name : Qt Designer
Program : G:\qt5\5.10.0\mingw53_32\bin\designer.exe
Arguments : $FilePath$
Working directory : $ProjectFileDir$
```
2. *QT Creator*，用于编辑*qrc*文件
```
Name : QT Creator
Program : G:\qt5\Tools\QtCreator\bin\qtcreator.exe
Arguments : $FilePath$
Working directory : $ProjectFileDir$
```
3. *uic*,用于将*ui*文件生成头文件，当*ui*文件修改时，应使用*uic*刷新头文件。
```
Name : uic
Program : G:\qt5\5.10.0\mingw53_32\bin\uic.exe
Arguments : $FileName$ -o ui_$FileNameWithoutExtension$.h
Working directory : $FileDir$
```
4. 其他，主要是用于国际化的翻译工具，若无需求可以忽略  
> For .cpp/.ui select LUpdate to create its translation file  
For .ts select Linguist and start the translating

```
LUpdate:

Program:   "PATH_TO_QT/QT_VERSION/QT_ARCH/lib/bin/lupdate")
Arguments: $FilePath$ -ts $FileNameWithoutExtension$.ts

Linguist:

Program:   "PATH_TO_QT/QT_VERSION/QT_ARCH/lib/bin/linguist")
Arguments: $FilePath$
```

## 配置vtk环境
> vtk的编译参见[VTK + QT Creator开发环境搭建]({% post_url 2018-10-15-VTK-QT-CREATOR %})

在CMakeLists.txt作如下修改：  

```
# 添加
set(VTK_DIR G:\\VTK)
find_package(VTK REQUIRED)
include(${VTK_USE_FILE})

#修改
target_link_libraries(${PROJECT_NAME} Qt5::Widgets ${VTK_LIBRARIES})
```
大功告成，比qmake简洁了很多。

## vtk_demo
[![wzx140/Demo_vtk - GitHub](https://gh-card.dev/repos/wzx140/Demo_vtk.svg?fullname)](https://github.com/wzx140/Demo_vtk)
