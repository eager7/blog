# Golang调度

## go调度含义

go调度问题是指go runtime将程序内众多routine按照一定算法调度到‘CPU’上运行，这里的‘CPU’实际上指的是线程资源，go语言自带调度器，因此成为原生支持并发。

## go的调度模型

go调度使用GPM模型，G-routine，P-processer，M-machine。 这里的P是一个逻辑处理器，M是线程资源，G绑定P，P绑定M，实现CPU调度资源的分配。 三者都在runtime2.go中定义，他们之间的关系如下：

* G需要绑定在M上才能运行；
* M需要绑定P才能运行；
* 程序中的多个M并不会同时都处于执行状态，最多只有GOMAXPROCS个M在执行。

早期版本的Golang是没有P的，调度是由G与M完成。 这样的问题在于每当创建、终止Goroutine或者需要调度时，需要一个全局的锁来保护调度的相关对象\(sched\)。 全局锁严重影响Goroutine的并发性能。 通过引入P，实现了一种叫做work-stealing的调度算法：

* 每个P维护一个G队列；
* 当一个G被创建出来，或者变为可执行状态时，就把他放到P的可执行队列中；
* 当一个G执行结束时，P会从队列中把该G取出；如果此时P的队列为空，即没有其他G可以执行， 就随机选择另外一个P，从其可执行的G队列中偷取一半。

该算法避免了在Goroutine调度时使用全局锁。

## 抢占式调度

注意到上面P的个数类似于通道个数，通过会设置成系统CPU核数，因为CPU核数是底层线程并发的真正数量，可以实现最好的性能，这个值是通过`runtime.GOMAXPROCS`函数设置的，通过使用默认值即可。 当P中某个G出现异常或者BUG，持续占用P时会导致其余P无法得到执行，因此Go的调度器支持弱抢占机制。 这里之所以称之为弱抢占机制是因为在实时系统中抢占式调度是通过CPU的时间片轮转实现的，Go的抢占式调度则是通过一个daemon协程sysmon实现的，它并不具备操作系统内核级的硬件中断能力，基于工作窃取的调度器实现，本质上属于 先来先服务的协作式调度。 异步抢占式调度的一种方式就与运行时系统监控有关，监控循环会将发生阻塞的 Goroutine 抢占， 解绑 P 与 M，从而让其他的线程能够获得 P 继续执行其他的 Goroutine。 这得益于 sysmon 中调用的 retake 方法。这个方法处理了两种抢占情况， 一是抢占阻塞在系统调用上的 P，二是抢占运行时间过长的 G。 其中抢占运行时间过长的 G 这一方式还会出现在垃圾回收需要进入 STW 时。 retake调用间隔为10ms。

```text
// forcePreemptNS is the time slice given to a G before it is// preempted.const forcePreemptNS = 10 * 1000 * 1000 // 10msfunc retake(now int64) uint32 {
```

## 参考材料：

\[也谈goroutine调度器\]\([https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)\) \[Golang调度器源码分析\]\([http://ga0.github.io/golang/2015/09/20/golang-runtime-scheduler.html](http://ga0.github.io/golang/2015/09/20/golang-runtime-scheduler.html)\)

