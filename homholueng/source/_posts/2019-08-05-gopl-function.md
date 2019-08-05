---
title: Go 语言圣经 - 5. 函数
tags:
  - Go
categories:
  - Go 语言圣经
date: 2019-08-05 21:33:25
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

拥有函数名的函数只能在包级语法块中被声明，通过函数字面量（function literal），我们可绕过这一限制，在任何表达式中表示一个函数值。

函数字面量允许我们在使用函数时，再定义它。通过这种技巧，我们可以改写之前对 `strings.Map` 的调用：

```go
strings.Map(func(r rune) rune {return r + 1}, "HAL-9000")
```

更为重要的是，通过这种方式定义的函数可以访问完整的词法环境（lexical environment），这意味着在函数中定义的内部函数可以引用该函数的变量，如下例所示：

```go
func squares() func() int {
    var x int
    return func() int {
        x++
        return x * x
    }
}
func main() {
    f := squares()
    fmt.Println(f()) // "1"
    fmt.Println(f()) // "4"
    fmt.Println(f()) // "9"
    fmt.Println(f()) // "16"
}
```
`squares` 的例子证明，函数值不仅仅是一串代码，还记录了状态。在 squares 中定义的匿名内部函数可以访问和更新 `squares` 中的局部变量，这意味着匿名函数和 `squares` 中，存在变量引用。这就是函数值属于引用类型和函数值不可比较的原因。Go 使用闭包（closures）技术实现函数值，Go 程序员也把函数值叫做闭包。

当匿名函数需要被递归调用时，我们必须首先声明一个变量，再将匿名函数赋值给这个变量。如果不分成两步，函数字面量无法与特定变量绑定，我们也无法递归调用该匿名函数。

```go
var visitAll func(items []string)
visitAll = func(items []string) {
    // ...
    visitAll(items)
    // ...
}
```

### 5.6.1. 警告：捕获迭代变量

> 本节，将介绍 Go 词法作用域的一个陷阱。请务必仔细的阅读，弄清楚发生问题的原因。即使是经验丰富的程序员也会在这个问题上犯错误。


考虑这样一个问题：你被要求首先创建一些目录，再将目录删除。在下面的例子中我们用函数值来完成删除操作。下面的示例代码需要引入 `os` 包。为了使代码简单，我们忽略了所有的异常处理。

```go
var rmdirs []func()
for _, d := range tempDirs() {
    dir := d // NOTE: necessary!
    os.MkdirAll(dir, 0755) // creates parent directories too
    rmdirs = append(rmdirs, func() {
        os.RemoveAll(dir)
    })
}
// ...do some work…
for _, rmdir := range rmdirs {
    rmdir() // clean up
}
```

你可能会感到困惑，为什么要在循环体中用循环变量 `d` 赋值一个新的局部变量，而不是像下面的代码一样直接使用循环变量 `dir`。

问题的原因在于循环变量的作用域。在上面的程序中，`for` 循环语句引入了新的词法块，循环变量 `dir` 在这个词法块中被声明。在该循环中生成的所有函数值都共享相同的循环变量。需要注意，函数值中记录的是循环变量的内存地址，而不是循环变量某一时刻的值。以 `dir` 为例，后续的迭代会不断更新 `dir` 的值，当删除操作执行时，`for` 循环已完成，`dir` 中存储的值等于最后一次迭代的值。这意味着，每次对 `os.RemoveAll` 的调用删除的都是相同的目录。

通常，为了解决这个问题，我们会引入一个与循环变量同名的局部变量，作为循环变量的副本。比如下面的变量 `dir`，虽然这看起来很奇怪，但却很有用。

```go
for _, dir := range tempDirs() {
    dir := dir // declares inner dir, initialized to outer dir
    // ...
}
```

这个问题不仅存在基于 `range` 的循环，在三段式的 `for` 循环中，也会存在这样的问题。

## 5.7. 可变参数

参数数量可变的函数称为可变参数函数。


在声明可变参数函数时，需要在参数列表的最后一个参数类型之前加上省略符号 `...`，这表示该函数会接收任意数量的该类型参数。

```go
func sum(vals...int) int {
    total := 0
    for _, val := range vals {
        total += val
    }
    return total
}
```
在函数体中，`vals` 被看作是类型为 `[]int` 的切片。`sum` 可以接收任意数量的 `int` 型参数：

```go
fmt.Println(sum(1, 2, 3, 4))
```

在上面的代码中，调用者隐式的创建一个数组，并将原始参数复制到数组中，再把数组的一个切片作为参数传给被调用函数。如果原始参数已经是切片类型，我们该如何传递给 `sum`？只需在最后一个参数后加上省略符。下面的代码功能与上个例子相同。

```go
values := []int{1, 2, 3, 4}
fmt.Println(sum(values...)) // "10"
```

## 5.8. Deferred 函数

你只需要在调用普通函数或方法前加上关键字 `defer`，就完成了 `defer` 所需要的语法。 **当执行到该条语句时，函数和参数表达式得到计算，但直到包含该 `defer` 语句的函数执行完毕时，`defer` 后的函数才会被执行，不论包含 `defer` 语句的函数是通过 `return` 正常结束，还是由于 panic 导致的异常结束。** 你可以在一个函数中执行多条 `defer` 语句，它们的执行顺序与声明顺序相反。

`defer` 语句经常被用于处理成对的操作，如打开、关闭、连接、断开连接、加锁、释放锁。通过 `defer` 机制，不论函数逻辑多复杂，都能保证在任何执行路径下，资源被释放。

调试复杂程序时，defer 机制也常被用于记录何时进入和退出函数。下例中的 `bigSlowOperation` 函数，直接调用 `trace` 记录函数的被调情况。**需要注意一点：不要忘记 `defer` 语句后的圆括号，否则本该在进入时执行的操作会在退出时执行，而本该在退出时执行的，永远不会被执行。**

