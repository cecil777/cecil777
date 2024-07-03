Duck Typing，鸭子类型，在维基百科里是这样定义的：

``` md
If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck.
```

翻译过来就是：如果某个东西长得像鸭子，游泳像鸭子，嘎嘎叫像鸭子，那它就可以被看成是一只鸭子。

它是动态编程语言的一种对象推断策略，它更关注对象能做什么，而不是对象的类型本身。

例如：在动态语言 Python 中，定义一个这样的函数：

``` python
def hello_world(duck):
    duck.say_hello()
```

当调用此函数的时候，可以传入任意类型，只要它实现了 ```say_hello()``` 就可以。如果没实现，运行过程中会出现错误。

Go 语言作为一门静态语言，它通过接口的方式完美支持鸭子类型。

### 接口类型

之前介绍的类型都是具体类型，而接口是一种抽象类型，是多个方法声明的集合。在 Go 中，只要目标类型实现了接口要求的所有方法，我们就说它实现了这个接口。

先来看一个例子：

``` go
package main

import "fmt"

// 定义接口，包含 Eat 方法
type Duck interface {
    Eat()
}

// 定义 Cat 结构体，并实现 Eat 方法
type Cat struct{}

func (c *Cat) Eat() {
    fmt.Println("cat eat")
}

// 定义 Dog 结构体，并实现 Eat 方法
type Dog struct{}

func (d *Dog) Eat() {
    fmt.Println("dog eat")
}

func main() {
    var c Duck = &Cat{}
    c.Eat()

    var d Duck = &Dog{}
    d.Eat()

    s := []Duck{
        &Cat{},
        &Dog{},
    }
    for _, n := range s {
        n.Eat()
    }
}
```

使用 ```type``` 关键词定义接口

``` go
type Duck interface {
    Eat()
}
```

接口包含了一个 ```Eat()``` 方法，然后定义两个结构体类型 ```Cat``` 和 ```Dog```，分别实现了 ```Eat``` 方法。

``` go
// 定义 Cat 结构体，并实现 Eat 方法
type Cat struct{}

func (c *Cat) Eat() {
    fmt.Println("cat eat")
}

// 定义 Dog 结构体，并实现 Eat 方法
type Dog struct{}

func (d *Dog) Eat() {
    fmt.Println("dog eat")
}
```

遍历接口切片，通过接口类型可以直接调用对应方法：

``` go
s := []Duck{
    &Cat{},
    &Dog{},
}
for _, n := range s {
    n.Eat()
}

// 输出
// cat eat
// dog eat
```

### 接口赋值

接口赋值分两种情况：

1. 将对象实例赋值给接口
2. 将一个接口赋值给另一个接口

下面来分别说说：

**将对象实例赋值给接口**

还是用上面的例子，因为 ```Cat``` 实现了 ```Eat``` 接口，所以可以直接将 ```Cat``` 实例赋值给接口。

``` go
var c Duck = &Cat{}
c.Eat()
```

在这里一定要传结构体指针，如果直接传结构体会报错：

``` go
var c Duck = Cat{}
c.Eat()
# command-line-arguments
./09_interface.go:25:6: cannot use Cat{} (type Cat) as type Duck in assignment:
    Cat does not implement Duck (Eat method has pointer receiver)
```

但是如果反过来呢？比如使用结构体来实现接口，使用结构体指针来赋值：

``` go
// 定义 Cat 结构体，并实现 Eat 方法
type Cat struct{}

func (c Cat) Eat() {
    fmt.Println("cat eat")
}

var c Duck = &Cat{}
c.Eat() // cat eat
```

没有问题，可以正常执行。

**将一个接口赋值给另一个接口**

还是上面的例子，可以直接将 ```c``` 的值直接赋值给 ```d```：

``` go
var c Duck = &Cat{}
c.Eat()

var d Duck = c
d.Eat()
```

再来，我再定义一个接口 ```Duck1```，这个接口包含两个方法 ```Eat``` 和 ```Walk```，然后结构体 ```Dog``` 实现两个方法，但是 ```Cat``` 只实现 ```Eat``` 方法。

