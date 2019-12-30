---
title: Go 语言圣经 - 7. 接口
tags:
  - Go
categories:
  - Go 语言圣经
date: 2019-12-30 16:00:08
---


接口类型是对其它类型行为的抽象和概括；因为接口类型不会和特定的实现细节绑定在一起，通过这种抽象的方式我们可以让我们的函数更加灵活和更具有适应能力。

## 7.1. 接口是合约


我们一直使用两个相似的函数来进行字符串的格式化：`fmt.Printf`，它会把结果写到标准输出，和 `fmt.Sprintf`，它会把结果以字符串的形式返回。得益于使用接口，我们不必可悲的因为返回结果在使用方式上的一些浅显不同就必需把格式化这个最困难的过程复制一份：

```go
package fmt

func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)
func Printf(format string, args ...interface{}) (int, error) {
    return Fprintf(os.Stdout, format, args...)
}
func Sprintf(format string, args ...interface{}) string {
    var buf bytes.Buffer
    Fprintf(&buf, format, args...)
    return buf.String()
}
```

Fprintf 函数中的第一个参数也不是一个文件类型。它是 io.Writer 类型，这是一个接口类型定义如下：

```go
package io

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    // Write writes len(p) bytes from p to the underlying data stream.
    // It returns the number of bytes written from p (0 <= n <= len(p))
    // and any error encountered that caused the write to stop early.
    // Write must return a non-nil error if it returns n < len(p).
    // Write must not modify the slice data, even temporarily.
    //
    // Implementations must not retain p.
    Write(p []byte) (n int, err error)
}
```

这使得 Fprintf 函数能够满足 LSP 里氏替换原则。

除了 `io.Writer` 这个接口类型，Fprintf 和 Fprintln 函数还向类型提供了一种控制它们值输出的途径，给一个类型定义 String 方法，可以让它满足最广泛使用之一的接口类型 `fmt.Stringer`：

```go
package fmt

// The String method is used to print values passed
// as an operand to any format that accepts a string
// or to an unformatted printer such as Print.
type Stringer interface {
    String() string
}
```

## 7.2. 接口类型

我们可以通过组合已有的接口来定义新的接口类型，下面是两个例子：

```go
type ReadWriter interface {
    Reader
    Writer
}
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

## 7.3. 实现接口的条件

一个类型如果拥有一个接口需要的所有方法，那么这个类型就实现了这个接口。

接口类型封装和隐藏具体类型和它的值。即使具体类型有其它的方法，也只有接口类型暴露出来的方法会被调用到：

```go
os.Stdout.Write([]byte("hello"))
os.Stdout.Close()

var w io.Writer
w = os.Stdout
w.Write([]byte("hello"))
w.Close() // compile error: io.Writer lacks Close method
```

这看上去好像没有用，但实际上 `interface {}` 被称为空接口类型是不可或缺的。因为空接口类型对实现它的类型没有要求，所以我们可以将任意一个值赋给空接口类型。

```go
var any interface{}
any = True
any = 12.34
any = "hello"
any = map[]string]int{"one": 1}
any = new(bytes.Buffer)
```

对于创建的一个 `interface {}` 值持有一个 boolean，float，string，map，pointer，或者任意其它的类型；我们当然不能直接对它持有的值做操作，因为 `interface {}` 没有任何方法。

每一个具体类型的组基于它们相同的行为可以表示成一个接口类型。不像基于类的语言，他们一个类实现的接口集合需要进行显式的定义，在 Go 语言中我们可以在需要的时候定义一个新的抽象或者特定特点的组，而不需要修改具体类型的定义。当具体的类型来自不同的作者时这种方式会特别有用。

## 7.4. flag.Value 接口

使用 `flag` 包能够帮助我们快速的解析命令行参数：

```go
var period = flag.Duration("period", 1*time.Second, "sleep period")

