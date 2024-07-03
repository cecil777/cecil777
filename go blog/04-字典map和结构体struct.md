## 字典

字典是一种非常常用的数据结构，Go中用关键词map表示，类型是```map[K]V```。K和V分别是字段的键和值的数据类型，其中键必须支持相等运算符，比如数字、字符串等。

### 创建字典

有两种方式可以创建字典，第一种是直接使用字面量创建；第二种使用内置函数```make```。

字面量方式创建：

``` go
// 字面量方式创建
var m = map[string]int{"a": 1, "b": 2}
fmt.Println(m) // map[a:1 b:2]
```

使用```make```创建

``` go
// 使用make创建
m1 := make(map[string]int)
fmt.Println(m1)
```

还可以初始化字典的长度。在已知字典长度的情况下，直接指定长度可以提升程序的执行效率。

``` go
// 指定长度
m2 := make(map[string]int, 10)
fmt.Println(m2)
```

字典的零值是```nil```，对值是```nil```的字典赋值会报错。

``` go
// 零值是nil
var m3 map[string]int
fmt.Println(m3 == nil,len(m3) == 0)

// nil 赋值报错
// m3["a"] = 1
// fmt.Println(m3)
```

### 使用字典

赋值：

``` go
// 赋值
m["c"] = 3
m["d"] = 3
fmt.Println(m) // map[a:1 b:2 c:3 d:4]
```

取值：

``` go
// 取值
fmt.Println(m["a"], m["d"]) // 1 4
fmt.Println(m["k"])         // 0
```

即使在 Key 不存在的情况下，也是不报错的。而是返回对应类型的零值。

删除元素：

``` go
// 删除
delete(m, "c")
delete(m, "f") // key 不存在也不报错
fmt.Println(m) // map[a:1 b:2 d:4]
```

获取长度

``` go
// 获取长度
fmt.Println(len(m)) // 3
```

判断键是否存在：

``` go
// 判断键是否存在
if value, ok := m["d"]; ok {
    fmt.Println(value) // 4
}
```

和```Python```对比起来看，这个用起来就很爽。

遍历：

``` go
// 遍历
for k, v := range m {
    fmt.Println(k, v)
}
```

### 引用类型

map 是引用类型，所以在函数间传递时，也不会制造一个映射的副本，这点和切片类似，都很高效。

``` go
package main

import "fmt"

func main() {
    ...

    // 传参
    modify(m)
    fmt.Println("main: ", m) // main:  map[a:1 b:2 d:4 e:10]
}

func modify(a map[string]int) {
    a["e"] = 10
    fmt.Println("modify: ", a) //   modify:  map[a:1 b:2 d:4 e:10]
}
```

## 结构体

结构体是一种聚合类型，包含零个或多个任意类型的命名变量，每个变量叫做结构体的成员。

### 创建结构体

首先使用 type 来自定义一个结构体类型 user，里面有两个成员变量，分别是：name 和 age。

``` go
// 声明结构体
type user struct {
    name string
    age  int
}
```

结构体的初始化有两种方式：

第一种是按照声明字段的顺序逐个赋值，这里需要注意，字段的顺序需要严格一致。

``` go
// 初始化
u1 := user{"zhangsan", 18}
fmt.Println(u1) // {zhangsan, 18}
```

这样做的缺点很明显，如果字段顺便变了，那么凡是涉及到这个结构初始化的部分都要跟着变。

所以，更推荐使用第二种方法，按照字段名字来初始化。

``` go
// 更好的方式
// u := user{
//     age: 20
// }
// fmt.Println(u) // { 20}
u := user{
    name: "zhangsan",
    age: 18,
} 
fmt.Println(u)
```

未初始化的字段会赋值相应类型的零值。

### 使用结构体

使用点号```.``` 来访问和赋值成员变量。

``` go
// 访问结构体成员
fmt.Println(u.name, u.age) // zhangsan 18
u.name = "lisi"
fmt.Println(u.name, u.age) // lisi 18
```

如果结构体的成员变量是可比较的，那么结构体也是可比较的。

``` go
// 结构体比较
u2 := user{
    age: 18,
    name: "zhangsan",
}
fmt.Println(u1 == u) // false
fmt.Println(u1 == u2) // true
```

### 结构体嵌套

现在我们已经定义了一个```user```结构体了，假设我们再定义两个结构体```admin``` 和 ```leader```, 如下

``` go
type admin struct {
    name string
    age int
    isAdmin bool
}

type leader struct {
    name    string
    age int
    isLeader    bool
}
```

那么问题就来了，有两个字段```name``` 和 ```age``` 被重复定义了多次。

懒是程序员的必修课。有没有什么办法可以复用这两个字段。答案解释结构体嵌套。

使用嵌套方法优化后变成了这样：

``` go
type admin struct {
    u   user
    isAdmin bool
}
```

代码看起来简洁了很多

### 匿名成员

但这样依然不是很完美，每次访问嵌套结构体的成员变量还是有点麻烦。

``` go
// 结构体嵌套
a := admin {
    u: u,
    isAdmin: true,
}

fmt.Println(a) // {{lisi 18} true}

a.u.name = "wangwu"

fmt.Println(a.u.name) // wangwu
fmt.Println(a.u.age) // 18
fmt.Println(a.isAdmin) // true
```

 这个时候就需要匿名成员登场了，不指定名称，只指定类型。

``` go
type admin1 struct {
    user
    isAdmin bool
}
```

通过这种方式可以省略掉中间变量，直接访问我们需要的成员变量。

``` go
// 匿名成员
a1 := amdin1{
    user: u,
    isAdmin: true,
}

a1.age = 20
a1.isAdmin = false

fmt.Println(a1) // {{lisi, 20} false}
fmt.Println(a1.name) // lisi
fmt.Println(a1.age) // 20
fmt.Println(a1.isAdmin) // false
```
