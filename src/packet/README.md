# PACKET

---

## TODOs

- [x] 添加包体参数列表
- [x] 添加终端学号信息的记录
- [x] 添加解码端的校验检查功能
- [x] `encode`后字段分隔符的调整
- [ ] 完成本文档
- [ ] 数据字段中的标记字符转义*

---

## Structure

数据包文件结构如下

```Shell
.
├── README.md
├── include
│   ├── packet.h
│   └── utils.h
├── src
│   └── packet.cpp
└── test
    ├── packet_test.cpp
    └── test.sh
```

---

## APIs

详见`include/packet.h`和`include/utils.h`

### 编译

```Shell
g++ -c src/packet.cpp -o packet.o -Iinclude -lssl -lcrypto -w
```

### 基本流程

1. 创建发送数据包
   1. 调用构造函数`Packet(std::string info, PacketType type, PacketID id, ContentType content)`，其中`info`字段为学号信息，`type`为数据包类型，`id`为数据包ID，`content`为具体数据包的内容
   2. 数据包类型、具体数据包内容定义在`include/utils.h`的枚举类中
2. 添加数据包参数
   1. 调用`addArg(std::string arg)`添加业务所需的参数，注意遵循协议要求
3. 编码生成字符
   1. 调用`encode()`方法生成编码字符串，用于传输
4. 发送
5. 接收
6. 创建接收数据包
   1. 调用构造函数`Packet(std::string info)`，其中`info`为对方学号信息用于校验
7. 解码接收数据
   1. 调用`decode(std::string data)`方法对接收到的编码数据`data`进行解码
   2. 解码完成和校验成功后，返回值为`true`，否则为`false`

---

## IDEA

### 总体思路

采用`std::string`进行传输，所有数据编码成固定格式的字符串用于网络传输

在终端，要传输的数据都存储在数据包结构`Packet`中，通过编码(*Encode*)成字符串流进行传输，然后在接收端进行解码(*Decode*)将字符串解码成数据包结构，然后处理数据

本质来说，**数据包格式就是一个有关字符串编码与解码的工作**

### 数据包格式

#### 总体格式

| 字段 | 字节数量 | 含义 | 值或范围 | 备注 |
| :--: | :--: | :--: | :--: | :--: |
| type | 1 | 数据包类型 | 1~3 | 1: 请求，2：响应，3:指示 |
| content | 1 | 数据包内容 | 1~6 | 数据包内容 |
| id | 1 | 数据包ID | 0~255 | 重复滚动 |
| length | 2 | 数据包长度 | 0~65535 | 包体部分的长度 |
| args | N | 包体 | N个字节 | 包体部分，N=长度字段 |
| checksum | 16 | 校验部分 | 大写十六进制 | 对包体部分+学号后的MD5值 |
| 结束标记 | 1 | 结束标记 | `\n` | 用于区分数据包的结束 |

##### 数据包类型

- 请求数据包(`PacketType::REQUEST`)，客户端调用
- 响应数据包(`PacketType::RESPONSE`)，服务端调用
- 指示数据包(`PacketType::ASSIGNMENT`)，服务端调用

#### 请求数据包