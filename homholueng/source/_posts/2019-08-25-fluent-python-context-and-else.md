---
title: Chapter 15 - 上下文管理器和 else 块
tags:
  - Python
categories:
  - Fluent Python
date: 2019-08-25 17:32:39
---


## 15.1　先做这个，再做那个：if 语句之外的 else 块

for/else、while/else 和 try/else 的语义关系紧密，不过与 if/else 差别很大：

- for：　仅当 for 循环运行完毕时（即 for 循环没有被 break 语句中止）才运行 else 块。
- while：仅当 while 循环因为条件为假值而退出时（即 while 循环没有被 break 语句中止）才运行 else 块。
- try：仅当 try 块中没有异常抛出时才运行 else 块。（注意，else 中抛出的异常不会由前面的 except 子句处理）

在所有情况下，如果异常或者 return、break 或 continue 语句导致 控制权跳到了复合语句的主块之外，else 子句也会被跳过。

## 15.2　上下文管理器和 with 块

上下文管理器对象存在的目的是管理 with 语句，就像迭代器的存在是为了管理 for 语句一样。

上下文管理器协议包含 `__enter__` 和 `__exit__` 两个方法。with 语句开始运行时，会在上下文管理器对象上调用 `__enter__` 方法。with 语句运行结束后，会在上下文管理器对象上调用 `__exit__` 方法，以此扮演 finally 子句的角色。

不管控制流程以哪种方式退出 with 块，都会在上下文管理器对象上调用 `__exit__` 方法，而不是在 `__enter__` 方法返回的对象上调用。

下面的例子使用一个精心制作的上下文管理器执行操作，以此强调上下文管理器与 `__enter__` 方法返回的对象之间的区别：

```python
In [5]: class LookingGlass:
   ...:
   ...:     def __enter__(self):
   ...:         import sys
   ...:         self.original_write = sys.stdout.write
   ...:         sys.stdout.write = self.reverse_write
   ...:         return 'JABBERWOCKY'
   ...:
   ...:     def reverse_write(self, text):
   ...:         self.original_write(text[::-1])
   ...:
   ...:     def __exit__(self, exc_type, exc_value, traceback):
   ...:         import sys
   ...:         sys.stdout.write = self.original_write
   ...:         if exc_type is ZeroDivisionError:
   ...:             print('Please DO NOT divide by zero!')
   ...:             return True
   ...:

In [6]: with LookingGlass() as what:
   ...:     print('Alice, Kitty and Snowdrop')
   ...:     print(what)
   ...:
pordwonS dna yttiK ,ecilA
YKCOWREBBAJ

In [7]: what
Out[7]: 'JABBERWOCKY'
```

- 如果一切正常，Python 调用 `__exit__` 方法时传入的参数是 `None`, `None`, `None`；如果抛出了异常，这三个参数是异常数据。
  - `exc_type`：异常类。
  - `exc_value`：异常实例。有时会有参数传给异常构造方法，例如错误消息，这些参数可以使用 `exc_value.args` 获取。
  - `traceback`：traceback 对象。
- 如果 `__exit__` 方法返回 `None`，或者 `True` 之外的值，with 块中的任何异常都会向上冒泡。

上下文管理器的具体工作方式参见下面的例子。在这个示例中，我们在 with 块之外使用 LookingGlass 类，因此可以手动调用 `__enter__` 和 `__exit__` 方法：

```python
In [8]: manager = LookingGlass()

In [9]: monster = manager.__enter__()

In [10]: monster == 'JABBERWOCKY'
Out[10]: eurT

In [11]: monster
Out[11]: 'YKCOWREBBAJ'

In [13]: manager.__exit__(None, None, None)

In [14]: monster
Out[14]: 'JABBERWOCKY'
```

## 15.3　contextlib 模块中的实用工具

### closing

如果对象提供了 `close()` 方法，但没有实现 `__enter__`/`__exit__` 协议，那么可以使用这个函数构建上下文管理器。

```python
from contextlib import closing
from urllib.request import urlopen

with closing(urlopen('http://www.python.org')) as page:
    for line in page:
        print(line)
```

### suppress

构建临时忽略指定异常的上下文管理器。

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove('somefile.tmp')

with suppress(FileNotFoundError):
    os.remove('someotherfile.tmp')
```

上面的代码和下面的代码是等价的：

```python
try:
    os.remove('somefile.tmp')
except FileNotFoundError:
    pass

try:
    os.remove('someotherfile.tmp')
except FileNotFoundError:
    pass

```

### @contextmanager

这个装饰器把简单的生成器函数变成上下文管理器，这样就不用创建类去实现管理器协议了。

```python
from contextlib import contextmanager

@contextmanager
def managed_resource(*args, **kwds):
    # Code to acquire resource, e.g.:
    resource = acquire_resource(*args, **kwds)
    try:
        yield resource
    finally:
        # Code to release resource, e.g.:
        release_resource(resource)

>>> with managed_resource(timeout=3600) as resource:
...     # Resource is released at the end of this block,
...     # even if code in the block raises an exception
```

### ContextDecorator

这是个基类，用于定义基于类的上下文管理器。这种上下文管理器也能用于装饰函数，在受管理的上下文中运行整个函数。

```python
from contextlib import ContextDecorator

class mycontext(ContextDecorator):
    def __enter__(self):
        print('Starting')
        return self

    def __exit__(self, *exc):
        print('Finishing')
        return False

>>> @mycontext()
... def function():
...     print('The bit in the middle')
...
>>> function()
Starting
The bit in the middle
Finishing

>>> with mycontext():
...     print('The bit in the middle')
...
Starting
The bit in the middle
Finishing
```

### ExitStack

这个上下文管理器能进入多个上下文管理器。with 块结束时，ExitStack 按照后进先出的顺序调用栈中各个上下文管理器的 `__exit__` 方法。如果事先不知道 with 块要进入多少个上下文管理器，可以使用这个类。例如，同时打开任意一个文件列表中的所有文件。

```python
with ExitStack() as stack:
    files = [stack.enter_context(open(fname)) for fname in filenames]
    # All opened files will automatically be closed at the end of
    # the with statement, even if attempts to open files later
    # in the list raise an exception
```

## 15.4　使用 @contextmanager

其实，`contextlib.contextmanager` 装饰器会把函数包装成实现 `__enter__` 和 `__exit__` 方法的类。

这个类的 `__enter__` 方法有如下作用：

1. 调用生成器函数，保存生成器对象（这里把它称为 gen）。
2. 调用 `next(gen)`，执行到 yield 关键字所在的位置。
3. 返回 `next(gen)` 产出的值，以便把产出的值绑定到 with/as 语句中的目标变量上。

with 块终止时，`__exit__` 方法会做一下几件事：

1. 检查有没有把异常传给 `exc_type`；如果有，调用 `gen.throw(exception)`，在生成器函数定义体中包含 yield 关键字的那一行抛出异常。
2. 否则，用 `next(gen)`，继续执行生成器函数定义体中 yield 语句之后的代码。

> 使用 @contextmanager 装饰器时，要把 yield 语句放在 try/finally 语句中（或者放在 with 语句中），这是无法避免的，因为我们永远不知道上下文管理器的用户会在 with 块中做什么。

