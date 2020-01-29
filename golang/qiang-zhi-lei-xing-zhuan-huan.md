# 强制类型转换

从问题开始，今天我想对一个struct的数组强制转为\[\]interface{}，发现报错，报错显示：Invalid type assertion。所以到底是咋回事呢？趁这个机会，抓紧补习下golang的类型转换，有哪些方式，以及有哪些方式是不允许类型转换的，还有就是自动类型转换发生在哪些情况下。

golang的类型转换分为强制类型转换和类型断言，下面就从这两个方面入手：

**强制类型转换**

普通类型变量（int,float,string）都是用type\(a）的形式进行类型转换，举如下例子：

```text
var a int32  = 10
var b int64 = int64(a)
var c float32 = 12.3
var d float64 =float64(c)
```

一般来讲int32、int64、int8之间可以通过强制类型互相转换，但是从长度大的int转到长度小的Int时，就需要考虑溢出问题了，这个是需要注意的，看下下面代码：

```text
    var intTest32 int32 = 2147483647
    var intTest64 int64 = 2147483648
    intTest32 = int32(intTest64)
    fmt.Println(intTest32)
    fmt.Println(intTest64)
```

结果：

```text
-2147483648
2147483648
```

再看看下面普通类型转换报错的代码：

```text
func main() {
    i := 0
    var a float32
    a = 0.9
    i = a //cannot use `a` (type float32) as type int assignment
    var b float64
    b = a//cannot use `a` (type float32) as type float64 assignment
    var intTest int32
    intTest = 12
    var intTest2 int8
    intTest2 = intTest ////cannot use `intTest` (type int32) as type int8 assignment
}
```

得出来的结论是：普通类型无法自动进行类型转换，就算是Int8到int32都不能自动转换。

顺便看下int,int8,int32,int64的区别，参见[https://golang.google.cn/ref/spec\#Numeric\_types](https://golang.google.cn/ref/spec#Numeric_types)。

```text
uint8       the set of all unsigned  8-bit integers (0 to 255)
uint16      the set of all unsigned 16-bit integers (0 to 65535)
uint32      the set of all unsigned 32-bit integers (0 to 4294967295)
uint64      the set of all unsigned 64-bit integers (0 to 18446744073709551615)
​
int8        the set of all signed  8-bit integers (-128 to 127)
int16       the set of all signed 16-bit integers (-32768 to 32767)
int32       the set of all signed 32-bit integers (-2147483648 to 2147483647)
int64       the set of all signed 64-bit integers (-9223372036854775808 to 9223372036854775807)
```

比如int8是8位bit组成的，范围在-128到127。你可以通过unsafe.sizeof函数查看int8的大小。

```text
    var i1 int = 1
    var i2 int8 = 2
    var i3 int16 = 3
    var i4 int32 = 4
    var i5 int64 = 5
    fmt.Println(unsafe.Sizeof(i1))
    fmt.Println(unsafe.Sizeof(i2))
    fmt.Println(unsafe.Sizeof(i3))
    fmt.Println(unsafe.Sizeof(i4))
```

结果是：

```text
8
1
2
4
8
```

看下unsafe.Sizeof的注释：

```text
// Sizeof takes an expression x of any type and returns the size in bytes sizeof可以接受任何类型的表达式，返回的是以比特为单位的大小
// of a hypothetical variable v as if v was declared via var v = x.
// The size does not include any memory possibly referenced by x.
// For instance, if x is a slice, Sizeof returns the size of the slice
// descriptor, not the size of the memory referenced by the slice.
// The return value of Sizeof is a Go constant.
func Sizeof(x ArbitraryType) uintptr
```

所以Int8、int32、int64之间只是用来存储的bit个数不一样，默认int是int64，所以大家平时要根据需要申请，别直接Int。

OK，强制类型转换就说到这。下面说下类型断言。

#### 类型断言

首先golang所有类型都实现了interface{},就像Java语言中的Object对象，golang中自定义的struct都是实现interface{}的，所以一般类型都能转换为Interface{}。但是Interface{}能直接强制类型转换为普通类型吗？比如string。实践一下就知道了：

```text
    var strTest string = "testing"
    t := interface{}(strTest)
    fmt.Println(t)
    fmt.Println(reflect.TypeOf(t))
```

结果：

```text
testing
string
```

下面测试下Interface{}能否直接强制类型转换为普通类型

```text
    var strTest string = "testing"
    t := interface{}(strTest)
    fmt.Println(string(t)) // error: cannot convert expression of type interface{} to type string
```

上面例子显示interface{}强转string会报错，但是我们明明知道这是string，但是转不了，咋办呢？这就需要用到类型转换了：

```text
var strTest string = "testing"
    t := interface{}(strTest)
    if newt, ok := t.(string)
        fmt.Println(newt)
        fmt.Println("转换成功")
        fmt.Println(reflect.TypeOf(newt))
    } else {
        fmt.Println("转换不成功")
        fmt.Println(newt)
        fmt.Println(reflect.TypeOf(newt))
    }
```

结果：

```text
testing
转换成功
string
```

我觉得这样的断言设计是为了在运行时动态判断传送过来的数据类型，进行动态避错，减少bug的发生，也能增强接口的容错性。就算转换不成功，也会有结果值，这样就非常好了，如下：

```text
    var strTest string = "testing"
    t := interface{}(strTest)
    if newt, ok := t.(int8); ok {//注意这里变了
        fmt.Println(newt)
        fmt.Println("转换成功")
        fmt.Println(reflect.TypeOf(newt))
    } else {
        fmt.Println("转换不成功")
        fmt.Println(newt)
        fmt.Println(reflect.TypeOf(newt))
    }
```

结果：

```text
转换不成功
0
int8
```

所以，就算类型转换不成功，他也会帮你把结果值初始化为你要转换类型的零值。这样的设计非常的好。我们在使用时也不需要强迫自己去判断Ok是否为true,反正有零值返回。当然也要看零值是否是你所需要的。

总结下：我个人比较喜欢类型断言，设计的非常棒

参考文章：

[https://studygolang.com/articles/21591](https://studygolang.com/articles/21591)

[https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-typecheck/](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-typecheck/)

[https://golang.org/doc/faq\#convert\_slice\_of\_interface](https://golang.org/doc/faq#convert_slice_of_interface)

推荐以下golang语言相关的问题可以去看下这个地址：[https://golang.org/doc/faq](https://golang.org/doc/faq)

