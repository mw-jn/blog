---
title: golang：map源码分析
mathjax: true
date: 2018-04-03 12:18:11
tags:
    - golang
    - map
    - 源码分析
categories: "golang"
---
​	GoLang 中的 map 仅仅就是一个哈希表（Hash Table）。数据存放在桶（Bucket）数组中，每个桶最多包含 8 个键值对。Hash 值低字节用于选择桶，每个桶都会包含 Hash 值的若干个 高字节用于区分桶中的每个数据实体。如果一个桶中多于 8 个键，那么就会增加桶，并且连接起来。
	当哈希表要扩展容量时，将以两倍的大小分配新的桶数组大小的空间，并将旧桶数组中的数据拷贝到新桶数组中。Map 迭代器遍历所有的通数组，并按照遍历的顺序返回键值。为了保证迭代语义，绝不在桶之间移动键值（如果移动了，键值可能返回 0 次或者 2 次）。当扩展哈希表示，迭代器仍然只迭代旧哈希表，并且为新表检查已经迭代过的数据是否已经被移动到新表中（本次分析的源码基于 go 1.10 版本）。

​	负载因子的选择：
		负载因子太大将会产生很多溢出桶，太小将会浪费很多空间。下面的数据展示了不同负载因子对应的状态数据（64-bit, 8 字节的键和值）。

| loadFactor | %overflow | bytes/entry | hitprobe | missprobe |
| :--------: | :-------: | :---------: | :------: | :-------: |
|    4.00    |   2.13    |    20.77    |   3.00   |   4.00    |
|    4.50    |   4.05    |    17.30    |   3.25   |   4.50    |
|    5.00    |   6.85    |    14.77    |   3.50   |   5.00    |
|    5.50    |   10.55   |    12.94    |   3.75   |   5.50    |
|    6.00    |   15.27   |    11.67    |   4.00   |   6.00    |
|    6.50    |   20.90   |    10.79    |   4.25   |   6.50    |
|    7.00    |   27.14   |    10.15    |   4.50   |   7.00    |
|    7.50    |   34.03   |    9.73     |   4.75   |   7.50    |
|    8.00    |   41.10   |    9.40     |   5.00   |   8.00    |

​	其中：
	%overflow = 产生溢出桶所占的百分比
	bytes/entry = 每个键值对所消耗的字节
	hitprobe = 查找存在键的的消耗
	missprobe = 查找不存在键的消耗

<!-- more -->
$GOROOT/src/runtime/hashmap.go 中定义的常量如下所示：

```go
const ( 
    // 一个桶可以存放键值对的最大数目
    bucketCntBits = 3
    bucketCnt     = 1 << bucketCntBits       // 当前设置的最大数目是 8（1 << 3）

    // 一个桶的最大平均负载可以触发增长到 6.5
    // 表达成整型表达式就为 13 / 2（loadFactorNum/loadFactDen）
    loadFactorNum = 13
    loadFactorDen = 2

    // Maximum key or value size to keep inline (instead of mallocing per element).
    // Must fit in a uint8.
    // Fast versions cannot handle big values - the cutoff size for
    // fast versions in ../../cmd/internal/gc/walk.go must be at most this value.
    maxKeySize   = 128
    maxValueSize = 128

    // 数据偏移量本应该是 bmap 结构体的大小，但是必须有有正确的内存对齐。
    // 对于 amd64p32 意味着即便指针占用 32 位，bmap也需要 64 位对齐（也就是 8 字节对齐）。
    // 假如 bamp 实际占用空间是 63 个字节，但是 64 位对齐，实际上分配的空间就是 64 字节
    // 如果这个数据结构后跟的是键值对数据，那么 v 刚好就是数据的偏移量
    dataOffset = unsafe.Offsetof(struct {
        b bmap
        v int64
    }{}.v)

    // Possible tophash values. We reserve a few possibilities for special marks.
    // Each bucket (including its overflow buckets, if any) will have either all or none of its
    // entries in the evacuated* states (except during the evacuate() method, which only happens
    // during map writes and thus no one else can observe the map during that time).
    empty          = 0 // cell is empty
    evacuatedEmpty = 1 // cell is empty, bucket is evacuated.
    evacuatedX     = 2 // key/value is valid.  Entry has been evacuated to first half of larger table.
    evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
    minTopHash     = 4 // minimum tophash for a normal filled cell.

    // flags
    iterator     = 1 // 可能是一个桶上的迭代器
    oldIterator  = 2 // 可能是一个在旧桶上的迭代器
    hashWriting  = 4 // 一个协程正在写 map 标记位
    sameSizeGrow = 8 // 当前 map 增长到跟新 map 一样大小的空间

    // sentinel bucket ID for iterator checks
    noCheck = 1<<(8*sys.PtrSize) - 1
)
```

