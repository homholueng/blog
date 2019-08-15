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

