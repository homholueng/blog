---
title: Go 语言圣经 - 4. 复合数据类型
tags:
  - Go
categories:
  - Go 语言圣经
---

## 4.1. 数组

数组是一个由固定长度的特定类型元素组成的序列，一个数组可以由零个或多个元素组成。因为数组的长度是固定的，因此在Go语言中很少直接使用数组。

默认情况下，数组的每个元素都被初始化为元素类型对应的零值：

```go
var a = [3]int  // array of 3 intergers with 0 value
```

我们也可以使用数组字面值语法用一组值来初始化数组：

```go
var q [3]int = [3]int{1, 2, 3}
var r [3]int = [3]int{1, 2}
fmt.Println(r[2]) // 0
```

在数组字面值中，如果在数组的长度位置出现的是 “...” 省略号，则表示数组的长度是根据初始化值的个数来计算：

```go
q := [...]int{1, 2, 3}
```

数组的长度是数组类型的一个组成部分，因此 `[3]int` 和 `[4]int` 是两种不同的数组类型。

数组也可以指定一个索引和对应值列表的方式初始化，在这种形式的数组字面值形式中，初始化索引的顺序是无关紧要的，而且没用到的索引可以省略，和前面提到的规则一样，未指定初始值的元素将用零值初始化。

```go
type Currency int

const (
    USD Currency = iota // 美元
    EUR                 // 欧元
    GBP                 // 英镑
    RMB                 // 人民币
)

symbol := [...]string{USD: "$", EUR: "€", GBP: "￡", RMB: "￥"}

fmt.Println(RMB, symbol[RMB]) // "3 ￥"
```

如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的，这时候我们可以直接通过 `==` 比较运算符来比较两个数组，只有当两个数组的所有元素都是相等的时候数组才是相等的。

**注意，GO 并不会隐式的将传递给函数参数的数组转换成为指针，如果函数需要接收一个数组并修改其内容，则需要显示的将其参数类型声明为数组指针。**

因为数组的类型包含了僵化的长度信息，并且不同长度的数组也被认作不同的类型，所以 GO 中很少直接使用数组，而是使用 `slice`。

## 4.2. Slice

Slice（切片）代表变长的序列，序列中每个元素都有相同的类型。一个 slice 类型一般写作 `[]T`，其中 `T` 代表 slice 中元素的类型；slice 的语法和数组很像，只是没有固定长度而已。

数组和 slice 之间有着紧密的联系。一个 slice 是一个轻量级的数据结构，提供了访问数组子序列（或者全部）元素的功能，而且 slice 的底层确实引用一个数组对象。

**一个 slice 由三个部分构成：指针、长度和容量。**

下图显示了表示一年中每个月份名字的字符串数组，还有重叠引用了该数组的两个 slice。数组这样定义：

```go
months := [...]string{1: "January", /* ... */, 12: "December"}
```

{% asset_img slice_on_array.png [切片示例] %}

slice 的切片操作 `s[i:j]`，其中 `0 ≤ i≤ j≤ cap(s)`，用于创建一个新的 slice，引用 `s` 的从第 `i` 个元素开始到第 `j-1` 个元素的子序列。

如果切片操作超出 `cap(s)` 的上限将导致一个 panic 异常，但是超出 `len(s)` 则是意味着扩展了 slice，因为新 slice 的长度会变大：

```go
a := [...]int{1, 2, 3, 4, 5}
s := a[0:2]
fmt.Println(s) // [1 2]
fmt.Printf("len: %d, cap: %d\n", len(s), cap(s))
new_s := s[:4] // len: 2, cap: 5
fmt.Printf("len: %d, cap: %d\n", len(new_s), cap(new_s)) // len: 4, cap: 5
```

因为 slice 值包含指向第一个 slice 元素的指针，因此向函数传递 slice 将允许在函数内部修改底层数组的元素。

要注意的是 slice 类型的变量 s 和数组类型的变量 a 的初始化语法的差异。slice 和数组的字面值语法很类似，它们都是用花括弧包含一系列的初始化元素，但是对于 slice 并没有指明序列的长度。**这会隐式地创建一个合适大小的数组，然后 slice 的指针指向底层的数组**。就像数组字面值一样，slice 的字面值也可以按顺序指定初始化值序列，或者是通过索引和元素值指定，或者用两种风格的混合语法初始化。

和数组不同的是，slice 之间不能比较，因此我们不能使用 `==` 操作符来判断两个 slice 是否含有全部相等元素。

**slice 唯一合法的比较操作是和 nil 比较，如果你需要测试一个 slice 是否是空的，使用 len (s) == 0 来判断，而不应该用 s == nil 来判断。**

```go
var s []int    // len(s) == 0, s == nil
s = nil        // len(s) == 0, s == nil
s = []int(nil) // len(s) == 0, s == nil
s = []int{}    // len(s) == 0, s != nil
```

内置的 `make` 函数创建一个指定元素类型、长度和容量的 slice。容量部分可以省略，在这种情况下，容量将等于长度。

```go
make([]T, len)
make([]T, len, cap) // same as make([]T, cap)[:len]
```

### 4.2.1. append 函数

内置的 `append` 函数用于向 slice 追加元素：

```
var runes []rune
for _, r := range "Hello, 世界" {
    runes = append(runes, r)
}
fmt.Printf("%q\n", runes) // "['H' 'e' 'l' 'l' 'o' ',' ' ' '世' '界']"
```

内置的 copy 函数可以方便地将一个 slice 复制另一个相同类型的 slice，`copy` 函数的第一个参数是要复制的目标 slice，第二个参数是源 slice，目标和源的位置顺序和 `dst = src` 赋值语句是一致的。

我们并不知道 `append` 调用是否导致了内存的重新分配，因此我们也不能确认新的 slice 和原始的 slice 是否引用的是相同的底层数组空间。同样，我们不能确认在原先的 slice 上的操作是否会影响到新的 slice。因此，通常是将 append 返回的结果直接赋值给输入的 slice 变量：

```go
runes = append(runes, r)
```

`append` 函数则可以追加多个元素，甚至追加一个 slice。

```go
var x []int
x = append(x, 1)
x = append(x, 2, 3)
x = append(x, 4, 5, 6)
x = append(x, x...) // append the slice x
fmt.Println(x)      // "[1 2 3 4 5 6 1 2 3 4 5 6]"
```

### 4.2.2. Slice 内存技巧

 给定一个字符串列表，下面的 `nonempty` 函数将在原有 slice 内存空间之上返回不包含空字符串的列表：

 ```go
 // Nonempty is an example of an in-place slice algorithm.
package main

import "fmt"

// nonempty returns a slice holding only the non-empty strings.
// The underlying array is modified during the call.
func nonempty(strings []string) []string {
    i := 0
    for _, s := range strings {
        if s != "" {
            strings[i] = s
            i++
        }
    }
    return strings[:i]
}
 ```

 比较微妙的地方是，输入的 slice 和输出的 slice 共享一个底层数组。这可以避免分配另一个数组，不过原来的数据将可能会被覆盖，正如下面两个打印语句看到的那样：

```go
data := []string{"one", "", "three"}
fmt.Printf("%q\n", nonempty(data)) // `["one" "three"]`
fmt.Printf("%q\n", data)           // `["one" "three" "three"]`
```

因此我们通常会这样使用 `nonempty` 函数：`data = nonempty(data)`。

