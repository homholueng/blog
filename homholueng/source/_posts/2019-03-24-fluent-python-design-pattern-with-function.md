---
title: Chapter 6 - 使用一等函数实现设计模式
tags:
  - Python
categories:
  - Fluent Python
date: 2019-03-24 19:52:18
---


1996 年，Peter Norvig 在题为“Design Patterns in Dynamic Languages”（http://norvig.com/design-patterns/) 的演讲中指出，在 Gamma 等人合著的《设计模式：可复用面向对象软件基础》一书中包含了 23 个模式，其中有 16 个在动态语言中 “不见了，或者简化了”。

Norvig 建议在有一等函数的语言中重新审视 “策略”，“命令”，“模板方法” 和 “访问者” 模式。通常，我们可以把这些模式中涉及的某些类的实例替换成简单的函数，从而减少样板代码。

### 策略模式

策略模式的目的在于将一系列的算法封装起来，并且使他们可以相互替换。如下所示：

{% asset_img strategy-pattern.png [策略模式] %}

这个模式使得算法可以独立于使用他们的调用者而变化。

我们使用类来描述一个策略，并在计算订单总额时根据顾客类型来确定并初始化相应的策略类。

但是在有了一等函数后，我们可以不再使用策略类，而是只使用函数类代表不同的策略，并在计算总额的时候将策略函数作为参数进行传递。Python 中的 `sort()` 的 `key` 参数就是策略函数的一个例子。

使用一等函数实现策略模式时，不同策略的管理可能是个问题，此时可以使用统一的模块来管理所有的策略函数，并通过 `inspect` 模块来获取所有的策略：

```python
import promotions # 策略模块

promos = [func for name, func in inspect.getmembers(promotions, inspect.isfunction)]

def best_promo(order):
    """
    选择最佳折扣
    """
    return max(promo(order) for promo in promos)
```

上述方案没有检查所有策略函数的参数，如果想更进一步，可以根据函数接收的参数再次过滤，避免在使用时抛出异常。


### 命令模式

命令模式通过把函数作为参数传递而简化，如下所示：

{% asset_img command-pattern.png [命令模式] %}

调用者只需要执行相应的命令，而不同的命令后有不同的接受者负责处理。该模式的目的是解耦调用操作的对象和提供实现的对象。

同样的，我们可以不为调用者提供一个 Command 实例，而是给它一个函数，此时调用者可以不用调用 `command.execute()`，而是直接调用 `command()` 即可。

## 小结

可以看到，上面两种模式都是将设计模式中的类替换成了函数，这样做的好处在于减少运行时的开销，因为使用类时需要每次都进行类对象的实例化，而使用函数时则不会有这个问题。