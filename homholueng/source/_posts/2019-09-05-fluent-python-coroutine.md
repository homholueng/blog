---
title: Chapter 16 - 协程
tags:
  - Python
categories:
  - Fluent Python
date: 2019-09-05 20:14:56
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

## 16.4　预激协程的装饰器


如果不预激，那么协程没什么用，为了简化协程的用法，有时会使用一个预激装饰器。

```python
from functools import wraps

def coroutine(func):
    @wraps(func)
    def primer(*args, **kwargs):
        gen = func(*args, **kwargs)
        next(gen)
        return gen
    return primer

In [2]: @coroutine
   ...: def simple_coroutine():
   ...:     print('-> coroutine started')
   ...:     x = yield
   ...:     print('-> coroutine received:', x)
   ...:

In [3]: gen =simple_coroutine()
-> coroutine started

In [4]: gen.send(42)
-> coroutine received: 42
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-4-72d0e54668c5> in <module>
----> 1 gen.send(42)

StopIteration:
```


## 16.5　终止协程和异常处理

协程中未处理的异常会向上冒泡，传给 next 函数或 send 方法的调用方（即触发协程的对象）。

这个特性允许我们发送某个哨符值，让协程退出。内置的 `None` 和 `Ellipsis` 等常量经常用作哨符值。`Ellipsis` 的优点是，数据流中不太常有这个值。

从 Python 2.5 开始，客户代码可以在生成器对象上调用两个方法，显式地把异常发给协程。

这两个方法是 throw 和 close：

- `generator.throw(exc_type[, exc_value[, traceback]])`：致使生成器在暂停的 yield 表达式处抛出指定的异常。如果生成器处理了抛出的异常，代码会向前执行到下一个 yield 表达式，而产出的值会成为调用 `generator.throw` 方法得到的返回值。如果生成器没有处理抛出的异常，异常会向上冒泡，传到调用方的上下文中。
- `generator.close()`：致使生成器在暂停的 yield 表达式处抛出 GeneratorExit 异常。 如果生成器没有处理这个异常，或者抛出了 StopIteration 异常（通常是指运行到结尾），调用方不会报错。如果收到 GeneratorExit 异常，生成器一定不能产出值，否则解释器会抛出 RuntimeError 异常。 生成器抛出的其他异常会向上冒泡，传给调用方。

```python
class DemoException(Exception):
    pass

def demo_exc_handling():
    print('-> coroutine started')
    while True:
        try:
            x = yield
        except DemoException:
            print('*** DemoException handled. Continuing...')
        else:
            print('-> coroutine received: {!r}'.format(x))
    
    raise RuntimeError('This line should never run.')

In [2]: exc_coro = demo_exc_handling()

In [3]: next(exc_coro)
-> coroutine started

In [4]: exc_coro.send(11)
-> coroutine received: 11

In [5]: exc_coro.send(22)
-> coroutine received: 22

In [6]: exc_coro.throw(DemoException())
*** DemoException handled. Continuing...

In [8]: inspect.getgeneratorstate(exc_coro)
Out[8]: 'GEN_SUSPENDED'

In [9]: exc_coro.throw(ZeroDivisionError())
---------------------------------------------------------------------------
ZeroDivisionError                         Traceback (most recent call last)
<ipython-input-9-e03abf340dfc> in <module>
----> 1 exc_coro.throw(ZeroDivisionError())

<ipython-input-1-cc1425411a81> in demo_exc_handling()
      6     while True:
      7         try:
----> 8             x = yield
      9         except DemoException:
     10             print('*** DemoException handled. Continuing...')

ZeroDivisionError:

In [11]: inspect.getgeneratorstate(exc_coro)
Out[11]: 'GEN_CLOSED'

In [12]: exc_coro = demo_exc_handling()

In [13]: next(exc_coro)
-> coroutine started

In [14]: exc_coro.send(22)
-> coroutine received: 22

In [15]: exc_coro.close()

In [16]: inspect.getgeneratorstate(exc_coro)
Out[16]: 'GEN_CLOSED'
```

## 16.6　让协程返回值

