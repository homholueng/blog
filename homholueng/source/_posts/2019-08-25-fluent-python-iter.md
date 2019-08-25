---
title: Chapter 14 - 可迭代的对象、迭代器和生成器
tags:
  - Python
categories:
  - Fluent Python
date: 2019-08-25 16:21:37
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

### 过滤

第一组是用于过滤的生成器函数：从输入的可迭代对象中产出元素的子集，而且不修改元素本身。

| 模块 | 函数 | 说明 |
| ----| ---- |---- |
|itertools|compress(it, selector_it)|并行处理两个可迭代对象；如果 selector_it 中的元素是真值，产出 it 中对应的元素|
|itertools|dropwhile(predicate, it)|处理 it，跳过 predicate 的计算结果为真值的元素，然后产出剩下的各个元素（不再进一步检查）|
|（内置）|filter|把 it 中的各个元素传给 predicate，如果 返回真值，那么产出对应的元素，如果 predicate 是 None，那么只产出真值元素|
|itertools|filterfalse(predicate, it)|与 filter 函数的作用类似，不过 predicate 的逻辑是相反的：predicate 返回假值时产出对应的元素|
|itertools|islice(it, stop) 或 islice(it, start, stop, step=1)|产出 it 的切片，作用类似于 `s[:stop]` 或 `s[start:stop:step]`，不过 it 可以是任何可迭代的对象，而且这个函数实现的是惰性操作|
|itertools|takewhile(predicate, it)|predicate 返回真值时产出对应的元素，然后立即停止，不再继续检查|

```python
In [1]: import itertools

In [2]: def vowel(c):
   ...:     return c.lower() in 'aeiou'
   ...:

In [3]: list(filter(vowel, 'Aardvark'))
Out[3]: ['A', 'a', 'a']

In [4]: list(itertools.filterfalse(vowel, 'Aardvark'))
Out[4]: ['r', 'd', 'v', 'r', 'k']

In [5]: list(itertools.dropwhile(vowel, 'Aardvark'))
Out[5]: ['r', 'd', 'v', 'a', 'r', 'k']

In [6]: list(itertools.takewhile(vowel, 'Aardvark'))
Out[6]: ['A', 'a']

In [7]: list(itertools.compress('Aardvark', [1, 0, 1, 1, 0, 1]))
Out[7]: ['A', 'r', 'd', 'a']

In [8]: list(itertools.islice('Aardvark', 4))
Out[8]: ['A', 'a', 'r', 'd']

In [9]: list(itertools.islice('Aardvark', 4, 7))
Out[9]: ['v', 'a', 'r']

In [10]: list(itertools.islice('Aardvark', 1, 7, 2))
Out[10]: ['a', 'd', 'a']
```

### 映射

下一组是用于映射的生成器函数：在输入的单个可迭代对象（map 和 starmap 函数处理多个可迭代的对象）中的各个元素上做计算，然后返回结果。

| 模块 | 函数 | 说明 |
| ----| ---- |---- |
|itertools|accumulate(it, [func])|产出累积的总和；如果提供了 func，那么把前两个元素传给它，然后把计算结果和下一个元素传给它，以此类推，最后产出结果|
|（内置）|enumerate(iterable, start=0)|产出由两个元素组成的元组，结构是 (index, item) ，其中 index 从 start 开始计数，item 则从 iterable 中获取|
|内置|map(func, it1, [it2, ..., itN])|把 it 中的各个元素传给func，产出结果；如果传入 N 个可迭代的对象，那么 func 必须能接受 N 个参数，而且要并行处理各个可迭代的对象|
|itertools|starmap(func, it)|把 it 中的各个元素传给 func，产出结果；输入的可迭代对象应该产出可迭代的元素 iit，然后以 `func(*iit)` 这种形式调用 func|

