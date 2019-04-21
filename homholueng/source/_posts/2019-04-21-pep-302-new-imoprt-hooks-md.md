---
title: PEP 302 - New Import Hooks
tags:
  - Python
categories:
  - Python Daily
date: 2019-04-21 16:36:44
---


## Abstract

该提议为了使得开发者能够对 Python 的模块导入机制进行更好的自定义，提出了一种新的导入钩子，这种新型的机制能够更好注入到现有的导入机制中，并且对模块的查找和导入提供了更细粒度的控制。

## Motivation

在该提议出现之前，如果开发者要自定义模块的导入机制，需要重载内置的 `__import__` 函数，但是重载该函数十分麻烦，需要完全实现整个 `import` 的机制，还需要处理各种复杂的情况。

## Use Case

该提议提出的方案用于加载一些与一般模块存储方式不同的模块，例如打包好的模块，pyc 文件，从网络或数据库中读取的模块等等。

## Rationale

较早的一个方案是在 `sys.path` 中添加非 `str` 的类来作为 hook，但这是一种十分丑陋的 hack（ugly hack），原因如下：

- 破坏了 `sys.path` 中数据类型的一致性。
- 与 `PYTHONPATH` 环境变量不兼容。

该提议提出的钩子机制设计如下：在 `sys` 模块中添加一个新的 `path_hooks` 列表，该列表中的对象会被遍历以检测其是否能够处理 `sys.path` 中的某一条记录，直到找到能够处理的对象为止。但是每进行一次导入操作就遍历一次 `sys.path_hooks` 的话对性能会有一定的影响，所以遍历 `sys.path_hooks` 的结果会被缓存到 `sys.path_importer_cache` 中，该缓存是 `sys.path` 中的记录到 `path_hooks` 返回的模块导入器的映射。

为了最小化对 `import.c` 的影响以及避免增加额外的性能开销。该方案默认不会为现有的文件系统导入逻辑添加显式的钩子和导入器对象，如果 `sys.path_hooks` 中没有任何一个对象能够处理 `sys.path` 中的记录的话，就会回归到内置的导入逻辑。同时，这条记录在 `sys.path_importer_cache` 中映射会被置为 `None`，避免再次进行 `sys.path_hooks` 的遍历。

那么问题来了：对于那些不需要处理 `sys.path` 的自定义导入器应该如何处理呢？

为了解决这个问题，该提议还给出了另外一个新的机制：在 `sys` 模块中添加一个新的导入器列表 `meta_path`，这个列表中的导入器会在 `sys.path` 之前进行遍历并询问其是否能够处理即将要被导入的模块，该列表默认为空。

## Specification part 1: The Importer Protocol

Python 解释器在遇到导入模块的语句时，会调用 `__import__` 函数，解释器会向其传递四个参数，其中包括**被导入的模块**以及当前全局命名空间。**然后 `__import__` 函数会检测正在进行导入操作的模块是一个包还是包中的子模块。如果它是一个包，那么 `__import__` 会先尝试从相对路径下进行导入，如果是包中的子模块，那么会尝试从该模块的父包中进行导入。**而且，类似 `import spam.ham` 这样的导入操作会在 `import spam` 完成后，才会将 `ham` 作为 `spam` 的子模块进行导入。

而导入器协议（importer protocol）会在更高一层执行，当一个 importer 收到 `spam.ham` 的导入请求时，说明 `spam` 模块已经被成功导入了，而导入协议中定义了两种对象：finder 和 loader。

### Finder

Finder 对象必须实现以下方法：

```python
finder.find_module(fullname, path=None)
```

传入的 `fullname` 是模块的完全限定名（e.g. `spam.ham`）。而对于第二个参数 `path`，如果当前导入的模块是顶层模块，那么 `path` 的值为 `None`；如果当前导入的不是顶层模块，那么 `path` 的值是 `package.__path__`。

如果 Finder 能够处理当前正在导入的模块，其 `find_module()` 方法应该返回一个 Loader 对象；反之应该返回 `None`。

该方法中抛出的异常会传播给调用者，且本次导入操作会被放弃。

### Loader

Finder 对象必须实现以下方法：

```python
loader.load_module(fullname)
```

传入的 `fullname` 是模块的完全限定名（e.g. `spam.ham`）。

