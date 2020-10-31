---
layout: post
title:  "浅层神经网络中的矩阵计算"
date:   2018-12-11
categories: MachineLearning
tags: Matrix DeepLearning
mathjax: true
author: wzx
---

- 目录
{:toc}

单隐层的神经网络中 正向传播 与 反向传播 向量化的具体实现与具体计算过程




## 结构
如图所示，单隐层神经网络，有 $$n_{0}$$ 个输入特征值，隐藏层有  $$n_{1}$$ 个节点
![]({{ site.url }}/assets/img/2018-12-12-1.png)

每个神经节点采用 sigmoid 激活函数

$$\sigma(z)=\frac{1}{1+e^{-z}}\qquad$$

采用交叉熵成本函数

$$
J(w,b)=-\frac{1}{m}\times\sum_{k=1}^{m}[ y\times ln\hat{y}+(1-y)\times ln(1-\hat{y})]
$$

### 表示符号
- $$n_{i}$$  ：表示第i层的节点个数
- $$t^{(i)}$$ ： 表示第 i 个样本的参数t
- $$t^{[0]}$$ ：表示第0层(即输入层)的参数t
- $$t^{[1]}$$ ：表示第1层(即隐藏层)的参数t
- $$t^{[2]}$$ ：表示第2层(即输出层)的参数t
- $$\hat{y}$$ ：表示预测值
- $$a^{[l]}$$ ：表示第 l 层的节点的输出值

## 正向传播
### 训练样本
输入样本，共有 m 个样本，每个样本有 $$n_{0}$$ 个特征值

$$
X=\left\{
  \begin{matrix}
   x^{(1)}_{1} & \cdots & x^{(m)}_{1} \\
   x^{(1)}_{2} & \cdots & x^{(m)}_{2} \\
   \vdots  & \ddots & \vdots \\
   x^{(1)}_{n_{0}}  & \cdots & x^{(m)}_{n_{0}}
   \end{matrix}
   \right\}_{n_{0}\times m}
$$

实际输出值

$$
Y=\left\{
  \begin{matrix}
   y^{(1)} & \cdots & y^{(m)}
   \end{matrix}
   \right\}_{1 \times m}
$$

### 第一层
#### 单个神经单元

$$
\begin{equation}
\begin{aligned}

Z&=w_{1}\times x_{1}+w_{2}\times x_{2}+\cdots+w_{n_{0}}\times x_{n_{0}}+b

\\&=\left\{
   \begin{matrix}
   w^{(1)}_{1} \\
   w^{(1)}_{2} \\
   \vdots \\
   w_{n_{0}}
   \end{matrix}
   \right\}_{n_{0}\times m}^{T}
\cdot
   \left\{
   \begin{matrix}
   x^{(1)}_{1} & \cdots & x^{(m)}_{1} \\
   x^{(1)}_{2} & \cdots & x^{(m)}_{2} \\
   \vdots  & \ddots & \vdots \\
   x^{(1)}_{n_{0}}  & \cdots & x^{(m)}_{n_{0}}
   \end{matrix}
   \right\}_{n_{0}\times m}
+
   \left\{
   \begin{matrix}
   b & \cdots & b
   \end{matrix}
   \right\}_{1\times n_{0}}
\\&=W^{T}X+b

\end{aligned}
\end{equation}
$$

#### 推广
将单个神经单元的情况推广至第一层所有的神经单元

$$
\begin{equation}
\begin{aligned}

Z^{[1]}&=\left\{
   \begin{matrix}
   Z_{1}^{[1](1)} & \cdots & Z_{1}^{[1](m)}\\
   Z_{2}^{[1](1)} & \cdots & Z_{2}^{[1](m)}\\
    \vdots  & \cdots & \vdots \\
   Z_{n_{1}}^{[1](1)} & \cdots & Z_{n_{1}}^{[1](m)}\\
   \end{matrix}
   \right\}_{n_{1}\times m}

\\&=\left\{
   \begin{matrix}
   ...W^{[1]T}_{1} ...\\
   ...W^{[1]T}_{2}... \\
   \cdots  \\
   ...W^{[1]T}_{n_{1}}...
   \end{matrix}
   \right\}_{n_{1}\times n_{0}}
\cdot
   \left\{
   \begin{matrix}
   x^{(1)}_{1} & \cdots & x^{(m)}_{1} \\
   x^{(1)}_{2} & \cdots & x^{(m)}_{2} \\
   \vdots  & \ddots & \vdots \\
   x^{(1)}_{n_{0}}  & \cdots & x^{(m)}_{n_{0}}
   \end{matrix}
   \right\}_{n_{0}\times m}
+
   \left\{
   \begin{matrix}
   b^{[1]}_{1} & \cdots & b^{[1]}_{1} \\
   b^{[1]}_{2} & \cdots & b^{[1]}_{2} \\
   \vdots  & \cdots & \vdots \\
   b^{[1]}_{n_{1}} & \cdots & b^{[1]}_{n_{1}} \\
   \end{matrix}
   \right\}_{n_{1}\times m}
