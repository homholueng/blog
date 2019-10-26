---
title: Chapter 20 - 属性描述符
tags:
  - Python
categories:
  - Fluent Python
---

**描述符是对多个属性运用相同存取逻辑的一种方式，是实现了特定协议的类，这个协议包括 `__get__`、`__set__` 和 `__delete__` 方法。**

## 20.1 描述符示例：验证属性

### 20.1.1　LineItem类第3版：一个简单的描述符

实现了 `__get__`、`__set__` 或 `__delete__` 方法的类是描述符。描述符的用法是，创建一个实例，作为另一个类的类属性。

在学习使用描述符前，需要先清除以下概念：

- 描述符类：实现描述符协议的类
- 托管类：把描述符实例声明为雷属性的类
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