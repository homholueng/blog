---
title: Go 语言圣经 - 5. 函数
tags:
  - Go
categories:
  - Go 语言圣经
---

## 5.1. 函数声明

函数声明包括函数名、形式参数列表、返回值列表（可省略）以及函数体。

```go
func name(parameter-list) (result-list) {
    body
}
```

返回值也可以像形式参数一样被命名。在这种情况下，每个返回值被声明成一个局部变量，并根据该返回值的类型，将其初始化为该类型的零值。 如果一个函数在声明时，包含返回值列表，该函数必须以 `return` 语句结尾，除非函数明显无法运行到结尾处。

如果一组形参或返回值有相同的类型，我们不必为每个形参都写出参数类型。下面 2 个声明是等价的：

```go
func f(i, j, k int, s, t string)                 { /* ... */ }
func f(i int, j int, k int,  s string, t string) { /* ... */ }
```

函数的类型被称为函数的签名。如果两个函数形式参数列表和返回值列表中的变量类型一一对应，那么这两个函数被认为有相同的类型或签名。形参和返回值的变量名不影响函数签名，也不影响它们是否可以以省略参数类型的形式表示。

实参通过值的方式传递，因此函数的形参是实参的拷贝。对形参进行修改不会影响实参。但是，如果实参包括引用类型，如指针，slice (切片)、map、function、channel 等类型，实参可能会由于函数的间接引用被修改。

你可能会偶尔遇到没有函数体的函数声明，这表示该函数不是以 Go 实现的。这样的声明定义了函数签名。


```go
package math

func Sin(x float64) float //implemented in assembly language
```

## 5.3. 多返回值


调用多返回值函数时，返回给调用者的是一组值，调用者必须显式的将这些值分配给变量:

```go
links, err := findLinks(url)
```

如果某个值不被使用，可以将其分配给 blank identifier:

```go
links, _ := findLinks(url) // errors ignored
```

当你调用接受多参数的函数时，可以将一个返回多参数的函数调用作为该函数的参数。虽然这很少出现在实际生产代码中，但这个特性在 debug 时很方便，我们只需要一条语句就可以输出所有的返回值。下面的代码是等价的：

```go
log.Println(findLinks(url))
links, err := findLinks(url)
log.Println(links, err)
```

*如果一个函数所有的返回值都有显式的变量名，那么该函数的 return 语句可以省略操作数。这称之为 bare return。*

**当一个函数有多处 return 语句以及许多返回值时，bare return 可以减少代码的重复，但是使得代码难以被理解。**

## 5.4. 错误

通常，导致失败的原因不止一种，尤其是对 I/O 操作而言，用户需要了解更多的错误信息。因此，额外的返回值不再是简单的布尔类型，而是 error 类型，内置的 error 是接口类型。

通常，当函数返回 non-nil 的 error 时，其他的返回值是未定义的（undefined），这些未定义的返回值应该被忽略。然而，有少部分函数在发生错误时，仍然会返回一些有用的返回值。比如，当读取文件发生错误时，`Read` 函数会返回可以读取的字节数以及错误信息。


### 5.4.1. 错误处理策略

当一次函数调用返回错误时，调用者应该选择合适的方式处理错误。根据情况的不同，有很多处理方式，让我们来看看常用的五种方式。

#### 1

首先，也是最常用的方式是传播错误。这意味着函数中某个子程序的失败，会变成该函数的失败。

```go
resp, err := http.Get(url)
if err != nil {
    return nil, err
}
```

当对 `html.Parse` 的调用失败时，`findLinks` 不会直接返回 `html.Parse` 的错误，因为缺少两条重要信息：1、发生错误时的解析器（html parser）；2、发生错误的 url。因此，findLinks 构造了一个新的错误信息，既包含了这两项，也包括了底层的解析出错的信息。

```go
doc, err := html.Parse(resp.Body)
resp.Body.Close()
if err != nil {
    return nil, fmt.Errorf("parsing %s as HTML: %v", url,err)
}
```

我们使用该函数添加额外的前缀上下文信息到原始错误信息。当错误最终由 `main` 函数处理时，错误信息应提供清晰的从原因到后果的因果链，就像美国宇航局事故调查时做的那样：

```
genesis: crashed: no parachute: G-switch failed: bad relay orientation
```

由于错误信息经常是以链式组合在一起的，所以错误信息中应避免大写和换行符。

编写错误信息时，我们要确保错误信息对问题细节的描述是详尽的。尤其是要注意错误信息表达的一致性，即相同的函数或同包内的同一组函数返回的错误在构成和处理方式上是相似的。

以 os 包为例，os 包确保文件操作（如 `os.Open`、`Read`、`Write`、`Close`）返回的每个错误的描述不仅仅包含错误的原因（如无权限，文件目录不存在）也包含文件名，这样调用者在构造新的错误信息时无需再添加这些信息。

#### 2

如果错误的发生是偶然性的，或由不可预知的问题导致的。一个明智的选择是重新尝试失败的操作。在重试时，我们需要限制重试的时间间隔或重试的次数，防止无限制的重试。

#### 3

如果错误发生后，程序无法继续运行，我们就可以采用第三种策略：输出错误信息并结束程序。需要注意的是，这种策略只应在 `main` 中执行。

```go
// (In function main.)
if err := WaitForServer(url); err != nil {
    fmt.Fprintf(os.Stderr, "Site is down: %v\n", err)
    os.Exit(1)
}
```

调用 `log.Fatalf` 可以更简洁的代码达到与上文相同的效果。log 中的所有函数，都默认会在错误信息之前输出时间信息。

```go
if err := WaitForServer(url); err != nil {
    log.Fatalf("Site is down: %v\n", err)
}
```

#### 4

第四种策略：有时，我们只需要输出错误信息就足够了，不需要中断程序的运行。我们可以通过 log 包提供函数

```go
if err := Ping(); err != nil {
    log.Printf("ping failed: %v; networking disabled", err)
}
```

或者标准错误流输出错误信息。

```go
if err := Ping(); err != nil {
    fmt.Fprintf(os.Stderr, "ping failed: %v; networking disabled\n", err)
}
```

log 包中的所有函数会为没有换行符的字符串增加换行符。

#### 5

第五种，也是最后一种策略：我们可以直接忽略掉错误。

### 5.4.2. 文件结尾错误（EOF）

io 包保证任何由文件结束引起的读取失败都返回同一个错误 —— `io.EOF`，该错误在 io 包中定义：

```
package io

import "errors"

// EOF is the error returned by Read when no more input is available.
var EOF = errors.New("EOF")
```

## 5.5. 函数值

在 Go 中，函数被看作第一类值（first-class values）：函数像其他值一样，拥有类型，可以被赋值给其他变量，传递给函数，从函数返回。

```go
func square(n int) int { return n * n }
func negative(n int) int { return -n }
func product(m, n int) int { return m * n }

f := square
fmt.Println(f(3)) // "9"

f = negative
fmt.Println(f(3))     // "-3"
fmt.Printf("%T\n", f) // "func(int) int"

f = product // compile error: can't assign func(int, int) int to func(int) int
```

函数类型的零值是 `nil`，函数值可以与 `nil` 比较。

但是函数值之间是不可比较的，也不能用函数值作为 map 的 key。

## 5.6. 匿名函数


