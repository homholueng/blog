---
title: python asyncio High-level APIs
tags: 
  - Python
categories:
  - Fluent Python
---

asyncio 是 Python 官方提供的用于编写并发程序的库，特别是在 Python 3.6 之后，官方通过添加 `async/await` 关键字来支持了原生的协程，这使得我们不需要再使用传统的生成器来编写协程，而在 Python 3.7 之后，官方对 asyncio 库进行了调整，提供了更为抽象的高层接口，使得这个库的易用程度大大提高，通过这些上层接口，我们能够：

- 并发的执行 Python 协程，病能够很好的对其进行管理
- 进行网络 IO 和进程间通信
- 控制子进程
- 通过队列来执行分布式任务
- 同步并发代码

## Croutines and Tasks

在 Python 3.6 之后，用 `async/await` 修饰的函数我们称之为协程，如下所示：

```python
import asyncio
async def main():
    print('hello')
    await asyncio.sleep(1)
    print('world')
```

上面的程序在打印了 hello 之后，会调用 `asyncio.sleep` 函数睡眠一秒，此时 main 会将执行权交出去，执行流程会回到当前运行循环中，让下一个准备好的协程执行。**需要注意的是，一旦我们的程序决定使用协程来实现，我们就不应该在协程中调用会阻塞当前线程的函数，不然会阻塞整个运行循环，就失去了并发程序间互相协作的意义，这就是为什么我们在这里低啊用 `asyncio.sleep` 而不是 `time.sleep` 的原因，前者会将控制权交还给运行循环。**

### Run a coroutine

我们有三种方式来执行一个协程：

1. `asyncio.run()`，一般通过这个函数来运行最顶层的协程
2. `await` 关键字，通过这个关键字我们能够在协程中调用其他的协程
3. `asyncio.create_task()`，使用这个函数能够以 task 的形式执行多个协程，使用这个方式来执行协程的意义在于，我们能够获得一个 `Task` 对象，通过这个对象我们能够实现对协程的一些控制行为

```python
import asyncio
import time

async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)

async def main():
    print(f"started at {time.strftime('%X')}")

    await say_after(1, 'hello')
    await say_after(2, 'world')

    print(f"finished at {time.strftime('%X')}")

asyncio.run(main())
```

上述代码中的 main 协程也可以替换成下面这种实现方式：

```python
async def main():
    task1 = asyncio.create_task(
        say_after(1, 'hello'))

    task2 = asyncio.create_task(
        say_after(2, 'world'))

    print(f"started at {time.strftime('%X')}")

    # Wait until both tasks are completed (should take
    # around 2 seconds.)
    await task1
    await task2

    print(f"finished at {time.strftime('%X')}")
```

### Awaitables

如果一个对象能够对其使用 `await` 关键字，那么该对象就是一个 `awaitable` 的对象。在 asyncio 中，主要有下面三种 awaitable 对象：

1. croutines：使用 `async def` 关键字定义的函数或是调用协程函数返回的对象
2. tasks：调用 `asyncio.create_task` 返回的 Task 对象，通常使用 Task 对象来同时调度多个协程
3. futures：这是一个底层实现中使用的对象，其表示一个异步操作最终会产生的结果；一般来说，使用高层 API 时我们都不会接触到这种对象

## Running Tasks Concurrently

如果要同时执行多个协程，asyncio 提供了一个便捷函数 gather 供我们使用：`awaitable asyncio.gather(*aws, return_exceptions=False)`。其会按照 aws 中传入的协程的顺序来并发的执行他们，如果 aws 中传入的对象是 awaitable 的，那么 gather 就会将其作为一个 Task 对象进行调度。

