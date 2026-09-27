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

跨进进/跨机器调用需要自己解决的完整清单：

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
