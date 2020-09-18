---
layout: post
title: 递归及其优化
categories: 语言
tags: 语言特性
---
{:toc}


## 递归的历史


## Python 中对递归的限制

```python
def sum0(nums):
    if nums == []:
        return 0
    else:
        return nums[0] + sum0(nums[1:])
```

## 使用 sys

```python
import sys
sys.setrecursionlimit(10000000)
```

## 分治降低递归树深度

```python
def sum1(nums):
    if nums == []:
        return 0
    else:
        mid = len(nums) // 2
        return sum1(nums[0:mid]) + nums[mid] + sum1(nums[mid+1:])
```


## 使用尾递归

```python
def sum1(nums):
    if nums == []:
        return 0
    else:
        mid = len(nums) // 2
        return sum1(nums[0:mid]) + nums[mid] + sum1(nums[mid+1:])
```