下面是 averager 协程的不同版本，这一版会返回结果：

```python
from collections import namedtuple

Result = namedtuple('Result', 'count average')

def averager():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield
        if term is None:
            break
        total += term
        count += 1
        average = total / count
    return Result(count, average)

In [2]: coro_avg = averager()

In [3]: next(coro_avg)

In [4]: coro_avg.send(10)

In [5]: coro_avg.send(30)

In [6]: coro_avg.send(35)

In [7]: coro_avg.send(None)
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-7-a9c80bbec98f> in <module>
----> 1 coro_avg.send(None)

StopIteration: Result(count=3, average=25.0)
```

发送 `None` 会终止循环，导致协程结束，返回结果。一如既往，生成器对象会抛出 StopIteration 异常。异常对象的 value 属性保存着返回的值。

```python
In [8]: coro_avg = averager()

In [9]: next(coro_avg)

In [10]: coro_avg.send(30)

In [11]: coro_avg.send(10)

In [12]: try:
    ...:     coro_avg.send(None)
    ...: except StopIteration as exc:
    ...:     result = exc.value
    ...:

In [13]: result
Out[13]: Result(count=2, average=20.0)
```

获取协程的返回值虽然要绕个圈子，但这是 PEP 380 定义的方式，当我们意识到这一点之后就说得通了：yield from 结构会在内部自动捕获 StopIteration 异常。这种处理方式与 for 循环处理 StopIteration 异常的方式一样：循环机制使用用户易于理解的方式处理异常。对 `yield from` 结构来说，解释器不仅会捕获 StopIteration 异常，还会把 value 属性的值变成 `yield from` 表达式的值。可惜，我们无法在控制台中使用交互的方式测试这种行为，因为在函数外部使用 `yield from`（以及 yield）会导致句法出错。


## 16.7　使用yield from

首先要知道，`yield from` 是全新的语言结构。它的作用比 yield 多很 多，因此人们认为继续使用那个关键字多少会引起误解。在其他语言中，类似的结构使用 await 关键字，这个名称好多了，因为它传达了至关重要的一点：在生成器 gen 中使用 `yield from subgen()` 时，subgen 会获得控制权，把产出的值传给 gen 的调用方，即调用方可以直接控制 subgen。与此同时，gen 会阻塞，等待 subgen 终止。

`yield from` 的主要功能是打开双向通道，把最外层的调用方与最内层的子生成器连接起来，这样二者可以直接发送和产出值，还可以直接传入异常，而不用在位于中间的协程中添加大量处理异常的样板代码。有了这个结构，协程可以通过以前不可能的方式委托职责。

若想使用 `yield from` 结构，就要大幅改动代码。为了说明需要改动的部分，PEP 380 使用了一些专门的术语：

- 委派生成器：包含 `yield from <iterable>` 表达式的生成器函数。
- 子生成器：从 `yield from` 表达式中 `<iterable>` 部分获取的生成器。
- 调用方：指代调用**委派生成器**的客户端代码。

下面的代码使用 `yield from` 计算平均值并输出统计报告：

```python
from collections import namedtuple

Result = namedtuple('Result', 'count average')

# 子生成器
def averager():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield
        if term is None:
            break
        total += term
        count += 1
        average = total / count
    return Result(count, average)

# 委派生成器
def grouper(results, key):
    while True:
        results[key] = yield from averager()

# 客户端代码，即调用方
def main(data):
    results = {}
    for key, values in data.items():
        group = grouper(results, key)
        next(group)
        for value in values:
            group.send(value)
        group.send(None)
    
    print(results)

data = {
  'girls;kg': [40.9, 38.5, 44.3, 42.2, 45.2, 41.7, 44.5, 38.0, 40.6, 44.5], 
  'girls;m': [1.6, 1.51, 1.4, 1.3, 1.41, 1.39, 1.33, 1.46, 1.45, 1.43], 
  'boys;kg': [39.0, 40.8, 43.2, 40.8, 43.1, 38.6, 41.4, 40.6, 36.3], 
  'boys;m': [1.38, 1.5, 1.32, 1.25, 1.37, 1.48, 1.25, 1.49, 1.46],
}

if __name__ == '__main__':
    main(data)
```

