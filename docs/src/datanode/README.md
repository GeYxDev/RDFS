# 数据节点 (DataNode)

![Role](https://img.shields.io/badge/Role-Worker_Node-blue.svg)
![IO](https://img.shields.io/badge/I%2FO-tokio::fs-orange.svg)
![Security](https://img.shields.io/badge/Security-HMAC_Token-green.svg)

DataNode 是 RDFS (Rust Distributed File System) 系统的存储底座。它的核心任务是将网络中流转的字节流安全、高效地落入物理磁盘，并随时准备将它们再通过网络发送出去。

⚠️ **核心原则：** DataNode 是“无脑”的。它不关心这些 Block 拼起来是什么文件，它只听从 NameNode 的指挥，并响应 Client 的读写请求。

## 🎯 核心职责

1. **底层物理存储 (Block Storage)：** 负责在本地文件系统上按 `block_id` 存储真实的数据碎片。必须严格维护每个 Block 的元数据（包括长度、校验和以及 **`gen_stamp` 世代版本号**）。
2. **强制端到端校验 (Data Integrity)：** 在收到写入请求时，**必须重新计算 Checksum** 并与 Client 传来的期望值比对，拒绝任何静默损坏的数据落盘。
3. **去中心化安检 (Token Validation)：** 拦截所有客户端的网络 I/O 请求首包，利用本地缓存的 MasterKey 实时验签 **Block Token**，绝不放行非法请求。
4. **双向流水线传输 (Pipeline & ACKs)：** 在 Client 写入数据时，不仅要将数据写入本地磁盘、向下游转发，还要在收到下游确认后，通过 **双向流 (Bidi-Stream)** 向上游回传异步 ACK。
5. **状态汇报与同步 (Heartbeat & Report)：** 周期性发送心跳，并 **接收 NameNode 下发的最新的 Token 密钥**；在 Block 落盘瞬间发送增量汇报 (`BlockReceived`)；定期发送全量块汇报 (`BlockReport`)。
6. **指令执行 (Command Execution)：** 坚决执行 NameNode 下发的管理指令 (删除、复制)，以及 Client 在管线恢复中发起的块截断与世代版本号更新请求。
7. **主动防御 (Block Scanner)：** 在后台低优先级运行块扫描器，定期读取本地数据比对 Checksum，提前发现磁盘静默损坏并上报。

## 🏗️ 内部架构与并发模型

DataNode 属于典型的高 I/O 密集型应用。在 Rust 中，必须极其小心地处理网络与磁盘的交火：

* **异步文件系统：** 严禁使用标准库 `std::fs`（会阻塞 Tokio 线程）。所有磁盘读写必须使用 `tokio::fs`。
* **零拷贝与背压流控：** 内存中绝对不能缓冲完整的 64MB Block。必须以 `64KB` 为 Chunk，利用 `bytes::Bytes` 进行无拷贝的流式传递；严格依赖 gRPC/HTTP2 底层的滑动窗口机制实现网络到磁盘的 **背压 (Backpressure)**，防止 OOM。
* **全双工与解耦：** Pipeline 涉及上游推数据、下游推数据、本地写磁盘、下游回传 ACK、向上游回传 ACK 五个并发动作。使用 `tokio::sync::mpsc` 与 `tokio::spawn` 将读写通道彻底解耦，防止环形死锁。

## 📂 模块划分建议 (`src/`)

```text
src/
├── main.rs                 # 启动入口：解析命令行参数，启动服务与后台守护进程
├── rpc_server.rs           # 实现 hdfs.proto 中 DataNodeService (支持双向流的 Pipeline)
├── auth/                   # 安全与鉴权子模块
│   ├── mod.rs
│   └── validator.rs        # 解析 gRPC 首包，执行 HMAC-SHA256 验签并维护本地 MasterKey
├── storage/                # 本地磁盘存储引擎子模块
│   ├── mod.rs
│   ├── disk_mgr.rs         # 目录结构初始化、容量统计
│   ├── block.rs            # 单个 Block 的读写、校验和计算、gen_stamp 校验
│   └── scanner.rs          # 后台块扫描器 (主动防御机制)
└── namenode_client/        # 与 NameNode 通信的客户端模块
    ├── mod.rs
    ├── heartbeat.rs        # 定期报活，解析执行 Command，并同步 MasterKey
    └── report.rs           # 负责增量汇报 (BlockReceived) 和全量汇报 (BlockReport)
```

## 🚀 开发里程碑

- [ ] **Phase 1：本地存储与强制校验**
    - 实现 `storage` 模块，跑通 Chunk 写入、Checksum 二次计算，配套保存 `.meta` 文件。
- [ ] **Phase 2：注册、心跳与密钥同步**
    - 启动后台任务定时发送 `Heartbeat`。
    - 从心跳响应中提取 `MasterKey` 列表，更新至本地内存。
    - 实现 `BlockReceived` 和 `BlockReport` 的 RPC 调用机制。
- [ ] **Phase 3：单节点 RPC 读写与鉴权闭环**
    - 实现 `DataNodeService`，在 Stream 首包拦截并校验 Block Token。
    - 跑通单节点的流式读写，并在 EOF 时触发落盘确认。
- [ ] **Phase 4：流水线复制与背压 (Bidi-Pipeline)**
    - 升级写入逻辑，实现 `Client -> DN1 -> DN2` 的双向流转发。
    - 实现“下游确认 + 本地落盘 -> 向上游回传 ACK”的逆流确认机制。
- [ ] **Phase 5：容错指令执行与主动扫描**
  - 解析 Heartbeat 返回的 `Command` 并执行删除/复制动作。
  - 启动 `scanner.rs` 后台循环扫描本地坏块。