下面是 map 数据结构的头部部分：

```go
// Go map 头结构
type hmap struct {
    // Note: the format of the Hmap is encoded in ../../cmd/internal/gc/reflect.go and
    // ../reflect/type.go. Don't change this structure without also changing that code!
    count     int // map 的实际大小（也就是 map 中元素的个数），内建函数 len() 返回的就是这个字段的值。
    flags     uint8  // 标记位
    B         uint8  // log_2(所有桶的数量)的值（可以持有最多 loadFactor * 2^B 个元素）
    noverflow uint16 // 一个合适的桶溢出数目大小，详细内容参见 incrnoverflow 函数
    hash0     uint32 // hash 种子值

    buckets    unsafe.Pointer // 2^B 桶数组，如果 count 为 0，该值也可能为 nil
    oldbuckets unsafe.Pointer // buckets 桶数组大小的一半，只有正在进行扩展 map 操作时, oldbuckets 才是非 nil
    nevacuate  uintptr   // 撤销计数(buckets 的大小小于该值的时候，表示所有元素都被转移了)

    extra *mapextra // 可选域
}
```

类型 mapextra 函数的解释：

```go
// mapextra 持有不在 map 中的域
type mapextra struct {
    // 如果键值都不包含指针，并且内联，该桶类型将被标记为不包含指针。这可以避免检查整个 map。
    // 然而，bamp.pointers 是一个指针。为了保证所有溢出 buckets 处于激活状态（否则会被 GC 回收），
    // 把指向所有溢出桶的指针值储存在 hmap.overflow 和 h.map.oldoverflow 中。
    // overflow 和 oldoverflow 仅在 key 和 vlaue 不包含指针的时候使用。
    // overflow 包含了 hmap.buckets 中溢出的桶
    // oldoverflow 包含了 hamp.oldoverflow 中的溢出桶
	// 这种间接性允许存储一个切片指针    
    overflow    *[]*bmap
    oldoverflow *[]*bmap

    // nextOverflow 持有一个指向自由溢出桶的指针
    nextOverflow *bmap
}
```

Go map 中的桶结构：

```go
// Go map 中的桶
type bmap struct {
    // 如果 tophash[0] < minTopHash，tophash[0]将表示桶的撤销状态
    // 一个桶里面只能包含 8 个键值对, tophash 储存的是相应的索引以及对应 key 的 tophash 值
    // 如果 tophash[0] < minTopHash，那么表示该桶里面的数据全部被转移到新桶里面了
    tophash [bucketCnt]uint8
    // 后面将跟着键集合（keys）
    // 后面紧接着跟着值集合(values)
    // 之所以将 key 和 value 分开存放，主要是节省内存空间
    // 后面还有一个桶溢出指针（用于解决桶碰撞问题）
}
```

几个简单的函数：

