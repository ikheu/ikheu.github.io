---
layout: post
title:  点堆中子动力学方程求解程序
excerpt: ''
categories: 物理
tags: Fortran
mathjax: true
---

* content
{:toc}

我学的是核科学与技术专业，本科和硕士期间写了不少程序，如计算堆芯中子动力学的，计算流体力学的。但多数仅针对具体问题，通用性有些不足。硕士修了一门仿真技术的课程，我的作业是求解点堆中子动力学程序，靠这个还拿了优秀。觉得这个程序还蛮通用的，对于同专业的搞个大作业，或用于毕设等或许有帮助。

详细的推导、验证过程、程序代码见：https://github.com/ikheu/point_reactor

## 概述

计算中子通量密度的瞬变特性，对反应堆动力系统的运行安全分析与仿真而言十分重要。点堆中子动力学模型是反应堆动态学中最常用的方法。

点堆中子动力学模型描述了中子密度和缓发中子先驱核浓度随时间变化的规律，基本方程如式（1）所示。由于瞬发中子寿命与缓发中子寿命间存在着数量级的差别，耦合的点堆动力学微分方程组存在着刚性，这给数值求解带来了一定困难。常规的显式方法，如欧拉法、龙格库塔法在计算该问题时会存在稳定性问题，这要求时间步长需取得很小，会带来计算耗时以及大的累计误差。针对点堆动力学方程组的刚性问题，发展了许多种处理方法，包括：
1. 线性多步法，如GEAR算法及其改进形式；
2. 基于积分方程的方法，如Hansen方法；
3. 分段多项式逼近法，如埃尔米特插值型；
4. 权重限制法；
5. 指数基函数法等。 

$$\begin{cases}
\frac{dn(t)}{dt} = \frac{\rho(t)-\beta}{dt}n(t)+\sum_{i=0}^I\lambda_{i}C_{i}(t)+q\\
\frac{dC_{i}(t)}{dt} = \frac{\beta_{i}}{\Lambda}n(t)-\lambda_{i}C_{i}(t)\quad\quad i=1,2,...,I
\end{cases}
\tag{1}$$

式中:

&emsp;&emsp;$n(t)$ 为中子密度；

&emsp;&emsp;$C_{i}(t)$ 为第i组缓发中子先驱核的浓度；

&emsp;&emsp;$\rho$ 为反应性；

&emsp;&emsp;$\Lambda$ 为中子代时间；

&emsp;&emsp;$\lambda_{i}$ 为第$i$组缓发中子先驱核衰变常数；

&emsp;&emsp;$\beta_{i}$ 为第$i$组缓发中子份额；

&emsp;&emsp;$\beta$ 为缓发中子的总份额；

&emsp;&emsp;$q$ 为外加中子源项。

本文使用一阶泰勒多项式积分方法求解点堆动力学方程，该方法能很好地解决方程的刚性问题，具有计算精度高、适用性好的特点。

## 一阶泰勒多项式积分方法

详细的推导过程有些繁琐，这里只说明其中两个关键的步骤。

第一个是积分处理。合并式 (1)，并在 $[t_{n}, t_{n+1}]$ 区间内积分，可获得式 (2)：

$$
n(t_{n+1}) - n({t_{n}}) = \int_{t_{n}}^{t_{n+1}}\frac{\rho(t)}{\Lambda}n(t)-\sum_{i=1}^{6}[C_{i}(t_{n+1}) - C_{i}(t_{n})] + q·h
\tag{2}
$$

第二个是对中子密度作一阶泰勒展开：

$$
n(\tau) = n(t_{n+1}) + n′(t_{n+1})(\tau-t_{n+1})
\tag{3}
$$

经过一系列的推导，可获得中子密度的表达式：

$$
n(t_{n+1}) = \frac
{n(t_{n}) + q \cdot h+ \sum_{i=1}^{6}\exp(-\lambda_{i}h)C_{i}(t_{n}) + \frac{(F_{2}-\sum_{i=1}^{6}G_{2,i})(\sum_{i=1}^{6}\lambda_{i}\exp(-\lambda_{i}h)C_{i}(t_{n})+q)}{1-\sum_{i=1}^{6}\frac{\lambda_{i}\beta_{i}}{\Lambda}G_{2,i}}}
{1-F_{1}+\sum_{i=1}^{6}\frac{\beta_{i}}{\Lambda}G_{1,i}-(F_{2}-\sum_{i=1}^{6}\frac{\beta_{i}}{\Lambda}G_{2,i})(\frac{\rho(t_{n+1})-\beta}{\Lambda}+\sum_{i=1}^{6}\frac{\lambda{i}\beta_{i}}{\Lambda}G_{1,i})/(1-\sum_{i=1}^{6}\frac{\lambda_{i}\beta_{i}}{\Lambda}G_{2,i})}\\
$$

$$
\tag{4}
$$

截断误差是该方法的主要误差来源，可考虑更高阶数的泰勒展开，当然这会使模型更加复杂（上式已经很复杂了），求解过程也不够简便。

<img src="http://cdn.moastro.cn/user_assets/5b5fc33c393e344a4f4dd646/20190705225812/calc_flow.png" width = "400px" />

首先给出初始条件，即 $t_{n}=0$ 时的 $n(t)$、 $C_{i}(t)$，确定已知项 $\rho(t)$、 $\lambda_{i}$、$\beta_{i}$、$q$ 等；然后给定计算条件：计算时长与时间步长；逐时刻依次计算下一时刻 $t_{n+1}=t_{n}+\Delta t$ 时的 $n(t_{n+1})$、$n$,$(t_{n+1})$ 与 $C_{i}(t_{n+1})$，直至达到设置的计算时间。

认为反应堆初始状态是稳定的，给定初始的中子密度 $n(0)$，根据式 (1) 可得：

$$
\frac{dC_{i}(t)}{dt} = 0 \Rightarrow C_{i}(0) = \frac{\beta_{i}}{\lambda_{i}\Lambda}n{0}
\tag{5}
$$

使用老掉牙的 Fortran90 程序编制的计算程序，可以在文章开头的 github 链接里找到。 

## 计算结果

这里给出了四种反应性引入，包括正反应性阶跃、负反应性阶跃、反应性线性变化、反应性正弦变化情形下，典型热中子堆的中子密度响应曲线：

![](http://cdn.moastro.cn/user_assets/5b5fc33c393e344a4f4dd646/20190705230207/results.jpg)

github链接中有个 [点堆模型与求解.pdf](https://github.com/ikheu/point_reactor/blob/master/%E7%82%B9%E5%A0%86%E6%A8%A1%E5%9E%8B%E4%B8%8E%E6%B1%82%E8%A7%A3.pdf) 的文件，可以在其中找到计算结果与解析解的比较和分析。计算程序还适用于快中子堆的求解。
