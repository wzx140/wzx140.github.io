---
layout: post
title:  "QT Creator与CMake开发环境搭建"
date:   2018-10-17
categories: Others
keywords: QT CMake QT-Creator
mathjax: false
author: wzx
---

*qmake* 是为 *Qt* 量身打造的，如果只是开发**轻量级**的qt应用程序，使用起来非常方便。*cmake* 虽然使用上不如*qmake*简单直接，但复杂换来的是强大的功能





## 为什么使用Cmake
*VTK*工程是用*CMake*进行管理的，而*Qt*自身就有*qmake*工具。对于一些规模较小的*Qt*工程而言，用*qmake*来构建工程确实很方便，但随着工程复杂度的增加或当所开发工程依赖于其他函数库时，使用*CMake*来管理工程会是一个明智的选择

### Qmake VS Cmake
**官方文档**是这样说的：
* 对简单的Qt工程，采用 qmake
* 对复杂度超过 qmake 处理能力的，采用 cmake
> 我们可以在做一个大项目当然用Cmake了，滑稽。

### Qmake 与 Cmake的不同
* qmake是Qt专用的项目管理工具，对应的工程文件是\*.pro，当然，在命令行下才会需要手动执行qmake，完全可以在qtcreator这个专用的IDE下面打开\*.pro文件，使用qmake命令的繁琐细节不用你管了
* make是跨平台项目管理工具，它用更抽象的语法来组织项目。cmake是抽象层次更高的项目管理工具，cmake命令执行的CMakeLists.txt文件

## 如何在QT Creator中使用CMake
1. [Setting Up a CMake Project](https://doc-snapshots.qt.io/qtcreator-4.0/creator-project-cmake.html)
2. [CMake Manual](http://doc.qt.io/qt-5/cmake-manual.html)
3. [CMake官方文档](https://cmake.org/cmake/help/v3.12/)
4. CMake基本语法不会的，[戳这里]({% post_url 2018-10-18-cmake %})
> 上面是QT官方文档描述的如何新建CMake项目，注意**强烈建议将git从环境变量中删去，不然会报错**，其他的按照官方文档按部就班操作就行。
