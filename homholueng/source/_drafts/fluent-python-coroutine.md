---
title: Chapter 16 - 协程
tags:
  - Python
categories:
  - Fluent Python
---

字典为动词 “to yield” 给出了两个释义：产出和让步。对于 Python 生成器中的 yield 来说，这两个含义都成立。`yield item` 这行代码会产出一个值，提供给 `next(...)` 的调用方；此外，还会作出让步，暂停执行生成器，让调用方继续工作，直到需要使用另一个值时再调用 `next()`。调用方会从生成器中拉取值。

yield 关键字甚至还可以不接收或传出数据。不管数据如何流动，yield 都是一种流程控制工具，使用它可以实现协作式多任务：协程可以把控制器让步给中心调度程序，从而激活其他的协程。

从根本上把 yield 视作控制流程的方式，这样就好理解协程了。

## 16.1 生成器如何进化成协程

协程的底层架构在 “PEP 342—Coroutines via Enhanced Generators”（https://www.python.org/dev/peps/pep-0342/)中定义，并在 Python 2.5（2006年）实现了。自此之后，yield 关键字可以在表达式中使用，而且生成器 API 中增加了 `.send(value)` 方法。生成器的调用方可以使用 `.send(...)` 方法发送数据，发送的数据会成为生成器函数中 yield 表达式的值。因此，生成器可以作为协程使用。协程是指一个过程，这个过程与调用方协作，产出由调用方提供的值。

除了 `.send(...)` 方法，PEP 342 还添加了 `.throw(...)` 和 `.close()` 方法：前者的作用是让调用方抛出异常，在生成器中处理；后者的作用是终止生成器。

协程最近的演进来自 Python 3.3（2012 年）实现的 “PEP 380—Syntax for Delegating to a Subgenerator”（https://www.python.org/dev/peps/pep0380/)。PEP 380 对生成器函数的句法做了两处改动，以便更好地作为协程使用。

- 现在，生成器可以返回一个值；以前，如果在生成器中给 return 语句提供值，会抛出 SyntaxError 异常。
- 新引入了 yield from 句法，使用它可以把复杂的生成器重构成小型的嵌套生成器，省去了之前把生成器的工作委托给子生成器所需的大量样板代码。

## 16.2　用作协程的生成器的基本行为

下面是一个最简单的协程的演示：

```python
In [1]: def simple_coroutine():
   ...:     print('-> coroutine started')
   ...:     x = yield
   ...:     print('-> coroutine received:', x)
   ...:

In [2]: my_coro = simple_coroutine()

In [3]: my_coro
Out[3]: <generator object simple_coroutine at 0x106cfeaf0>

In [4]: next(my_coro)
-> coroutine started

In [5]: my_coro.send(42)
-> coroutine received: 42
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-5-7c96f97a77cb> in <module>
----> 1 my_coro.send(42)

StopIteration:
```

首先，协程使用生成器函数定义，即定义体中含有 yield 关键字，与创建生成器的方式一下，调用函数得到生成器对象；然后调用 next 函数，因为生成器还没启动，没有在 yield 处暂停。调用 send 发送数据后，yield 表达式会计算出发送过去的值，协程恢复执行，一直运行到下一个 yield 表达式或者终止，实例中控制权流到了协程定义体的末尾，导致生成器像往常一样抛出了 StopIteration 异常。

协程会处于一下四个状态中的其中一个，使用 `inspect.getgeneratorstate` 能够获取协程的状态：

- `GEN_CREATED`：等待开始执行。
- `GEN_RUNNING`：解释器正在执行。
- `GEN_SUSPENDED`：在 yield 表达式处暂停。
- `GEN_CLOSED`：执行结束。
  
```python
In [6]: import inspect

In [7]: inspect.getgeneratorstate(my_coro)
Out[7]: 'GEN_CLOSED'

In [8]: my_coro = simple_coroutine()

In [9]: inspect.getgeneratorstate(my_coro)
Out[9]: 'GEN_CREATED'

In [10]: next(my_coro)
-> coroutine started

In [11]: inspect.getgeneratorstate(my_coro)
Out[11]: 'GEN_SUSPENDED'

In [12]: my_coro.send(42)
-> coroutine received: 42
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-12-7c96f97a77cb> in <module>
----> 1 my_coro.send(42)

StopIteration:

In [13]: inspect.getgeneratorstate(my_coro)
Out[13]: 'GEN_CLOSED'
```

因为 send 方法的参数会成为暂停的 yield 表达式的值，所以，仅当协程处于暂停状态时才能调用 send 方法。

如果创建协程对象后立即把 `None` 之外的值发给它，会出现下述错误：

```python
In [14]: my_coro = simple_coroutine()

In [15]: my_coro.send(42)
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-15-7c96f97a77cb> in <module>
----> 1 my_coro.send(42)

TypeError: can't send non-None value to a just-started generator
```

最先调用 `next(my_coro)` 函数这一步通常称为“预激”（prime）协程 （即，让协程向前执行到第一个 yield 表达式，准备好作为活跃的协程使用）。

## 16.3　示例：使用协程计算移动平均值

