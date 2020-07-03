# go-defer

\[TOC\]

## 源码

`runtime/runtime2.go` defer数据结构：

```text
// A _defer holds an entry on the list of deferred calls.
// If you add a field here, add code to clear it in freedefer.
type _defer struct {
   siz     int32 //是参数和结果的内存大小；
   started bool
   sp      uintptr // sp at time of defer
   pc      uintptr //sp 和 pc 分别代表栈指针和调用方的程序计数器；
   fn      *funcval //fn 是 defer 关键字中传入的函数；
   _panic  *_panic // panic that is running defer //_panic 是触发延迟调用的结构体，可能为空；
   link    *_defer
}
```

可以看到defer是一个链表，定义多个defer会组织成一条链依次调用： 

## 预计算参数

defer后面跟的是一个函数，如果是一个正常函数，根据golang值传递原理，在定义defer时值就会被拷贝，因此虽然函数是延时调用的，但是值却是实时赋值。 如果想实现值的延时赋值，则需要为defer传递一个闭包，闭包的原理就是一个函数指针，那么传递给defer的是指针拷贝，而不是函数拷贝，就能实现值的延时赋值。

## 编译

编译源码：

```text
func (s *state) stmt(n *Node) {


    switch n.Op {


    case ODEFER:


        s.call(n.Left, callDefer)


    }


}
```

从这里看到，编译器把defer当做了函数调用，因此`defer fmt.Println(1)`本质上如同`defer(fmt.Println(1))`，这里就解释了预计算参数的问题，defer是函数，那么后面的函数就是参数了。

## defer执行

defer是在return之前执行的。这个在 官方文档中是明确说明了的。要使用defer时不踩坑，最重要的一点就是要明白，return xxx这一条语句并不是一条原子指令! 函数返回的过程是这样的：先给返回值赋值，然后调用defer表达式，最后才是返回到调用函数中。 defer表达式可能会在设置函数返回值之后，在返回到调用函数之前，修改返回值，使最终的函数返回值与你想象的不一致。 其实使用defer时，用一个简单的转换规则改写一下，就不会迷糊了。改写规则是将return语句拆成两句写，return xxx会被改写成:

```text
返回值 = xxx
调用defer函数
空的return
```

