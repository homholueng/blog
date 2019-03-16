---
title: Chapter 2 - 数据结构
date: 2019-03-16 21:50:03
categories: 
- Fluent Python
tags:
- Python
---

## 列表推导和生成器表达式

### 用列表推到生成笛卡尔积

用列表推导可以生成两个或以上的可迭代类型的笛卡儿积。

```python
colors = ['blck', 'white']
sizes = ['S', 'M', 'L']

tshirts = [(color, size) for color in colors for size in sizes]
# [('black', 'S'), ('black', 'M'), ('black', 'L'), ('white', 'S'), ('white', 'M'), ('white', 'L')]

tshirts = [(color, size) for size in sizes 
                         for color in colors]
# [('black', 'S'), ('white', 'S'), ('black', 'M'), ('white', 'M'), ('black', 'L'), ('white', 'L')]
```

## 元组不仅仅是不可变的列表

### 具名元组的一些方法 

- `_fields()`: 类方法，返回某个具名元组中的字段。
- `_make(iterable)`: 类方法，接收一个可迭代对象来生成类的实例。
- `_asdict()`: 实例方法，返回以当前实例字段名为键，字段值为值的顺序字典。

演示如下：

```
In [17]: City = collections.namedtuple('City', 'name country population coordinated')

In [18]: LatLong = collections.namedtuple('LatLong', 'lat long')

In [19]: delhi_data = ('Delhi NCR', 'IN', 21.935, LatLong(28.613889, 77.208889))

In [20]: delhi = City._make(delhi_data)

In [21]: City._fields
Out[21]: ('name', 'country', 'population', 'coordinated')

In [22]: delhi._asdict()
Out[22]:
OrderedDict([('name', 'Delhi NCR'),
             ('country', 'IN'),
             ('population', 21.935),
             ('coordinated', LatLong(lat=28.613889, long=77.208889))])
```

## 切片

### slice 对象

了解切片对象能够让我们更好的了解 Python 切片操作实现的原理，并在需要进行大量且复杂的切片时提升代码的可读性。

当我们要对一个序列进行切片时，背后会调用该序列的 `__getitem__` 方法，并且传入切片对象，例如 `a[1:10:2]` 等价于 `a.__getitem__(slice(1, 10, 2)`

利用这一特性，能够让切片操作可读性更强

```python
DESCRIPTION = slice(6, 40)
QUANTITY = slice(52, 55)

for item in items:
    print(item[DESCRIPTION])
```

## 对序列使用+和*

### * 时采用的是浅拷贝

需要注意的是，对序列使用 * 时得到的新序列中的元素是旧序列中元素的浅拷贝。

```python
In [34]: a = []

In [35]: b= [a]

In [36]: b
Out[36]: [[]]

In [37]: c = b * 4

In [38]: c
Out[38]: [[], [], [], []]

In [39]: a.append(1)

In [40]: a
Out[40]: [1]

In [41]: b
Out[41]: [[1]]

In [42]: c
Out[42]: [[1], [1], [1], [1]]

```

## 序列的增量赋值

### 不可变序列的增量赋值

对不可变序列进行增量赋值时，返回的是一个新的不可变序列。

```python
In [58]: t = (1, 2, 3)

In [59]: id(t)
Out[59]: 4488186256

In [60]: t *= 2

In [61]: t
Out[61]: (1, 2, 3, 1, 2, 3)

In [62]: id(t)
Out[62]: 4470965640
```

虽然 `str` 也是不可变序列，但因为对 `str` 进行 `+=` 操作是十分普遍的，所以 CPython 对其进行了优化，在为 `str` 初始化内存时，会额外分配内存空间。

### 三个教训

1. 不要把可变对象放在元组中
2. 增量赋值不是一个原子操作。
3. 查看 Python 的字节码并不难


## list.sort方法和内置函数sorted

### list.sort 和 sorted 的区别

`list.sort` 会改变原序列的顺序，而 `sorted` 只会返回一个新的序列，不会改变原序列的顺序。

## 用bisect来管理已排序的序列

### 使用 bisect 来搜索

使用 bisect 模块中的 `bisect()` 函数能够为我们在一个有序的序列中查找某个元素在不破坏原序列有序的情况下应该存在的位置：

```python
In [48]: import bisect

In [49]: HAYSTACK = [1, 4, 5, 6, 8, 12, 15, 20, 21, 23, 23, 26, 29, 30]

In [50]: bisect.bisect(HAYSTACK, 18)
Out[50]: 7
```

### 用bisect.insort插入新元素

使用 bisect 模块中的 `insort()` 函数能够在不破坏序列有序的情况下，将一个元素插入一个有序序列中。

## 当列表不是首选时

### 什么时候应该使用数组

如果我们需要一个只包含数字的列表，那么 `array.array` 比 `list` 更高效。

与 list 相比，array 中存储的元素类型是在创建后就固定了的，且能够存储的元素类型也是固定的：

```text
    array(typecode [, initializer]) -> array
    
    Type code   C Type             Minimum size in bytes
    'b'         signed integer     1
    'B'         unsigned integer   1
    'u'         Unicode character  2 (see note)
    'h'         signed integer     2
    'H'         unsigned integer   2
    'i'         signed integer     2
    'I'         unsigned integer   2
    'l'         signed integer     4
    'L'         unsigned integer   4
    'q'         signed integer     8 (see note)
    'Q'         unsigned integer   8 (see note)
    'f'         floating point     4
    'd'         floating point     8
```

### 内存视图

`memoryview` 是一个内置类，它能让用户在不复制内容的情况下操作同一个数组的不同切片。

并且 `memoryview` 允许我们以不同的方式读写同一块内存区域：

```python
In [69]: numbers = array.array('h', [-2, -1, 0, 1, 2])

In [70]: memv = memoryview(numbers)

In [71]: memv_oct = memv.cast('B')

In [72]: memv_oct
Out[72]: <memory at 0x10b154b88>

In [73]: memv_oct.tolist()
Out[73]: [254, 255, 255, 255, 0, 0, 1, 0, 2, 0]

In [74]: memv.tolist()
Out[74]: [-2, -1, 0, 1, 2]

In [75]: memv_oct[5] = 4

In [76]: memv_oct.tolist()
Out[76]: [254, 255, 255, 255, 0, 4, 1, 0, 2, 0]

In [77]: numbers
Out[77]: array('h', [-2, -1, 1024, 1, 2])
```