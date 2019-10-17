---
title: Chapter 19 - 动态属性和特性
tags:
  - Python
categories:
  - Fluent Python
date: 2019-10-17 10:24:43
---


> 特性至关重要的地方在于，特性的存在使得开发者可以非常安全并且确定可行地将公共数据属性作为类的公共接口的一部分开放出来。

### 19.1.3　使用 `__new__` 方法以灵活的方式创建对象

我们通常把 `__init__` 称为构造方法，这是从其他语言借鉴过来的术语。其实，用于构建实例的是特殊方法 `__new__`：这是个类方法（使用特殊方式处理，因此不必使用 `@classmethod` 装饰器），必须返回一个实例。返回的实例会作为第一个参数（即 self）传给 `__init__` 方法。因为调用 `__init__` 方法时要传入实例，而且禁止返回任何值，所以 `__init__` 方法其实是“初始化方法”。真正的构造方法是 `__new__`。我们几乎不需要自己编写 `__new__` 方法，因为从 object 类继承的实现已经足够了。

刚才说明的过程，即从 `__new__` 方法到 `__init__` 方法，是最常见的，但不是唯一的。`__new__` 方法也可以返回其他类的实例，此时，解释器不会调用 `__init__` 方法。

### 19.1.4 shelve 模块

标准库中有个 shelve（架子）模块，这名字听起来怪怪的，可是如果知道 pickle（泡菜）是 Python 对象序列化格式的名字，还是在那个格式与对象之间相互转换的某个模块的名字，就会觉得以 shelve 命名是合理的。泡菜坛子摆放在架子上，因此 shelve 模块提供了 pickle 存储方式。

`shelve.open` 高阶函数返回一个 `shelve.Shelf` 实例，这是简单的键 值对象数据库，背后由 dbm 模块支持，具有下述特点：

- `shelve.Shelf` 是 `abc.MutableMapping` 的子类，因此提供了处理映射类型的重要方法。
- `shelve.Shelf` 类还提供了几个管理 I/O 的方法，如 sync 和 close；它也是一个上下文管理器。
- 只要把新值赋予键，就会保存键和值。
- 键必须是字符串。
- 值必须是 pickle 模块能处理的对象。

```python
import shelve

db = shelve.open(DB_NAME)
db[key] = SomeObject()

obj = db[key]
```

### 19.1.5 使用特性获取链接的记录

#### 防止实例属性覆盖类属性

在使用特性时，如果我们要使用对象的类属性，为了防止对象的实例属性覆盖掉我们对雷属性的访问，建议按照以下的方式访问类属性：

```python
class Obj:

    @property
    def attr(self):
        self.__class__.some_attr
```

#### 使用特性覆盖类的实例属性

我们可以使用特性覆盖我们实例中的某些属性，但是，当我们要在类中访问实例的属性时，就需要按照如下方式进行访问：

```python
class Obj:

    @property
    def attr(self):
        self.__dict__['attr']
```

## 19.2 使用特性验证属性

### 19.2.2 能验证值的特性

下面所展示的类使用 property 来向外暴露内部的属性，并且防止用户将属性设置为非法的值：

```python
class LineItem:
    
    def __init__(self, description, weight, price):
        self.description = description
        self.weight = weight  # 在这里就使用了 setter 来赋值
        self.price = price
        
    def subtoal(self):
        return self.weight * self.price
    
    @property
    def weight(self):
        return self.__weight  # 真正存储值的地方
    
    @weight.setter
    def weight(self, value):
        if value > 0:
            self.__weight = value
        else:
            raise ValueError('value must be > 0')
```

## 19.3 特性全解析

虽然内置的 property 经常用作装饰器，但它其实是一个类。property 构造方法的完整签名如下：

```python
property(fget=None, fset=None, fdel=None, doc=None)
```

经典的特性设置方式其实是这样的：

```python
class LineItem:

    def set_weight(self, value):
        pass
    
    def get_weight(self):
        pass

    weight = property(get_weight, set_weight)

```

而且，我们知道实例属性会覆盖类属性，虽然特性是类属性，**但是实例属性并不会覆盖类的特性**。

虽然特性是个强大的功能，不过有时更适合使用简单的或底层的替代方案。

## 19.6 处理属性的重要属性和函数

### 19.6.1 影响属性处理方式的特殊属性

- `__class__`：对象所属类的引用（即 `obj.__class__` 与 `type(obj)` 的作用相同）。Python 的某些特殊方法，例如 `__getattr__`，只在对象的类中寻找，而不在实例中寻找。
- `__dict__`：一个映射，存储对象或类的可写属性。有 `__dict__` 属性的对象，任何时候都能随意设置新属性。如果类有 `__slots__` 属性，它的实例可能没有 `__dict__` 属性。
- `__slot__`：类可以定义这个这属性，限制实例能有哪些属性。

### 19.6.2 处理属性的内置函数

- `dir`：列出对象的大多数属性。dir 函数的目的是交互式使用，因此没有提供完整的属性列表，只列出一组“重要的”属性名。

### 19.6.3 处理属性的特殊方法

- `__delattr__(self, name)`：只要使用 del 语句删除属性，就会调用这个方法。
- `__dir__(self)`：把对象传给 dir 函数时调用，列出属性。
- `__getattr__(self, name)`：仅当获取指定的属性失败，搜索过 obj、Class 和超类之后调用。
- `__getattribute__(self, name)`：尝试获取指定的属性时总会调用这个方法，不过，寻找的属性是特殊属性或特殊方法时除外。
- `__setattr__(self, name, value)`：尝试设置指定的属性时总会调用这个方法。