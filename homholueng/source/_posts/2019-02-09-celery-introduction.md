---
title: Celery 使用指南
date: 2019-02-09 16:03:44
categories: 
- Python Daily
tags:
- Celery
- Python
---

## 概述

Celery 是 Python 生态中一个比较有名的分布式任务队列，其具有轻量，灵活及可靠等特性。事实证明，在生产环境下，Celery 也同样值得信赖，有效的使用 Celery 能够帮助我们事半功倍的完成各种工作；另外，深入了解 Celery 的一些特性能够帮助开发者更好的使用它以及避免重复造轮子。本文会从 Celery 的若干个核心功能点出发，讲解 Celery 在日常使用中的一些基本特性及某些不常使用的深度功能，帮助开发者对 Celery 有更加全面和深入的了解。

由于 Celery 在系统中的定位是任务队列，所以其只提供了任务、Worker 的定义及一些周边的功能。Celery 本身需要依赖消息队列来进行消息的分发，即 Broker。本文中的例子所使用的语言及相关组件的版本如下：

- Python：3.6.1
- Celery：4.2.0
- Rabbitmq：3.7.8



## 基础对象 —— Application

> The Celery library must be instantiated before use, this instance is called an application (or *app* for short).

虽说 Celery 是一个开箱即用的框架，但是在使用它之前，我们还是需要做一些基础的配置及初始化的工作。首先，我们要先认识一下整个 Celery 库中的基础对象：**Celery Application**（后文简称 app）。当我们完成 app 的初始化后，Celery 就算是进入一个完全可用的状态了。可以将 app 比作整个 Celery 框架的心脏，整个框架都会围绕 app 对象来运行。



### 让心脏开始跳动 —— 创建一个 APP

其实创建一个 app 十分简单：

```python
from celery import Celery

app = Celery()
app  # <Celery __main__ at 0x103e93c50>
```

只需要短短两句代码，就能够完成了一个主模块名为 `__main__` 的 app 的创建。在开始下面的内容之前，需要了解一个十分重要的概念：**主模块名**。当 Celery 无法决定某一个任务定义函数属于哪一个模块时，其会使用 app 的主模块名作为该函数所属的模块名，如下所示：

```python
from celery import Celery

app = Celery()

@app.task
def a_task():
    return

a_task  # <@task: __main__.a_task of __main__ at 0x11038be48> 任务名为 __main__.a_task
```

当然，我们能够在初始化 app 的时候手动配置主模块名：

```python
from celery import Celery

app = Celery('main_module_name')

@app.task
def a_task():
    return

a_task  # <@task: main_module_name.a_task of main_module_name at 0x110939550> 此时任务的名称变成了 main_module_name.a_task
```

通过上面的两个例子可以看到，Celery 为了降低使用成本，在初始化 app 时会将许多配置设置为默认值。如果 Celery 设置的默认值不符合你的需求，那么你可以通过 app 对象来修改这些配置。



### 使用 app 配置 Celery



通过改变 app 的配置，能够改变 Celery 运行的行为模式。Celery 提供了多种手段来让开发者进行 app 的配置，包括：

- 直接修改 `app.conf` 中的值
- 通过 config 对象进行配置
- 通过环境变量进行配置



举个例子，假设我们要让 Celery 在 `Asia/Shanghai` 时区下工作，并且使用 `broker-host.com` 提供的 Broker，那么我们可以使用以下几种方法来进行配置：



#### 1. 直接修改 `app.conf` 中的值

```python
from celery import Celery

app = Celery()

app.conf.timezone = 'Asia/Shanghai'
app.conf.enable_utc = True
app.conf.broker_url = 'amqp://guest@broker-host.com//'
```



#### 2. 通过 config 对象进行配置

此处的 config 对象可以是一个定义了配置属性的模块，也可以是一个拥有配置属性的对象。如果你喜欢将配置集中放在某个模块下，可以这样进行配置：

```python
# ./conf.py

timezone = 'Asia/Shanghai'
enable_utc = True
broker_url = 'amqp://guest@broker-host.com//'

# ./app.py

from celery import Celery

import conf

app = Celery()
app.config_from_object(conf)
```

或者你也可以选择使用类来管理这些配置：

```python
from celery import Celery

app = Celery()

class Config:
    timezone = 'Asia/Shanghai'
    enable_utc = True
    broker_url = 'amqp://guest@broker-host.com//'

app.config_from_object(Config)
```



#### 3. 通过环境变量进行配置

如果我们能够预置好若干个配置模块，然后在 Celery 启动的时候再根据当前的环境选择特定的配置模块，就会让我们的代码更加的灵活。好在 Celery 允许开发者通过环境变量来传递配置模块名，使得我们得以实现这一功能：

```python
import os
from celery import Celery

# 设置默认的配置模块名
os.environ.setdefault('CELERY_CONFIG_MODULE', 'celeryconfig')

app = Celery()
app.config_from_envvar('CELERY_CONFIG_MODULE')
```

