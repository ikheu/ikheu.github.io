---
layout: post
title:  用于水和水蒸汽物性计算的 Python 模块——iapws
categories: 物理
tags: Python
---
{:toc}


在进行热力循环分析、流动传热计算时，需获得水和水蒸汽的物性参数。网上主流的水蒸汽物性计算程序是上海成套所的杨宇教授开发的，有 Fortran、C、C#、VB 等多个语言版本，还有桌面版本，被本专业学生和研究人员广泛使用。可以说杨教授为同行的便利做了很大贡献，本想贴一下他的个人博客的，但发现他的博客链接挂了。

最近打算使用混合编程，写个 Python 版本的水蒸汽物性计算的接口程序，搞个在线查询的页面，然而 google 后发现已经有了用于水和水蒸汽物性计算的 Python 模块—— iapws，不得不安利给大家。其实传统的工科领域用 Python 的不多，要是有人用的话希望别重复造轮子。

详见：[iapws 官网](ttps://github.com/jjgomera/iapws/tree/master/docs)

---

## 简介

iapws 是 [IAPWS](http://www.iapws.org/release.html) 标准的 Python 实现，包含以下几个模块：

- IAPWS-IF97——水蒸汽
- IAPWS-95——水蒸汽
- IAPWS-06——冰
- IAPWS-08——海水
- IAPWS-17—— 重水
- ......

iapws 依赖于 numpy-scipy 科学计算模块。本文主要介绍 IAPWS-IF97 模块的使用。IAPWS-IF97 实现了 5 个区域的基本方程(下图)。可以看出压力、温度的范围是很宽的，足够满足一般工程需要。

![](/assets/img/steam.png)


## 使用

直接在控制台执行：pip install iapws，安装 iapws 时会自动安装 numpy、scipy 这两个模块。

### APWS97 类

可以使用该 IAPWS97 类创建一个特定热力学状态的对象，该类的构造函数的关键字参数包括：

- T：温度[K]
- P：压力[MPa]
- h：比焓[kJ/kg]
- s：比熵[kJ/kgK]
- x：干度[-]

有效的参数组合有：

- T, P：对两相无效
- P, h
- P, s
- h, s
- T, x：仅适用于两相
- P, x：仅适用于两相

计算的物性参数如下表所示。我将自己认为常用的参数列在前面了。许多参数不知道什么意思，翻译也不知道有没有问题。

|-|-|-|
|-|-|-|
|P：压力[MPa]|a：亥姆霍兹自由能[kJ / kg]|joule：焦耳 - 汤姆森系数[K / MPa]|
|T：温度[K]|Z：压缩系数[ - ]|deltat：等温节流系数[kJ / kg·MPa]|
|v：比容量[m³/ kg]|fi：逸度系数[ - ]|region：地区|
|rho：密度[kg /m³]|f：逸度[MPa]|v0：理想比容[m³/ kg]|
|h：比焓[kJ / kg]|γ：等熵指数[ - ]|u0：理想的内能[kJ / kg]|
|u：特定内能[kJ / kg]|alfav：等压膨胀系数[1 / K]|h0：理想比焓[kJ / kg]|
|s：比熵[kJ / kg·K]|xkappa：等温压缩率[1 / MPa]|s0：理想比熵[kJ / kg·K]|
|cp：定压比热[kJ / kg·K]|kappas：绝热可压缩率[1 / MPa]|a0：理想亥姆霍兹自由能[kJ / kg]|
|cv：定容比热[kJ / kg·K]|alfap：相对压力系数[1 / K]|g0：理想比吉布斯自由能[kJ / kg]|
|g：比Gibbs自由能[kJ / kg]|betap：等温应力系数[kg /m³]|cp0：理想定压比热[kJ / kg·K]|
|n：折射率[ - ]|Pr：折算压力[ - ]|cv0：理想定容比热[kJ / kg·K]|
|Prandt：普朗特数[ - ]|Tr：折算温度[ - ]|Svap：蒸发熵[kJ / kg·K]|
|μ：动态粘度[Pa·s]|w0：理想音速[m / s]|gamma0：理想等熵指数[ - ]|
|nu：运动粘度[m²/ s]|k：导热系数[W / m·K]|epsilon：介电常数[ - ]|
|w：音速[m / s]|alfa：热扩散系数[m²/ s]||
|Hvap：汽化热[kJ / kg]|sigma：表面张力[N / m]||

查看源码发现，使用 IAPWS97 类进行物性查询时，先执行 calculable 方法判断输入条件是否可计算，若可以计算则执行 calculo 方法，判断输入参数确定的物性状态所处的区域，随后计算各物性。下面是示例：

```
>>> from iapws import IAPWS97
#>>>> 常压常温水 <<<<
>>> water=IAPWS97(T=24+273.15,P=0.013)
# 焓值
>>> water.h
100.66509664191254
# 密度
>>> water.rho
997.2595184928771
#>>>> 高压蒸汽 <<<<
>>> vapor = IAPWS97(P=15.5, x=1.0)
# 温度
>>> vapor.T
617.9415516035506
# 饱和汽焓
>>> vapor.h
2596.2167214338015
# 饱和水焓
>>> IAPWS97(P=15.5, x=0).h
1629.8502994294881
```

当输入无效的参数组合时，实例会正常产生但不进行物性计算，实例的 calculable 方法返回空字符串；当输入有效的参数组合时，若可查询则 calculable 方法返回查询组合，若超出查询范围则抛出异常。

```
#>>>> 无效的输入 <<<<
>>> test1 = IAPWS97(h=2000.0, T=300.0)
# 未绑定属性h
>>> test1.h
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'IAPWS97' object has no attribute 'h'
# calculable返回空字符串
>>> test1.calculable
''
#>>>> 有效的输入 <<<<
>>> vapor = IAPWS97(P=15.5, x=1.0)
# calculable返回输入的组合
>>> vapor.calculable
'Px'
#>>>> 超出查询范围的输入 <<<<
>>> test2 = IAPWS97(P=100.0, T=45.0)
Traceback (most recent call last):
...
NotImplementedError: Incoming out of bound
```

### 基本方程

IAPWS97 类提供了丰富的物性计算，若仅是为了查询物性，直接使用该类就行了。但在瞬态计算或其他需要频繁计算物性的场合下，通过 IAPWS97 类计算物性是不够明智的。因为一般情况下仅仅需要输出一个物性参数就足够了，如根据压力、焓值计算温度，而 IAPWS97 类计算了许多我们不需要的参数，这会拖慢程序的运行效率。本来程序具有超实时的运行能力，由于物性查询方法的错误使用，程序可能变得巨慢。物性查询成为程序的性能瓶颈，这显然是不合理的。

iapws.iapws97 模块提供了一系列的基本方程（fundamental equations）和向后方程（backward equation），用于特定场合下的物性计算。

iapws.iapws97 模块提供了如下基本方程，用于不同区域内的物性计算。形如 _Regionr 的函数，根据输入的参数，确定区域 r 内的状态，返回一个字典；_TSat_P、_PSat_T 这两个函数计算饱和线上的物性。

- _Region1(T, P)
- _Region2(T, P)
- _Region3(rho, T)
- _Region4(P, x)
- _Region5(s)
- _TSat_P(P)
- _PSat_T(s)

_Region1 和 _Region2 的输入均是 T、P，那么在使用时要预先判断 T、P 所指示的区域：

```
from iapws import iapws97
# >>>> 区域1计算示例 <<<<
# 确定区域
>>> iapws97._Bound_TP(300,3)
1
# 获得比容
>>> iapws97._Region1(300,3)['v']
0.0010021516796866943
# 获得比焓
>>> iapws97._Region1(300,3)['h']
115.3312730214384
# >>>> 区域2计算示例 <<<<
# 确定区域
>>> iapws97._Bound_TP(700,3)
2
# 获得比容
>>> iapws97._Region1(300,3)['v']
0.0010021516796866943
# 获得比熵
>>> iapws97._Region1(300,3)['s']
0.39229479240262427
```

饱和线上的物性计算：

```
# 计算饱和温度
>>> iapws97._TSat_P(15.5)
617.9415516035506
# 计算饱和压力
>>> iapws97._PSat_T(100+273.15)
0.10141797792131013
```

### 向后方程

向后方程指形如 _Backwardr_x_yz 的方程，其中 r 为物性区域，yz 为输入参数，x 为返回参数。

- _Backward1_T_Ph
- _Backward1_T_Ps
- _Backward1_P_hs
- _Backward2_T_Ph
- ...

需要注意，和基本方程一样，向后方程也不进行区域判断。当选择错误的函数时，将输出离奇的计算结果：

```
>>> from iapws import iapws97
# 错误的使用示例
>>> iapws97._Backward2_T_Ph(3,500)
4.1313215739117547e+21
# 错误的使用示例
>>> iapws97._Backward3_T_Ph(3,500)
-1637746.3600011615
# 判断区域
>>> iapws97._Bound_Ph(3,500)
1
# 正确的使用示例
>>> iapws97._Backward1_T_Ph(3,500)
391.7985087624256
```