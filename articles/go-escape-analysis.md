<!---
markmeta_author: wongoo
markmeta_date: 2020-01-03
markmeta_title: Go 逃逸分析
markmeta_categories: 编程语言
markmeta_tags: golang,escape
-->

# go 逃逸分析

回顾问题:
1. 逃逸机制是什么?
2. 分析逃逸的工具方法？
3. 常见逃逸场景？



## 1. 关于堆和栈

- 栈上分配的内存在方法退出的时候即被释放，无需GC，对性能吴影响
- 堆上分配的内存有助于共享, 由GC进行内存回收；

## 2. 逃逸机制（Escape Mechanics）

（内存的）完整性在于确保对任何值的访问始终是准确、一致和高效的。
任何时候一个值被分享到函数栈帧范围之外，它都会在堆上被重新分配。
这种情况由逃逸分析算法负责发现和控制。

**逃逸分析（Escape analysis）**：指由编译器在编译阶段决定内存分配的位置。

```go
package main

type user struct {
    name  string
}

func main() {
    u1 := createUserV1()
    u2 := createUserV2()

    println("u1", &u1, "u2", &u2)
}

//go:noinline
func createUserV1() user {
    u := user{
        name:  "Bill",
    }

    println("V1", &u)
    return u
}

//go:noinline
func createUserV2() *user {
    u := user{
        name:  "Bill",
    }

    println("V2", &u)
    return &u // 返回指针被外部引用,发生逃逸
}
```

- 使用 go:noinline 指令，阻止在 main 函数中，编译器使用内联代码替代函数调用
- `&` 操作符对应单词 "sharing", 提高可读性


## 3. 编译器报告（Compiler Reporting）

查看编译器（关于逃逸分析）报告: `go build` 的时候，打开 `-gcflags` 开关，并带上 `-m` 选项。
总共可以使用 4 个 `-m`，（但）超过 2 个级别的信息就已经太多了, 一般使用 2 个 `-m` 的级别。

```bash
$ go build -gcflags "-m -m"
./c.go:15:6: cannot inline createUserV1: marked go:noinline
./c.go:25:6: cannot inline createUserV2: marked go:noinline
./c.go:7:6: cannot inline main: function too complex: cost 133 exceeds budget 80
./c.go:26:2: moved to heap: u   --> 说明变量u已经逃逸到堆
```

## 4. 逃逸例子

### 4.1 空间不足逃逸、动态大小逃逸
```go
func F() {
	a := make([]int, 0, 20)     // 空间小，分配于栈空间
	b := make([]int, 0, 20000) // 空间过大,占空间不足, 逃逸，分配于堆空间
 
	l := 20
	c := make([]int, 0, l) // 动态分配不定空间，逃逸，分配于堆空间
}
```

### 4.2 动态类型逃逸

很多函数参数为interface类型，比如fmt.Println(a …interface{})，编译期间很难确定其参数的具体类型，也能产生逃逸。

```go
package main

import "fmt"

func main() {
    s := "Escape"
    fmt.Println(s)
}
```

```bash
go build -gcflags "-m -l"
./c.go:7:16: main ... argument does not escape
./c.go:7:16: s escapes to heap
```

### 4.3 闭包引用对象逃逸

Fibonacci数列的函数：
```go
package main

import "fmt"

func Fibonacci() func() int {
    a, b := 0, 1
    return func() int {
        a, b = b, a+b
        return a
    }
}

func main() {
    f := Fibonacci()

    for i := 0; i < 10; i++ {
        fmt.Printf("Fibonacci: %d\n", f())
    }
}
```

```bash
 go build -gcflags "-m -l"
./c.go:6:5: moved to heap: a
./c.go:6:8: moved to heap: b
./c.go:7:12: func literal escapes to heap
./c.go:17:19: main ... argument does not escape
./c.go:17:40: f() escapes to heap
```
