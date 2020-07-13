# Channel实现

\[TOC\]

## 数据结构

源码位于`/usr/local/go/src/runtime/chan.go`：

```text
type hchan struct {
    qcount   uint           // total data in the queue 队列中的所有数据数
    dataqsiz uint           // size of the circular queue 环形队列的大小
    buf      unsafe.Pointer // points to an array of dataqsiz elements 指向大小为 dataqsiz 的数组
    elemsize uint16 //元素大小
    closed   uint32 //是否关闭
    elemtype *_type // element type 元素类型
    sendx    uint   // send index 发送索引
    recvx    uint   // receive index 接收索引
    recvq    waitq  // list of recv waiters recv 等待列表，即（ <-ch ）
    sendq    waitq  // list of send waiters send 等待列表，即（ ch<- ）


    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
- buf里存放channel数据，是一个环形队列
- elemtype是channel里存放的数据类型，这里和反射一样用type类型存储
- sendx，recvx是接收和发送的偏移
- recvq，sendq是和channel相关的协程队列
```

## 创建channel

make一个channel实际上就是分配了上面的结构体对象，然后返回指针，因此channel本质上是一个结构体指针。 创建过程会根据数据类型进行不同的分支操作，但本质上都是在堆上分配一块内存给channel存储数据，所以channel如果不显式释放也会被垃圾回收器处理。

## 写channel

### 写空channel

```text
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {
        if !block {
            return false
        }
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
```

如果向一个未初始化的channel写数据，会panic。

### 写已关闭channel

```text
    lock(&c.lock)


    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
```

写已关闭channel也会panic。

### 写有等待者的channel

```text
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
```

recvq是上面结构中的等待协程队列，如果能从中找到协程，说明有其他协程正在阻塞等待这个channel的数据，此时，直接调用send把数据拷贝到对端协程的对象中，也就是不需要走channel的buf了。少了中间商时间差，同时，在将数据拷贝到等待协程后，会把等待协程放到调度器上层，让调度器可以立即调度到等待协程，实现写入即被接收的效果。

### 写有缓冲，无等待者的channel

队列中的数据还未填满buf，则将数据拷贝到buf中返回：

```text
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
```

### 写阻塞channel

获取协程变量，将自己挂到channel的发送协程队列中，然后挂起自己，等待被接收协程唤醒。

```text
    // Block on the channel. Some receiver will complete our operation for us.
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
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg)
    goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
```

## 读channel

### 读空channel

读空channel和写空channel一样，会导致fatal：

```text
    if c == nil {
        if !block {
            return
        }
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
```

### 读已关闭且无数据channel

读关闭channel不会报错，但是如果channel中无数据，则直接返回

### 读有发送方的channel

当channel上有阻塞的发送者时，会从对端直接拷贝数据，不走buf，同时唤醒对方协程。

### 无等待者，有buf

从buf拷贝数据，返回

### 读buf为空的channel

将自己挂在等待协程队列中，同时挂起自己等待被发送者唤醒

## 关闭channel

首先对 Channel 上锁，而后依次将阻塞在 Channel 的 g 添加到一个 gList 中，当所有的 g 均从 Channel 上移除时，可释放锁，并唤醒 gList 中的所有接收方和发送方。