```python
In [11]: sample = [5, 4, 2, 8, 7, 6, 3, 0, 9, 1]

In [12]: import itertools

In [13]: list(itertools.accumulate(sample))
Out[13]: [5, 9, 11, 19, 26, 32, 35, 35, 44, 45]

In [14]: list(itertools.accumulate(sample, min))
Out[14]: [5, 4, 2, 2, 2, 2, 2, 0, 0, 0]

In [15]: list(itertools.accumulate(sample, max))
Out[15]: [5, 5, 5, 8, 8, 8, 8, 8, 9, 9]

In [16]: import operator

In [17]: list(itertools.accumulate(sample, operator.mul))
Out[17]: [5, 20, 40, 320, 2240, 13440, 40320, 0, 0, 0]

In [18]: list(itertools.starmap(operator.mul, enumerate('albatroz', 1)))
Out[18]: ['a', 'll', 'bbb', 'aaaa', 'ttttt', 'rrrrrr', 'ooooooo', 'zzzzzzzz']
```

### 合并

接下来这一组是用于合并的生成器函数，这些函数都从输入的多个可迭代对象中产出元素。

| 模块 | 函数 | 说明 |
| ----| ---- |---- |
|itertools|chain(it1, ..., itN)|先产出 it1 中的所有元素，然后产出 it2 中的所有元素，以此类推，无缝连接在一起|
|itertools|chain.from_iterable(it)|产出 it 生成的各个可迭代对象中的元素，一个接一个，无缝连接在一起；it 应该产出可迭代的元素，例如可迭代的对象列表|
|itertools|product(it1, ..., itN, repeat=1)|计算笛卡儿积：从输入的各个可迭代对象中获取元素，合并成由 N 个元素组成的元组，与嵌套的 for 循环效果一样；repeat 指明重复处理多少次输入的可迭代对象|
|（内置）|zip(it1, ..., itN)|并行从输入的各个可迭代对象中获取元素，产出由 N 个元素组成的元组，只要有一个可迭代的对象到头了，就默默地停止|
|itertools|zip_longest(it1, ..., itN, fillvalue=None)|并行从输入的各个可迭代对象中获取元素，产出由 N 个元素组成的元组，等到最长的可迭代对象到头后才停止，空缺的值使用 fillvalue 填充|

```python
In [1]: import itertools

In [2]: list(itertools.chain('ABC', range(2)))
Out[2]: ['A', 'B', 'C', 0, 1]

In [3]: list(itertools.chain(enumerate('ABC')))
Out[3]: [(0, 'A'), (1, 'B'), (2, 'C')]

In [4]: list(itertools.chain.from_iterable(enumerate('ABC')))
Out[4]: [0, 'A', 1, 'B', 2, 'C']

In [5]: list(zip('ABC', range(5)))
Out[5]: [('A', 0), ('B', 1), ('C', 2)]

In [6]: list(zip('ABC', range(5), [10, 20, 30, 40]))
Out[6]: [('A', 0, 10), ('B', 1, 20), ('C', 2, 30)]

In [8]: list(itertools.zip_longest('ABC', range(5)))
Out[8]: [('A', 0), ('B', 1), ('C', 2), (None, 3), (None, 4)]

In [9]: list(itertools.zip_longest('ABC', range(5), fillvalue='?'))
Out[9]: [('A', 0), ('B', 1), ('C', 2), ('?', 3), ('?', 4)]

In [10]: list(itertools.product('ABC', range(2)))
Out[10]: [('A', 0), ('A', 1), ('B', 0), ('B', 1), ('C', 0), ('C', 1)]

In [11]: list(itertools.product('ABC'))
Out[11]: [('A',), ('B',), ('C',)]

In [12]: list(itertools.product('ABC', repeat=2))
Out[12]:
[('A', 'A'),
 ('A', 'B'),
 ('A', 'C'),
 ('B', 'A'),
 ('B', 'B'),
 ('B', 'C'),
 ('C', 'A'),
 ('C', 'B'),
 ('C', 'C')]

In [13]: list(itertools.product('ABC', range(2), repeat=2))
Out[13]:
[('A', 0, 'A', 0),
 ('A', 0, 'A', 1),
 ('A', 0, 'B', 0),
 ('A', 0, 'B', 1),
 ('A', 0, 'C', 0),
 ('A', 0, 'C', 1),
 ('A', 1, 'A', 0),
 ('A', 1, 'A', 1),
 ('A', 1, 'B', 0),
 ('A', 1, 'B', 1),
 ('A', 1, 'C', 0),
 ('A', 1, 'C', 1),
 ('B', 0, 'A', 0),
 ('B', 0, 'A', 1),
 ('B', 0, 'B', 0),
 ('B', 0, 'B', 1),
 ('B', 0, 'C', 0),
 ('B', 0, 'C', 1),
 ('B', 1, 'A', 0),
 ('B', 1, 'A', 1),
 ('B', 1, 'B', 0),
 ('B', 1, 'B', 1),
 ('B', 1, 'C', 0),
 ('B', 1, 'C', 1),
 ('C', 0, 'A', 0),
 ('C', 0, 'A', 1),
 ('C', 0, 'B', 0),
 ('C', 0, 'B', 1),
 ('C', 0, 'C', 0),
 ('C', 0, 'C', 1),
 ('C', 1, 'A', 0),
 ('C', 1, 'A', 1),
 ('C', 1, 'B', 0),
 ('C', 1, 'B', 1),
 ('C', 1, 'C', 0),
 ('C', 1, 'C', 1)]
```

