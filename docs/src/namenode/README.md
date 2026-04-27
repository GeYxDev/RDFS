# 元数据节点 (NameNode)

![Role](https://img.shields.io/badge/Role-Master_Node-blue.svg)
![Concurrency](https://img.shields.io/badge/Concurrency-Arc%3CRwLock%3E-orange.svg)

NameNode 是 RDFS (Rust Distributed File System) 系统的“大脑”。它负责管理整个文件系统的命名空间（目录树）、控制客户端对文件的访问，并维护所有数据块（Block）到物理数据节点（DataNode）的映射关系。

⚠️ **核心原则：** NameNode **绝对不**触碰实际的文件数据。所有的实际数据传输均由 Client 直接与 DataNode 交互完成。

## 🎯 核心职责

1. **命名空间管理 (Namespace Management)：** 维护内存中的目录树（Inode Tree），处理文件和目录的创建、删除、重命名等元数据操作。
2. **数据块寻址与版本控制 (Block Mapping & Versioning)：** 记录每个文件由哪些 Block 组成、副本存放在哪些 DataNode 上，并统筹分配世代版本号以防止分布式脑裂。
3. **租约与并发控制 (Lease Management)：** 颁发并维护客户端的写锁，即租约。处理客户端的定期续约请求，并在客户端异常崩溃时触发租约超时回收，保证文件状态一致性。
4. **集群状态监控 (Cluster Management)：** 接收来自 DataNode 的持续心跳（Heartbeat），监控节点健康状态。统筹处理节点的宕机下线、副本重平衡（Re-balancing）以及坏块恢复。
5. **持久化 (Persistence) [计划中]：** 通过写前日志（EditLog / WAL）和内存快照（FsImage）保证重启后元数据不丢失。

## 🏗️ 内部架构与并发模型

为了处理高并发的 RPC 请求，NameNode 在 Rust 中的内存模型必须经过精心设计：

* **全局状态共享：** 整个 NameNode 的核心状态通常被包裹在 `Arc<RwLock<T>>` 或使用 `dashmap`（并发哈希表）中，以便多个 `tokio` 异步任务可以安全地并发读写。
* **异步 RPC 服务：** 基于 `tonic` 框架，监听 `9000` 端口，实现 `hdfs.proto` 中定义的 `NameNodeService`。

## 📂 模块划分建议 (`src/`)

```text
src/
├── main.rs              # 启动入口：初始化全局状态，启动 gRPC Server
├── rpc_server.rs        # 实现 hdfs.proto 中 NameNodeService 的各个接口逻辑
├── metadata/            # 元数据管理子模块
│   ├── mod.rs
│   ├── inode.rs         # 内存目录树结构定义
│   └── block_map.rs     # block_id 到 DataNode 的映射表，及 gen_stamp 管理
├── cluster/             # 集群与并发控制子模块
│   ├── mod.rs
│   ├── tracker.rs       # 记录各个 DataNode 的心跳时间、容量、状态
│   └── lease.rs         # 客户端租约管理器
└── persistence/         # [待开发] FsImage 与 EditLog 逻辑
```

## 🚀 开发里程碑

- [ ] **Phase 1: 内存模型建立**
  - 定义 `Inode` 结构，支持多级目录的并发读写。
  - 定义全局 `MetaDataManager`，跑通核心数据结构（目录树 + BlockMap）的单测。
- [ ] **Phase 2: 基础 RPC 与租约接入**
  - 引入 `common` 包中的 protobuf 定义，启动 Tonic 服务。
  - 实现 `CreateFile`、`AllocateBlock`、`CompleteFile` 核心流转。
  - 实现 `LeaseManager` 及 `RenewLease` 接口。
- [ ] **Phase 3: 集群状态与容错闭环**
  - 实现心跳接收器，用后台定时任务剔除超时的 DataNode。
  - 对接 DataNode 的 `BlockReport` 和 `BlockReceived` 逻辑。
  - 实现下发节点剔除与 `gen_stamp` 升级的指令逻辑。
- [ ] **Phase 4: 持久化机制**
  - 引入 `sled` 或手写 WAL，确保元数据落盘，实现系统崩溃恢复。