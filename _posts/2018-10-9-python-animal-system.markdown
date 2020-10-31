---
layout: post
title:  "动物识别专家系统"
date:   2018-10-9
categories: 算法和数据结构
tags: DataStructure Python Flask Topological_sorting
mathjax: true
author: wzx
---

- 目录
{:toc}

今后一定好好学习





## 基于web的动物识别专家系统

作为**机械**专业的学生竟然能遇到这个作业，上图。

![](https://raw.githubusercontent.com/wzx140/animal_system/master/static/img.jpg)

看起来很容易啊,不就几个if-else就解决了吗？但最后编写的时候才发现，if-else的**顺序**是有讲究的，这里就需要用到**拓扑排序**。

> 对一个有向无环图(*Directed Acyclic Graph*简称DAG)进行拓扑排序，是将G中所有顶点排成一个线性序列，使得图中任意一对顶点u和v，若边$(u,v)\in E(G)$，则u在线性序列中出现在v之前。通常，这样的线性序列称为满足拓扑次序(Topological Order)的序列，简称拓扑序列。简单的说，由某个集合上的一个偏序得到该集合上的一个全序，这个操作称之为拓扑排序。

通俗点说就是“排序”具有依赖关系的任务。下面我来分析一下作业中的问题，这是一张用`MindMaster`绘制的思维导图，可以看出图中右边的节点都是依赖于左边的节点（也就是说只有先进行左边的判断才能进行右边的判断）。

![](https://raw.githubusercontent.com/wzx140/animal_system/master/mind.jpg)

&#8195;&#8195;下面就需要用到一下算法进行拓扑排序：

![]({{ site.url }}/assets/img/2018-10-9-1.jpg)

**1. 寻找入度为0的顶点**
**2. 输出此顶点**
**3. 删除该顶点，并删除以它为起点的有向边**
**4. 检查是否有入度为0的顶点，若有下一个循环，若没有结束。**

> 1. 当然这些顶点不能相互依赖。
> 2. **入度**：通俗来说即被箭头所指的个数。
> 3. **有向边**：指带有箭头的那条线。

> 哈哈，这些解释好小白，我也是初学者。

### 上代码

```python
# -*- coding: utf-8 -*-
import collections


# 根据路径加载规则
def loads(path):
    # 条件
    P_list = []
    # 结果
    Q_list = []
    with open(path, encoding='utf-8') as f:
        lines = f.readlines()
        for line in lines:
            # 去除回车
            line = line.strip('\n')
            if line:
                data_list = line.split(' ')
                # print(data_list[:-1])
                P_list.append(data_list[:-1])
                Q_list.append(data_list[-1])
    # print('规则加载完成')
    return P_list, Q_list


# 拓扑排序
def topological(P_list, Q_list):
    # 规则
    rule = collections.OrderedDict()

    # 计算入度
    ind_list = []
    for i in P_list:
        sum = 0
        for x in i:
            if Q_list.count(x) > 0:
                sum += Q_list.count(x)
        ind_list.append(sum)

    # 拓扑排序
    while (1):
        if ind_list.count(-1) == len(ind_list):
            break

        for i, ind in enumerate(ind_list):
            if ind == 0:
                rule[tuple(P_list[i])] = Q_list[i]
                ind_list[i] = -1
                # 更新入度
                for j, P in enumerate(P_list):
                    if Q_list[i] in P:
                        ind_list[j] -= 1
    return rule


# 推断
def infer(input_list, rule, display=False):
    result = []
    flag = False
    process = ''
    for key in rule.keys():
        # 如果所有规则都在输入中
        if list_in_set(key, input_list):
            # 添加推导出的条件
            input_list.append(rule[key])
            result.append(rule[key])
            process += '{key} 推导出 {value}\n'.format(key=key, value=input_list[-1])
            flag = True

    if flag:
        if display:
            return process + '\n' + '最终推出: ' + str(result[-1])
        else:
            return '最终推出: ' + str(result[-1])
    else:
        return '无法推出'


# 判断list中所有元素是否都在集合set中
def list_in_set(list, set):
    for i in list:
        if i not in set:
            return False
    return True


def run(text, display):
    if display == 'true':
        dis = True
    else:
        dis = False
    input_list = text.split(' ')
    P_list, Q_list = loads('data.txt')
    rule = topological(P_list, Q_list)
    return infer(input_list, rule, display=dis)


# 单机控制台使用
if __name__ == '__main__':
    print('每个特征之间用单个空白符相隔，请使用完整特征')
    print('如：蹄类->有蹄类，暗斑->暗斑点，x->x类(如 鸟->鸟类)，黑白->黑白色')
    print('输入1退出')
    while (1):
        print('请输入特征:')
        text = input()
        if text == '1':
            break
        print(run(text, display='true'))
```

### 关于web
本人用的`Flask`，做起来不是很难。  
[![wzx140/animal_system - GitHub](https://gh-card.dev/repos/wzx140/animal_system.svg?fullname)](https://github.com/wzx140/animal_system)
