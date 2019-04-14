---
title: Chapter 9 - 符合 Python 风格的对象
date: 2019-04-14 15:51:43
tags:
  - Python
categories:
  - Fluent Python
---


得益于 Python 数据模型，自定义类型的行为可以像内置类型那样自然。实现如此自然的行为，靠的不是集成，而是鸭子类型（duck typing）。

## 对象表示形式

Python 提供了两种获取对象表示形式的方法:

- `repr()`：以便于开发者理解的方式返回对象的字符串表示形式。
- `str()`：以便于用户理解的方式返回对象的字符串表示形式。

为了支持以上两种方式，我们要实现 `__repr__` 和 `__str__` 两个特殊方法。还有另外两个特殊方法能够提供对象的其他表示形式：`__bytes__` 和 `__format__`。前者在 `bytes()` 函数中使用，而后者会在内置的 `format()` 函数和 `str.format()` 方法中使用。

**在 Python2 中，`__repr__` 和 `__str__` 不应该返回 `Unicode` 对象，而在 Python3 中，它们必须返回 Unicode 对象！**

下面是一个设计良好的对象的示例：

```python
import math
from array import array

class Vector2d:
    typecode = 'd'

    def __init__(self, x, y): 
        self.x = float(x) 
        self.y = float(y)

    def __iter__(self): 
        return (i for i in (self.x, self.y))

    def __repr__(self): 
        class_name = type(self).__name__ 
        return '{}({!r}, {!r})'.format(class_name, *self)
    
    def __str__(self): 
        return str(tuple(self))
    
    def __bytes__(self): 
        return (bytes([ord(self.typecode)]) + 
                bytes(array(self.typecode, self)))

    def __eq__(self, other): 
        return tuple(self) == tuple(other)

    def __abs__(self): 
        return math.hypot(self.x, self.y)

    def __bool__(self): 
        return bool(abs(self))
```

## 备选构造方法

下面我们为上一小节定义的 `Vector2d` 类定义一个备选的构造方法，这个构造方法读取一个字节序列并返回一个 `Vector2d` 对象：

```python
@classmethod
def frombytes(cls, octets):
    typecode = chr(octets[0])
    memv = memoryview(octets[1:]).cast(typecode) 
    return cls(*memv)
```

## classmethod 与 staticmethod

classmethod 改变了调用方法的方式，因此类方法的第一个参数是类本身，而不是实例。classmethod 最常见的用途是定义备选构造方法。

staticmethod 装饰器也会改变方法的调用方式，但是第一个参数不是特殊的值。

## 格式化显示

内置的 `format()` 函数和 `str.format()` 方法把各个类型的格式化方式委托给相应的 `.__format__(format_spec)` 方法。`format_spec` 是格式说明符，它是：

- `format(my_obj, format_spec)` 的第二个参数，或
- `str.format()` 方法的格式字符串，`{}` 里代换字段中冒号后面的部分

举个例子：

```python
In [1]: brl = 1/2.34

In [2]: brl
Out[2]: 0.4273504273504274

In [3]: format(brl, '0.4f') # format_spec 是 '0.4f'
Out[3]: '0.4274'

In [4]: '1 BRL = {rate:0.2f} USD'.format(rate=brl) # format_spec 是 '0.2f'
Out[4]: '1 BRL = 0.43 USD'
```

格式说明符使用的表示法叫格式规范微语言([formatspec](https://docs.python.org/3/library/string.html#formatspec))。格式规范微语言是可以扩展的，各个类可以自行决定如何解释 `format_spec` 参数。例如 datetime 模块中的类，他们的 `__format__` 方法使用的格式代码就与 `strftime()` 函数一样：

```python
In [5]: from datetime import datetime

In [6]: now = datetime.now()

In [7]: format(now, '%H:%M:%S')
Out[7]: '15:00:46'
```

## 可散列的 Vector2d

要创建可散列的类型，只需要正确的实现 `__hash__` 方法和 `__eq__` 方法即可。实现 `__hash__` 方法时，官方推荐使用位运算符异或 `^` 混合各分量的散列值：

```python
def __hash__(self):
    return hash(self.x) ^ hash(self.y)
```

## Python 的私有属性和“受保护的”属性

Python 不像 Java 那样使用 private 修饰符创建私有属性，但是 Python 有个简单的机制，能避免子类意外覆盖“私有”属性。

计入我们在类 `Foo` 中以 `__attr` 的形式命名实例属性，Python 会把属性名存入实例的 `__dict__` 属性中，而且会在前面加上一个下划线和类型。因此，`__attr` 会变成 `_Foo__attr`；这个语言特性叫名称改写（name mangling）。

```python
In [8]: class Foo:
   ...:     def __init__(self):
   ...:         self.__attr = 'attr'
   ...:

In [9]: foo = Foo()

In [10]: foo.__dict__
Out[10]: {'_Foo__attr': 'attr'}
```

不是所有 Python 程序员都喜欢名称改写功能，也不是所有人都喜欢 `self.__x` 这种不对称的名称。有些人不喜欢这种句法，他们约定使用一个下划线前缀编写“受保护”的属性（如 `self._x`）。Python 解释器虽然不会对使用单个下划线的属性名做特殊处理，不过这是很多 Python 程序员严格遵守的约定，他们不会在类外部访问这种属性。

## 使用 __slots__ 类属性节省空间

默认情况下，Python 在各个实例中名为 `__dict__` 的字典里存储实例属性。而字典会消耗大量的内存，如果要处理数百万个属性不多的实例，通过 `__slots__` 类属性能够节省大量内存。

> 继承自父类的 `__slots__` 属性没有效果。Python 只会使用各个类中定义的 `__slots__` 属性。

定义 `__slots__` 的方式是，创建一个类属性，使用 `__slots__` 这个名字，并把它的值设为一个字符串构造的可迭代对象，其中各个元素表示各个实例属性：

```python
class Vector2d:
    __slots__ = ('x', 'y')
```

> 在类中定义 `__slots__` 属性之后，实例不能再有 `__slots__` 中所列名称之外的其他属性。这只是一个副作用，不是 `__slots__` 存在的真正原因。不要使用 `__slots__` 属性禁止类的用户新增实例属性。

此外，还需要注意的一个实例属性就是 `__weakref__` 属性，为了让对象支持弱引用，必须有这个属性。用户自定义的类中默认就有该属性，可是，如果类中定义了 `__slots__` 属性，而且想把实例作为弱引用的目标，那么就要把 `__weakref__` 添加到 `__slots__` 中。

**如果你的程序不用处理数百万个实例，或许不值得费劲去创建不寻常的类，仅当权衡当下的需求并仔细搜集资料后证明确实有必要时，才应该使用 `__slots__` 属性。**

## 覆盖类属性

如果父类中的方法希望获取用户在自定义子类中覆盖的类属性，那么就不应该硬编码获取属性时的类，而是通过如下方式获取：

```python
def method(self):
    Class.attr # Wrong
    type(self).attr
```

## 总结

**符合 Python 风格的对象应该正好符合所需，而不是堆砌语言特性。**