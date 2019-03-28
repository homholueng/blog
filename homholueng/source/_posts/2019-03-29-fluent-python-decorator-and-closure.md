---
title: Chapter 7 - 函数装饰器和闭包
tags:
  - Python
categories:
  - Fluent Python
date: 2019-03-29 00:06:19
---


## 使用装饰器改进策略模式

在 Python 策略模式中。我们可以使用装饰器来帮助我们统一管理自定义的策略函数：

```python
promos = []

def promotion(promo_func):
    promos.append(promo_func)
    return promo_func

@promotion
def fidelity(order):
    pass

@promotion
def bulk_item(order):
    pass

@promotion
def large_order(order):
    pass

def best_promo(order):
    return max(promo(order) for promo in promos)

```

使用装饰器来管理策略函数的好处在于：策略函数能够定义在系统的任何地方，只要其使用 `@promotion` 来进行修饰我们就能够对其进行统一管理。

对于一些需要开放出去让开发者进行自定义，但是系统内部又需要进行统一管理的组件，使用装饰器是一个不错的选择。另外，如果我们需要进行统一管理的组件内部包含了更多的信息，此时函数可能相对来说就显得太轻量了，这时候，我们不发考虑使用元类来实现分散组件的统一管理。


## 闭包

闭包指延伸了作用域的函数，其中包含函数定义体中引用、但是不在定义体中定义的非全局变量。

举个例子：

```python
def make_averager():
    series = []

    def averager(new_value):
        series.append(new_value)
        total = sum(series)
        return total / len(series)
    
    return averager

In [2]: avg = make_averager()

In [3]: avg(10)
Out[3]: 10.0

In [4]: avg(11)
Out[4]: 10.5

In [5]: avg(12)
Out[5]: 11.0
```

在 `averager` 函数中，`series` 是自由变量（free variable），这是一个技术术语，指未在本地作用域中绑定的变量，如下图所示，`averager` 的闭包延伸到其作用域之外，并包含了自由变量 `series` 的绑定：

{% asset_img free_variable.png [自由变量] %}

继续深入探查 `avg` 函数：

```python
In [6]: avg.__code__.co_varnames
Out[6]: ('new_value', 'total')

In [7]: avg.__code__.co_freevars
Out[7]: ('series',)

In [8]: avg.__closure__
Out[8]: (<cell at 0x104affd98: list object at 0x104bd8688>,)

In [9]: avg.__closure__[0].cell_contents
Out[9]: [10, 11, 12]
```

可以看到，`series` 绑定在 `avg` 函数的 `__closure__` 属性中。`avg.__closure__` 中的各个元素对应于 `avg.__code__.co_freevars` 中的一个名称。这些元素的 `cell_contents` 属性保存着该变量真正的值。

综上，闭包是一种函数，它会保留定义函数时存在的自由变量的绑定，这样调用函数时，虽然定义作用域不可用了，但是仍能使用那些绑定。

## nonlocal 声明

上面例子中实现的 `make_averager` 函数的效率其实并不高，因为我们将所有的值都存在了一个数组中，然后每次都会调用 `sum` 来求和并且算平均值；更加高效的做法是只记录历史总值和元素个数，然后使用这两个数计算平均值。

```python
def make_averager():
    count = 0
    total = 0

    def averager(new_value):
        count += 1
        total += new_value
        return total / count
    
    return averager

In [2]: avg = make_averager()

In [3]: avg(10)
---------------------------------------------------------------------------
UnboundLocalError                         Traceback (most recent call last)
<ipython-input-3-ace390caaa2e> in <module>
----> 1 avg(10)

<ipython-input-1-97b2ca7b5c4e> in averager(new_value)
      4
      5     def averager(new_value):
----> 6         count += 1
      7         total += new_value
      8         return total / count

UnboundLocalError: local variable 'count' referenced before assignment
```

但是在使用的时候却跑出了 `UnboundLocalError` 错误，出现这个错误的原因是当 `count` 是数字或任何不可变类型时，`count += 1` 语句的作用其实与 `count = count + 1` 一样。因此，我们在 averager 的定义体中为 `count` 赋值了，这会把 `count` 变成局部变量，这样，`count` 就不是自由变量了，因此不会保存在闭包中。

为了解决这个问题，Python 3 引入了 `nonlocal` 声明。它的作用是把变量标记为自由变量，即使在函数中为变量赋予新值了，也会变成自由变量：

```python
def make_averager():
    count = 0
    total = 0

    def averager(new_value):
        nonlocal count, total
        count += 1
        total += new_value
        return total / count
    
    return averager
```

> 在 python2 中，还没有引入 `nonlocal` 关键字，此时可以使用一个可变对象（如字典）把这些变量存储起来，并让该可变对象称为一个自由变量。


