# sync.Map 的用法

## 问题

```go
package main

import (
 "fmt"
 "sync"
)

func main(){
 var m sync.Map
 m.Store("address",map[string]string{"province":"江苏","city":"南京"})
 v,_ := m.Load("address")
 fmt.Println(v["province"]) 
}
```

- A，江苏；
- B`，v["province"]`取值错误；
- C，`m.Store`存储错误；
- D，不知道

## 解析

`invalid operation: v["province"] (type interface {} does not support indexing)`
因为 `func (m *Map) Store(key interface{}, value interface{})`
所以 `v`类型是 `interface {}` ，这里需要一个类型断言

``` go
fmt.Println(v.(map[string]string)["province"]) //江苏
```
