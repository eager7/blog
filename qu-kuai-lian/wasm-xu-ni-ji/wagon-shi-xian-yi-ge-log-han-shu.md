# Wagon实现一个log函数

\[TOC\]

## 实现方案

如果要实现一个自己的函数，有两种实现方式，一种是在虚拟机内部直接以内置指令的方式实现，如add函数等，但是这种方式需要修改编译器，并且通用性不够好，另一种就是通过解决外部引用的方式实现，这种的兼容性较好，需要解决的问题就是在内部解析引用的时候实现内部导入。

## 案例分析

如果我们需要实现一个log函数，那么就需要解决外部符号导入问题，如下面代码：

```text
int main() { 


  Println("hello world");



  return 0;



}
```

它的wast文件为：

```text
(module


 (type $FUNCSIG$i (func (result i32)))



 (type $FUNCSIG$ii (func (param i32) (result i32)))



 (import "env" "Println" (func $Println (param i32) (result i32)))



 (table 0 anyfunc)



 (memory $0 1)



 (data (i32.const 16) "hello world\00")



 (export "memory" (memory $0))



 (export "main" (func $main))



 (func $main (; 1 ;) (result i32)



  (drop



   (call $Println



    (i32.const 16)



   )



  )



  (i32.const 0)



 )



)
```

它的wasm文件内容如下：

```text
pct@Chandler:~/go/src/github.com/go-interpreter/wagon/cmd/wasm-run$ wasm-dump -s hello.wasm
hello.wasm: module version: 0x1


contents of section type:
0000000e  02 60 00 01 7f 60 01 7f  01 7f                    |.`...`....|


contents of section import:
0000001e  01 03 65 6e 76 07 50 72  69 6e 74 6c 6e 00 01     |..env.Println..|


contents of section function:
00000033  01 00                                             |..|


contents of section table:
0000003b  01 70 00 00                                       |.p..|


contents of section memory:
00000045  01 00 01                                          |...|


contents of section global:
0000004e  00                                                |.|


contents of section export:
00000055  02 06 6d 65 6d 6f 72 79  02 00 04 6d 61 69 6e 00  |..memory...main.|
00000065  01                                                |.|


contents of section code:
0000006c  01 89 80 80 80 00 00 41  10 10 00 1a 41 00 0b     |.......A....A..|


contents of section data:
00000081  01 00 41 10 0b 0c 68 65  6c 6c 6f 20 77 6f 72 6c  |..A...hello worl|
00000091  64 00                                             |d.|
```

解析器在解析文件时需要找到env中的Println函数，然后将hello world放入堆栈，接着调用call函数执行Println函数打印出hello world，那么我们需要解决的就是在虚拟机内部实现env中Println函数的解析部分。

## 解析流程

在上一章节中我们分析过，文件的解析主要在函数wasm.ReadModule\(f, importer\)中，它首先循环读取各个分区：

```text
    for {


        done, err := m.readSection(reader)



        if err != nil {



            return nil, err



        } else if done {



            break



        }



    }
```

然后再去解决导入的函数：

```text
    if m.Import != nil && resolvePath != nil {


        err := m.resolveImports(resolvePath)



        if err != nil {



            return nil, err



        }



    }
```

这里的解决办法是从env的wasm文件中读取导入的函数：

```text
    for _, importEntry := range module.Import.Entries {


        importedModule, ok := modules[importEntry.ModuleName]



        if !ok {



            var err error



            importedModule, err = resolve(importEntry.ModuleName)



            if err != nil {



                return err



            }



            modules[importEntry.ModuleName] = importedModule



        }



        index := exportEntry.Index



        switch exportEntry.Kind {



        case ExternalFunction:



            fn := importedModule.GetFunction(int(index))



            if fn == nil {



                return InvalidFunctionIndexError(index)



            }



            module.FunctionIndexSpace = append(module.FunctionIndexSpace, *fn)



            module.Code.Bodies = append(module.Code.Bodies, *fn.Body)



            module.imports.Funcs = append(module.imports.Funcs, funcs)



            funcs++
```

然后再创建新的vm时会将函数列表放入vm中：

```text
vm, err := exec.NewVM(m)
vm.newFuncTable()
vm.funcTable[ops.Call] = vm.call
```

最后根据堆栈来调用相关函数：

```text
    index := vm.fetchUint32()


    vm.funcs[index].call(vm, int64(index))
```

函数从module到vm的拷贝过程如下：

