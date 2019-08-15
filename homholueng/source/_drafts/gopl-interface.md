---
title: Go 语言圣经 - 7. 接口
tags:
  - Go
categories:
  - Go 语言圣经
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

