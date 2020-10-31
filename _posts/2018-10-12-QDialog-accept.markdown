---
layout: post
title:  "QDialog中accept的问题"
date:   2018-10-12
categories: Others
keywords: QT
mathjax: false
author: wzx
---

要学好QT：
* C++基本语法一定要扎实啊，建议先看*c++ primer plus*这本书，有能力的可以看看英文的
* 遇到问题要习惯于看文档和源码解决
* 多实践，多打代码





## 问题
1. 在`QDialog`里放入一个`Push Button`，关联`pushButton`的`clicked()``信号和`QDialog`的`accept()`槽。
2. 代码

```c++
#include "mainwindow.h"
#include
#include "logindlg.h"

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    MainWindow w;
    LoginDlg dlg;                        // LoginDlg继承QDialog
    if(dlg.exec() == QDialog::Accepted) // 利用Accepted返回值判断按钮是否被按下
    {
        w.show();                      // 如果被按下，显示主窗口
        return a.exec();              // 程序一直执行，直到主窗口关闭
    }
    else return 0;          //如果没有被按下，则不会进入主窗口，整个程序结束运行
}
```
**但是**问题来了，根据文档：  
* `accept()`：  *Hides the modal dialog and sets the result code to Accepted.*  
* `exec()`：  *Shows the dialog as a modal dialog, blocking until the user closes it. The function returns a DialogCode result.*

**通过源码查看accept内部也只是调用setVisible函数**

点击按钮之后明明只是QDialog只是隐藏，`exec()`为什么会返回参数呢？

## 解决
原来只要hide QDialog就会exec()停止阻塞并返回参数。可以参考[QDialog在hide()之后，就被销毁的原因](https://blog.csdn.net/lengyuezuixue/article/details/81012763)
