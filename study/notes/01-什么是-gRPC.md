# 01. 什么是 gRPC

> 会话 01　|　✅ 已完成
> 阅读顺序：1 是什么 → 2 为什么 → 3 怎么实现 → 4/5 优缺点 → 6 实践问答

---

## 1. 这是什么

**gRPC 是一套现成的方案，让一个程序能调用另一个程序里的函数 —— 用起来就像调用自己代码里的函数一样。**

举个你熟悉的场景：应用程序想读取内核里的设备状态。没有 gRPC 的话，你得自己规定"怎么把设备名发过去、怎么把结果收回来、出错怎么表示"。

有了 gRPC，你只需要写一份**接口说明书**，剩下的代码它帮你生成。

这份说明书叫 **`.proto` 文件**。

### gRPC 由三个部分组成

| 部分 | 干什么 | 类比 |
|---|---|---|
| **接口定义**（`.proto` 文件） | 写清楚有哪些方法、参数和返回值长什么样 | 像 Modbus 的寄存器地址表，但描述的是"方法 + 结构化数据" |
| **代码生成** | 工具读取 `.proto`，自动生成可调用的 C# 代码 | 像 ORM 根据数据库 schema 生成实体类 |
| **传输** | 真正运行时，按 HTTP/2 的规则把数据发过去 | 像 TCP 之上的一层约定 |

### 一个最小的例子

**你手写的 `.proto`：**

```proto
service DeviceService {
  rpc GetTemperature (DeviceRequest) returns (TemperatureReply);
}

message DeviceRequest {
  string device_id = 1;
}

message TemperatureReply {
  double celsius = 1;
}
```

**自动生成的 C# 代码（你不用写）：**

```csharp
var client = new DeviceServiceClient(channel);
var reply = await client.GetTemperatureAsync(new DeviceRequest { DeviceId = "PLC-01" });
Console.WriteLine(reply.Celsius);
```

注意最后一行 —— `GetTemperatureAsync(...)` 看起来完全像一个本地方法。

**这就是 gRPC 的核心卖点：像调用本地方法一样调用远程方法。**

---

## 2. 为什么要这样设计

### 要解决的问题

两个程序要通信，如果没有任何现成方案，你得**自己解决这十件事**：

| # | 要解决的问题 |
|---|---|
| 1 | 消息怎么变成字节（编码） |
| 2 | 字节怎么变回消息（序列化 / 反序列化） |
| 3 | 连接断了怎么办、什么时候重连 |
| 4 | **哪个响应对应哪个请求**（同时有多个请求在飞的时候） |
| 5 | 错误怎么传过去（异常没法跨网络） |
| 6 | 超时怎么设、怎么取消 |
| 7 | **对方加了字段，我这边的老代码会不会崩** |
| 8 | 怎么持续不断地传数据（流式） |
| 9 | 怎么支持多种编程语言 |
| 10 | 怎么调试 |

**每一项都不算特别难，但十项加起来就是几个月的工作量，而且每个团队造出来的都不一样。**

后果是：A 团队和 B 团队的协议互相不兼容，出了问题没人知道怎么查，新人接手要从头读一遍自定义协议文档。

**gRPC 的答案：把这十件事全部标准化。**

### 关键设计决策：为什么选 HTTP/2

gRPC 面临的第一个选择是：**自己设计一套传输协议，还是复用现成的？**

**它选择了复用 HTTP/2。** 三个理由：

| 理由 | 说明 |
|---|---|
| **省事** | 请求响应配对、流式传输、超时取消 —— HTTP/2 全都现成，不用自己造 |
| **能过基础设施** | 企业防火墙、nginx、Envoy、云负载均衡都认识 HTTP/2。**自研 TCP 协议会被直接拦掉** |
| **工具生态** | 抓包、代理、监控都有现成方案 |

这个决定**非常关键** —— 它带来了 gRPC 最重要的两个特征，一好一坏：

- ✅ **好**：能无缝融入现有网络基础设施（见第 4 节）
- ❌ **坏**：传输层不归 gRPC 管，导致有些能力永远做不到（见第 5 节）

---

## 3. 现在是怎么实现的

### 3.1 C# 有两个实现，千万别搞混

| | **Grpc.Core**（老） | **grpc-dotnet**（新，就是这个仓库） |
|---|---|---|
| 底层 | C 语言写的 native 库 | **纯托管代码**，构建在 ASP.NET Core / HttpClient 上 |
| 客户端入口 | `new Channel("host", port)` | `GrpcChannel.ForAddress(...)` |
| 服务端入口 | `new Server { ... }` | `WebApplication` + `MapGrpcService<T>()` |
| 状态 | 2021-05 起**维护模式**，未来会废弃 | 官方推荐 |

> **实用判据**：看到一个教程里写 `new Channel(...)` 或 `new Server(...)`，
> 就说明它是老资料，直接关掉换一篇。网上现在还有大量这种过时教程。

### 3.2 分层：那十件事分别由谁负责

