---
title: Chapter 8 - 对象引用、可变性和垃圾回收
date: 2019-04-07 15:52:43
tags:
  - Python
categories:
  - Fluent Python
---


## 标识、相等性和别名

```python
In [1]: charles = {'name': 'Charles L. Dodgson', 'born': 1832}

In [2]: lewis = charles

In [3]: lewis == charles
Out[3]: True

In [4]: lewis is charles
Out[4]: True

In [5]: alex = {'name': 'Charles L. Dodgson', 'born': 1832}

In [6]: alex == charles
Out[6]: True

In [7]: alex is charles
Out[7]: False
```

`is` 运算符比较两个对象的 `id()`。

## 默认做浅复制

复制列表（或多数内置的可变集合）最简单的方式是使用内置的类型构造方法。例如：

```python
In [8]: l1 = [3, [55, 44], (7, 8, 9)]

In [9]: l2 = list(l1)

In [10]: l2
Out[10]: [3, [55, 44], (7, 8, 9)]

In [11]: l2 == l1
Out[11]: True

In [12]: l2 is l1
Out[12]: False
```

然而，构造方法或者 `[:]` 做的是浅复制，如果集合中有可变元素，可能会导致意想不到的问题：

```python
In [13]: l1[1].append(33)

In [14]: l1
Out[14]: [3, [55, 44, 33], (7, 8, 9)]

In [15]: l2
Out[15]: [3, [55, 44, 33], (7, 8, 9)]
```

## 函数的参数作为引用时

Python 唯一支持的参数传递模式是共享传参（call by sharing），指的是函数的各个形式参数获得实参中各个引用的副本，也就是说，函数内部的形参是实参的别名。

## del 与垃圾回收

`del` 语句删除名称，而不是对象。`del` 命令可能会导致对象被当做垃圾回收，但是仅当删除的变量保存的是对象的最后一个引用，或者无法得到对象时（孤岛）。

## 弱引用

弱引用不会增加对象的引用数量。引用的目标对象称为所指对象（referent）。因此我们说，弱引用不会妨碍所指对象被当作垃圾回收。

```python
In [1]: import weakref

In [2]: a_set = {0, 1}

In [3]: wref = weakref.ref(a_set)

In [4]: wref() is None
Out[4]: False

In [5]: a_set = {2, 3, 4}

In [6]: wref() is None
Out[6]: True
```

> Python 控制台会自动把 `_` 变量 绑定到结果不为 `None` 的表达式结果上。这凸显了一个实际问题：微观管理内存时，往往会得到意外的结果，因为不明显的隐式赋值会为对象创建新引用。控制台中的 `_` 变量是一例。调用跟踪对象也常导致意料之外的引用。

`weakref` 模块的文档（http://docs.python.org/3/library/weakref.html) 指出，`weakref.ref` 类其实是低层接口，供高级用途使用，多数程序最 好使用 weakref 集合和 finalize。也就是说，应该使用 `WeakKeyDictionary`、`WeakValueDictionary`、`WeakSet` 和 finalize（在内部使用弱引用），不要自己动手创建并处理 `weakref.ref` 实例。

不是每个 Python 对象都能够作为弱引用的目标。基本的 `list` 和 `dict` 实例不能作为所指出对象，但是它们的子类可以轻地解决这个问题：

```python
In [18]: import weakref

In [19]: weakref.ref([1, 2, 3])
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-19-7bab81ad732a> in <module>
----> 1 weakref.ref([1, 2, 3])

TypeError: cannot create weak reference to 'list' object

In [20]: class MyList(list):
    ...:     pass
    ...:

In [21]: weakref.ref(MyList([1, 2, 3]))
Out[21]: <weakref at 0x10b0c5688; dead>
```

这些局限基本上是 CPython 的实现细节，在其他 Python 解释器中情况可能不一样。