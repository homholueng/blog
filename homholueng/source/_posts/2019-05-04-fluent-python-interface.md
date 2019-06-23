---
title: Chapter 11 - 函数装饰器和闭包
tags:
  - Python
categories:
  - Fluent Python
date: 2019-05-04 20:23:55
---


## 11.1 Python 文化中的接口和协议

接口在动态类型语言中是怎么运作的呢？首先，基本的事实是，Python 语言没有 `interface` 关键字，而且除了抽象基类，每个类都有接口： 类实现或继承的公开属性（方法或数据属性），包括特殊方法，如 `__getitem__` 或 `__add__`。

另一方面，不要觉得把公开数据属性放入对象的接口中不妥，因为如果需要，总能实现读值方法和设值方法，把数据属性变成特性，使用 `obj.attr` 句法的客户代码不会受到影响。

关于接口，这里有个实用的补充定义：对象公开方法的子集，让对象在系统中扮演特定的角色。Python 文档中的“文件类对象”或“可迭代对象”就是这个意思，这种说法指的不是特定的类。接口是实现特定角色的方法集合，这样理解正是 Smalltalk 程序员所说的协议，其他动态语言社区都借鉴了这个术语。协议与继承没有关系。一个类可能会实现多个接口，从而让实例扮演多个角色。

协议是接口，但不是正式的（只由文档和约定定义），因此协议不能像正式接口那样施加限制。一个类可能只实现部分接口，这是允许的。

## 11.2 Python 喜欢序列

下图展示了定义为抽象基类的 `Sequence` 正式接口：

{% asset_img sequence.png %}

现在让我们看看下面的代码中定义的 Foo 类，它没有继承 `abc.Sequence`，而且只实现了序列协议的一个方法：`__getitem__`：

```python
In [1]: class Foo: 
   ...:     def __getitem__(self, pos): 
   ...:         return range(0, 30, 10)[pos] 
   ...:                                                                                 
In [2]: f = Foo()                                                                       

In [3]: f[1]                                                                            
Out[3]: 10

In [4]: for i in f: print(i)                                                            
0
10
20

In [5]: 20 in f                                                                         
Out[5]: True
```

虽然没有实现 `__iter__` 方法，但是 Foo 实例时可迭代的对象，因为发现其实现了  `__getitem__` 方法时，Python 会调用它。

所以，鉴于序列协议的重要性，如果没有 `__iter__` 和 `__contains__` 方法，Python 会调用 `__getitem__` 方法，设法让迭代和 `in` 运算符可用。

## 11.3 使用 Monkey Patch 在运行时实现协议

**猴子补丁：在运行时修改类或模块，而不改动源码。**

猴子补丁很强大，但是打补丁的代码与要打补丁的程序耦合十分紧密，而且往往要处理隐藏和没有文档的部分。

## 11.4 白鹅类型（goose typing）


白鹅类型指，只要 `cls` 是抽象基类，即 `cls` 的元类是 `abc.ABCMeta`，就可以使用 `isinstance(obj, cls)`。

`collections.abc` 中有很多有用的抽象类（Python 标准库的 `numbers` 模块中还有一些）。

## 11.5 定义抽象基类的子类

在模块导入阶段（类对象初始化）时，Python 不会检查类中的抽象方法是否被实现，只有在实例化类时才会抛出异常。

## 11.6 标准库中的抽象基类

### collections.abc 模块中的抽象基类

该模块中定义了 16 个抽象基类，如下图所示：

{% asset_img collection-abc-uml.png %}

- `Iterable`、`Container` 和 `Sized`：各个集合应该继承这三个抽象基类，或者至少实现兼容的协议。
- `Sequence`、`Mapping` 和 `Set`：这三个是主要的不可变集合类型，而且各自都有可变的子类。
- `MappingView`：在 Python 3 中，映射方法 `.items()`、`.keys()` 和 `.values()` 返回的对象分别是 `ItemsView`、`KeysView` 和 `ValuesView` 的实例。
- `Callable` 和 `Hashable`：这两个抽象基类与集合没有太大的关系，只不过因为 `collections.abc` 是标准库中定义抽象基类的第一个模块，而它们又太重要了，因此才把它们放到 collections.abc 模块中。
- `Iterator`：迭代器，注意它是 Iterable 的子类。

