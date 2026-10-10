这是一份为你整理的 **gRPC (C++) 核心 API 速查手册与面试深度剖析指南**，与之前的 Protobuf 笔记配合食用，便构成了完整的微服务网络传输与序列化知识闭环。

---

# 📚 gRPC (C++) 核心架构、API 与面试深度笔记

---

## 模块一：核心概念与 C++ API 速查手册

### 1. 核心架构三剑客（底层角色定位）
* **`Channel`（连接通道）**：表示到指定服务端的**长连接/连接池抽象**。负责底层的连接建立、HTTP/2 多路复用、负载均衡与心跳重试。**线程安全，属于重量级对象，全局/跨线程复用**。
* **`Stub`（客户端存根）**：由 `protoc` 生成的**客户端调用代理**。内部包装了 Channel，提供了与 `.proto` 文件中完全一致的方法签名，轻量级对象。
* **`Context`（请求上下文）**：分为 `ClientContext` 与 `ServerContext`。负责单次 RPC 调用的生命周期治理（**超时控制、请求取消、TraceID 元数据隐式透传**）。**不可跨 RPC 复用，单次调用独立使用**。

---

### 2. 常用 C++ API 速查表

| 场景分类                | API 签名 / 示例                                              | 功能说明与高频细节                                           |
| :---------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **创建连接**            | `grpc::CreateChannel(addr, creds);`                          | `creds` 常选用 `grpc::InsecureChannelCredentials()`（测试）或 SSL 凭证。 |
| **初始化存根**          | `auto stub = Greeter::NewStub(channel);`                     | 将 Stub 绑定到底层 Channel 上。                              |
| **超时设置 (客户端)**   | `context.set_deadline(std::chrono::system_clock::now() + 3s);` | **极度重要**。为防止服务端处理慢拖垮客户端，必须显式限制最大等待时间。 |
| **取消感知 (服务端)**   | `if (context->IsCancelled()) { return CANCELLED; }`          | 轮询/检测客户端是否已提前挂断或超时，及时退出，**释放计算资源**。 |
| **元数据注入 (客户端)** | `context.AddMetadata("token", "xyz");`                       | 类似 HTTP Header，用于传递鉴权 Token、TraceID 等辅助追踪信息。 |
| **元数据读取 (服务端)** | `const auto& client_meta = context->client_metadata();`      | 获取请求头集合（Key-Value multimap 结构）。                  |
| **服务端流式写出**      | `writer->Write(reply_msg);`                                  | 服务端源源不断向客户端推流数据，若写失败（对端挂断）会返回 `false`。 |
| **客户端流式读取**      | `while (reader->Read(&reply)) { ... }`                       | 客户端阻塞式从网络缓冲区拉取下一条消息，直至流终结返回 `false`。 |
| **流结束处理**          | `Status status = reader->Finish();`                          | 客户端拉取完数据流后，**必须调用 `Finish()`** 获取服务端的最终执行状态码。 |
| **返回状态码**          | `return Status::OK;`<br>`return Status(StatusCode::NOT_FOUND, "msg");` | gRPC 推荐使用内置的规范化业务状态码（`OK`, `NOT_FOUND`, `UNAVAILABLE` 等）。 |

---

## 模块二：面试高频考察题（底层机制深度拆解）

### 考点 1：gRPC 为什么选择基于 HTTP/2 协议，它比 HTTP/1.1 强在哪里？

**面试回答核心要点（按优先级展开）：**

1. **多路复用（Multiplexing - 破除队头阻塞）**：
   * *HTTP/1.1 的痛点*：即便开启了 Keep-Alive，单个 TCP 连接在同一时刻也只能处理一个 HTTP“请求-响应”。后续请求必须等待前一个响应完全返回，这导致了应用层的**队头阻塞（Head-of-Line Blocking）**。
   * *HTTP/2 的机制*：引入了**二进制分帧层（Binary Framing Layer）**。所有的通信都在单一的 TCP 连接上完成，每个请求被划分成不同的**流（Stream）**，流由携带 Stream ID 的数据**帧（Frame）**交织传输。双向通信在同一个 TCP 连接上并发乱序穿插，彻底消除了应用层的阻塞等待，连接复用率接近 100%。
2. **头部压缩（HPACK 算法）**：
   * RPC 调用通常参数极小（几个字节），如果每次通信都带上一堆纯文本、重复度极高的 Header（Cookie, User-Agent, Token），网络带宽极大被浪费。
   * HTTP/2 在通信两端共同维护**静态字典与动态字典**，重复传输的 Header 只需传递一个索引数字（例如 1 字节代表 `method: POST`），平均能压缩 80% 以上的报头体积。
3. **原生支持 Server Push 与全双工数据流（Streaming）**：
   * HTTP/1.1 是严格的“单向请求-单向应答”模型，做推送只能靠低效的轮询或升级为 WebSocket。
   * HTTP/2 将消息切分为更小的 `DATA` 帧传输，使得**服务端主动推流、客户端推流、全双工对冲聊天**原生支持，非常契合微服务复杂交互。

---