func main() {
    flag.Parse()
    fmt.Printf("Sleeping for %v...", *period)
    time.Sleep(*period)
    fmt.Println()
}
```

我们可以这样使用这个程序：

```shell
$ go build gopl.io/ch7/sleep
$ ./sleep
Sleeping for 1s...
$ ./sleep -period 50ms
Sleeping for 50ms...
$ ./sleep -period 2m30s
Sleeping for 2m30s...
$ ./sleep -period 1.5h
Sleeping for 1h30m0s...
$ ./sleep -period "1 day"
invalid value "1 day" for flag -period: time: invalid duration 1 day
```

其实为我们自己的数据类型定义新的标记符号是简单容易的。我们只需要定义一个实现 `flag.Value` 接口的类型，如下：

```go
package flag

// Value is the interface to the value stored in a flag.
type Value interface {
    String() string
    Set(string) error
}
```

String 方法格式化标记的值用在命令行帮助消息中；这样每一个 `flag.Value` 也是一个 `fmt.Stringer`。Set 方法解析它的字符串参数并且更新标记变量的值。实际上，Set 方法和 String 是两个相反的操作，所以最好的办法就是对他们使用相同的注解方式。

让我们定义一个允许通过摄氏度或者华氏温度变换的形式指定温度的 celsiusFlag 类型：

```go
package tempconv

import (
	"flag"
	"fmt"
)

type Celsius float64
type Fahrenheit float64

func CToF(c Celsius) Fahrenheit { return Fahrenheit(c*9.0/5.0 + 32.0) }
func FToC(f Fahrenheit) Celsius { return Celsius((f - 32.0) * 5.0 / 9.0) }

func (c Celsius) String() string { return fmt.Sprintf("%g°C", c) }

type celsiusFlag struct{ Celsius }

func (f *celsiusFlag) Set(s string) error {
	var unit string
	var value float64
	fmt.Sscanf(s, "%f%s", &value, &unit)
	switch unit {
	case "C", "°C":
		f.Celsius = Celsius(value)
		return nil
	case "F", "°F":
		f.Celsius = FToC(Fahrenheit(value))
		return nil
	}
	return fmt.Errorf("invalid temperature %q", s)
}

func CelsiusFlag(name string, value Celsius, usage string) *Celsius {
	f := celsiusFlag{value}
	flag.CommandLine.Var(&f, name, usage)
	return &f.Celsius
}
```

## 7.5. 接口值

概念上讲一个接口的值，接口值，由两个部分组成，一个具体的类型和那个类型的值。它们被称为接口的动态类型和动态值。对于像 Go 语言这种静态类型的语言，类型是编译期的概念；因此一个类型不是一个值。

```go
var w io.Writer
```

在 Go 语言中，变量总是被一个定义明确的值初始化，即使接口类型也不例外。对于一个接口的零值就是它的类型和值的部分都是 nil，如下图所示

{% asset_img interface_nil.png %}

当我们将一个 `*os.File` 类型的变量赋值给 w 时：

```go
w = os.Stdout
```

就会发生一次具体类型到接口类型的隐式转换，这和显式的使用 `io.Writer (os.Stdout)` 是等价的。这类转换不管是显式的还是隐式的，都会刻画出操作到的类型和值。这个接口值的动态类型被设为 `*os.File` 指针的类型描述符，它的动态值持有 `os.Stdout` 的拷贝；这是一个代表处理标准输出的 `os.File` 类型变量的指针：

{% asset_img interface_assignment.png %}

调用一个包含 `*os.File` 类型指针的接口值的 Write 方法，使得 `(*os.File).Write` 方法被调用。这个调用输出 “hello”。

```go
w.Write([]byte("hello")) // "hello"
```

通常在编译期，我们不知道接口值的动态类型是什么，所以一个接口上的调用必须使用动态分配。因为不是直接进行调用，所以编译器必须把代码生成在类型描述符的方法 Write 上，然后间接调用那个地址。这个调用的接收者是一个接口动态值的拷贝，os.Stdout。效果和下面这个直接调用一样：

```go
os.Stdout.Write([]byte("hello")) // "hello"
```

当我们再次把 nil 赋值给接口变量时，它所有的部分都设为 nil 值，变量 w 恢复到和它之前定义时相同的状态。

```go
w = nil
```

一个接口值可以持有任意大的动态值，从概念上讲，不论接口值多大，动态值总是可以容下它。

接口值可以使用 == 和 !＝ 来进行比较。**两个接口值相等仅当它们都是 nil 值，或者它们的动态类型相同并且动态值也根据这个动态类型的 == 操作相等。** 因为接口值是可比较的，所以它们可以用在 map 的键或者作为 switch 语句的操作数。

然而，如果两个接口值的动态类型相同，但是这个动态类型是不可比较的（比如切片），将它们进行比较就会失败并且 panic:

```go
var x interface{} = []int{1, 2, 3}
fmt.Println(x == x) // panic: comparing uncomparable type []int
```

当我们处理错误或者调试的过程中，得知接口值的动态类型是非常有帮助的。所以我们使用 fmt 包的 %T 动作:

```go
var w io.Writer
fmt.Printf("%T\n", w) // "<nil>"
w = os.Stdout
fmt.Printf("%T\n", w) // "*os.File"
w = new(bytes.Buffer)
fmt.Printf("%T\n", w) // "*bytes.Buffer"
```

### 7.5.1. 警告：一个包含 nil 指针的接口不是 nil 接口

一个不包含任何值的 nil 接口值和一个刚好包含 nil 指针的接口值是不同的，思考下面的程序：

思考下面的程序。当 debug 变量设置为 true 时，main 函数会将 f 函数的输出收集到一个 `bytes.Buffer` 类型中。

```go
const debug = true

