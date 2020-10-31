---
layout: post
title:  "QT Creator的promote to自定义控件"
date:   2018-10-15
categories: Others
keywords: QT
mathjax: false
author: wzx
---

`promote to`将QT自带的那些控件（如*QLable，QPushButton*之类）的派生类作为自定义控件可以再*design*界面中使用。





## 如何实现
1. 自己编写继承至某个控件的类，比如从*QTableWidget*继承而来的*Spreadsheet*类。

2. 在design界面中拖入QTableWidget控件，右键**promote to**

3. 修改*base class name* 为 *QTableWidget*，*Promoted class name*  为 *Spreadsheet*，*Header file* 为 *Spreadsheet.h*。**建议注意这里的文件名是否正确**，头文件名称系统会自动生成，但如果你的命名习惯与QT不一样，这里的名称很有可能是错的。
![]({{ site.url }}/assets/img/2018-10-15-14.png)

4. 点击Add，再点击Promote，大功告成。

## 插件法
如果插件法可行，建议还是使用插件法。可以看这篇文章，[Qt编写自定义控件插件路过的坑及注意事项](https://www.cnblogs.com/feiyangqingyun/p/6182320.html)。