### 考点 2：gRPC 是如何实现“级联超时控制”与“请求取消”的？

**问题背景：** 微服务场景中，服务 A 调用服务 B，服务 B 又调用服务 C。如果服务 C 卡死，整个调用链路会不会被耗尽线程而雪崩？

**详细解答机制：**

1. **Deadline（截止时间）机制而非 Timeout（超时时长）**：
   * gRPC 在网络报头中传递的是**绝对时间点（Deadline）**，而不是“耗时 3 秒”。
   * 当 A 设定了 `3 秒后截止` 调用 B 时，B 收到请求能计算出剩余时间。如果 B 在处理业务前已经过了 1 秒，B 调用 C 时，其透传的 Deadline 只剩下 2 秒。这消除了跨网络节点网络抖动对倒计时的计算偏差。
2. **RST_STREAM 帧与级联取消（Cascading Cancellation）**：
   * 当 A 的客户端达到 Deadline 判定超时，A 内部会直接销毁本次调用，并立刻向 B 发送一个 HTTP/2 的 **`RST_STREAM` 错误帧（Reset Stream）**，强制终止这一条 Stream，而无需关闭底层的 TCP 连接。
   * B 的 gRPC 框架收到该帧后，会自动将此请求上下文的标志位置位：`context->IsCancelled() = true`。
   * 如果 B 的业务代码在轮询检查（或者在调用 C 的 Stub 时把自己的 context 传递进去了），B 就会立即中断对 C 的请求，同样向 C 发送 `RST_STREAM`。
   * **价值**：这种级联式阻断能让整个微服务调用链迅速释放正在阻塞的 CPU、内存和线程资源，是**防止微服务雪崩效应（Cascading Failure）的核心利器**。

---

### 考点 3：gRPC 的四种通信模式，其应用场景是什么？

**详细解答：**

1. **Unary RPC（简单一元 RPC）**：
   * **表现**：1 Request $\rightarrow$ 1 Response。
   * **场景**：经典的一问一答，替代传统的 RESTful 增删改查（CRUD）、查询用户详情、订单支付等。
2. **Server Streaming RPC（服务端流式 RPC）**：
   * **表现**：1 Request $\rightarrow$ N Responses。
   * **场景**：
     * **行情/消息推送**：比如证券交易所的实时股票 K 线推送。
     * **大模型 LLM 对话**：如 ChatGPT 逐字输出打字机效果。
     * **大体量报表导出**：数据库百万级行数据，服务端边查边推，避免服务端一次性将数据全塞入内存 OOM。
3. **Client Streaming RPC（客户端流式 RPC）**：
   * **表现**：N Requests $\rightarrow$ 1 Response。
   * **场景**：
     * **大文件分块断点续传**：客户端把一个 2GB 的文件按 64KB 切片，连续多次 `Write()` 推送给服务端，传输完毕后，服务端返回文件落地成功及 MD5。
     * **边缘物联网（IoT）连续打点采集**：传感器不断采集温度数据连续上传，直到达到一批次，服务端返回一次确认。
4. **Bidirectional Streaming RPC（双向流式 RPC）**：
   * **表现**：N Requests $\leftrightarrow$ M Responses（互相完全异步收发）。
   * **场景**：
     * **实时聊天应用（Chat）**。
     * **实时多人对战游戏**：客户端不断发送玩家操纵摇杆的坐标，服务端不断下发周围其他玩家的位置与技能同步信息。

---

### 考点 4：C++ gRPC 的同步模型（Sync）与异步模型（Async/CompletionQueue）对比

**问题：** 为什么大厂高性能核心网关极少使用本教程演示的简单同步 API？

**详细解答：**

* **同步 API（Sync - Thread-per-request）的局限**：
  * 本教程 Demo 使用的是**同步线程池模型**。每个到来的 RPC 请求都会阻塞一个工作线程，直到整个流程处理完毕返回。
  * 当遇到高延迟的 I/O 操作（访问数据库、远端调用）或慢客户端时，线程会被挂起。随着并发增加（如 10,000 并发），需要创建上万个线程，**上下文切换开销（Context Switch）巨大，内存直接被耗尽崩溃**。
* **异步 API（Async - Event-driven 基于 CompletionQueue）的设计**：
  * **基于 Reactor 模式**：gRPC C++ 提供了 `CompletionQueue`（底层基于 Linux `epoll` 实现）。
  * **非阻塞**：一个线程发起网络读/写后**不等待**，直接把请求标记（Tag）挂到 `CompletionQueue` 上，线程去处理其他任务。
  * **事件循环（Event Loop）**：少量的 I/O 线程（比如等于 CPU 核心数）不断从 `CompletionQueue->Next(&tag, &ok)` 中取出就绪事件并执行回调。
  * **总结**：**同步 API 编码门槛低、可读性极好，适合普通业务；异步 API 状态机复杂、无回调地狱但代码反人类，但能在极少线程下扛住几万乃至数十万 QPS 的超高并发。**（*注：现代 gRPC 也开始逐步引入 Callback API 改善异步编程体验*）。