func main() {
    var buf *bytes.Buffer
    if debug {
        buf = new(bytes.Buffer) // enable collection of output
    }
    f(buf) // NOTE: subtly incorrect!
    if debug {
        // ...use buf...
    }
}

// If out is non-nil, output will be written to it.
func f(out io.Writer) {
    // ...do something...
    if out != nil {
        out.Write([]byte("done!\n"))
    }
}
```

我们可能会预计当把变量 debug 设置为 false 时可以禁止对输出的收集，但是实际上在 `out.Write` 方法调用时程序发生了 panic：


```go
if out != nil {
    out.Write([]byte("done!\n")) // panic: nil pointer dereference
}
```

当 main 函数调用函数 f 时，它给 f 函数的 out 参数赋了一个 `* bytes.Buffer` 的空指针，所以 out 的动态值是 nil。然而，它的动态类型是 `*bytes.Buffer`，意思就是 out 变量是一个包含空指针值的非空接口，如下图，所以防御性检查 `out != nil` 的结果依然是 true。

{% asset_img nil_value_interface.png %}

动态分配机制依然决定 `(*bytes.Buffer).Write` 的方法会被调用，但是这次的接收者的值是 nil。

问题在于尽管一个 nil 的 `*bytes.Buffer` 指针有实现这个接口的方法，它也不满足这个接口具体的行为上的要求。特别是这个调用违反了 `(*bytes.Buffer).Write` 方法的接收者非空的隐含先觉条件，所以将 nil 指针赋给这个接口是错误的。解决方案就是将 main 函数中的变量 buf 的类型改为 `io.Writer`，因此可以避免一开始就将一个不完整的值赋值给这个接口：

```go
var buf io.Writer
if debug {
    buf = new(bytes.Buffer) // enable collection of output
}
f(buf) // OK
```

## 7.6. sort.Interface接口

Go 语言的 sort.Sort 函数不会对具体的序列和它的元素做任何假设。相反，它使用了一个接口类型 sort.Interface 来指定通用的排序算法和可能被排序到的序列类型之间的约定。

```go
package sort

type Interface interface {
    Len() int
    Less(i, j int) bool // i, j are indices of sequence elements
    Swap(i, j int)
}
```

下面是一个对字符串切片实现 `sort.Interface` 的例子：

```go
type StringSlice []string
func (p StringSlice) Len() int           { return len(p) }
func (p StringSlice) Less(i, j int) bool { return p[i] < p[j] }
func (p StringSlice) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }
```

使用方式也很简单：

```go
var s StringSlice = []string{"c", "a", "b"}

