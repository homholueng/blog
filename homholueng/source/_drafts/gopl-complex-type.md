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

**我们并不知道 `append` 调用是否导致了内存的重新分配，因此我们也不能确认新的 slice 和原始的 slice 是否引用的是相同的底层数组空间。同样，我们不能确认在原先的 slice 上的操作是否会影响到新的 slice。** 因此，通常是将 append 返回的结果直接赋值给输入的 slice 变量：

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

## 4.3. Map

在 Go 语言中，一个 map 就是一个哈希表的引用，map 类型可以写为 `map[K]V`，其中 `K` 和 `V` 分别对应 key 和 value。

内置的 make 函数可以创建一个 map：

```go
ages := make(map[string]int) // mapping from strings to ints
```

我们也可以用 map 字面值的语法创建 map，同时还可以指定一些最初的 key/value：

```go
ages := map[string]int{
    "alice":   31,
    "charlie": 34,
}
```

使用内置的 `delete` 函数可以删除元素：

```go
delete(ages, "alice") // remove element ages["alice"]
```

**在 go 的 map 上，如果一个查找失败将返回 value 类型对应的零值。**

但是 map 中的元素并不是一个变量，因此我们不能对 map 的元素进行取址操作：

```go
_ = &ages["bob"] // compile error: cannot take address of map element
```

禁止对 map 元素取址的原因是 map 可能随着元素数量的增长而重新分配更大的内存空间，从而可能导致之前的地址无效。

要想遍历 map 中全部的 key/value 对的话，可以使用 range 风格的 for 循环实现，和之前的 slice 遍历语法类似。下面的迭代语句将在每次迭代时设置 name 和 age 变量，它们对应下一个键 / 值对：

```go
for name, age := range ages {
    fmt.Printf("%s\t%d\n", name, age)
}
```

如果要按顺序遍历 key/value 对，我们必须显式地对 key 进行排序，可以使用 sort 包的 Strings 函数对字符串 slice 进行排序。下面是常见的处理方式：

```go
import "sort"

var names []string
for name := range ages {
    names = append(names, name)
}
sort.Strings(names)
for _, name := range names {
    fmt.Printf("%s\t%d\n", name, ages[name])
}
```

因为我们一开始就知道 names 的最终大小，因此给 slice 分配一个合适的大小将会更有效。下面的代码创建了一个空的 slice，但是 slice 的容量刚好可以放下 map 中全部的 key：

```go
names := make([]string, 0, len(ages)) // type, len, cap
```

map 类型的零值是 `nil`，也就是没有引用任何哈希表。

```go
var ages map[string]int
fmt.Println(ages == nil)    // "true"
fmt.Println(len(ages) == 0) // "true"
```

map 上的大部分操作，包括查找、删除、`len` 和 `range` 循环都可以安全工作在 `nil` 值的 map 上，它们的行为和一个空的 map 类似。但是向一个 `nil` 值的 map 存入元素将导致一个 panic 异常：

```go
ages["carol"] = 21 // panic: assignment to entry in nil map
```

如果我们需要确认 map 取值返回的究竟是不存在的零值还是存在的零值，可以使用如下的方式：

```go
age, ok := ages["bob"]
if !ok { /* "bob" is not a key in this map; age == 0. */ }

// or

if age, ok := ages["bob"]; !ok { /* ... */ }
```

和 slice 一样，map 之间也不能进行相等比较；唯一的例外是和 `nil` 进行比较。要判断两个 map 是否包含相同的 key 和 value，我们必须通过一个循环实现。

Go 语言中并没有提供一个 set 类型，但是 map 中的 key 也是不相同的，可以用 map 实现类似 set 的功能。

有时候我们需要一个 map 或 set 的 key 是 slice 类型，但是 map 的 key 必须是可比较的类型，但是 slice 并不满足这个条件。不过，我们可以通过两个步骤绕过这个限制。第一步，定义一个辅助函数 `k`，将 slice 转为 map 对应的 string 类型的 key，确保只有 `x` 和 `y` 相等时 `k (x) == k (y)` 才成立。然后创建一个 key 为 string 类型的 map，在每次对 map 操作时先用 `k` 辅助函数将 slice 转化为 string 类型。

