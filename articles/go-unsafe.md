<!---
markmeta_author: wongoo
markmeta_date: 2019-03-12
markmeta_title: Go unsafe
markmeta_categories: 编程语言
markmeta_tags: golang
-->

# Go unsafe

## 1. 概览
unsafe包包含三个方法,两个类型:
```go
func Alignof(x ArbitraryType) uintptr
func Offsetof(x ArbitraryType) uintptr
func Sizeof(x ArbitraryType) uintptr
type ArbitraryType int
type Pointer *ArbitraryType
```

- `ArbitraryType`: 是以int为基础定义的一个新类型，但是Go 语言unsafe包中，对ArbitraryType赋予了特殊的意义，通常，我们把interface{}看作是任意类型，那么ArbitraryType这个类型，在Go 语言系统中，比interface{}还要随意。
- `Pointer`: Pointer 以 ArbitraryType 指针类型为基础的新类型, 可以理解为c语言的 `void*`。它可以让任意类型的指针实现相互转换，也可以将任意类型的指针转换为 uintptr 进行指针运算
- `Alignof()`: 返回变量对齐字节数量
- `Offsetof()`: 返回变量指定属性的偏移量，所以如果变量是一个struct类型，不能直接将这个struct类型的变量当作参数，只能将这个struct类型变量的属性当作参数。
- `Sizeof()`:  返回变量在内存中占用的字节数，切记，如果是slice，则不会返回这个slice在内存中的实际占用长度。


## 2. uintptr和unsafe.Pointer的区别
- unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算；
- 而uintptr是用于指针运算的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象， uintptr 类型的目标会被回收；
- unsafe.Pointer 可以和 普通指针 进行相互转换；
- unsafe.Pointer 可以和 uintptr 进行相互转换。

## 3. 内存对齐、偏移量、内存大小

### 3.1 为什么要内存对齐?
- 很多CPU只从对齐的地址开始加载数据，而有的CPU这样做，只是更快一点。
- 外部总线从内存一次获取的数据往往不是1byte，而是4bytes或许8bytes,或者更多。

这意味着CPU不是一次仅仅抓取一个byte，而是很4个或者8个byte。
这样的话，假设你请求2到3bytes的内容，其实CPU一次抓取了4或8byte的内容。

### 3.2 对齐准则

结构体字节对齐的细节和具体编译器实现相关，但一般而言满足三个准则：
1. 结构体变量的首地址能够被其最宽基本类型成员的大小所整除；
2. 结构体每个成员相对结构体首地址的偏移量(offset)都是成员大小的整数倍，如有需要编译器会在成员之间加上填充字节(internal adding)；
3. 结构体的总大小为结构体最宽基本类型成员大小的整数倍，如有需要编译器会在最末一个成员之后加上填充字节{trailing padding}。

对于以上规则的说明如下：
- 第一条：编译器在给结构体开辟空间时，首先找到结构体中最宽的基本数据类型，然后寻找内存地址能被该基本数据类型所整除的位置，作为结构体的首地址。将这个最宽的基本数据类型的大小作为上面介绍的对齐模数。
- 第二条：为结构体的一个成员开辟空间之前，编译器首先检查预开辟空间的首地址相对于结构体首地址的偏移是否是本成员大小的整数倍，若是，则存放本成员，反之，则在本成员和上一个成员之间填充一定的字节，以达到整数倍的要求，也就是将预开辟空间的首地址后移几个字节。
- 第三条：结构体总大小是包括填充字节，最后一个成员满足上面两条以外，还必须满足第三条，否则就必须在最后填充几个字节以达到本条要求。

内存地址详细的分析可以参考<sup>4</sup>.

### 3.3 golang 变量对齐字节数
```
Alignof string :  8
Sizeof  string :  16
Alignof bool :  1
Sizeof  bool :  1
Alignof byte :  1
Sizeof  byte :  1
Alignof int8 :  1
Sizeof  int8 :  1
Alignof int16 :  2
Sizeof  int16 :  2
Alignof int32 :  4
Sizeof  int32 :  4
Alignof int64 :  8
Sizeof  int64 :  8
Alignof int :  8
Sizeof  int :  8
Alignof uint32 :  4
Sizeof  uint32 :  4
Alignof uint64 :  8
Sizeof  uint64 :  8
Alignof float32 :  4
Sizeof  float32 :  4
Alignof float64 :  8
Sizeof  float64 :  8
```

### 3.4 范例

