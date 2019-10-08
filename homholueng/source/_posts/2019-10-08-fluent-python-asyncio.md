---
title: Chapter 18 - 使用 asyncio 包处理并发
tags:
  - Python
categories:
  - Fluent Python
date: 2019-10-08 21:23:35
---


> 并发是指同时处理多件事
> 
> 并行是指同时做多件事
> 
> 二者不同，但是有联系
> 
> 一个关于结构，一个关于执行
> 
> 并发用于指定方案，用来解决可能（但未必）并行的问题
> 
> —— Rob Pike（Go 语言创造者之一）


## 18.1 线程与协程对比

在编写使用线程来编写程序时，我们需要使用锁来保护程序中重要的部分。而协程默认会做好全方位保护，以防止中断。我们必须显式产出才能让程序的余下部分运行。对协程来说，无需保留锁，在多个线程之间同步操作，协程自身就会同步，因为在任意时刻只有一个协程运行。

### 18.1.1　asyncio.Future：故意不阻塞

`asyncio.Future` 类的 `.result()` 方法没有参数，因此不能指定超时时间。此外，如果调用 `.result()` 方法时期物还没运行完毕，那么 `.result()` 方法不会阻塞去等待结果，而是抛出 asyncio.InvalidStateError` 异常。

然而，获取 `asyncio.Future` 对象的结果通常使用 yield from，从中产出结果。使用 `yield from` 处理期物，等待期物运行完毕这一步无需我们关心，而且不会阻塞事件循环，因为在 asyncio 包中，`yield from` 的作用是把控制权还给事件循环。

### 18.1.2　从期物、任务和协程中产出

在 asyncio 包中，期物和协程关系紧密，因为可以使用 `yield from` 从 `asyncio.Future` 对象中产出结果。这意味着，如果 foo 是协程函数（调用后返回协程对象），抑或是返回 Future 或 Task 实例的普通函数，那么可以这样写：`res = yield from foo()`。这是 asyncio 包的 API 中很多地方可以互换协程与期物的原因之一。

为了执行这些操作，必须排定协程的运行时间，然后使用 `asyncio.Task` 对象包装协程。对协程来说，获取 Task 对象有两种主要方式：

- `asyncio.async(coro_or_future, *, loop=None)`：这个函数统一了协程和期物：第一个参数可以是二者中的任何一个。如果是 Future 或 Task 对象，那就原封不动地返回。如果是协程，那么 async 函数会调用 `loop.create_task(...)` 方法创建 Task 对象。`loop=` 关键字参数是可选的，用于传入事件循环；如果没有传入，那么 async 函数会通过调用 `asyncio.get_event_loop()` 函数获取循环对象。
- `BaseEventLoop.create_task(coro)`：这个方法排定协程的执行时间，返回一个 `asyncio.Task` 对象。

在 asyncio 包的文档中，“18.5.3. Tasks and coroutines”一节(https://docs.python.org/3/library/asyncio-task.html)说明了协程、期物和任务之间的关系。

## 18.2　使用 asyncio 和 aiohttp 包下载

`asyncio.wait(...)` 协程的参数是一个由期物或协程构成的可迭代对象；wait 会分别把各个协程包装进一个 Task 对象。最终的结果是，wait 处理的所有对象都通过某种方式变成 Future 类的实例。wait 是协程函数，因此返回的是一个协程或生成器对象。

`wait_coro` 运行结束后返回一个元组，第一个元素是一系列结束的期物，第二个元素是一系列未结束的期物。wait 函数有两个关键字参数，如果设定了可能会返回未结束的期物；这两个参数是 `timeout` 和 `return_when`。详情参见 `asyncio.wait` 函数的文档(https://docs.python.org/3/library/asyncio-task.html#asyncio.wait)。


使用 asyncio 包时，我们编写的异步代码中包含由 asyncio 本身驱动的协程（即委派生成器），而生成器最终把职责委托给 asyncio 包或第三方库（如 aiohttp）中的协程。这种处理方式相当于架起了管道，让 asyncio 事件循环（通过我们编写的协程）驱动 执行低层异步 I/O 操作的库函数。

## 18.3　避免阻塞型调用

有两种方法能避免阻塞型调用中止整个应用程序的进程：

- 在单独的线程中运行各个阻塞型操作

- 把每个阻塞型操作转换成非阻塞的异步调用使用

多个线程是可以的，但是各个操作系统线程（Python 使用的是这种线程）消耗的内存达兆字节（具体的量取决于操作系统种类）。如果要处理几千个连接，而每个连接都使用一个线程的话，我们负担不起。

为了降低内存的消耗，通常使用回调来实现异步调用。这是一种底层概念，类似于所有并发机制中最古老、最原始的那种——硬件中断。使用回调时，我们不等待响应，而是注册一个函数，在发生某件事时调用。这样，所有调用都是非阻塞的。

## 18.4　改进 asyncio 下载脚本

`asyncio.Semaphore` 对象维护着一个内部计数器，若在对象上调用 `.acquire()` 协程方法，计数器则递减；若在对象上调用 `.release()` 协程方法，计数器则递增。计数器的初始值在实例化 Semaphore 时设定：

```python
semaphore = asyncio.Semaphore(concur_req)
```

如果计数器大于零，那么调用 `.acquire()` 方法不会阻塞；可是，如果计数器为零，那么 `.acquire()` 方法会阻塞调用这个方法的协程，直到其他协程在同一个 Semaphore 对象上调用 `.release()` 方法，让计数器递增。

使用 `with` 语句能够完成上述两种操作：

```python
with (yield from semaphore):
    ...
