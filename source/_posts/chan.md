---
title: channel 和 select
date: 2020-08-30 20:46:40
tags:
categories:
---
<meta name="referrer" content="no-referrer" />
 golang中经典的就是goroutine和channel这两个的设计，其中channel是引用CSP理论设计的，不要用内存来通讯，用通讯来贡献内存。channel提供一种跨goroutine通讯的方式。

## channel 结构

channel类型的具体结构由`runtime/chan.go`中`hchan`定义，结构如下

chan.go
```go
type hchan struct {
   qcount   uint           // 队列中数据总量
   dataqsiz uint           // 环形队列的大小，> 0表示有缓冲，= 0表示无缓冲
   buf      unsafe.Pointer // 指向元素数组的指针，一个环形队列
   elemsize uint16         // 单个元素的大小
   closed   uint32         // 表明是否close了
   elemtype *_type                  // 元素类型，interface类型相关
   sendx    uint                    // send数组的索引, c <- i
   recvx    uint                    // receive 数组的索引 <- c
   recvq    waitq                   // 等待recv 数据的goroutine的链表
   sendq    waitq                   // 等待send数据的goroutine链表
   lock mutex
}
```

channel内由一个环形队列`buf`存储通讯的数据，`closed`代表是否关闭,`recvq`和`sendq`分别是等待读取数据的goroutine的链表和等待send数据的goroutine链表

`waitq`是一个双向列表
```go
type waitq struct {
	first *sudog  // 头节点
	last  *sudog  // 尾节点
}

// sudogo 存储goroutine和前后节点的双向链标的节点
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g // goroutine

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool // 是否在select中
	next     *sudog // 后一个节点
	prev     *sudog // 前一个节点
	elem     unsafe.Pointer // 传输的数据指针

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot binary tree
	waitlink    *sudog // g.waiting list or semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // sudog channel
}
```

## 创建channel
调用make函数,可以创建channel,最后会调用makechan函数来生成

```go
// t是传入的类型, size是长度
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
    // 超出长度抛异常
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```
 创建channel过程
 1. 检查参数合法性size是否合法，创建容量是否超过规定大小。channel size不能大于1<<16
 2. 若创建所需buf内存为0，则是一个无buffchannel，直接申请对应内存不为buf分配对应内存
 3. 若size不为0，为buf申请所需内存

## 往channel发送数据
 `c <- 1`这样是往channel发送数据，编译器会自动生成中间代码调用`chansend1`函数发送数据，`chansend1`调用`chansend`来发送数据，主要逻辑都在`chansend`中

```go
// chansend 是往channel发送数据主要实现逻辑
// c:发送数据的channel，
// ep:发送数据的指针,
// block: 发送是否阻塞调用，如`c <- 1`这样的发送过程都是阻塞调用，若buf满了会阻塞当前goroutine，非阻塞用于select中，这个后面会讲。
// callerpc: debug相关
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 往空channel发送数据会一直阻塞
    // gopark是goroutine调度相关，让一个goroutine进入gowait状态
    if c == nil {
        // 非阻塞直接返回
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// 非阻塞情况channel为空时或数据写满且没有等待接收数据的goroutine时候返回失败
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}
   
    // 上锁 防止data race
	lock(&c.lock)

    // 往已经关闭的channel写数据抛出异常	
    if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

    // 从接受者等待队列找出一个接受者，若有则调用send,让接受者接收
    // send函数这里不展开，主要完成一下操作
    // 1. 将数据复制到sudog的data中
    // 2. 调用goready让等待goroutine进入goready状态可以被重新唤醒调用
    // 3. 解除hchan的锁
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }   
    
    // 检查buf是否满了，若未满写入buf中返回true
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

    // 非阻塞情况，不会阻塞当前goroutine 直接返回
	if !block {
		unlock(&c.lock)
		return false
	}

    // 下面为阻塞当前goroutine的操作
    // 获取当前goroutine
    // 填充sud😯g对应值，然后写入到等待发送队列中
    // 调用gopark，将当前goroutine进入gowait状态
    // 等待当前goroutine被唤醒
	gp := getg()          // 获取当前goroutine
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
    // 调用gopark，将当前goroutine进入gowait状态，解除当前hchan的锁 进行阻塞
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
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
```

