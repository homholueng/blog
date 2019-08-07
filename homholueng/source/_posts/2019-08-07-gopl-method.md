---
title: Go 语言圣经 - 6. 方法
tags:
  - Go
categories:
  - Go 语言圣经
date: 2019-08-07 20:02:28
---


## 6.1. 方法声明

在函数声明时，在其名字之前放上一个变量，即是一个方法。这个附加的参数会将该函数附加到这种类型上，即相当于为这种类型定义了一个独占的方法。

```go
package geometry

import "math"

type Point struct{ X, Y float64 }

// traditional function
func Distance(p, q Point) float64 {
    return math.Hypot(q.X-p.X, q.Y-p.Y)
}

// same thing, but as a method of the Point type
func (p Point) Distance(q Point) float64 {
    return math.Hypot(q.X-p.X, q.Y-p.Y)
}
```

上面的代码里那个附加的参数 `p`，叫做方法的接收器（receiver），早期的面向对象语言留下的遗产将调用一个方法称为 “向一个对象发送消息”。

在 Go 语言中，我们并不会像其它语言那样用 this 或者 self 作为接收器；我们可以任意的选择接收器的名字。由于接收器的名字经常会被使用到，所以保持其在方法间传递时的一致性和简短性是不错的主意。这里的建议是可以使用其类型的第一个字母，比如这里使用了 Point 的首字母 `p`。

而每种类型都有其方法的命名空间，让我们来定义一个 Path 类型，这个 Path 代表一个线段的集合，并且也给这个 Path 定义一个叫 `Distance` 的方法。

```go
type path []Point

func (path Path) Distance() float64 {
    sum := 0.0
    for i := range path {
        if i > 0 {
            sum += path[i-1].Distance(path[i])
        }
    }
    return sum
}

```

在能够给任意类型定义方法这一点上，Go 和很多其它的面向对象的语言不太一样。因此在 Go 语言里，我们为一些简单的数值、字符串、slice、map 来定义一些附加行为很方便。我们可以给同一个包内的任意命名类型定义方法，只要这个命名类型的底层类型不是指针或者 interface。

## 6.2. 基于指针对象的方法

当调用一个函数时，会对其每一个参数值进行拷贝，如果一个函数需要更新一个变量，或者函数的其中一个参数实在太大我们希望能够避免进行这种默认的拷贝，这种情况下我们就需要用到指针了。对应到我们这里用来更新接收器的对象的方法，当这个接受者变量本身比较大时，我们就可以用其指针而不是对象来声明方法，如下：

```go
func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}
```

这个方法的名字是 `(*Point).ScaleBy`。在现实的程序里，一般会约定如果 Point 这个类有一个指针作为接收器的方法，那么所有 Point 的方法都必须有一个指针接收器，即使是那些并不需要这个指针接收器的函数。

只有类型（Point）和指向他们的指针 (`*Point`)，才可能是出现在接收器声明里的两种接收器。此外，为了避免歧义，在声明方法时，如果一个类型名本身是一个指针的话，是不允许其出现在接收器中的，比如下面这个例子：

```go
type P *int
func (P) f() { /* ... */ } // compile error: invalid receiver type
```
想要调用指针类型方法 `(*Point).ScaleBy`，只要提供一个 Point 类型的指针即可，像下面这样。

```go
r := &Point{1, 2}
r.ScaleBy(2)

p := Point{1, 2}
pptr := &p
pptr.ScaleBy(2)

p := Point{1, 2}
(&p).ScaleBy(2)

```

但是，go 语言本身在这种地方会帮到我们。如果接收器 `p` 是一个 Point 类型的变量，并且其方法需要一个 Point 指针作为接收器，我们可以用下面这种简短的写法：

```go
p.ScaleBy(2)
```

编译器会隐式地帮我们用 `&p` 去调用 `ScaleBy` 这个方法。这种简写方法只适用于 “变量”，包括 struct 里的字段比如 `p.X`，以及 array 和 slice 内的元素比如 `perim[0]`。我们不能通过一个无法取到地址的接收器来调用指针方法，比如临时变量的内存地址就无法获取得到：


