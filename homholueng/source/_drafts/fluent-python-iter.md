---
title: Chapter 14 - 可迭代的对象、迭代器和生成器
tags:
  - Python
categories:
  - Fluent Python
---

## 14.1　Sentence 类第1版：单词序列

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


## 14.3 Sentence 类第2版：典型的迭代器

在这一个版本中，我们根据《设计模式：可复用面向对象软件的基础》一书中给出的模型来实现典型的迭代器设计模式，但这并不符合 Python 的习惯做法。

```python

import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)
    
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        return SentenceIterator(self.words)

class SentenceIterator:

    def __init__(self, words):
        self.words = words
        self.index = 0
    
    def __next__(self):
        try:
            word = self.words[self.index]
        except IndexError:
            raise StopIteration()
        self.index += 1
        return word
    
    def __iter__(self):
        return self
```

这一版的工作量很大（对于懒惰的 Python 程序员来说的确如此）。

### 把 Sentence 变成迭代器：BAD IDEA ！

构建可迭代的对象和迭代器时经常会出现错误，原因是混淆了二者。要知道，可迭代对象有个 `__iter__` 方法，每次都实例化一个新的迭代器；而迭代器要实现 `__next__` 方法，返回单个元素，此外还要实现 `__iter__` 方法，返回迭代器本身。

**切忌将 Sentence 类实现为迭代器！因为可迭代对象必须能够返回多个独立的迭代器，而每个迭代器要能维护自身内部的状态！**

## 14.4 Sentence 类第3版：生成器函数

实现相同的功能，Python 习惯的方式是用生成器函数代替 SentenceIterator 类：

```python

import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
        self.words = RE_WORD.findall(text)
    
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        for word in self.words:
            yield word
        return
```

只要 Python 函数的定义体中有 `yield` 关键字，该函数就是生成器函数。调用生成器函数时，会返回一个生成器对象。也就是说，生成器函数是生成器工厂。

下面一个特别简单的函数说明生成器的行为：

```python
In [3]: def gen123():
   ...:     yield 1
   ...:     yield 2
   ...:     yield 3
   ...:

In [4]: gen123
Out[4]: <function __main__.gen123()>

In [8]: gen123()
Out[8]: <generator object gen123 at 0x10287ff10>

In [10]: for i in gen123():
    ...:     print(i)
    ...:
1
2
3

In [11]: g = gen123()

In [12]: next(g)
Out[12]: 1

In [13]: next(g)
Out[13]: 2

In [14]: next(g)
Out[14]: 3

In [15]: next(g)
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-15-e734f8aca5ac> in <module>
----> 1 next(g)

StopIteration:
```

生成器函数会创建一个生成器对象，包装生成器函数的定义体。把生成器传给 next 函数时，生成器函数会向前，执行函数定义体中的下一个 `yield` 语句，返回产出的值，并在函数定义体的当前位置暂停。

这一版 Sentence 类比前一版简短多了，但是还不够懒惰。如今，人们认为惰性是好的特质，至少在编程语言和 API 中是如此。惰性实现是指尽可能延后生成值。这样做能节省内存，而且或许还可以避免做无用的处理。

## 14.5 Sentence 类第4版：惰性实现

目前实现的几版 Sentence 类都不具有惰性，因为 `__init__` 方法在一开始就构建好了文本中的单词列表，然后将其绑定到 `self.words` 属性上。

为了解决这个问题，我们可以使用 `re.finditer` 函数，`re.finditer` 函数是 `re.findall` 函数的惰性版本，返回的不是列表，而是一个生成器，按需生成 `re.MatchObject` 实例。

```python

import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
    
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        for match in RE_WORD.finditer(self.text):
          yield match.group()
```

## 14.6 Sentence 类第5版：生成器表达式

生成器表达式可以理解为列表推导的惰性版本：不会迫切地构建列表，而是返回一个生成器，按需惰性生成元素。也就是说，如果列表推导是制造列表的工厂，那么生成器表达式就是制造生成器的工厂。

下面是列表推导和生成器表达式的对比：

```python
In [1]: def gen_AB():
   ...:     print('start')
   ...:     yield 'A'
   ...:     print('continue')
   ...:     yield 'B'
   ...:     print('end.')
   ...:

In [2]: res1 = [x for x in gen_AB()]
start
continue
end.

In [4]: res1
Out[4]: ['A', 'B']

In [5]: res2 = (x for x in gen_AB())

In [6]: res2
Out[6]: <generator object <genexpr> at 0x1027dbca8>

In [8]: for i in res2:
   ...:     print(i)
   ...:
start
A
continue
B
end.
```

使用生成器表达式，我们能够进一步减少 Sentence 实现的代码：

```python

import re
import reprlib

RE_WORD = re.compile('\w+')

class Sentence:

    def __init__(self, text):
        self.text = text
    
    def __repr__(self):
        return 'Sentence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        return (match.group() for match in RE_WORD.finditer(self.text))
```

生成器表达式是语法糖：完全可以替换成生成器函数，不过有时使用生成器表达式更便利。

## 14.7 何时使用生成器表达式

生成器表达式是创建生成器的简洁句法，这样无需先定义函数再调用。不过，生成器函数灵活得多，可以使用多个语句实现复杂的逻辑，也可以作为协程使用。

选择使用哪种句法很容易判断：如果生成器表达式要分成多行写，推荐定义生成器函数，以便提高可读性。此外，生成器函数有名称，因此可以重用。

## 14.9 标准库中的生成器函数