这是理解 gRPC 最重要的一张表 —— **`.proto` 只负责其中一部分**：

| 要解决的问题 | 由谁解决 | 具体机制 |
|---|---|---|
| 1. 消息编码 | **`.proto` + protobuf** | 字段号 + varint 二进制编码（第 02 节） |
| 2. 序列化 / 反序列化 | **生成的代码** | 编译期生成的读写代码 |
| 3. 连接管理 / 重连 | **客户端通道层** | `GrpcChannel` + `SocketsHttpHandler` |
| 4. 请求-响应配对 | **HTTP/2** | stream ID |
| 5. 错误传递 | **gRPC 规范** | trailer 里的 `grpc-status` |
| 6. 超时 / 取消 | **HTTP/2 + gRPC** | `grpc-timeout` 头 / `RST_STREAM` |
| 7. 版本兼容 | **`.proto`** | 字段号机制（第 03 节） |
| 8. 流式传输 | **HTTP/2** | 一条 stream 承载多条消息 |
| 9. 多语言 | **`.proto`** + 各语言代码生成 | 同一份 IDL |
| 10. 调试 | **工具生态** | `grpcurl` + 服务反射 |

**结论：`.proto` 只承担第 1、7、9 项（契约层）。传输机制归 HTTP/2，连接管理归通道层。**

> 所以 gRPC 能做得这么"薄"，是因为**它把最难的部分（配对、流、取消）外包给了 HTTP/2**。
> 它本质上不是"自研协议"，而是 **"自研消息格式 + 复用 HTTP/2 的传输语义"**。

### 3.3 四种调用形态

```
1. Unary（一问一答）
     客户端 ──请求──▶ 服务端
     客户端 ◀──响应── 服务端

2. Server streaming（服务端推流）
     客户端 ──请求──▶ 服务端
     客户端 ◀─流式响应─ 服务端

3. Client streaming（客户端推流）
     客户端 ──流式请求─▶ 服务端
     客户端 ◀──响应── 服务端

4. Bidirectional streaming（双向流）
     客户端 ◀──流式──▶ 服务端   （两个方向互相独立）
```

**第 04 节会逐个展开。** 现在只需要知道它们都存在，且都是原生支持的（不像 REST 要靠 SSE / WebSocket 外挂）。

### 3.4 调用形态怎么选（先给结论，理由在第 04 节）

| 你的语义 | 该用哪个 |
|---|---|
| 一问一答（**哪怕每秒 10 次**） | Unary |
| 服务端持续推数据 | Server streaming |
| 客户端持续传数据 | Client streaming |
| 两个方向真正双向对话 | Bidirectional |

**关键判据：只有当「两个方向的生命周期绑定」时（一方结束另一方才结束），才用双向流。**

不要因为"双向流最灵活"就什么都用它 —— 具体代价见第 5 节。

---

## 4. 优点

| # | 优点 | 说明 |
|---|---|---|
| 1 | **契约强制** | 接口和实现对不上就**编译不过**。REST 里文档过期、字段对不上是常见事故 |
| 2 | **跨语言** | 同一份 `.proto`，C# / Java / Python / Go 都能用 |
| 3 | **性能** | 二进制比 JSON 更小、解析更快。**但这是次要优点**，别把它当核心理由 |
| 4 | **流式原生支持** | 四种调用形态都是一等公民，不用外挂 |
| 5 | **能过企业基础设施** | 因为它就是 HTTP/2，防火墙 / nginx / Envoy 全都能直接用 |
| 6 | **工具生态现成** | `grpcurl`、服务反射、健康检查、OpenTelemetry 都有官方支持 |

---

## 5. 缺点 / 代价

| # | 代价 | 说明 |
|---|---|---|
| 1 | **改接口比较麻烦** | 改 `.proto` 比改 JSON 字段麻烦。需要理解版本兼容规则（第 03 节） |
| 2 | **不能直接 `curl`** | 二进制格式，人眼不可读。调试要用 `grpcurl` + 服务反射 |
| 3 | **浏览器不能直连** | 浏览器发不了 HTTP/2 的 gRPC 请求，需要 gRPC-Web 转换 |
| 4 | **TCP 层队头阻塞** | 一条 TCP 上所有流共享窗口，**丢一个包，所有流一起卡** |
| 5 | **传输层不可控** ⚠️ | 见下方详细说明 |
| 6 | **概念较多** | proto、流式、deadline、metadata、status code 等 |

### 关于第 5 条，需要展开说 —— 这是个真实的坑

**因为传输层归 `HttpClient` 和 Kestrel 管，gRPC 没有权限去控制它。**

直接的后果是：**grpc-dotnet 做不到 keepalive（保活）**。

`doc/implementation_comparison.md` 里明确写了（第 151-165 行）：gRPC for .NET **不支持 keepalive，也不支持 connection backoff**，原因是 `HttpClient` 和 Kestrel 都不提供这些能力。

**这意味着什么？**

