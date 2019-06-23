---
title: Chapter 10 - 序列的修改、散列和切片
tags:
  - Python
categories:
  - Fluent Python
date: 2019-04-27 12:07:46
---



## 10.2 与旧的类兼容

> 序列类型的构造方法最好接受可迭代的对象为参数，因为所有内置的序列类型都是这样做的。

`reprlib` 能够为某些对象生成长度有限的表示形式:

```python
In [1]: import reprlib

In [3]: a
Out[3]:
[0,
 1,
 2,
 3,
 4,
 5,
 6,
 7,
 8,
 9,
 ...
  999]

In [4]: reprlib.repr(a)
Out[4]: '[0, 1, 2, 3, 4, 5, ...]'
```

注意，在 Python2 中，`reprlib` 模块的名字是 `repr`。

> 调用 `repr()` 函数的目的是调试，因此绝对不能抛出异常。如果 `__repr__` 方法的实现有问题，那么必须处理，尽量输出有用的内容，让用户能够识别目标对象。

## 10.3 协议和鸭子类型

在面向对象编程中，协议是非正式的接口，只在文档中定义，在代码中不定义。例如，Python 的序列协议只需要 `__len__` 和 `__getitem__` 两 方法。任何类（如 `Spam`），只要使用标准的签名和语义实现了这两个方法，就能用在任何期待序列的地方。`Spam` 是不是哪个类的子类无关紧要，只要提供了所需的方法即可。

因此，Spam 的行为就像是一个序列，人们称其为鸭子类型（duck typing）。

协议是非正式的，没有强制力，因此如果你知道类的具体使用场景，通常只需要实现一个协议的部分。例如，为了支持迭代，只需实现 `__getitem__` 方法，没必要提供 `__len__` 方法。

> 协议可以视为鸭子类型语言使用的非正式接口。

## 10.4 可切片的序列

### 10.4.1 切片原理

```python
In [5]: class MySeq:
   ...:     def __getitem__(self, index):
   ...:         return index
   ...:

In [6]: s = MySeq()

In [7]: s[1]
Out[7]: 1

In [8]: s[1:4]
Out[8]: slice(1, 4, None)

In [9]: s[1:4:2]
Out[9]: slice(1, 4, 2)

In [10]: s[1:4:2, 9]
Out[10]: (slice(1, 4, 2), 9) 

In [11]: s[1:4:2, 7:9]
Out[11]: (slice(1, 4, 2), slice(7, 9, None))
```

这些步骤需要重点说明：

- 9：从 1 开始，到 4 结束，步幅为 2。
- 10：如果 [] 中有逗号，那么 `__getitem__` 收到的是元组。
- 11：元组中可以包含多个切片对象。

这里值得一提的是，`slice` 内置了 `indices` 方法，该方法开放了内置序列实现的棘手逻辑，用于优雅地处理缺失索引和负数索引，以及长度超过目标序列的切片：

```
S.indices(len) -> (start, stop, stride)
```

给定长度为 `len` 的序列，计算 `S` 表示的扩展切片的起始（start）和结尾（stop），以及步幅（stride）。超出边界的索引会被截掉，这与常规切片的处理方式相同。我们来看看这个方法的实际效果：

```python
In [12]: slice(None, 10, 2).indices(5)
Out[12]: (0, 5, 2)

In [13]: slice(-3, None, None).indices(5)
Out[13]: (2, 5, 1)
```

### 10.4.2 能处理切片的 `__getitem__` 方法

下面展示了一个将 `__getitem__` 委托给内部序列的实现：

```python
    def __getitem__(self, index):
        cls = type(self)
        if isinstance(index, slice):
            # 1
            return cls(self._components[index])
        # 2
        elif isinstance(index, numbers.Integral):
            return self._components[index]
        else:
            msg = '{cls.__name__} indices must be integers'
            raise TypeError(msg.format(cls=cls))
```

这里需要说明这些细节：

- 1：对于切片操作，返回自身类型的实例而不是内部序列的切片。
- 2：测试 `index` 是否是整数时用的是 `numbers.Integral`，这是一个抽象基类。在 `isinstance` 中使用抽象基类做测试能让 API 更灵活且更容易更新。

## 10.5 动态存取属性

属性查找失败后，解释器会调用 `__getattr__` 方法。简单来说，对 `my_obj.x` 表达式，Python 会检查 `my_obj` 实例有没有名为 `x` 的属性；如果没有，到类（`my_obj.__class__`）中查找；如果还没有，顺着继承树继续查找。如果依旧找不到，调用 `my_obj` 所属类中定义的 `__getattr__` 方法，传入 `self` 和属性名称的字符串形式（如 `'x'`）。

## 10.6 散列和快速等值

如果要快速比较两个序列是否相同，可以使用 `zip` 函数来实现：

```python
def __eq__(self, other):
    return len(self) == len(other) and \
    all(a == b for a, b in zip(self, other))
```