```go
// bucketShift 返回 1<<b
func bucketShift(b uint8) uintptr {
    if sys.GoarchAmd64|sys.GoarchAmd64p32|sys.Goarch386 != 0 {
        b &= sys.PtrSize*8 - 1 // help x86 archs remove shift overflow checks
    }
    return uintptr(1) << b
}

// bucketMask 返回 1<<b - 1
func bucketMask(b uint8) uintptr {
    return bucketShift(b) - 1
}

// tophash 为 hash 计算 tophash 值
func tophash(hash uintptr) uint8 {
    top := uint8(hash >> (sys.PtrSize*8 - 8))    // 取最高的一个字节数据
    if top < minTopHash {
        top += minTopHash
    }
    return top
}

// 判断该桶是否已经撤销
func evacuated(b *bmap) bool {
    h := b.tophash[0]
    return h > empty && h < minTopHash
}

// 获得溢出桶结构
func (b *bmap) overflow(t *maptype) *bmap {
    return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize))
}

// 添加溢出桶
func (b *bmap) setoverflow(t *maptype, ovf *bmap) {
    *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize)) = ovf
}

// 获得 keys 的首地址
func (b *bmap) keys() unsafe.Pointer {
    return add(unsafe.Pointer(b), dataOffset)
}
```

overflow 相关函数介绍：

```go
// incrnoverflow 增加 h.noverflow 的大小
// noverflow 统计溢出桶的数目
// 这个变量也被用于触发同样大小的 map 增长
// 为了保证 hmap 足够小，noverflow 是 uint16 类型
// 当拥有很少的 buckets 时，noverflow 是一个精确的值
// 当拥有大量的 buckets 时，noverflos 是一个近试值
func (h *hmap) incrnoverflow() {
    // We trigger same-size map growth if there are
    // as many overflow buckets as buckets.
    // We need to be able to count to 1<<h.B.
    if h.B < 16 {
        h.noverflow++
        return
    }
    // Increment with probability 1/(1<<(h.B-15)).
    // When we reach 1<<15 - 1, we will have approximately
    // as many overflow buckets as buckets.
    mask := uint32(1)<<(h.B-15) - 1
    // Example: if h.B == 18, then mask == 7,
    // and fastrand & 7 == 0 with probability 1/8.
    if fastrand()&mask == 0 {
        h.noverflow++
    }
}

// 创建一个溢出桶
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
    var ovf *bmap
    if h.extra != nil && h.extra.nextOverflow != nil {
        // 判定是否有可以使用的空闲桶
        ovf = h.extra.nextOverflow
        if ovf.overflow(t) == nil {
            // We're not at the end of the preallocated overflow buckets. Bump the pointer.
            h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
        } else {
            // This is the last preallocated overflow bucket.
            // Reset the overflow pointer on this bucket,
            // which was set to a non-nil sentinel value.
            ovf.setoverflow(t, nil)
            h.extra.nextOverflow = nil
        }
    } else {
        ovf = (*bmap)(newobject(t.bucket))  // 创建一个新的溢出桶
    }
    h.incrnoverflow()
    if t.bucket.kind&kindNoPointers != 0 {
        h.createOverflow()
        *h.extra.overflow = append(*h.extra.overflow, ovf)
    }
    b.setoverflow(t, ovf)    // 将 新创建的 ovf 挂载到溢出桶结构中
    return ovf
}

// 创建存储溢出桶的结构
func (h *hmap) createOverflow() {
    if h.extra == nil {
        h.extra = new(mapextra)
    }
    if h.extra.overflow == nil {
        h.extra.overflow = new([]*bmap)
    }
}
```

创建一个 map 可能要使用的函数：

