---
title: golang：channel源码分析
mathjax: true
date: 2018-03-11 21:06:53
tags:
    - golang
    - channel
categories: "golang"
---

Go 中的 chan 管道类型定义在 \$GOROOT/src/runtime/chan.go 中，完整的源码也可以参看[chan.go](https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/chan.go)，下面的代码踢除了-race、-msan标记执行的代码块。-race 用于检测是否存在竞争，-msan是内存布局是否跟C/C++一致。

```go
package runtime
// 本文件包含了 Go channels 的实现

// 不变关系:
// c.sendq 和 c.recvq 两个队列至少有一个为空
// select 中使用 send 和 receive 的无缓冲 channel 单协程将会阻塞
// c.sendq 和 c.recvq 的长度限制于 select 语句的的大小
//
// 对于有缓冲的 channels 有如下的判定：
// c.qcount > 0 意味着 c.recvq 为空。
// c.qcount < c.dataqsiz 意味着 c.sendq 为空

import (
	"runtime/internal/atomic"
	"unsafe"
)

const (
    maxAlign  = 8          // 最大对齐值(8字节对齐)
    // 8 字节对齐的最小字节数，如 unsafe.Sizeof(hchan{})= 1，那么在分配内存的时候会分配 8 字节的空间；
    // 又如 unsafe.Sizeof(hchan{})=513，那么内存分配器会分配 520 个字节，最后分配的一定是 8 的倍数
	hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
	debugChan = false       // 关闭 debug 标识
)

type hchan struct {
	qcount   uint           // 所有队列中数据的大小
	dataqsiz uint           // 循环 buf 的大小
    buf      unsafe.Pointer // 数据元素缓冲区(大小为dataqsiz)
    // 数据大小，如 chan uint64，那么 elemsize 的大小为 8
	elemsize uint16         // 数据大小
	closed   uint32         // chan 关闭标识, closed=1 表示关闭
    // 如 chan uint64，那么指向 uint64 位类型地址空间(相同类型的地址空间是共享的)
	elemtype *_type         // 数据元素类型
	sendx    uint   // 发送队列索引
	recvx    uint   // 接收数据索引
	recvq    waitq  // 等待接收数据协程队列
	sendq    waitq  // 等待发送数据协程队列

    // 持有该锁的协程不能改变另外协程的状态(如：唤醒另外的协程),否则退栈的时候会死锁
	lock mutex        // 保护 hchan 中的所有数据域
}

type waitq struct {
	first *sudog     // 等待队列的首指针
	last  *sudog     // 等待队列的尾指针
}
```
<!-- more -->
```go
//go:linkname 标识告诉编译器代码中 reflect.makechan 函数将会使用 reflect_makechan 函数来替代
//go:linkname reflect_makechan reflect.makechan
func reflect_makechan(t *chantype, size int) *hchan {
	return makechan(t, size)
}

func makechan64(t *chantype, size int64) *hchan {
	if int64(int(size)) != size {
		panic(plainError("makechan: size out of range"))
	}

	return makechan(t, int(size))
}

// 代码中调用 ch := make(chan uint, 100) 等价于
// var ch *hchan
// ch = makechan(uint, 100)    // 这里的 "uint" 带表类型
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

    // 编译器检查下面的三个条件，但是一般都不会有异常，可以认为是安全的
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	if size < 0 || uintptr(size) > maxSliceCap(elem.size) || uintptr(size)*elem.size > _MaxMem-hchanSize {
		panic(plainError("makechan: size out of range"))
	}
    // 上面的检查不是核心逻辑，所以可以忽略不管，看了也没有用，都是编译器做好的

    // 当缓冲区中的元素不包含指针时，hchan 不含有 GC 感兴趣的指针
	// 缓冲区指向同一片内存，元素类型是数据在不变的内存空间中
	// SudoG 被自己的线程引用，因此不能被回收。
	var c *hchan
	switch {
	case size == 0 || elem.size == 0:
        // 队列大小为 0 或者元素大小为 0
        // 其中 struct{} 类型的元素大小就为 0
        // 仅仅分配 hchanSize 大小的空间就足够了
		c = (*hchan)(mallocgc(hchanSize, nil, true))
        // Race 监测器使用此项来定位同步
		c.buf = unsafe.Pointer(c)
	case elem.kind&kindNoPointers != 0:    // 判定元素类型是否是指针
        // 非指针元素使用一个调用来分配 hchan 和缓冲区的大小
        // hchan 和缓冲区的地址空间是连续的
		c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 指针元素分开分配空间，hchan 的空间和缓冲区的空间可能是非连续的地址空间
		c = new(hchan)
		c.buf = mallocgc(uintptr(size)*elem.size, elem, true)
	}

	c.elemsize = uint16(elem.size)      // 单个元素所占用空间大小
	c.elemtype = elem                   // 存储元素类型
	c.dataqsiz = uint(size)             // 数据大小就是要分配的缓冲区的大小

    // 调试 chan 的 debug 标识位，默认是关闭状态
	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
	}
	return c
}

// 获取 hchan 结构中缓冲区中第 i 个槽的地址
func chanbuf(c *hchan, i uint) unsafe.Pointer {
	return add(c.buf, uintptr(i)*uintptr(c.elemsize))
}

// 编译器读到代码中 c <- x 替换为函数 chansend1(c, x)
//go:nosplit    编译器指令
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

/*
 * 当管道操作没有完成，且 block 不为空时，协定这种行为不会睡眠而是直接返回
 * 正在睡眠的管道已经被关闭时，可以使用 g.param == nil 来唤醒
 */
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
        // 如果 c == nil 且设置了非阻塞状态，发送操作会立即返回
		if !block {
			return false
		}
        // 如果 c == nil 且设置了阻塞状态，将会直接报错，抛出异常
        // c == nil 是永远不会发送的了数据还阻塞，如果不抛出异常，协程将会处于永远等待状态
        // 造成死锁，浪费系统资源
		gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

    // 略过，编译的时候加上"-race"选项才会生效下面的代码
	if raceenabled {
		racereadpc(unsafe.Pointer(c), callerpc, funcPC(chansend))
	}

    /*
     * 不需要获取锁快速地检查非阻塞管道发送操作失败
     * 在确认管道没有被关闭后，会检查是否处于准备发送状态，每步检查就是一个单字大小的读操作
     * 关闭的管道不能从"准备发送"状态切换到"非准备发送状态"，即使管道在检测中间被关闭，也可
     * 以理解为当管道都没有关闭且没有准备发送的某一时刻，我们认为在这个时刻发送操作不能被处理
     *
     * 下面的情况读操作将会被重新安排：管道没有准备发送并且没有关闭意味着在第一步检查的时候管道
     * 没有被关闭
     */
    // 检查条件：
    // 1.非阻塞，且
    // 2.未关闭，且
    // 3.缓冲区为空且没有等待的接收者 或者 缓冲区不为空且缓冲区已满
    // 直接返回失败，不能进行发送操作
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()          // 获取 cpu 时钟    
	}
    
    // 拿到锁，进入到临界区
	lock(&c.lock)
  	// 管道关闭，释放持有的锁，panic 错误
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

  	// 找到一个等待接收者，将通过管道缓冲将要发送的数据直接丢给接收者
    // 一般都是无缓冲区管道会执行下面的逻辑
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	
  	// 缓冲区未满，将数据放入发送缓冲区
	if c.qcount < c.dataqsiz {
		qp := chanbuf(c, c.sendx)         // 从缓冲区中取出一个元素空间用于存放将要发送的数据
		if raceenabled {                  // 忽略 -race 监测
			raceacquire(qp)
			racerelease(qp)
		}
		typedmemmove(c.elemtype, qp, ep)  // 复制数据到缓冲中
		c.sendx++                         // 发送索引自增
		if c.sendx == c.dataqsiz {        // 达到缓冲区最大值时，重置索引，形成循环缓冲区
			c.sendx = 0
		}
		c.qcount++                        // 缓冲中数据大小
		unlock(&c.lock)                   // 释放锁
		return true
	}
	
  	// 缓冲区已满，非阻塞管道释放持有的锁直接返回 false
	if !block {
		unlock(&c.lock)
		return false
	}

  	// 管道处于阻塞状态
	gp := getg()             // 获取当前执行协程数据结构
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
    // 把当前协程修改成等待状态并且释放锁，当前协程将会被重新调度运行
	goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)
	// 协程被挂起
    
    // 数据被其它线程取出就会唤醒本线程，然后做后续的数据处理操作
    // 协程被唤醒，继续执行协程
    // 判定当前协程的等待执行数据在挂起是否被修改
    // 被占用直接抛出异常
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	return true
}

/*
 * send 函数处理空管道上的发送操作。发送者发送的数据 ep 将被复制给接收者 sg，
 * 然后接收者被唤醒。
 * 管道 c 一定为空且被锁定。send 函数使用 unlockf 函数来释放锁
 * sg 必定是已经从管道 c 接收者队列取下来的数据
 * 数据 ep 必须是非空且是指向堆或者调用者栈上的数据
 */
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if raceenabled {         // 冲突监测代码不用管
		if c.dataqsiz == 0 {
			racesync(c, sg)
		} else {
			// Pretend we go through the buffer, even though
			// we copy directly. Note that we need to increment
			// the head/tail locations only when raceenabled.
			qp := chanbuf(c, c.recvx)
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
			c.recvx++
			if c.recvx == c.dataqsiz {
				c.recvx = 0
			}
			c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
		}
	}
    // 上面是冲突监测代码，不影响正常的逻辑，忽略上面的代码
    
    // 
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
    // 以参数的形式挂在 sg 到接收者协程上
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)       // 唤醒接收者协程
}

/*
 * 对于无缓冲或者空缓冲的管道，发送和接收的唯一操作就是一个协程给另一个协程的
 * 栈上写数据。
 */
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// Once we read sg.elem out of sg, it will no longer
	// be updated if the destination's stack gets copied (shrunk).
	// So make sure that no preemption points can happen between read & use.
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}

func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// dst is on our stack or the heap, src is on another stack.
	// The channel is locked, so src will not move during this
	// operation.
	src := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}

func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(unsafe.Pointer(c), callerpc, funcPC(closechan))
		racerelease(unsafe.Pointer(c))
	}

	c.closed = 1

	var glist *g

	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, unsafe.Pointer(c))
		}
		gp.schedlink.set(glist)
		glist = gp
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, unsafe.Pointer(c))
		}
		gp.schedlink.set(glist)
		glist = gp
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	for glist != nil {
		gp := glist
		glist = glist.schedlink.ptr()
		gp.schedlink = 0
		goready(gp, 3)
	}
}

// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not ready for receiving, we observe that the
	// channel is not closed. Each of these observations is a single word-sized read
	// (first c.sendq.first or c.qcount, and second c.closed).
	// Because a channel cannot be reopened, the later observation of the channel
	// being not closed implies that it was also not closed at the moment of the
	// first observation. We behave as if we observed the channel at that moment
	// and report that the receive cannot proceed.
	//
	// The order of operations is important here: reversing the operations can lead to
	// incorrect behavior when racing with a close.
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(unsafe.Pointer(c))
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}

// recv processes a receive operation on a full channel c.
// There are 2 parts:
// 1) The value sent by the sender sg is put into the channel
//    and the sender is woken up to go on its merry way.
// 2) The value received by the receiver (the current G) is
//    written to ep.
// For synchronous channels, both values are the same.
// For asynchronous channels, the receiver gets its data from
// the channel buffer and the sender's data is put in the
// channel buffer.
// Channel c must be full and locked. recv unlocks c with unlockf.
// sg must already be dequeued from c.
// A non-nil ep must point to the heap or the caller's stack.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
		}
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}

// compiler implements
//
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}

// compiler implements
//
//	select {
//	case v = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbrecv(&v, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

// compiler implements
//
//	select {
//	case v, ok = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if c != nil && selectnbrecv2(&v, &ok, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
	// TODO(khr): just return 2 values from this function, now that it is in Go.
	selected, *received = chanrecv(c, elem, false)
	return
}

//go:linkname reflect_chansend reflect.chansend
func reflect_chansend(c *hchan, elem unsafe.Pointer, nb bool) (selected bool) {
	return chansend(c, elem, !nb, getcallerpc())
}

//go:linkname reflect_chanrecv reflect.chanrecv
func reflect_chanrecv(c *hchan, nb bool, elem unsafe.Pointer) (selected bool, received bool) {
	return chanrecv(c, elem, !nb)
}

//go:linkname reflect_chanlen reflect.chanlen
func reflect_chanlen(c *hchan) int {
	if c == nil {
		return 0
	}
	return int(c.qcount)
}

//go:linkname reflect_chancap reflect.chancap
func reflect_chancap(c *hchan) int {
	if c == nil {
		return 0
	}
	return int(c.dataqsiz)
}

//go:linkname reflect_chanclose reflect.chanclose
func reflect_chanclose(c *hchan) {
	closechan(c)
}

func (q *waitq) enqueue(sgp *sudog) {
	sgp.next = nil
	x := q.last
	if x == nil {
		sgp.prev = nil
		q.first = sgp
		q.last = sgp
		return
	}
	sgp.prev = x
	x.next = sgp
	q.last = sgp
}

func (q *waitq) dequeue() *sudog {
	for {
		sgp := q.first
		if sgp == nil {
			return nil
		}
		y := sgp.next
		if y == nil {
			q.first = nil
			q.last = nil
		} else {
			y.prev = nil
			q.first = y
			sgp.next = nil // mark as removed (see dequeueSudog)
		}

		// if a goroutine was put on this queue because of a
		// select, there is a small window between the goroutine
		// being woken up by a different case and it grabbing the
		// channel locks. Once it has the lock
		// it removes itself from the queue, so we won't see it after that.
		// We use a flag in the G struct to tell us when someone
		// else has won the race to signal this goroutine but the goroutine
		// hasn't removed itself from the queue yet.
		if sgp.isSelect {
			if !atomic.Cas(&sgp.g.selectDone, 0, 1) {
				continue
			}
		}

		return sgp
	}
}

func racesync(c *hchan, sg *sudog) {
	racerelease(chanbuf(c, 0))
	raceacquireg(sg.g, chanbuf(c, 0))
	racereleaseg(sg.g, chanbuf(c, 0))
	raceacquire(chanbuf(c, 0))
}
```

