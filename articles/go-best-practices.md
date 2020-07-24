<!---
markmeta_author: wongoo
markmeta_date: 2019-07-16
markmeta_title: Go Best Practices
markmeta_categories: 编程语言
markmeta_tags: golang,practices
-->

# Go Best Practices

- 不要频繁的使用`<-time.After`,它会创建大量channel和goroutine, 可以使用`time.Ticker`类似时间轮的算法解决；
- 不用频繁的创建`[]byte` 和 `bytes.Buffer`对象, 缓存并重复使用他们；
- 多使用缓存池`sync.Pool`


## Fasthttp best practices

* Do not allocate objects and `[]byte` buffers - just reuse them as much
  as possible. Fasthttp API design encourages this.
* [sync.Pool](https://golang.org/pkg/sync/#Pool) is your best friend.
* [Profile your program](http://blog.golang.org/profiling-go-programs)
  in production.
  `go tool pprof --alloc_objects your-program mem.pprof` usually gives better
  insights for optimization opportunities than `go tool pprof your-program cpu.pprof`.
* Write [tests and benchmarks](https://golang.org/pkg/testing/) for hot paths.
* Avoid conversion between `[]byte` and `string`, since this may result in memory
  allocation+copy. Fasthttp API provides functions for both `[]byte` and `string` -
  use these functions instead of converting manually between `[]byte` and `string`.
  There are some exceptions - see [this wiki page](https://github.com/golang/go/wiki/CompilerOptimizations#string-and-byte)
  for more details.
* Verify your tests and production code under
  [race detector](https://golang.org/doc/articles/race_detector.html) on a regular basis.
* Prefer [quicktemplate](https://github.com/valyala/quicktemplate) instead of
  [html/template](https://golang.org/pkg/html/template/) in your webserver.

