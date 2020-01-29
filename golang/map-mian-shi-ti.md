# Map面试题

关于golang面试题：摘自[https://goquiz.github.io/\#subslice-grow](https://goquiz.github.io/#subslice-grow)

Correct two mistakes in line A and B.

```text
package main
​
func main() {
    var m map[string]int //A
    m["a"] = 1
    if v := m["b"]; v != nil { //B
        println(v)
    }
}
​
```

解惑：

A行，就这个申明不初始化，是没问题的，你可以读空map，但是不能写，你必须初始化后才能赋值。如果不初始化 map，那么就会创建一个 nil map。nil map 不能用来存放键值对。

改为: var m = make\(map\[string\]int\)

这边顺便说下golang中的nil到底是怎么回事？

首先nil并不是golang的关键字，甚至nil还可以被赋值，不信的话你可以试一下，是可以通过编译的

在Golang中，如果你没有对一个变量进行赋值，那么会有一个默认的零值，Go的文档中说到，_nil是预定义的标识符，代表指针、通道、函数、接口、映射或切片的零值_，先看下下面的零值：

```text
bool      -> false                              
numbers -> 0                                 
string    -> ""      
​
pointers -> nil
slices -> nil
maps -> nil
channels -> nil
functions -> nil
interfaces -> nil
```

这边稍带介绍下ponter和interface两个与nil的关系：

**Pointer**

```text
    var p *int
    fmt.Println(p == nil)    // true
    *p = 1     //panic: runtime error: invalid memory address or nil pointer dereference
```

指针是指向内存地址的，如果对空指针进行赋值的话会报错。因为指针的零值是Nil所以比较判断会报true。空指针有个妙用（也不知道算不算妙用，我就觉得之前没见过），就是指针方法，指针方法就算指针是nil的情况下也能正常调用。例如下面的例子

```text
type tree struct {
  v int
  l *tree
  r *tree
}
​
// 现在这个指针方法，就算t为nil也能被正常调用。
func (t *tree) Sum() int {
   if t == nil {
    return 0
  }
  return t.v + t.l.Sum() + t.r.Sum()
}
```

**interface**

interface并不是一个指针，它的底层实现由两部分组成，一个是类型，一个值，也就是类似于：\(Type, Value\)。只有当类型和值都是`nil`的时候，才等于`nil`。\(这一点在判断interface是不是nil的时候很关键\)

```text
type  doError interface{
​
}
​
func testNilInterface() doError{
    var err *doError
    return err 
}
​
func main() {
    fmt.Println(testNilInterface() == nil)
}
```

结果：false.这边虽然声明了指针err，未初始化，其值是nil，但是interface的type不是空值，所以与nil是不相等。interface要类型和值都为nil,interface才会和nil相等，例如如下函数：

```text
func testNilInterface1() doError{
    return nil
}
func main() {
    fmt.Println(testNilInterface1() == nil)
}
```

就会返回true了

B行，会报编译错误：cannot convert nil to type int，原因是，map中不存在key为b的键值对，就会返回nil，而nil不能强转为int

应该改为：

if v , ok:= m\["b"\]; ok {...}

现在就算map没有b这个key，v也会被初始化为Int的零值就是0

参考资料：

[https://www.youtube.com/watch?v=ynoY2xz-F8s](https://www.youtube.com/watch?v=ynoY2xz-F8s)  


