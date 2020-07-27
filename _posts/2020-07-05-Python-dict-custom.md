---
layout: post
title: Python 自定义字典
categories: 语言特性
tags: Python
---
{:toc}

对 Python 字典进行扩展，支持像访问属性一样访问字典数据。

## 继承 dict

直接继承 `dict` 类，分别使用特殊方法 `__getattr__` 和 `__setattr__` 来支持字典数据的读和写。

```python
# 初版 JsDict，继承内置的 dict
class JsDict(dict):
    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError("JsDict object has no attribute '%s'" % key)
    def __setattr__(self, key, value):
        self[key] = value
```

特殊方法的使用除了要满足目标外，还要保证其他相关函数有正常的行为。判断对象的属性是否存在的 `hasattr` 函数与 `__getattr__` 有关。`hasattr` 调用 `getattr` 函数，看是否抛出 `AttributeError` 异常。`getattr(obj, 'attr')` 等价于点操作 `obj.attr`，它首先在 `__dict__` 中查找属性，没找到就会调用 `__getattr__`。因此在 `__getattr__` 中必须捕获 `KeyError` 并抛出 `AttributeError`，这样使 `hasattr` 有合理的输出，而不抛出令人疑惑的 `KeyError`。

`getattr`，`__getattr__` 和 `__getattribute__` 在命名和行为上容易搞错，涉及这些方法的定义时要小心处理。

下面是使用 `JsDict` 的一个示例，似乎已经大功告成了。

```python
>>> student = JsDict({'name': 'Bob', 'age': 12})
>>> student.name
'Bob'
>>> hasattr(student, 'gender')
False
```

没这么简单，还要考虑一些特殊的键名。字典有内置的 `values` 方法用于获取字典的所有值，而当字典也有名为 "values" 的键会怎样？下面进一步的测试可以看出，点操作的属性名为 "values" 时，可以修改但无法读取字典结构。很幸运我们还能使用 `values` 方法，但字典的行为变得不一致了。而当属性名为关键字 "class" 时，直接报语法错误了。

```python
>>> student.values = [90]
>>> student
{'name': 'Bob', 'age': 12, 'values': [90]}
>>> student.values
<built-in method values of JsDict object at 0x107dc4f10>
>>> student['class'] = 'A'
>>> student.class
  File "<stdin>", line 1
    student.class
                ^
SyntaxError: invalid syntax
```

解决办法是内部悄悄地把键置换成别的名字，如加个后缀 "_"。上述示例的预期行为是：

```python
>>> student = JsDict({'name': 'Bob', 'age': 12})
>>> student.values = [90]
>>> student
{'name': 'Bob', 'age': 12, 'values_': [90]}
>>> student.values_
[90]
>>> student['class'] = 'A'
>>> student
{'name': 'Bob', 'age': 12, 'values_': [90], 'class_': 'A'}
>>> student.class_
'A'
```

进行 [] 操作会调用 `__setitem__`，所以可以给类增加 __setitem__ 方法：

```python
class JsDict(dict):
    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError("JsDict object has no attribute '%s'" % key)
    
    def __setattr__(self, key, value):
        self[key] = value
    
    def __setitem__(self, key, value):
        print('set', key)
        super().__setitem__(key, value)
```

然而测试发现，没有 `print` 的输出，初始化时 `__setitem__` 未被调用。

```
>>> student = JsDict({'name': 'Bob', 'age': 12,  'class': 'A'})
>>> student
{'name': 'Bob', 'age': 12, 'class': 'A'}
```

实际上，内置的类型 `dict` 使用 C 语言实现，在执行上会走一些捷径，可能会忽略用户定义的方法，导致子类的行为与预期不符。可以使用 `collections.UserDict `来进行扩展，而不要使用内置类型。

## 继承 UserDict

`UserDict` 使用 Python 实现了一遍字典。其内部维护了一个 `dict` 实例 `self.data`，操作 `self.data` 而非 `self` 可以有效避免无限递归。`collections` 中还有 `UserList` 和 `UserString`，分别用于列表和字符串的扩展。

下面是子类化 `UserDict` 的完整代码：

```python
import keyword
from collections import UserDict


class JsDict(UserDict):
    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError("JsDict object has no attribute '%s'" % key)
    
    def __setattr__(self, key, value):
        if key == 'data':   # <1>
            super().__setattr__(key, value)
            return
        self[key] = value
    
    def __setitem__(self, key, value):
        key = self._get_key(key) # <2>
        value = self.handle_value(value) # <3>
        super().__setitem__(key, value) # <4>
    
    def __getitem__(self, key):
        key = self._get_key(key)
        return super().__getitem__(key) # <5>
    
    # 处理键
    def _get_key(self, key):
        if hasattr(self.__class__, key) or keyword.iskeyword(key): # <6>
            key += '_'
        return key

    # 处理嵌套的值
    def handle_value(self, value):
        if isinstance(value, dict):
            value = JsDict(value)
        elif isinstance(value, list):
            for i, item in enumerate(value):
                value[i] = self.handle_value(item)
        return value
```

对代码中标注的序号作说明：

1. 当键为 `data` 时按超类行为处理。`UserDict` 在初始话时新建属性 `data`，字典中也可能有同名的键，所以需要避免在执行属性赋值时覆盖 `data`。
2. 处理键，详见 <6>。
3. 考虑 `value` 可能存在嵌套字典和数组的情况，这时要支持形如：`obj.attr1.attr2`，`obj.attr3[0].attr4` 的形式。
4. 鉴于 `UserDict` 已经实现了 `__setitem__`，它将键值对存储在 `self.data` 中，因此这里仅仅处理键而不修改超类的行为。
5. `UserDict`  `__getitem__` 方法对 `__missing__` 属性做了特殊处理，来处理未找到键的情况。所以这里和上面一样直接调用超类的方法，保证了自定义类功能的齐全。
6. 当 `key` 已经是类属性或关键字时，增加后缀 '_'。

可以发现，即使只是对字典支持简单操作这种简单扩展，也不是那么容易的。需要考虑的问题有很多，如异常处理、特殊方法的正确使用以及类的合理选择等。
