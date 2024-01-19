---
layout: post
title:  "QT如何实现Go to slot的自动连接"
date:   2018-10-12
categories: Others
keywords: QT
mathjax: false
author: wzx
---

执行**Go to slot**后会在该组件的父级容器内产生*on_xx_xx*函数，但在其构造函数内却没有看见*connect*函数。





在构造函数里调用了`ui->setupUi(this);`,查看源码发现其实是调用了**QMetaObject::connectSlotsByName(QObject *o)**这个函数来实现connect的。
```c++
void QMetaObject::connectSlotsByName(QObject *o)
{
     if (!o)
        return;
     const QMetaObject *mo = o->metaObject();
     Q_ASSERT(mo);
//第7行
     const QObjectList list = qFindChildren<QObject *>(o, QString());
//第8行
     for (int i = 0; i < mo->methodCount(); ++i) {
         const char *slot = mo->method(i).signature();
         Q_ASSERT(slot);
         if (slot[0] != 'o' || slot[1] != 'n' || slot[2] != '_')
             continue;
         bool foundIt = false;
         for(int j = 0; j < list.count(); ++j) {
             const QObject *co = list.at(j);
             QByteArray objName = co->objectName().toAscii();
             int len = objName.length();
//第11行
             if (!len || qstrncmp(slot + 3, objName.data(), len) || slot[len+3] != '_')
                 continue;
             const QMetaObject *smo = co->metaObject();
             int sigIndex = smo->indexOfMethod(slot + len + 4);
             if (sigIndex < 0) { // search for compatible signals
                 int slotlen = qstrlen(slot + len + 4) - 1;
//第14行
                 for (int k = 0; k < co->metaObject()->methodCount(); ++k) {
                     if (smo->method(k).methodType() != QMetaMethod::Signal)
                         continue;

                     if (!qstrncmp(smo->method(k).signature(), slot + len + 4, slotlen)) {
                         sigIndex = k;
                         break;
                     }
                 }
             }
//第33行
             if (sigIndex < 0)
                 continue;
             if (QMetaObject::connect(co, sigIndex, o, i)) {
                 foundIt = true;
//第38行
                 break;
             }
         }
         if (foundIt) {
             // we found our slot, now skip all overloads
             while (mo->method(i + 1).attributes() & QMetaMethod::Cloned)
                   ++i;
         } else if (!(mo->method(i).attributes() & QMetaMethod::Cloned)) {
             qWarning("QMetaObject::connectSlotsByName: No matching signal for %s", slot);
         }
     }
 }
```

看**connectSlotsByName**的实现，可以注意到以下几个地方：

1. 第7行，取得o的所有子对象

2. 第8行，是一个遍历o的方法的循环，o的信号和槽就在其中

3. 第11行，对于方法名称不是"on_"开头的方法跳过不处理，这也说明，如果你在一个QObject子类里定义了"on_"开头的槽的话，一定会被connectSlotsByName函数进行搜索匹配的操作的

4. 第14行开始到33行，开始遍历o的所有的子对象，试图匹配到与槽名称以及信号名称相应的子对象。首先取出其objectName()与槽名称里的第一个‘_’和第二个‘_’做名称匹配。其次取出子对象的所有信号，与第二个‘_’之后部分做匹配。

5. 如果匹配成功，则会执行36行的连接代码。连接成功的话，就会在38行break中断循环。

看到第5点，已经很明了了，对于同名的控件，**connectSlotsByName只会连接子对象链表里的第一个对象的信号到槽上**
