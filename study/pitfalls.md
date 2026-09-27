# 坑位记录

随手记，不做成正式交付物。遇到具体问题时回查，或者同事那边碰到了一起补。

| # | 坑 | 说明 | 来源 |
|---|---|---|---|
| 1 | **不支持 keepalive** | `HttpClient` 和 Kestrel 都不提供，grpc-dotnet 无法实现。长连接在 NAT / 工业防火墙下会被静默断开 | `doc/implementation_comparison.md` L151-165 |
| 2 | **不支持 connection backoff** | 同上。断线后 gRPC 层不会自动重连，需要应用层自己做 | `doc/implementation_comparison.md` L151-165 |
| 3 | **重试只能用于幂等操作** | `RetryPolicy` 默认会重试失败调用。写 PLC 寄存器、下指令这类非幂等操作必须显式排除 | 待验证（第 09 节） |
| 4 | **TLS 在 macOS 上有历史限制** | .NET 8 之前 macOS 上的服务端 TLS 支持有限 | `doc/implementation_comparison.md` L103 |
| 5 | **`GrpcChannelOptions.HttpHandler` 与负载均衡冲突** | 自己传 `SocketsHttpHandler` 时，`BalancerHttpHandler` 内部也会设 `ConnectCallback`（`Balancer/Internal/BalancerHttpHandler.cs` L69-71），做 UDS / named pipe 时要注意 | 源码 L69-71 |

## 待办：现场网络相关

工业现场最容易出问题的是"连接看着还在、实际已经死了"。第 15 节会专门讲，届时补一份应用层心跳的方案。