### 扩展

有些生成器函数会从一个元素中产出多个值，扩展输入的可迭代对象。

| 模块 | 函数 | 说明 |
| ----| ---- |---- |
|itertools|combinations(it, out_len)|把 it 产出的 out_len 个元素组合在一起，然后产出|
|itertools|combinations_with_replacement(it, out_len)|把 it 产出的 out_len 个元素组合在一起，然后产出，包含相同元素的组合|
|itertools|count(start=0, step=1)|从 start 开始不断产出数字，按 step 指定的步幅增加|
|itertools|cycle(it)|从 it 中产出各个元素，存储各个元素的副本，然后按顺序重复不断地产出各个元素|
|itertools|permutations(it, out_len=None)|把 out_len 个 it 产出的元素排列在一起，然后产出这些排列；out_len 的默认值等于 `len(list(it))`|
|itertools|repeat(item, [times])|重复不断地产出指定的元素，除非提供 times，指定次数|

```python
In [1]: import itertools

In [2]: list(itertools.combinations('ABC', 2))
Out[2]: [('A', 'B'), ('A', 'C'), ('B', 'C')]

In [3]: list(itertools.combinations_with_replacement('ABC', 2))
Out[3]: [('A', 'A'), ('A', 'B'), ('A', 'C'), ('B', 'B'), ('B', 'C'), ('C', 'C')]

In [4]: list(itertools.permutations('ABC', 2))
Out[4]: [('A', 'B'), ('A', 'C'), ('B', 'A'), ('B', 'C'), ('C', 'A'), ('C', 'B')]
```

### 重排列

| 模块 | 函数 | 说明 |
| ----| ---- |---- |
|itertools|groupby(it, key=None)|产出由两个元素组成的元素，形式为 (key, group) ，其中 key 是分组标准，group 是生成器，用于产出分组里的元素|
|（内置）|reversed(seq)|从后向前，倒序产出 seq 中的元素；**seq 必须是序 列，或者是实现了 `__reversed__` 特殊方法的对象**|
|itertools|tee(it, n=2)|产出一个由 n 个生成器组成的元组，每个生成器 用于单独产出输入的可迭代对象中的元素|