## 4.4. 结构体

结构体是一种聚合的数据类型，是由零个或多个任意类型的值聚合成的实体。

下面两个语句声明了一个叫 Employee 的命名的结构体类型，并且声明了一个 Employee 类型的变量 `dilbert`：

```go
type Employee struct {
    ID        int
    Name      string
    Address   string
    DoB       time.Time
    Position  string
    Salary    int
    ManagerID int
}

var dilbert Employee
```

`dilbert` 结构体变量的成员可以通过点操作符访问，比如 `dilbert.Name` 和 `dilbert.DoB`。

点操作符也可以和指向结构体的指针一起工作：

```go
var employeeOfTheMonth *Employee = &dilbert
employeeOfTheMonth.Position += " (proactive team player)"
```

相当于下面语句

```go
(*employeeOfTheMonth).Position += " (proactive team player)"
```

下面的 `EmployeeByID` 函数将根据给定的员工 ID 返回对应的员工信息结构体的指针。我们可以使用点操作符来访问它里面的成员：

```go
func EmployeeByID(id int) *Employee { /* ... */ }

fmt.Println(EmployeeByID(dilbert.ManagerID).Position) // "Pointy-haired boss"

id := dilbert.ID
EmployeeByID(id).Salary = 0 // fired for... no real reason
```

后面的语句通过 `EmployeeByID` 返回的结构体指针更新了 `Employee` 结构体的成员。如果将 `EmployeeByID` 函数的返回值从 `*Employee` 指针类型改为 `Employee` 值类型，那么更新语句将不能编译通过，因为在赋值语句的左边并不确定是一个变量（译注：调用函数返回的是值，并不是一个可取地址的变量）。

通常一行对应一个结构体成员，成员的名字在前类型在后，不过如果相邻的成员类型如果相同的话可以被合并到一行，就像下面的 Name 和 Address 成员那样：

```go
type Employee struct {
    ID            int
    Name, Address string
    DoB           time.Time
    Position      string
    Salary        int
    ManagerID     int
}
```

**如果结构体成员名字是以大写字母开头的，那么该成员就是导出的；这是 Go 语言导出规则决定的。一个结构体可能同时包含导出和未导出的成员。**

结构体类型的零值是每个成员都是零值。通常会将零值作为最合理的默认值。

如果结构体没有任何成员的话就是空结构体，写作 `struct{}`。它的大小为 0，也不包含任何信息，但是有时候依然是有价值的。有些 Go 语言程序员用 map 来模拟 set 数据结构时，用它来代替 map 中布尔类型的 value，只是强调 key 的重要性，但是因为节约的空间有限，而且语法比较复杂，所以我们通常会避免这样的用法。

```go
seen := make(map[string]struct{}) // set of strings
// ...
if _, ok := seen[s]; !ok {
    seen[s] = struct{}{}
    // ...first time seeing s...
}
```

### 4.4.1. 结构体字面值

结构体值也可以用结构体字面值表示，结构体字面值可以指定每个成员的值。


```go
type Point struct{ X, Y int }

p := Point{1, 2}
```

这里有两种形式的结构体字面值语法，上面的是第一种写法，要求以结构体成员定义的顺序为每个结构体成员指定一个字面值。它要求写代码和读代码的人要记住结构体的每个成员的类型和顺序，不过结构体成员有细微的调整就可能导致上述代码不能编译。

其实更常用的是第二种写法，以成员名字和相应的值来初始化：

```go
p := Point{X: 1, Y: 2}
```

在这种形式的结构体字面值写法中，如果成员被忽略的话将默认用零值。另外，需要注意的是，两种不同形式的写法不能混合使用。

因为结构体通常通过指针处理，可以用下面的写法来创建并初始化一个结构体变量，并返回结构体的地址：

```go
pp := &Point{1, 2}
```

它和下面的语句是等价的

```go
pp := new(Point)
*pp = Point{1, 2}
```