```text
    for i, fn := range module.FunctionIndexSpace {


        if fn.IsHost() {



            vm.funcs[i] = goFunction{



                typ: fn.Host.Type(),



                val: fn.Host,



            }



            nNatives++



            continue



        }



        disassembly, err := disasm.Disassemble(fn, module)



        if err != nil {



            return nil, err



        }



        totalLocalVars := 0



        totalLocalVars += len(fn.Sig.ParamTypes)



        for _, entry := range fn.Body.Locals {



            totalLocalVars += int(entry.Count)



        }



        code, table := compile.Compile(disassembly.Code)



        vm.funcs[i] = compiledFunction{



            code: code,



            branchTables: table,



            maxDepth: disassembly.MaxDepth,



            totalLocalVars: totalLocalVars,



            args: len(fn.Sig.ParamTypes),



            returns: len(fn.Sig.ReturnTypes) != 0,



        }



    }
```

那么这里的问题就是如何在保存现有解析新wasm文件的基础上，跳过对Println函数的解析呢？而且需要根据fn的规则实现一个内部的Println。

## 工程改造

首先是跳过部分，这部分比较简单，我们在解析外部引用时手动判断env并跳过即可：

```text
for _, importEntry := range module.Import.Entries {
   if importEntry.ModuleName == "env" {
      fmt.Println("Module Name:", importEntry.ModuleName, "- Filed Name:", importEntry.FieldName)
      if importEntry.Kind == ExternalFunction {
         //get the function type         funcType := module.Types.Entries[importEntry.Type.(FuncImport).Type]
         var code []byte         code = append(code, 0x41)
         code = append(code, 0x0b)
         fn := &Function{IsEnv: true, Name: importEntry.FieldName, Sig: &FunctionSig{ParamTypes: funcType.ParamTypes, ReturnTypes: funcType.ReturnTypes}, Body: &FunctionBody{Code:code}}
         module.FunctionIndexSpace = append(module.FunctionIndexSpace, *fn)
         module.Code.Bodies = append(module.Code.Bodies, *fn.Body)
         module.imports.Funcs = append(module.imports.Funcs, funcs)
         funcs++
      }
   }
```

这里就跳过了对env中Println的解析，注意这里需要添加code，否则会导致空指令的执行，这里的41表示call指令，我们需要在函数执行call时对自己的函数进行处理。 还有就是我们添加了两个变量，IsEnv和Name，这两个变量可以判断函数是否是我们自己实现的，从而实现外部函数的分支调用。

下一步就是创建新的VM了，创建新的VM时需要在函数列表中添加上我们加入的两个变量：

```text
    vm.funcs[i] = compiledFunction{
      code:           code,      branchTables:   table,      maxDepth:       disassembly.MaxDepth,      totalLocalVars: totalLocalVars,      args:           len(fn.Sig.ParamTypes),      returns:        len(fn.Sig.ReturnTypes) != 0,      IsEnv:          fn.IsEnv,      Name:           fn.Name,   }
}
```

然后函数执行时就会调用到我们放入的code：

```text
func (vm *VM) ExecCode(fnIndex int64, args ...uint64) (rtrn interface{}, err error) {
   compiled, ok := vm.funcs[fnIndex].(compiledFunction)
   if !ok {
      panic(fmt.Sprintf("exec: function at index %d is not a compiled function", fnIndex))
   }
   if len(vm.ctx.stack) < compiled.maxDepth {
      vm.ctx.stack = make([]uint64, 0, compiled.maxDepth)
   }
   vm.ctx.locals = make([]uint64, compiled.totalLocalVars)
   vm.ctx.pc = 0   vm.ctx.code = compiled.code
   vm.ctx.curFunc = fnIndex

   for i, arg := range args {
      vm.ctx.locals[i] = arg
   }

   res := vm.execCode(compiled)
```

code里是0x41和0x2b，执行VM的函数调用：

```text
func (vm *VM) execCode(compiled compiledFunction) uint64 {
outer:
   for int(vm.ctx.pc) < len(vm.ctx.code) {
      op := vm.ctx.code[vm.ctx.pc]
      vm.ctx.pc++
      switch op {
      default:
         fmt.Printf("Execte Code:0x%02x\n", op)
         vm.funcTable[op]()
      }
   }
```

funcTable是在创建VM时设置的，实际调用的是vm的call函数：

```text
func (vm *VM) call() {
   index := vm.fetchUint32()
   fun, ok := vm.funcs[index].(compiledFunction)
   if ok {
      if fun.IsEnv {
         if fun.Name == "Println" {
            fmt.Println("Println Call log aba")
            vm.popUint64()
            vm.pushUint64(0)
            return         }
      }
   }
   vm.funcs[index].call(vm, int64(index))
}
```

我们可以在这个函数中做分支，实现自己的API方法，当然，现在的Println函数并没有对堆栈进行处理，只是最简单的流程，如果涉及到堆栈处理，就更复杂一些了。

