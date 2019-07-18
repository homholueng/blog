---
title: Go 语言圣经 - 1. 入门
categories: 
- Go 语言圣经
tags:
- Go
---

## 1.2. 命令行参数

通过 `os.Args` 切片能够拿到程序启动时得到的命令行参数：

```go
package main

import (
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