```go
Point{1, 2}.ScaleBy(2) // compile error: can't take address of Point literal
```

go 的编译器会隐式为我们 **解引用** 或 **取地址**。、

### 6.2.1. Nil 也是一个合法的接收器类型

就像一些函数允许 nil 指针作为参数一样，方法理论上也可以用 nil 指针作为其接收器，尤其当 nil 对于对象来说是合法的零值时，比如 map 或者 slice。

当你定义一个允许 nil 作为接收器值的方法的类型时，在类型前面的注释中指出 nil 变量代表的意义是很有必要的。

## 6.3. 通过嵌入结构体来扩展类型

来看看 ColoredPoint 这个类型：

```go
import "image/color"

type Point struct{ X, Y float64 }

type ColoredPoint struct {
    Point
    Color color.RGBA
}
```

我们通过类型为 Point 的匿名成员将 Point 类的方法也引入了 ColoredPoint 中。这种方式可以使我们定义字段特别多的复杂类型，我们可以将字段先按小类型分组，然后定义小类型的方法，之后再把它们组合起来。

在类型中内嵌的匿名字段也可能是一个命名类型的指针，这种情况下字段和方法会被间接地引入到当前的类型中。

方法只能在命名类型（像 Point）或者指向类型的指针上定义，但是多亏了内嵌，有些时候我们给匿名 struct 类型来定义方法也有了手段。

下面是一个小 trick。这个例子展示了简单的 cache：

```go
var cache = struct {
    sync.Mutex
    mapping map[string]string
}{
    mapping: make(map[string]string),
}


func Lookup(key string) string {
    cache.Lock()
    v := cache.mapping[key]
    cache.Unlock()
    return v
}
```

## 6.4. 方法值和方法表达式

我们经常选择一个方法，并且在同一个表达式里执行，比如常见的 `p.Distance()` 形式，实际上将其分成两步来执行也是可能的。

```go
p := Point{1, 2}
q := Point{4, 6}

distanceFromP := p.Distance        // method value
fmt.Println(distanceFromP(q))      // "5"
var origin Point                   // {0, 0}
fmt.Println(distanceFromP(origin)) // "2.23606797749979", sqrt(5)
```

我们将 `distanceFromP` 称为 `Distance` 绑定了接受者 `p` 上的方法值。

和方法 “值” 相关的还有方法表达式：

```go
p := Point{1, 2}
q := Point{4, 6}

distance := Point.Distance   // method expression
fmt.Println(distance(p, q))  // "5"
```

## 6.6. 封装

封装提供了三方面的优点。首先，因为调用方不能直接修改对象的变量值，其只需要关注少量的语句并且只要弄懂少量变量的可能的值即可。

第二，隐藏实现的细节，可以防止调用方依赖那些可能变化的具体实现，这样使设计包的程序员在不破坏对外的 api 情况下能得到更大的自由。

把 `bytes.Buffer` 这个类型作为例子来考虑。这个类型在做短字符串叠加的时候很常用，所以在设计的时候可以做一些预先的优化，比如提前预留一部分空间，来避免反复的内存分配。又因为 Buffer 是一个 struct 类型，这些额外的空间可以用附加的字节数组来保存，且放在一个小写字母开头的字段中。这样在外部的调用方只能看到性能的提升，但并不会看得到这个附加变量。

```go
type Buffer struct {
    buf     []byte
    initial [64]byte
    /* ... */
}
```

封装的第三个优点也是最重要的优点，是阻止了外部调用方对对象内部的值任意地进行修改。

封装并不总是理想的。 虽然封装在有些情况是必要的，但有时候我们也需要暴露一些内部内容，比如：`time.Duration` 将其表现暴露为一个 `int64` 数字的纳秒，使得我们可以用一般的数值操作来对时间进行对比，甚至可以定义这种类型的常量：

```go
const day = 24 * time.Hour
fmt.Println(day.Seconds()) // "86400"
```

