---
layout: post
title: 连分数及其程序实现
categories: 数学
tags: 连分数
mathjax: true
---
{:toc}

在数学中，将具有以下形式的分数称为连分数。所有分子都为 1，所以又称为正规连分数。当分子可以是其他值时，则属于广义连分数。对于正规的连分数，可以使用 $$[a_0, a_1, ..., a_n]$$ 这样的数组来表示。

![](/assets/img/contined_fraction.svg){:width="35%"}



## 连分数及其求解

对于有理数，可以表示为有限的连分数。对于无理数，自然没有办法使用有限的连分数来精确表示，而只能去近似表示。例如：

![](/assets/img/sqrt2.svg){:width="45%"}

对应的数组是：`[1, 2, 2, 2, ...]`，2 会重复出现。根 3 对应的数组是 `[1, 1, 2, 1, 2, ...]`，1 和 2 会重复出现。这种形式的连分数属于循环连分数，只有[二次无理数](https://zh.wikipedia.org/wiki/%E4%BA%8C%E6%AC%A1%E7%84%A1%E7%90%86%E6%95%B8)可以表示为这种形式。

基于连分数的形式，可以自然地使用递归来求解。下面是一种计算方法。`cf` 的参数 `f` 是一个函数，用来计算连分数数组。`count` 用来限制数组的大小。`fgen` 是一个生成器函数，用来生成数组的每一项元素。当生成器结束时，使用 `float('inf')` 来截断连分数。

```python
def cf(f, count):
    def _cf(gen):
        try:
            return next(gen) + 1 / _cf(gen)
        except StopIteration:
            return float('inf')
    gen = fgen(f, count)
    return _cf(gen)

def fgen(f, count):
    i = 0
    while i <= count:
        yield f(i)
        i += 1
```

使用程序计算几个连分数：

```python
# 1.2345
>>> cf(lambda x: [1, 4, 3, 1, 3, 1, 1, 2, 5][x], 8)
1.235
# √2
>>> cf(lambda x: 1 if x == 0 else 2, 10)
1.41421355164605467
# √3
>>> cf(lambda x: 1 if x == 0 or x % 2 == 1 else 2, 10)
1.7320490367775832
```

对于数学常数 $$e$$，因其不是二次无理数，连分数不属于循环连分数。但它的连分数也呈现一定的规律：

![](/assets/img/e.svg){:width="55%"}

下面是对常数 $$e$$ 的计算：

```python
def fe(x):
    if x == 0:
        return 2
    if x % 3 == 2:
        return (x + 1) // 3 * 2
    else:
        return 1

>>> cf(fe, 10)
2.7182817182817183
```

另一个常数圆周率 $$\pi$$ 的连分数却不具备规律：

![](/assets/img/pi.svg){:width="50%"}


## 有理数转化成连分数

计算一个有理数的连分数，可以采用辗转相除的方式。函数`f2cf` 可以接受 `float` 或者形如 `'xx.xxx'`, `'xx/xxx'` 的字符串形式。内部都会转化成分子分母为整数的形式进行处理，这样可以避免处理浮点数带来的误差问题。

```python
def f2cf(val):
    if isinstance(val, float):
        val = str(val)
    x, y = re.compile(r'[/, .]').split(val)
    if '.' in val:
        tmp = 10**len(y)
        x = int(x) * tmp + int(y)
        y = tmp
    else:
        x, y = int(x), int(y)
    arr = []
    gen = gcd(x, y)
    for item in gen:
        arr.append(item)
    return arr

def gcd(x, y):
    while y:
        yield x // y
        x, y = y, x % y
```

几个计算示例：

```python
>>> import math
# 有理数
>>> f2cf(1.333)
[1, 3, 333]
# √3 和 π 是无理数，由于是精度有限，仍可以计算
# √3，由于是近似形式，1，2 重复出现的规律会被打破
>>> f2cf(math.sqrt(3))
[1, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 1, 1, 40, 1, 2, 2, 1, 13, 1, 3, 2, 1, 1, 5, 1, 4, 3, 2]
# π
>>> f2cf(math.pi)
[3, 7, 15, 1, 292, 1, 1, 1, 2, 1, 3, 1, 14, 4, 2, 3, 1, 12, 5, 1, 5, 20, 1, 11, 1, 1, 1, 2]
```

## 广义连分数

上面提到 $$\pi$$ 无法的连分数不具有规律，但它可以表示成具有规律的广义连分数形式： 

![](/assets/img/pi1.svg){:width="80%"}

扩展函数 `cf`，以支持这种分子不是定值的情况：

```python
def cf_general(f1, f2, count):
    def _cf(gen1, gen2):
        try:
            num1, num2 = next(gen1), next(gen2)
            return num1 + num2 / _cf(gen1, gen2)
        except StopIteration:
            return float('inf')
    gen1, gen2 = fgen(f1, count), fgen(f2, count)
    return _cf(gen1, gen2)
```

对上图中最后一种广义连分数进行测试：

```python
>>> cf_general(lambda x: 0 if x == 0 else 2*x-1, lambda x: 4 if x == 0 else x**2, 20)
3.141592653589791
```

## 参考

- [什么是数学](https://book.douban.com/subject/10455982/)
- [连分数](https://zh.wikipedia.org/wiki/%E8%BF%9E%E5%88%86%E6%95%B0)