## 实现一个简单的装饰器

我们知道，在函数上使用装饰器会使得我们丢失函数的真正信息，这个时候，可以借助 `functools.wraps` 装饰器来把被装饰的函数相关属性复制到装饰后的函数中：

```python
import time
import functools

def clock(func):
    @functools.wraps(func)
    def clocked(*args, **kwargs):
        t0 = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - t0
        
        name = func.__name__ 
        arg_lst = [] 
        if args:
            arg_lst.append(', '.join(repr(arg) for arg in args)) 
        if kwargs: 
            pairs = ['%s=%r' % (k, w) for k, w in sorted(kwargs.items())]
            arg_lst.append(', '.join(pairs)) 
        
        arg_str = ', '.join(arg_lst) 
        print('[%0.8fs] %s(%s) -> %r ' % (elapsed, name, arg_str, result))
        
        return result
    return clocked
```

## 标准库中的装饰器

### 使用 functools.lru_cache 做备忘

`functools.lru_cache` 会为其装饰的函数添加基于 LRU（Least Recently Used）缓存，避免一些耗时操作的重复计算，下面我们使用斐波那契数的递归计算来作为例子：

```python

@clock
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 2) + fibonacci(n - 1)

In [9]: fibonacci(6)
[0.00000000s] fibonacci(0) -> 0
[0.00000095s] fibonacci(1) -> 1
[0.00012326s] fibonacci(2) -> 1
[0.00000119s] fibonacci(1) -> 1
[0.00000119s] fibonacci(0) -> 0
[0.00000000s] fibonacci(1) -> 1
[0.00003910s] fibonacci(2) -> 1
[0.00008202s] fibonacci(3) -> 2
[0.00026703s] fibonacci(4) -> 3
[0.00000072s] fibonacci(1) -> 1
[0.00000000s] fibonacci(0) -> 0
[0.00000000s] fibonacci(1) -> 1
[0.00003386s] fibonacci(2) -> 1
[0.00006771s] fibonacci(3) -> 2
[0.00000000s] fibonacci(0) -> 0
[0.00000119s] fibonacci(1) -> 1
[0.00003695s] fibonacci(2) -> 1
[0.00000095s] fibonacci(1) -> 1
[0.00000191s] fibonacci(0) -> 0
[0.00000095s] fibonacci(1) -> 1
[0.00003672s] fibonacci(2) -> 1
[0.00009012s] fibonacci(3) -> 2
[0.00015903s] fibonacci(4) -> 3
[0.00026083s] fibonacci(5) -> 5
[0.00056100s] fibonacci(6) -> 8
Out[9]: 8
```

可以看到，对于相同参数的调用重复了很多次，让我们加上缓存试试：

```python
import functools

@functools.lru_cache()
@clock
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 2) + fibonacci(n - 1)

In [11]: fibonacci(6)
[0.00000000s] fibonacci(0) -> 0
[0.00000215s] fibonacci(1) -> 1
[0.00014687s] fibonacci(2) -> 1
[0.00000191s] fibonacci(3) -> 2
[0.00020289s] fibonacci(4) -> 3
[0.00000191s] fibonacci(5) -> 5
[0.00024819s] fibonacci(6) -> 8
Out[11]: 8
```

加上缓存之后，调用次数明显减少，另外，这个装饰器是会接收配置参数的：

- maxsize：指定最多存储多少个调用的结果
- typed：将不同类型的参数分开保存（例如将 1 与 1.0 区分开），顺带一说，`lru_cache` 使用字典存储结果，而且键根据调用时传入的位置参数和关键字参数创建，所以被其装饰的函数的所有参数都必须是可散列的。

```python
functools.lru_cache(maxsize=128, typed=False)
```

### 单分派泛函数

有些时候我们可能需要根据函数参数类型来确定接下来要执行的逻辑，由于 Python 中没有泛型的概念，所以这个时候一般的做法是使用 `if/else` 语句来判断参数类型，或者使用字典来维护每种不同类型的参数所对应的处理逻辑。

但是在 Python 3.4 之后，我们有了新的选择：

```python
from functools import singledispatch
import numbers

@singledispatch
def process_it(obj):
    print('This is a object')

@process_it.register(str)
def _(text):
    print('This is a str')

@process_it.register(numbers.Integral)
def _(n):
    print('This is a interger')

In [17]: process_it('str')
This is a str

In [18]: process_it(1)
This is a interger

In [19]: process_it(object())
This is a object

```

> PyPI 中 的 singledispatch 包（https://pypi.python.org/pypi/singledispatch） 可以向后兼容 Python 2.6 到 Python 3.3

另外，对于一些功能比较特殊的装饰器，最好通过实现 `__call__` 方法的类实现。