```go
type t1 struct {
	b   bool
	i16 int16
	i32 int32
}
type t2 struct {
	b   bool
	i32 int32
	i16 int16
}

x1 := t1{}
x2 := t2{}

t.Logf("t1 Alignof: %d, Sizeof: %d", unsafe.Alignof(x1), unsafe.Sizeof(x1)) // 4, 8
t.Logf("t2 Alignof: %d, Sizeof: %d", unsafe.Alignof(x2), unsafe.Sizeof(x2)) // 4, 12

t.Logf("t1 Offsetof b: %d, i16: %d , i32: %d", unsafe.Offsetof(x1.b), unsafe.Offsetof(x1.i16), unsafe.Offsetof(x1.i32)) // 0, 2, 4
t.Logf("t2 Offsetof b: %d , i32: %d, i16: %d", unsafe.Offsetof(x2.b), unsafe.Offsetof(x2.i32), unsafe.Offsetof(x2.i16)) // 0, 4, 8
```

分析:
- 成员b的对齐值是1，i16的对齐值是2，i32的对齐值是4
- t1,t2 的对齐值都是4，即其最大成员变量i32的对齐值
- 对于t1, 其分配内存的首地址需要能够被t1的对齐值4整除；首先分配b从相对0的位置开始占1个字节； 分配i16时需要其首地址需要能够被2整除，刚分配b的时候只占一个字节，需要再填充一个字节，故i16分配从相对位置2的地方开始分配2个字节；分配i32的时候首地址需要能够被4整除，前面b和i16已经分配了4个字节刚好够整，所以i32就在相对位置4的地方开始分配； 总结b,i16,i32三个字段的offset分别为0，2，4。此时已分配8字节，刚好能被4整除，不用再其后填充字节。
- 对于t2，b占一个字节；分配i32时为使得起始地址能被4整除前填充3个字节；分配i16其前面不用再填充字节；b、i32、i16三个字段的offset分别为0，4，8； 此时已分配10个字节，10不能被4整除，故在最后填充两个字节，使得t2占用12个字节。


## 4. Pointer 操作

uintptr是Go 语言的内置类型，是能存储指针的整型， uintptr 的底层类型是int，它和unsafe.Pointer可相互转换。

提供四类操作:

- `任何类型的指针` 转换为 `Pointer`
- `Pointer` 转换为 `任何类型的指针` 
- `uintptr` 转换为 `Pointer`
- `Pointer` 转换为 `uintptr`

### 5.1 `*T1`的指针 转换为 指向 `*T2` 的Pointer

如果类型T2所占内存空间不超过类型T1, 且它们能适应相同的内存数据结构，则可以进行类型转换.

```go
func Float64bits(f float64) uint64 {
  return *(*uint64)(unsafe.Pointer(&f))
}
```

当然，unsafe.Pointer是不会检查类型所占空间是不是足够！

```go
type x struct {
  f1 int
  f2 int
}

i1 := 123
i2 := 456

t.Logf("i1 addr: %p, value: %d", &i1, i1) // i1 addr: 0xc00009e290, value: 123
t.Logf("i2 addr: %p, value: %d", &i2, i2) // i2 addr: 0xc00009e298, value: 456

xp := (*x)(unsafe.Pointer(&i1))
t.Logf("xp addr: %p, value: %v", xp, xp) // xp addr: 0xc00009e290, value: &{123 456}

x1 := *xp
t.Logf("x1 addr: %p, value: %v", &x1, x1) // x1 addr: 0xc00009e2f0, value: {123 456}
```

以上i1、i2是连续分配的两个整形，位置紧挨着，刚好和struct x的内存结构一致，所以可以将i1开始地址作为一个x的指针地址。
`x1 := *xp` 则是复制了整个内存结构赋值给了一个新的变量。

上面例子的变量类型都是int整形，任何内存位置的内容都可以转换为int整形，**但如果转换类型为指针就危险了，将未知的值作为指针去访问指针指向位置的值将报错！**

```go
type x struct {
  f1 int
  f2 *int // <-------pointer
}
i1 := 123
i2 := 456

t.Logf("i1 addr: %p, value: %d", &i1, i1) // i1 addr: 0xc00009e2e0, value: 123
t.Logf("i2 addr: %p, value: %d", &i2, i2) // i2 addr: 0xc00009e2e8, value: 456

xp := (*x)(unsafe.Pointer(&i1))
t.Logf("xp addr: %p, value: %v", xp, xp) // xp addr: 0xc00009e2e0, value: &{123 0x1c8}

x1 := *xp
t.Logf("x1 addr: %p, value: %v", &x1, x1) // x1 addr: 0xc0000903d0, value: {123 0x1c8}

t.Log(*x1.f2) // panic: runtime error: invalid memory address or nil pointer dereference 
```

