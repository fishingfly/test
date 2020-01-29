# 指针方法&值方法

Golang官方文档对于指针方法和值方法的描述：[https://golang.org/doc/effective\_go.html\#pointers\_vs\_values](https://golang.org/doc/effective_go.html#pointers_vs_values)，下面是文档中的英文内容

#### Pointers vs. Values

As we saw with `ByteSize`, methods can be defined for any named type \(except a pointer or an interface\); the receiver does not have to be a struct.（接收者不一定必须是struct）

In the discussion of slices above, we wrote an `Append` function. We can define it as a method on slices instead. To do this, we first declare a named type to which we can bind the method, and then make the receiver for the method a value of that type.（设置了一个场景，就是Slice的append函数，用slice作为方法的接收者）

```text
type ByteSlice []byte
​
func (slice ByteSlice) Append(data []byte) []byte {
    // Body exactly the same as the Append function defined above.
}
```

This still requires the method to return the updated slice（这边谈到值方法必须把更新的值返回出去，也就是reture出去，这样就会比较麻烦）. We can eliminate that clumsiness by redefining the method to take a _pointer_ to a `ByteSlice` as its receiver, so the method can overwrite the caller's slice（这就引出了指针方法，可以重写这个值）.

```text
func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // Body as above, without the return.
    *p = slice
}
```

In fact, we can do even better. If we modify our function so it looks like a standard `Write` method, like this,

```text
func (p *ByteSlice) Write(data []byte) (n int, err error) {
    slice := *p
    // Again as above.
    *p = slice
    return len(data), nil
}
```

then the type `*ByteSlice` satisfies the standard interface `io.Writer`, which is handy. For instance, we can print into one.

```text
    var b ByteSlice
    fmt.Fprintf(&b, "This hour has %d days\n", 7)
```

We pass the address of a `ByteSlice` because only `*ByteSlice` satisfies `io.Writer`. The rule about pointers vs. values for receivers is that value methods can be invoked on pointers and values, but pointer methods can only be invoked on pointers.这句话是说值方法可以被指针和值所调用，但是指针方法只能被指针所调用。

This rule arises because pointer methods can modify the receiver; invoking them on a value would cause the method to receive a copy of the value, so any modifications would be discarded.（上面这条规则的原因是指针方法可以修改receiver，但是值方法的调用只是让方法接收到这个给值的一份拷贝，对这个值的改变会被抛弃掉） The language therefore disallows this mistake. There is a handy exception, though. When the value is addressable（当只可以被寻址时）, the language takes care of the common case of invoking a pointer method on a value by inserting the address operator automatically. In our example, the variable `b` is addressable, so we can call its `Write` method with just `b.Write`. The compiler will rewrite that to `(&b).Write` for us.

By the way, the idea of using `Write` on a slice of bytes is central to the implementation of `bytes.Buffer`.

严格来讲，基本类型的值只能调用他的值方法，而在实际中使用时，值可以调用指针方法，Golang会帮你自动转换，使得值可以调用到指针方法，原理就是：golang会先取到值的地址，用值的地址去调用指针方法。

