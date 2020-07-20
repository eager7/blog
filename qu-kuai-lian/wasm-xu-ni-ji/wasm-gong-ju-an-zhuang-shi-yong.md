# Wasm工具安装使用

\[TOC\]

## 编译工具

Emscripten是一个编译C，C++到wasm文件的工具链，安装过程如下：

```text
git clone [https://github.com/juj/emsdk.git](https://github.com/juj/emsdk.git)
cd emsdk
./emsdk install --build=Release sdk-incoming-64bit binaryen-master-64bit
./emsdk activate --global --build=Release sdk-incoming-64bit binaryen-master-64bit
source ./emsdk_env.sh
```

参考链接： [https://developer.mozilla.org/zh-CN/docs/WebAssembly/C\_to\_wasm](https://developer.mozilla.org/zh-CN/docs/WebAssembly/C_to_wasm) 此时就可以使用emcc命令编译一个文件了：

```text
emcc hello.c -s WASM=1 -o hello.html
```

然后使用emrun来实现浏览器的显示：

```text
emrun --no_browser --port 8080 .
```

通过浏览器打开[http://localhost:8080/hello.html](http://localhost:8080/hello.html) 就可以看到输出信息了。

## 运行工具

可以运行wasm的工具有多个，C++ 版本有两个：

```text
[https://github.com/AndrewScheidecker/WAVM](https://github.com/AndrewScheidecker/WAVM)
[https://github.com/WebAssembly/wabt](https://github.com/WebAssembly/wabt)
```

前者功能更强大一些，后者是官方的解析工具。

```text
pct@Chandler:~/workspace/ABA/Codes/wasm$ ls WAVM/cmake-build-debug/bin/


Assemble Disassemble HashMapTest HashSetTest Test wavix wavm
```

wavm可以运行wasm文件。并且支持部分c库函数，如printf，即它在内部对env作了一部分处理。

```text
pct@Chandler:~/workspace/ABA/Codes/wasm$ ls wabt/bin/


spectest-interp wabt-unittests wasm2c wasm2wat wasm-interp wasm-objdump wasm-opcodecnt wasm-validate wast2json wat2wasm wat-desugar
```

wasm-interp工具可以执行wasm文件。不支持printf函数。 go版本的有一个：

```text
https://github.com/go-interpreter/wagon
```

它可以编译出两个工具：

```text
pct@Chandler:~/go/src/github.com/go-interpreter/wagon/cmd$ tree


.



├── wasm-dump



│   ├── hex.go



│   ├── main



│   └── main.go



└── wasm-run



    ├── basic.wasm



    ├── hello.wasm



    ├── main
```

wasm-run可以执行wasm文件，不支持c库函数，也不支持参数，需要自己添加。

## 调试工具

我们可以通过hexdump -C命令来查看wasm文件的字节内容，也可以通过wagon项目的wasm-dump命令来查看文件内容。 还可以通过wabt的工具来查看：

```text
wast2wasm simple.wat -v
```