上述的若干个例子中仅仅展示了 Celery 配置选项中的冰山一角，如果想要了解完整的配置选项，请移步[配置手册](https://celery.readthedocs.io/en/latest/userguide/configuration.html#configuration)。



## 任务队列中的任务 —— Task

如果说 app 是 Celery 的心脏，那么任务就是 Celery 的灵魂。在 Celery 中，任务能过通过任意可调用对象创建出来，如下所示：

```python
from celery import Celery

app = Celery()

@app.task()
def add(x, y):
    return x + y
```

这里要引入一个绑定任务（bound task）的概念，如果我们将某个可调用对象设定为绑定任务，那么在任务执行时，我们就能够拿到当前任务对象（Task）：

```python
from celery import Celery

app = Celery()

@task(bind=True)
def add(self, x, y):
    return x + y
```

当某个可调用对象被设置成绑定任务后，该对象被调用后第一个参数一定是当前被执行的任务对象实例（Task），与 Python 中的绑定方法（bound method）相似。在该任务实例中，我们能够拿到一些比较有用的信息：

```python
from celery import Celery

app = Celery()

@task(bind=True)
def add(self, x, y):
    logger.info(self.request.id)  # 获取该次任务的唯一 ID
    logger.info(self.request.args)  # 获取该次任务请求的位置参数
    logger.info(self.request.kwargs)  # 获取该次任务请求的关键字参数
    logger.info(self.acks_late)  # 该任务是否是延迟确认任务
    logger.info(self.backend)  # 该任务的结果存储后端
    ...
```



### 任务的命名

> Every task must have a unique name.



为什么每个任务都需要拥有一个唯一的名字呢？因为 Celery 内部会通过 `name -> task` 的映射关系来维护用户注册的任务。当我们向 Celery 发起一个执行任务的请求时，Celery 并不会将整段任务定义的代码放到队列中，而是将任务的名字放入执行请求中，当 worker 收到该请求后，则会根据任务名字来找到任务的定义。所以任务的名字必须是唯一的。

但是在上述的一些例子中，我们在定义任务时并没有显示的为任务设置名称，那么 Celery 是如何找到这些任务的呢？

在我们没有显式的设置任务名称时，Celery 会为我们定义的任务设置默认名，这个默认名会根据**任务函数所在的模块**及**任务函数的命名**来确定。也就是说，如果我们在 `tasks.py` 文件下定义了一个名为 `add` 的任务，那么 Celery 默认会将其名字设置为 `tasks.add`：

```python
# tasks.py

@app.task
def add(x, y):
    return x + y

print(add.name)  # 'tasks.add'
```

当然，你也可以在定义任务时显式的为这些任务命名：

```python
# tasks.py

@app.task(name='task.tasks.add')
def add(x, y):
    return x + y

print(add.name)  # 'task.tasks.add'
```



#### 请在使用自动命名时确保一致的导入方式

Celery 为任务设置默认名的行为的确方便了开发者，但是也带来了一些潜在的问题，假如我们有下面这样的一个目录结构：

```
task
├── __init__.py
└── tasks.py
```

然后在 `task/tasks.py` 文件中定义了这样的一个任务：

```python
@app.task
def add(x, y):
    return x + y
```

如果我们尝试在不同的目录下导入该任务，就会出现任务名不一致的情况：

```python
>>> from tasks import add
>>> add.name
'tasks.add'

>>> from task.tasks import add
>>> add.name
'task.tasks.add'
```

也就是说，如果 worker 与主程序导入任务的方式不同，那么在主程序发送执行任务的请求时 worker 就会抛出 `NotRegistered` 错误，因为主程序看到的任务名与 worker 看到的任务名是不同的。除了保持一致的任务导入方式外，我们还可以通过显式的为任务命名来规避这个问题，当我们显式的为某个任务设置名称后，该任务的名字就不会因导入的方式变化而变化了：

```python
@app.task(name='task.tasks.add')
def add(x, y):
    return x + y

>>> from tasks import add
>>> add.name
'task.tasks.add'

>>> from task.tasks import add
>>> add.name
'task.tasks.add'
```



### 任务注册表

在上面一个小节中，我们不止一次提到任务注册表这个概念，其实在 Celery 中，要获取任务注册表非常简单，app 对象中的 `tasks` 属性就是 Celery 的任务注册表：

```python
>>> from proj.celery import app
>>> app.tasks
{'celery.chord_unlock':
    <@task: celery.chord_unlock>,
 'celery.backend_cleanup':
    <@task: celery.backend_cleanup>,
 'celery.chord':
    <@task: celery.chord>}
```

需要注意的是，只有当定义任务的模块在代码中被导入了，这些任务才会出现在注册表中。所以，Celery worker 和主程序中都维护了这样的一个字典，并通过唯一的任务名来映射到具体的任务。



## Make some noise —— Calling Tasks

任务队列最迷人的地方不是它对任务的定义，而是其执行任务的能力。在我们定义好各种各样的任务之后，我们还需要去调用他们，发挥其真正的价值。但是在 Celery 中，调用任务的姿势（方法）也是一门小学问。比如对于一个简单的 `add` 任务，就能够通过以下几种方式去执行：

```python
from tasks import add

add.delay(1, 2)
add.apply_async(args=(1, 2))
add.apply_async((1, 2), countdown=10)
add(1, 2)
```

看似复杂，但其实 Celery 调用任务的方式能够总结为以下三种：

1. `apply_async(args[, kwargs[, …]])`：调用任务的标准 API，调用者能够按需传入各种执行选项，上述例子中的 `add.apply_async((1, 2), countdown=10)` 调用即传入了 `countdown` 选项，这使得该任务会在 10 秒后被执行。
2. `delay(*args, **kwargs)`：`apply_async` 的捷径方法，不支持传入执行选项。
3. 直接调用（`__call__`）：直接调用该任务，如上述代码中的 `add(1, 2)`。若使用这种调用方式，则任务不会被发送到 Worker 中，而是在当前线程中执行。



当我们不需要使用执行选项时，笔者推荐各位尽量使用 `delay` 来执行任务，使用 `delay` 会让发起任务的代码看起来更加自然：

```python
task.delay(arg1, arg2, kwarg1='x', kwarg2='y')  # 使用 delay
task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})  # 使用 apply_async
```



### 执行选项

当我们对任务的某次执行有一些特殊的要求时，就需要用到 `apply_async` 接口提供的各种执行选项了，例如你希望某个任务在 30 秒后再开始执行：

```python
from datetime import datetime, timedelta

# 以下两个调用是等价的
add.apply_async((2, 2), countdown=30)
add.apply_async((2, 2), eta=datetime.utcnow() + timedelta(seconds=30))
```

或者是你希望为某个任务设置一个过期时间（当 worker 收到一个过期的任务时，会放弃执行该任务）：

```python
from datetime import datetime, timedelta

# 以下两个调用是等价的
add.apply_async((10, 10), expires=5)
add.apply_async((10, 10), expires=datetime.utcnow() + timedelta(seconds=5))
```

该方法支持的执行选项还有很多，就不在此一一赘述，感兴趣的读者可以到 [API 手册](https://celery.readthedocs.io/en/latest/reference/celery.app.task.html#celery.app.task.Task.apply_async)中了解。笔者推荐各位了解一下各个执行选项的作用，这样能够使你在遇到一些新的需求时能够更加快速的找到解决方案，避免重复造轮子。



## 周期任务 —— Periodic Tasks



Celery 自带了一个名为 **celery beat** 的调度器，这个调度器能够让我们周期性的执行一些我们预先定义好的任务。



### 注册周期任务

只要将任务加入到调度列表中，调度器就能够周期性的执行这些任务：

```python
from celery import Celery
from celery.schedules import crontab

app = Celery()

@app.on_after_configure.connect
def setup_periodic_tasks(sender, **kwargs):
    # 每 10s 调用一次 test('hello')
    sender.add_periodic_task(10.0, test.s('hello'), name='add every 10')

    # 每 30s 调用一次 test('hello')
    sender.add_periodic_task(30.0, test.s('world'), expires=10)

    # 在每周一的早上 7:30 调用一次 test('Happy Mondays')
    sender.add_periodic_task(
        crontab(hour=7, minute=30, day_of_week=1),
        test.s('Happy Mondays!'),
    )

@app.task
def test(arg):
    print(arg)
```

上述代码中，我们使用 `on_after_configure` 信号在 app 完成配置后执行周期任务设置的函数，并通过 `add_periodic_task` 方法将定义好的任务添加到调度器的调度列表中。我们添加到调度列表的周期任务，其实也维护在 app 内部的一个字典中：

```python
>>> app.conf.beat_schedule
{'add every 10': {'schedule': 10.0,
  'task': 'periodic_tasks.test',
  'args': ('hello',),
  'kwargs': {},
  'options': {}},
 "periodic_tasks.test('world')": {'schedule': 30.0,
  'task': 'periodic_tasks.test',
  'args': ('world',),
  'kwargs': {},
  'options': {'expires': 10}},
 "periodic_tasks.test('Happy Mondays!')": {'schedule': <crontab: 30 7 1 * * (m/h/d/dM/MY)>,
  'task': 'periodic_tasks.test',
  'args': ('Happy Mondays!',),
  'kwargs': {},
  'options': {}}}
```

不难发现，周期任务的注册表也是维护在 `app.conf` 下的，这就意味着，如果你不喜欢通过在代码中调用 API 来注册周期任务，可以选择通过配置的方式来设定周期任务：

```python
# conf.py
from celery.schedules import crontab

beat_schedule = {'add every 10': {'schedule': 10.0, 
'task': 'periodic_tasks.test',
'args': ('hello',),
'kwargs': {},
'options': {}},
"periodic_tasks.test('world')": {'schedule': 30.0,
'task': 'periodic_tasks.test',
'args': ('world',),
'kwargs': {},
'options': {'expires': 10}},
"periodic_tasks.test('Happy Mondays!')": {'schedule': crontab(hour=7, minute=30, day_of_week=1),
'task': 'periodic_tasks.test',
'args': ('Happy Mondays!',),
'kwargs': {},
'options': {}}}

# tasks.py

from celery import Celery

import conf

app = Celery()
app.config_from_object(conf)

@app.task
def test(arg):
    print(arg)
```





### 调度策略



如果我们仅仅希望某个任务在固定的时间间隔后被执行，那么只需要在注册周期任务时传递固定的调度间隔即可：

```python
# 每 10s 调用一次 test('hello')
sender.add_periodic_task(10.0, test.s('hello'), name='add every 10')
```

但在某些场景下， 可能需要更加复杂的调度规则，如在某些天的某个时间点进行调度，这个时候就可以考虑使用 Celery 内置的一些调度策略。其中一个比较常用的调度策略为 `crontab`：

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    # 在每周一的早上 7:30 调用一次 test('Happy Mondays')
    'add-every-monday-morning': {
        'task': 'tasks.add',
        'schedule': crontab(hour=7, minute=30, day_of_week=1),
        'args': (16, 16),
    },
}
```

`crontab` 本身还支持十分灵活的表达式，关于其更详细的用法，在此不做赘述，有兴趣的读者请移步 [crontab API 手册](http://docs.celeryproject.org/en/v4.2.0/reference/celery.schedules.html#celery.schedules.crontab)。另外一个比较有意思的调度策略是 Celery 提供的 [solar schedule](http://docs.celeryproject.org/en/v4.2.0/userguide/periodic-tasks.html#solar-schedules)，其会根据地球上某个地点的日出日落或是黄昏黎明时间来触发任务执行。



### 启动调度器



完成了周期任务的注册和策略的选择后，只需要把调度器启动起来，就大功告成了:

```shell
$ celery -A tasks beat
```



### Django 用户看这里



如果你是 Django 用户，在使用 celery beat 之前不妨先了解一下 [django-celery-beat](https://pypi.python.org/pypi/django-celery-beat/) 这个扩展，其通过扩展 celery beat 的 `Scheduler` 类，提供了以下功能：

- 将周期任务配置持久到数据库中
- 通过 Django admin 来管理周期任务
- 为每个 crontab 策略分配不同的时区



## 职责划分 —— Routing Tasks



在我们没有手动设置任务路由的情况下，Celery 会为我们在 Rabbitmq 创建一个默认队列，所有的任务执行请求都会被发送到该队列中，我们启动的 worker 也只会默认从该队列中获取消息。这种行为在一些较为简单的场景下并不会出现问题，但假设我们的任务平均耗时相差很大，那么一些平均耗时较长的任务很可能会阻塞后面到来的平均耗时较短的任务，导致系统中的任务平均等待时间较长。

这时候可以考虑设置多个队列，将不同类型的任务路由到不同的队列中，每个队列都分配一定数量的 worker 来处理。其实在实际场景下，并不一定是根据任务的耗时来进行分配，也可以选择根据任务的**重要程度，性质（计算密集或 IO 密集），甚至是内部实现细节**来进行任务路由的配置。而换句话说，设置任务的路由实际上就是对 worker 进行职责划分，如果两个 worker 处理的队列不同，那么我们可以认为其在系统中所属的角色不同，即职责不同，通过更细化的配置来更好地利用计算机资源。

下面的例子就展示了如何将两个任务分配到不同的队列中：

```python
from celery import Celery

app = Celery()

app.conf.task_routes = {
    'task.tasks.task_1': {'queue': 'queue_1'},
    'task.tasks.task_2': {'queue': 'queue_2'}
}

@app.task(name='task.tasks.task_1')
def task_1():
    return

@app.task(name='task.tasks.task_2')
def task_2():
    return
```

当然，也可以使用通配符来配置任务的路由：

```python
from celery import Celery

app = Celery()

app.conf.task_routes = {
    'task.tasks.task_*': {'queue': 'queue_1'}
}

@app.task(name='task.tasks.task_1')
def task_1():
    return

@app.task(name='task.tasks.task_2')
def task_2():
    return
```

在完成了任务路由的配置后，我们需要在启动 worker 时指定其监听的队列：

```shell
$ celery worker -A tasks -Q queue_1  # 监听 queue_1
$ celery worker -A tasks -Q queue_1,celery  # 同时监听 queue_1 及 celery
```





## 简单的工作流 —— Work-flow



除了提供单个任务的定义和执行能力之外，Celery 还允许我们将多个任务组合串联起来，形成一整个工作流，在学习如何使用这些能力之前，必须要先了解任务签名这一概念。



### 任务签名

在 Celery 中，**任务签名代表了单个任务的一次调用及相关的参数和执行选项**，换句话说，任务签名表示一次任务调用的行为。任务签名的存在能够使得开发者将任务调用这一行为进行序列化，或在参数中进行传递。创建一个任务签名十分简单：

```python
from celery import signature
from tasks import add

# 以下两种创建方式是等价的
s1 = add.signature((2, 2), countdown=10)
s2 = signature('tasks.add', args=(2, 2), countdown=10)

# 捷径方法，无法设置执行选项
s3 = add.s(2, 2)
s4 = add.s(2, 2).set(countdown=10).set(retry=False)  # 通过 set 方法来设置执行选项
```

既然任务签名代表了一次调用的命令，我们自然能够使用这个命令来完成任务的调用，但是需要注意的是，任务签名中的执行选项会被调用执行函数时传入的执行选项覆盖：

```python
from tasks import add

s = add.s(2, 2)

s()  # 在当前进程调用
s.apply_async()  # 发送到 worker 执行
s.delay()  # 发送到 worker 执行的捷径方式

s_with_option = add.s(2, 2).set(countdown=1)
s_with_option.apply_async(countdown=10)  # 任务会在 10s 后开始执行，而不是 1s 
```

相信大家都知道 Python 中 `functools` 模块下的 `partial` 函数能预置已有函数的某些参数以创建出一个新的函数。而在 Celery 中，任务签名也支持这一特性，在创建任务签名时我们不必将所有的参数都预先设置好，可以预留一些参数在真正执行时再传入：

```python
from tasks import add

partial_s = add.s(2)

# 以下两个调用是等价的
partial_s.delay(4)
partial_s.apply_async((2,))
```

有了这个特性后，我们就能够在任务的连接和编排中将父任务的结果作为参数传递到子任务中。

了解了任务签名这个概念后，让我们来看一个使用任务签名来实现任务回调的简单例子，在这个例子中，我们要通过 `add` 任务来实现 `(4 + 4) + 8` 这一工作，由于 `add` 任务单次只能完成两个操作数的相加，所以我们需要组合两个 `add` 任务来完成这项工作，第一个 `add` 计算 `4 + 4`，并把结果传递给第二个 `add`，第二个 `add` 则负责完成 `8 + 8` 的计算，代码如下：

```python
from celery import Celery

app = Celery()

@app.task
def add(x, y):
    return x + y

s = add.s(8)
add.apply_async((4, 4), link=s)  # 使用 link 选项来指定 add 的回调任务
```

可以看到，通过使用任务签名和回调，我们能够将多个任务串联起来执行，并在任务间传递执行结果。这里需要注意的是：**回调只会在当前任务执行成功的情况下才会发生，并且，父任务的返回值会作为参数传递给回调任务（子任务）**。



### 执行原语



在了解了任务签名及回调的概念后，就可以正式开始学习工作流的编排了。Celery 通过各种执行原语的组合来实现工作流的编排，而这些原语本身也是任务签名，所以开发者能够通过将原语进行再组合来形成更加复杂的任务流。



#### group

`group` 原语能够让一组任务并行执行：

```python
from celery import group
from tasks import add

g = group(add.s(2, 2), add.s(4, 4))

# 以下三个调用是等价的
g()
g.delay()
g.apply_async()
```

执行 `group` 后会返回一个 `GroupResult` 对象，这个对象中记录了这一组任务的执行结果和一些执行信息：

```python
>>> from celery import group
>>> from tasks import add

>>> job = group([
...             add.s(2, 2),
...             add.s(4, 4),
...             add.s(8, 8),
...             add.s(16, 16),
...             add.s(32, 32),
... ])

>>> result = job.apply_async()

>>> result.ready()  # 是否所有任务都完成了
True
>>> result.successful() # 是否所有任务都成功了
True
>>> result.get()
[4, 8, 16, 32, 64]
```



#### chain

`chain` 原语允许我们将任务像链表一样串起来，一个接着一个的执行：

```python
from celery import chain
from tasks import add

# 以下两个执行是等价的
c = chain(add.s(2, 2), add.s(4), add.s(8))
c = add.s(2, 2) | add.s(4) | add.s(8)
```

在这里，我们要先了解一下 Celery 中对于父任务和子任务的定义，我们在上述代码中创建的任务链如下图所示：

{% asset_img WechatIMG72.png [任务图示] %}

**在 Celery 中，某个任务的前置任务被称为其父任务**，也就是说，上图中的 task1 是 task2 的父任务，而 task2 是 task3 的父任务。

了解了这个概念之后，我们再来看看 Celery 为我们提供的一些 API。我们在调用 `chain` 之后，会拿到一个 `AsyncResult` 对象，通过调用其 `get()` 方法我们能够获取任务链的调用结果，但是这个结果是任务链最后的一个任务返回的结果，如果你需要获取链条中其他任务的执行结果，可以通过 `parent` 属性向前获取其父任务的执行结果：

```python
from celery import chain
from tasks import add

c = add.s(2, 2) | add.s(4) | add.s(8)
r = c.apply_async()
r.get()  # return 16
r.parent.get()  # return 8 
r.parent.parent.get()  # return 4
r.parent.parent.parent  # None
```

不仅如此，我们还能为任务链添加错误处理函数，当链条中的某一个任务出错时，就会调用这个处理函数：

```python
import logging

from celery import Celery

logger = logging.getLogger('celery')

app = Celery('tasks')

@app.task(name='task.tasks.task_1')
def task_1(fail=False):
    if fail:
        raise Exception()

@app.task(name='task.tasks.task_2')
def task_2(fail=False):
    if fail:
        raise Exception()

@app.task(name='task.tasks.log_error')
def log_error(request, exc, traceback):
    logger.error('bad thing happened!')
    logger.error('--\n\n{0} {1} {2}'.format(request.id, exc, traceback))
```

让我们来试一下：

```python
>>> c = task_1.s(True) | task_2.s()
>>> c.apply_async(link_error=log_error.s())
<AsyncResult: 2339608b-b735-4510-85f8-cad6abf7cbc4>
# 观察 worker 的输出，正确打印出了出错任务 id，异常和堆栈信息

>>> c.on_error(log_error.s()).delay()
<AsyncResult: 871e2a58-d9b1-49be-84d4-be7514eeb6f6>
# 观察 worker 的输出，正确打印出了出错任务 id，异常和堆栈信息

>>> c = task_1.s() | task_2.s(True)
>>> c.on_error(log_error.s()).delay()
...
TypeError: task_2() takes from 0 to 1 positional arguments but 2 were given
```

可以看到链条中的任何一个任务出错后，处理函数都会被调用，但是，当我们尝试让第二个任务出错时，貌似抛出的异常和预期设置的不一样。

这是因为 Celery 会把父任务的返回值向子任务传递，但是在 `c = task_1.s() | task_2.s(True)` 语句中我们却给 `task_2` 手动设置了 `fail` 的值，这样 `task_2` 在调用时就会收到两个参数，自然会抛出位置参数过多的异常。要解决这个问题，只需要使用**不可变签名**即可：

```python
# 以下两个调用是等价的
si = task_2.signature(args=(True,), immutable=True)
si = task_2.si(True)
```

**不可变签名在创建后就不会再接收其他的参数**，所以父任务传过来的参数也自然会被忽略：

```python
>>> c = task_1.s() | task_2.si(True)
>>> c.on_error(log_error.s()).delay()
# 这次即可看到正确的异常被抛出
```

如果你在任务链条中，不希望某个任务接收父任务的返回值，请务必使用不可变签名。



#### chord

使用 `chord` 原语能够让某个任务在一组任务完成执行后再开始执行。

例如我们要计算 `1 + 1 + 2 + 2 + ... + n + n` 的结果，我们可以将这个任务分为两个步骤（这是一个十分刻意的例子，只是为了说明 `chord` 的使用方法，最佳解决方案是直接计算 `sum(i + i for i in range(n))`）：

1. 完成 `n` 个 `i + i` 的计算
2. 将第 1 步的所有结果进行求和

那么首先我们需要将任务定义好：

```python
from celery import Celery

app = Celery()

@app.task
def add(x, y):
    return x + y

@app.task
def tsum(numbers):
    return sum(numbers)
```

然后，动手：

```python
>>> from celery import chord
>>> from tasks import add, tsum

>>> chord(add.s(i, i) for i in range(100))(tsum.s()).get()
9900
```

首先 `chord(add.s(i, i) for i in range(100))` 返回了一个可调用对象 `chord`，然后我们调用了这个对象并将 `tsum` 的签名传了进去，最后调用 `get()` 取得我们想要的结果。上述的例子也能被拆解为以下的方式来调用：

```python
>>> callback = tsum.s()
>>> header = [add.s(i, i) for i in range(100)]
>>> result = chord(header)(callback)
>>> result.get()
9900
```

也就是说，`chord` 操作分为两个主要的部分：`header` 和 `callback`，`callback` 会在 `header` 中所有任务都返回后才开始执行，且 `header` 中的任务的返回值会作为参数传递到  `callback` 中。但是要注意的是，如果 `header` 中的任意一个任务执行失败了，那么整个 `chord` 就会进入失败状态，不能再继续往下执行，对 `chord` 的失败处理与 `chain` 类似，可以参考 [chord error-handling](https://celery.readthedocs.io/en/latest/userguide/canvas.html#error-handling)。



#### map & starmap



`map` 和 `startmap` 都是在某个序列上重复的执行一个任务，但是其与 `group` 不同之处在：

- 只会执行一次任务
- 执行是串行的

`map` 的使用如下：

```python
>>> tsum.map([range(10), range(100), range(1000)])()
[45, 4950, 499500]
```

等价于执行了以下任务：

```python
@app.task
def temp():
    return [tsum(range(10)), tsum(range(100)), tsum(range(1000))]
```

而 `startmap` 的使用如下：

```python
>>> add.starmap(zip(range(10), range(10)))()
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

等价于执行了以下任务：

```python
@app.task
def temp():
    return [add(i, i) for i in range(10)]
```

那么 `map` 和 `startmap` 到底有什么区别呢？如果你要调用的任务只接受一个参数，那请使用 `map`，如果该任务接收多个参数，请使用 `starmap`。



#### chunk



最后让我们来看看 `chunk` ，`chunk` 能够让我们对任务进行分块，假如你有一个任务需要处理 100 个对象，那么你可以使用 `chunk` 将其分成 10 个任务，每个任务处理 10 个对象。如下所示：

```python
>>> add.chunks(zip(range(100), range(100)), 10)().get()
[[0, 2, 4, 6, 8, 10, 12, 14, 16, 18],
 [20, 22, 24, 26, 28, 30, 32, 34, 36, 38],
 [40, 42, 44, 46, 48, 50, 52, 54, 56, 58],
 [60, 62, 64, 66, 68, 70, 72, 74, 76, 78],
 [80, 82, 84, 86, 88, 90, 92, 94, 96, 98],
 [100, 102, 104, 106, 108, 110, 112, 114, 116, 118],
 [120, 122, 124, 126, 128, 130, 132, 134, 136, 138],
 [140, 142, 144, 146, 148, 150, 152, 154, 156, 158],
 [160, 162, 164, 166, 168, 170, 172, 174, 176, 178],
 [180, 182, 184, 186, 188, 190, 192, 194, 196, 198]]
```

观察 worker 的输出，不难发现，`chunk` 背后使用了 `startmap` 来实现：

```text
[2019-02-09 17:42:42,901: INFO/MainProcess] Received task: celery.starmap[c1a74285-4029-4389-829d-ba4a224343bc]
[2019-02-09 17:42:42,904: INFO/ForkPoolWorker-2] Task celery.starmap[c1a74285-4029-4389-829d-ba4a224343bc] succeeded in 0.0017125179874710739s: [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
[2019-02-09 17:42:42,904: INFO/MainProcess] Received task: celery.starmap[d42ce391-f6c2-4c2d-ac53-ffe0b12f6bdf]
[2019-02-09 17:42:42,906: INFO/MainProcess] Received task: celery.starmap[6f1688fa-6339-41dc-8060-5111aa961427]
[2019-02-09 17:42:42,908: INFO/MainProcess] Received task: celery.starmap[9ebe51a4-de65-46ba-8df8-5de02641bf12]
[2019-02-09 17:42:42,908: INFO/ForkPoolWorker-1] Task celery.starmap[d42ce391-f6c2-4c2d-ac53-ffe0b12f6bdf] succeeded in 0.002243652008473873s: [20, 22, 24, 26, 28, 30, 32, 34, 36, 38]
[2019-02-09 17:42:42,908: INFO/ForkPoolWorker-2] Task celery.starmap[6f1688fa-6339-41dc-8060-5111aa961427] succeeded in 0.0008158769924193621s: [40, 42, 44, 46, 48, 50, 52, 54, 56, 58]
[2019-02-09 17:42:42,909: INFO/MainProcess] Received task: celery.starmap[949611d3-7a49-4231-8c62-5c04d2e4bde2]
....
```

使用 `group` 方法，我们能够将一个 `chunk` 转成 `group`，这样就能够将 `chunk` 和 `chord` 结合起来使用： 

```python
@app.task(name='task.tasks.lsum')
def lsum(chunks):
    return sum([sum(l) for l in chunks])

>>> chord(group(add.s(i, i) for i in range(100)))(tsum.s()).get()  # chord1
9900

>>> chord(add.chunks(zip(range(100), range(100)), 10).group())(lsum.s()).get()  # chord2
9900
```

虽然上面两个 `chord` 的结果是相同的，但是执行的过程是不一样的：

- chord1 会先启动 100 个 `add` 任务计算 1 - 100 的合，完成后再启动 `tsum` 求和。
- chord2 会先启动 10 个 `add` `starmap` 任务计算 1 - 100 的合，完成后再启动 `lsum` 求和。

所以，根据实际使用场景，合理的使用 `chunk` 和 `group` 能够让我们的任务流更加清晰，也更加的高效。



### 小建议

Celery 虽然提供了任务工作流的功能，但是笔者并不推荐各位用这些原语来构造十分复杂和冗长的任务流，因为 Celery 只是提供了任务的串联和编排功能，对整个任务流程在执行过程中的一些控制（如暂停，失败重试，跳过等）能力却几乎为0，这意味着如果你构造出了一个十分复杂的流程，那么你就要去对这个流程执行中各种可能的情况进行预测和处理。如果你需要复杂流程编排、执行及控制的能力，请尝试使用工作流引擎来完成。



## 关于 worker 

关于 Celery，还有另外一个十分重要的组件就是 worker 了，可以说 worker 为 Celery 提供了无穷无尽的计算能力，在整个框架中承担着十分重要的职责。那么，深入的了解 worker 的一些功能，能够在我们日常开发和生产中解决问题时提供更多的思路。下面会重点讲解 worker 的探测和远程控制，并提及一些注意事项。



### 探测器

通过使用探测器，我们能够及时的获取当前所有可用 worker 及其状态，探测器同样从 app 实例中获取：

```python
>>> from celery import current_app as app
>>> i = app.control.inspect()  # 探测所有 worker

>>> app.control.inspect().ping()  # 对所有 worker 广播 ping 消息
{'worker1.example.com': {'ok': 'pong'}}

>>> i.registered()  # 获取所有 worker 中注册的任务
[{'worker1.example.com': ['tasks.add',
                          'tasks.sleeptask']}]

>>> i.active()  # 获取所有 worker 当前正在执行的任务
[{'worker1.example.com':
    [{'name': 'tasks.sleeptask',
      'id': '32666e9b-809c-41fa-8e93-5ae0c80afbbf',
      'args': '(8,)',
      'kwargs': '{}'}]}]

>>> i.scheduled()  # 获取所有 worker 的调度任务信息
[{'worker1.example.com':
    [{'eta': '2010-06-07 09:07:52', 'priority': 0,
      'request': {
        'name': 'tasks.sleeptask',
        'id': '1a7980ea-8b19-413e-91d2-0b74f3844c4d',
        'args': '[1]',
        'kwargs': '{}'}},
     {'eta': '2010-06-07 09:07:53', 'priority': 0,
      'request': {
        'name': 'tasks.sleeptask',
        'id': '49661b9a-aa22-4120-94b7-9ee8031d219d',
        'args': '[2]',
        'kwargs': '{}'}}]}]
```

上面列出的只是探测器的部分功能，有兴趣的读者可以看一看[探测器 API 手册](https://celery.readthedocs.io/en/latest/reference/celery.app.control.html#celery.app.control.Inspect)。



### 远程控制

Celery 能够通过 broker 中的一个高优先级队列来向所有的 worker 广播或是向某一些 worker 发送消息，从而实现远程控制的功能，上一小节的探测器功能就是通过远程控制来实现的。同时，worker 也能够响应这些消息，类似上一小节中的 `ping` 命令，但是由于我们无法得知集群中到底有多少个 worker，所以在使用一些响应式命令时，我们需要配置超时时间。如果某些 worker 在超时前始终没有回复我们发送的消息，并不代表这个 worker 不可用了，有可能因为网络延迟或是该 worker 正忙于处理其他命令而导致超时。



#### 广播函数

Celery 中的 `broadcast` 函数就是用于向所有的 worker 广播消息的，在远程控制的所有接口中，有一些（`ping`，`ratelimit`）则是广播函数的高阶实现，下面让我们看一下广播函数的使用示例，下面的代码向所有的 worker 发送了一条 rate_limit 消息，让所有的 worker 对名为 myapp.mytask 的任务进行限速：

```python
>>> from celery import current_app as app
>>> app.control.broadcast('rate_limit',
...                          arguments={'task_name': 'myapp.mytask',
...                                     'rate_limit': '200/m'})
```

想要深入了解广播函数，请移步 [broadcast 函数参考](https://celery.readthedocs.io/en/latest/userguide/workers.html#the-broadcast-function)。



#### 撤销任务

有些时候，我们可能想要撤销一些已经发送出去的任务，这个时候 `revoke` 函数就派上用场了：

```python
>>> from celery import current_app as app
>>> res = add.s(2, 2).apply_async(countdown=30)

>>> res.revoke()

>>> app.control.revoke(res.id)
```

当我们调用 `revoke` 之后，Celery 会向 worker 发送一条消息，告知 worker 某个 ID 的任务执行请求已经被撤销了，当 worker 取出相应 ID 的请求后，就会放弃执行这条请求。但是如果在我们调用 `revoke` 之前，任务已经被执行了，那么这次调用相当于是无效的。但是，Celery 还给了我们另一个选择，我们能够终止正在执行某个任务的 worker 子进程：

```python
>>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
...                    terminate=True)

>>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
...                    terminate=True, signal='SIGKILL')  # 自定义向 worker 子进程发送的信号
```

注意！上述操作是十分危险的！执行到一半的任务被强制中断可能会导致系统中出现数据不一致的情况。

Celery 提供的远程控制能力不只上面提到的这些，感兴趣的读者可以移步[远程控制用户指南](https://celery.readthedocs.io/en/latest/userguide/workers.html#remote-control)。



### 注意事项

在笔者的日常使用中发现，Celery 的 worker 在 prefork 模式下子进程存在内存泄漏的问题，并且在 github 上也有相关的一些 issue，如果读者们也遇到了 worker 子进程内存泄漏的问题的话，不妨考虑在启动 worker 进程时加上下面两个配置：

- [max task per child](https://celery.readthedocs.io/en/latest/userguide/workers.html#max-tasks-per-child-setting)
- [max memory per child](https://celery.readthedocs.io/en/latest/userguide/workers.html#max-memory-per-child-setting)



如果你配置了 `max task per child`，那么 worker 进程会在任意一个 worker 子进程执行的任务达到一定数量后用一个新的进程来替换该子进程，若配置了 `max memory per child`，则子进程会在占用内存达到一定的值后被新的进程替换掉。

使用这两个选项来缓解 Celery worker 内存占用过高的问题，从而避免 worker 因内存占用过多被操作系统强制杀死。



## 总结



我们首先了解了如何启动和配置 Celery，在有了 Celery 的核心 app 之后，我们又学习了如何定义和调用任务。而通过任务签名我们能够将一些简单的任务进行连接和编排而形成一个任务流。其次，Celery 本身也提供了一些周边的功能，如周期任务，探测器和远程控制等。希望各位在读完本文后对 Celery 有了更加深刻的理解，并在日常使用中能够玩的开心。



总的来说，Celery 虽然不是一个完美的异步任务队列，但它一定是一个值得信赖的异步任务队列。