该方法应该返回一个已经加载完成的模块或抛出 `ImporteError` 异常；如果要求该方法加载不能被当前 Loader 加载的模块，也应该抛出 `ImporteError` 异常。而且，`load_module()` 方法在其执行模块中的代码之前必须履行以下职责：

- 如果 `sys.modules` 中存在名为 `fullname` 模块对象，Loader 必须使用该现有模块（否则 `reload（`）将无法正常工作）。反之，如果 `sys.modules` 中不存在名为 `fullname` 的模块，则Loader 必须创建一个新模块对象并将其添加到 `sys.modules` 中。**注意，在 Loader 执行模块代码之前，必须先将模块对象添加到 `sys.modules` 中，因为模块可以（直接或间接）导入自身；事先将它添加到 `sys.modules` 可以防止无限制的递归或是多次加载。**如果模块加载失败， Loader 需要删除它可能已插入 `sys.modules` 的任何模块；如果模块已经位于 `sys.modules` 中，那么 Loader 可以不处理该模块。
- 必须为模块设置 `__file__` 属性，且类型必须是字符串，但它可以是一个虚拟的值，例如 `“<frozen>”`。
- 必须为模块设置 `__name__` 属性。
- 如果导入的是包，那么必须设置 `__path__` 属性，且该类型必须是 `list`。
- 必须将模块的 `__loader__` 属性设置为 Laoder 自身，用于自省和重载，也可用于获取 `importer` 的信息。
- 必须微模块设置 `__package__` 属性。

以下是一个 `load_module()` 的一个实现模板：

```python
def load_module(self, fullname):
    code = self.get_code(fullname)
    ispkg = self.is_package(fullname)
    mod = sys.modules.setdefault(fullname, imp.new_module(fullname))
    mod.__file__ = "<%s>" % self.__class__.__name__
    mod.__loader__ = self
    if ispkg:
        mod.__path__ = []
        mod.__package__ = fullname
    else:
        mod.__package__ = fullname.rpartition('.')[0]
    exec(code, mod.__dict__)
    return mod
```

## Specification part 2: Registering Hooks

在新的导入钩子机制下，存在两种类型的钩子：

- Meta hooks：在导入过程最开始的时候调用；通过将 Finder 对象添加到 `sys.meta_path` 中实现注册。
- Path hooks：在处理 `sys.path` 过程中遇到相关路径的时候调用；通过将导入器工厂添加到 `sys.path_hooks` 中实现注册。

`sys.path_hooks` 是一个可调用对象（导入器工厂）的列表，这些对象会以 `sys.path` 中的每一条数据作为参数进行调用。如果某个对象能够处理当前传入的 path，那么其应该返回一个导入器对象；反之则应该抛出 `ImportError`。

如之前章节中所描述的，这些导入器对象会被缓存在 `sys.path_importer_cache` 中，如果在某些情况下需要重新执行一次 Path hook，可以手动删除 `sys.path_importer_cache` 某些映射。

## Packages and the role of `__path__`

如果一个模块拥有 `__path__` 属性的话，那么 Python 的导入机制就会将该模块当成包来对待。当导入该模块下的子模块时，会使用该模块的 `__path__` 属性而不是 `sys.path` 中的某一项，而应用在 `sys.path` 上的规则同样适用于 `pkg.__path__`，也就是说在遍历 `pkg.__path__` 时，也会查询 `sys.path_hooks` 是否有能够处理的导入器工厂。


## Optional Extensions to the Importer Protocol

```python
loader.get_data(path)
```

该方法应该从底层存储中获取文件的内容并返回；如果文件不存在的话，应该抛出 `IOError`。

```python
loader.is_package(fullname)
```

该方法应该判断指定模块是否是一个包。

```python
loader.get_code(fullname)
```

该方法应该返回与指定模块相符的 code 对象；如果指定模块是内置或扩展模块的话，则返回 `None`。

```python
loader.get_source(fullname)
```

该方法应该返回指定模块的源码；如果无法获取该模块的源码，则返回 `None`；如果当前 Loader 无法处理该模块，则应该抛出 `ImportError`。

```python
loader.get_filename(fullname)
```

该方法应该返回指定模块的 `__file__` 属性的值；如果模块不存在，则应该抛出 `ImportError`。

