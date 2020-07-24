<!---
markmeta_author: wongoo
markmeta_date: 2019-03-12
markmeta_title: Go scheduler code analysis
markmeta_categories: 编程语言
markmeta_tags: golang,scheduler
-->

# go scheduler code analysis

## 总体介绍

- $GOROOT/src/pkg/runtime目录很重要，值得好好研究，源代码可以从runtime.h开始读起。
- proc.c中是实现的线程调度相关。
- goroutine实现的是自己的一套线程系统，语言级的支持，与pthread或系统级的线程无关。
- 结构体定义在runtime.h中。两个重要的结构体是G和M
- 结构体G名字是goroutine的缩写，相当于操作系统中的进程控制块，在这里就是线程的控制结构，是对线程的抽象。其中包括: goid(线程ID), status(线程状态)，如Gidle,Grunnable,Grunning,Gsyscall,Gwaiting,Gdead等
- 有个常驻的寄存器`extern register G* g` 被使用, 这个是当前线程的线程控制块指针。
- amd64中这个寄存器是使用R15，在x86中使用0(GS)  分段寄存器
- 结构体M名字是machine的缩写。是对机器的抽象，其实是对应到操作系统线程。

## goroutine的生老病死
go关键字最终被弄成了`runtime.newproc`. 就以这个为出发点看整个调度器吧.

`runtime.newproc` 功能是创建一个新的g.这个函数不能用分段栈,真正的工作是调用`newproc1`完成的.
`newproc1` 的动作包括:
1. 分配一个g的结构体
2. 初始化这个结构体的一些域
3. 将g挂在就绪队列
4. 引发一次matchmg

初始化`newg` 的域时,会将调用参数保存到g的栈;将sp,pc等上下文环境保存在g的sched域.这样当这个g被分配了一个m时就可以运行了.

接下来看`matchmg`函数.这个函数就是做个匹配,只要m没有突破上限GOMAXPROCS,就拿一个m绑定一个g.
如果m的waiting队列中有就从队列中拿,否则就要新建一个m,调用`runtime.newm`

runtime.newm 功能跟 newproc 相似,前者分配一个goroutine,而后者是分配一个machine.调用的`runtime.newosproc`函数.
其实一个machine就是一个操作系统线程的抽象,可以看到它会调用`runtime.newosproc`.
这个新线程会以`mstart` 作为入口地址.
当m和g绑定后,mstart会恢复g的sched域中保存的上下文环境,然后继续运行.

> 随便扫一下runtime.newosproc还是蛮有意思的,代码在thread_linux.c文件中(平台相关的),它调用了runtime.clone(平台相关). runtime.clone是用汇编实现的,代码在sys_linux_386.s.可以看到上面有 `INT        $0x80`
看到这个就放心了,只要有一点汇编基础,你懂的.可以看出,go的runtime果然跟c的runtime半毛钱关系都没有啊

回到`runtime.newm` 函数继续看,它调用 `runtime.newosproc` 建立了新的线程,线程是以 `runtime.mstart` 为入口的,那么接下来看`mstart` 函数.

`mstart` 是 `runtime.newosproc` 新建的线程的入口地址,新线程执行时会从这里开始运行.
> 新线程的执行和goroutine的执行是两个概念,由于有m这一层对机器的抽象,是m在执行g而不是线程在执行g.所以线程的入口是mstart,g的执行要到schedule才算入口.函数mstart的最后调用了schedule.

终于到了schedule了!

如果从mstart进入到schedule的,那么schedule中逻辑非常简单,前面省了一大段代码.大概就这几步:
1. 找到一个等待运行的g
2. 将它搬到m->curg,设置好状态为Grunning
3. 直接切换到g的上下文环境,恢复g的执行

goroutine从newproc出生一直到运行的过程分析,到此结束!

虽然按这样a调用b,b调用c,c调用d,d调用e的方式去分析源代码谁看都会晕掉,但我还是想重复一遍这里的读代码过程后再往下写些有意思的,希望真正感兴趣的读者可以拿着注释过的源码按顺序走一遍:
```
newproc -> newproc1 -> newprocreadylocked -> matchmg -> (可能引发)newm -> newosproc -> (线程入口)mstart -> schedule -> gogo跳到goroutine运行
```

以上状态变化经历了`Gwaiting->Grunnable->Grunning`,经历了创建,到挂在就绪队列,到从就绪队列拿出并运行.

下面将从其它几种状态变化继续看调度器,从`runtime.entersyscall`开始.

`runtime.entersyscall` 做的事情大致是设置g的状态为Gsyscall,减少mcpu.
如果mcpu减少之后小于mcpumax了并且有处于就绪态的g,则matchmg

`runtime.exitsyscall` 函数中,如果退出系统调用后mcpu小于mcpumax,直接设置g的状态Grunning.表示让它继续运行.
否则如果mcpu达到上限了,则设置`readyonstop`,表示下一次schedule中将它改成`Grunnable` 了放到就绪队列中

现在`Gwaiting,Grunnable,Grunning,Gwaiting` 都出现过的,接下来看最后两种状态`Gmoribund`和`Gdead`.
看`runtime.goexit`函数.这个函数直接把g的状态设置成`Gmoribund`,然后调用gosched,进入到schedule中.
在schedule中如果遇到状态为Gmoribund的g,直接设置g的状态为Gdead,将g与m分离,把g放回到free队列.


## 参考
- http://bbs.mygolang.com/en/thread-163-1-1.html