在上述的例子中，grouper 发送的每个值都会经由 `yield from` 处理，通过管道传给 averager 实例。grouper 会在 `yield from` 表达式处暂停，等待 averager 实例处理客户端发来的值。averager 实例运行完毕后，返回的值绑定到 `results[key]` 上。while 循环会不断创建 averager 实例，处理更多的值。

而在 main 中，我们把各个 value 传给 grouper。传入的值最终到达 averager 函数中 `term = yield` 那一行；grouper 永远不知道传入的值是什么。

最后，把 None 传入 grouper，导致当前的 averager 实例终止，也让 grouper 继续运行，再创建一个 averager 实例，处理下一组值。

整个过程如下：

- 外层 for 循环每次迭代会新建一个 grouper 实例，赋值给 group 变量；group 是委派生成器。
- 调用 `next(group)`，预激委派生成器 grouper，此时进入 `while True` 循环，调用子生成器 averager 后，在 `yield from` 表达式处暂停。
- 内层 for 循环调用 `group.send(value)`，直接把值传给子生成器 averager。同时，当前的 grouper 实例（group）在 `yield from` 表达式处暂停。
- 内层循环结束后，group 实例依旧在 `yield from` 表达式处暂停， 因此，grouper 函数定义体中为 `results[key]` 赋值的语句还没有执行。
- 如果外层 for 循环的末尾没有 `group.send(None)`，那么 averager 子生成器永远不会终止，委派生成器 group 永远不会再次激活，因此永远不会为 `results[key]` 赋值。
- 外层 for 循环重新迭代时会新建一个 grouper 实例，然后绑定到 group 变量上。前一个 grouper 实例（以及它创建的尚未终止的 averager 子生成器实例）被垃圾回收程序回收。

main，grouper，averager 的关系就如下图所示：

{% asset_img yield_from.png [yield from] %}


## 16.8　yield from 的意义

PEP 380 中对 `yield from` 行为的说明如下：

- 子生成器产出的值都直接传给委派生成器的调用方（即客户端代码）。
- 使用 `send()` 方法发给委派生成器的值都直接传给子生成器。如果 发送的值是 `None`，那么会调用子生成器的 `__next__()` 方法。如果发送的值不是 `None`，那么会调用子生成器的 `send()` 方法。如 果调用的方法抛出 StopIteration 异常，那么委派生成器恢复运行。任何其他异常都会向上冒泡，传给委派生成器。
- 生成器退出时，生成器（或子生成器）中的 `return expr` 表达式会触发 `StopIteration(expr)` 异常抛出。
- `yield from` 表达式的值是子生成器终止时传给 StopIteration 异常的第一个参数。
- 传入委派生成器的异常，除了 GeneratorExit 之外都传给子生成器的 `throw()` 方法。如果调用 `throw()` 方法时抛出 StopIteration 异常，委派生成器恢复运行。StopIteration 之外的异常会向上冒泡，传给委派生成器。
- 如果把 GeneratorExit 异常传入委派生成器，或者在委派生成器 上调用 `close()` 方法，那么在子生成器上调用 `close()` 方法，如果它有的话。如果调用 `close()` 方法导致异常抛出，那么异常会向上冒泡，传给委派生成器；否则，委派生成器抛出 GeneratorExit 异常。

在不考虑在委派生成器上调用 throw 和 close 及子生成器不抛出异常的情况下，`RESULT = yield from EXPR` 等价于下列代码：

```python
_i = iter(EXPR) # 子生成器，因为需要处理可迭代对象，所以调用了 iter 方法，对生成器对象调用 iter 会返回生成器自身
try:
    _y = next(_i)  # 子生成器产出的值
except StopIteration as _e:
    _r = _e.value  # 结果
else:
    while 1:
        _s = yield _y  # 客户端发送过来的值
        try:
            _y = _i.send(_s)
        except StopIteration as _e:
            _r = _e.value
            break

RESULT = _r
```

