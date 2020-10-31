---
layout: post
title:  "QString总结"
date:   2018-10-11
categories: Others
keywords: QT String
mathjax: false
author: wzx
---

*QT*库对*C++*做了很多封装，其中**QString**就可以大大减小工作量。




## QString类
### 文档
* [中文文档](http://www.kuqin.com/qtdocument/qstring.html)
* [官方文档](http://doc.qt.io/qt-5/qstring.html)

### 常用方法总结
#### 字符串拼接
```c++
//字符串有如下几个操作符
//QString提供了一个二元的"+"操作符用于组合两个字符串，并提供了一个"+="操作符用于将一个字符串追加到另一个字符串的末尾，例如：
QString str1="welcome";
str1=str1+"to you !";//str1="welcome to you !"
QString str2="hello ,";
str2+="world!"//str2="hello ,world!"
```
#### 字符串组合
```c++
//组合字符串的另一个函数QString::sprintf(),此函数支持的格式定义符和C++库中的函数sprintf()定义一样，例如
QString str;
str.sprintf("%s","welcome");//str="welcome"
str.sprintf("%s","to you!");//str="to you!"
str.sprintf("%s %s","welcome ","to you");//str="welcome to you"
void Dialog::add()
{
    QString s1=edit1->text();
    QString s2=edit2->text();
    QString s3=s1+s2;
    s3+="end!";
    /*
     * QString内部维护了字符串数组
    */
    const char *arr=s3.toStdString().data();
    QString s4=arr;
    QString s5;
    //注意sprintf的参数是字符串，而不是QString对象
    s5.sprintf("%s","welcome ");
    s5.sprintf("%s","to you!");
    s5.sprintf("%s - %s",s1.toStdString().data(),s2.toStdString().data());
    label1->setText(s5);
}
```

#### debug输出
```c++
qDebug("Items in list: %d", myList.size());

//If you include <QtDebug>, a more convenient syntax is also available:
qDebug() << "Brush:" << myQBrush << "Other value:" << i;

//去除双引号
QString str1="welcome";
qDebug("%s",qPrintable(str1));
qDebug()<<qPrintable(str1);//两种方法都可以

```


#### 字符串比较
```c++
//比较两个字符串也是经常使用的功能，QString提供了多种比较手段。
operator<(const QString &);//比较一个字符V换是否小于另一个字符串，如果是，则返回true。
operator<=(const QString &);//比较一个字符串是否小于等于另一个字符串，如果是，则返回true。
operator==(const QString &);//比较两个字符串是否相等，如果相等，则返回true。
operator>=(const QString &);//比较一个字符串是否大于等于另一个字符串，如果是，则返回true。
```

#### 其他
```c++
//QString::toInt()函数将字符串转换为整型数值，类似的函数还有toDouble、toFloat()、toLong()、toLongLong()等。下面例子说明其用法：
QString str="125";
bool ok=false;
int hex=str.toInt(&ok,16);//ok=true,hex=293
int dec=str.toInt(&ok,10);//ok=true,dec=125

/*一个NULL字符串就是使用QString的默认构造函数或者使用"(const char *)0"作为参数的构造函数创建的QString字符串对象；而一个空字符串是
*一个大小为0的字符串。一个NULL字符串一定是一个空字符串，二一个空字符串未必是一个NULL字符串。例如：*/
QString().isNull();//结果为true
QString().isEmpty();//结果为true
QString("").isNull();//结果为false
QString("").isEmpty();//结果为true
```
