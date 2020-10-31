---
layout: post
title:  "QT绘图体系"
date:   2019-9-8
categories: Others
keywords: QT Graphics Painter
mathjax: false
author: wzx
---

简单概述 `qgraphics` 绘图系统





## QPainter绘图系统
### 组件
- `QPainter`：用于执行绘图操作，`QPainter::setPen(const QPen&)`设置画笔，`QPainter::setBrush(const QBrush&)`设置填充，`QPainter::setFont(const QFont)`设置字体
- `QPaintDevice`：二维空间的抽象层，`QPainter`绘图的画布
- `QPaintEngine`：提供了统一设备绘图接口，由`QPainter`和`QPaintDevice`内部调用

### 事件
- `QWidget::paintEvent(QPaintEvent*)`：`QWidget`的绘图事件，`QWidget::update()`与`QWidget::repaint()`，触发此事件

### 坐标系统
- 物理坐标：`QPaintDevice`基本的坐标系，以左上角为原点，xy轴向右与下递增，单位为像素

- 逻辑坐标：由物理坐标变换而来，一般中心点为原点，xy轴向右与下递增，单位没有实际意义

- 视口坐标系：`QPaintDevice`中以**物理坐标**为基础的**矩形绘图区域**

- 窗口坐标系：`QPaintDevice`中以**逻辑坐标**为基础的矩形区域，与视口坐标系的矩形为**同一个**

![]({{ site.url }}/assets/img/2019-8-28-1.png)
> 左边为视口坐标系，右边窗口坐标系，窗口坐标系使得无论 widget 的缩放，其逻辑坐标保持一致，便于绘图

## QGraphics绘图体系
更高级的图形视图框架，特别对于大规模的图形项渲染较快
### 组件
- `QGraphicsItem`：基本的图形项，自定义图形项要实现`QGraphicsItem::paint(QPainter *, const QStyleOptionGraphicsItem*, QWidget*)`绘图，和`QGraphicsItem::boundingRect()`绘图区域，qt本身已经实现了一些图形项
![]({{ site.url }}/assets/img/2019-8-28-4.png)

- `QGraphicsScene`：`QGraphicsItem`的容器，可以附加到多个视图中
- `QGraphicsView`：用于显示`QGraphicsScene`，提供滚动条
- `QGraphicsItem`：基本的图形项

### 坐标系统
- 图形项坐标：如图所示，中心为原点，单位为像素，不会随着 widget 放缩
    - 图形项在父坐标系的位置是其**中心点在父坐标系的位置**
    - 未设置父图形项的子图形项为顶层图形项，父坐标系为场景坐标系
    - 类内部操作时，一般只需考虑**局部坐标**，`QGraphicsItem::pos()`QGraphicsItem::setPos()例外，返回父坐标中的位置
- 场景坐标：如图所示，用于确定**顶层图形项**的位置，单位为像素，不会随着 widget 放缩
- 视图坐标：如图所示，widget 的物理坐标，与场景和图形项坐标无关，需要通过**坐标映射**之后再操作

![]({{ site.url }}/assets/img/2019-8-28-2.png)

在需要用到**不同** `item` 的坐标时，一定要用 `mapxx`函数来 **统一坐标系**。

### 布局
> 函数原型省去了部分元素，具体参见文档

- `QGraphicsView::sceneRect()`：视图中的场景的可视化范围。当视图小于场景时，只对`QGraphicsView::sceneRect`内的图像生成滚动条，之外的图像将不能被显示
    - 计算对齐方式时，会以`QGraphicsView::sceneRect`为标准
    - 默认与`QGraphicsScene::sceneRect`（**两者不同**）具有相同的值，并且随之变化（未设置的话）

- `QGraphicsScene::sceneRect()`：场景的边界矩形，用于管理图形项
    - 如果设置了`QGraphicsView::sceneRect`，那么`QGraphicsScene::sceneRect`与视图的显示无关，超出边界的item也可能被显示出来

- `QGraphicsItem::boundingRect()`：图形项的边界矩形。`QGraphicsView`通过它来**重绘图形项**，所以所有绘图内容必须在其内。不受图形项变换的影响

- `QGraphicsItem::shape()`：返回图形项的形状，由 `QGraphicsItem::contains()` 和 `QGraphicsItem::collidesWithPath()`的默认调用实现碰撞检测，命中等

![]({{ site.url }}/assets/img/2019-8-28-3.png)

- `QGraphicsItem::zValue()`：图形项的堆叠顺序，0为顶层

### `QGraphicsItem`父子关系
- `父item`属性改变，`子item` 也会跟着改变，如`enable`，`visible`

- `父item`会**传播位置改变和transform**给`子item`，即子跟着父移动，变换

- 绘图顺序：先绘制`父item`，再绘制`子item`

- `子item`的 `stack order` 默认在`父item`之前，为`子item` 设置 `ItemStacksBehindParent`，可以改变这个顺序

### `QGraphicsItem`场景事件
`QGraphicsScene`将事件打包传递给`子item`，注意`父item`不会过滤掉`子item`的事件，除非设置了`QGraphicsItem::setFiltersChildEvents(bool)`，事件的触发与 `item` 的`zValue`和`stack`顺序有关
- `QVariant QGraphicsItem::itemChange(GraphicsItemChange change, const QVariant &value)`：描述了item的改变，需要设置`QGraphicsItem::ItemSendsGeometryChanges`才会触发

- `hoverEnterEvent` `hoverLeaveEvent` `hoverMoveEvent`：鼠标徘徊事件，需要`setAcceptHoverEvents(true)`才会触发

## REFERENCE
[1]王维波, 栗宝鹃, 侯春望. Qt 5.9 C++开发指南[M]. 人民邮电出版社, 2018.  
[2][QGraphicsItem Class](https://doc.qt.io/qt-5/qgraphicsitem.html) | Qt Widgets 5.13.1[EB/OL].