但是，现实情况要复杂一些，因为要处理客户对 `.throw(...)` 和 `.close()` 方法的调用，而这两个方法执行的操作必须传入子生成器。 此外，子生成器可能只是纯粹的迭代器，不支持 `.throw(...)` 和 `.close()` 方法，因此 `yield from` 结构的逻辑必须处理这种情况。。如果子生成器实现了这两个方法，而在子生成器内部，这两个方法都会触发异常抛出，这种情况也必须由 `yield from` 机制处理。调用方可能会无缘无故地让子生成器自己抛出异常，实现 `yield from` 结构时也必须处理这种情况。最后，为了优化，如果调用方调用 `next(...)` 函数或 `.send(None)` 方法，都要转交职责，在子生成器上调用 `next(...)` 函数；仅当调用方发送的值不是 `None` 时，才使用子生成器的 `.send(...)` 方法。

为了方便对比，下面列出 PEP 380 中扩充 `yield from` 表达式的完整伪代码：

```python
_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    while 1:
        try:
            _s = yield _y
        except GeneratorExit as _e:  # 这一部分用于关闭委派生成器和子生成器。因为子生成器可以是任 何可迭代的对象，所以可能没有 close 方法。
            try:
                _m = _i.close
            except AttributeError:
                pass
            else:
                _m()
            raise _e
        except BaseException as _e:  # 这一部分处理调用方通过 .throw(...) 方法传入的异常。同样，子生成器可以是迭代器，从而没有 throw 方法可调用——这种情况会导 致委派生成器抛出异常。
            _x = sys.exc_info()
            try:
                _m = _i.throw
            except AttributeError:
                raise _e
            else: # 如果子生成器有 throw 方法，调用它并传入调用方发来的异常。子生成器可能会处理传入的异常（然后继续循环）；可能抛出 StopIteration 异常（从中获取结果，赋值给 _r，循环结束）；还可能不处理，而是抛出相同的或不同的异常，向上冒泡，传给委派生成器。
                try:
                   _y = m(*_x)
                except StopIteration as _e:
                    _r = _e.value
                    break
        else:  # 如果产出值时没有异常……
            try:
                if _s is None:
                    _y = next(_i)  # 如果调用方最后发送的值是 None，在子生成器上调用 next 函数， 否则调用 send 方法。
                else:
                    _y = _i.send(_s)
            except StopIteration as _e:
                _r = _e.value
                break
RESULT = _r     
```

在伪代码的顶部，有行代码（`_y = next(_i)`）揭示了一个重要的细节：要预激子生成器。这表明，用于自动预激的装饰器与 `yield from` 结构不兼容。

## 16.9 使用协程做离散事件仿真

离散事件仿真（Discrete Event Simulation，DES）是一种把系统建模成一系列事件的仿真类型。在离散事件仿真中，仿真“钟”向前推进的量不是固定的，而是直接推进到下一个事件模型的模拟时间。假如我们抽象模拟出租车的运营过程，其中一个事件是乘客上车，下一个事件则是乘客下车。

离散事件仿真能够使用多线程或在单个线程中使用面向事件的编程技 术（例如事件循环驱动的回调或协程）实现。可以说，为了实现连续仿真，在多个线程中处理实时并行的操作更自然。而协程恰好为实现离散事件仿真提供了合理的抽象。

> 在仿真领域，进程这个术语指代模型中某个实体的活动，与操作系统中的进程无关。仿真系统中的一个进程可以使用操作系统中的一个进程实现，但是通常会使用一个线程或一个协程实现。



## 小结

> 生成器有三种不同的代码编写风格：有传统的拉取式（迭代器）、推送式（计算平均值的示例）、还有任务时。

使用协程做面向事件编程时，协程会不断把控制权让步给主循环，激活并向前运行其他协程，从而执行各个并发活动。这是一种协作式多任务：协程显式自主地把控制权让步给中央调度程序。而多线程实现的是抢占式多任务。调度程序可以在任何时刻暂停线程（即使在执行一个语句的过程中），把控制权让给其他线程。