```go
// 调用 m := make(map[uint]struct{}, uint64(10)) 编译器会翻译成调用该函数
func makemap64(t *maptype, hint int64, h *hmap) *hmap {
    if int64(int(hint)) != hint {
        hint = 0
    }
    return makemap(t, int(hint), h)
}

// makemap_small 实现了 Go map 的两种创建方式：
// make(map[k]v) 以及，
// 当 hint 在编译的时候就知道桶大小时，make(map[k]v, hint)
// map 内存需要在堆上分配
func makemap_small() *hmap {
    h := new(hmap)
    h.hash0 = fastrand()
    return h
}

// makemap 实现了 Go map 的创建函数 make(map[k]v, hint)。
// 如果编译器决定在栈上创建 map 或创建第一个桶，h 和(或)桶可能非 nil。
// 如果 h 非 nil，map 可以直接在 h 上被创建
// 如果 h.buckets 非 nil，桶指向的位置可以被当做第一个桶使用
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // 在 64 位平台上，hamp 的大小应该占用 48 个字节
    // 在 32 位平台上，hamp 的大小应该占用 28 个字节
    // 编译器算好的基本上都不会执行该逻辑，只不过例行检查而已
    if sz := unsafe.Sizeof(hmap{}); sz != 8+5*sys.PtrSize {
        println("runtime: sizeof(hmap) =", sz, ", t.hmap.size =", t.hmap.size)
        throw("bad hmap size")
    }

    // 如果桶数目的值小于 0，直接设置 0，或者，
    // 要声明的桶数目要大于系统中能声明的桶的数目也直接置 0
    if hint < 0 || hint > int(maxSlicegCap(t.bucket.size)) {
        hint = 0
    }

    // 如果为 nil，创建一个 hmap 结构
    if h == nil {
        h = (*hmap)(newobject(t.hmap))
    }
    h.hash0 = fastrand()    // 产生一个随机种子

    // 找到一个合适的大小能存的下请求所有元素的大小
    B := uint8(0)
 	// 根据负载因子产生一个合适的 B
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B

    // 分配一个初始化的哈希表
    // 如果 B 为 0，buckets 的内存分配才用懒惰延迟的方法(写 map 的时候分配)
    // 如果 hint 比较大，清理这块内存需要花一点时间
    if h.B != 0 {
        var nextOverflow *bmap
        h.buckets, nextOverflow = makeBucketArray(t, h.B)
        if nextOverflow != nil {
            h.extra = new(mapextra)
            h.extra.nextOverflow = nextOverflow
        }
    }

    return h
}

// makBuckextArray 产生桶数组，可能也分配一个溢出桶
func makeBucketArray(t *maptype, b uint8) (buckets unsafe.Pointer, nextOverflow *bmap) {
    base := bucketShift(b)
    nbuckets := base         // 产生桶的个数
    // 对一个很小的 b，溢出桶不可能产生
    // 避免计算开销
    if b >= 4 {
        // 如果 b >= 4，将计算溢出桶，用于保存数据
        nbuckets += bucketShift(b - 4)
        sz := t.bucket.size * nbuckets     // 应该分配的大小
        up := roundupsize(sz)              // mallocgc 实际分配的大小
        if up != sz {                      // 前后不一致，以实际分配的大小存储的大小为准 
            nbuckets = up / t.bucket.size  // 计算实际分配的内存能装的下多少桶
        }
    }
    buckets = newarray(t.bucket, int(nbuckets))    // 分配桶数组
    if base != nbuckets {
        // 预分配一些溢出桶
        // 为了最小代码跟踪这些预分配的空间，约定一个规则：如果预分配桶中的 overflow 指针是 nil，
        // 有更多的 bumping the pointer 可用。
        // 最后一个溢出桶需要一个安全的非 nil 指针, 可以使用 buckets。
        nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
        last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
        last.setoverflow(t, (*bmap)(buckets))
    }
    return buckets, nextOverflow
}
```

map 的访问方式一共有两种，主要是是看返回值是一个参数还是两个参数，代码如下：