### 5.2 Pointer 转换为 uintptr (不能转换回Pointer)

转换Pointer为uintptr, 即获取其值指向的内存地址整形值。 通常使用uinptr只是为了打印显示。

一般将uintptr转换回Pointer是非法的。

uintptr是整形，不是引用。
转换Pointer为uintptr创建了一个整形，其没有指针语义. 
尽管uintptr包含某个对象的地址，但对象移动的时候垃圾回收器并不会更新uintptr的值，也不会让对象避免被回收。

接下来这些方法模式才是合法的从uintptr到Pointer的转换。

### 5.3 Pointer 转换为 uintptr 再转换回来(算术操作)

> `算术操作`: 指操作只能在一个表达式里面

如果p(Pointer)指向一个已分配地址的对象, 能够直接将其转换为uintptr，uintptr加上一个offset就能转换回Pointer。

```go
p = unsafe.Pointer(uintptr(p) + offset)
```

最常用这种方式访问struct的一个字段或者array数组中的一个元素:
```go
// equivalent to f := unsafe.Pointer(&s.f)
f := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f))

// equivalent to e := unsafe.Pointer(&x[i])
e := unsafe.Pointer(uintptr(unsafe.Pointer(&x[0])) + i*unsafe.Sizeof(x[0]))
```

使用这种方式加上或减去offset都是合法的。
还可以使用`&^`对指针进行round(取舍)操作。
所有的这些操作，需要保证结果还是指向原来分配地址的对象。

不像c语言，创建一个超过原始分配地址的结束位置的指针是非法的。
```go
// INVALID: end points outside allocated space.
var s thing
end = unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Sizeof(s))

// INVALID: end points outside allocated space.
b := make([]byte, n)
end = unsafe.Pointer(uintptr(unsafe.Pointer(&b[0])) + uintptr(n))
```

> 注意: 转换过程需包含在一个表达式中，其中只包括对这些中间数据的算术操作:
```go
// INVALID: uintptr cannot be stored in variable before conversion back to Pointer.
u := uintptr(p)
p = unsafe.Pointer(u + offset)
```

> 注意: Pointer只能是对已经分配地址的对象进行操作，不能是nil.
```go
// INVALID: conversion of nil pointer
u := unsafe.Pointer(nil)
p := unsafe.Pointer(uintptr(u) + offset)
```

### 5.4 调用syscall.Syscall时将Pointer转换为uintptr

syscall包下面的Syscall方法传递uintptr参数给操作系统，接下来可能根据调用情况将其中一些参数转换为指针。
也就是系统调用隐式的将某些参数由uintptr转换回指针。

如果指针必须转换为uintptr参数，转换必须在调用表达式中。

```go
syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))
```

以下这种方式就会报错:
```go
// INVALID: uintptr cannot be stored in variable before implicit conversion back to Pointer during system call.
u := uintptr(unsafe.Pointer(p))
syscall.Syscall(SYS_READ, uintptr(fd), u, uintptr(n))
```

编译器处理调用function参数列表中将Pointer转换为uintptr时，是用汇编的方式实现的，通过编排引用已分配对象，使其被保留直到调用完成前不会被移动，尽管这些类型将其孤立(没有引用)使得调用期间对象可能已经不再需要了。
> 上面翻译有点难懂，保留原文: The compiler handles a Pointer converted to a uintptr in the argument list of a call to a function implemented in assembly by arranging that the referenced allocated object, if any, is retained and not moved until the call completes, even though from the types alone it would appear that the object is no longer needed during the call.

### 5.5 reflect.Value.Pointer或者reflect.Value.UnsafeAddr 的结果 uintptr 转换为 Pointer.

reflect包中的Value的`Pointer()`和`UnsafeAddr()`两个方法返回uintptr类型，而不是unsafe.Pointer类型， 
这样可以避免调用者在引入unsafe包操作前将结果转变为一个任意类型。
这意味着，方法结果是不稳定的，需要调用后立刻转变为Pointer类型。

```go
p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
```

上面的例子，如果将结果存到一个变量则是非法的：
```go
// INVALID: uintptr cannot be stored in variable
// before conversion back to Pointer.
u := reflect.ValueOf(new(int)).Pointer()
p := (*int)(unsafe.Pointer(u))
```