## 从channel读取数据
`a := <- c` 这个是从channel读取数据，编译器会自动生成中间代码调用`chansend1`函数发送数据，`chanrecv1`调用`chanrecv`来发送数据，主要逻辑都在`chanrecv`中，与发送数据有些相似

```go
// chanrecv 从channel读取数据主要实现逻辑
// c:读取数据的channel，
// ep:接收数据的指针,
// block: 发送是否阻塞调用，如`a := <- c`这样的发送过程都是阻塞调用，若buf满了会阻塞当前goroutine，非阻塞用于select中，这个后面会讲。
// callerpc: debug相关
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 往一个空的channel读取数据会一直阻塞
    if c == nil {
        // 非阻塞直接返回
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
 
    // 非阻塞情况下，channel的buf为空且没有发送等待队列也为空时候可以直接返回
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}
    
    // 上锁防止data race
 	lock(&c.lock)

    // 已经关闭的channel且buf为空，直接返回
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	// 从发送者等待队列找出一个发送者，若有则调用recv,让发送者接收
    // send函数这里不展开，主要完成一下操作
    // 1. 从sudog的data中复制数据到ep
    // 2. 调用goready让等待goroutine进入goready状态可以被重新唤醒调用
    // 3. 解除hchan的锁
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
    // buf中有数据，从buf中读取后返回
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

    // 非阻塞直接返回
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 下面为阻塞操作
    // 获取当前goroutine
    // 填充sud😯g对应值，然后写入到等待接收队列中
    // 调用gopark，将当前goroutine进入gowait状态
    // 等待当前goroutine被唤醒
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
    // 调用gopark，将当前goroutine进入gowait状态，解除当前hchan的锁 进行阻塞
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```

## 关闭channel
`close(c)`调用close函数会关闭chanel，编译后代码会调用`closechan`关闭chanel

```go
func closechan(c *hchan) {
    // 关闭一个空channel抛出异常
	if c == nil {
		panic(plainError("close of nil channel"))
	}

    // 上锁
	lock(&c.lock)
    // 关闭已经关闭的channel抛出异常	
    if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}
    // 设置关闭标记位
	c.closed = 1

    // goroutine链表
	var glist gList

	// 从等待接受者队列中找出所有接受者goroutine
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
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// 从等待接受者队列中找出所有接受者goroutine
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
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// 遍历链表，逐个调用goready，唤醒器goroutine
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

## select关键字
 使用select关键字可以从case中任一个可读或可写的channel进行操作，有点像io中select模型。其中会分为四种情况
 
 1. 空case如`select{}`
 2. 单case，如 `selct{case <-c: }`
 3. 两个case一个channel一个default，如`select{case <-c: ;default:}`
 4. 多case，除上述任意情况，这个情况调用`selectgo`函数
 
### 空case
 空case时候编译时，编译器会做处理，这个时候直接调用block函数，让当前goroutine处于阻塞状态

### 单case
 这个时候编译器做处理，将单case情况，转换成对单个chanel读或写, `selct{case <-c: }`转换成`<-c`,`selct{case c<-1: }`转换成`c<-1`

### 两个case一个channel一个default
 这个时候就会用到，上面发送或接收中的非阻塞状态，编译器处理如下
 
 ```go
 select {
   case <-c:
   default:
 }
 ``` 

等价如

```go
_, ok := chanrecv(hch, data, false, nil)
if ok {
// case中xxx
} else {
// default中xxx
}
```

## selectgo 函数
 selct中其他情况会调用到这个函数，首先将case包装成scase结构
 
 ```go
const (
	caseNil = iota
	caseRecv
	caseSend
	caseDefault
)

