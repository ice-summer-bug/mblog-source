---
layout: post
title: golang 内存管理浅析
categories: golang
tags: [golang, memory management, malloc]
date: 2018-05-16 21:00:00
description: golang 内存管理源码阅读分析
---


## 前言

**_从一个问题和回复开始了解 go语言内存管理_**

### How do I know whether a variable is allocated on the heap or the stack?

        From a correctness standpoint, you don't need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.

        The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function's stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.

        In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.


从 [go语言官网](https://golang.org/doc/faq "go 语言官网") 的问题列表中有上面这个问题，简单翻译如下

### 我们如何知道变量是分配在堆上还是栈上？

    明确地说，你不需要知道答案。 go语言的每个变量当它存在引用的时候它就会一直存在，语言对存储位置的选择与语言的语义无关。

    存储位置确实会影响编写高效的程序。如果可能，Go编译器将在函数的栈中给本地变量分配存储空间。但是，如果编译器在函数返回后无法证明变量未被引用，则编译器必须在堆上分配变量以避免空指针。此外，如果局部变量非常大，将它存储在堆而不是栈上可能更有意义。

    在当前主流的编译器中，如果变量有其他访问地址，则该变量是堆上分配的候选变量。但是，基本的逃逸分析可以识别某些情况，这些生命周期只在函数周期内的变量将分配在栈上。


这个问题和答案告诉我们两个事情，一个是简单介绍了 `逃逸分析`，另一个就是 go语言官方开发者不认为我们需要了解 `内存分配`，现在我们需要去了解 `逃逸分析` 和 `内存管理`。


## 逃逸分析

先看看 `维基百科` 上关于 `逃逸分析` 的说明


        「逃逸分析」是编译程序优化理论中确定指针动态范围的方法 ———— 分析程序哪些地方可以访问指针，它涉及到指针分析和形状分析。
        当一个变量在子程序中被分配时，一个指向变量的指针可能会逃逸到其它执行程序中，或者去调用子程序。如果使用尾递归优化（通常在函数编程语言中是需要的），对象也可能逃逸到被调用的子程序中。 如果一个子程序分配一个对象并返回一个该对象的指针，该对象可能在程序中的任何一个地方被访问到——这样指针就成功“逃逸”了。如果指针存储在全局变量或者其它数据结构中，它们也可能发生逃逸，这种情况是当前程序中的指针逃逸。 逃逸分析需要确定指针所有可以存储的地方，保证指针的生命周期只在当前进程或线程中。

### golang 逃逸分析示例

```go
// go run -gcflags="-m -l" escape_ayalysis_demo.go
func main() {
	s1 := returnString1()
	s2 := returnStringPrt1()
	println("s1:", s1, ", addr: ", &s1, "\ns2:", s2)
}

//go:noinline
func returnString1() (rs1 string) {
	rs1 = "this is a variable in func will return"
	println(rs1)
	return
}

//go:noinline
func returnStringPrt1() (sp1 *string) {
	rspt1 := "this is a variable in func will return"
	println(rspt1)
	return &rspt1
}
```

这里我们使用 `go:noinline` 来禁止编译器使用内联代码来替换函数调用，<br/>
然后我们使用  `gcflags="-m -m"` 来查看编译器报告，`gcflags` 最多可以有四个 `-m`, 一般来说两个 `-m` 的信息已经足够了

        > go run -gcflags="-m -m" escape_ayalysis_demo.go
        # command-line-arguments
        ./escape_ayalysis_demo.go:11:6: cannot inline returnString1: marked go:noinline
        ./escape_ayalysis_demo.go:18:6: cannot inline returnStringPrt1: marked go:noinline
        ./escape_ayalysis_demo.go:4:6: cannot inline main: non-leaf function
        ./escape_ayalysis_demo.go:21:9: &rspt1 escapes to heap
        ./escape_ayalysis_demo.go:21:9:         from sp1 (return) at ./escape_ayalysis_demo.go:21:2
        ./escape_ayalysis_demo.go:19:2: moved to heap: rspt1
        ./escape_ayalysis_demo.go:7:33: main &s1 does not escape
        this is a variable in func will return
        this is a variable in func will return
        s1: this is a variable in func will return , addr:  0xc42005ff68
        s2: 0xc42000e020

这里可以看到函数 `returnString1` 中没有发生逃逸，因为 `returnString1` 的返回值类型是 `string`, `main` 函数通过值复制将 `rs1` 的值复制一份而得到 `s1`；
而函数 `returnStringPrt1` 的返回值是 `string` 类型的指针，在 `main` 函数中调用了 `returnStringPrt1` 时，可以访问指针 `sp1` 指向的变量，此时指针 `sp1` 和指针 `s2` 指向同一个变量，因此变量 `rspt1` 分配在堆空间中。


## 内存分配

Memory allocator.
内存分配

This was originally based on tcmalloc, but has diverged quite a bit.
`golang` 的内存分配的方法是在 `TCMalloc` 的基础上进行了一些改动，`TCMalloc(Thread-caching Malloc)` 介绍请看
http://goog-perftools.sourceforge.net/doc/tcmalloc.html

### `TCMalloc(Thread-caching Malloc)`
`TCMalloc` 分配的内存主要是两个地方：`线程私有缓存(Thread Local Cache)` 和 `全局缓存堆（Central Heap）`；
`TCMalloc` 为每个线程分配一个私有缓存，把 <=32k 的对象视为 `小对象(Small Object)`，小对象的的内存分配先尝试从线程私有缓存上分配，如果线程私有缓存空间不足，再从全局缓存堆中申请一部分空间作为线程私有缓存，而周期性的垃圾回收会将线程私有缓存的空间归还给全局缓存堆；而大于 32k 的对象视为 `大对象(Large Object)`，大对象的分配直接在全局缓存堆上；

### 小对象内存分配（Small Object Allocation）

线程本地缓存是一个单向链表 L1，链表 L1 的每个元素也是一个具有相同大小的空间对象的链表 l<sub>n</sub>

![](/assets/picture/theadheap.gif "Thread Local Cache Heap")

`TCMalloc` 将线程本地缓存拆分成约170种大小类型的空闲对象列表，依次以 8bytes、16bytes、32bytes等大小递增

小对象内存分配具体步骤如下：
1. 计算对象大小，找到大于对象大小的最小的大小类型 m；
2. 在当前线程中找到该大小类型的空闲对象链表 l<sub>m</sub>；
3. 如果链表 l<sub>m</sub> 不为空，从链表中删除第一个空闲对象用于内存分配；这个操作过程中，`TCMalloc` 不需要加锁，这个对提高内存分配的效率很有帮助；
4. 如果链表 l<sub>m</sub> 为空，从所有线程共享的 `Central Free List` 中获取一堆空闲对象，放到当前线程的私有缓存队列上去，在重复步骤3，从线程私有缓存中返回一个空闲对象


The main allocator works in runs of pages.
Small allocation sizes (up to and including 32 kB) are
rounded to one of about 70 size classes, each of which
has its own free set of objects of exactly that size.
Any free page of memory can be split into a set of objects
of one size class, which are then managed using a free bitmap.

主要的内存分配器是工作在运行中的 ***?内存页?*** 上。
小对象分配的

golang 将内存切分成约70种固定大小的内存块，me

The allocator's data structures are:

fixalloc: a free-list allocator for fixed-size off-heap objects,
	used to manage storage used by the allocator.
mheap: the malloc heap, managed at page (8192-byte) granularity.
mspan: a run of pages managed by the mheap.
mcentral: collects all spans of a given size class.
mcache: a per-P cache of mspans with free space.
mstats: allocation statistics.

Allocating a small object proceeds up a hierarchy of caches:

1. Round the size up to one of the small size classes
   and look in the corresponding mspan in this P's mcache.
   Scan the mspan's free bitmap to find a free slot.
   If there is a free slot, allocate it.
   This can all be done without acquiring a lock.

2. If the mspan has no free slots, obtain a new mspan
   from the mcentral's list of mspans of the required size
   class that have free space.
   Obtaining a whole span amortizes the cost of locking
   the mcentral.

3. If the mcentral's mspan list is empty, obtain a run
   of pages from the mheap to use for the mspan.

4. If the mheap is empty or has no page runs large enough,
   allocate a new group of pages (at least 1MB) from the
   operating system. Allocating a large run of pages
   amortizes the cost of talking to the operating system.

Sweeping an mspan and freeing objects on it proceeds up a similar
hierarchy:

1. If the mspan is being swept in response to allocation, it
   is returned to the mcache to satisfy the allocation.

2. Otherwise, if the mspan still has allocated objects in it,
   it is placed on the mcentral free list for the mspan's size
   class.

3. Otherwise, if all objects in the mspan are free, the mspan
   is now "idle", so it is returned to the mheap and no longer
   has a size class.
   This may coalesce it with adjacent idle mspans.

4. If an mspan remains idle for long enough, return its pages
   to the operating system.

Allocating and freeing a large object uses the mheap
directly, bypassing the mcache and mcentral.

Free object slots in an mspan are zeroed only if mspan.needzero is
false. If needzero is true, objects are zeroed as they are
allocated. There are various benefits to delaying zeroing this way:

1. Stack frame allocation can avoid zeroing altogether.

2. It exhibits better temporal locality, since the program is
   probably about to write to the memory.

3. We don't zero pages that never get reused.



P: 运行是管理 G 并把他们映射到 Logical Processor，称之为P；P可以看作是一个抽象的资源或者一个上下文，它需要获取以便操作系统线程(称之为M)可以运行G。
M: 操作系统线程
G: goroutine，golang 中比线程还要轻量级的协程

`fixalloc`: 固定大小的

```go
type fixalloc struct {
	size   uintptr  // 固定
	first  func(arg, p unsafe.Pointer) // called first time p is returned
	arg    unsafe.Pointer
	list   *mlink
	chunk  uintptr // use uintptr instead of unsafe.Pointer to avoid write barriers
	nchunk uint32
	inuse  uintptr // in-use bytes now
	stat   *uint64
	zero   bool // zero allocations
}
```