```python
import asyncio

async def factorial(name, number):
    f = 1
    for i in range(2, number + 1):
        print(f"Task {name}: Compute factorial({i})...")
        await asyncio.sleep(1)
        f *= i
    print(f"Task {name}: factorial({number}) = {f}")

async def main():
    # Schedule three calls *concurrently*:
    await asyncio.gather(
        factorial("A", 2),
        factorial("B", 3),
        factorial("C", 4),
    )

asyncio.run(main())

# Expected output:
#
#     Task A: Compute factorial(2)...
#     Task B: Compute factorial(2)...
#     Task C: Compute factorial(2)...
#     Task A: factorial(2) = 2
#     Task B: Compute factorial(3)...
#     Task C: Compute factorial(3)...
#     Task B: factorial(3) = 6
#     Task C: Compute factorial(4)...
#     Task C: factorial(4) = 24
```

如果 `return_exceptions` 参数为 `True`，则协程中抛出的异常会被当做结果返回，否则 gather 函数会抛出异常。

```python
simport asyncio


async def coro1():
    raise Exception()


async def coro_n(num):
    print('coro', num)
    await asyncio.sleep(num)
    return num


async def main():
    results = await asyncio.gather(coro1(), coro_n(2), coro_n(3), return_exceptions=True)
    print(results)

asyncio.run(main())

# Expected output:
# coro 2
# coro 3
# [Exception(), 2, 3]
```

如果 `return_exceptions` 为 `False`，则会抛出异常：

```
<ipython-input-3-cf6ac787bf10> in main()
     13 
     14 async def main():
---> 15     results = await asyncio.gather(coro1(), coro_n(2), coro_n(3), return_exceptions=False)
     16     print(results)
     17 

<ipython-input-3-cf6ac787bf10> in coro1()
      3 
      4 async def coro1():
----> 5     raise Exception()
      6 
      7 
```

## Shielding From Cancellation

我们可以使用 `shield` 函数来防止一个协程被 `calcel()` 调用影响：

```python
res = await shield(something())
```

但是如果正在执行 `something` 的协程被取消了，虽然此时 `something` 本身没有被取消，但是这条 `await` 语句还是会抛出 `CancelledError`。

## Timeouts

`asyncio.wait_for(aw, timeout, *)` 等待 `aw` 协程在超时时间内完成，否则抛出 `TimeoutError` 异常：

```python
async def eternity():
    # Sleep for one hour
    await asyncio.sleep(3600)
    print('yay!')

async def main():
    # Wait for at most 1 second
    try:
        await asyncio.wait_for(eternity(), timeout=1.0)
    except asyncio.TimeoutError:
        print('timeout!')

asyncio.run(main())
```

## Waiting Primitives

`asyncio.wait(aws, *, timeout=None, return_when=ALL_COMPLETED)` 会执行 `aws` 并阻塞到 `return_when` 参数中指定的条件满足位置。

```python
done, pending = await asyncio.wait(aws)
```

`return_when` 有以下选项：

- `FIRST_COMPLETED`：函数将在任意一个 future 完成或被取消后返回。
- `FIRST_EXCEPTION`：函数将在任意一个 future 抛出异常后返回，如果没有任何异常抛出，其等同于 `ALL_COMPLETED`。
- `ALL_COMPLETED`：函数将在所有 future 完成或被取消后返回。

## Scheduling From Other Threads

`asyncio.run_coroutine_threadsafe(coro, loop)` 能够让我们在另一个线程中执行协程：

```python
# Create a coroutine
coro = asyncio.sleep(1, result=3)

# Submit the coroutine to a given loop
future = asyncio.run_coroutine_threadsafe(coro, loop)

# Wait for the result with an optional timeout argument
assert future.result(timeout) == 3
```

## Introspection

- `asyncio.current_task(loop=None)`：返回当前正在执行的 Task 实例，如果当前没有任务在执行则返回 `None`
- `asyncio.all_tasks(loop=None)`：返回 loop 中尚未执行完成的 Task 集合

如果 `loop` 参数为 `None`，函数内部会使用 `get_running_loop()` 来获取当前运行循环。