type scase struct {
	c           *hchan         // case中的channel，caseDefault时候为nil
	elem        unsafe.Pointer // data element
	kind        uint16 // kind 种类 有接收类型 读取类型和Default,对应case中操作
	pc          uintptr // race pc (for race detector / msan)
	releasetime int64
}
```

调用`selectgo`函数，会保证对每一个case随机，不会优先判断某个case
```go
    // 输入的case数组
    cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
    // 判断case的顺序
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
    // channel上锁顺序
	lockorder := order1[ncases:][:ncases:ncases]

	// Replace send/receive cases involving nil channels with
	// caseNil so logic below can assume non-nil channel.
    // 检查scases，为caseDefault类型赋值一个空struct
	for i := range scases {
		cas := &scases[i]
		if cas.c == nil && cas.kind != caseDefault {
			*cas = scase{}
		}
	}
```

保证随机打散order，和打散channel上锁顺序
```go
	// generate permuted order
    // 使用fastrandn函数打乱order
	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[i] = pollorder[j]
		pollorder[j] = uint16(i)
	}

	// 根据每个case中的channel的地址来做怼排序决定上锁顺序
    // 因为内存中地址比较随机，可以认为是随机的顺序
	for i := 0; i < ncases; i++ {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	for i := ncases - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}

	// 根据锁顺序上锁
	sellock(scases, lockorder)
```

判断每一个case是否可以执行，这里是`selectgo`函数的关键部分
```go
loop:
	// pass 1 - look for something already waiting
	var dfli int
	var dfl *scase
	var casi int
	var cas *scase
	var recvOK bool
    // 遍历ncases 这个时候case已经被打乱
	for i := 0; i < ncases; i++ {
		casi = int(pollorder[i])
		cas = &scases[casi]
		c = cas.c

		switch cas.kind {
		case caseNil:
			continue

		case caseRecv:
            // 判断发送等待队列是否有发送者若有，recv操作和channel操作一样
            sg = c.sendq.dequeue()
			if sg != nil {
				goto recv
			}
            // 判断buf是否为空，不为空从buf读
			if c.qcount > 0 {
				goto bufrecv
			}
			if c.closed != 0 {
                // 从已经关闭channel读这里会返回
				goto rclose
			}

		case caseSend:
            // channel已经关闭，这里会panic
			if c.closed != 0 {
				goto sclose
			}
            // 从接收者等待队列获取，若有接受者则发送和发送channel流程一样
            sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
            // 判断buf是否满了，未满写入buf，和写入channelbuf流程一样
			if c.qcount < c.dataqsiz {
				goto bufsend
			}

		case caseDefault:
            // 先赋值default，case
			dfli = casi
			dfl = cas
		}
	}
    // 遍历case结束，走到这里，是每一个case中的channel都没有一个符合

    // 判断是否有default，case有则直接返回
	if dfl != nil {
		selunlock(scases, lockorder)
		casi = dfli
		cas = dfl
		goto retc
	}
```

如果没有default case且case中也没有一个channel符合要求，这个时候会阻塞
```go
	// 获取当前goroutine
	gp = getg()
    // 如果一个有一个等待时间抛出异常
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
    // 按照入锁顺序遍历
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		if cas.kind == caseNil {
			continue
		}
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// Construct waiting list in lock order.
		*nextp = sg
		nextp = &sg.waitlink
		
		switch cas.kind {
        // 如果是接收，进入hch的接受者等待队列
		case caseRecv:
			c.recvq.enqueue(sg)
        // 如果是发送，进入hch的发送者等待队列
		case caseSend:
			c.sendq.enqueue(sg)
		}
	}

	// 等待被唤醒
	gp.param = nil
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)
	gp.activeStackChans = false

    // 重新获取锁
	sellock(scases, lockorder)

	gp.selectDone = 0
	sg = (*sudog)(gp.param)
	gp.param = nil

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
	casi = -1
	cas = nil
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

    // 按照上锁顺序遍历，
    // 其余goroutine出队
	for _, casei := range lockorder {
		k = &scases[casei]
		if k.kind == caseNil {
			continue
		}
		if sglist.releasetime > 0 {
			k.releasetime = sglist.releasetime
		}
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			casi = int(casei)
			cas = k
		} else {
			c = k.c
			if k.kind == caseSend {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}
```

## 小结
 channel是golang实现CSP理论的关键部分，结合select和channel源码能更好理解channel