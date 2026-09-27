# grpc-dotnet 学习笔记

> 分支：`study/reading`　|　基线：`7db6c142`（与 upstream/master 同步）
> 同步上游：`git fetch upstream && git checkout master && git merge --ff-only upstream/master`

## 学习目标

| 阶段 | 目标 | 说明 |
|---|---|---|
| 短期 | **言之有物** | 熟悉概念，能在团队 gRPC 讨论中听懂并说对话 |
| 长期 A | 理解设计思路 | 不求会写，求看懂"为什么这么设计" |
| 长期 B | 工业控制场景落地 | 设备网关 / HMI / 数据采集方向的实际应用 |

背景：7 年 C# 经验（工业控制软件），同事已在做 gRPC 相关工作，后续可能参与。

---

## 时间节奏

- **每天 1 小时**，但 **1h 是上限不是指标**。996 下状态差就跳过，**不补偿**。
- 计划预留 20-30% 空转缓冲。冲刺期没有产出是正常的。
- 不要用周末补工作日的欠账 —— 这是最容易导致放弃的模式。

### 每次会话的固定格式

| 时间 | 内容 | 谁主导 |
|---|---|---|
| 0-10 min | 背景概念：这块为什么存在、解决什么问题 | 我讲 |
| 10-45 min | 设计思路 + 关键代码片段（不逐行读） | 我带读 |
| 45-60 min | 你提问，我把结论写进 `study/notes/` | 你主导 |

**你不需要写代码**，只需要理解和提问。笔记由我产出，你 review 即可。

---

## 第一段：概念地图（约 3 周｜15 节）

**目标是言之有物。这段完全不读源码。**

- [ ] 01. gRPC 是什么：与 REST 的区别、与 `Grpc.Core`（已进维护模式）的区别、为什么基于 HTTP/2
- [ ] 02. Protocol Buffers 基础：`.proto` 语法、生成哪些代码、为什么比 JSON 高效
- [ ] 03. proto3 类型系统与**版本兼容规则**（字段号、`reserved`、不要复用字段号）
- [ ] 04. 四种调用形态：unary / server streaming / client streaming / duplex，各自适用场景
- [ ] 05. Metadata 与 Trailer：一次调用在 HTTP 层面到底传了什么
- [ ] 06. 状态码与错误处理：`StatusCode` 取值 + `RpcException` + 富错误模型（`google.rpc.Status`）
- [ ] 07. Deadline 与 Cancellation：语义、在调用链上如何传播、与 HTTP 超时的区别
- [ ] 08. 拦截器：客户端/服务端，等价于你熟悉的什么（AOP / 中间件）
- [ ] 09. 重试与对冲：`RetryPolicy` / `HedgingPolicy`、**幂等性陷阱**
- [ ] 10. 连接与通道：`GrpcChannel` 生命周期、连接复用、为什么不要频繁创建
- [ ] 11. 传输与端口：HTTP/2 vs HTTP/1.1、明文 vs TLS、端口共享、UDS / named pipe
- [ ] 12. 负载均衡与名字解析：pick_first / round_robin、DNS resolver
- [ ] 13. 压缩与性能：gzip / deflate、什么时候真的有意义
- [ ] 14. 生态与部署：健康检查、服务反射、gRPC-Web、JSON transcoding、OpenTelemetry
- [ ] 15. **已知限制与坑**：keepalive / connection backoff 不支持、NAT 断连、HTTP/2 多路复用队头阻塞

---

## 第二段：设计思路 + 代码风格（约 4 周｜15 节）

**这段开始读源码，但只读设计，不逐行。**

### 2A. 调用链设计（8 节）

- [ ] 16. 从 `examples/Greeter` 看生成的代码：`ClientBase` / `ServerBase` / `BindService` 长什么样
- [ ] 17. `src/Grpc.Net.Client/GrpcChannel.cs`：怎么把 `HttpClient` 变成 `CallInvoker`
- [ ] 18. `src/Grpc.Core.Api/CallInvoker.cs`：为什么抽象成 4 个方法
- [ ] 19. `src/Grpc.Net.Client/Internal/HttpClientCallInvoker.cs` + `GrpcCall.cs`：一次调用的对象模型
- [ ] 20. 服务端路由：`GrpcEndpointRouteBuilderExtensions.cs` + `ServerCallHandlerFactory.cs`
- [ ] 21. `src/Grpc.AspNetCore.Server/Internal/CallHandlers/ServerCallHandlerBase.cs` 精读
- [ ] 22. `src/Grpc.AspNetCore.Server/Internal/HttpContextServerCallContext.cs`：适配器模式范例
- [ ] 23. 四个 `*ServerCallHandler.cs` 的差异对比

