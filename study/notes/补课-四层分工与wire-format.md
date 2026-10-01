# 回顾补课：四层分工 + tag/varint

> 触发原因：第 01、02 节回顾时，R2（四层分工）和 R5（tag 结构 / varint）没答上来。
> 结论：**抽象机制和机械细节，不"用一次"就会掉。** 所以这份补课用"动手"而不是"重读"。

---

## 第一部分：四层分工

### 不要背表，用一个测试推

上次给的 10 行对应表是**结论**，不是**方法**，所以记不住。正确的做法是用这个测试现场推导：

| 换掉什么 | 什么会坏 | 所以这一层负责 |
|---|---|---|
| protobuf 换成 JSON | 消息字节变了，**但调用照样成功** | 只管**消息内部**长什么样 |
| HTTP/2 换成 HTTP/1.1 | 没法多路复用 / 双向流 / trailer 带状态码 | 只管**消息之间**的关系 |
| gRPC 换成 REST | 错误怎么表达、超时怎么传 变了 | 只管**语义怎么表达** |
| 换掉 `GrpcChannel` | 连哪个后端、断了怎么办 变了 | 只管**连接生命周期** |

> **判别口诀：换掉它会不会坏？坏了就说明是它的职责。**

### 一次调用的完整轨迹

```
应用:  await client.GetTemperatureAsync(req)
  │
  │ ① 通道层    GrpcChannel 决定用哪个连接 / 哪个后端
  │ ② protobuf  req 对象 → 字节
  │ ③ HTTP/2    开一条 stream（分配 stream ID=7），字节作为 body 发出
  ▼
内核:  收到请求
  │ ④ HTTP/2    知道这是 stream 7
  │ ⑤ protobuf  字节 → req 对象
  │ ⑥ 你的服务方法被调用
  │ ⑦ protobuf  reply 对象 → 字节
  │ ⑧ gRPC 规范 把状态码写进 trailer：grpc-status: 0
  │ ⑨ HTTP/2    用 stream 7 把响应 + trailer 发回
  ▼
应用:  ⑩ HTTP/2    stream 7 的响应到了 → 这就是我等的那次调用   ★配对在这
  │ ⑪ protobuf  字节 → reply 对象
  │ ⑫ 通道层    如果这次失败了，决定是否切后端 / 重连
```

**盯住 ③ 和 ⑩** —— 配对的全过程。protobuf 在这两步**完全没参与**。

**再看 ⑧** —— `grpc-status` 是 **gRPC 规范**定义的语义，HTTP/2 只是**帮忙运**它。
类比快递：快递公司负责运，但"易碎品"这个标记的**含义**是发货方定义的。

### 快速自测

1. 响应里带着 `grpc-status: 5` —— 由**哪一层定义**？（答：gRPC 规范。HTTP/2 只负责运）
2. 一条连接上 50 个并发请求，怎么知道哪条响应对应哪条请求？（答：HTTP/2 的 stream ID）
3. `GrpcChannel.ForAddress(...)` 影响哪一层？（答：通道层 —— 连接生命周期）

---

## 第二部分：tag + varint

### 公式 1：tag

```
tag = (字段号 << 3) | wire_type
```

| wire type | 用于 |
|---|---|
| `0` | int32 / int64 / bool / enum（VARINT） |
| `1` | double / fixed64（I64） |
| `2` | string / bytes / 嵌套 message（LEN） |
| `5` | float / fixed32（I32） |

### 公式 2：varint

```
v < 128:          [v]
128 ≤ v < 16384:  [(v & 0x7F) | 0x80,  v >> 7]
```

### 关键：tag 和 value 是两个东西，永远 tag 在前

```proto
message M { int32 a = 1; }   // a = 1
```

```
08 01
│  └─ value：varint(1) = 01        ← "值"
└──── tag：(1<<3)|0 = 08           ← "字段号 + 类型"
```

> ⚠️ 常见错误：把某个值的 varint 字节混进 tag 位置。
> **tag 描述字段，value 才是数据**，两者独立计算。

### 常见 tag 对照

| 字段号 | wire type | tag |
|---|---|---|
| 1 | 0 (VARINT) | `08` |
| 2 | 0 (VARINT) | `10` |
| 1 | 2 (LEN) | `0A` |
| 2 | 1 (I64) | `11` |
| 4 | 1 (I64) | `21` |

### 小值 varint 对照（背下来）

| 值 | varint |
|---|---|
| 0 | `00` |
| 1 | `01` |
| 5 | `05` |
| 127 | `7F` |
| **128** | **`80 01`** ← 分界点 |
| 150 | `96 01` |
| 300 | `AC 02` |

⚠️ **128~255 区间的陷阱**：这个区间内 varint 的第一字节**等于值本身**（因为值的第 7 位本来就是 1，和续位重合）。
例如 150 的十六进制是 `0x96`，`varint(150)` 的第一字节也是 `0x96` —— 别从这里推导规律。

---

## 动手练习（含答案）

```proto
message Reading {
  string device_id = 1;   // 值 = "AB"
  int32  count     = 2;   // 值 = 5
  int32  offset    = 3;   // 值 = 300
  double celsius   = 4;   // 值忽略，只需说占几字节
  int32  delta     = 5;   // 值 = -1
}
```

### 1. `device_id = "AB"`

```
字段 1，wire type 2 (LEN)
tag = (1 << 3) | 2 = 10 = 0x0A
长度 = 2 → 0x02
内容 = 41 42

→ 0A 02 41 42
```

### 2. `count = 5`

```
字段 2，wire type 0 (VARINT)
tag = (2 << 3) | 0 = 16 = 0x10
varint(5) = 05

→ 10 05
```

### 3. `offset = 300`

```
字段 3，wire type 0 (VARINT)
tag = (3 << 3) | 0 = 24 = 0x18
varint(300)：300 & 0x7F = 0x2C, |0x80 = 0xAC；300 >> 7 = 2
  → AC 02

→ 18 AC 02
```

### 4. `celsius`（double）

```
字段 4，wire type 1 (I64)
tag = (4 << 3) | 1 = 33 = 0x21
value = 固定 8 字节

→ 21 + 8 字节 = 共 9 字节
```

### 5. `delta = -1`（int32）⚠️ 有坑

```
字段 5，wire type 0 (VARINT)
tag = (5 << 3) | 0 = 40 = 0x28

int32 -1 → 符号扩展到 64 位 → 0xFFFFFFFFFFFFFFFF
varint(0xFFFFFFFFFFFFFFFF) = 10 字节

→ 28 FF FF FF FF FF FF FF FF FF 01     ← 总共 11 字节
```

**对比：如果声明成 `sint32`**

```
ZigZag(-1) = 1
→ 28 01                                  ← 只有 2 字节
```

**同一个值：11 字节 vs 2 字节。** 这就是"负数必须用 `sint`"的直观感受。

---

## 补课小结

| 要记的 | 内容 |
|---|---|
| **四层分工** | 不用背。用「换掉它会不会坏」推 |
| **tag** | `(字段号 << 3) \| wire_type`，**永远是第一个字节** |
| **varint** | `v < 128` 就是 `[v]`；否则切 7 位组、加续位 |
| **负数** | `int32` 负数 = **10 字节**；`sint32` 通常 1~2 字节 |
| **速算** | `128 ≤ v < 16384` → `[(v & 0x7F) \| 0x80, v >> 7]` |
