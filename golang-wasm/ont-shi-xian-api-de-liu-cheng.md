# ONT实现API的流程

\[TOC\]

## 实现方式

本体是在原有VM的基础上添加了memory部分，实现wasm虚拟机的堆栈处理部分。 在调用合约时会执行下面函数：

```text
func (this *WasmVmService) Invoke() (interface{}, error) {


    stateMachine := NewWasmStateMachine()



    //register the "CallContract" function



    stateMachine.Register("ONT_CallContract", this.callContract)



    stateMachine.Register("ONT_MarshalNativeParams", this.marshalNativeParams)



    stateMachine.Register("ONT_MarshalNeoParams", this.marshalNeoParams)
```

这里注册了ONT的API，注册函数将函数执行和名称存在一个map里：

```text
func (i *WasmStateReader) Register(name string, handler func(*exec.ExecutionEngine) (bool, error)) bool {


    if _, ok := i.serviceMap[name]; ok {



        return false



    }



    i.serviceMap[name] = handler



    return true



}
```

等待执行时取出，实现体内部则是调用自己实现的memory：

```text
func (this *WasmVmService) runtimeLog(engine *exec.ExecutionEngine) (bool, error) {


    vm := engine.GetVM()



    envCall := vm.GetEnvCall()



    params := envCall.GetParams()



    if len(params) != 1 {



        return false, errors.NewErr("[RuntimeLog]parameter count error ")



    }



    item, err := vm.GetPointerMemory(params[0])



    if err != nil {



        return false, err



    }



    context := this.ContextRef.CurrentContext()



    txHash := this.Tx.Hash()



    event.PushSmartCodeEvent(txHash, 0, event.EVENT_LOG, &event.LogEventArgs{TxHash: txHash, ContractAddress: context.ContractAddress, Message: string(item)})



    vm.RestoreCtx()



    return true, nil



}
```

将获取的结果存入堆栈中，并返回指针。 在执行代码时，首先解析module部分，然后创建新的VM：

```text
func (e *ExecutionEngine) call(caller common.Address,


    code []byte,



    input []byte,



    actionName string,



    ver byte) (returnbytes []byte, er error) {



    if ver > 0 { //production contract version



        methodName := CONTRACT_METHOD_NAME //fix to "invoke"



        //1. read code



        bf := bytes.NewBuffer(code)



        //2. read module



        m, err := wasm.ReadModule(bf, importer)



        if err != nil {



            return nil, errors.NewErr("[Call]Verify wasm failed!" + err.Error())



        }



        //3. verify the module



        //already verified in step 2



        //4. check the export



        //every wasm should have at least 1 export



        if m.Export == nil {



            return nil, errors.NewErr("[Call]No export in wasm!")



        }



        vm, err := NewVM(m)
```

load部分就是将解析出的module加载到虚拟机，这个过程会调用vm.newFuncTable\(\)来生成函数列表：

```text
func (vm *VM) newFuncTable() {


    vm.funcTable[ops.I32Clz] = vm.i32Clz



    vm.funcTable[ops.I32Ctz] = vm.i32Ctz



    vm.funcTable[ops.I32Popcnt] = vm.i32Popcnt



    vm.funcTable[ops.I32Add] = vm.i32Add



    vm.funcTable[ops.I32Sub] = vm.i32Sub


    vm.funcTable[ops.Call] = vm.call


    vm.funcTable[ops.CallIndirect] = vm.callIndirect
```

vm.call里面就实现了我们上面定义的API：

```text
func (vm *VM) call() {


    index := vm.fetchUint32()



    vm.doCall(vm.compiledFuncs[index], int64(index))



}
```

```text
func (vm *VM) doCall(compiled compiledFunction, index int64) {


        vm.envCall.envPreCtx = prevCtxt



        v, ok := vm.Services[compiled.name]



        if ok {



            rtn, err := v(vm.Engine)
```

这里还有一个问题存疑，就是在解析wasm文件时，调用的ONT API是作为外部函数的，解析时会找不到，这个是通过编译器解决的还是其他方式还未看懂。