```go
// mapaccess1 返回指向 h[key] 的指针。
// 绝不返回 nil，即使 map 中没有对应的 key 值，访问会返回该类型对应的 0 值对象的指针
// 注意：返回的指针存在 map 的整个生命周期找那个，所有不要持有太长时间
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // ...   此处表示原代码中有代码，对整个逻辑没有应该，删除这部分代码，看完整代码，请参考源文件
    // ...
    // 如果没有分配空间，或者当前没有元素，直接返回 0 值
    if h == nil || h.count == 0 {
        return unsafe.Pointer(&zeroVal[0])
    }
    
    // 判定当前是否有写操作，如果有，直接抛出异常
    // 不能在读的时候，并发写
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
    alg := t.key.alg       // 取出 key 类型上的运算，一般默认的都有产生 hash 值和 比较操作
    hash := alg.hash(key, uintptr(h.hash0))   // 根据 key 和 hash 种子为 key 产生 hash 值
    m := bucketMask(h.B) // 产生桶选择掩码，假如桶的数目为 16(h.B=4)，那么掩码为 0xFF，用于选择桶
    // 根据 hash 值和桶掩码选择一个桶 b
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    // 判断是否还有数据在旧桶中
    if c := h.oldbuckets; c != nil {
        // 判定旧桶数组和新桶数组时候是等同大小增长的？
        // 如果不是 则掩码右移 1 位，相当于新桶数组大小除以 2 后的掩码
        // 旧桶数组增长方式只有两种：同等大小增长或者 2 倍增长
        if !h.sameSizeGrow() {
            m >>= 1
        }
        // 选择该 key 在旧桶数组中的桶
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        
        // 判定该桶中的数据是否全部转移到新桶中，如果已经转移则不需要管
        // 没有转移的话，那么该桶中的 key 就是要找的数值
        if !evacuated(oldb) {
            b = oldb
        }
    }
    top := tophash(hash)    // 取出 hash 值 top 哈希值
    
    // 遍历桶以及相应的溢出桶
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            // 检测 tophash 值是否匹配
            if b.tophash[i] != top {
                continue
            }
            // 匹配的话直接取出桶里面的 key
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            // 判断该 key 是否是间接访问的，如果是间接访问，需要取地址转化一次
            if t.indirectkey {
                k = *((*unsafe.Pointer)(k))
            }
            // 判定存储的 k 是否与要查找的匹配
            if alg.equal(key, k) {
                v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                if t.indirectvalue {
                    v = *((*unsafe.Pointer)(v))
                }
                return v        // 返回对应的值
            }
        }
    }
    // 否者没有找到合适的值，返回类型默认的 0 值
    return unsafe.Pointer(&zeroVal[0])
}

// v, ok := map[k] 这种形式的调用会调用 mapaccess2 函数，流程和 mapaccess1 函数基本一致
// 唯一不同的地方就是，该反函数返回两个值
// 函数原型如下，具体逻辑参看源代码
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
    // ....
    return unsafe.Pointer(&zeroVal[0]), false
}
```

hash 迭代结构：

```go
// hash 迭代结构
type hiter struct {
    key         unsafe.Pointer // 键，必须放在第一个位置上，nil 表示迭代器结束 (see cmd/compile/internal/gc/range.go).
    value       unsafe.Pointer // 值，必须放在第二个位置上 (see cmd/internal/gc/range.go).
    t           *maptype       // map 类型信息，如 map[uint]uint 和 map[uint]struct{} 是不同的类型
    h           *hmap          // map 头部信息
    buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
    bptr        *bmap          // 当前的桶
    overflow    *[]*bmap       // 保证 hamp.buckets 中的溢出桶处以激活状态
    oldoverflow *[]*bmap       // 保证 hmap.oldbuckets 中的溢出桶处于激活状态
    startBucket uintptr        // 迭代桶的起始编号
    offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
    
    // 假设一共有 hash 一共有 16 个桶，下标依次是 0，1，...，15
    // 又假如产生的随机起始点为 5，迭代器将依次从 5 开始遍历，遍历到 15 的时候就会把 wrapped 设置为 true
    wrapped     bool    // 标记是否从起始位置遍历到桶的最大位置
    B           uint8   // 初始化的时候将 hmap.B 保存到该变量中，用于在遍历过程中是否有新的 map 扩展操作
    i           uint8   // 当前桶内键值对的偏移量
    bucket      uintptr // 当前遍历的桶索引
    checkBucket uintptr // 
}
```

