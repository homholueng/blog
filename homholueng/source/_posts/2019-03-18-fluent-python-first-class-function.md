---
title: Chapter 5 - 一等函数
date: 2019-03-18 22:48:59
tags:
  - Python
categories:
  - Fluent Python
---


在 Python 中，函数是一等对象。编程语言理论家把 “一等对象” 定义为满足下述条件的程序实体：

- 在运行时创建
- 能赋值给变量或数据结构中的元素
- 能作为参数传递给函数
- 能作为函数的返回结果

## 函数内省

与用户定义的常规类一样，函数使用 `__dict__` 属性存储赋予它的用户属性。但是，函数也拥有一些专有而用户定义的**一般对象**没有的属性：

- `__annotations__`：`dict` 类型，参数和返回值的注解。
- `__call__`：`method-wrapper` 类型，实现 `()` 运算符，即可调用对象协议。
- `__closure__`：`tuple` 类型，函数闭包，即自由变量的绑定。
- `__code__`：`code` 类型，编译成字节码的函数元数据和函数定义体。
- `__defaults__`：`tuple` 类型，形式参数的默认值。
- `__get__`：`method-wrapper` 类型，实现只读描述符协议。
- `__globals__`：`dict` 类型，函数所在模块中的全局变量。
- `__kwdefaults__`：`dict` 类型，仅限关键字形式参数的默认值。
- `__name__`：`str` 类型，函数名称。
- `__qualname__`：`str` 类型，函数的限定名称，如 `Random.choice`。

在这里要说明一下，仅限关键字参数是 Python 3 中的新特性，仅限关键字参数只能通过关键字参数指定，它一定不会捕获未命名的定位参数：

```python
# cls 为仅限关键字参数
def tag(name, *content, cls=None, **attrs):
    pass

# b 也是仅限关键字参数
def f(a, *, b):
    pass
```

### 获取关于参数的信息

我们以在 `clip.py` 模块下定义的 `clip` 函数作为示例：

```python
def clip(text, max_len=80):
    """
    在 max_len 前面或后面的第一个空格处截断文本
    """
    end = None
    if len(text) > max_len:
        space_before = text.rfind(' ', 0, max_len)
        if space_before >= 0:
            end = space_before
        else:
            space_after = text.rfind(' ', max_len)
            if space_after >= 0:
                end = space_after

    if end is None:
        end = len(text)
    
    return text[:end].rstrip()


In [25]: clip.__code__.co_varnames
Out[25]: ('text', 'max_len', 'end', 'space_before', 'space_after')

In [26]: clip.__code__.co_argcount
Out[26]: 2
```

通过 `__code__` 对象来获取函数信息未免有些不方便，我们一般会使用 `inspect` 模块来获取函数相关信息：

```python
In [29]: from inspect import signature

In [30]: sig = signature(clip)

In [31]: sig
Out[31]: <Signature (text, max_len=80)>

In [32]: for name, param in sig.parameters.items():
    ...:     print(param.kind, ':', name, '=', param.default)
    ...:
POSITIONAL_OR_KEYWORD : text = <class 'inspect._empty'>
POSITIONAL_OR_KEYWORD : max_len = 80
```

`inspect.signature` 函数返回一个 `inspect.Signature` 对象，它有一个 parameters 属性，这是一个有序映射，把参数名 `inspect.Parameter` 对象对应起来。`inspect.Parameter` 对象的 `kind` 属性有以下五种值：

- POSITIONAL_OR_KEYWORD：可以通过定位参数和关键字参数传入的形参。
- VAR_POSITIONAL：定位参数元组。
- VAR_KEYWORD：关键字参数字典。
- KEYWORD_ONLY：仅限关键字参数（Python 3 新增）。
- POSITIONAL_ONLY：仅限定位参数；目前，Python 声明函数的句法不支持，但是有些使 用 C 语言实现且不接受关键字参数的函数（如 `divmod`）支持。

`inspect.Signature` 对象有个 `bind` 方法，它可以把任意个参数绑定到签名中的形参上，所用的规则与实参到形参的匹配方式一样。框架可以使用这个方法在真正调用函数前验证参数：