不过 `&Point{1, 2}` 写法可以直接在表达式中使用，比如一个函数调用。

### 4.4.2. 结构体比较

如果结构体的全部成员都是可以比较的，那么结构体也是可以比较的，那样的话两个结构体将可以使用 `==` 或 `=` 运算符进行比较。

可比较的结构体类型和其他可比较的类型一样，可以用于 map 的 key 类型。

### 4.4.3. 结构体嵌入和匿名成员

Go 语言有一个特性让我们只声明一个成员对应的数据类型而不指名成员的名字；这类成员就叫匿名成员。匿名成员的数据类型必须是命名的类型或指向一个命名的类型的指针。下面的代码中，Circle 和 Wheel 各自都有一个匿名成员。我们可以说 Point 类型被嵌入到了 Circle 结构体，同时 Circle 类型被嵌入到了 Wheel 结构体。

```go
type Circle struct {
    Point
    Radius int
}

type Wheel struct {
    Circle
    Spokes int
}
```

得益于匿名嵌入的特性，我们可以直接访问叶子属性而不需要给出完整的路径：

```go
var w Wheel
w.X = 8            // equivalent to w.Circle.Point.X = 8
w.Y = 8            // equivalent to w.Circle.Point.Y = 8
w.Radius = 5       // equivalent to w.Circle.Radius = 5
w.Spokes = 20
```

其中匿名成员 Circle 和 Point 都有自己的名字 —— 就是命名的类型名字 —— 但是这些名字在点操作符中是可选的。我们在访问子成员的时候可以忽略任何匿名成员部分。

不幸的是，结构体字面值并没有简短表示匿名成员的语法， 因此下面的语句都不能编译通过：

```go
w = Wheel{8, 8, 5, 20}                       // compile error: unknown fields
w = Wheel{X: 8, Y: 8, Radius: 5, Spokes: 20} // compile error: unknown fields
```

结构体字面值必须遵循形状类型声明时的结构，所以我们只能用下面的两种语法，它们彼此是等价的：

```go
w = Wheel{Circle{Point{8, 8}, 5}, 20}

w = Wheel{
    Circle: Circle{
        Point:  Point{X: 8, Y: 8},
        Radius: 5,
    },
    Spokes: 20, // NOTE: trailing comma necessary here (and at Radius)
}

fmt.Printf("%#v\n", w)
// Output:
// Wheel{Circle:Circle{Point:Point{X:8, Y:8}, Radius:5}, Spokes:20}

w.X = 42

fmt.Printf("%#v\n", w)
// Output:
// Wheel{Circle:Circle{Point:Point{X:42, Y:8}, Radius:5}, Spokes:20}
```

*需要注意的是 `Printf` 函数中 `%v` 参数包含的 `#` 副词，它表示用和 Go 语言类似的语法打印值。对于结构体类型来说，将包含每个成员的名字。*

**因为匿名成员也有一个隐式的名字，因此不能同时包含两个类型相同的匿名成员，这会导致名字冲突。同时，因为成员的名字是由其类型隐式地决定的，所以匿名成员也有可见性的规则约束。** 在上面的例子中，Point 和 Circle 匿名成员都是导出的。即使它们不导出（比如改成小写字母开头的 point 和 circle），我们依然可以用简短形式访问匿名成员嵌套的成员

**但是在包外部，因为 circle 和 point 没有导出，不能访问它们的成员，因此简短的匿名成员访问语法也是禁止的。**

其实任何命名的类型都可以作为结构体的匿名成员。但是为什么要嵌入一个没有任何子成员类型的匿名成员类型呢？

答案是匿名类型的方法集。**简短的点运算符语法可以用于选择匿名成员嵌套的成员，也可以用于访问它们的方法。实际上，外层的结构体不仅仅是获得了匿名成员类型的所有成员，而且也获得了该类型导出的全部的方法。这个机制可以用于将一些有简单行为的对象组合成有复杂行为的对象。**

> 组合是 Go 语言中面向对象编程的核心

## 4.5. JSON

