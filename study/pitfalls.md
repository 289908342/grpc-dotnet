# 坑位记录

随手记，不做成正式交付物。遇到具体问题时回查，或者同事那边碰到了一起补。

| # | 坑 | 说明 | 来源 |
|---|---|---|---|
| 1 | **不支持 keepalive** ⚠️ **可能已过时** | 仓库文档说 "HttpClient and Kestrel don't provide support"，但那是 .NET Core 3.x 时代写的。**.NET 5+ 已提供 `SocketsHttpHandler.KeepAlivePingDelay` / `KeepAlivePingTimeout`（发 HTTP/2 PING 帧）**。详见下方"坑 1 更正" | `doc/implementation_comparison.md` L151-165 |
| 2 | **不支持 connection backoff** | 断线后 gRPC 层不会自动重连，需要应用层自己做 | `doc/implementation_comparison.md` L151-165 |
| 3 | **重试只能用于幂等操作** | `RetryPolicy` 默认会重试失败调用。写 PLC 寄存器、下指令这类非幂等操作必须显式排除 | 待验证（第 09 节） |
| 4 | **TLS 在 macOS 上有历史限制** | .NET 8 之前 macOS 上的服务端 TLS 支持有限 | `doc/implementation_comparison.md` L103 |
| 5 | **`GrpcChannelOptions.HttpHandler` 与负载均衡冲突** | 自己传 `SocketsHttpHandler` 时，`BalancerHttpHandler` 内部也会设 `ConnectCallback`（`Balancer/Internal/BalancerHttpHandler.cs` L69-71），做 UDS / named pipe 时要注意 | 源码 L69-71 |
| 6 | **L4 负载均衡对 gRPC 基本无效** | L4 按 TCP 连接分发，而 gRPC 把所有调用挤在一条连接上 → **全打到一个后端**。必须用客户端侧 LB 或 L7 代理（Envoy / YARP） | [官方文档](https://learn.microsoft.com/en-us/aspnet/core/grpc/performance) |
| 7 | **流式调用忘记 dispose 会拖累服务端** | 不 dispose 不只是客户端泄漏 —— **服务端会一直留着这条流**。大量泄漏的流会影响服务端稳定性 | [官方文档](https://learn.microsoft.com/en-us/aspnet/core/grpc/performance) |
| 8 | **HTTP/2 并发流有上限（默认 100）** | 超过上限后新调用在客户端**排队**。长流式调用容易占满配额。可设 `SocketsHttpHandler.EnableMultipleHttp2Connections = true` | [官方文档](https://learn.microsoft.com/en-us/aspnet/core/grpc/performance) |
| 9 | **大消息整个进内存** | 消息发送前和接收后都完整驻留内存；>85,000 字节会进**大对象堆（LOH）**。大二进制载荷建议分块流式或改用 HTTP 端点 | [官方文档](https://learn.microsoft.com/en-us/aspnet/core/grpc/performance) |

## 坑 1 更正：keepalive 其实是**部分支持**的

`doc/implementation_comparison.md` 说 keepalive "not supported"，这个说法**停留在 .NET Core 3.x 时代**。

.NET 5+ 的实际情况 —— 客户端可以配置 HTTP/2 PING：

```csharp
var handler = new SocketsHttpHandler
{
    PooledConnectionIdleTimeout = Timeout.InfiniteTimeSpan,
    KeepAlivePingDelay = TimeSpan.FromSeconds(60),
    KeepAlivePingTimeout = TimeSpan.FromSeconds(30),
    EnableMultipleHttp2Connections = true
};

var channel = GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
{
    HttpHandler = handler
});
```

`KeepAlivePingDelay` / `KeepAlivePingTimeout` 会在空闲时发 **HTTP/2 PING 帧** —— 正是 gRPC keepalive 规范想要的效果。

### ⚠️ 但有一个硬前提

官方文档用 Important 标注：

> Keep alive pings **require the cooperation of the server**. Do not enable keep alive pings in the client without verifying that the server supports them. A server which does not support keep alive pings will usually ignore the first few pings, and will then send a **`GOAWAY`** message, **closing the active HTTP/2 connection**.

**服务端不支持时，客户端单方面开启反而会被 GOAWAY 踢掉连接。**

### 结论

| 判断 | 结论 |
|---|---|
| 客户端能做 HTTP/2 keepalive 吗 | ✅ 能（.NET 5+） |
| 能直接开吗 | ❌ 必须**先确认内核侧也支持**，否则适得其反 |
| 等同 gRPC 完整 keepalive 语义吗 | ❌ 不等同（如 `permit-without-calls` 等策略未覆盖） |
| 还要自己做应用层心跳吗 | 视内核侧情况而定 —— **先验证再决定** |

**行动项**：确认内核侧的 gRPC 服务端（Kestrel 的 `KestrelServerLimits.Http2`）是否允许 / 响应 keepalive PING。第 15 节回来验证。