```python
In [1]: import itertools

In [2]: list(itertools.combinations('ABC', 2))
Out[2]: [('A', 'B'), ('A', 'C'), ('B', 'C')]

In [3]: list(itertools.combinations_with_replacement('ABC', 2))
Out[3]: [('A', 'A'), ('A', 'B'), ('A', 'C'), ('B', 'B'), ('B', 'C'), ('C', 'C')]

In [4]: list(itertools.permutations('ABC', 2))
Out[4]: [('A', 'B'), ('A', 'C'), ('B', 'A'), ('B', 'C'), ('C', 'A'), ('C', 'B')]

In [5]: import itertools

In [6]: list(itertools.groupby('LLLLAAGGG'))
Out[6]:
[('L', <itertools._grouper at 0x10eaaf7b8>),
 ('A', <itertools._grouper at 0x10eaaf7f0>),
 ('G', <itertools._grouper at 0x10eaaf8d0>)]

In [7]: for char, group in itertools.groupby('LLLLAAGGG'):
   ...:     print(char, '->', list(group))
   ...:
L -> ['L', 'L', 'L', 'L']
A -> ['A', 'A']
G -> ['G', 'G', 'G']

In [8]: animals = ['duck', 'eagle', 'rat', 'giraffe', 'bear', 'bat', 'dolphin', 'shark', 'lion']

In [9]: animals.sort(key=len)

In [10]: animals
Out[10]: ['rat', 'bat', 'duck', 'bear', 'lion', 'eagle', 'shark', 'giraffe', 'dolphin']

In [11]: for length, group in itertools.groupby(animals, len):
    ...:     print(length, '->', list(group))
    ...:
3 -> ['rat', 'bat']
4 -> ['duck', 'bear', 'lion']
5 -> ['eagle', 'shark']
7 -> ['giraffe', 'dolphin']

In [12]: for length, group in itertools.groupby(reversed(animals), len):
    ...:     print(length, '->', list(group))
    ...:
7 -> ['dolphin', 'giraffe']
5 -> ['shark', 'eagle']
4 -> ['lion', 'bear', 'duck']
3 -> ['bat', 'rat']

In [15]: list(zip(*itertools.tee('ABC')))
Out[15]: [('A', 'A'), ('B', 'B'), ('C', 'C')]

In [16]: list(zip(*itertools.tee('ABC', 3)))
Out[16]: [('A', 'A', 'A'), ('B', 'B', 'B'), ('C', 'C', 'C')]
```

## 14.10　Python 3.3中新出现的句法：yield from

如果生成器函数需要产出另一个生成器生成的值，传统的解决方法是使用嵌套的 for 循环，例如下面自己实现的 chain 的例子：

```python
def chain(*iterables):
  for it in iterables:
    for i in it:
      yield i
```

在 PEP380 中引入了一个新的句法：`yield from`，它能够替代我们内层的 for 循环：

```python
def chain(*iterables):
  for it in iterables:
    yield from it
```

`yield from` 不仅仅是语法糖，其还会创建通道，把内层生成器直接与外层生成器的客户端联系起来。把生成器当成协程使用时，这个通道特别重要，不仅能为客户端代码生成值，还能使用客户端代码提供的值。

## 14.12　深入分析 iter 函数

如前所述，在 Python 中迭代对象 x 时会调用 `iter(x)`。

可是，iter 函数还有一个鲜为人知的用法：传入两个参数，使用常规的函数或任何可调用的对象创建迭代器。这样使用时，第一个参数必须是可调用的对象，用于不断调用（没有参数），产出各个值；第二个值是哨符，这是个标记值，当可调用的对象返回这个值时，触发迭代器抛出 StopIteration 异常，而不产出哨符。

```python
In [21]: def d6():
    ...:     return randint(1, 6)
    ...:

In [22]:

In [22]: def d6():
    ...:     return randint(1, 6)
    ...:

In [23]: d6_iter = iter(d6, 1)

In [24]: d6_iter
Out[24]: <callable_iterator at 0x10ee55b70>

In [25]: for roll in iter(d6, 1):
    ...:     print(roll)
    ...:
4
2
5
5
6
4
5
```

内置函数 iter 的文档中有个实用的例子。这段代码逐行读取文件，直到遇到空行或者到达文件末尾为止：

```python
with open('mydata.txt') as fp:
  for line in iter(fp.readline, '\n'):
    process_line(line)
```