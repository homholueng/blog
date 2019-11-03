---
title: Chapter 20 - 属性描述符
tags:
  - Python
categories:
  - Fluent Python
date: 2019-11-03 20:37:52
---


**描述符是对多个属性运用相同存取逻辑的一种方式，是实现了特定协议的类，这个协议包括 `__get__`、`__set__` 和 `__delete__` 方法。**

## 20.1 描述符示例：验证属性

### 20.1.1　一个简单的描述符

实现了 `__get__`、`__set__` 或 `__delete__` 方法的类是描述符。描述符的用法是，创建一个实例，作为另一个类的类属性。

在学习使用描述符前，需要先清除以下概念：

- 描述符类：实现描述符协议的类
- 托管类：把描述符实例声明为类属性的类
- 描述符实例：描述符类的各个实例，声明为托管类的类属性
- 托管实例：托管类的实例
- 存储属性：托管实例中存储自身托管属性的属性
- 托管属性：托管类中由描述符实例处理的公开属性

```python

class Quantity:
    
    def __init__(self, storage_name):
        self.storage_name = storage_name
        
    def __set__(self, instance, value):
        if value > 0:
            instance.__dict__[self.storage_name] = value
        else:
            raise ValueError('value must be > 0')
    
class LineItem:
    weight = Quantity('weight')
    price = Quantity('price')
    
    def __init__(self, description, weight, price):
        self.description = description
        self.weight = weight
        self.price = price
    
    def subtotal(self):
        return self.weight * self.price

```

> 编写 `__set__` 方法时，要记住 `self` 和 `instance` 参数的意思：`self` 是描述符实例，`instance` 是托管实例。

### 20.1.3 一种新型描述符

我们可以将描述符定义中通用的部分提取出来，将一些可能有开发者自定义的部分放开让子类来实现，采用模板方法的设计模式来进行抽象和简化：

{% asset_img new_style_descriptor.png %}

代码如下：

```python
import abc

class AutoStorage:
    __counter = 0
    
    def __init__(self):
        cls = self.__class__
        prefix = cls.__name__
        index = cls.__counter__
        self.storage_name = '_{}#{}'.format(prefix, index)
        cls.__counter += 1
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        
        return getattr(instance, self.storage_name)

    def __set__(self, instance, value):
        setattr(instance, self.storage_name, value)

class Validated(abc.ABC, AutoStorage):
    
    def __set__(self, instance, value):
        value = self.validate(instance, value)
        super().__set__(instance, value)
    
    @abc.abstractmethod
    def validate(self, instance, value):
        """return validated value or raise ValueError"""
        
    
class Quantity(Validated):
    
    def validate(self, instance, value):
        if value <= 0:
            raise ValueError('value must be > 0')
            
        return value

class BonBlank(Validated):
    
    def validate(self, isinstance, value):
        value = value.strip()
        if len(value) == 0:
            raise ValueError('value cannot be empty or blank')
            
        return value
```

上述的两个例子演示了描述符的典型用途：管理数据属性。这种描述符也叫覆盖型描述符，因为描述符的 `__set__` 方法使用托管实例中的同名属性覆盖(即插手接管)了要设置的属性。

## 20.2 覆盖型与非覆盖型描述符对比

### 20.2.1 覆盖型描述符

实现 `__set__` 方法的描述符属于覆盖型描述符，因为虽然描述符是类属性，但是实现 `__set__` 方法的话，会覆盖对实例属性的赋值操作。

### 20.2.2 没有 `__get__` 方法的覆盖型描述符

此时，只有写操作由描述符处理。通过实例读取描述符会返回描述符对象本身，因为没有处理读操作的 `__get__` 方法。如果直接通过实例的 `__dict__` 属性创建同名实例属性，以后再设置那个属性时，仍会由 `__set__` 方法插手接管，但是读取那个属性的话，就会直接从实例中返回新赋予的值，而不会返回描述符对象。

### 20.2.3 非覆盖型描述符

没有实现 `__set__` 方法的描述符是非覆盖型描述符。如果设置了同名的实例属性，描述符会被遮盖，致使描述符无法处理那个实例的那个属性。

## 20.3 方法是描述符

在类中定义的函数属于绑定方法(bound method)，因为用户定义的函数都有 `__get__` 方法，所以依附到类上时，就相当于描述符。

描述符一样，通过托管类访问时，函数的 `__get__` 方法会返回自身的引用。但是，通过实例访问时，函数的 `__get__` 方法返回的是绑定方法对象：一种可调用的对象，里面包装着函数，并把托管实例(例如 obj)绑定给函数的第一个参数(即 self)。

下面我们来测试一下：

```python
import collections

class Text(collections.UserString):
    def __repr__(self):
        return 'Text({!r})'.format(self.data)
    
    def reverse(self): 
        return self[::-1]
```

```
>>> word = Text('forward') 
>>> word
Text('forward')
>>> word.reverse()
Text('drawrof')
>>> Text.reverse(Text('backward'))
Text('drawkcab')
>>> type(Text.reverse), type(word.reverse)
(<class 'function'>, <class 'method'>)
>>> list(map(Text.reverse, ['repaid', (10, 20, 30), Text('stressed')])) ['diaper', (30, 20, 10), Text('desserts')]
>>> Text.reverse.__get__(word)
<bound method Text.reverse of Text('forward')>
>>> Text.reverse.__get__(None, Text)
<function Text.reverse at 0x101244e18>
>>> word.reverse
<bound method Text.reverse of Text('forward')>
>>> word.reverse.__self__ # 绑定方法对象有个 __self__ 属性，其值是调用这个方法的实例引用。
Text('forward')
>>> word.reverse.__func__ is Text.reverse
True
```

## 20.4 描述符用法建议

- 使用特性来创建只读属性：创建只读属性最简单的方式是使用特性，没必要再自定义一个描述符
- 只读描述符必须有 `__set__` 方法：如果使用描述符类实现只读属性，要记住，`__get__` 和 `__set__` 两个方法必须都定义，否则，实例的同名属性会遮盖描述符。
- 用于验证的描述符可以只有 `__set__` 方法：对仅用于验证的描述符来说，`__set__` 方法应该检查 value 参数获得的值，如果有效，使用描述符实例的名称为键，直接在实例的 `__dict__` 属性中设置。
- 仅有 `__get__` 方法的描述符可以实现高效缓存：如果只编写了 `__get__` 方法，那么创建的是非覆盖型描述符。这种描述符可用于执行某些耗费资源的计算，然后为实例设置同名属性，缓存结果。同名实例属性会遮盖描述符，因此后续访问会直接从实例的 `__dict__` 属性中获取值，而不会再触发描述符的 `__get__` 方法。