每个类的具体说明可参考[官方文档](https://docs.python.org/3/library/collections.abc.html#collections-abstract-base-classes)。

### numbers 模块中的抽象基类

`numbers` 包定义的是“数字塔”（即各个抽象基类的层次结构是线性的），其中 `Number` 是位于最顶端的超类，随后是 `Complex` 子类，依次往下，最底端是 `Integral` 类：

- `Number`
- `Complex`
- `Real`
- `Rational`
- `Integral`

> 与之类似，如果一个值可能是浮点数类型，可以使用 `isinstance(x, numbers.Real)` 检查。

## 11.7 定义并使用一个抽象基类

### 抽象基类的父类

自定义的抽象基类要继承 `abc.ABC`，在 Python3.4 之前，由于没有 `abc.ABC` 类，需要在 `class` 语句中使用 `metaclas=` 关键字,并将值设置为 `abc.ABCMeta`：

```python
class AbstractClass(metaclass=abc.ABCMeta):
    pass
```

而在 Python 2 中，由于没有 `metaclass=` 关键字，必须使用 `__metaclass__` 类属性：

```python
class AbstractClass(object):
    __metaclass__ = abc.ABCMeta
```

### 抛出异常

在自定义类中抛出异常是，可以考虑复用 Python 中预先定义好的异常，具体可参考[异常层次结构](https://docs.python.org/dev/library/exceptions.html#exception-hierarchy)。

### 抽象方法

与其他方法描述符一起使用时，`@abstractmethod` 应该放在最里层。另外，`@abstractclassmethod`、`@abstractstaticmethod` 和 `@abstractproperty` 三个装饰器从 Python3.3 起就废弃掉了，所以，推荐使用以下方式声明抽象类方法：

```python
class AbstractClass(abc.ABC):
    @classmethod
    @abc.abstractmethod 
    def an_abstract_classmethod(cls):
        pass

```

### 虚拟子类

注册虚拟子类的方式是在抽象基类上调用 `register` 方法。这么做之后，注册的类会变成抽象基类的虚拟子类，而且 `issubclass` 和 `isinstance` 等函数都能识别，**但是注册的类不会从抽象基类中继承任何方法或属性**。

像这样：

```python
@SuperClass.register
class VirtualSubclass(list):
    pass
```

如果是 Python 3.3 或之前的版本，不能把 `.register` 当作类装饰器使用，必须使用标准的调用句法：

```python
class VirtualSubclass(list):
    pass

SuperClass.register(TomboList)
```

虽然我们注册了虚拟子类，但是其 `__mro__` 中并不会列出该子类的虚拟超类。并且，一个类的 `__subclasses__` 方法只会返回该类的直接子类列表，而 `_abc_registry` 会返回一个包含该抽象类的所有虚拟子类的 `WeakSet` 对象。

## 鹅的行为有可能像鸭子

在 Python 中，即便不注册虚拟子类，抽象基类也能把一个类识别为虚拟子类：

```python
class Struggle:
    def __len__(self):
        return 23

In [5]: from collections import abc

In [7]: isinstance(Struggle(), abc.Sized)
Out[7]: True

In [8]: issubclass(Struggle, abc.Sized)
Out[8]: True
```

出现这种现象的原因是 `abc.Sized` 实现了一个特殊的类方法：`__subclasshook__`：

```python
class Sized(metaclass=ABCMeta):

    __slots__ = ()

    @abstractmethod
    def __len__(self):
        return 0

    @classmethod
    def __subclasshook__(cls, C):
        if cls is Sized:
            if any("__len__" in B.__dict__ for B in C.__mro__):
                return True
        return NotImplemented
```

也就是说，其会检测传入类对象本身及其所继承的类中是否能够处理 `__len__` 方法的调用，如果可以，即认为该类是自身的子类。


`__subclasshook__` 在白鹅类型中添加了一些鸭子类型的踪迹。我们可以使用抽象基类定义正式接口，可以始终使用 `isinstance` 检查，也可以完全使用不相关的类，只要实现特定的方法即可（或者做些事情让 `__subclasshook__` 信服）。

但是，不建议在我们自定义的抽象类中实现该方法，这可能会使得你的类可靠性变得很低。