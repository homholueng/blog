---
title: go-tips
tags:
---

## 字符串替换

在 Go 中，`strings` 包提供了用于替换字符串中特定字符的便利函数和类：

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
    fmt.Println(strings.Replace("oink oink oink", "k", "ky", 2)) // output: oinky oinky oink
    fmt.Println(strings.Replace("oink oink oink", "k", "ky", -1)) // output: oinky oinky oinky
    fmt.Println(strings.ReplaceAll("oink oink oink", "k", "ky")) // output: oinky oinky oinky

}
```
`Replace` 及 `ReplaceAll` 函数都能够用于替换字符串中特定字符的场景，但是这两个函数一次只能替换一个特定的模式，如果要进行多种模式的替换，就需要用到 `strings` 中的 `Replacer` 类：

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	replacer := strings.NewReplacer("one", "1", "two", "2")
	fmt.Println(replacer.Replace("one 2 two one"))
}
```

`NewReplacer` 接受 `old, new, ...` 模式的参数，返回一个用于查找并替换多种匹配替换模式的 `Replacer`。

