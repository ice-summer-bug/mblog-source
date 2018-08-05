---
layout: post
title: golang 环境变量和 golang 脚本工具
categories: golang
tags: golang
date: 2018-01-10 21:00:00
description: golang 的环境变量和脚本工具介绍
---

# golang 环境变量

想要了解 go 的环境变量，我们可以通过 `go help environment` 指令来查看详细介绍，这里尝试翻译这一些详细介绍，并给出一些个人的认识，如有错漏，欢迎指正

## 通用环境变量(General-purpose environment variables)

### GCCGO
`gccgo` 属于 `gcc` 编译器集合，是 `gcc` 针对go 语言的前端实现；`gccgo` 的编译速度比gc较慢一点，但是可以生成更优的代码，因此程序执行速度会更快。
golang 的默认编译器是 `gc`, `gc` 编译器已支持主流的处理器，而 `gccgo` 也对 `gc` 不支持的处理器进行了支持测试；
通过Go正式版本安装的go命令已经可以支持 `gccgo`，需要使用 -compiler选项：`go build -compiler=gccgo` 。
对于用户，如果需要更好编译优化，或者是使用 `gc` 所不支持的处理器或操作系统，`gccgo` 可能是一个更好的选择。

### GOARCH
编译源代码的机器的处理器架构，它的值可以是 386、amd64 或 arm。

### GOBIN
`go install` 编译出来的可执行文件的存放位置，`GOBIN` 的默认值是`GOPATH/bin`
如果 `GOBIN` 设置了值，编译出来的可执行脚本将放置到到 `GOBIN` 设置的文件夹，而不是 .go 文件所在的src 文件夹同级的 bin 文件夹

### GOOS
编译源代码的机器的操作系统, 它的值可以是 linux, darwin, windows, netbsd。

### GOPATH

`GOPATH` 列举了机器上所有go代码所在位置，在Unix系统中，该值是以冒号分隔的字符串。
在Unix 系统中`GOPATH` 默认值是 `%HOME/go`， windows 系统中默认值是 `%USERPROFILE%\go`
`GOPATH` 中文件夹结构如下:

				GOPATH=/home/user/go

				/home/user/go/
		         src/
		             foo/
		                 bar/               (go code in package bar)
		                     x.go
		                 quux/              (go code in package main)
		                     y.go
		         bin/
		             quux                   (installed command)
		         pkg/
		             linux_amd64/
		                 foo/
		                     bar.a          (installed package object)


`src`: 存放 go 源文件
`bin`: package main 中的go 文件编译之后产生的可执行文件存放位置
`pkg`: 非 package main 中的go 文件编译之后产生的静态库文件(\*.a)存放位置

设计开发中，需要将所有存放go 代码的位置都添加到 `GOPATH` 中去

```
export GOPATH=:
```

### GORACE
竞争监测相关参数，详见 https://golang.org/doc/articles/race_detector.html.
_待补充_

### GOROOT
go 安装目录

### GOTMPDIR
The directory where the go command will write
temporary source files, packages, and binaries.

### GOCACHE
存放go 编译系统编译过程中产生的缓存文件，如果这个文件过大，可以执行 `go clean --cache` 去清理这个文件夹

## cgo 相关环境变量

### CC
`cgo` 编译 c语言代码时候使用的编译器, 需要用户额外安装

### CGO_ENABLED
是否支持 cgo， 取值是 0 或者 1

### CGO_CFLAGS
cgo 编译 c 代码时传递的参数

### CGO_CFLAGS_ALLOW
出于安全考虑，cgo 编译 c 代码时只能允许有限的参数， `CGO_CFLAGS_ALLOW` 的取值是一个 `正则表达式`，涵盖所有允许的参数名称

### CGO_CFLAGS_DISALLOW

和 `CGO_CFLAGS_ALLOW` 相反，`CGO_CFLAGS_ALLOW` 的取值也是一个 `正则表达式`，涵盖所有不允许的参数名称

