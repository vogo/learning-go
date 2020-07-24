<!---
markmeta_author: wongoo
markmeta_date: 2019-06-20
markmeta_title: Go 内存优化
markmeta_categories: 编程语言
markmeta_tags: golang,optimization
-->

# go 内存优化

- 自定义内存复用接口
- 优化[]byte的获取和回收
- 使用writev替代write减少内存分配和拷贝,减少锁力度；

## Reference



