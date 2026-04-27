# 数据节点 (DataNode)

![Role](https://img.shields.io/badge/Role-Worker_Node-blue.svg)
![IO](https://img.shields.io/badge/I%2FO-tokio::fs-orange.svg)

DataNode 是 RDFS (Rust Distributed File System) 系统的存储底座。它的核心任务是将网络中流转的字节流安全、高效地落入物理磁盘，并随时准备将它们再通过网络发送出去。

⚠️ **核心原则：** DataNode 是“无脑”的。它不关心这些 Block 拼起来是什么文件，它只听从 NameNode 的指挥，并响应 Client 的读写请求。

## 🎯 核心职责

1. **底层物理存储 (Block Storage)：** 负责在本地文件系统上按 `block_id` 存储真实的数据碎片。必须严格维护每个 Block 的元数据（包括长度、校验和以及 **`gen_stamp` 世代版本号**）。
2. **流水线传输 (Pipeline Replication)：** 在 Client 写入数据时，不仅要将数据写入本地磁盘，还要作为代理，将数据流式转发给下游 DataNode。
3. **状态汇报 (Heartbeat & Report)：** 周期性发送心跳（报活与容量）；在 Block 落盘瞬间发送**增量汇报 (`BlockReceived`)**；定期发送**全量块汇报 (`BlockReport`)** 进行最终一致性对账。
4. **指令执行 (Command Execution)：** 坚决服从 NameNode 在心跳响应中下发的指令，异步执行本地废弃块的删除（Delete）或向其他节点推送副本（Replicate）。
5. **主动防御 (Block Scanner)：** 在后台低优先级运行块扫描器，定期读取本地数据比对 Checksum，提前发现磁盘静默损坏并上报。

## 🏗️ 内部架构与并发模型

DataNode 属于典型的高 I/O 密集型应用。在 Rust 中，我们必须极其小心地处理网络与磁盘的交火：

* **异步文件系统：** 严禁使用标准库 `std::fs`（会阻塞 Tokio 线程）。所有磁盘读写必须使用 `tokio::fs`。
* **零拷贝与背压流控：** 内存中绝对不能缓冲完整的 64MB Block。必须以 `64KB` 为 Chunk，利用 `bytes::Bytes` 进行无拷贝的流式传递；严格依赖 gRPC/HTTP2 底层的滑动窗口机制实现网络到磁盘的**背压 (Backpressure)**，防止 OOM。
* **读写解耦：** 在 Pipeline 写入时，采用一边落盘一边网络转发的设计。内部使用 `tokio::sync::mpsc` (多生产者单消费者通道) 解耦接收端和发送/落盘端。

## 📂 模块划分建议 (`src/`)

```text
src/
├── main.rs                 # 启动入口：解析命令行参数，启动服务与后台守护进程
├── rpc_server.rs           # 实现 hdfs.proto 中 DataNodeService (读写 Stream 流水线)
├── storage/                # 本地磁盘存储引擎子模块
│   ├── mod.rs
│   ├── disk_mgr.rs         # 目录结构初始化、容量统计
│   ├── block.rs            # 单个 Block 的读写、校验和计算、gen_stamp 校验
│   └── scanner.rs          # 后台块扫描器 (主动防御机制)
└── namenode_client/        # 与 NameNode 通信的客户端模块
    ├── mod.rs
    ├── heartbeat.rs        # 定期报活，并解析执行 NameNode 下发的 Command
    └── report.rs           # 负责增量汇报 (BlockReceived) 和全量汇报 (BlockReport)
```

## 🚀 开发里程碑

- [ ] **Phase 1: 本地存储与防脑裂引擎**
    - 实现 `storage` 模块，按 `block_id` 落盘，并配套保存 `.meta` 文件（存放 CRC32 与 `gen_stamp`）。
- [ ] **Phase 2: 注册、心跳与汇报双链路**
    - 启动后台任务定时发送 `Heartbeat`。
    - 实现 `BlockReceived` 和 `BlockReport` 的 RPC 调用机制。
- [ ] **Phase 3: 单节点 RPC 读写闭环**
    - 实现 `DataNodeService`，跑通单节点的流式读写，并在 EOF 时触发落盘确认。
- [ ] **Phase 4: 流水线复制 (Pipeline) 与背压**
    - 升级写入逻辑，通过 `mpsc` 实现收到上游 Chunk 后边写本地、边向下游转发。
- [ ] **Phase 5: 容错指令执行与主动扫描**
  - 解析 Heartbeat 返回的 `Command` 并执行删除/复制动作。
  - 启动 `scanner.rs` 后台循环扫描本地坏块。