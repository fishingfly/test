# 切片长度

关于切片的面试题：摘自[https://goquiz.github.io/\#subslice-grow](https://goquiz.github.io/#subslice-grow)

```text
func Subslice() {
   s := []int{1, 2, 3,4,5,6,7,8,9}
   ss := s[3:6]
   fmt.Printf("len ss : %d\n", len(ss))
   fmt.Printf("Cap ss : %d\n", cap(ss))
   ss = append(ss, 4)
   fmt.Printf("len ss : %d\n", len(ss))
   fmt.Printf("Cap ss : %d\n", cap(ss))
   for _, v := range ss {
      v += 10
   }
​
   for i := range ss {
      ss[i] += 10
   }
​
   fmt.Println(s)
}
```

![Click and drag to move](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

大家可以看一下结果是什么，结果是：

```text
len ss : 3
Cap ss : 6
len ss : 4
Cap ss : 6
[1 2 3 14 15 16 14 8 9 0 1 3]
```

![Click and drag to move](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

下面是解析过程：

首先ss := s\[3:6\],结果就是截取索引在\[3, 6）上的数据，所以len\(ss\)是3，那ss的cap容量为啥是9呢？

一个切片的容量就是该切面在底层数组山的开始位置向右扩展至数组的结束位置，在这边就是索引位置3到索引位置8 ，一共六个元素，append\(\)操作并没有导致容量增加，因为切片容量为6，加上元素4长度才是4，不会导致切片扩容。因为切片没有扩容，引用的还是底层s数组，所以更改ss上的元素的大小，数组s上的值也会跟着改变。

大家再看下这道题的打印结果是什么：

```text
func subSlice2() {
   s := []int{1, 2, 3}
   ss := s[1:]
   ss = append(ss, 4)
​
   for _, v := range ss {
      v += 10
   }
​
   for i := range ss {
      ss[i] += 10
   }
   fmt.Println(s)
}
```

![Click and drag to move](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

结果是：

```text
[1 2 3]
```

![Click and drag to move](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

下面是解析过程：

想不通的是为啥改了ss的值，但是数组s不变呢。原因是，ss:=s\[1:\]后，ss长度为2，cap为2。append后长度大于2所以切片扩容，扩容后的切片指向新的数组，与原数组s无关，所以对ss修改值，原数组的值不会变。

下面，介绍下底层append的代码，让大家明白这个过程。

首先append函数是go的内置函数，所以但是go并不开放内置函数的详细代码（如果有人看到源码，请在博客评论中告诉我，谢谢），所以我只在builtin.go中找到如下代码：

```text
// The append built-in function appends elements to the end of a slice. If
// it has sufficient capacity, the destination is resliced to accommodate the
// new elements. If it does not, a new underlying array will be allocated.
// Append returns the updated slice. It is therefore necessary to store the
// result of append, often in the variable holding the slice itself:
// slice = append(slice, elem1, elem2)
// slice = append(slice, anotherSlice...)
// As a special case, it is legal to append a string to a byte slice, like this:
// slice = append([]byte("hello "), "world"...)
func append(slice []Type, elems ...Type) []Type
```

![Click and drag to move](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

对于append函数的细节，注释里讲了，内置函数将新元素接到切片的后面，如果切面有足够的容量，则切片可以容纳新元素。如果切面容量不够，一个新的底层数组将会被分配。李笑来老师说过：english + compute skills = freedom。大家多看看源码还是有好处的。继续看下源码slice.go下的这个growslice 函数（主要用于处理append过程中Slice的增长），主要看下cap容量这个参数是怎么变化的？分析过程见下面的代码中文注释

```text
// growslice handles slice growth during append.
// （这句话告诉你传入参数有哪些？元素类型、老的slice、需要的容量）It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
// The new slice's length is set to the old slice's length,
// NOT to the new requested capacity.
// This is for codegen convenience. The old slice's length is used immediately
// to calculate where to write new values during an append.
// TODO: When the old backend is gone, reconsider this decision.
// The SSA backend might prefer the new length or to return only ptr/cap and save stack space.
func growslice(et *_type, old slice, cap int) slice {
   if raceenabled {//常量false，不用管
      callerpc := getcallerpc()
      racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
   }
   if msanenabled {//常量False不用管
      msanread(old.array, uintptr(old.len*int(et.size)))
   }
​
   if cap < old.cap {//需要的容量比老容量小，就报错
      panic(errorString("growslice: cap out of range"))
   }
​
   if et.size == 0 {//类型大小为0就不
      // append should not create a slice with nil pointer but non-zero len.
      // We assume that append doesn't need to preserve old.array in this case.
      return slice{unsafe.Pointer(&zerobase), old.len, cap}
   }
    //下面正式开始新的容量计算
   newcap := old.cap //老容量给newcap
   doublecap := newcap + newcap //两倍老容量
   if cap > doublecap { //如果需要的容量大于两倍老容量，那新容量就是需要的容量
      newcap = cap
   } else {//需要的容量小于两倍老容量
      if old.len < 1024 {//且老切片长度小于1024
         newcap = doublecap//则新容量为两倍老容量
      } else {//老切片长度大于1024
         // Check 0 < newcap to detect overflow
         // and prevent an infinite loop.
         for 0 < newcap && newcap < cap {
            newcap += newcap / 4 //老容量以125%增长率增长，知道newcap增加到比需要的容量大
         }
         // Set newcap to the requested cap when
         // the newcap calculation overflowed.
         if newcap <= 0 {
            newcap = cap //额外情况，老容量为0，那新容量就是需要的容量
         }
      }
   }
//下面计算溢出的，就不用看了
   var overflow bool
   var lenmem, newlenmem, capmem uintptr
   // Specialize for common values of et.size.
   // For 1 we don't need any division/multiplication.
   // For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
   // For powers of 2, use a variable shift.
   switch {
   case et.size == 1:
      lenmem = uintptr(old.len)
      newlenmem = uintptr(cap)
      capmem = roundupsize(uintptr(newcap))
      overflow = uintptr(newcap) > maxAlloc
      newcap = int(capmem)
   case et.size == sys.PtrSize:
      lenmem = uintptr(old.len) * sys.PtrSize
      newlenmem = uintptr(cap) * sys.PtrSize
      capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
      overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
      newcap = int(capmem / sys.PtrSize)
   case isPowerOfTwo(et.size):
      var shift uintptr
      if sys.PtrSize == 8 {
         // Mask shift for better code generation.
         shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
      } else {
         shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
      }
      lenmem = uintptr(old.len) << shift
      newlenmem = uintptr(cap) << shift
      capmem = roundupsize(uintptr(newcap) << shift)
      overflow = uintptr(newcap) > (maxAlloc >> shift)
      newcap = int(capmem >> shift)
   default:
      lenmem = uintptr(old.len) * et.size
      newlenmem = uintptr(cap) * et.size
      capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
      capmem = roundupsize(capmem)
      newcap = int(capmem / et.size)
   }
​
   // The check of overflow in addition to capmem > maxAlloc is needed
   // to prevent an overflow which can be used to trigger a segfault
   // on 32bit architectures with this example program:
   //
   // type T [1<<27 + 1]int64
   //
   // var d T
   // var s []T
   //
   // func main() {
   //   s = append(s, d, d, d, d)
   //   print(len(s), "\n")
   // }
   if overflow || capmem > maxAlloc {
      panic(errorString("growslice: cap out of range"))
   }
​
   var p unsafe.Pointer
   if et.kind&kindNoPointers != 0 {
      p = mallocgc(capmem, nil, false)
      // The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
      // Only clear the part that will not be overwritten.
      memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
   } else {
      // Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
      p = mallocgc(capmem, et, true)
      if writeBarrier.enabled {
         // Only shade the pointers in old.array since we know the destination slice p
         // only contains nil pointers because it has been cleared during alloc.
         bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
      }
   }
   memmove(p, old.array, lenmem)
​
   return slice{p, old.len, newcap}
}
```

![Click and drag to move](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

容量增长的过程：

1、输入参数，切片，需要的容量

2、要是需要的容量比两倍老容量都大，那新的容量大小就是需要的容量大小

3、要是需要的容量比两倍老容量小：a、且老切片的长度小于1024个，则新容量大小为老容量的2倍 b、老切片长度大于1024，则老切片长度\*（1.25\)^n直达老切片长度大小超过需要的容量。

所以第二题的cap为4，原先cap为2，需要的cap为3，因为老切片长度为2小于2014所以直接老切片长度两倍就可以了。然后第二题的数组s的值没有变的原因是，Append发现老切片cap已经不够了，需要申请新切片。新切片底层数组就不是原来的数组了，所以对切片ss每个元素+10，是不影响老数组的元素的。

