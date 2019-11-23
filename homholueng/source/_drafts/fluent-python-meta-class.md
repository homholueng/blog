---
title: Chapter 21 - 类元编程
tags:
  - Python
categories:
  - Fluent Python
---

## 21.1 类工厂函数

其实我们经常使用的 `collections.namedtuple` 就是一个类工厂函数，我们把类名和几个属性名传给这个函数，它就会创建一个 tuple 的子类。

```python
def record_factory(cls_name, field_names):
    try:
        field_names = field_names.replace(',', ' ').split()
    except AttributeError:
        pass

    field_names = tuple(field_names)
    
    def __init__(self, *args, **kwargs):
        attrs = dict(zip(self.__slots__, args))
        attrs.update(kwargs)
        for name, value in attrs.items():
            setattr(self, name, value)
    
    def __iter__(self):
        for name in self.__slots__:
            yield getattr(self, name)
    
    def __repr__(self):
        values =  ', '.join('{}={!r}'.format(*i) for i in zip(self.__slots__, self))
        return '{}({})'.format(self.__class__.__Name__, values)

    cls_attrs = dict(
        __slots__=field_names,
        __init__=__init__,
        __iter__=__iter__,
        __repr__=__repr__
    )
    
    return type(cls_name, (object,), cls_attrs)
```

通常，我们把 type 视作函数，因为我们像函数那样使用它，例如，调用 `type(my_object)` 获取对象所属的类——作用与 `my_object.__class__` 相同。然而，`type` 是一个类。当成类使用时，传入三个参数可以新建一个类:

```python
MyClass = type('MyClass', (MySuperClass, MyMixin), {'x': 42 'x2': lambda self : self.x * 2})
```

`type` 的三个参数分别是 `name`、`bases` 和 `dict`。最后一个参数是一个映射，指定新类的属性名和值。上述代码的作用与下述代码相同:

```python
class MyClass(MySUperClass, MyMixin):
    x = 42

    def x2(self):
        return self.x * 2

```

## 21.2 定制描述符的类装饰器

类装饰器与函数装饰器非常类似，是参数为类对象的函数，返回原来的类或修改后的类。

```python
def entity(cls):
    for key, attr in cls.__dict__.items():
        if isintance(attr, Validated):
            type_name = type(attr).__name__
            attr.storage_name = '_{}#{}'.format(type_name, key)
    
    return cls
```

类装饰器有个重大缺点: 只对直接依附的类有效。这意味着，被装饰的类的子类可能继承也可能不继承装饰器所做的改动，具体情况视改动的方式而定。

## 21.3 导入时和运行时比较

导入模块时，解释器会执行顶层的 def 语句，可是这么做有什么作用呢?解释器会编译函数的定义体(首次导入模块时)，把函数对象绑定到对应的全局名称上，但是显然解释器不会执行函数的定义体。

对类来说，情况就不同了:在导入时，解释器会执行每个类的定义体，甚至会执行嵌套类的定义体。执行类定义体的结果是，定义了类的属性和方法，并构建了类对象。

## 21.4 元类基础知识

元类是制造类的工厂，不过不是函数，而是类。

根据 Python 的对象模型，类是对象，因此类肯定是另外某个类的实例。

```python
In [1]: 'spam'.__class__                  
Out[1]: str

In [2]: str.__class__                  
Out[2]: type

In [3]: str.__class__                 
Out[3]: type

In [4]: type.__class__
Out[4]: type
```
为了避免无限回溯，`type` 是其自身的实例，如最后一行所示。

除了 `type`，标准库中还有一些别的元类，例如 `ABCMeta`。

```python
In [1]: import collections

In [2]: collections.Iterable.__class__
Out[2]: abc.ABCMeta

In [3]: import abc

In [4]: abc.ABCMeta.__class__  
Out[4]: type

In [5]: abc.ABCMeta.__mro__
Out[5]: (abc.ABCMeta, type, object)
```

