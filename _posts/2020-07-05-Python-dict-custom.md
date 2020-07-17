---
layout: post
title: Python 自定义字典
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