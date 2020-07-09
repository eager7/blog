# Wagon实现log函数的第二种方法

\[TOC\]

前面我们介绍了一种实现方法，是自己构造function的方式，第二种其实也是一样的方式，通过构造env模块跳过外部引用，但是更符合项目的设计规范，wagon在设计之初就预留了host接口，作为本地函数实现接口，我们可以通过这个接口实现env模块的导入。

## host接口

在创建新的虚拟机时，下面代码实现host‘的导入：

```text
func NewVM(module *wasm.Module) (*VM, error) {
   var vm VM
   vm.funcs = make([]function, len(module.FunctionIndexSpace))
   vm.globals = make([]uint64, len(module.GlobalIndexSpace))
   vm.newFuncTable()
   vm.module = module
   nNatives := 0   for i, fn := range module.FunctionIndexSpace {      if fn.IsHost() {
         vm.funcs[i] = goFunction{
            typ: fn.Host.Type(),            val: fn.Host,         }
         nNatives++
         continue      }
```

如果函数是host类型，则直接添加到函数索引空间并且不再继续往下执行，那么就需要在解析外部引用时将函数设置为host类型。

## 外部导入部分

这里有两种方式，一种就是和我们在前面介绍的方法相同，在resolveImports函数中通过外部导入的包名引入函数，还有一种方式是通过main中的importer函数实现导入，两种方式都可以实现，前者是侵入式的更改，不过更简单，后者是非侵入式的更改，如果是引用wagon代码，可以考虑使用第二种。

### resolveImports中解决引用的方法

```text
func (module *Module) resolveImports(resolve ResolveFunc) error {
   if module.Import == nil {
      return nil
   }
   modules := make(map[string]*Module)
   native  := NewNativeFun()
   var funcs uint32   for _, importEntry := range module.Import.Entries {
      //TODO import/global/memory      if importEntry.ModuleName == "env" {
         switch importEntry.Kind {
            case ExternalFunction:
            funcType := module.Types.Entries[importEntry.Type.(FuncImport).Type]
            host := native.GetValue(importEntry.FieldName)
            fn := &Function{Sig: &FunctionSig{ParamTypes: funcType.ParamTypes, ReturnTypes: funcType.ReturnTypes}, Body: &FunctionBody{}, Host:host}
            module.FunctionIndexSpace = append(module.FunctionIndexSpace, *fn)
            module.Code.Bodies = append(module.Code.Bodies, *fn.Body)
            module.imports.Funcs = append(module.imports.Funcs, funcs)
            funcs++
            }
      }
```

当外部模块是env时，我们定义了一个新的Function对象，并且定义了它的Host字段，现在它就成为了一个host的方法，然后将这个方法添加到函数索引空间即可。这个host是我们自行定义的函数的类型反射：reflect.ValueOf\(i\)，go语言可以通过反射知道这个函数的类型，从而调用到这个函数。

### importer方式解决引用

```text
func mImporter(name string) (*wasm.Module, error) {
   fmt.Println("import name:", name)
   m := wasm.NewModule()
   m.Types = &wasm.SectionTypes{      Entries: []wasm.FunctionSig{
         {
            Form:        0,            ParamTypes:  []wasm.ValueType{wasm.ValueTypeI32},            ReturnTypes: []wasm.ValueType{wasm.ValueTypeI32},         },      },   }
   m.FunctionIndexSpace = []wasm.Function{
      {
         Sig:  &m.Types.Entries[0],         Host: reflect.ValueOf(add3),         Body: &wasm.FunctionBody{},      },   }
   m.Export = &wasm.SectionExports{
      Entries: map[string]wasm.ExportEntry{
         "add3": {
            FieldStr: "add3",            Kind:     wasm.ExternalFunction,            Index:    0,         },   }
   return m, nil
}
```

其实解决思路是一样的，也是通过Host字段实现本地函数，然后通过反射注册函数类型，这种方式相对来说更繁琐一点。