range 遍历 map 使用到下面的函数：

```go
// map 迭代器使用的函数，返回 key 和 value。
// 逻辑同 access1 函数
// 不同之处在于：
// 1.若 map 没有初始化，或者 map 中没有键值对，直接返回 nil, nil
// 2.若没有查到 key 对应的值，直接返回 nil, nil
// 3.若查到对应的 key，则返回 k, v 键值对的值
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer) {
    // ...
    return nil, nil
}
```

```go
// mapiterinit 初始化用于遍历 map 的 hiter 结构。it 指向的 hiter 结构由编译器分配在栈上或者
// 由 reflect_mapiterinit 在堆上分配。
// 这两种方式都要清 0 hiter，因为这个结构包含了指针
func mapiterinit(t *maptype, h *hmap, it *hiter) {
    // ... 省略了代码，详细请查看源代码

    if h == nil || h.count == 0 {
        return
    }

    //  验证 hiter{} 占用的空间是否正确
    if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
        throw("hash_iter size incorrect") // see ../../cmd/internal/gc/reflect.go
    }
    it.t = t
    it.h = h

    // 保存桶状态快照
    it.B = h.B
    it.buckets = h.buckets
    if t.bucket.kind&kindNoPointers != 0 {
        // Allocate the current slice and remember pointers to both current and old.
        // This preserves all relevant overflow buckets alive even if
        // the table grows and/or overflow buckets are added to the table
        // while we are iterating.
        h.createOverflow()
        it.overflow = h.extra.overflow
        it.oldoverflow = h.extra.oldoverflow
    }

    // 随机一个桶起始点
    // 所以每次遍历 map 的顺序可能都不一样
    r := uintptr(fastrand())
    if h.B > 31-bucketCntBits {
        r += uintptr(fastrand()) << 31
    }
    it.startBucket = r & bucketMask(h.B)           // 起始桶索引
    it.offset = uint8(r >> h.B & (bucketCnt - 1))  // 桶类起始偏移量

    // iterator state
    it.bucket = it.startBucket          

    // 设置标记位表示当前有迭代器在 buckerts 和 oldbuckets 上进行遍历
    // 这个地方有可能在并发遍历，因此使用原子操作对标记位进行设置
    if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
        atomic.Or8(&h.flags, iterator|oldIterator)
    }

    mapiternext(it)
}
```

