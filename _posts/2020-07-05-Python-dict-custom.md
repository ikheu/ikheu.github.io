---
layout: post
title: yield from 原理
categories: 语言特性
tags: Python
---
{:toc}



```python
from collections import UserDict

class JsDict(UserDict):
    def __getattr__(self, key):
        return self.data[key]

    def __setattr__(self, key, value):
        self.data[key] = value

```