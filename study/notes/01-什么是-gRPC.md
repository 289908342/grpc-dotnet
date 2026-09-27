# 01. 什么是 gRPC

> 日期：会话 01　|　状态：✅
> 关键词：IDL、代码生成、HTTP/2 多路复用、契约优先、Grpc.Core vs grpc-dotnet

## 一句话定义

> gRPC 是让你"像调用本地方法一样调用远程方法"的一整套**标准化**方案。

三个组成部分：

| 组成 | 是什么 | 在哪 |
|---|---|---|
| **IDL**（接口定义语言） | `.proto` 文件，定义服务方法 + 消息结构 | 不在 grpc-dotnet 仓库 |
| **代码生成** | 从 `.proto` 生成 client stub / server base | 生成代码由 `Grpc.Tools` 产出（在 grpc/grpc 仓库）；`src/dotnet-grpc` 是工具链 |
| **传输协议** | 基于 HTTP/2 的消息交换格式 | **本仓库主体**：`Grpc.Net.Client` / `Grpc.AspNetCore.Server` |

## 它真正解决的问题

常见误解：gRPC 的价值是"比 REST 快"。**这是次要的。**

跨进程 / 跨机器调用需要自己解决的完整清单：

- 消息编码（二进制布局、字节序）
- 序列化 / 反序列化
- 连接建立、断开、重连
- **请求与响应的配对**
- 错误传递（异常无法直接跨网络）
- 超时与取消
- **版本兼容**（对方加字段，老代码会不会崩）
- 流式传输
- 多语言支持

gRPC 把整张清单标准化了。核心价值是：

> **不需要重新发明这些，也不需要和别的团队争论二进制协议怎么设计。**

### 清单不是 `.proto` 单独承载的 —— 是四层分工

| 清单项 | 由谁解决 | 机制 |
|---|---|---|
| 消息编码 | **protobuf + `.proto`** | 字段号 + varint 二进制编码 |
| 序列化 / 反序列化 | **生成的代码** + `Marshaller` | 编译期生成的读写代码 |
| **版本兼容** | **`.proto`** | 字段号机制（第 03 节） |
| 多语言 | **`.proto`** + 各语言 codegen | 同一份 IDL |
| **请求-响应配对** | **HTTP/2** | stream ID |
| **流式传输** | **HTTP/2** | 一条 stream 承载多条消息 |
| **错误传递** | **gRPC 规范** | HTTP trailer 里的 `grpc-status` |
| **超时 / 取消** | **HTTP/2 + gRPC** | `grpc-timeout` 头 / `RST_STREAM` |
| **连接管理 / 重连** | **客户端通道层** | `GrpcChannel` + `SocketsHttpHandler` + Balancer |

`.proto` 只承载**契约层**那四件事。传输机制归 HTTP/2 和 gRPC 规范，连接管理归客户端通道层。

> **关键洞察：gRPC 之所以能这么"薄"，是因为它把最难的部分（配对、流、取消）外包给了 HTTP/2。**
> 它不是"自研 TCP 协议"，而是 **"自研消息格式 + 复用 HTTP/2 的传输语义"**。

### 「复用 HTTP/2」的三条代价

| 代价 | 说明 |
|---|---|
| TCP 层队头阻塞 | 丢一个包，这条连接上所有流一起卡 |
| **无法控制传输层** | ← **这就是 keepalive 做不到的根源**（传输层归 HttpClient / Kestrel 管） |
| 调试困难 | 二进制，不能 `curl`，需 `grpcurl` + 反射 |

**收益**：企业防火墙、nginx、Envoy、云负载均衡、API 网关**全都能直接用**。自研 TCP 协议过不了这些基础设施。

详见 `pitfalls.md` 第 1、2 条。

## 为什么基于 HTTP/2

| | HTTP/1.1 | HTTP/2 |
|---|---|---|
| 格式 | 文本 | **二进制分帧** |
| 并发 | 一请求一连接，或应用层队头阻塞 | **单连接多路复用** |
| 头部 | 每次明文重复 | **HPACK 压缩**（跨请求维护表） |
| 流式 | 别扭 | **原生双向** |
| 服务端推送 | 无 | 有（gRPC 基本不用） |

最关键的是**多路复用**：一条 TCP 连接上可跑多条互相独立的流。这是四种调用形态的技术基础。

### 代价：TCP 层队头阻塞仍在

**HTTP/2 解决的是 HTTP/1.1 的「应用层」队头阻塞，「TCP 层」的还在。**

一条 TCP 连接上所有 HTTP/2 流共享同一个 TCP 接收窗口 —— **丢一个包，这条连接上所有流一起卡住**等重传。

这正是 HTTP/3（QUIC over UDP）存在的理由：让每条流有独立的丢包恢复。

grpc-dotnet 有 HTTP/3 处理代码：`GrpcProtocolConstants.IsHttp3()`，以及 HTTP/3 专属的流重置码 `Http3ResetStreamCancel = 0x010c`（`Grpc.AspNetCore.Server/Internal/GrpcProtocolConstants.cs`）。

> 含义：多路复用省了连接，但**放大了单次丢包的影响范围**。

## 和 REST 的本质区别

不是性能，是这三点：

