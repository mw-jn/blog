---
mathjax: true
date: 2018-03-13 11:56:02
title: "golang: slice 切片类型源码解析"
tags:
    - golang
    - type
categories: "golang"
---

```go
// golang 中切片 slice 的实现
package runtime

import (
	"unsafe"
)

// slice 结构实现
type slice struct {
	array unsafe.Pointer     // slice 切片指针，连续空间
	len   int                // 当前元素个数
    cap   int                // 最大存储元素个数(在不重新分配空间的情况下)
}

// An notInHeapSlice is a slice backed by go:notinheap memory.
type notInHeapSlice struct {
	array *notInHeap
	len   int
	cap   int
}

// 可以分配最大元素个数的快速查询表
// 表索引就是元素所占用字节大小
// 如 var a []uint64 
// uint64 的大小是 8 个字节
// 那么最大可以分配的内存就是 maxElems[8] = _MaxMem / 8 个元素
// _MaxMem 在非windows 64 位平台下大小是 1 << 39
var maxElems = [...]uintptr{
	^uintptr(0),
	_MaxMem / 1, _MaxMem / 2, _MaxMem / 3, _MaxMem / 4,
	_MaxMem / 5, _MaxMem / 6, _MaxMem / 7, _MaxMem / 8,
	_MaxMem / 9, _MaxMem / 10, _MaxMem / 11, _MaxMem / 12,
	_MaxMem / 13, _MaxMem / 14, _MaxMem / 15, _MaxMem / 16,
	_MaxMem / 17, _MaxMem / 18, _MaxMem / 19, _MaxMem / 20,
	_MaxMem / 21, _MaxMem / 22, _MaxMem / 23, _MaxMem / 24,
	_MaxMem / 25, _MaxMem / 26, _MaxMem / 27, _MaxMem / 28,
	_MaxMem / 29, _MaxMem / 30, _MaxMem / 31, _MaxMem / 32,
}
```
<!-- more -->