``` go
type Duck1 interface {
    Eat()
    Walk()
}

// 定义 Dog 结构体，并实现 Eat 方法
type Dog struct{}

func (d *Dog) Eat() {
    fmt.Println("dog eat")
}

func (d *Dog) Walk() {
    fmt.Println("dog walk")
}
```

那么在赋值时，使用 ```Duck1``` 赋值给 ```Duck``` 是可以的，反过来就会报错。

``` go
var c1 Duck1 = &Dog{}
var c2 Duck = c1
c2.Eat()
```

所以，已经初始化的接口变量 ```c1``` 直接赋值给另一个接口变量 ```c2```，要求 ```c2``` 的方法集是 ```c1``` 的方法集的子集。

### 空接口

具有 0 个方法的接口称为空接口，它表示为 ```interface {}```。由于空接口有 0 个方法，所以所有类型都实现了空接口。

``` go
func main() {
    // interface 形参
    s1 := "Hello World"
    i := 50
    strt := struct {
        name string
    }{
        name: "AlwaysBeta",
    }
    test(s1)
    test(i)
    test(strt)
}

func test(i interface{}) {
    fmt.Printf("Type = %T, value = %v\n", i, i)
}
```

### 类型断言

类型断言是作用在接口值上的操作，语法如下：

``` go
x.(T)
```

其中 ```x``` 是接口类型的表达式，```T``` 是断言类型。

作用是判断操作数的动态类型是否满足指定的断言类型。

有两种情况：

1. ```T``` 是具体类型
2. ```T``` 是接口类型

下面来分别举例说明：

**具体类型**

类型断言会检查 ```x``` 的动态类型是否为 ```T```，如果是，则输出 ```x``` 的值；如果不是，程序直接 ```panic```。

``` go
func main() {
    // 类型断言
    var n interface{} = 55
    assert(n) // 55
    var n1 interface{} = "hello"
    assert(n1) // panic: interface conversion: interface {} is string, not int
}

func assert(i interface{}) {
    s := i.(int)
    fmt.Println(s)
}
```

**接口类型**

类型断言会检查 ```x``` 的动态类型是否满足接口类型 ```T```，如果满足，则输出 ```x``` 的值，这个值可能是绑定实例的副本，也可能是指针的副本；如果不满足，程序直接 ```panic```。

``` go
func main() {
    // 类型断言
    assertInterface(c) // &{}
}

func assertInterface(i interface{}) {
    s := i.(Duck)
    fmt.Println(s)
}
```

如果有两个接收值，那么断言不会在失败时崩溃，而是会多返回一个布尔值，一般命名为 ```ok```，来表示断言是否成功。

``` go
func main() {
    // 类型断言
    var n1 interface{} = "hello"
    assertFlag(n1)
}

func assertFlag(i interface{}) {
    if s, ok := i.(int); ok {
        fmt.Println(s)
    }
}
```

### 类型查询

语法类似类型断言，只需将 ```T``` 直接用关键词 ```type``` 替代。

作用主要有两个：

1. 查询一个接口变量绑定的底层变量类型
2. 查询一个接口变量的底层变量是否还实现了其他接口

``` go
func main() {
    // 类型查询
    SearchType(50)         // Int: 50
    SearchType("zhangsan") // String: zhangsan
    SearchType(c)          // dog eat
    SearchType(50.1)       // Unknown type
}

func SearchType(i interface{}) {
    switch v := i.(type) {
    case string:
        fmt.Printf("String: %s\n", i.(string))
    case int:
        fmt.Printf("Int: %d\n", i.(int))
    case Duck:
        v.Eat()
    default:
        fmt.Printf("Unknown type\n")
    }
}
```

### 总结

本文从鸭子类型引出 ```Go``` 的接口，然后用一个例子简单展示了接口类型的用法，接着又介绍了接口赋值，空接口，类型断言和类型查询。
