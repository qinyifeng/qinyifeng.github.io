---
layout: post
title:  "线代随笔15-常系数线性矩阵微分方程"
categories: [linear-algebra]
---


## 常系数线性微分方程

$$
  \frac{dy}{dt} = ay
$$


其解为，

$$
  y = Ce^{at}
$$

直接代入可以验证。

## 扩展到矩阵形式

$$
  \frac{du}{dt}=Au
$$

通解形式，

$$
  u = \sum_{i=1}^n{C_ie^{\lambda_i t}x_i} \qquad (1)
$$

其中$x_i$是特征向量，$\lambda_i$是特征值，$C_i$是常数，通过初始向量$u(0)$与特征向量$x_1,\cdots,x_n$确定。

## $2 \times 2$矩阵

$$
  \frac{du_1}{dt}=-u_1+2u_2,
  \frac{du_2}{dt}=u_1-2u_2,
  u(0)=\begin{bmatrix}1 \\ 0\end{bmatrix}
$$

将上面的方程组用向量与矩阵表示，如下

$$
  \frac{du}{dt}=\begin{bmatrix}-1 & 2 \\ 1 &-2\end{bmatrix}
  \begin{bmatrix}u_1 \\ u_2\end{bmatrix} = Au
$$

A是奇异矩阵，那么必有$\lambda_1 = 0$，再由特征值之和等于矩阵迹，容易得到$\lambda_2=-3$，后面再用通用计算方法验证一下。

根据特征值，计算出特征向量如下，

$$
  x_1=\begin{bmatrix}2 \\ 1\end{bmatrix},
  x_2=\begin{bmatrix}1 \\ -1\end{bmatrix}
$$

$2\times 2$矩阵解的通用形式如下

$$
  u(t)=C_1e^{\lambda_1t}x_1 + C_2e^{\lambda_2t}x_2 
$$

纯解，可以将(1)代入到微分方程，很容易得到两边相等，剩下的问题就是确定常数$C$。上面的公式恒等，那么在$t=0$的情况下也成立，直接代入，等价解一个二元线性方程组，如下

$$
  C_1\begin{bmatrix}2 \\ 1\end{bmatrix} 
  + C_2\begin{bmatrix}1 \\ -1\end{bmatrix}
  = \begin{bmatrix}1 \\ 0\end{bmatrix}
$$

最后得到$C_1=\frac{1}{3},C_2=\frac{1}{3}$，完成解如下：

$$
  u(t)=\frac{1}{3}e^{0 \cdot t} \begin{bmatrix}2 \\ 1\end{bmatrix}
  + \frac{1}{3}e^{-3t} \begin{bmatrix}1 \\ -1\end{bmatrix}
$$

随着时间推移，$u(0)$的最后达到稳态，$u_0$一部分转移到$u_1$，在无限大时，可得到如下

$$
  u(\infty) = \frac{1}{3} \begin{bmatrix}2 \\ 1\end{bmatrix}
$$

上面最后收敛有点类似马尔科夫矩阵，列和为0，总有一个特征值为0，最后也具有稳定状态。最后稳定后，各个元素的之和与初始矩阵个元素之和相等。

如果最大特征值是0,其他特征值实数部分小于0，那么具有稳定状态，虚数部分只负责转圈，不会对模长度产生影响。但是，如果矩阵反号，那么所有**特征值反号**，没有稳定态，最后会发散。

对于$2 \times 2$矩阵，如果需要达到稳定状态，特征值需要满足下面的要求，

* $Re(\lambda_1)+Re(\lambda_2) \le 0$ 
* $Re(\lambda_1) Re(\lambda_2) \ge 0$

一旦满足上面的条件，可以保证符合稳定的条件,$A=[0]_{2\times 2}$等号成立。

## 简化表示

下面定义一种更加优雅的形式，表示上述微分方程解，首先定义幂矩阵，启发来自$e^x$的泰勒展开

