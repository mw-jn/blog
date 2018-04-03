---
title: GoLang：理解GC
date: 2018-03-17 18:39:10
tags:
    - 面试
    - gc
categories: "golang"
---
虽然写 GoLang 差不多两年，但是还不知道 GoLang 中 GC 是怎么实现的？就查了各种资料了解下 GoLang 中 GC 的实现原理。GoLang 中的 GC 是一种并发三色标记清除的垃圾回收器。

![golang-gc](/images/golang-gc.png)
<!-- more -->
GC 算法执行阶段：
1、GC 关闭阶段；
2、写避障(Write Barrier)开启阶段，该阶段 GC 将做三件事：
​	GC 将全局和各协程栈上的指针变量作为根节点(root 节点)进行扫描
​	沿着根节点标记所有对象直到所有对象被遍历
​	标记结束( STW：Stop The World ) ，重新扫描全局和变化的栈数据，完成标记，退栈等操作
3、清理阶段，回收没有被标记的对象，调整下一轮 GC 触发的时机
4、重复阶段1

下面展示 GC 的工作原理，参考 Golang 实时 GC 理论和实践这个篇文章 [Golang’s Real-time GC in Theory and Practice](https://making.pusher.com/golangs-real-time-gc-in-theory-and-practice/index.html)
GoLang GC 实现并发执行的核心是三色标记-清除算法。怎样让 GC 和程序并发运行实际上暂停时期的调度问题，调度器让 GC 短时间运行，并且和程序交替运行。

```go
var A LinkedListNode;
var B LinkedListNode;
// ...
B.next = &LinkedListNode{next: nil};
// ...
```

上面的代码主要用于维护一个链表，创建了 A、B 两个链表节点，节点 A 和 B 在栈上，因此是根节点，也就是 GC 扫描的起始对象。 B.next 指向心创建的节点 C(在堆上被 B 引用)。GC 将这些对象分成三类：黑色、灰色和白色。由于 GC 还没有运行，所以当前所有节点都是白色的。如下图所示：

![golang-gc-ph1](/images/golang-gc-ph1.png)

```go
// ...
A.next = &LinkedListNode{next: nil};
// ...
```

程序把节点 D 的地址付给了 A.next。D 作为一个新对象被划分为灰色。当一个指针域发生改变，被指向的对象会被着色。由于所有新对象的地址会被其它变量引用，所有这些新对象会被立即划分为灰色。

![golang-gc-ph2](/images/golang-gc-ph2.png)

GC 开始阶段，所有的根节点被划分为灰色，这样的话节点 A、B、D都为灰色。如下图所示：

![golang-gc-ph3](/images/golang-gc-ph3.png)

由于程序和 GC 是并发执行的，我们将看到程序和 GC 交替执行过程。

GC 执行阶段：
​	GC 扫描对象的时候会将该对象划分为黑色，将该对象的孩子划分为灰色。对象 A 有一个孩子 D，并且 D 已经属于灰色集合了。在任何阶段, GC 都能统计剩余移动对象的次数，如下图还需要移动：2*|grey| + |white|。GC 在每个阶段至少移动一次，当剩余移动次数为0时，GC 扫描就结束了。

![golang-gc-ph4](/images/golang-gc-ph4.png)

```go
// ...
*(B.next).next = &LinkedListNode{next: nil};
B.next = *(B.next).next;
// ...
```

程序执行阶段：
​	1、程序创建新对象
​	程序将对象 E 的地址赋值给 C.next。对象 E 被划分为灰色。这个操作会增加 GC 剩余执行步骤。大量分配心对象会延迟 GC 最后的清理阶段。注意，白色集合仅仅增加 GC 对象的大小。当 GC 清理堆的时候将会重新使用白色集合。

![golang-gc-ph5](/images/golang-gc-ph5.png)

​	2、程序移动指针
​	程序将对象 E 的地址赋给了 B.next。这个操作将会是对象 C 变得不能被程序引用。对象 C 将被留在白色集合中，在 GC 最后的处理阶段被回收。

![golang-gc-ph6](/images/golang-gc-ph6.png)

GC 执行阶段：
​	D 没有后代要放入灰色集合中。对象 D 被划分为黑色。

![golang-gc-ph7](/images/golang-gc-ph7.png)

```go
// ...
B.next = nil;
// ...
```

程序执行阶段：
​	程序设置 B.next = nil 将使得对象 E 变得不可被程序引用。但是对象 E 现在属于灰色集合，因此它在本阶段不能被回收。难道这就意味了产生内存泄漏了么？实际上这样做事可以的：对象 E 将会在写一次 GC 循环中被回收。三色算法仅仅能保证在 GC 开始的时候不可被引用的对象在 GC 结束的时候被回收。

![golang-gc-ph8](/images/golang-gc-ph8.png)

GC 执行阶段：
​	GC 选择扫描对象 E，对象 E 被移动到黑色集合中。注意：对象 C 不能被移动到黑色集合中，它指向了对象 E，但是没有被对象 E 引用。

![golang-gc-ph9](/images/golang-gc-ph9.png)

GC 执行阶段：
​	GC 扫描灰色集合中的最后一个对象 B，并将其放入黑色集合中。灰色集合现在变成空的了，这意味着没有对象会被移动了，将会是本轮 GC 的结束阶段。

![golang-gc-ph10](/images/golang-gc-ph10.png)

GC 执行阶段：
​	GC 中没有可以移动的对象，进入清理阶段，对象 C 被回收。

![golang-gc-ph11](/images/golang-gc-ph11.png)

为了下一轮 GC 做准备，GC 不会重新着色所有对象，仅需要将黑色和白色集合互换一下。

![golang-gc-ph12](/images/golang-gc-ph12.png)

至此，GC 的着色-清理过程就解释完了。
