---
layout: post
title:  "CMake之find_package"
date:   2018-10-31
categories: Others
tags: CMake
mathjax: false
author: wzx
---

*find_package* 可以被用来在系统中自动查找配置构建工程所需的程序库。*CMake* 自带的模块文件里有大半是对各种常见开源库的 *find_package* 支持，支持库的种类非常多。





## 原理
*find_package* 可以根据一些预先定义好的变量<sup>[1]</sup>去搜索库文件，就是去搜索一个叫 *Find<package>.cmake* 的文件，并将搜索结果存入到特定变量<sup>[1]</sup>中以供使用者调用。
> [1] 这些变量有关目标库的设置，比如 *xxx_ROOT,xxx_INCLUDEDIR* 之类的，你可以修改这些变量去帮助 *find_package* 找到库  
> [2] 这些变量就是 *find_package* 的搜索结果，比如 *xxx_FOUND,xxx_LIBRARY_DIRS* 等  

> 具体可以查看[CMake_modules](https://cmake.org/cmake/help/git-master/manual/cmake-modules.7.html#find-modules)里对应库的说明

### 搜索路径
cmake会先在${CMAKE_MODULE_PATH}和<CMAKE_ROOT>/share/cmake-x.y/Modules/下寻找Find<name>.cmake，如果是linux系统会从/usr/local/lib下搜索模块，如果是windows会从环境变量目录以及注册表的模块目录下搜索模块

## 格式
```
find_package(<package> [version] [EXACT] [QUIET] [MODULE]
             [REQUIRED] [[COMPONENTS] [components...]]
             [OPTIONAL_COMPONENTS components...]
             [NO_POLICY_SCOPE])
```
* *QUIET* ，如果找不到包，将不会显示消息
* *REQUIRED* ，如果找不到包，将停止处理并显示错误消息
* *COMPONENTS* ，或REQUIRED之后，列出项目所需组件列表

## 操作

### 如何寻找支持的库
* 在 [CMake_modules](https://cmake.org/cmake/help/v3.8/manual/cmake-modules.7.html#all-modules) 寻找支持的库
* 在对应库的官方文档里可能有相关cmake配置

> 官方文档里可能有更完整的配置

### 如何导入库
```
#前面可能需要设置一些其他变量
#比如要导入xxx库
find_package(xxx)
#包含头文件，find_package把头文件地址保存进了xxx_INCLUDE_DIRS中
include_directories(${xxx_INCLUDE_DIRS})
add_executable(${PROJECT_NAME} files)
#连接库文件
target_link_libraries(your_target_name ${xxx_LIBRARIES})
```
> 具体操作查看官方文档  

参考文献[cmake](https://cmake.org/cmake/help/v3.0/command/find_package.html)