### CGO_CPPFLAGS, CGO_CPPFLAGS_ALLOW, CGO_CPPFLAGS_DISALLOW
类似于 `CGO_CFLAGS`, `CGO_CFLAGS_ALLOW`, `CGO_CFLAGS_DISALLOW`, 不过是用于 `c` 预处理器

### CGO_CXXFLAGS, CGO_CXXFLAGS_ALLOW, CGO_CXXFLAGS_DISALLOW
类似于 `CGO_CFLAGS`, `CGO_CFLAGS_ALLOW`, `CGO_CFLAGS_DISALLOW`, 不过是用于 `c++` 编译器

### CGO_FFLAGS, CGO_FFLAGS_ALLOW, CGO_FFLAGS_DISALLOW
类似于 `CGO_CFLAGS`, `CGO_CFLAGS_ALLOW`, `CGO_CFLAGS_DISALLOW`, 不过是用于 `Fortran` 编译器

### CGO_LDFLAGS, CGO_LDFLAGS_ALLOW, CGO_LDFLAGS_DISALLOW
类似于 `CGO_CFLAGS`, `CGO_CFLAGS_ALLOW`, `CGO_CFLAGS_DISALLOW`, 不过是用于 `c` 链接器

### CXX
cgo 编译 `C++`的编译器

### PKG_CONFIG
取值是 指向 `pkg_config` 工具的绝对路径

## 架构相关的特殊目的的环境变量

### GOARM
当 `GOARCH=arm` 时，arm 架构的处理器，它的取值是 5,6,7

### GO386
当 `GOARCH=386` 时，浮点指令集，它的取值是  387, sse2

### GOMIPS
当 `GOARCH=mips{,le}` 时，指定是软浮点还是硬浮点

## 特殊目的的环境变量

### GOROOT_FINAL
当go 根目录和go 安装目录不一致时，将 `GOROOT_FINAL` 设置为go 的根目录

### GIT_ALLOW_PROTOCOL
`go get` 指令使用 `git fetch/clone` 获取 go 代码的时候允许使用的 `schema`, 多个 `schema` 用冒号分割；
如果 `GIT_ALLOW_PROTOCOL` 不包含某个 `schema`， `go get` 认为它是不安全的

### GO_EXTLINK_ENABLED
Whether the linker should use external linking mode
when using -linkmode=auto with code that uses cgo.
Set to 0 to disable external linking mode, 1 to enable it.



# go 提供的脚本工具

# gofmt
格式化 go 语言代码

				使用示例: gofmt [flags] [path ...]
				-cpuprofile string
					write cpu profile to this file
				-d	在控制台输出格式化后的代码和源代码的对比
				-e	report all errors (not just the first 10 on different lines)
				-l	list files whose formatting differs from gofmt's
				-r string
					重写规则，如 'a[b:len(a)] -> a[b:]' 将 `a[b:len(a)]` 替换成 `a[b:len(a)]`
				-s	简化源代码
				-w	将格式化结果写回源文件，而不是输出到控制台

### go install

### go get

### go build

#### 交叉编译 (cross compile)

Golang 支持交叉编译，在一个平台上生成另一个平台的可执行程序，这里备忘一下。

Mac 下编译 Linux 和 Windows 64位可执行程序

				CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
				CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go

Linux 下编译 Mac 和 Windows 64位可执行程序

				CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
				CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go

Windows 下编译 Mac 和 Linux 64位可执行程序

					SET CGO_ENABLED=0
					SET GOOS=darwin
					SET GOARCH=amd64
					go build main.go

					SET CGO_ENABLED=0
					SET GOOS=linux
					SET GOARCH=amd64
					go build main.go


GOOS：目标平台的操作系统（darwin、freebsd、linux、windows）
GOARCH：目标平台的体系架构（386、amd64、arm）
交叉编译不支持 CGO 所以要禁用它

### gdb
