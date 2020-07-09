# wagon外部参数和内部参数的统一

\[TOC\]

## 参数存在的问题

上一章节我们说到下面的合约代码会出现问题：

```text
int test(char *string){
  func(string);
  func("string");
}
```

在编译完成后对两个参数的处理是不同的：

```text
(module
   (type $FUNCSIG$i (func (result i32)))
   (type $FUNCSIG$ii (func (param i32) (result i32)))
   (import "env" "func" (func $func (param i32) (result i32)))
   (table 0 anyfunc)
   (memory $0 1)
   (data (i32.const 16) "string\00")
   (export "memory" (memory $0))
   (export "test" (func $test))
   (func $test (; 1 ;) (param $0 i32) (result i32)
      (drop
         (call $func            (get_local $0)
         )
      )
      (drop
         (call $func            (i32.const 16)
         )
      )
      (get_local $0)
   )
)
```

一种是通过外部传入指针，然后转换，一种是通过从虚拟机内部memory的偏移获取到参数，两者处理逻辑不同，这就会导致func在执行时出错，因此需要一种统一的方式来处理参数，或许还是需要借鉴ONT的方式了。 因为两个参数存在不同的地方，外部传入参数是存在local中的，内部参数是存在memory中的，没法通过一个相同的方法获取到参数变量，因此需要一种统一的方法。

## ONT处理方式

ONT的处理方式是全面改写虚拟机内存，然后将外部参数和内部参数统一从memory中获取：  从图中可以看到，右边的ONT代码中VM的memory被修改成了自己的memory，而原生wagon memory则是一个byte序列，ONT memory结构如下：

```text
type VMmemory struct {
   Memory          []byte   AllocedMemIdex  int   PointedMemIndex int   ParamIndex      int //args analyze pointer   MemPoints       map[uint64]*TypeLength
}
```

如果是字节序列，则会将字节序列拷贝到memory中：

```text
func (vm *VMmemory) copyMemAndGetIdx(b []byte, p_type PType) (int, error) {
   idx, err := vm.MallocPointer(len(b), p_type)
   if err != nil {
      return 0, err
   }
   copy(vm.Memory[idx:idx+len(b)], b)

   return idx, nil
}
```

从而实现内外部参数统一。

## 一种新的方式？

是否存在一种新方式，可以最大化保留原本wagon特性，并且实现内外部参数统一呢？

