# Protocon

Protocon 是一个基于 TCP 的轻量协议。用于 ies-con-connector 后端与终端设备的命令。

后端为服务端，监听指定端口；终端设备为客户端，各个设备需要主动连接到服务端的指定端口。

## 命令执行流程

本协议在命令发起中不区分服务端与客户端，命令可以由任意一方发起。称发起命令的一方为`主动方`，另一方为`被动方`。
服务端和客户端建立 TCP 连接后，双方即可发起命令。具体的流程如下：

1. `主动方`希望`被动方`执行一次特定操作，于是将操作所需的信息编码成一段数据报，该段数据报称为请求（Request）
2. `主动方`通过 TCP 协议向`被动方`发送该请求
3. `被动方`接收到该请求并相应地执行操作
4. `被动方`将这些操作得到的结果编码成另一段数据报，该段数据报成为响应（Response）
5. `被动方`通过 TCP 协议向`主动方`发送该响应

其中，一次命令中的请求与响应都具有相同的命令标识符（Command Identifier）。
上层应用可通过该标识符来确定请求与响应的对应关系，以此实现请求的并发处理。

## Request

请求由以下字段依序组成

| Name               | Type             | Bytes |
| ------------------ | ---------------- | ----- |
| Command Identifier | unsigned integer | 2     |
| Gateway Identifier | unsigned integer | 8     |
| Client Identifier  | unsigned integer | 8     |
| Time               | unsigned integer | 8     |
| API Version        | unsigned integer | 2     |
| Type               | unsigned integer | 2     |
| Length             | unsigned integer | 4     |
| Data               | UTF-8 string     | -     |

### Command Identifier

该请求的命令标识符，用于与 Response 相对应。
标识符由`主动方`生成。
本协议规定生成的标识符应当递增，即任意一次请求的标识符皆为其前一次的标识符加一，上层应用如果并发处理请求则可使用原子量来实现。如果标识符自增后溢出，则重新从零开始。

有两种特殊情况：服务端发起的请求与客户端发起的请求的命令标识符可以重复，上层应用需要区分处理自身发送的请求与接收到的请求的标识符；不同客户端的命令标识符也可以重复，服务端需要区分处理不同客户端的标识符。

**注意**：如果多台设备经由同一个网关与平台交互，命令标识符应当由网关生成。

### Gateway Identifier

网关标识符，即为网关设备的客户端标识符，具体可参见后文。
有效的网关标识符为**正数**，如果为零则为无网关，即直连设备。

直连设备与平台交互时，网关标识符应当为零。
非直连设备与平台交互时，网关标识符为其所属的网关的客户端标识符。
网关设备可以认为是直连设备。

### Client Identifier

客户端标识符，可以缩写为 Client ID。
指明命令中客户端的标识符。
有效的标识符为**正数**，如果为零则标识符无效，但请求不一定无效。
对于客户端标识符为零的请求，上层应用通常应当施加某种限制。

对于客户端而言，该标识符为零表示该请求为匿名请求，向服务端发送的匿名请求通常为客户端注册等。

对于服务端而言，向客户端发送的请求中该标识符必须与客户端的真实标识符相同，客户端在接收到请求后也应当验证请求中的标识符是否与自身标识符一致。

### Time

请求发送的时间戳。其值为从格林威治时间 1970 年 01 月 01 日 00 时 00 分 00 秒起至请求发送时的总秒数，即 Unix timestamp。

### API Version

接口的版本号。
一套接口指定了所有可用的命令类型（Type），并规定各个 Type 所对应的操作与响应数据格式。

接口版本更新时，命令类型可能会发生变化，同一类型所规定的操作与响应也可能发送变化。
因此`被动方`在接收到请求后应当验证请求中接口版本号是否与自身的接口版本号一致，避免执行错误的操作导致崩溃。

### Type

指定命令的类型。

`主动方`通过指定命令类型来指定`被动方`需要执行的操作，并在 Data 中向`被动方`提供所需的数据。
`被动方`可根据命令类型来解析 Data 并从中获取本次命令所需的数据，并执行该类型所规定的操作。

### Length

指定 Data 的长度，即字节数量。
上层应用应当从 TCP 流中读取 Length 个字节，并将这些字节解码为 Data。

### Data

一段 JSON 数据，Root 必须是 Object 而不能是 Array。
编码为 UTF-8。
Object 中的键值对根据 Type 不同而不同。
长度由 Length 指定。

## Response

响应由以下字段依序组成

| Name               | Type             | Bytes |
| ------------------ | ---------------- | ----- |
| Command Identifier | unsigned integer | 2     |
| Time               | unsigned integer | 8     |
| Status             | unsigned integer | 1     |
| Length             | unsigned integer | 4     |
| Data               | UTF-8 string     | -     |

### Command Identifier

命令标识符，应当与响应所对应请求的命令标识符保持一致。

### Time

响应发送的时间戳，为 Unix timestamp。

### Status

命令的状态，表示命令是否成功，以及失败原因。

如果成功则为零，失败则为非零的错误码。
对于所有命令，错误码的取值必须统一。

**注意**：成功和失败可能会导致 Data 中的键值对产生变化，比如失败时 Data 可能包含错误信息。

### Length

指定 Data 的长度。

### Data

长度由 Length 指定。Data 中的键值对（即响应所要返回的信息）由命令类型指定。
此外，Data 还会受命令状态影响，例如：如果命令状态为成功，则 Data 中的信息可以为请求所要求的操作结束后的的设备状态；如果命令失败，则 Data 中的信息通常为错误产生的原因以及与错误相关的其他信息。
