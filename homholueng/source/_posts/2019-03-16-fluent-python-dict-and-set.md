---
title: Chapter 3 - 字典和集合
date: 2019-03-16 21:50:45
categories: 
- Fluent Python
tags:
- Python
---

## 映射的弹性键查询

### `defaultdict`：处理找不到的键的一个选择

`defaultdict` 会在在初始化时接收一个构造方法，并在 `__getietm__` 碰到找不到的键的时候调用该方法让 `__getitem__` 返回某种默认值。

例：

```python
from collections import defaultdict

dd = defaultdict(list)

dd['key_not_exist'].append(1)

dd['another_key_not_exist'].append(1)
```

**注意，`defaultdict` 里的 `default_factory` 只会在 `__getitem__` 里被调用，在其他的方法里完全不会发挥作用。比如，`dd` 是个 `defaultdict`，`k` 是个找不到的键，`dd.get(k)` 会返回 `None`。**

### 特殊方法 `__missing__`

所有的映射类型在在 `__getitem__` 碰到找不到的键的时候，Python 都会调用其 `__missing__` 方法，而不是抛出 `KeyError`。

**注意，`__missing__` 方法只会被 `__getitem__` 调用（比如在表达式 `d[k]` 中）。提供 `__missing__` 方法对 `get` 或者 `__contains__`（`in` 运算符会用到这个方法）这些方法的使用没有 影响。**

像 `k in my_dict.keys()` 这种操作在 Python 3 中是很快的，而且即便映射类型对象很庞大也没关系。这是因为 `dict.keys()` 的返回值是一个“视图”。视图就像一个集合，而且跟字典类似的是，在视图里查找一个元素的速度很快（https://docs.python.org/3/library/stdtypes.html#dictionary-view-objects）。

但 Python 2 的 `dict.keys()` 返回的是个列表，所以在处理庞大的映射类型对象时会比较慢。

## 字典的变种

### `collections.OrderedDict`

这个类型在添加键的时候会保持顺序，因此键的迭代次序总是一致的。

### `collections.ChainMap`

该类型可以容纳数个不同的映射对象，然后在进行键查找操作的时候，这些对象会被当作一个整体被逐个查找，直到键被找到为止。这个功能在给有嵌套作用域的语言做解释器的时候很有用，可以用一个映射对象来代表一个作用域的上下文。

例如：

```python
import builtins 
pylookup = ChainMap(locals(), globals(), vars(builtins))
```

### `collections.Counter`

这个映射类型会给键准备一个整数计数器。每次更新一个键的时候都会增加这个计数器。所以这个类型可以用来给可散列表对象计数，或者是当成多重集来用——多重集合就是集合里的元素可以出现不止一 次。`Counter` 实现了 `+` 和 `-` 运算符用来合并记录，还有像 `most_common([n])` 这类很有用的方法。

```python
>>> ct = collections.Counter('abracadabra') 
>>> ct 
Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1}) 
>>> ct.update('aaaaazzz') 
>>> ct 
Counter({'a': 10, 'z': 3, 'b': 2, 'r': 2, 'c': 1, 'd': 1}) 
>>> ct.most_common(2) 
[('a', 10), ('z', 3)]
```

### `colllections.UserDict`

这个类其实就是把标准 `dict` 用纯 Python 又实现了一遍，但它是专门让用户继承来写子类的。

## 子类化 `UserDict`

而更倾向于从 `UserDict` 而不是从 `dict` 继承的主要原因是，后者有时 会在某些方法的实现上走一些捷径，导致我们不得不在它的子类中重写 这些方法，但是 `UserDict` 就不会带来这些问题。

另外一个值得注意的地方是，`UserDict` 并不是 `dict` 的子类，但是 `UserDict` 有一个叫作 `data` 的属性，是 `dict` 的实例，这个属性实际上 是 `UserDict` 最终存储数据的地方。

## 不可变映射类型

从 Python 3.3 开始，types 模块中引入了一个封装类名叫 `MappingProxyType`。如果给这个类一个映射，它会返回一个只读的映 射视图。虽然是个只读视图，但是它是动态的。

## 集合论

集合实现了许多基础的中缀运算符，给定两个集合 a 和 b：

- `a | b` 返回的是它们的合集
- `a & b` 返回的是它们的交集
- `a - b` 返回的是它们的差集
- `a in b` 返回 a 是否属于 b
- `a <= b` 返回 a 是否是 b 的子集
- `a < b` 返回 a 是否是 b 的真子集

当我们要从集合 s 中移除一个元素时，如果我们关注该元素的存在性，则可以使用 `s.remove(e)` 方法，该方法在 e 不存在时会抛出 `KeyError` 异常；如果我们不关注该元素的存在性，则使用 `s.discard(e)` 即可。