```go
// 可以分配的最大元素数目
func maxSliceCap(elemsize uintptr) uintptr {
	if elemsize < uintptr(len(maxElems)) {    // 元素大小小于 32 个字节，直接使用查找表来查找
		return maxElems[elemsize]             // 读取内存的速度比除法运算要快很多，提高效率
	}
	return _MaxMem / elemsize           // 其它大小的元素动态计算
}

// slc := make([]uint, 10, 100) 将会调用
// slc := makeslce(uint, 10, 100)
func makeslice(et *_type, len, cap int) slice {
	// 获取可以存储的最大元素数目
	maxElements := maxSliceCap(et.size)
    // 对传入的参数进行检查
	if len < 0 || uintptr(len) > maxElements {
		panic(errorString("makeslice: len out of range"))
	}

	if cap < len || uintptr(cap) > maxElements {
		panic(errorString("makeslice: cap out of range"))
	}

    // 分配内存
	p := mallocgc(et.size*uintptr(cap), et, true)
	return slice{p, len, cap}
}

func makeslice64(et *_type, len64, cap64 int64) slice {
	len := int(len64)
	if int64(len) != len64 {
		panic(errorString("makeslice: len out of range"))
	}

	cap := int(cap64)
	if int64(cap) != cap64 {
		panic(errorString("makeslice: cap out of range"))
	}

	return makeslice(et, len, cap)
}

// growslice handles slice growth during append.
// It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
// The new slice's length is set to the old slice's length,
// NOT to the new requested capacity.
// This is for codegen convenience. The old slice's length is used immediately
// to calculate where to write new values during an append.
// TODO: When the old backend is gone, reconsider this decision.
// The SSA backend might prefer the new length or to return only ptr/cap and save stack space.
// 保留上面的解释
// 使用 append 操作增长 slice
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if et.size == 0 {
		if cap < old.cap {
			panic(errorString("growslice: cap out of range"))
		}
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
    // 要分配的空间 cap 比原始空间两倍都多，那就分配要分配的空间
	if cap > doublecap {
		newcap = cap
	} else {
        // 原始 slice 实际大小小于 1024 ，分配原始 cap 两倍大小
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// 原来 cap 的 1/4 倍进行递增
            // 跟 C++ 分配方式类型，如果一直是两倍递增，后面会非常浪费内存，到后面放缓内存增长速度
            // 这种分配方式有可能会发生溢出
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
            
            // 溢出检查，如果分配溢出，直接就是要分配的大小
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
    // 计算各个连续空间内存大小
	var lenmem, newlenmem, capmem uintptr
    // 计算指针的大小
	const ptrSize = unsafe.Sizeof((*byte)(nil))
	switch et.size {   // 类型大小
	case 1:   // 1个字节
		lenmem = uintptr(old.len)                // 原始 slice 中 len 大小元素占用内存大小
		newlenmem = uintptr(cap)                 // 要分配的空间 cap 占用内存大小
		capmem = roundupsize(uintptr(newcap))    // newcap 字节对齐后占用内存大小
		overflow = uintptr(newcap) > _MaxMem     // 判定是否溢出
		newcap = int(capmem)                     // 新 cap 大小，字节对齐后的大小
	case ptrSize:    // 指针类型大小
		lenmem = uintptr(old.len) * ptrSize               // 和上面一样,不过指针的大小不是1而已
		newlenmem = uintptr(cap) * ptrSize
		capmem = roundupsize(uintptr(newcap) * ptrSize)
		overflow = uintptr(newcap) > _MaxMem/ptrSize
		newcap = int(capmem / ptrSize)
	default:         // 默认值
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem = roundupsize(uintptr(newcap) * et.size)
		overflow = uintptr(newcap) > maxSliceCap(et.size)
		newcap = int(capmem / et.size)
	}

    // 检查数目是否溢出(overflow (uintptr(newcap) > maxSliceCap(et.size)))
    // 也要检查对齐后的内存大小是否会溢出(capmem > _MaxMem)
    // 下面的程序在 32 位机器上会触发段错误
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
    
	if cap < old.cap || overflow || capmem > _MaxMem {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.kind&kindNoPointers != 0 {
        // 非指针类型，分配连续的空间
		p = mallocgc(capmem, nil, false)
		memmove(p, old.array, lenmem)

        // 增新增长部分的内存空间置 0
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
        // 分配新的 slice 空间
		p = mallocgc(capmem, et, true)
        // 从老的 slice 拷贝元素到新分配的 slice 中
		if !writeBarrier.enabled {
			memmove(p, old.array, lenmem)
		} else {
			for i := uintptr(0); i < lenmem; i += et.size {
				typedmemmove(et, add(p, i), add(old.array, i))
			}
		}
	}

	return slice{p, old.len, newcap}
}

// 切片 slice 拷贝
// a := make([]uint, 0)
// b := make([]uint, 0)
// copy(a, b)
// width 是元素类型的大小
func slicecopy(to, fm slice, width uintptr) int {
	if fm.len == 0 || to.len == 0 {
		return 0
	}

	n := fm.len
	if to.len < n {
		n = to.len
	}

	if width == 0 {
		return n
	}

	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicecopy)
		racewriterangepc(to.array, uintptr(n*int(width)), callerpc, pc)
		racereadrangepc(fm.array, uintptr(n*int(width)), callerpc, pc)
	}
	if msanenabled {
		msanwrite(to.array, uintptr(n*int(width)))
		msanread(fm.array, uintptr(n*int(width)))
	}

    // 拷贝内存，对size大小为1作了特殊处理，直接复制一次
	size := uintptr(n) * width
	if size == 1 { // common case worth about 2x to do here
		// TODO: is this still worth it with new memmove impl?
		*(*byte)(to.array) = *(*byte)(fm.array) // known to be a byte pointer
	} else {
		memmove(to.array, fm.array, size)
	}
	return n
}

// go 1.10 新支持的 copy 方法,编译器会根据类型自动确定调用哪个函数
// a := make([]byte, 0)
// b := "12345"
// copy(a, b)
func slicestringcopy(to []byte, fm string) int {
	if len(fm) == 0 || len(to) == 0 {
		return 0
	}

    // 取两者中最小的长度
	n := len(fm)
	if len(to) < n {
		n = len(to)
	}

	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicestringcopy)
		racewriterangepc(unsafe.Pointer(&to[0]), uintptr(n), callerpc, pc)
	}
	if msanenabled {
		msanwrite(unsafe.Pointer(&to[0]), uintptr(n))
	}

    // 内存复制
	memmove(unsafe.Pointer(&to[0]), stringStructOf(&fm).str, uintptr(n))
	return n
}
```