```

### 18.4.2　使用Executor对象， 防止阻塞事件循环

asyncio 的事件循环在背后维护着一个 ThreadPoolExecutor 对象，我们可以调用 `run_in_executor` 方法，把可调用的对象发给它执行，避免某些方法的调用阻塞整个事件循环。

```python
loop = asyncio.get_event_loop()
loop.run_in_executor(None, io_function, *args, **kwargs) # 第一个参数是 Executor 实例；如果设为 None，使用事件循环的默认 ThreadPoolExecutor 实例
```

## 18.5　从回调到期物和协程

Python 中的回调地狱：

```python
def stage1(response1): 
    request2 = step1(response1) 
    api_call2(request2, stage2)

def stage2(response2): 
    request3 = step2(response2) 
    api_call3(request3, stage3)

def stage3(response3): 
    step3(response3)

api_call1(request1, stage1)
```

上述例子中组织代码的方式导致代码难以阅读，也更难编写：每个函数做一部分工作，设置下一个回调，然后返回，让事件循环继续运行。这样，所有本地的上下文都会丢失。执行下一个回调时（例如 stage2），就无法获取 request2 的值。如果需要那个值，那就必须依靠闭包，或者把它存储在外部数据结构中，以便在处理过程的不同 阶段使用。

在这个问题上，协程能发挥很大的作用。在协程中，如果要连续执行 3 个异步操作，只需使用 `yield` 3 次，让事件循环继续运行。3 次操作都在同一个函数定义体中，像是顺序代码，能在处理过程中使用局部变量保留整个任务的上下文：

```python
@asyncio.coroutine 
def three_stages(request1): 
    response1 = yield from api_call1(request1) 
    # 第一步 
    request2 = step1(response1) 
    response2 = yield from api_call2(request2) 
    # 第二步 
    request3 = step2(response2) 
    response3 = yield from api_call3(request3) 
    # 第三步 
    step3(response3)

loop.create_task(three_stages(request1))

# 必须显式调度执行
```

如果这时候还需要为每一步可能发生的错误编写处理逻辑，那么第二种方式带来的好处就更为明显。传统的回调方式需要传入错误处理函数，导致代码阅读难度更大；而使用协程，我们只需要简单的使用 `try ··· except` 块来处理即可。

这么做比陷入回调地狱好多了，但是我不会把这种方式称为协程天堂，毕竟我们还要付出代价。我们不能使用常规的函数，必须使用协程，而且要习惯 `yield from`。另外，我们必 须使用事件循环显式排定协程的执行时间，或者在其他排定了执行时间的协程中使用 `yield from` 表达式把它激活。

