---
layout: post
title:  "CMAKE常用指令"
date:   2018-11-30
categories: Others
keywords: CMake
mathjax: false
author: wzx
---

*CMake* 包含许多指令可以方便构建





### 基本指令

```
# c++标准
set(CMAKE_CXX_STANDARD 14)

# 查找指定目录下的所有源文件，然后将结果存进指定变量名
aux_source_directory(<dir> <variable>)

# 指定运行此配置文件所需的 CMake 的最低版本
cmake_minimum_required(xx)

# 该命令表示项目的名称
project(name)

# 添加链接库,指明可执行文件 main 需要连接一个名为 library_name 的链接库
target_link_libraries(target_name library_name)

# 用于输出信息
message(“message to display”)
```

## add 指令
```
# 向 C/C++编译器添加-D 定义, 参数之间用空格分割, 如果要添加其他的编译器开关,可以通过 CMAKE_C_FLAGS 变量和 CMAKE_CXX_FLAGS 变量设置
add_definitions(-xx -xxx)

# 添加 math 子目录,指明本项目包含一个子目录 math，这样 math 目录下的 CMakeLists.txt 文件和源代码也会被处理
add_subdirectory(math)

# 添加target依赖, 确保在编译本 target 之前,其他的 target 已经被构建。
add_dependencies(target-name depend-target1 depend-target2 ...)

# 生成可执行文件
add_executable(target_name main.cpp)

# 生成动态静态库
add_library(libname [SHARED|STATIC|MODULE] source1 source2 ...)
```

## 测试指令
```
# 用来控制 Makefile 是否构建 test 目标,涉及工程所有目录, 一般情况这个指令放在工程的主CMakeLists.txt 中
enable_testing()


# 添加测试的 target 以及传入参数, make test 执行测试
add_test(testname Exename arg1 arg2 ...)
```

## find 指令
### find_path
如果找到则将路径保存在 VAR 中（此路径为一个绝对路径），如果没有找到则结果为 NOTFOUND。默认的情况下，VAR 会被保存在 Cache 中，这时候我们需要清除 VAR 才可以进行下一次查询（使用 unset 命令）

```
# 用于查找包含文件 name1 的路径
find_path(<VAR> name1 [path1 path2 …])

# 使用范例  
find_path(LUA_INCLUDE_PATH lua.h ${LUA_INCLUDE_FIND_PATH})
if(NOT LUA_INCLUDE_PATH)
   message(SEND_ERROR "Header file lua.h not found")
endif()
```
### find_library, find_package
*find_library*类似*find_package*，常用*find_package*，参加[find_package]({% post_url 2018-10-31-cmake-find-package %})

## file 指令
```
# 写一条消息到名为filename的文件中。如果文件已经存在，该命令会覆盖已有的文件；如果文件不存在，它将创建该文件
file(WRITE filename "message to write"... )

# 写一条消息到名为filename的文件中，附加到文件末尾
file(APPEND filename "message to write"... )

# 拷贝文件
file(COPY files... DESTINATION <dir>
       [FILE_PERMISSIONS permissions...]
       [DIRECTORY_PERMISSIONS permissions...]
       [NO_SOURCE_PERMISSIONS] [USE_SOURCE_PERMISSIONS]
       [FILES_MATCHING]
       [[PATTERN <pattern> | REGEX <regex>]
        [EXCLUDE] [PERMISSIONS permissions...]] [...])

# READ选项将会读一个文件中的内容并将其存储在变量里。读文件的位置从offset开始，最多读numBytes个字节。如果指定了HEX参数，二进制代码将会转换为十六进制表达方式，并存储在变量里。
file(READ filename variable [LIMIT numBytes] [OFFSET offset] [HEX])

# 从一个文件中将一个ASCII字符串的list解析出来，然后存储在variable变量中，文件中的二进制数据会被忽略。回车换行符会被忽略，并通过换行符分割list
file(STRINGS filename variable [LIMIT_COUNT num]
       [LIMIT_INPUT numBytes] [LIMIT_OUTPUT numBytes]
       [LENGTH_MINIMUM numBytes] [LENGTH_MAXIMUM numBytes]
       [NEWLINE_CONSUME] [REGEX regex]
       [NO_HEX_CONVERSION])

file(RENAME <oldname> <newname>)

# 类似rm，包括在子路径下的文件
file(REMOVE [file1 ...])

# 类似rm -r，删除给定的文件以及目录，包括非空目录
file(REMOVE_RECURSE [file1 ...])

# 类似于mkdir，如果父目录不存在时，同样也会创建
file(MAKE_DIRECTORY [directory1 directory2 ...])

# 确定从direcroty参数到指定文件的相对路径
file(RELATIVE_PATH variable directory file)

file(DOWNLOAD url file [TIMEOUT timeout] [STATUS status] [LOG log] [EXPECTED_MD5 sum] [SHOW_PROGRESS])
```
