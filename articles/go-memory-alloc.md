<!---
markmeta_author: wongoo
markmeta_date: 2019-10-24 
markmeta_title: Go 内存分配
markmeta_categories: 编程语言
markmeta_tags: golang,module
-->

# GO 内存分配

## 分配堆外内存

```go
const chunkSize = 64 * 1024

const chunksPerAlloc = 1024

// Allocate offheap memory, so GOGC won't take into account cache size.
// This should reduce free memory waste.
data, err := syscall.Mmap(-1, 0, chunkSize*chunksPerAlloc, syscall.PROT_READ|syscall.PROT_WRITE, syscall.MAP_ANON|syscall.MAP_PRIVATE)
if err != nil {
	panic(fmt.Errorf("cannot allocate %d bytes via mmap: %s", chunkSize*chunksPerAlloc, err))
}
```