### 5.6 reflect.SliceHeader 或者 reflect.StringHeader 的数据字段 转换为 Pointer（或从Pointer转换回）.

上一种情况， reflect数据结构 `SliceHeader` 和 `StringHeader` 定义了字段一个uintptr类型的字段Data， 让调用者在引入unsafe操作之前将结果转换为任意类型。
这意味着  `SliceHeader` 和 `StringHeader` 是唯一合法的方式（将uintptr赋值给一个变量），但只有指向的实际内容是slice或者string的时候。 

```go
var s string
hdr := (*reflect.StringHeader)(unsafe.Pointer(&s)) // 第1种形式
hdr.Data = uintptr(unsafe.Pointer(p))              // 第6种形式 (也就是当前形式)
hdr.Len = n
```

`hdr.Data`的使用是一种间接的方式去引用string头部指针, 而不是直接使用uintptr变量本身。

一般来说`reflect.SliceHeader`和`reflect.StringHeader` 只能以指针的方式指向实际的slice或者string，而不能用原始struct类型。
定义或分配这样的struct类型的变量都是不允许的。

```go
// INVALID: a directly-declared header will not hold Data as a reference.
var hdr reflect.StringHeader
hdr.Data = uintptr(unsafe.Pointer(p))
hdr.Len = n
s := *(*string)(unsafe.Pointer(&hdr)) // p possibly already lost
```

`reflect.SliceHeader`是slice的运行时表示形式， `reflect.StringHeader` 为string的运行时表示形式。
```go
type StringHeader struct {
  Data uintptr
  Len  int
}
type SliceHeader struct {
  Data uintptr
  Len  int
  Cap  int
}
```

以下测试可以进一步了解: 

```go
a := "hello"
b := "world"
t.Logf("a addr: %p, value: %s", &a, a) // a addr: 0xc0000c6000, value: hello
t.Logf("b addr: %p, value: %s", &b, b) // b addr: 0xc0000c6010, value: world

var p *string
t.Logf("p addr: %p", p) // p addr: 0x0

p = &a
t.Logf("p addr: %p", p) // p addr: 0xc0000c6000

sh1 := (*reflect.StringHeader)(unsafe.Pointer(p))
sh2 := (*reflect.StringHeader)(unsafe.Pointer(&b))
s1 := (*string)(unsafe.Pointer(sh1))
s2 := (*string)(unsafe.Pointer(sh2))

t.Logf("sh1 addr: %p, value: %v", sh1, sh1) // sh1 addr: 0xc0000c6000, value: &{18065496 5}
t.Logf("sh2 addr: %p, value: %v", sh2, sh2) // sh2 addr: 0xc0000c6010, value: &{18065581 5}
t.Logf("s1 addr: %p, value: %s", s1, *s1) // s1 addr: 0xc0000c6000, value: hello
t.Logf("s2 addr: %p, value: %s", s2, *s2) // s2 addr: 0xc0000c6010, value: world

sh1.Data = uintptr(unsafe.Pointer(&b)) // <---- 错误的将Data的值赋值为一个错误的，注意&b的地址值并不是其内部表示形式字段Data的值
t.Logf("sh1 addr: %p, value: %v", sh1, sh1) // sh1 addr: 0xc0000c6000, value: &{824634531856 5}
t.Logf("s1 addr: %p,  value: %s", s1, *s1) // s1 addr: 0xc0000c6000,  value: ��    <---- 值已经变了

sh1.Data = sh2.Data  // <---- 赋值为b的内部表示的值
sh1.Len = sh2.Len  // <---- 赋值正确的长度
t.Logf("s1 addr: %p, value: %s", s1, *s1) // s1 addr: 0xc0000c6000,  value: world
t.Logf("a addr: %p, value: %s", &a, a) // a addr: 0xc0000c6000, value: world
t.Logf("b addr: %p, value: %s", &b, b) // b addr: 0xc0000c6010, value: world
// 以上a、b 地址不同，但值相同,实际数据指向也只有一份
```

## Reference

1. Matt Layher, unsafe.Pointer and system calls, https://blog.gopheracademy.com/advent-2017/unsafe-pointer-and-system-calls/
2. package unsafe, https://golang.org/pkg/unsafe/
3. Go Slices: usage and internals, https://blog.golang.org/go-slices-usage-and-internals
4. C语言字节对齐问题详解, https://www.cnblogs.com/clover-toeic/p/3853132.html

