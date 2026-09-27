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

### gRPC 和 REST 是什么关系

**REST 不是协议，是一种架构风格**（Roy Fielding, 2000）。核心四条：

1. **资源** —— 一切皆资源，用 URL 标识（`/devices/42/temperature`）
2. **统一接口** —— 用 HTTP 动词操作资源（GET 读 / POST 建 / PUT 改 / DELETE 删）
3. **无状态** —— 每个请求自带全部上下文，服务端不记得上次是谁
4. **表述** —— 资源的表现形式（JSON / XML），可协商

机械层面就是 HTTP 请求-响应：`方法 + URL + 头 + 体` → `状态码 + 头 + 体`

| | REST | gRPC |
|---|---|---|
| 传输 | HTTP/1.1（通常） | HTTP/2（必须） |
| 载荷 | 文本（JSON） | 二进制（protobuf） |
| 接口标识 | URL 路径 + 动词 | 服务名 + 方法名（`/pkg.Service/Method`） |
| 契约 | OpenAPI（**可选、事后补**） | `.proto`（**强制、编译期**） |
| 调用单元 | 一次请求-响应 | **一条流** |
| 流式 | 外挂（SSE / WebSocket） | 原生 |
| 人眼可读 | 是（`curl` 就能调） | 否（需 `grpcurl` + 反射） |

**最本质的区别：面向资源 vs 面向方法**

- **REST**：`GET /devices/42/temperature` —— 你在**操作一个资源**
- **gRPC**：`DeviceService.GetTemperature(...)` —— 你在**调用一个函数**

gRPC 表面上也用 HTTP POST，但它**故意违反了 REST 的资源导向** —— 把 URL 当**方法名**用。

> **所以两者是两条不同的路线，不是新旧替代关系。** 可以共存（gRPC 甚至提供 JSON transcoding，能把 gRPC 服务自动暴露成 REST 接口）。

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

## 6. 工程实践问答（你来答，我来批）

> **这一节只给问题，不给答案。**
>
> 你先自己想、自己写答案，然后我们聊一遍 —— 我负责挑错、补充、以及指出你没考虑到的角度。
>
> 答不上来很正常，**猜也行**。猜错的地方才是真正学到东西的地方。
> 讨论完一题就勾掉一个。

---

- [x] **Q1** 看到一个教程里写 `new Channel("localhost", 5000)`，这段代码能直接用吗？如果不能，为什么？

- [x] **Q2** 我们每秒调用 10 次，这算频繁吗？如果有人**每次调用都** `GrpcChannel.ForAddress(...)` 新建一个 channel，会发生什么？

- [x] **Q3** gRPC 是 REST 的替代品吗？我们现有的 REST 接口应该全部换成 gRPC 吗？

- [x] **Q4** 什么情况下**不该**用 gRPC？举一个你们系统里可能遇到的例子。

- [ ] **Q5** 都说 gRPC 比 REST 快。这个"快"具体来自哪里？在什么情况下这个优势会消失？
  ⏳ **待重答** —— 第 02 节讲完 protobuf 编码后回来重做这题（"快"主要来自 protobuf，不是 HTTP/2）

- [x] **Q6** grpc-dotnet 不支持 keepalive。**为什么**？这个限制是 gRPC 设计上的疏忽，还是一种必然的取舍？
  （提示：回顾第 2 节"为什么选 HTTP/2"，以及第 5 节第 5 条）

- [x] **Q7** 如果你们要新做一个"应用读取内核状态"的接口，你会选 unary 还是 server streaming？
  说出你的理由，**以及你不确定的地方**。

---

## 7. 讨论后修正的关键认知（会话 01）

> 答完第 6 节后，被纠正和强化的认知。**这才是这节真正的收获。**