向上追溯，`ABCMeta` 最终所属的类也是 `type`。所有类都直接或间接地是 `type` 的实例，不过只有元类同时也是 `type` 的子类。若想理解元类，一定要知道这种关系:元类(如 `ABCMeta`)从 `type` 类继承了构建类的能力。

{% asset_img meta.png %}

我们要抓住的重点是，所有类都是 `type` 的实例，但是元类还是 `type` 的子类，因此可以作为制造类的工厂。具体来说，元类可以通过实现 `__init__` 方法定制实例。元类的 `__init__` 方法可以做到类装饰器能做的任何事情，但是作用更大。元类的 `__init__` 方法有四个参数：

- `self`：要初始化的类对象（一般改名为 `cls`）
- `name`：类名
- `bases`：基类列表
- `dic`：类的属性名和值

## 21.5 定制描述符的元类

```python
class EntityMeta(type):
    
    def __init__(cls, name, bases, attr_dict):
        super().__init__(name, bases, attr_dict)
        for key, attr in attr_dict.items():
            if isinstance(attr, Validated):
                type_name = type(attr).__name__
                attr.storage_name = '_{}#{}'.format(type_name, key)

class Entity(metaclass=EntityMeta):
    """带有验证字段的业务实体"""
```

因为元类会影响使用期作为元类的类及所有子类的初始化，所以我们可以在元类中定义一些行为，这样就能够作用在整个类继承链条的类上。

## 21.6 元类的特殊方法 `__prepare__`

在某些应用中，可能需要知道类的属性定义的顺序。如前所述，`type` 构造方法及元类的 `__new__` 和 `__init__` 方法都会收到要计算的类的定义体，形式是名称到属性的映像。然而在默认情况下，那个映射是字典;也就是说，元类或类装饰器获得映射时，属性在类定义体中的顺序已经丢失了。

这个问题的解决办法是，使用 Python 3 引入的特殊方法 `__prepare__`。 这个特殊方法只在元类中有用，而且必须声明为类方法(即要使用 `@classmethod` 装饰器定义)。解释器调用元类的 `__new__` 方法之前会先调用 `__prepare__` 方法，使用类定义体中的属性创建映射。

`__prepare__` 方法的第一个参数是元类，随后两个参数分别是要构建的*类的名称*和*基类组成的元组*，返回值必须是**映射**。元类构建新类时，`__prepare__` 方法返回的映射会传给 `__new__` 方法的最后一个参数，然后再传给 `__init__` 方法。

```python
import collections

class EntityMeta(type):
    
    @classmethod
    def __prepare__(cls, name, bases):
        return collections.OrderedDict()
    
    def __init__(cls, name, bases, attr_dict):
        super().__init__(name, bases, attr_dict)
        cls._field_names = []
        for key, attr in attr_dict.items():
            if isinstance(attr, Validated):
                type_name = type(attr).__name__
                attr.storage_name = '{}#{}'.format(type_name, key)
                cls._field_names.append(key)
```

在上述例子中，由于 `__prepare__` 方法返回的是一个有序的字典，所以在 `__init__` 中的 for 循环里我们能够根据属性定义的顺序来进行遍历。

在现实世界中，框架和库会使用元类协助程序员执行很多任务，例如:

- 验证属性 
- 一次把装饰器依附到多个方法上 
- 序列化对象或转换数据 
- 对象关系映射
- 基于对象的持久存储 
- 动态转换使用其他语言编写的类结构

## 21.7 类作为对象

`cls.__bases__`：由类的基类组成的元组。
`cls.__qualname__`：Python 3.3 新引入的属性，其值是类或函数的限定名称，即从模块的全局作用域到类的点分路径。
`cls.__subclasses__()`：这个方法返回一个列表，包含类的直接子类。
`cls.mro()`：构建类时，如果需要获取储存在类属性 `__mro__` 中的超类元组，解释器会调用这个方法。