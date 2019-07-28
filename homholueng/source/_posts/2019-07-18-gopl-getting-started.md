---
title: Go 语言圣经 - 1. 入门
tags:
  - Go
categories:
  - Go 语言圣经
date: 2019-07-18 21:59:05
---


## 1.2. 命令行参数

通过 `os.Args` 切片能够拿到程序启动时得到的命令行参数：

```go
package main

import 
	"fmt"
	"os"
)

func main() {
	var s, sep string
	for _, arg := range os.Args {
		s += sep + arg
		sep = " "
	}
	fmt.Println(s)
}
```

运行以下命令会输出：

```shell
$ go run echo1.go 1 2 3 4 5
/var/folders/8v/11x3hg3x5g758rwj8kl4c42w0000gn/T/go-build732224661/b001/exe/echo1 1 2 3 4 5
```

## 1.3. 查找重复的行

类似于 C 或其它语言里的 `printf` 函数，`fmt.Printf` 函数对一些表达式产生格式化输出。该函数的首个参数是个格式字符串，指定后续参数被如何格式化，`Printf` 有一大堆这种转换，Go 程序员称之为动词（verb）：

```text
%d          十进制整数
%x, %o, %b  十六进制，八进制，二进制整数。
%f, %g, %e  浮点数： 3.141593 3.141592653589793 3.141593e+00
%t          布尔：true或false
%c          字符（rune） (Unicode码点)
%s          字符串
%q          带双引号的字符串"abc"或带单引号的字符'c'
%v          变量的自然形式（natural format）
%T          变量的类型
%%          字面上的百分号标志（无操作数）
```

## 1.8. 本章要点

在你开始写一个新程序之前，最好先去检查一下是不是已经有了现成的库可以帮助你更高效地完成这件事情。你可以在 [https://golang.org/pkg](https://golang.org/pkg) 和 [https://godoc.org](https://godoc.org) 中找到标准库和社区写的 package。godoc 这个工具可以让你直接在本地命令行阅读标准库的文档。比如下面这个例子。

```shell
$ go doc http.ListenAndServe
package http // import "net/http"
func ListenAndServe(addr string, handler Handler) error
    ListenAndServe listens on the TCP network address addr and then
    calls Serve with handler to handle requests on incoming connections.
...
```

