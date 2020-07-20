# go版本wasm解析器分析

\[TOC\]

## 解析部分

第一步是打开我们传入的文件：

```text
f, err := os.Open(flag.Arg(0))
```

然后调用下面函数来解析文件：

```text
m, err := wasm.ReadModule(f, importer)
```

参数importer是用来解决传入的wasm文件导入函数问题的。 ReadModule函数负责将文件进行解析：

```text
func ReadModule(r io.Reader, resolvePath ResolveFunc) (*Module, error) {
    magic, err := readU32(reader)


    if err != nil {



        return nil, err



    }



    if magic != Magic {



        return nil, ErrInvalidMagic



    }



    if m.Version, err = readU32(reader); err != nil {



        return nil, err



    }



    for {



        done, err := m.readSection(reader)



        if err != nil {



            return nil, err



        } else if done {



            break



        }



    }
    m.LinearMemoryIndexSpace = make([][]byte, 1)


    if m.Table != nil {



        m.TableIndexSpace = make([][]uint32, int(len(m.Table.Entries)))



    }



    if m.Import != nil && resolvePath != nil {



        err := m.resolveImports(resolvePath)



        if err != nil {



            return nil, err



        }



    }
```

首先判断魔数和版本号，这个是固定的：

```text
const (


    Magic uint32 = 0x6d736100



    Version uint32 = 0x1



)
```

然后解析各个分区：

```text
Type* — Function signature declarations


Import — Import declarations



Function* — Function declarations



Table — Indirect function table and other tables



Memory — Memory attributes



Global — Global declarations



Export — Exports



Start — Start function declaration



Element — Elements section



Code* — Function bodies



Data — Data segments
```

关于文件格式分区可参考链接： [https://rsms.me/wasm-intro](https://rsms.me/wasm-intro) 解析完成后有两个变量需要格外处理，一个是表格，一个是引用，可以参考链接： [https://developer.mozilla.org/zh-CN/docs/WebAssembly/Understanding\_the\_text\_format](https://developer.mozilla.org/zh-CN/docs/WebAssembly/Understanding_the_text_format) 表格是一个存储函数引用的替代，解决动态操作的问题，而引用则是使用其他wasm的export部分，也可以是虚拟机内部实现的部分。 m.resolveImports调用的就是main函数中的importer方法，它从当前文件中重新打开相应的wasm文件并使用。 我们可以做个测试。参见下面的引用导入测试。 这样我们就可以在检测到env时手动解决导入问题，从而实现区块链的API。

## 执行部分

执行过程则是根据export出来的部分进行顺序执行：

```text
    for name, e := range m.Export.Entries {



        i := int64(e.Index)



        fidx := m.Function.Types[int(i)]



        ftype := m.Types.Entries[int(fidx)]



        switch len(ftype.ReturnTypes) {



        case 1:



            fmt.Printf("%s() %s => ", name, ftype.ReturnTypes[0])



        case 0:



            fmt.Printf("%s() => ", name)



        default:



            log.Printf("running exported functions with more than one return value is not supported")



            continue



        }



        if len(ftype.ParamTypes) > 0 {



            log.Printf("running exported functions with input parameters is not supported")



            continue



        }



        o, err := vm.ExecCode(i)



        if err != nil {



            fmt.Printf("\n")



            log.Printf("err=%v", err)



        }



        if len(ftype.ReturnTypes) == 0 {



            fmt.Printf("\n")



            continue



        }



        fmt.Printf("%[1]v (%[1]T)\n", o)



    }
```

具体执行过程则是线性堆栈的入栈和出栈操作，我们需要关心的是如何实现自己的API。本体是通过实现了一个自己的memory包来对堆栈进行处理的。

那么函数是怎么一步一步被调用的呢，首先是上面的解析过程，解析时会得到一个module的Type，Code和Data几个部分，然后会调用populateFunctions将函数添加到FunctionIndexSpace列表中：

```text
    for codeIndex, typeIndex := range m.Function.Types {



        if int(typeIndex) >= len(m.Types.Entries) {



            return InvalidFunctionIndexError(typeIndex)



        }



        fn := Function{



            Sig: &m.Types.Entries[typeIndex],



            Body: &m.Code.Bodies[codeIndex],



        }



        m.FunctionIndexSpace = append(m.FunctionIndexSpace, fn)



    }
```

然后调用NewVM生成虚拟机对象，同时生成函数列表：

```text
vm.newFuncTable()
    for i, fn := range module.FunctionIndexSpace {



        if fn.IsHost() {



            vm.funcs[i] = goFunction{



                typ: fn.Host.Type(),



                val: fn.Host,



            }



            nNatives++



            continue



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
```

goFunction带了一个call的方法，在执行函数时就是调用这个call方法，对虚拟机堆栈进行一些处理。

## 引用导入测试

首先编写一个wasm文件，并引用其他的方法：

```text
int add()



{



  return 1;



}
```

它的wat文件为：

```text
(module



 (table 0 anyfunc)



 (memory $0 1)



 (export "memory" (memory $0))



 (export "add" (func $add))



 (func $add (; 0 ;) (result i32)



  (i32.const 1)



 )



)
```

编写一个add函数，并编译成env.wasm，然后编写main并编译成main.wasm：

```text
int main()



{



  return add();



}
```

它的wat文件为：

```text
(module



 (type $FUNCSIG$i (func (result i32)))



 (import "env" "add" (func $add (result i32)))



 (table 0 anyfunc)



 (memory $0 1)



 (export "memory" (memory $0))



 (export "main" (func $main))



 (func $main (; 1 ;) (result i32)



  (call $add)



 )



)
```

可以看到，这里import了env的add方法，wasm使用两级命名空间，这里表示要从env模块导入add方法，因此需要在env.wasm中实现add方法，下面执行调用：

```text
pct@Chandler:~/Downloads$ wasm-run -v main.wasm 



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 5



section.go:142: section type



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 11



section.go:149: section import



section.go:310: importing function



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 2



section.go:156: section function



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 4



section.go:163: section table



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 3



section.go:170: section memory



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 1



section.go:177: section global



section.go:441: 0 global entries



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 17



section.go:184: section export



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 10



section.go:205: section code



section.go:627: 1 function bodies



section.go:630: Reading function 0



section.go:688: bodySize: 4, localCount: 0



section.go:691: Read 3 bytes for function body



section.go:222: <nil>



section.go:96: Reading section ID



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 5



section.go:142: section type



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 2



section.go:156: section function



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 4



section.go:163: section table



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 3



section.go:170: section memory



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 1



section.go:177: section global



section.go:441: 0 global entries



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 16



section.go:184: section export



section.go:222: <nil>



section.go:96: Reading section ID



section.go:105: Reading payload length



section.go:125: Section payload length: 10



section.go:205: section code



section.go:627: 1 function bodies



section.go:630: Reading function 0



section.go:688: bodySize: 4, localCount: 0



section.go:691: Read 3 bytes for function body



section.go:222: <nil>



section.go:96: Reading section ID



index.go:77: There are 0 entries in the global index spaces.



module.go:142: There are 1 entries in the function index space.



index.go:77: There are 0 entries in the global index spaces.



module.go:142: There are 2 entries in the function index space.



memory() i32 => 1 (uint32)



main() i32 => 1 (uint32)
```