```python
In [33]: import inspect

In [34]: sig = inspect.signature(clip)

In [35]: my_clip = {'text': '1 2 3 4 5', 'max_len': 5}

In [39]: bound_args = sig.bind(**my_clip)

In [40]: bound_args
Out[40]: <BoundArguments (text='1 2 3 4 5', max_len=5)>

In [41]: for name, value in bound_args.arguments.items():
    ...:     print(name, '=', value)
    ...:
text = 1 2 3 4 5
max_len = 5

In [42]: del my_clip['text']

In [43]: sig.bind(**my_clip)
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-43-1821306ba99f> in <module>
----> 1 sig.bind(**my_clip)

~/.pyenv/versions/3.6.4/lib/python3.6/inspect.py in bind(*args, **kwargs)
   2966         if the passed arguments can not be bound.
   2967         """
-> 2968         return args[0]._bind(args[1:], kwargs)
   2969
   2970     def bind_partial(*args, **kwargs):

~/.pyenv/versions/3.6.4/lib/python3.6/inspect.py in _bind(self, args, kwargs, partial)
   2881                             msg = 'missing a required argument: {arg!r}'
   2882                             msg = msg.format(arg=param.name)
-> 2883                             raise TypeError(msg) from None
   2884             else:
   2885                 # We have a positional argument to process

TypeError: missing a required argument: 'text'
```

## 支持函数式编程的包

### operator 模块

比较哟用的两个函数是 `itemgetter` 及 `attrgetter`，由于 `itemgetter` 使用的是 `[]` 运算符来取值，所以其不仅仅支持序列，还支持映射和任何实现 `__getitem__` 方法的类：

```python
In [1]: from operator import itemgetter

In [2]: metro_data = [  ('Tokyo', 'JP', 36.933, (35.689722, 139.691667)),  ('Delhi NCR', 'IN', 21.935, (28.613889, 77.208889)),  ('Mexi
   ...: co City', 'MX', 20.142, (19.433333, -99.133333)),  ('New York-Newark', 'US', 20.104, (40.808611, -74.020386)),  ('Sao Paulo', '
   ...: BR', 19.649, (-23.547778, -46.635833)),  ]

In [3]: for city in sorted(metro_data, key=itemgetter(1)):
   ...:     print(city)
   ...:
('Sao Paulo', 'BR', 19.649, (-23.547778, -46.635833))
('Delhi NCR', 'IN', 21.935, (28.613889, 77.208889))
('Tokyo', 'JP', 36.933, (35.689722, 139.691667))
('Mexico City', 'MX', 20.142, (19.433333, -99.133333))
('New York-Newark', 'US', 20.104, (40.808611, -74.020386))

In [4]: cc_name = itemgetter(1, 0)

In [5]: for city in metro_data:
   ...:     print(cc_name(city))
   ...:
('JP', 'Tokyo')
('IN', 'Delhi NCR')
('MX', 'Mexico City')
('US', 'New York-Newark')
('BR', 'Sao Paulo')
```

`attrgetter` 与 `itemgetter` 作用类似，它创建的函数根据名称提取对象的属性。如果把多个属性名传给 attrgetter，它也会返回提取的值构成的元组。此外，如果参数名中包含 `.`（点号），`attrgetter` 会深入嵌套对象，获取指定的属性。

在 `operator` 模块余下的函数中，最后介绍一下 `methodcaller`，其创建的函数会在对象上调用参数指定的方法：

```python
In [11]: from operator import methodcaller

In [12]: s = 'The time has come'

In [13]: upcase = methodcaller('upper')

In [15]: upcase(s)
Out[15]: 'THE TIME HAS COME'

In [16]: hiphenate = methodcaller('replace', ' ', '-')

In [17]: hiphenate(s)
Out[17]: 'The-time-has-come'
```

### functools 模块

另外，我们也可以透过 `functools` 中的 `partial` 和 `partialmethod` 函数来将现有函数的某些参数固定来生成新的函数，减少重复并提升代码的可读性，这两个函数在此不多做赘述。