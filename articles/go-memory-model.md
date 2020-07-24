<!---
markmeta_author: wongoo
markmeta_date: 2018-12-21
markmeta_title: Go内存模型
markmeta_categories: 编程语言
markmeta_tags: golang,memory
-->

# Go内存模型

原文《[The Go Memory Model](https://golang.org/ref/mem)》, 已有[中文翻译](https://go-zh.org/ref/mem)，整体翻译不错，但主要是一一对照翻译，有些翻译有错，加上英文版本有些地方写得也很教材化，翻译有些晦涩难懂，所以想自己来翻译一下，按照理解进行翻译，对重点内容进行译注说明，希望能做到简单明了。

文章将对一些关键字不进行翻译：
- goroutine
- channel


# 1. 介绍

本文说明如何确保一个goroutine能够读取到另一个goroutine对一个变量写入的值。

# 2. 忠告

如果存在多个goroutine同时修改数据的情况，则需要串行化请求来保护数据。串行化可以通过channel信道或如sync、sync/atomic包下的同步工具来实现。

如果你需要读本文才能理解你的程序的行为，你正变得更聪明，但别自作聪明。

> 译注: 你应该保持你的代码清晰简洁, 要明白语言同步特性，不要滥用，适得其反。

# 3. 事件顺序

在单个goroutine中，读取和写入的表现必须与代码指定的执行顺序相一致。按照Go语言规范，编译器和处理器可能会对单个goroutine中的读取和写入的代码进行重新排序，只要重新排序不会改变goroutine的行为。由于存在重新排序，一个goroutine中的执行顺序可能和另外一个goroutine中的执行顺序不一致。例如一个goroutine执行 `a = 1; b = 2;` 这个语句，另外一个goroutine可能会监测到变量`b`比变量`a`先更新。


为了详细论述读取和写入的必要条件，我们定义了`事件发生顺序`的概念，它表示Go程序中内存操作执行的 偏序关系（`partial order`）。 若事件 e1 发生在 e2 之前， 那么我们就说 e2 发生在 e1 之后。 换言之，若 e1 既未发生在 e2 之前， 又未发生在 e2 之后，那么我们就说 e1 与 e2 是并发的。

**单个goroutine中事件发生的顺序即为程序所表达的顺序**。

若以下条件均成立，则对变量 v 的读取操作 r 就可以监测到对变量 v 的写入操作 w ：
- r 不发生在 w 之前。
- 在 w 之后 r 之前，不存在其它对 v 进行的写入操作 w'。


为确保对变量 v 的读取操作 r 能够监测到特定的对 v 进行写入的操作 w，需确保 w 是唯一允许被 r 监测的写入操作，也就是以下条件均成立：
- w 发生在 r 之前。
- 对共享变量 v 的其它任何写入操作都只能发生在 w 之前或 r 之后。

这对条件的要求比第一对更强，它需要确保没有其它写入操作与 w 或 r 并发。

在单个goroutine中并不存在并发，因此这两条定义是等价的：读取操作 r 可监测最近的写入操作 w 对 v 写入的值。当多个goroutine访问共享变量 v 时，它们必须通过同步事件来建立顺序条件，以此确保读取操作能监测到预期的写入。

以变量 v 所属类型的零值来对 v 进行初始化，其表现如同在内存模型中进行的写入操作。

对大于单个机器字的值进行读取和写入，其表现如同以不确定的顺序对多个机器字大小的值进行操作。

```
# https://go-zh.org/ref/mem
单goroutine的情形：
-- w0 ---- r1 -- w1 ---- w2 ----  r2 ---- r3 ------>

这里不仅是个偏序关系，还是一个良序关系：所有 r/w 的先后顺序都是可比较的。

双goroutine的情形：
-- w0 -- r1 -- r2 ---- w3 ----  w4 ---- r5 -------->
-- w1 ----- w2 -- r3 ----  r4 ---- w5 -------->

单goroutine上的事件都有先后顺序；而对于两条goroutine，情况又有所不同。即便在时间上 r1 先于 w2 发生，
但由于每条goroutine的执行时长都像皮筋一样伸缩不定，因此二者在逻辑上并无先后次序。换言之，即二者并发。
对于并发的 r/w，r3 读取的结果可能是前面的 w2，也可能是上面的 w3，甚至 w4 的值；
而 r5 读取的结果，可能是 w4 的值，也能是 w1、w2、w5 的值，但不可能是 w3 的值。


双goroutine交叉同步的情形：
-- r0 -- r1 ---|------ r2 ------------|-- w5 ------>
-- w1 --- w2 --|-- r3 ---- r4 -- w4 --|------->

现在上面添加了两个同步点，即 | 处。这样的话，r3 就是后于 r1 ，先于 w5 发生的。
r2 之前的写入为 w2，但与其并发的有 w4，因此 r2 的值是不确定的：可以是 w2，也可以是 w4。
而 r4 之前的写入的是 w2，与它并发的并没有写入，因此 r4 读取的值为 w2。
```

# 4. 同步

## 4.1 初始化

程序的初始化运行在单个goroutine中，但该Go程可能会创建其它并发运行的gouroutine。

**若包 p 导入了包 q，则 q 的 init 函数会在 p 的任何函数开始前完成。**

**函数 main.main 会在所有的 init 函数结束后开始。**


## 4.2 Goroutine 创建

启动一个新的goroutine的go语句执行后，这个新的goroutine才开始执行。
> 译注: goroutine创建后，需要runtime去调度才会执行。

例如:

```
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```
调用`hello()`方法将在之后的某一个时间点打印`“hello, world”`（可能是`hello()`方法已经返回了之后）。

## 4.3 Goroutine 销毁

一个goroutine不保证在程序的任何事件之前退出，如下:

```
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

对变量a的赋值没有添加在同步事件之后，就不能保证能被其他goroutine监测到。实际上，一个积极的编译器可能会将整个go语句都删掉。

要确保一个goroutine被其他goroutine监测到，需要使用类似lock或channel通信的同步机制来建立顺序关系。


## 4.4 Channel 通信

channel通信是goroutine之间主要的同步方式。一个channel上的每一个send动作都对应这个channel上的一个receive动作, send和receive通常在不同的goroutine中.

**(缓冲的)channel上的一个send发生在对应的receive完成前。**

> 原文: A send on a channel happens before the corresponding receive from that channel completes.

> 译注: 注意这里是`receive完成`前, 而不是`receive开始`前。channel通信是通过共享内存的方式实现，send动作会去唤醒receive的goroutine变为ready状态，send还没完receive可能已经开始了，channel的内部结构hchan本身包含锁机制，可以保证send对hchan的修改在receive的修改之前完成。

例如:

```
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```

以上代码确保可以打印出"hello, world"：
- 对变量`a`的写入发生在对channel `c`的send之前；
- 对channel `c` 的send动作又发生在`c`的receive动作完成之前；
- receive动作又发生在print打印之前。

如果在receive之前close了channel，因为channel已经关闭，所以receive将接收到零值。

上一个例子中，替换`c <- 0`为`close(c) `可以同样保证相同的行为。


**对于非缓冲的channel，receive动作发生在send完成之前。**
> 原文: A receive from an unbuffered channel happens before the send on that channel completes.

> 译注: 注意这里和缓冲channel的差异，非缓冲channel的receive动作比send先完成，而缓冲channel的send动作比receive先完成。

以下例子同上例, 只是send和receive语句互换，并且使用非缓冲channel:

```
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}
func main() {
	go f()
	c <- 0
	print(a)
}
```

这个例子也能保证打印出"hello, world"：
- 对变量a的写入发生在对c的receive之前,
- 对c的receive又发生在对c的send之前，
- 对c的send又发生在print之前。

如果channel是缓冲的(例如`c = make(chan int, 1)`),则该程序不能保证打印出 "hello, world"，有可能打印空字符串，或者崩溃，或者其他情况。
> 译注: 打印空字符串比较好理解，至于`崩溃`或`其他情况` 请参考[这里](https://stackoverflow.com/questions/55372640/trying-to-understand-golang-chan-cause-crash-or-do-something-else)


**对于容量为C的缓冲channel，第k次receive发生在第k+C次send完成之前。**
> 原文: The kth receive on a channel with capacity C happens before the k+Cth send from that channel completes.

> 译注: 这句话有点教科书式，较难理解。缓冲 channel 类似一个有容量的队列。当队列满的时候send goroutine会阻塞；当队列空的时候receive goroutine会阻塞。假设channel已经满了，也就是已经有C次的send动作，如果再进行第1+C次send，这次send的goroutine就会阻塞并被加到send等待队列中。当第1次receive发生的时候，会唤醒被阻塞的第1+C次send的 goroutine开始执行。当第1次receive动作完成并释放锁后，就可以让第1+C次的send完成动作。


这个规则是对于前面缓冲channel规则的概括。即缓冲channel建模了一个计数信号量：channel中元素个数表示channel正被使用的数量, channel的容量决定了最大同时使用数。send一个元素需要获取信号量,receive一个元素释放信号量。这就是一般用于限制并发的做法。


以下代码对于每一个work列表项启动一个goroutine，这些goroutine通过一个channel限制，确保同一时间只有3个工作项在执行。

```
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