```go
func bigSlowOperation() {
    defer trace("bigSlowOperation")() // don't forget the extra parentheses

    time.Sleep(10 * time.Second)
}

func trace(msg string) func() {
    start := time.Now()
    log.Printf("enter: %s", msg)
    return func() {
        log.Printf("exit %s (%s)", msg, time.Since(start))
    }
}
```

我们知道，`defer` 语句中的函数会在 `return` 语句更新返回值变量后再执行，又因为在函数中定义的匿名函数可以访问该函数包括返回值变量在内的所有变量，所以，对匿名函数采用 defer 机制，可以使其观察函数的返回值。

被延迟执行的匿名函数甚至可以修改函数返回给调用者的返回值：

```go
func triple(x int) (result int) {
    defer func() { result += x }()
    return double(x)
}
fmt.Println(triple(4)) // "12"
```

在循环体中的 `defer` 语句需要特别注意，因为只有在函数执行完毕后，这些被延迟的函数才会执行。下面的代码会导致系统的文件描述符耗尽，因为在所有文件都被处理之前，没有文件会被关闭。

```go
for _, filename := range filenames {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // NOTE: risky; could run out of file descriptors
    // ...process f…
}
```

## 5.9. Panic 异常

Go 的类型系统会在编译时捕获很多错误，但有些错误只能在运行时检查，如数组访问越界、空指针引用等。这些运行时错误会引起 `painc` 异常。

一般而言，当 panic 异常发生时，程序会中断运行，并立即执行在该 goroutine 中被延迟的函数（defer 机制）。随后，程序崩溃并输出日志信息。日志信息包括 panic value 和函数调用的堆栈跟踪信息。panic value 通常是某种错误信息。对于每个 goroutine，日志信息中都会有与之相对的，发生 panic 时的函数调用堆栈跟踪信息。

不是所有的 panic 异常都来自运行时，直接调用内置的 `panic` 函数也会引发 panic 异常；panic 函数接受任何值作为参数。当某些不应该发生的场景发生时，我们就应该调用 panic。比如，当程序到达了某条逻辑上不可能到达的路径：

```go
switch s := suit(drawCard()); s {
    case "Spades":                                // ...
    case "Hearts":                                // ...
    case "Diamonds":                              // ...
    case "Clubs":                                 // ...
    default:
        panic(fmt.Sprintf("invalid suit %q", s)) // Joker?
}
```

虽然 Go 的 panic 机制类似于其他语言的异常，但 panic 的适用场景有一些不同。由于 panic 会引起程序的崩溃，因此 panic 一般用于严重错误，如程序内部的逻辑不一致。勤奋的程序员认为任何崩溃都表明代码中存在漏洞，所以对于大部分漏洞，我们应该使用 Go 提供的错误机制，而不是 panic，尽量避免程序的崩溃。在健壮的程序中，任何可以预料到的错误，如不正确的输入、错误的配置或是失败的 I/O 操作都应该被优雅的处理，最好的处理方式，就是使用 Go 的错误机制。但是在某些场景下，我们希望程序一定不能抛出错误，否则就引发 panic，go 的标准库中为了满足这种情况，许多函数都提供了 `Must` 前缀的包装版本：

```go
package regexp
func Compile(expr string) (*Regexp, error) { /* ... */ }
func MustCompile(expr string) *Regexp {
    re, err := Compile(expr)
    if err != nil {
        panic(err)
    }
    return re
}
```

为了方便诊断问题，`runtime` 包允许程序员输出堆栈信息。在下面的例子中，我们通过在 `main` 函数中延迟调用 `printStack` 输出堆栈信息。

```go
func main() {
    defer printStack()
    f(3) // will panic
}
func printStack() {
    var buf [4096]byte
    n := runtime.Stack(buf[:], false)
    os.Stdout.Write(buf[:n])
}
```

## 5.10. Recover 捕获异常

通常来说，不应该对 panic 异常做任何处理，但有时，也许我们可以从异常中恢复，至少我们可以在程序崩溃前，做一些操作。举个例子，当 web 服务器遇到不可预料的严重问题时，在崩溃前应该将所有的连接关闭；如果不做任何处理，会使得客户端一直处于等待状态。如果 web 服务器还在开发阶段，服务器甚至可以将异常信息反馈到客户端，帮助调试。

如果在 deferred 函数中调用了内置函数 `recover`，并且定义该 `defer` 语句的函数发生了 panic 异常，`recover` 会使程序从 panic 中恢复，并返回 panic value。导致 panic 异常的函数不会继续运行，但能正常返回。在未发生 panic 时调用 `recover`，`recover` 会返回 `nil`。

```go
func panicFunc(x int) int {
	return 1 / x
}

func main() {

	defer func() {
		if p := recover(); p != nil {
			err = fmt.Errorf("internal error: %v", p)
		}
	}()

	fmt.Println(panicFunc(0))
}

```

不加区分的恢复所有的 panic 异常，不是可取的做法；因为在 panic 之后，无法保证包级变量的状态仍然和我们预期一致。比如，对数据结构的一次重要更新没有被完整完成、文件或者网络连接没有被关闭、获得的锁没有被释放。此外，如果写日志时产生的 panic 被不加区分的恢复，可能会导致漏洞被忽略。

虽然把对 panic 的处理都集中在一个包下，有助于简化对复杂和不可以预料问题的处理，但作为被广泛遵守的规范，你不应该试图去恢复其他包引起的 panic。公有的 API 应该将函数的运行失败作为 error 返回，而不是 panic。同样的，你也不应该恢复一个由他人开发的函数引起的 panic，比如说调用者传入的回调函数，因为你无法确保这样做是安全的。

