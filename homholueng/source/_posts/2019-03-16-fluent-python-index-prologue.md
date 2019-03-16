---
title: Chapter 1 - 序章
date: 2019-03-16 21:08:37
categories: 
- Fluent Python
tags:
- Python
---

## 一摞 Python 风格的纸牌

### 通过 `__getitem__` 来自定义下标访问

某些时候，我们需要重载自定义类的下标访问操作来实现 Python 的一致性，这个时候就需要通过实现 `__getitem__` 方法来完成：

```python
class FrenchDeck:
    ranks = [str(n) for n in range(2, 11)] + list('JQKA')
    suits = 'spades diamonds clubs hearts'.split()

    def __init__(self):
        self._cards = [Card(rank, suit) for suit in self.suits
                                        for rank in self.ranks]
        
    def __len__(self):
        return len(self._cards)

    def __getitem__(self, position):
        return self._cards[position]
```

### 使用 random 来随机选择序列中的值

使用 `random` 模块中的 `choice` 函数能够随机返回一个非空序列中的某个值：

```python
import random

seq = [1, 2, 3, 4, 5]

random.choice(seq)

```

### 通过代理来实现 Python 语言的一致性

何为一致性，即对不同类型的对象都能使用同一个方法或函数来实现特定的功能。如，无论是对于 `list`, `str` 还是 `dict` 对象，我们都能够使用 `len()` 函数来获取他们的长度。

若在自定义的类中，我们可以通过将这些一致性的方法或函数代理给类中的内部对象来实现 Python 语言的一致性，如实现 `__len__` 方法将 `len()` 的操作代理给类中的列表对象。

## 如何使用特殊方法

### 自定义布尔值

默认情况下，我们自己定义的类的实例总被认为是真的，除非这个类对 `__bool__` 或者 `__len__` 函数有自己的实现。`bool(x)` 的背后是调用 `x.__bool__()` 的结果；如果不存在 `__bool__` 方法，那么 `bool(x)` 会尝试调用 `x.__len__()`。若返回 0，则 `bool` 会返回 `False`；否则返回 `True`。