## 4.5 锁

`sync`包内实现了两种锁 `sync.Mutex` 和 `sync.RWMutex`。



**对于sync.Mutex 或 sync.RWMutex 的变量 l，当n < m时，第n次调用`l.Unlock()`发生在第m次调用`l.Lock()`返回之前。**
> 原文: For any sync.Mutex or sync.RWMutex variable l and n < m, call n of l.Unlock() happens before call m of l.Lock() returns.

> 译注：`l.Lock()` 和 `l.Unlock()` 必须是成对依次调用的，下一次（或后续的某一次第m次）的`l.Lock()` 必须在上一次 `l.Unlock()` （第n次）完成之后才能完成。 `l.Lock()` 返回之前，上一次`l.Unlock()`就肯定已经先返回完成了。 这个m不一定等于n+1, 也就是同时有多个`l.Lock()`加锁请求 n+1,n+2 ... m,m+1 ...，但第n次解锁后，后续的随机某一个加锁请求成功了。

```
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

以上代码可以确保打印出"hello, world"。
- 第1次调用f()中`l.Unlock()`发生在main中第2次`l.Lock()`返回之前；
- 第2次`l.Lock()`又发生在print之前。


**对于`sync.RWMutex`类型变量`l`,存在n,使得 `l.RLock`调用返回发生在第n次调用`l.Unlock`之后，对应`l.RUnlock`发生在第n+1次调用`l.Lock`之前。**
> 原文: For any call to l.RLock on a sync.RWMutex variable l, there is an n such that the l.RLock happens (returns) after call n to l.Unlock and the matching l.RUnlock happens before call n+1 to l.Lock.

> 译注: 读锁(read lock)加锁`l.RLock`和解锁`l.RUnlock`成对依次出现，可并行多个；写锁(write lock)加锁`l.Lock`和解锁`l.Unlock`成对依次出现，并行只能有一个。但`读锁对`和`写锁对`不能并行，也就是同一时间只能有read lock或者write lock。以上这句话主要是说明write lock解锁后才能加read lock, 相应read lock解锁后才能加write lock。

## 4.6 Once单次锁

sync包通过Once单次锁类型提供一种安全的多goroutine初始化机制。多个线程(`译注:原文是threads，而不是goroutine`) 执行`once.Do(f)`调用方法f, 但只会有一个会执行`f()`方法，其他的调用会在`f()`方法调用返回之前被阻塞住。


**通过`once.Do(f)`单次调用`f()`方法发生在任何调用`once.Do(f)`返回之前。**
> 原文: A single call of f() from once.Do(f) happens (returns) before any call of once.Do(f) returns.

> 译注: `once.Do(f)`保证只会有一个真正调用`f()`, 方法`f()`调用完成返回后，所有的once.Do(f)调用才会完成返回。

```
var a string
var once sync.Once

