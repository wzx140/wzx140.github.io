---
layout: post
title:  "CMAKE之变量"
date:   2018-11-30
categories: Others
keywords: CMake
mathjax: false
author: wzx
---

*CMake* 中有许多显式或隐式的预定义好的变量，也可以自己定义变量





## 预定义变量

### 二进制目录
CMAKE_BINARY_DIR, PROJECT_BINARY_DIR, <projectname\>_BINARY_DIR  
这三个变量指代的内容是一致的，如果是 in-source 编译，指的就是工程顶层目录，如果是 out-of-source 编译,指的是工程编译发生的目录

### 源码目录
CMAKE_SOURCE_DIR, PROJECT_SOURCE_DIR, <projectname\>_SOURCE_DIR  
这三个变量指代的内容是一致的,不论采用何种编译方式,都是工程顶层目录。

### CMAKE_CURRENT_SOURCE_DIR
指的是当前处理的 CMakeLists.txt 所在的路径

### CMAKE_CURRRENT_BINARY_DIR
如果是 in-source 编译,它跟 CMAKE_CURRENT_SOURCE_DIR 一致,如果是 out-of-source 编译, 他指的是 target 编译目录

### 变量所在处
CMAKE_CURRENT_LIST_FILE, CMAKE_CURRENT_LIST_LINE  
输出调用这个变量的 CMakeLists.txt 的完整路径, 所在行

### CMAKE_MODULE_PATH
这个变量用来定义自己的 cmake 模块所在的路径。如果你的工程比较复杂,有可能会自己编写一些 cmake 模块,这些 cmake 模块是随你的工程发布的,为了让 cmake 在处理CMakeLists.txt 时找到这些模块,你需要通过 SET 指令,将自己的 cmake 模块路径设置一下。比如SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)这时候你就可以通过 INCLUDE 指令来调用自己的模块了。

### out-of-source 目录
EXECUTABLE_OUTPUT_PATH, LIBRARY_OUTPUT_PATH
分别用来重新定义最终结果的存放目录

### PROJECT_NAME
返回通过 PROJECT 指令定义的项目名称。

### CMAKE_BUILD_TYPE
设置Release, Debug

## 变量的传递
### set
用set命令定义的变量能传递到子目录，同级目录和父目录则不能传递
`set(var_name var1 var2 ...)`

### set_property get_property
使用set_property实现共享变量的方法，不会将变量写入CMakeCache.txt，而是写入内存中
```
# GLOBAL 代表全局
set_property(GLOBAL PROPERTY var_name var1 var2 ...)

# 将 var_name_src 的值读入 var_name_temp 中
get_property(var_name_temp GLOBAL PROPERTY var_name_src)
```