长连接在 NAT、工业防火墙下会被**静默掐断** —— 没有任何通知，两端都以为连接还在。而 gRPC 层不会自动重连或保活。

> 这就是"复用 HTTP/2"这个选择的代价。它和第 4 节的第 5 条优点是**同一件事的两面**。

应对方式：**在应用层自己做心跳**（比如周期性的 Ping 调用或哨兵消息）。详见 `pitfalls.md` 第 1、2 条。

---

## 6. 工程实践问答

### Q1：我看到一个教程写 `new Channel("localhost", 5000)`，这个能用吗？

**那是老实现 Grpc.Core 的写法。** 它从 2021 年 5 月起进入维护模式，未来会废弃。

这个仓库（grpc-dotnet）的写法是 `GrpcChannel.ForAddress("https://localhost:5000")`。

**判据**：看到 `new Channel(...)` 或 `new Server(...)` → 资料过时，换一篇。

---

### Q2：我们每秒调用 10 次，这算频繁吗？会有性能问题吗？

**对 gRPC 来说完全不算频繁。** 单条连接上跑几百上千 QPS 都是常态。

但要注意的是**别每次都新建 channel**。`GrpcChannel` 是设计成**长期复用**的（内部管理连接池、负载均衡等）。频繁创建会不断重建连接，这才是真正的问题。

```csharp
// ❌ 错误：每次调用都建 channel
var reply = await new DeviceServiceClient(GrpcChannel.ForAddress(url)).GetTemperatureAsync(req);

// ✅ 正确：channel 建一次，长期复用
var channel = GrpcChannel.ForAddress(url);   // 应用启动时建
var client = new DeviceServiceClient(channel);
```

---

### Q3：gRPC 是 REST 的替代品吗？我该把现有的 REST 接口都换掉吗？

**不是"替代"，是两条不同的路线。**

- **REST**：面向**资源**。`GET /devices/42/temperature` —— 你在操作一个资源
- **gRPC**：面向**方法**。`DeviceService.GetTemperature(...)` —— 你在调用一个函数

gRPC 表面上也在用 HTTP POST，但它**故意违反了 REST 的资源导向**，把 URL 当方法名用。

**建议**：
- 内部服务之间、对性能有要求、需要流式 → gRPC
- 对外开放、需要浏览器直接访问、需要人眼可读 → 保留 REST

两者可以共存（gRPC 甚至有 JSON transcoding，能自动把 gRPC 服务暴露成 REST 接口）。

---

### Q4：什么时候**不该**用 gRPC？

| 场景 | 原因 |
|---|---|
| 浏览器要直接调用 | 浏览器发不了 HTTP/2 gRPC 请求，得加 gRPC-Web 转换层 |
| 需要人能直接看懂、能 `curl` 调试 | gRPC 是二进制，人眼不可读 |
| 极简的一次性脚本、对外公开 API | 引入 proto + 代码生成的成本不划算 |
| 需要 HTTP 缓存语义（CDN、ETag） | gRPC 用 POST，天然不吃这些缓存 |

---

### Q5：都说 gRPC 比 REST 快，到底快多少？

**通常更快，但常常没有想象中那么多，而且这不是它最重要的价值。**

它快的原因是：二进制编码比 JSON 小、解析不用字符串匹配、HTTP/2 头部压缩。

**但有几个前提容易被忽略：**
- 如果不复用连接（每次新建），HTTP/2 多路复用的优势就没了
- 如果消息很小、网络很快，差距会缩小
- 真正拉开差距的是**高频小消息**和**流式场景**

**更重要的认知**：gRPC 的核心价值是**标准化**（第 2 节那十件事），不是速度。选它主要应该因为契约、跨语言、流式，而不是因为"快"。

---

### Q6：怎么快速判断一份 gRPC 资料是不是过时的？

看两个地方：

| 看什么 | 过时（Grpc.Core） | 当前（grpc-dotnet） |
|---|---|---|
| 客户端怎么建 | `new Channel("host", port)` | `GrpcChannel.ForAddress(...)` |
| 服务端怎么建 | `new Server { Services = {...} }` | `WebApplication` + `MapGrpcService<T>()` |

还有一个信号：如果资料在讲 `.csproj` 里手工配 `<Protobuf>` 项而没有提到 `Grpc.Tools`，也可能比较旧。

---

## 附：本节涉及的文件（供以后查阅，现在不用看）

| 文件 | 说明 |
|---|---|
| `doc/implementation_comparison.md` | 两个 C# 实现的完整对比。**本仓库最好的入口文档之一** |
| `src/Grpc.AspNetCore.Server/Internal/GrpcProtocolConstants.cs` | HTTP/2 / HTTP/3 协议常量 |
| `src/Grpc.Net.Client/GrpcChannel.cs` | 客户端入口（第 17 节详读） |
| `src/Grpc.AspNetCore.Server/GrpcEndpointRouteBuilderExtensions.cs` | 服务端入口 `MapGrpcService` |