$$
  e^x = 1 + x + \frac{1}{2}x^2+\frac{1}{6}x^3+\cdots+\frac{1}{n!}x^n+\cdots
$$

那么，将形如$e^{At}$按照上面形式展开，结果仍然是矩阵！

$$
  e^{At} = I + At + \frac{1}{2}(At)^2+\frac{1}{6}(At)^3+\cdots+\frac{1}{n!}(At)^n+\cdots
$$

对$t$求导，得到如下值

$$
  \frac{d(e^{At})}{dt} = A + At+A\frac{1}{2}(At)^2+\cdots+A\frac{1}{(n-1)!}(At)^{n-1}+\cdots = Ae^{At}
$$

所以$u(t) = e^{At}u(0)$是微分方程的解。

假设特向量足够，那么可以将上面的公式进一步简化，**得到与之前微分方程解的联系**，

$$
\begin{align}
  e^{At}&=I + S \Lambda S^{-1}t + \frac{1}{2}(S \Lambda S^{-1}t)(S \Lambda S^{-1}t) + \cdots \\
        &=S(I + \Lambda t+ \frac{1}{2} \Lambda^2t + \cdots)S^{-1} \\
        &=Se^{\Lambda t}S^{-1} \qquad (2)
\end{align}
$$

将(2)代入解中，

$$
  u(t) = Se^{\Lambda t}S^{-1}u(0) = Se^{\Lambda t}S^{-1}Sc=Se^{\Lambda t}c 
  =\begin{bmatrix} x_1 & \cdots & x_n \end{bmatrix} 
  \begin{bmatrix} 
    e^{\lambda_1 t} &   &   \\
    & \ddots & \\
    & & e^{\lambda_n t} \\
  \end{bmatrix} 
  \begin{bmatrix} C_1 \\ \vdots \\ C_n \end{bmatrix} 
  =\sum_{i=1}^n{C_i e^{\lambda_i t} x_i}
$$

## 应用：添加常系数

在网络扩展建模中会用到此形式，求解下面微分方程

$$
  \frac{du}{dt}=\beta Au
$$

通解仍是

$$
  u = C e^{\beta \lambda t}x
$$

代入微分方程

$$
  Left = \frac{d(C e^{\beta \lambda t}x)}{dt} = C \beta e^{\beta \lambda t} \lambda x
$$

$$
  Right =\beta A C e^{\beta \lambda t}x = C \beta e^{\beta \lambda t} Ax =C \beta  e^{\beta \lambda t} \lambda x
$$


常量$C$计算与之前一致，完毕！

## 应用：高阶线性微分方程

求解

$$
  ay^{(3)}+by^{(2)}+cy^{(1)}+dy + e = 0
$$

变化

$$
  y^{(3)}= -\frac{b}{a}y^{(2)} - \frac{c}{a}y^{(1)} -\frac{d}{a}y - \frac{e}{a}, a \ne 0
$$

设

$$
  u=\begin{bmatrix}y^{(2)} \\ y^{(1)} \\  y \\ 1 \end{bmatrix}
$$


构建线性矩阵微分方程


$$
  \frac{du}{dt} 
  = \begin{bmatrix}y^{(3)} \\ y^{(2)} \\  y^{(1)} \\ 0 \end{bmatrix}
  =
  \begin{bmatrix}  
   -\frac{b}{a} & - \frac{c}{a} & -\frac{d}{a} & - \frac{e}{a} \\  
  1 & 0 & 0 & 0 \\  
  0 & 1 & 0 & 0 \\ 
  0 & 0 & 0 & 0 \end{bmatrix}
  \begin{bmatrix}y^{(2)} \\ y^{(1)} \\  y \\ 1 \end{bmatrix}
$$

使用特征向量计算微分方程，线性代数与微积分还是有联系的，当时看到这里我就震惊了，有点脑洞大开的感觉，而且用级数表示矩阵，也感觉有点小惊喜。果然看到的越多，才越知道自己渺小。
