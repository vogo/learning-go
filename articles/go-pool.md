<!---
markmeta_title: golang pool
markmeta_author: 望哥
markmeta_date: 2020-01-09
markmeta_tags: golang,pool
markmeta_categories: lang
-->

# golang pool

## 1. sync.Pool

golang sdk已经提供了性能优良的pool工具 `sync.Pool`, 在实践过程需要使用pool的尽量使用`sync.Pool`即可。

`sync.Pool`关键实现：

- MPG模型P级别单对象缓存,大多情况都直接返回，不存在竞争；
- MPG模型P级别池对象链表,取头部对象,也不存在竞争；
- 只有在P级别自身池对象链表没有的情况，从其他P的池对象链表尾部CAS方式偷取对象，尽量减少竞争；
- 如果所有P对象链表都无法获得对象，则New一个对象；
- runtime级别支持GC垃圾回收池对象；

`sync.Pool`的特点:

- runtime级别最小化同步竞争，性能极好；
- 支持GC对象回收，资源利用伸缩自如；

`sync.Pool`使用注意:

- 应该使用指针作为池对象，避免潜在的对象复制；
- 如果缓存slice需要判断传入slice的容量cap，缓存一致大小的slice避免出现内存泄露；

