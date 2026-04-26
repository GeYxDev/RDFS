# 数据节点 (DataNode)

![Role](https://img.shields.io/badge/Role-Worker_Node-blue.svg)
![IO](https://img.shields.io/badge/I%2FO-tokio::fs-orange.svg)

DataNode 是 RDFS (Rust Distributed File System) 系统的“苦力”与存储底座。它的核心任务是将网络中流转的字节流安全、高效地落入物理磁盘，并随时准备将它们再通过网络发送出去。

⚠️ **核心原则：** DataNode 是“无脑”的。它不关心这些 Block 拼起来是什么文件，它只听从 NameNode 的指挥，并响应 Client 的读写请求。

## 🎯 核心职责

1. **底层物理存储 (Block Storage):** 负责在本地文件系统上按 `block_id` 存储真实的数据碎片，并维护每个 Block 的元数据（如长度、Checksum 校验和）。
2. **流水线传输 (Pipeline Replication):** 在 Client 写入数据时，DataNode 不仅要将数据写入本地磁盘，还要作为代理，将数据流式转发给下一个 DataNode。
3. **高并发流式 I/O (Streaming I/O):** 高效处理来自多个 Client 和其他 DataNode 的 gRPC Stream 读写请求，保证极低的内存占用（Zero-copy / Chunked transfer）。
4. **状态汇报 (Heartbeat & Report):** 周期性地向 NameNode 发送心跳，汇报自身的存活状态、磁盘容量，以及本地真实存在的 Block 列表。

## 🏗️ 内部架构与并发模型

DataNode 属于典型的高 I/O 密集型应用。在 Rust 中，我们必须极其小心地处理磁盘 I/O，以防阻塞异步运行时：

* **异步文件系统：** 严禁使用标准库 `std::fs`（它会阻塞线程）。所有磁盘读写必须使用 `tokio::fs`。
* **内存流水线：** 在处理写入时，采用一边落盘一边通过网络转发的设计。内部使用 `tokio::sync::mpsc` (多生产者单消费者通道) 来做读写解耦的内存缓冲。
* **大文件切片：** 内存中绝对不能直接 `read_to_end` 读取完整的 64MB Block，必须以 `64KB` 为 Chunk 循环流式读取和发送。

## 📂 模块划分建议 (`src/`)

```text
src/
├── main.rs                 # 启动入口：解析命令行参数(端口, 存储目录)，启动服务
├── rpc_server.rs           # 实现 hdfs.proto 中 DataNodeService (读写 Stream)
├── storage/                # 本地磁盘存储引擎子模块
│   ├── mod.rs
│   ├── disk_mgr.rs         # 目录结构初始化、容量统计
│   └── block.rs            # 单个 Block 的读、写、校验和(Checksum)计算逻辑
└── namenode_client/        # 与 NameNode 通信的客户端模块
    ├── mod.rs
    └── heartbeat.rs        # 后台死循环任务：定期向 NameNode 报活
```

## 🚀 开发里程碑 (Local Milestones)

- [ ] **Phase 1: 本地存储引擎**
    - 实现 `storage` 模块，能够在本地指定目录下按 `block_id` 创建文件并追加写入 `bytes`。
    - 实现基本的 CRC32 校验和验证。
- [ ] **Phase 2: NameNode 注册与心跳**
    - 实现作为 Client 连接 NameNode，启动后台任务定时发送 `Heartbeat`。
- [ ] **Phase 3: 单节点 RPC 读写**
    - 实现 `DataNodeService`，跑通 Client -> DataNode 的单节点流式写入与读取。
- [ ] **Phase 4: 流水线复制 (Pipeline)**
    - 升级写入逻辑，实现收到上游 Chunk 后，一边写本地，一边向下游发起 gRPC 请求转发。