\\&=W^{[1]}X+b^{[1]}

\end{aligned}
\end{equation}
$$

#### 激活函数

$$
\begin{equation}
\begin{aligned}

A^{[1]}=\sigma(Z^{[1]})&=\left\{
   \begin{matrix}
    \sigma(Z_{1}^{[1](1)})  & \cdots & \sigma(Z_{1}^{[1](m)}) \\
  \sigma(Z_{2}^{[1](1)})  & \cdots & \sigma(Z_{2}^{[1](m)}) \\
   \vdots  & \cdots & \vdots \\
  \sigma(Z_{n_{1}}^{[1](1)})  & \cdots & \sigma(Z_{n_{1}}^{[1](m)}) \\
   \end{matrix}
   \right\}_{n_{1}\times m}
\\&=\left\{
   \begin{matrix}
    a_{1}^{[1](1)}  & \cdots & a_{1}^{[1](m)} \\
  a_{2}^{[1](1)}  & \cdots & a_{2}^{[1](m)} \\
   \vdots  & \cdots & \vdots \\
  a_{n_{1}}^{[1](1)}  & \cdots & a_{n_{1}}^{[1](m)} \\
   \end{matrix}
   \right\}_{n_{1}\times m}

\end{aligned}
\end{equation}
$$

### 第二层
节点个数 n<sub>1</sub> = 0  
输入为矩阵 A<sup>[1]</sup>

$$
\begin{equation}
\begin{aligned}

Z^{[2]}&=\left\{
   \begin{matrix}
   ...W^{[2]T}_{n_{2}} ...
   \end{matrix}
   \right\}_{1\times n_{1}}
\cdot
   \left\{
   \begin{matrix}
   a_{1}^{[1](1)} & \cdots & a_{1}^{[1](m)}\\
   a_{2}^{[1](1)} & \cdots & a_{2}^{[1](m)}\\
    \vdots  & \cdots & \vdots \\
   a_{n_{1}}^{[1](1)} & \cdots & a_{n_{1}}^{[1](m)}\\
   \end{matrix}
   \right\}_{n_{1}\times m}
+
   \left\{
   \begin{matrix}
   b^{[2]} & \cdots & b^{[2]}
   \end{matrix}
   \right\}_{1\times m}
\\&=W^{[2]}A^{[1]}+b^{[2]}

\end{aligned}
\end{equation}
$$  

计算激活函数

$$
\begin{equation}
\begin{aligned}

\widehat{Y}=A^{[2]}=\sigma(Z^{[2]})&=
   \left\{
   \begin{matrix}
    \sigma(Z^{[2](1)})  & \cdots & \sigma(Z^{[2](m)}) \\
   \end{matrix}
   \right\}_{1\times m}
\\&=\left\{
   \begin{matrix}
    A^{[2](1)}  & \cdots & A^{[2](m)} \\
   \end{matrix}
   \right\}_{1\times m}

\end{aligned}
\end{equation}
$$

至此，正向传播计算完毕

## 反向传播
反向传播采用梯度下降算法
### 计算成本函数

$$
J(w,b)=-\frac{1}{m}\times\sum_{k=1}^{m}[ y\times ln\hat{y}+(1-y)\times ln(1-\hat{y})]
\\令 L(a,y)=-[ y\times lna+(1-y)\times ln(1-a)]
\\则 J(w,b)=\frac{1}{m}(L^{(1)}+L^{(2)}+\cdots+L^{(m)})
$$

> $$ \hat{y}$$与a表示相同的值

### 计算偏导
计算偏导的顺序要从后往前，运用

$$ \frac{\partial J}{\partial w} =\frac{\partial J}{\partial a} \times \frac{\partial a}{\partial z}\times\frac{\partial z}{\partial w} $$

#### 第二层

$$
 \frac{\partial L}{\partial a} = -\frac{y}{a}+\frac{1-y}{1-a}
\\ \frac{\partial a}{\partial z} = \frac{e^{-z}}{(1+e^{-z})^{2}}
\\ \Rightarrow \frac{\partial L}{\partial z} = \frac{\partial L}{\partial a}\times\frac{\partial a}{\partial z}=a-y
\\
$$

$$
\frac{\partial z}{\partial w} = a^{[1]}
\\ \frac{\partial z}{\partial b} = 1
\\ \Rightarrow \frac{\partial J}{\partial w}=\frac{1}{m}\sum_{k=1}^{m}x^{(i)}(a^{(i)}-y^{(i)})
\\ \frac{\partial J}{\partial b}=\frac{1}{m}\sum_{k=1}^{m}(a^{(i)}-y^{(i)})
$$

由以上公式实现向量化