sort.Sort(s)
sort.Sort(sort.Reverse(s))
```


另外，值得注意的是，sort 内部实现 Reverse 操作的方式：

```go
package sort

type reverse struct{ Interface } // that is, sort.Interface

func (r reverse) Less(i, j int) bool { return r.Interface.Less(j, i) }

func Reverse(data Interface) Interface { return reverse{data} }
```

尽管对长度为 n 的序列排序需要 O(nlogn) 次比较操作，检查一个序列是否已经有序至少需要 n-1 次比较。sort 包中的 IsSorted 函数帮我们做这样的检查。像 `sort.Sort` 一样，它也使用 `sort.Interface` 对这个序列和它的排序函数进行抽象，但是它从不会调用 Swap 方法：这段代码示范了 IntsAreSorted 和 Ints 函数在 IntSlice 类型上的使用：

```go
values := []int{3, 1, 4, 1}
fmt.Println(sort.IntsAreSorted(values)) // "false"
sort.Ints(values)
fmt.Println(values)                     // "[1 1 3 4]"
fmt.Println(sort.IntsAreSorted(values)) // "true"
sort.Sort(sort.Reverse(sort.IntSlice(values)))
fmt.Println(values)                     // "[4 3 1 1]"
fmt.Println(sort.IntsAreSorted(values)) // "false"
```

## 7.7. http.Handler接口

```go
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```

ListenAndServe 函数需要一个例如 “localhost:8000” 的服务器地址，和一个所有请求都可以分派的 Handler 接口实例。它会一直运行，直到这个服务因为一个错误而失败（或者启动失败），它的返回值一定是一个非空的错误。

## 7.8. error接口

Go 提供了一种预定义的 error 类型，这个类型有一个返回错误信息的单一方法：

```go
type error interface {
    Error() string
}
```

创建一个 error 最简单的方法就是调用 `errors.New` 函数，它会根据传入的错误信息返回一个新的 error。整个 errors 包仅只有4行：

```go
package errors

func New(text string) error { return &errorString{text} }

type errorString struct { text string }

func (e *errorString) Error() string { return e.text }
```

调用 `errors.New` 函数是非常稀少的，因为有一个方便的封装函数 `fmt.Errorf`，它还会处理字符串格式化。我们曾多次在第 5 章中用到它。

```go
package fmt

import "errors"

func Errorf(format string, args ...interface{}) error {
    return errors.New(Sprintf(format, args...))
}
```

## 7.10. 类型断言

类型断言是一个使用在接口值上的操作。语法上它看起来像 `x.(T)` 被称为断言类型，这里 x 表示一个接口的类型和 T 表示一个类型。一个类型断言检查它操作对象的动态类型是否和断言的类型匹配。

这里有两种可能。第一种，如果断言的类型 T 是一个**具体类型**，然后类型断言检查 x 的动态类型是否和 T 相同。如果这个检查成功了，类型断言的结果是 x 的动态值，当然它的类型是 T。换句话说，具体类型的类型断言从它的操作对象中获得具体的值。如果检查失败，接下来这个操作会抛出 panic。例如：

```go
var w io.Writer
w = os.Stdout
f := w.(*os.File)      // success: f == os.Stdout
c := w.(*bytes.Buffer) // panic: interface holds *os.File, not *bytes.Buffer
```

第二种，如果相反地断言的类型 T 是一个**接口类型**，然后类型断言检查是否 x 的动态类型满足 T。如果这个检查成功了，动态值没有获取到；这个结果仍然是一个有相同动态类型和值部分的接口值，但是结果为类型 T。换句话说，对一个接口类型的类型断言改变了类型的表述方式，改变了可以获取的方法集合（通常更大），但是它保留了接口值内部的动态类型和值的部分。

```go
var w io.Writer
w = os.Stdout
rw := w.(io.ReadWriter) // success: *os.File has both Read and Write
w = new(ByteCounter)
rw = w.(io.ReadWriter) // panic: *ByteCounter has no Read method
```

如果断言操作的对象是一个 nil 接口值，那么不论被断言的类型是什么这个类型断言都会失败。

经常地，对一个接口值的动态类型我们是不确定的，并且我们更愿意去检验它是否是一些特定的类型。如果类型断言出现在一个预期有两个结果的赋值操作中，例如如下的定义，这个操作不会在失败的时候发生 panic，但是替代地返回一个额外的第二个结果，这个结果是一个标识成功与否的布尔值：

```go
var w io.Writer = os.Stdout
f, ok := w.(*os.File)      // success:  ok, f == os.Stdout
b, ok := w.(*bytes.Buffer) // failure: !ok, b == nil
```

这个 ok 结果经常立即用于决定程序下面做什么。if 语句的扩展格式让这个变的很简洁：

```go
if f, ok := w.(*os.File); ok {
    // ...use f...
}
```

## 7.12. 通过类型断言询问行为

我们可以定义一个只有某个方法的新接口并且使用类型断言来检测某个接口变量的动态类型是否满足这个新接口：

```go
// writeString writes s to w.
// If w has a WriteString method, it is invoked instead of w.Write.
func writeString(w io.Writer, s string) (n int, err error) {
    type stringWriter interface {
        WriteString(string) (n int, err error)
    }
    if sw, ok := w.(stringWriter); ok {
        return sw.WriteString(s) // avoid a copy
    }
    return w.Write([]byte(s)) // allocate temporary copy
}

