---
title: golang：内建基础类型
tags:
  - golang
  - type
categories: golang
date: 2018-03-05 20:56:28
mathjax: true﻿
---

Go 语言的类型主要有两种:内建类型和用户自定义类型。内建类型又分为基础类型和复合类型。这篇文章主要介绍内建类型中的基础类型，主要参考源码 $GOROOT/src/builtin/builtin.go 来介绍。本文使用的 Go 版本是 go1.10 linux/amd64，Go 版本信息可以使用下面的命令来查看：

```shell
#go version
```

Go 内建基础类型：

|                             类型                             |              描述              | 字节大小(Byte) |                           取值范围                           |
| :----------------------------------------------------------: | :----------------------------: | :------------: | :----------------------------------------------------------: |
| [bool](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L14) |            布尔类型            |       1        |                        [true，false]                         |
| [uint8](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L24) |         无符号8位整型          |       1        |                   [0，255] 或 [0，$2^8$-1]                   |
| [uint16](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L28) |         无符号16位整型         |       2        |                [0，65535] 或 [0，$2^{16}$-1]                 |
| [uint32](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L32) |         无符号32位整型         |       4        |              [0，4294967295] 或 [0，$2^{32}$-1]              |
| [int8](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L40) |         有符号8位整型          |       1        |                [-128，127]或[-$2^7$，$2^7$-1]                |
| [int16](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L44) |         有符号16位整型         |       2        |           [-32788,32767]或[-$2^{15}$，$2^{15}$-1]            |
| [int32](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L48) |         有符号32位整型         |       4        |      [-2147483648，2147483647]或[-$2^{31}$，$2^{31}$-1]      |
| [int64](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L52) |         有符号64位整型         |       8        | [-9223372036854775808，9223372036854775807]或[-$2^{63}$，$2^{63}$-1] |
| [float32](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L54) |          32位浮点类型          |       4        |   [IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) 32位    |
| [float64](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L62) |          64位浮点类型          |       8        |   [IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) 64位    |
| [complex64](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L62) | 复数，实部和虚部都是32为浮点型 |       8        |                              -                               |
| [complex128](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L66) | 复数，实部和虚部都是64为浮点型 |       16       |                              -                               |
| [string](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L71) |           字符串类型           |       -        |                              -                               |
| [int](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L75) |   有符号整型，占用空间随系统   |      4或8      |                       参见int32或int64                       |
| [uint](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L79) |   无符号整型，占用空间随系统   |      4或8      |                      参见uint32或unit64                      |
| [uintptr](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L83) |  指针数值类型，占用空间随系统  |      4或8      |                      参见uint32或unit64                      |
| [byte](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L88) |            字节类型            |       1        |                          参见uint8                           |
| [rune](https://github.com/golang/go/blob/release-branch.go1.10/src/builtin/builtin.go?utf8=✓#L92) |         UTF-8字符类型          |       4        |                          参见int32                           |

其中源代码中对 byte 和 rune 的定义如下：

```go
type byte = uint8
type rune = int32
```

也就是说 byte 就是一个 uint8 类型，rune 就是一个 int32 类型。以上就是 GoLang 中的所有类型。

另外我们还需要特别注意一个小的细节，GoLang 中的数字（1,2,3，...）以及 iota 是无类型的整型，但是在 Linux 平台下默认类型是 int。但是在接口中传递的话会自动转变成 int 类型，所以写代码的时候要特别注意。

```go
const a = iota
func main() {
    fmt.Printf("%T\n", a)    // 输出 int
    f(0)                     // 输出 int   这个地方很容易出错，直接传 0，然后转为 uint 就会失败
    f(a)                     // 输出 int
    f(uint(0))               // 输出 uint 0
}

func f(i interface{}) {
    fmt.Printf("%T\n", i)
    if p, ok := i.(uint); ok {
        fmt.Println(p)
        //...
    }
}
```

