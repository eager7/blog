# 解析wasm二进制文件

\[TOC\]

## 文件内容

hello的源文件如下：

```text
int main()


{



  Println("hello world\n");



  return 0;



}
```

wat文件如下：

```text
(module


 (type $FUNCSIG$i (func (result i32)))



 (type $FUNCSIG$ii (func (param i32) (result i32)))



 (import "env" "Println" (func $Println (param i32) (result i32)))



 (table 0 anyfunc)



 (memory $0 1)



 (data (i32.const 16) "hello world\n\00")



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

hello的wasm文件如下：

```text
var wasmCode = new Uint8Array([0,97,115,109,1,0,0,0,1,138,128,128,128,0,2,96,0,1,127,96,1,127,1,127,2,143,128,128,128,0,1,3,101,110,118,7,80,114,105,110,116,108,110,0,1,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,1,10,143,128,128,128,0,1,137,128,128,128,0,0,65,16,16,0,26,65,0,11,11,147,128,128,128,0,1,0,65,16,11,13,104,101,108,108,111,32,119,111,114,108,100,10,0]);
```

16进制读取结果为：

```text
pct@Chandler:~/go/src/github.com/go-interpreter/wagon/cmd/wasm-run$ hexdump -C hello.wasm
00000000  00 61 73 6d 01 00 00 00  01 8a 80 80 80 00 02 60  |.asm...........`|
00000010  00 01 7f 60 01 7f 01 7f  02 8f 80 80 80 00 01 03  |...`............|
00000020  65 6e 76 07 50 72 69 6e  74 6c 6e 00 01 03 82 80  |env.Println.....|
00000030  80 80 00 01 00 04 84 80  80 80 00 01 70 00 00 05  |............p...|
00000040  83 80 80 80 00 01 00 01  06 81 80 80 80 00 00 07  |................|
00000050  91 80 80 80 00 02 06 6d  65 6d 6f 72 79 02 00 04  |.......memory...|
00000060  6d 61 69 6e 00 01 0a 8f  80 80 80 00 01 89 80 80  |main............|
00000070  80 00 00 41 10 10 00 1a  41 00 0b 0b 92 80 80 80  |...A....A.......|
00000080  00 01 00 41 10 0b 0c 68  65 6c 6c 6f 20 77 6f 72  |...A...hello wor|
00000090  6c 64 00                                          |ld.|
00000093
```

wasm-dump解析结果如下：

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

## 二进制解析

### 魔数和版本号

下面表格是wasm规格书中定义的code：  首先因为大小端问题，这里的存放和源文件是不同的，wasm的魔数为0x6d736100，版本号为0x01，所以前四个为魔数00 61 73 6d，后四个是版本号01 00 00 00，大小端颠倒了。

### Type Section

接下来的数字是01 8a 80 80 80 00，01是type，表示函数签名的Type Section，后面的8a 80 80 80 00是LEB128编码方式表示长度为0a。   也就是：

```text
02 60  00 01 7f 60 01 7f 01 7f
```

02表示entries数量为2，60表示是个函数的签名，00表示参数为0，那么也就省略了param\_types，01表示有一个返回值，7f表示返回值为i32类型，这个是main的函数声明； 同理，60 01 7f 01 7f表示一个带有i32类型并返回i32类型的函数声明，也就是Println的函数声明。

### Import Section

再往下是02，表示Import Section，    长度为15， 02 8f 80 80 80 00，内容为：

```text
01 03 65 6e 76 07 50 72  69 6e 74 6c 6e 00 01
```

01则只有一个引用，module长度为3，名称为env，field长度为7，名称为Println，最后的kind表示引入的类型，00表示引入的是个函数，01则表示函数的索引。

### Function Section

然后是03，Function Section，  长度为82 80 80 80 00，内容为：

```text
01 00
```

01表示签名索引数量为1，00表示在type区中的索引值为0，即main函数的签名。

### Table Section

接下来是04， Table Section，     长度为84 80 80 80 00 ，内容为：

```text
01 70 00 00
```

01表示数量为1，类型为70，目前只能是70，表示anyfunc。

### Memory Section

05-Memory Section，   长度为83 80 80 80 00，内容为：

```text
01 00 01
```

### Global Section

06-Global Section，  长度为81 80 80 80 00，内容为：

```text
00
```

### Export Section

07-Export Section，  长度为91 80 80 80 00，内容为：

```text
02 06 6d  65 6d 6f 72 79 02 00 04 6d 61 69 6e 00 01
```

02表示export数量为2,06表示第一个entry长度，名称为memory，02表示类型为memory类型，index为00，表示在对应索引空间的索引值； 04表示第二个entry长度为4，名称为main，类型为00，function类型，索引为01

### Code Section

下面就是Code Section了，中间的Start Section， Element Section在这个模块中没有体现：   0a-Code Section，长度为8f 80 80 80 00，内容为：

```text
01 89 80 80 80 00 00 41  10 10 00 1a 41 00 0b
```

01表示数量为1，89 80 80 80 00表示size为9,00是locals数量，为0则locals就没有，41 10 10 00 1a 41 00是code，0b是end。

### Data Section

0b-Data Section，  长度为92 80 80 80 00，内容为：

```text
 01 00 41 10 0b 0c 68 65  6c 6c 6f 20 77 6f 72 6c  64 00
```

01表示数量为1,00表示在线性内存中的索引为0，41表示iConst.32， 10表示call操作码， 0b表示offset，0c表示参数长度为12，68 65 6c 6c 6f 20 77 6f 72 6c 64 00表示我们打印的内容为hello world 至此，文件解析完毕。

