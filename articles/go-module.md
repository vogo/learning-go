<!---
markmeta_author: wongoo
markmeta_date: 2018-12-21
markmeta_title: Go Module
markmeta_categories: 编程语言
markmeta_tags: golang,module
-->

# GO Module

一个模块是相关Go包的集合。
modules是源代码交换和版本控制的单元。 
go命令直接支持使用modules，包括记录和解析对其他模块的依赖性。
modules替换旧的基于GOPATH的方法来指定在给定构建中使用哪些源文件。

go module从1.11开始支持，需要[下载](https://golang.org/dl/)版本；

## GO111MODULE 

取值为 off, on, or auto (默认值，因此前面例子里需要注意2个重点)。

- off: GOPATH mode，查找vendor和GOPATH目录
- on：module-aware mode，使用 go module，忽略GOPATH目录
- auto：如果当前目录不在$GOPATH 并且 当前目录（或者父目录）下有go.mod文件，则使用 GO111MODULE， 否则仍旧使用 GOPATH mode。



## 基本功能

```
> go help mod

Go mod provides access to operations on modules.

Note that support for modules is built into all the go commands,
not just 'go mod'. For example, day-to-day adding, removing, upgrading,
and downgrading of dependencies should be done using 'go get'.   <--- 使用 go get 管理依赖
See 'go help modules' for an overview of module functionality.

Usage:

	go mod <command> [arguments]

The commands are:
        download    download modules to local cache (下载依赖的module到本地cache))
        edit        edit go.mod from tools or scripts (编辑go.mod文件)
        graph       print module requirement graph (打印模块依赖图))
        init        initialize new module in current directory (再当前文件夹下初始化一个新的module, 创建go.mod文件))
        tidy        add missing and remove unused modules (增加丢失的module，去掉未用的module)
        vendor      make vendored copy of dependencies (将依赖复制到vendor下,只会下载你代码中引用的库)
        verify      verify dependencies have expected content (校验依赖)
        why         explain why packages or modules are needed (解释为什么需要依赖)

Use "go help mod <command>" for more information about a command.
```
## go.mod 语法

每个模块下有两个文件:
- go.mod : 项目依赖配置
- go.sum : 依赖项目的hash值 

go.mod 格式如下:
```
module my/project

require other/project v1.0.2
require new/project v2.3.4


exclude old/project v1.2.3

replace bad/project v1.4.5 => good/project v1.4.5
replace golang.org/x/text => github.com/golang/text v0.3.0
```

相同的部分可以写在一起:
```
require (
	other/project v1.0.2
	new/project v2.3.4
)
```

部分特殊格式:
```
require (
	github.com/natefinch/lumberjack v2.0.0+incompatible
	github.com/golang/mock v1.2.0 // indirect
)
```
部分特殊格式:
- `+incompatible`: 不兼容的
- `indirect`: 间接依赖项

## 版本编号

可以通过 `go get github.com/group/project@<version>` 添加依赖项目, 其中`<version>`的格式可以是:

- latest: 最新版本，如果有release版本则是最新的release版本；否则为master的最新commit版本；
- <version>

部分特殊格式:
```
require (
	gopkg.in/yaml.v2 <=v2.2.1   
	gopkg.in/tomb.v1 v1.0.0-20141024135613-dd632973f1e7
	golang.org/x/time v0.0.0-20181108054448-85acf8d2951c // indirect
	github.com/natefinch/lumberjack v2.0.0+incompatible
	github.com/golang/mock v1.2.0 // indirect
)
```
部分特殊格式:
- `<=v2.2.1`: 小于特定版本号
- 

## go get 升级
- go get -u : 将会升级到最新的次要版本或者修订版本(x.y.z, z是修订版本号， y是次要版本号)
- go get -u=patch : 将会升级到最新的修订版本
- go get package@version : 将会升级到指定的版本号version