| | REST | gRPC |
|---|---|---|
| **契约** | 可选（OpenAPI 事后补） | **强制**（`.proto` 是唯一真相源） |
| **调用模型** | 资源 + 动词（GET/POST/PUT） | **方法调用**（就是函数） |
| **流式** | 别扭（SSE / WebSocket 都是外挂） | **一等公民**（4 种形态原生支持） |
| 浏览器 | 原生 | 需 gRPC-Web 转换 |

**最重要是「契约」**：REST 里接口定义与实现对不上是常见事故；gRPC 里 `.proto` 编译不过就是编译不过。

**代价也在契约上**：改 `.proto` 比改 JSON 字段麻烦得多 → 见第 03 节版本兼容规则。

## 两个 C# 实现（避坑必读）

| | Grpc.Core（老） | grpc-dotnet（新，本仓库） |
|---|---|---|
| 底层 | C 语言 native core 库 | **纯托管**，基于 ASP.NET Core / HttpClient |
| 客户端入口 | `new Channel("host", port)` | `GrpcChannel.ForAddress(...)` |
| 服务端入口 | `new Server{...}` | `WebApplication` + `MapGrpcService<T>()` |
| 状态 | 2021-05 起**维护模式**，未来废弃 | 官方推荐 |

**读资料的实用判据**：看到 `new Channel(...)` 或 `new Server(...)` → **老资料，直接关掉**。

参考：`doc/implementation_comparison.md`

## 四种调用形态（预览，详见第 04 节）

```
1. Unary                    客户端 ──请求──▶ 服务端
                            客户端 ◀──响应── 服务端

2. Server streaming         客户端 ──请求──▶ 服务端
                            客户端 ◀─流式响应─ 服务端

3. Client streaming         客户端 ──流式请求─▶ 服务端
                            客户端 ◀──响应── 服务端

4. Bidirectional streaming  客户端 ◀──流式──▶ 服务端   (两个方向互相独立)
```

## 调用形态怎么选（不是按"灵活性"选）

双向流**是**最通用的形态（unary 是它的退化情况：一问一答然后关流），但**不能都用它**。六个理由：

| # | 理由 | 说明 |
|---|---|---|
| 1 | **请求-响应配对自己造** | 用双向流做请求响应，必须自己发明 correlation ID —— 而这正是 unary 免费给的（HTTP/2 stream ID）。**等于把 gRPC 帮你解决的问题捡回来自己做一遍** |
| 2 | 错误隔离 | 一个错误**终止整条流**。一条流承载 100 个逻辑请求，一个失败全断 |
| 3 | **重试基本失效** | retry 对四种调用都会装配，但收到响应头即"committed"。长流响应头立刻到 → **实际只有一次机会** |
| 4 | 负载均衡会「粘住」 | 负载均衡按**每次调用**选后端；流一旦建立就固定在一台上了 |
| 5 | 流控耦合 | 两个方向共享同一 stream 流控窗口 + 同一 TCP 连接，一边堵住牵连整个流 |
| 6 | 容量模型不同 | 1000 条长流 ≠ 1000 QPS unary（连接、内存、每流状态） |

### 第 3 点的源码依据

- `HttpClientCallInvoker.CreateRootGrpcCall()` 装配 retry 时**不区分** `method.Type` —— 四种调用都有 retry 包装
- 但 `src/Grpc.Net.Client/Internal/Retry/CommitReason.cs` 第一个枚举值是 `ResponseHeadersReceived` —— **收到响应头即 committed，重试窗口关闭**

| 调用形态 | 响应头何时到 | 重试是否有效 |
|---|---|---|
| Unary | 通常在最后 | ✅ 失败可重试 |
| 长生命周期双向流 | 几乎立刻 | ❌ 实际只有一次机会 |

### 选择判据

| 语义 | 用什么 |
|---|---|
| 一问一答（哪怕每秒 10 次） | **Unary** |
| 服务端持续推 | Server streaming |
| 客户端持续传 | Client streaming |
| 真正的双向对话 | Bidirectional |

> 只有当**两个方向的生命周期绑定**（一方结束另一方才结束）时，才用双向流。
> 如果是"你问我答、反复进行"，unary 更好。

## 三个检验问题

1. gRPC 选 HTTP/2 而不是自研 TCP 协议，除了性能还有什么原因？这个选择带来了什么代价？
2. 周期推送：用 Unary 每秒调 10 次 vs. 一条 server streaming 长连接，本质差别在哪？（连接管理、头部开销、流控、错误隔离）
3. "像调用本地方法一样调用远程方法" —— 在软实时场景下，这句话的哪部分是幻觉？

## 涉及的文件

| 文件 | 说明 |
|---|---|
| `doc/implementation_comparison.md` | 两个 C# 实现的完整对比（**本仓库最佳入口文档之一**） |
| `src/Grpc.AspNetCore.Server/Internal/GrpcProtocolConstants.cs` | HTTP/2 / HTTP/3 协议常量与判断 |
| `src/Grpc.Net.Client/GrpcChannel.cs` | 客户端入口（第 17 节详读） |
| `src/Grpc.AspNetCore.Server/GrpcEndpointRouteBuilderExtensions.cs` | 服务端入口 `MapGrpcService` |