```go
func mapiternext(it *hiter) {
    h := it.h
    // ... 省略一些代码

    // 当前如果有写操作，直接抛出异常
    if h.flags&hashWriting != 0 {
        throw("concurrent map iteration and map write")
    }
    t := it.t
    bucket := it.bucket
    b := it.bptr
    i := it.i
    checkBucket := it.checkBucket
    alg := t.key.alg

next:
    // 没有新的桶了
    if b == nil {
        // 桶索引回到起始点，表示所有桶都执行完了
        // it.wrapped 为 true 表示已经遍历到了最大值了
        if bucket == it.startBucket && it.wrapped {
            // end of iteration
            it.key = nil
            it.value = nil
            return
        }
        // 当前正在扩展，并且 it.B 等于 h.B（说明 hiter 被初始化后，就没有被扩展过）
        if h.growing() && it.B == h.B {
            // 遍历旧桶（数据还没有被迁移完）
            
            // 获得旧桶索引号
            oldbucket := bucket & it.h.oldbucketmask()
            // 获得旧桶实际数据位置
            b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
            // 判定该桶中的数据是否已经被迁移出去了
            if !evacuated(b) {
                checkBucket = bucket
            } else {
                b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
                checkBucket = noCheck
            }
        } else {
            b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
            checkBucket = noCheck
        }
        // 增加当前遍历桶索引的计数
        bucket++
        if bucket == bucketShift(it.B) {
            bucket = 0         // 到达最大值，直接清零，相当于是一个循环的数组
            it.wrapped = true  // 设置已经到达过最大桶索引的标记位
        }
        i = 0     // 桶内键值对偏移量清 0
    }
    for ; i < bucketCnt; i++ {
        offi := (i + it.offset) & (bucketCnt - 1)    // 随机偏移位置开始遍历
        if b.tophash[offi] == empty || b.tophash[offi] == evacuatedEmpty {
            continue
        }
        k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
        if t.indirectkey {
            k = *((*unsafe.Pointer)(k))
        }
        v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.valuesize))
        
        // ... 省略各种条件检查，不符合条件直接 continue
        
        if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
            !(t.reflexivekey || alg.equal(k, k)) {
            // This is the golden data, we can return it.
            // OR
            // key!=key, so the entry can't be deleted or updated, so we can just return it.
            // That's lucky for us because when key!=key we can't look it up successfully.
            it.key = k
            if t.indirectvalue {
                v = *((*unsafe.Pointer)(v))
            }
            it.value = v
        } else {
            // The hash table has grown since the iterator was started.
            // The golden data for this key is now somewhere else.
            // Check the current hash table for the data.
            // This code handles the case where the key
            // has been deleted, updated, or deleted and reinserted.
            // NOTE: we need to regrab the key as it has potentially been
            // updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
            rk, rv := mapaccessK(t, h, k)
            if rk == nil {
                continue // key has been deleted
            }
            it.key = rk
            it.value = rv
        }
        it.bucket = bucket     // 保存当前迭代的桶索引，以便下一次从该位置开始
        if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
            it.bptr = b      // 保存当前正在读的实际桶（包含溢出桶）
        }
        it.i = i + 1           // 保存桶内偏移量，以便下一次从该位置开始
        it.checkBucket = checkBucket
        return
    }
     b = b.overflow(t)    // 遍历溢出桶
    i = 0                 // 桶类偏移量清 0
    goto next
}
```

写 map（即给 map 赋值）