$$
\frac{\partial L}{\partial Z^{[2]}}=\left\{
   \begin{matrix}
   A^{[2](1)}-y^{(1)}  & \cdots & A^{[2](m)}-y^{(m)}
   \end{matrix}
   \right\}_{1\times m}=A^{[2]}-Y
$$

$$
\begin{equation}
\begin{aligned}
   \frac{\partial J}{\partial W^{[2]}}&=\frac{1}{m}
   \left\{\begin{matrix}
   A^{[2](1)}-y^{(1)}  & \cdots & A^{[2](m)}-y^{(m)}
   \end{matrix}
   \right\}_{1\times m}
\cdot
    \left\{
   \begin{matrix}
    a_{1}^{[1](1)}  & \cdots & a_{1}^{[1](m)} \\
  a_{2}^{[1](1)}  & \cdots & a_{2}^{[1](m)} \\
   \vdots  & \cdots & \vdots \\
  a_{n_{1}}^{[1](1)}  & \cdots & a_{n_{1}}^{[1](m)} \\
   \end{matrix}
   \right\}_{n_{1}\times m}^{T}
\\&=(\frac{1}{m}\times(A^{[2]}-Y)\cdot A^{[1]T})_{1\times n_{1}}
\\ \frac{\partial J}{\partial b^{[2]}}&=\frac{1}{m}sum(A^{[2](m)}-y^{(m)})
\end{aligned}
\end{equation}
$$

#### 第一层

$$
\frac{\partial L}{\partial Z^{[2]}} = A^{[2]}-Y
\\ \frac{\partial Z^{[2]}}{\partial A^{[1]}} = W^{[2]}
\\ \frac{\partial A^{[1]}}{\partial Z^{[1]}}=\frac{e^{-Z^{[1]}}}{(1+e^{-Z^{[1]}})^{2}}
\\ \frac{\partial Z^{[1]}}{\partial W^{[1]}} = x
\\ \frac{\partial Z^{[1]}}{\partial b^{[1]}} = 1
$$

由以上公式实现向量化

$$
\begin{equation}
\begin{aligned}
   \frac{\partial L}{\partial A^{[1]}}&=\left\{\begin{matrix}
   \cdots & W^{[2]}_{n_{2}} & \cdots
   \end{matrix}
   \right\}_{1 \times n_{1}}^{T}
\cdot
   \left\{\begin{matrix}
   A^{[2](1)}-y^{(1)}  & \cdots & A^{[2](m)}-y^{(m)}
   \end{matrix}
   \right\}_{1\times m}
\\&=(W^{[2]T} \cdot (A^{[2]}-Y))_{n_{1} \times m}
\end{aligned}
\end{equation}
$$

$$
\begin{equation}
\begin{aligned}
\\ \frac{\partial L}{\partial Z^{[1]}}=\frac{\partial L}{\partial A^{[1]}} \cdot \frac{\partial A^{[1]}}{\partial Z^{[1]}}&=   
   \left\{
   \begin{matrix}
   w^{[2]}_{1} \cdot (A^{[2](1)}-y^{(1)}) & \cdots & w^{[2]}_{1} \cdot (A^{[2](m)}-y^{(m)}) \\
   w^{[2]}_{2} \cdot (A^{[2](1)}-y^{(1)})  & \cdots & w^{[2]}_{2} \cdot (A^{[2](m)}-y^{(m)}) \\
   \vdots  & \ddots & \vdots \\
   w^{[2]}_{n_{1}} \cdot (A^{[2](1)}-y^{(1)})   & \cdots & w^{[2]}_{n_{1}} \cdot (A^{[2](m)}-y^{(m)})
   \end{matrix}
   \right\}_{n_{1}\times m} * (\frac{\partial A^{[1]}}{\partial Z^{[1]}})_{n_{1} \times m}
\\&=(W^{[2]T} \cdot (A^{[2]}-Y) \cdot \frac{e^{-Z^{[1]}}}{(1+e^{-Z^{[1]}})^{2}})_{n_{1} \times m}
\end{aligned}
\end{equation}
$$

> \* 代表矩阵对应元素相乘

以下计算类似第二层，故只给出结果

$$
\frac{\partial J}{\partial W^{[1]}}=\frac{1}{m} \cdot \frac{\partial L}{\partial Z^{[1]}} \cdot X^{T}
\\ \frac{\partial J}{\partial b^{[1]}}=\frac{1}{m}sum(\frac{\partial L}{\partial Z^{[1]}},axis=1)
$$

> 第二条公式表示矩阵按行求和

至此所有偏导数求完，之后需要用$$\frac{\partial J}{\partial W^{[1]}}、\frac{\partial J}{\partial b^{[1]}}、\frac{\partial J}{\partial W^{[2]}}、\frac{\partial J}{\partial b^{[2]}}$$ 以及学习率对W<sup>[1]</sup>，W<sup>[2]</sup>，b<sup>[1]</sup>，b<sup>[2]</sup>更新，并持续循环
