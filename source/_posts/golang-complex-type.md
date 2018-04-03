---
title: golang：内建复合类型
mathjax: true
date: 2018-03-09 10:42:27
tags:
    - golang
categories: "golang"
---

Go 源码中有两处定义了该结构，分别是：\$GOROOT/src/reflect/type.go 中的 [_type](https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/type.go?utf8=✓#L28) 以及 \$GOROOT/src/runtime/type.go 中的 [rtype](https://github.com/golang/go/blob/release-branch.go1.10/src/reflect/type.go?utf8=✓#L297)，下面将使用 \$GOROOT/src/reflect/type.go 中的定义来解释各个字段的含义。

```go
type tflag uint8        // tflag本质上就是一个uint8类型
type nameOff int32      // 类型名偏移量类型
type typeOff int32      // 类型偏移量类型

// 默认实现类型的两个操作,hash和equal
type typeAlg struct {
    // 第一个参数是该类型对象的指针，后面是产生hash的seed
    hash func(unsafe.Pointer, uintptr) uintptr
    // 两个参数分别表示该类型对象的指针
    equal func(unsafe.Pointer, unsafe.Pointer) bool
}
```

有了上面的类型定义，就能得到所有复合数据类型的基本定义，定义如下：

```go
type _type struct {
    size       uintptr   // 类型大小
    ptrdata    uintptr   // 数据大小(字节数目)
    hash       uint32    // 该类型的hash值
    tflag      tflag     // 类型额外信息标记
    align      uint8     // 数据对齐基准值
    fieldalign uint8     // 该类型在struct中数据对齐的基准值
    kind       uint8     // 类型的枚举值
    alg        *typeAlg  // 该类型上的行为(操作),默认行为:hash和equal
    gcdata    *byte      // gc数据
    str       nameOff    // 索引类型名字的偏移量
    ptrToThis typeOff    // 索引类型的偏移量
}
```
<!-- more -->
1、接口类型底层数据结构

```go
// 接口中定义的方法信息
type imethod struct {
    name nameOff       // 方法名偏移量
    ityp typeOff       // 方法类型偏移量
}
// 接口类型的定义
type interfacetype struct {
    typ     _type      // 接口本身的类型
    pkgpath name       // 引入该类型的包路径
    mhdr    []imethod  // 接口中定义的方法的集合
}
```

2、数组类型底层数据结构

```go
// array 数组类型定义信息
type arraytype struct {
	typ   _type       // 数组本身的类型
	elem  *_type      // 数组存储的数据类型
    slice *_type      // slice切片类型( slice 可以指向 array s)
	len   uintptr     // 数组长度
}
```

3、slice 切片类型底层数据结构

```go
// slice 切片类型定义信息
type slicetype struct {
	typ  _type
	elem *_type
}

// slice 完整的类型定义
// 定义在 $GOROOT/src/runtime/slice.go
type slice struct {
    // slice 数据域,其中包含了slicetype信息,参看$GOROOT/src/runtime/malloc.go 中 mallocgc 函数
    array unsafe.Pointer
    len   int             // slice 中实际元素的个数
    cap   int             // slice 在不动态调整的情况下最多存储的元素个数
}
```

4、chan 管道类型底层数据结构

```go
// chan 类型定义信息
type chantype struct {
    typ  _type      // chan 管道本身的信息
    elem *_type     // 管道中元素的类型
    dir  uintptr    // 管道方向
}

// chan 完整的类型定义
// 定义在 $GOROOT/src/runtime/chan.go
type hchan struct {
    qcount   uint           // 队列中所有数据的数目
    dataqsiz uint           // buf 中数据的大小
    buf      unsafe.Pointer // 当前 chan 缓存大小, make(chan int, 8)：8个int空间
    elemsize uint16     // 元素大小,例如 chan int：4；chan int64：8
    closed   uint32     // 标识该管道是否已经关闭的标志位
    elemtype *_type     // 管道中通过的元素类型,如 chan int 类型,此处的类型就是 int 对应的类型
    sendx    uint       // 发送数据队列索引 
    recvx    uint       // 接收数据队列索引
    recvq    waitq      // 接收数据队列, waitq 就是一个队列,里面包含了协程的挂起和唤醒操作
    sendq    waitq      // 发送数据队列
    lock mutex          // 保护 hchan 里面的数据,防止协程间竞争 hchan 结构中的数据
}
```

5、指针类型

```go
// 指针类型定义信息
type ptrtype struct {
	typ  _type      // 指针本身的类型
	elem *_type     // 指向元素类型
}
```

6、函数类型

```go
// 函数类型定义信息
type functype struct {
	typ      _type      // 函数本身的类型信息
	inCount  uint16     // 函数参数个数
	outCount uint16     // 函数返回值个数
}
```

7、struct 结构体类型

```go
// 结构体中每个域类型定义
type structfield struct {
	name       name        // 变量名
	typ        *_type      // 变量类型
    // 该域在结构体中的偏移量(fieldIndex << 1 | isAnonymous),并且标记是否是匿名域
    offsetAnon uintptr
}

// 结构信息的完整定义
type structtype struct {
	typ     _type         // 结构体本身的类型信息
	pkgPath name          // 结构体所在的包名
	fields  []structfield // 结构体中各域信息,按照 offset 排序
}
```