```go
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    if h == nil {
        panic(plainError("assignment to entry in nil map"))
    }
    // ... 此处省略无关代码
    
    // 检查写标记位是否被设置，被设置直接抛出异常
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }
    alg := t.key.alg
    hash := alg.hash(key, uintptr(h.hash0))

    // 在调用 alg.hash 后设置 hashWriteing 标识位
    // 因为 alg.hash 可能会 panic，在这种情况下，将不能完成写操作
    h.flags |= hashWriting

    // makemap 中的懒惰延迟初始化就在这个地方初始话
    if h.buckets == nil {
        h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
    }

again:
    // 选择元素要插入的桶索引
    bucket := hash & bucketMask(h.B)
    // 判定当前 map 是否处于扩展中，如果正在扩展，就进行扩展操作
    if h.growing() {
        growWork(t, h, bucket)
    }
    // 取得桶结构
    b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    top := tophash(hash)
    var inserti *uint8
    var insertk unsafe.Pointer
    var val unsafe.Pointer
    for {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                if b.tophash[i] == empty && inserti == nil {
                    // 找到第一个可以插入数据的槽位
                    inserti = &b.tophash[i]         // 记录要要插入的 tophash 空间
                    insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize)) // 要插入的 key 地址
                    val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))  // 要插入的 value 地址
                }
                continue
            }
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey {
                k = *((*unsafe.Pointer)(k))
            }
            // 桶中的 key 被取出来用于和 k 进行比较，不一致则继续查找
            if !alg.equal(key, k) {
                continue
            }

            // 根据 map 类型确定 key 是否需要更新，如果 map[uint]uint，那么 t.needkeyupdate 就是false
            // key 值是不需要更新的，但是 map 中的 key 是 指针切片或者map类型的时候，
            // 那么 needkeyupdate 就需要更新
            if t.needkeyupdate {
                typedmemmove(t.key, k, key)
            }
            // 更新要操作的值元素地址
            val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
            goto done              // 跳转到 done 标签处
        }
        // 否则的话 检查溢出桶
        ovf := b.overflow(t)
        if ovf == nil {
            break
        }
        b = ovf
    }
    
    // 没有发现映射的 key
    
    // 先判定是否正在扩展，是否达到最大的负载因子，判定是否有太多的溢出桶
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)    // 扩展操作
        goto again        // 重试 
    }

    // 没有找到可以插入的空间
    if inserti == nil {
        // 所有的桶都满了，分配一个新的
        newb := h.newoverflow(t, b)
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        val = add(insertk, bucketCnt*uintptr(t.keysize))
    }

    // 在要插入点，保存 key/value
    if t.indirectkey {
        kmem := newobject(t.key)
        *(*unsafe.Pointer)(insertk) = kmem
        insertk = kmem
    }
    if t.indirectvalue {
        vmem := newobject(t.elem)
        *(*unsafe.Pointer)(val) = vmem
    }
    typedmemmove(t.key, insertk, key)
    *inserti = top
    // 增加计数
    h.count++
    done:
    
    // 结束的时候判定该值是否被修改过
    // 如果修改过则发生了并发写数据
    if h.flags&hashWriting == 0 {
        throw("concurrent map writes")
    }
    
    // 去除标记状态
    h.flags &^= hashWriting
    if t.indirectvalue {
        val = *((*unsafe.Pointer)(val))
    }
    return val
}
```

从 map 中删除 key 值

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
    
    // ... 省略没有用的代码
    
    // 当 map 为空或者没有键值对时直接返回
    if h == nil || h.count == 0 {
        return
    }
    
    // 判断写标记位是否重置过，如果当前正在写，直接抛出异常
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }

    alg := t.key.alg
    hash := alg.hash(key, uintptr(h.hash0))

    // 之所有在 alg.hash 过后设置该标记位原因同上面的函数
    h.flags |= hashWriting

    bucket := hash & bucketMask(h.B)
    if h.growing() {
        growWork(t, h, bucket)
    }
    b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
    top := tophash(hash)
    // 查找桶中的 key 值
search:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                continue
            }
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            k2 := k
            if t.indirectkey {
                k2 = *((*unsafe.Pointer)(k2))
            }
            if !alg.equal(key, k2) {
                continue
            }

            // 如果是指针类型，直接清除 k 为 nil
            if t.indirectkey {
                *(*unsafe.Pointer)(k) = nil
            } else if t.key.kind&kindNoPointers == 0 {
                // 清除 k 值
                memclrHasPointers(k, t.key.size)
            }

            // 同上，清除值
            if t.indirectvalue || t.elem.kind&kindNoPointers == 0 {
                v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                if t.indirectvalue {
                    *(*unsafe.Pointer)(v) = nil
                } else {
                    memclrHasPointers(v, t.elem.size)
                }
            }
            // 对应的 tophash 值清 0
            b.tophash[i] = empty
            // 长度减少
            h.count--
            break search
        }
    }

    // 判定是否有并发写，抛出异常
    if h.flags&hashWriting == 0 {
        throw("concurrent map writes")
    }
    // 清除写标记位
    h.flags &^= hashWriting
}
```

从上面 map range 和 delete 的过程可以看出，在 range 中 delete map 中的 key 并不会影响迭代器，原因是删除的过程只是清除了对应桶中 key/value 的值，对应的存储空间并没有删除，而 range 的时候就是根据对应储存空间的索引来查找的，所以没有任何影响。

map 源码中还有其它的函数，在此不一一说明，感兴趣的可以查看  go 源文件。
