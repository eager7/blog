# protobuf简介

\[TOC\]

## \[protobuf简介\]\([https://developers.google.com/protocol-buffers/docs/proto3\#top\_of\_page](https://developers.google.com/protocol-buffers/docs/proto3#top_of_page)\)

protobuf是Google开发的一款跨语言，跨平台的序列化协议。 目前开源的protobuf共有两种协议规则，proto2和proto3，两者有部分数据不兼容，因此最好不要混用。

## protobuf语法

先看一个例子：

```text
syntax = "proto3";

package protocol;
import "core/abc_types.proto";

option go_package = "github.com/BlockABC/wallet_permission_server/grpc/core";


enum AbcPermFlag {
    Null        = 0;//权限清空    Valid       = 1;//权限有效    Forbidden   = 2;//权限被禁}
/*** 代币和用户公用此权限单位，但是用户不区分平台，代币区分平台*/message AbcPermission {
    AbcPermFlag flag    = 1[json_name="flag"];          //权限标识,0表示清空,1表示有效,2表示封禁    string platform     = 3[json_name="platform"];      //平台信息，当设置平台时，允许用逗号区分多个平台进行批量设置}
```

`syntax`表示protobuf版本，这里使用第三版，官方现在也推荐使用proto3。 `package`定义了protobuf的包空间，避免命名冲突问题，这个通常会配合下面的go\_package使用，将相同类型数据导入到不同包中。 `option`是自定义选项，这里指定了go的包名，也可以指定其他语言的包名，具体的参数设置可以参考：\[Options\]\([https://developers.google.com/protocol-buffers/docs/proto3\#options](https://developers.google.com/protocol-buffers/docs/proto3#options)\) enum是枚举类型，只能是数字格式，且标签数值必须从0开始。 message是结构体类型，内部数据标签不能为0。 注意，同一消息体内，数字标签不能重复。

## protobuf支持类型

可以从官网查看[支持类型](https://developers.google.com/protocol-buffers/docs/proto3#scalar) 在proto3中还增加了map，any，oneof等类型。 此外还可以用repeated加在类型前面构建数组。

## protobuf编码

不同于json的字符串编码格式，protobuf是一种二进制编码方式。并且编码依赖设置的数字tag，也就是编码并不关心字段名，只关心tag和值，即便字段名变化依旧可以解析。 同理，如果变更了tag，则可能会导致解析的向前兼容问题，所以一旦tag固定后最好不要变动。 需要特别提醒的是标签 1–15 标识的字段编码仅占用 1 个字节（包括字段类型和标识标签），更多详细的信息请参考 [ProtoBuf 编码](https://developers.google.com/protocol-buffers/docs/encoding.html) 。 数值标签 16–2047 标识的字段编码占用 2 个字节。因此，你应该将标签 1–15 留给那些在你的消息类型中使用频率高的字段。记得预留一些空间（标签 1–15）给将来可能添加的高频率字段。

## 代码生成

代码生成需要安装[protoc](https://github.com/protocolbuffers/protobuf) golang还需安装\[proto-gen-go\]\([https://github.com/golang/protobuf](https://github.com/golang/protobuf)\)。 可以用下面命令直接安装：

```text
export GOPROXY=https://goproxy.io && GO111MODULE=on go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
export GOPROXY=https://goproxy.io && GO111MODULE=on go get -u github.com/micro/protobuf/{proto,protoc-gen-go}
export GOPROXY=https://goproxy.io && GO111MODULE=on go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger
```

编译命令为：

```text
protoc -I=. -I=/Users/plainchant/go/src --go_out=plugins=grpc,paths=source_relative:. hello.proto world.proto
```

`I`指向头文件地址，`go_out`表示输出go文件，`paths`表示使用相对路径，不会根据包内package生成绝对路径。

## 三方工具

gogoproto-gen-go是一款第三方的生成go代码的工具，它可以定制很多生成规则，生成的代码更美观，但是它是针对proto2开发，对proto3兼容性并不好，不建议使用。