### 2B. 代码风格专题（4 节）⭐ 这个仓库写法确实值得学

- [ ] 24. **async 快路径模式**（见下方"范例"）—— 最值得学的一处
- [ ] 25. `[LoggerMessage]` 源生成器高性能日志（全仓库 161 处）
- [ ] 26. 分配规避：`ArrayPool` / `Span` / `ReadOnlySequence` / `ValueTask`（47 处）
- [ ] 27. 项目工程化规范：`TreatWarningsAsErrors` / `Nullable` / 集中包管理 / SourceLink

### 2C. 横切机制（3 节）

- [ ] 28. `src/Grpc.AspNetCore.Server/Internal/ServerCallDeadlineManager.cs`：deadline 怎么实现
- [ ] 29. `src/Shared/Server/InterceptorPipelineBuilder.cs`：拦截器管线设计
- [ ] 30. `src/Grpc.AspNetCore.Server/Internal/PercentEncodingHelpers.cs` + 线格式细节

---

## 第三段：按需深入（遇到再读）

- [ ] `src/Grpc.Net.Client/Balancer/`（25 个文件）—— 仅在需要服务发现/负载均衡时
- [ ] `test/` 测试架构 —— 在需要写测试或提 PR 时
- [ ] `src/dotnet-grpc` 工具链 —— 在需要管理 proto 时
- [ ] `src/Grpc.Net.ClientFactory/` —— 在需要 DI 集成时

---

## 明确不读（噪音）

| 内容 | 原因 |
|---|---|
| `#if` 条件编译（全仓库 171 处） | 多目标框架（net462 → net10.0）的代价，是唯一影响可读性的地方 |
| `src/Shared/ThrowHelpers/*` | 参数校验样板 |
| `NullableAttributes.cs` / `IsExternalInit.cs` / `CodeAnalysisAttributes.cs` | 老框架兼容 shim |
| `kokoro/` / `build/` / `.github/` | CI 脚本 |

---

## 范例：为什么这个仓库值得学写法

`src/Grpc.AspNetCore.Server/Internal/CallHandlers/ServerCallHandlerBase.cs` 第 73-100 行：

```csharp
var handleCallTask = HandleCallAsyncCore(httpContext, serverCallContext);

if (handleCallTask.IsCompletedSuccessfully)
{
    return serverCallContext.EndCallAsync();
}
else
{
    return AwaitHandleCall(serverCallContext, MethodInvoker.Method, handleCallTask);
}

static async Task AwaitHandleCall(HttpContextServerCallContext serverCallContext,
                                  Method<TRequest, TResponse> method, Task handleCall)
{
    try
    {
        await handleCall;
        await serverCallContext.EndCallAsync();
    }
    catch (Exception ex)
    {
        await serverCallContext.ProcessHandlerErrorAsync(ex, method.Name);
    }
}
```

三个可以带走的东西：

1. **async 快路径**：`IsCompletedSuccessfully` 判断，已完成就同步返回，避免状态机分配。在同步完成的常见路径上省掉一次 `Task` 分配。
2. **`static` 局部函数**：标记 `static` 后编译器不会捕获闭包，避免额外的闭包对象分配。
3. **注释解释"为什么"**：第 126-127 行 `IsReadOnly could be true if middleware has already started reading the request body` —— 写的是"为什么会出现这种情况"，不是"这行在干什么"。全仓库的注释都是这个风格。

这类模式在工业控制的高频路径上直接可用。

### 工程质量证据

| 指标 | 数值 | 含义 |
|---|---|---|
| `TreatWarningsAsErrors` | `true` | 警告即错误 |
| `Nullable` | `enable`（**0 处 `#nullable disable` 残留**） | 可空引用类型全面启用 |
| `.editorconfig` | 406 行 | 微软官方风格配置 |
| `LangVersion` | 12.0 | 现代语法 |
| `[LoggerMessage]` 源生成器 | 161 处 | 零装箱高性能日志 |
| `internal sealed class` | 122 处 | 默认封闭，纪律性强 |
| 测试项目 | 11 个 | `test/` 下分层（UnitTests / FunctionalTests / IntegrationTests） |

---

## 笔记目录

按日期存放，每次会话一份：

```
study/
  README.md          ← 本文件（进度追踪）
  notes/
    YYYY-MM-DD-<主题>.md
  pitfalls.md        ← 随时记录遇到的坑，不做成正式交付物
```