func writeHeader(w io.Writer, contentType string) error {
    if _, err := writeString(w, "Content-Type: "); err != nil {
        return err
    }
    if _, err := writeString(w, contentType); err != nil {
        return err
    }
    // ...
}
```

上面的 writeString 函数使用一个类型断言来获知一个普遍接口类型的值是否满足一个更加具体的接口类型；并且如果满足，它会使用这个更具体接口的行为。这个技术可以被很好的使用，不论这个被询问的接口是一个标准如 `io.ReadWriter`，或者用户定义的如 stringWriter 接口。

## 7.13. 类型分支

在最简单的形式中，一个类型分支像普通的 switch 语句一样，它的运算对象是 `x.(type)` —— 它使用了关键词字面量 `type` —— 并且每个 case 有一到多个类型，和普通 switch 语句一样，每一个 case 会被顺序的进行考虑，并且当一个匹配找到时，这个 case 中的内容会被执行。

```go
switch x.(type) {
case nil:       // ...
case int, uint: // ...
case bool:      // ...
case string:    // ...
default:        // ...
}
```

同时，类型分支语句有一个扩展的形式，它可以将提取的值绑定到一个在每个 case 范围内都有效的新变量。

```go
switch x := x.(type) { /* ... */ }
```

## 7.15. 一些建议

当设计一个新的包时，新手 Go 程序员总是先创建一套接口，然后再定义一些满足它们的具体类型。这种方式的结果就是有很多的接口，它们中的每一个仅只有一个实现。不要再这么做了。这种接口是不必要的抽象；它们也有一个运行时损耗。你可以使用导出机制来限制一个类型的方法或一个结构体的字段是否在包外可见。接口只有当有两个或两个以上的具体类型必须以相同的方式进行处理时才需要。

当一个接口只被一个单一的具体类型实现时有一个例外，就是由于它的依赖，这个具体类型不能和这个接口存在在一个相同的包中。这种情况下，一个接口是解耦这两个包的一个好方式。

因为在 Go 语言中只有当两个或更多的类型实现一个接口时才使用接口，它们必定会从任意特定的实现细节中抽象出来。结果就是有更少和更简单方法的更小的接口（经常和 `io.Writer` 或 `fmt.Stringer` 一样只有一个）。当新的类型出现时，小的接口更容易满足。对于接口设计的一个好的标准就是 ask only for what you need（只考虑你需要的东西）

我们完成了对方法和接口的学习过程。Go 语言对面向对象风格的编程支持良好，但这并不意味着你只能使用这一风格。不是任何事物都需要被当做一个对象；独立的函数有它们自己的用处，未封装的数据类型也是这样。