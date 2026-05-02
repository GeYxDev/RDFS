# 元数据节点 (NameNode)

![Role](https://img.shields.io/badge/Role-Master_Node-blue.svg)
![Concurrency](https://img.shields.io/badge/Concurrency-Actor_%2B_DashMap-orange.svg)
![Security](https://img.shields.io/badge/Security-HMAC_Token-green.svg)

NameNode 是 RDFS (Rust Distributed File System) 系统的“大脑”。它负责管理整个文件系统的命名空间（目录树）、控制客户端对文件的访问，并维护所有数据块（Block）到物理数据节点（DataNode）的映射关系。

⚠️ **核心原则：** NameNode **绝对不** 触碰实际的文件数据。所有的实际数据传输均由 Client 直接与 DataNode 交互完成。

## 🎯 核心职责

1. **命名空间管理 (Namespace Management)：** 维护内存中的目录树（Inode Tree），处理文件和目录的创建、删除、重命名等元数据操作。
2. **数据块寻址与版本控制 (Block Mapping & Versioning)：** 记录每个文件由哪些 Block 组成、副本存放在哪些 DataNode 上，并统筹分配世代版本号以防止分布式脑裂。
3. **细粒度安全与鉴权 (Access Token Issuer)：** 维护集群周期轮转的 MasterKey。为合法的读写请求实时计算并签发基于 HMAC-SHA256 的 **Block Token**，并将其通过心跳下发给 DataNode。
4. **租约与并发控制 (Lease Management)：** 颁发并维护客户端的写锁，即租约。处理客户端的定期续约请求，并在客户端异常崩溃时触发租约超时回收，保证文件状态一致性。
5. **集群状态监控 (Cluster Management)：** 接收来自 DataNode 的持续心跳（Heartbeat），监控节点健康状态。统筹处理节点的宕机下线、副本重平衡（Re-balancing）以及坏块恢复。
6. **持久化 (Persistence)：** 通过写前日志（EditLog / WAL）和内存快照（FsImage）保证重启后元数据不丢失。

## 🏗️ 内部架构与并发模型

为了处理高并发的 RPC 请求，NameNode 在 Rust 中的内存模型使用以下设计：

* **分段锁并发哈希表 (Sharded Lock)：** 核心命名空间采用扁平化的 `DashMap<u64, Inode>` 存储，放弃嵌套的 `Arc<RwLock<T>>` 树状结构。通过逻辑 Inode ID 寻址，将读请求的锁竞争物理隔离到不同的 Shard，实现路径解析与元数据查询的高并发无阻塞。
* **读写分离模型 (Read Write Separation)：** 核心命名空间 `NamespaceMap` 使用 `DashMap` 实现扁平化存储。所有修改元数据的操作全部由唯一的 **NameSystem Actor** 在单线程中串行处理，彻底根除多线程写锁竞争与死锁风险；高频读操作则绕过 Actor，直接无锁并发访问 `DashMap`，达到近线性的多核扩展。
* **异步 RPC 服务 (Async gRPC)：** 基于 `tonic` 框架，监听 `9000` 端口，实现 `hdfs.proto` 中定义的 `NameNodeService`。

## 📂 模块划分建议 (`src/`)

```text
src/
├── main.rs              # 启动入口：初始化全局状态，启动 gRPC Server
├── rpc_server.rs        # 实现 hdfs.proto 中 NameNodeService 的各个接口逻辑
├── metadata/            # 元数据管理子模块
│   ├── mod.rs
│   ├── inode.rs         # 基于 DashMap 的并发扁平化目录树
│   ├── actor.rs         # NameSystem Actor，串行处理所有写命令
│   └── block_map.rs     # block_id 到 DataNode 的映射表，及 gen_stamp 管理
├── cluster/             # 集群与并发控制子模块
│   ├── mod.rs
│   ├── tracker.rs       # 记录各个 DataNode 的心跳时间、容量、状态
│   └── lease.rs         # 客户端租约管理器
├── auth/                # 安全鉴权子模块
│   ├── mod.rs
│   └── secret_mgr.rs    # MasterKey 轮转管理与 Block Token HMAC 签名生成
└── persistence/         # FsImage 快照与 EditLog 增量操作日志
```

## 🚀 开发里程碑

- [ ] **Phase 1：高并发内存模型建立**
  - 定义 `Inode` 结构，实现 `NamespaceMap` 与 `NameSystemActor` 单线程写循环。
  - 实现 `BlockMap` 与初始化的 `gen_stamp` 分配逻辑。
- [ ] **Phase 2：基础 RPC 与租约接入**
  - 启动 Tonic 服务，实现 `CreateFile`、`AllocateBlock`、`CompleteFile` 核心流转。
  - 实现 `SecretManager`，在 `AllocateBlock` 时计算并下发 Block Token。
  - 实现 `LeaseManager` 及 `RenewLease` 接口。
  - 实现 `CompleteFile` 最小副本数校验，不足时拒绝提交。
- [ ] **Phase 3：集群状态与容错闭环**
  - 实现心跳接收器，用后台定时任务剔除超时的 DataNode，在心跳 Response 中将 MasterKey 下发给 DataNode。
  - 对接 DataNode 的 `BlockReport` 和 `BlockReceived` 逻辑。
  - 实现故障节点剔除后的副本重平衡指令下发（`ReplicateBlock` / `DeleteBlock`）。
- [ ] **Phase 4：持久化机制**
  - 实现 EditLog 的追加写入（包含 Token 密钥的持久化）。
  - 实现 FsImage 的加载与保存，为 SNN 预留 Checkpoint 接口。