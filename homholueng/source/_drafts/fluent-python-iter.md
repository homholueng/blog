---
title: Chapter 14 - 可迭代的对象、迭代器和生成器
tags:
  - Python
categories:
  - Fluent Python
---

## 14.1　Sentence类第1版：单词序列

我们要实现一个 Sentence 类，以此打开探索可迭代对象的旅程。我们向这个类的构造方法传入包含一些文本的字符串，然后可以逐个单词迭代。

```python
import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)
    
    def __getitem__(self, index):
        return self.words[index]
    
    def __len__(self):
        return len(self.words)
    
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)
```

注意，`reprlib.repr` 这个实用函数用于生成大型数据结构的简略字符串表示。

### 序列可以迭代的原因：iter函数

解释器需要迭代对象 x 时，会自动调用 `iter(x)`。

内置的 iter 函数有以下作用。

1. 检查对象是否实现了 `__iter__` 方法，如果实现了就调用它，获取一个迭代器。
2. 如果没有实现 `__iter__` 方法，但是实现了 `__getitem__` 方法， Python 会创建一个迭代器，尝试按顺序（从索引 0 开始）获取元素。
3. 如果尝试失败，Python 抛出 `TypeError` 异常，通常会提示“C object is not iterable”（C 对象不可迭代），其中 C 是目标对象所属的类。

任何 Python 序列都可迭代的原因是，它们都实现了 `__getitem__` 方法。其实，标准的序列也都实现了 `__iter__` 方法，因此你也应该这么做。之所以对 `__getitem__` 方法做特殊处理，是为了向后兼容，而未来可能不会再这么做。

**从 Python 3.4 开始，检查对象 x 能否迭代，最准确的方法是：调用 `iter(x)` 函数，如果不可迭代，再处理 `TypeError` 异常。**

## 14.2　可迭代的对象与迭代器的对比

可迭代对象：

- 实现了能返回迭代器的 `__iter__` 方法的对象
- 实现了 `__getitem__` 方法，且参数是从 0 开始的索引的对象

Python 会从可迭代对象中获取**迭代器**来进行迭代操作。

例如下面的例子，背后是有迭代器的：

```python
s = 'ABC'
for char in s:
    print(char)
```

如果没有 `for` 语句的话，则需要使用 `while` 来模拟：

```python
s = 'ABC'
it = iter(s)
while True:
    try:
        print(next(it))
    except StopIteration:
        del it
        break
```

标准的迭代器接口有两个方法：

- `__next__`：返回下一个可用的元素
- `__iter__`：返回 `self`，以便在可使用迭代对象的地方使用迭代器

这个接口在 `collections.abc.Iterator` 抽象基类中制定，该接口使用了 `__subclasshook__` 来判断某个类是否是其子类：

```python

class Iterator(Iterable):

    @classmethod
    def __subclasshook__(cls, C):
        if (any("__next__" in B.__dict__ for B in C.__mro__)) and
        (any("__iter__" in B.__dict__ for B in C.__mro__)):
            return True
        return NotImplemented
```

> `__subclasshook__(subclass)`必须定义为类方法。该方法检查 subclass 是否是该抽象基类的子类。该方法必须返回 `True`, `False` 或是 `NotImplemented`。如果返回 `True`，subclass 就会被认为是这个抽象基类的子类。如果返回 `False`，无论正常情况是否应该认为是其子类，统一视为不是。如果返回 `NotImplemented`，子类检查会按照正常机制继续执行。