func setup() {
	a = "hello, world"
}

func doprint() {
	once.Do(setup)
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

调用`twoprint`方法会打印 "hello, world" 两次. 只有第一次调用`doprint` 会执行`setup`一次.
> 译注: 原文`The first call to doprint runs setup once`可能描述不准确，应该是只有第一次调用`once.Do(setup)` 会执行`setup`一次.

# 5. 错误的同步

注意，虽然某一次读r可能会监测到和它同步的某一次写w，但这并不表明r之后的读能够监测到在w之前的写。

请看以下例子：

```
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```
这个例子g()方法可能会先打印2然后打印0。

**这个事实让一些惯用语法失效了。**

请反复检查锁机制避免同步过多反而无效。例如，`twoprint`的例子可能错误的写为如下的形式:

```
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```
这里并不保证在 `doprint` 方法中通过监测对变量`done`的写来间接监测到对变量`a`的写。这个版本可能会错误的打印空字符串而不是"hello, world"。

另外一种错误的语法是忙等一个值，例如：

```
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

如上一个例子，这个例子也无法确保main方法中通过监测对变量`done`的写来间接监测到对变量`a`的写，这个程序也可能打印空字符串。
可能更糟糕的是，也不能确保对变量`done`的写能够被`main`监测到，因为两个goroutine之间没有同步事件。`main`方法中的循环不保证会结束。
> 译注: 其中 `不能确保对变量done的写能够被main监测到，因为两个goroutine之间没有同步事件` 需注意，在多核cpu中每个cpu有自己的高速缓存，变量会被拷贝到高速缓存中再进行操作。如果main goroutine的运行cpu已经获得done的值并拷贝到cpu高速缓存中，另外一个goroutine运行cpu对变量done的修改最终会更新到主存，但main goroutine运行cpu并不知道这个改变，不会更新自身缓存中的值，就会造成无限循环下去。可以通过atomic原子读取更新解决这个问题。 参考[内存模型](https://doc.sisopipo.com/#/cs/memory-model)以及[CAS](https://doc.sisopipo.com/#/cs/cas)。


类似还有一些微妙的变体版本，例如这段代码:

```
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```
尽管如果`main`方法能够监测到`g != nil`退出了循环，但也不能保证能够监测到`g.msg`的初始化值。

在以上所有错误的例子，解决方案都是一样的：**明确使用同步语法！**


# A. 参考
1. The Go Memory Model, https://golang.org/ref/mem
2. source code of golang channel,  https://github.com/golang/go/blob/master/src/runtime/chan.go
3. Go内存模型, https://go-zh.org/ref/mem
4. 深入理解 Go Channel, http://legendtkl.com/2017/07/30/understanding-golang-channel/
5. Go Channel 源码剖析，http://legendtkl.com/2017/08/06/golang-channel-implement/
6. Golang 内存管理, http://legendtkl.com/2017/04/02/golang-alloc/

# B. 编辑历史
1. 2019-02-15, wangoo, 修改译注，说明多核cpu可能无法读取到变量更改的原因
2. 2018-12-06, wangoo, 调整格式,修改文字错误，修改部分译注说明
3. 2018-07-19, wangoo, 初版




