<!---
markmeta_author: wongoo
markmeta_date: 2019-01-16
markmeta_title: Go Build
markmeta_categories: 编程语言
markmeta_tags: golang,build
-->

# go build

## 删除调试符号

```bash
go build -ldflags "-s -w" [<your/package]
```

这里 `-ldflags` 参数最终会在 `go tool link` 的时候传给它， `go tool link -h` 解释如下

```
 -s    disable symbol table
 -w    disable DWARF generation
```

删除掉调试符号的另一个好处就是，显著减小了文件大小（平均20%）.

## 静态编译

go程序编译后的程序依赖glibc, 如果需要完全不依赖第三方库，需要静态编译，这时候需要glibc-static库。

```bash
CGO_ENABLED=0 go build -a -ldflags '-extldflags "-static"' .
```

## 二进制文件压缩

加一个UPX壳，还可以进一步压缩原文件大小。

```bash
# 压缩二进制文件
./upx <BINARY_FILE>

# 解压文件
./upx -d <BINARY_FILE>
```


## Reference

1. [DWARF](http://dwarfstd.org), Debugging With Attributed Record Formats. DWARF is a debugging file format used by many compilers and debuggers to support source level debugging. 
2. [upx](https://github.com/upx/upx), UPX is an advanced executable file compressor. UPX will typically reduce the file size of programs and DLLs by around 50%-70%, thus reducing disk space, network load times, download times and other distribution and storage costs. Programs and libraries compressed by UPX are completely self-contained and run exactly as before, with no runtime or memory penalty for most of the supported formats.


