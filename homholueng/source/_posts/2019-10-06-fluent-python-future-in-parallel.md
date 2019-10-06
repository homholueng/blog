---
title: Chapter 17 - 使用期物处理并发
tags:
  - Python
categories:
  - Fluent Python
date: 2019-10-06 17:15:18
---


### 17.1.2 concurrent.futures 模块

`concurrent.futures` 模块的主要特色是 ThreadPoolExecutor 和 ProcessPoolExecutor 类，这两个类实现的接口能分别在不同的线程或进程中执行可调用的对象。这两个类在内部维护着一个工作线程或进程池，以及要执行的任务队列。


```python
from concurrent import futures

with futures.ThredPoolExecutor(worker_count) as executor:
    res = executor.map(download, address_list)
```

在上面代码展示的例子中，`executor.__exist__` 方法糊调用 `executor.shutdown(wait=True)` 方法，它会在所有线程都执行完毕前阻塞主线程。

### 17.1.3 期物

从 Python 3.4 起，标准库中有两个名为 Future 的类：`concurrent.futures.Future` 和 `asyncio.Future`。这两个类的作用相同：两个 Future 类的实例都表示可能已经完成或者尚未完成的 延迟计算。这与 Twisted 引擎中的 Deferred 类、Tornado 框架中的 Future 类，以及多个 JavaScript 库中的 Promise 对象类似。

期物封装待完成的操作，可以放入队列，完成的状态可以查询，得到结果（或抛出异常）后可以获取结果（或异常）。

我们要记住一件事：**通常情况下自己不应该创建期物，而只能由并发框架（concurrent.futures 或 asyncio）实例化。原因很简单：期物表示终将发生的事情，而确定某件事会发生的唯一方式是执行的时间已经排定。**

上述两种期物所拥有的下列方法比较在日常使用中会经常用到：
- `.done()`：这个方法不阻塞，返回值是布尔值，指明期物链接的可调用对象是否已经执行。
- `.add_done_callback()`：这个方法只有一个参数，类型是可调用的对象，期物运行结束后会调用指定的可调用对象。
- `.result()`：在期物运行结束后调用的话，这个方法 在两个 Future 类中的作用相同：返回可调用对象的结果，或者重新抛出执行可调用的对象时抛出的异常。可是，如果期物没有运行结束，result 方法在两个 Future 类中的行为相差很大：
  - `concurrency.futures.Future`：result 的调用会阻塞调用方所在的线程，直到有结果可返回。此时，result 方法可以接收可选的 timeout 参数，如果在指定的时间内期物没有运行完毕，会抛出 TimeoutError 异常。
  - `asyncio.Future`：result 方法不支持设定超时时间，在这个库中获取期物的结果最好使用 `yield from` 结构。

## 17.2　阻塞型I/O和GIL

CPython 解释器本身就不是线程安全的，因此有全局解释器锁（GIL）， 一次只允许使用一个线程执行 Python 字节码。因此，一个 Python 进程通常不能同时使用多个 CPU 核心。

然而，标准库中所有执行阻塞型 I/O 操作的函数，在等待操作系统返回结果时都会释放 GIL。这意味着在 Python 语言这个层次上可以使用多线程，而 I/O 密集型 Python 程序能从中受益：一个 Python 线程等待网络响应时，阻塞型 I/O 函数会释放 GIL，再运行一个线程。

## 17.3 使用 concurrent.futures 模块启动进程

ProcessPoolExecutor 和 ThreadPoolExecutor 类都实现了通用的 Executor 接口，如果需要做 CPU 密集型处理，使用这个模块能绕开 GIL，利用所有可用的 CPU 核心。

> 如果使用 Python 处理 CPU 密集型工作，应该试试 PyPy（http://pypy.org）。

## 17.4 实验 Executor.map 方法

```python

from time import sleep, strftime
from concurrent import futures

def display(*args):
    print(strftime('[%H:%M:%S]'), end=' ')
    print(*args)

def loiter(n):
    msg = '{}loiter({}): doing nothing for {}s...'
    display(msg.format('\t' * n, n, n))
    sleep(n)
    msg = '{}loiter({}): done.'
    display(msg.format('\t' * n, n))
    return n * 10

def main():
    display('Script starting.')
    executor = futures.ThreadPoolExecutor(max_workers=3)
    results = executor.map(loiter, range(5))
    display('results:', results)
    display('Waiting for individual results:')
    for i, result in enumerate(results):
        display('result {}: {}'.format(i, result))

main()

```

上述程序输出如下：

```
[16:57:09] Script starting.
[16:57:09] loiter(0): doing nothing for 0s...
[16:57:09] loiter(0): done.
[16:57:09]      loiter(1): doing nothing for 1s...
[16:57:09]              loiter(2): doing nothing for 2s...
[16:57:09] results: <generator object Executor.map.<locals>.result_iterator at 0x10d21f258>
[16:57:09]                      loiter(3): doing nothing for 3s...
[16:57:09] Waiting for individual results:
[16:57:09] result 0: 0
[16:57:10]      loiter(1): done.
[16:57:10]                              loiter(4): doing nothing for 4s...
[16:57:10] result 1: 10
[16:57:11]              loiter(2): done.
[16:57:11] result 2: 20
[16:57:12]                      loiter(3): done.
[16:57:12] result 3: 30
[16:57:14]                              loiter(4): done.
[16:57:14] result 4: 40
```

Executor.map 函数易于使用，不过有个特性需要注意：**这个函数返回结果的顺序与调用开始的顺序一致。如果第一个调用生成结果用时 10 秒，而其他调用只用 1 秒，在获取 map 方法返回的生成器产出的第一个结果时代码会阻塞 10 秒。在此之后，获取后续结果时不会阻塞，因为后续的调用已经结束。**

> `executor.submit` 和 `futures.as_completed` 这个组合比 `executor.map` 更灵活，因为 submit 方法能处理不同的可调用对 象和参数，而 `executor.map` 只能处理参数不同的同一个可调用对象。此外，传给 `futures.as_completed` 函数的期物集合可以来 自多个 Executor 实例

## 17.5　显示下载进度并处理错误

使用 TQDM 包 （https://github.com/noamraph/tqdm）可以实现的文本动画进度条。


```python
import time
from tqdm import tqdm

for i in tqdm(range(1000)):
    time.sleep(.01)
```

随后，进度条会显示在下方：

```
 49%|███████████████████████████████████████████████████████████████▊                                                                  | 491/1000 [00:05<00:05, 87.69it/s]
 ```

 