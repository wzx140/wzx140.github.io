---
layout: post
title:  "maven与IDEA的目录结构"
date:   2019-7-5
categories: Java
tags: Java Maven IDEA
mathjax: false
author: wzx
---

- 目录
{:toc}

今天在写代码时，出现了在IDEA可以运行单元测试，但是通过`mvn test`却找不到测试文件的奇怪情况。所以对IDEA的目录结构配置以及*maven*的目录结构配置做一个总结





## maven项目默认目录
- `src/main/java`	：项目源代码目录
- `src/main/resources` ：项目资源目录
- `src/test/java`	：测试源代码目录
- `src/test/resources` ：测试资源目录
- `target/classes` ：项目源代码编译文件目录（**项目资源目录**里的文件也会复制到这里）
- `target/surefire-reports` ：测试结果的报告
- `target/test-classes` ：测试源代码编译文件目录（**测试资源目录**里的文件也会复制到这里）

## maven项目目录设置
```xml
<build>
    <!--  项目源代码目录    -->
    <sourceDirectory>src/main/java</sourceDirectory>
    <!--  测试源代码目录    -->
    <testSourceDirectory>src/test/java</testSourceDirectory>
    <!--  项目源代码编译目录    -->
    <outputDirectory>target/classes</outputDirectory>
    <!--  测试源代码编译目录    -->
    <testOutputDirectory>target/test-classes</testOutputDirectory>

    <!--  项目资源目录    -->
    <resources>
        <resource>
            <directory>src/main/resource</directory>
        </resource>
    </resources>

    <!--  测试资源目录    -->
    <testResources>
        <testResource>
            <directory>src/test/resource</directory>
        </testResource>
    </testResources>
</build>
```
## IDEA Content roots
IDEA的*Content roots*是项目中源代码，脚本，单元测试等文件的集合，通过*Content roots*下不同类型的目录来分类

![]({{ site.url }}/assets/img/2019-7-5-1.png)

- `source` ：项目源代码，使用IDEA编译时，效果等同于maven里的`src/main/java`

- `resource` ：项目资源文件，使用IDEA编译时，效果等同于maven里的`src/main/resources`

- `test resource` ：测试源代码，使用IDEA编译时，效果等同于maven里的`src/test/java`

- `tests` ：测试资源文件，使用IDEA编译时，效果等同于maven里的`src/test/resources`

- `exclude` ：应该被忽略的文件，如缓存或者编译产生文件，图中的target被标记成了exclude

> IDEA的目录设置与*pom.xml*的目录设置不是相关联的

## 获取资源文件
`Class.class.getResource(String path)`

- path**不以`/`**开头时，从此类的**源文件同级目录**下取资源；
- path**以`/`** 开头时，则是从classpath根目录下获取；

- 若为测试类（类源文件处在**测试源代码目录**里），相对路径在`target/test-classes/`下（**测试资源文件**目录里的文件也会拷贝到其中）
- 若不为为测试类（类源文件处在**项目源代码目录**里），相对路径在`target/classes/`下（**项目资源文件**目录里的文件也会拷贝到其中）

## 总结
在IDEA中，相对路径默认为**项目根目录**。IDEA的目录设置与*pom.xml*的目录设置**不是相关联**的，同一个目录可以在*pom.xml*中标记为`testResources`，却在IDEA标记为`resource`。另外，IDEA标记目录的属性文件保存在`.idea`里。所以在实际开发中，**应该让两者的标记相同**。
