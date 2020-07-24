<!---
markmeta_title: 解读golang gctrace 信息
markmeta_date: 2018-09-11 
markmeta_tags: golang,gc
markmeta_categories: 编程语言
markmeta_author: 望哥
-->

# 解读golang gctrace 信息

##  如何开启打印GC信息

只要在程序执行之前加上环境变量`GODEBUG=gctrace=1` ，如：
```
GODEBUG=gctrace=1 ./xxxx
```

或者:
```
GODEBUG=gctrace=1 go run main.go
```

程序将会显示GC信息，如下:

```
gc 1 @2.104s 0%: 0.018+1.3+0.076 ms clock, 0.054+0.35/1.0/3.0+0.23 ms cpu, 4->4->3 MB, 5 MB goal, 4 P
gc 2 @2.241s 0%: 0.019+2.4+0.077 ms clock, 0.079+0/2.4/6.4+0.30 ms cpu, 5->6->5 MB, 6 MB goal, 4 P
gc 3 @2.510s 0%: 0.011+3.2+0.063 ms clock, 0.047+0.10/2.9/9.0+0.25 ms cpu, 11->11->10 MB, 12 MB goal, 4 P
gc 4 @3.021s 0%: 0.013+6.6+0.076 ms clock, 0.053+0.34/6.2/18+0.30 ms cpu, 21->21->20 MB, 22 MB goal, 4 P
gc 5 @3.725s 0%: 0.015+15+0.079 ms clock, 0.062+0.35/15/45+0.31 ms cpu, 40->40->39 MB, 41 MB goal, 4 P
gc 6 @4.741s 0%: 0.008+35+0.17 ms clock, 0.035+0.19/35/100+0.70 ms cpu, 76->76->75 MB, 78 MB goal, 4 P
gc 7 @6.688s 0%: 0.020+117+0.34 ms clock, 0.082+11/117/330+1.3 ms cpu, 147->148->146 MB, 151 MB goal, 4 P
gc 8 @68.645s 0%: 0.019+146+0.30 ms clock, 0.078+0.006/146/407+1.2 ms cpu, 285->285->248 MB, 292 MB goal, 4 P
scvg0: inuse: 426, idle: 0, sys: 427, released: 0, consumed: 427 (MB)
gc 9 @175.448s 0%: 0.030+60+0.12 ms clock, 0.12+0.013/60/177+0.51 ms cpu, 484->484->248 MB, 496 MB goal, 4 P
gc 10 @236.621s 0%: 0.006+59+0.11 ms clock, 0.025+0/59/173+0.47 ms cpu, 484->484->248 MB, 496 MB goal, 4 P
gc 11 @285.967s 0%: 0.027+57+0.22 ms clock, 0.11+0/57/163+0.89 ms cpu, 484->484->248 MB, 496 MB goal, 4 P
scvg1: inuse: 331, idle: 175, sys: 507, released: 0, consumed: 507 (MB)
gc 12 @333.817s 0%: 0.009+52+0.18 ms clock, 0.036+0/52/155+0.72 ms cpu, 484->484->248 MB, 496 MB goal, 4 P
```

## GC信息的含义

查看官方的[runtime说明](https://golang.org/pkg/runtime/)：

```
gctrace: setting gctrace=1 causes the garbage collector to emit a single line to standard
error at each collection, summarizing the amount of memory collected and the
length of the pause. Setting gctrace=2 emits the same summary but also
repeats each collection. The format of this line is subject to change.
Currently, it is:
	gc # @#s #%: #+#+# ms clock, #+#/#/#+# ms cpu, #->#-># MB, # MB goal, # P
where the fields are as follows:
	gc #        the GC number, incremented at each GC
	@#s         time in seconds since program start
	#%          percentage of time spent in GC since program start
	#+...+#     wall-clock/CPU times for the phases of the GC
	#->#-># MB  heap size at GC start, at GC end, and live heap
	# MB goal   goal heap size
	# P         number of processors used
The phases are stop-the-world (STW) sweep termination, concurrent
mark and scan, and STW mark termination. The CPU times
for mark/scan are broken down in to assist time (GC performed in
line with allocation), background GC time, and idle GC time.
If the line ends with "(forced)", this GC was forced by a
runtime.GC() call and all phases are STW.

Setting gctrace to any value > 0 also causes the garbage collector
to emit a summary when memory is released back to the system.
This process of returning memory to the system is called scavenging.
The format of this summary is subject to change.
Currently it is:
	scvg#: # MB released  printed only if non-zero
	scvg#: inuse: # idle: # sys: # released: # consumed: # (MB)
where the fields are as follows:
	scvg#        the scavenge cycle number, incremented at each scavenge
	inuse: #     MB used or partially used spans
	idle: #      MB spans pending scavenging
	sys: #       MB mapped from the system
	released: #  MB released to the system
	consumed: #  MB allocated from the system
```

## 垃圾回收信息

```
gc 1 @2.104s 0%: 0.018+1.3+0.076 ms clock, 0.054+0.35/1.0/3.0+0.23 ms cpu, 4->4->3 MB, 5 MB goal, 4 P。
```

- gc 1 : 表示第一次执行
- @2.104s :  表示程序执行的总时间
- 0%  : 垃圾回收时间占用的百分比
- 0.018+1.3+0.076 ms clock :  垃圾回收的时间，分别为0.018ms STW（stop-the-world）清扫的时间, 1.3ms并发标记和扫描的时间，0.076ms STW标记的时间
- 0.054+0.35/1.0/3.0+0.23 ms cpu  : 垃圾回收占用cpu时间, 0.054ms STW清扫占用的cpu时间，0.35ms辅助时间，1.0ms后台gc时间，3.0ms空闲gc时间，0.23ms STW标记终止时间。
- 4->4->3 MB : 堆的大小，gc后堆的大小，存活堆的大小
- 5 MB goal  : 整体堆的大小
- 4 P  : 使用的processor（执行单元）数量

## 系统内存回收信息

这个很直白，看单词就知道大概意思了
```
scvg0: inuse: 426, idle: 0, sys: 427, released: 0, consumed: 427 (MB)
```
- `scvg0` 系统内存回收序号
- `inuse: 426` 使用多少MB内存
- `idle: 0` 剩下要回收的内存
- `sys: 427` 系统映射的内存
- `released: 0` 释放的系统内存
- `consumed: 427 (MB)` 申请的系统内存



## History

1. 2018-09-11, wongoo, created