| # | 原来的认知 | 修正后 |
|---|---|---|
| 1 | 新建 channel 的问题是"构造对象耗时" | **对象构造几乎不花钱（纳秒级）**。贵的是连接建立握手链：socket → TCP → **TLS** → HTTP/2 → 发起调用。差距 1~2 个数量级（亚毫秒 vs 10~50ms） |
| 2 | gRPC 快是因为用了 HTTP/2 | **主要来自 protobuf**（不传字段名、varint、按字段号跳转）。HTTP/2 贡献的是**并发效率**和**省掉重复握手**，不是单条消息更快 |
| 3 | 连接不稳定会让 gRPC 优势消失 | **反了**。不稳定会**放大劣势**（TCP 队头阻塞），不是削弱优势。优势真正消失于：不复用连接 / 消息太小 / 大块二进制 / 瓶颈在别处 |
| 4 | keepalive 做不到是因为 HTTP/2 协议没有 | **协议里有 PING 帧**，是 .NET 实现没暴露。而且 **.NET 5+ 已经支持了**（`SocketsHttpHandler.KeepAlivePingDelay`）—— 见 `pitfalls.md` 坑 1 更正 |
| 5 | unary 每次都要重新建立通讯 | **unary 复用 TCP 连接**，只是新建一条 stream。差别是 **stream 的生命周期**，不是 TCP 连接 |
| 6 | 同机跨进程"不太可能断连" | 内核重启、应用重启、安全软件干预、对端进程退出都会断。**"不太可能" ≠ "不会"；假设不断连 = 断连时行为未定义** |
| 7 | 选 server streaming 是因为"持续连接" | unary 也是持续连接。**真正理由是：变化是事件驱动的，不知道下次变化何时到来** |

### 两个新的重要认知

#### ① gRPC 的定位是「软实时」，不是「硬实时」

| 适合 | 不适合 |
|---|---|
| 监控、HMI、配置下发 | 控制环、安全联锁 |

原因是**没有延迟上界**（GC 暂停、TCP 重传、线程调度），**不是"不稳定"**。

> ⚠️ 说"不稳定"会误导你去加重试 —— 但**重试解决不了延迟问题**，只会让延迟更长。

硬实时的逻辑应该留在内核内部，用共享内存或本地调用。**这是架构分层问题，和 gRPC 无关。**

#### ② Server streaming 有一个被低估的风险：应用端可能拖住内核

```
应用消费慢（UI 卡顿 / GC）
    → HTTP/2 流控窗口耗尽
        → 服务端（内核侧）写入阻塞   ⚠️
```

**在工控系统里这可能是不可接受的：应用端的 UI 卡顿不应该影响内核的实时性。**

缓解手段（Kestrel 配置，**是缓解不是消除**）：

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    var http2 = options.Limits.Http2;
    http2.InitialConnectionWindowSize = 1024 * 1024 * 2;  // 2 MB
    http2.InitialStreamWindowSize = 1024 * 1024;          // 1 MB
});
```

### 两个设计约束（官方文档明确提到）

1. **Server streaming 的客户端只能用「取消」来停止流**（它没有请求流）。
   如果取消的开销影响服务端，官方建议**改用双向流** —— 客户端完成请求流即为一个优雅的停止信号。

2. **流式调用必须 dispose。**
   否则不只是客户端泄漏内存和资源 —— **服务端会一直留着这条流**。
   大量泄漏的流会影响服务端稳定性。

### 参考

- [Performance best practices with gRPC](https://learn.microsoft.com/en-us/aspnet/core/grpc/performance)
- [Inter-process communication with gRPC](https://learn.microsoft.com/en-us/aspnet/core/grpc/interprocess)

---

## 附：本节涉及的文件（供以后查阅，现在不用看）

| 文件 | 说明 |
|---|---|
| `doc/implementation_comparison.md` | 两个 C# 实现的完整对比。**本仓库最好的入口文档之一** |
| `src/Grpc.AspNetCore.Server/Internal/GrpcProtocolConstants.cs` | HTTP/2 / HTTP/3 协议常量 |
| `src/Grpc.Net.Client/GrpcChannel.cs` | 客户端入口（第 17 节详读） |
| `src/Grpc.AspNetCore.Server/GrpcEndpointRouteBuilderExtensions.cs` | 服务端入口 `MapGrpcService` |
