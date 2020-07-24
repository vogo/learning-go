<!---
markmeta_author: wongoo
markmeta_date: 2019-03-12
markmeta_title: Go unsafe
markmeta_categories: 编程语言
markmeta_tags: golang
-->

# Go unsafe

unsafe包包含三个方法,两个类型:
```go
func Alignof(x ArbitraryType) uintptr
func Offsetof(x ArbitraryType) uintptr
func Sizeof(x ArbitraryType) uintptr
type ArbitraryType int
type Pointer *ArbitraryType
```

- `ArbitraryType`: 是以int为基础定义的一个新类型，但是Go 语言unsafe包中，对ArbitraryType赋予了特殊的意义，通常，我们把interface{}看作是任意类型，那么ArbitraryType这个类型，在Go 语言系统中，比interface{}还要随意。
- `Pointer`: Pointer 以 ArbitraryType指针类型为基础的新类型, 可以理解为c语言的 `void*`。它可以让任意类型的指针实现相互转换，也可以将任意类型的指针转换为 uintptr 进行指针运算
- `Alignof()`: 返回变量对齐字节数量
- `Offsetof()`: 返回变量指定属性的偏移量，所以如果变量是一个struct类型，不能直接将这个struct类型的变量当作参数，只能将这个struct类型变量的属性当作参数。
- `Sizeof()`:  返回变量在内存中占用的字节数，切记，如果是slice，则不会返回这个slice在内存中的实际占用长度。


## Pointer 操作

uintptr是Go 语言的内置类型，是能存储指针的整型， uintptr 的底层类型是int，它和unsafe.Pointer可相互转换。

提供四类操作:

- `任何类型的指针` 转换为 `Pointer`
- `Pointer` 转换为 `任何类型的指针` 
- `uintptr` 转换为 `Pointer`
- `Pointer` 转换为 `uintptr`

### 转换 `*T1`的指针 为 指向 `*T2`的Pointer

如果T2所占内存空间不超过T1, 且它们能适应相同的内存数据结构，则可以进行类型转换.

```go
func Float64bits(f float64) uint64 {
  return *(*uint64)(unsafe.Pointer(&f))
}
```


## uintptr和unsafe.Pointer的区别:
- unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算；
- 而uintptr是用于指针运算的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象， uintptr 类型的目标会被回收；
- unsafe.Pointer 可以和 普通指针 进行相互转换；
- unsafe.Pointer 可以和 uintptr 进行相互转换。

## FAQ

### 为什么要内存对齐?
- 很多CPU只从对齐的地址开始加载数据，而有的CPU这样做，只是更快一点。
- 外部总线从内存一次获取的数据往往不是1byte，而是4bytes或许8bytes,或者更多

这意味着CPU不是一次仅仅抓取一个byte，而是很4个或者8个byte。
这样的话，假设你请求2到3bytes的内容，其实CPU一次抓取了4或8byte的内容。

内存地址详细的分析可以参考<sup>4</sup>.


## Reference

1. Matt Layher, unsafe.Pointer and system calls, https://blog.gopheracademy.com/advent-2017/unsafe-pointer-and-system-calls/
2. package unsafe, https://golang.org/pkg/unsafe/
3. Go Slices: usage and internals, https://blog.golang.org/go-slices-usage-and-internals
4. C语言字节对齐问题详解, https://www.cnblogs.com/clover-toeic/p/3853132.html

