# 里程碑与未来计划 (Milestones)

构建一个分布式文件系统是一项浩大的工程。为了保持开发节奏并确保每个阶段都有可验证的成果，我们将整个 RDFS 的开发周期拆分为多个渐进式的里程碑（Milestones）。

本路线图展示了通往 V1.0（最小可行性产品 MVP）的详细路径，以及对未来架构演进的展望。

---

## 1. 通往 V1.0 的演进路径 (The Road to V1.0)

### 📌 v0.1: 内存单机版 (In-Memory Skeleton)
*目标：在不涉及网络通信的情况下，构建系统的“大脑”。*
- [x] 搭建 Cargo Workspace 目录结构。
- [x] 完成核心架构文档（Design Docs）编写。
- [ ] 实现 `Inode Tree` 的纯内存增删改查逻辑。
- [ ] 编写覆盖率大于 80% 的单元测试，确保路径解析、哈希表操作无死锁。

### 📌 v0.2: 通信基建与“假”分布式 (RPC & Mock Distribution)
*目标：引入 gRPC，让 NameNode 和 DataNode 能够跨进程说话。*
- [ ] 在 `common` 模块中使用 `tonic` 编译 `hdfs.proto`。
- [ ] 实现 `DataNode` 的注册与周期性 `Heartbeat`。
- [ ] 实现 `NameNode` 的 `DataNodeTracker`，能够基于心跳检测节点存活。
- [ ] 使用本地磁盘上的 Mock 数据跑通 Client -> NameNode 的元数据请求（创建、列出目录）。

### 📌 v0.3: 核心数据流 (Data Pipeline)
*目标：实现分布式存储的灵魂——“流水线复制”。*
- [ ] `DataNode` 实现本地存储引擎，能够按 `block_id` 落盘数据流。
- [ ] Client 实现大文件的 64MB 自动切片逻辑。
- [ ] **[高难度]** 实现 Client -> DN1 -> DN2 -> DN3 的流水线并发写入与 ACK 确认机制。
- [ ] 实现基于拓扑排序的并发就近读取（Read Block）。

### 📌 v0.4: 容错与恢复 (Resilience & Recovery)
*目标：剪断网线，拔掉硬盘，系统依然可用。*
- [ ] `DataNode` 实现按 Chunk 的 CRC32 数据校验和计算与比对。
- [ ] `NameNode` 实现后台扫描任务，发现 Under-replicated 副本匮乏块。
- [ ] `NameNode` 通过心跳向 `DataNode` 下发复制指令，完成数据块的跨节点重平衡。

### 📌 v0.5 (V1.0 MVP): 记忆持久化与可用性 (Persistence)
*目标：重启不丢数据，并提供友好的用户交互。*
- [ ] `NameNode` 实现基于序列化的 `EditLog` (WAL) 追加写。
- [ ] `NameNode` 实现定期的 `FsImage` 快照合并（Checkpointing）。
- [ ] 完善 `rdfs-cli` 命令行工具（支持 `put`, `get`, `ls`, `rm`）。

---

## 2. 未来演进展望 (Beyond V1.0: Advanced Explorations)

当 V1.0 完成后，RDFS 将具备作为一个基础分布式存储引擎的所有特质。但在真实的工业生产环境中，这仅仅是个开始。以下是未来可能探索的高级架构升级方向：

### 🔬 高可用大脑 (High Availability NameNode)
当前的单节点 NameNode 是系统的单点故障（SPOF）。未来可以引入 **Standby NameNode**。
* 引入 `Raft` 或 `Zookeeper` 协议实现主从选举。
* 使用分布式日志系统（如 `JournalNode` 集群）实现 Active 和 Standby 节点之间的元数据实时同步。

### 🔬 纠删码存储 (Erasure Coding)
目前采用的 3 副本策略会造成 200% 的存储空间浪费。未来可以在冷数据存储上引入纠删码（如 Reed-Solomon 算法）。
* 将 1.5 倍的存储开销（例如 6 个数据块 + 3 个校验块）转化为极高的数据冗余容错能力，大幅降低硬件成本。

### 🔬 真正的并发控制架构优化 (Finer-Grained Locking)
目前 NameNode 的目录树采用了全局读写锁机制，在应对海量小文件并发写入时会成为瓶颈。
* 探索将全局的 `RwLock<HashMap>` 升级为分段锁哈希表（如 `dashmap`）。
* 或者采用更硬核的细粒度目录级加锁机制（Lock Coupling / Directory-tree Locking）。

### 🔬 快照与追加写 (Snapshots & Append)
* **快照：** 允许用户瞬间冻结某个目录的状态，防止误删，利用写时复制（Copy-on-Write）技术实现。
* **追加写：** 打破 V1.0 中的“一次写入”限制，允许客户端在已完成的文件末尾追加新数据，为支持流式计算（如 Kafka 后端存储）提供基础。

---

## 3. 已知限制 (Known Limitations)

任何系统都有其边界，请在开发和使用过程中牢记 RDFS V1.0 的以下限制：
* 不支持文件内容的随机修改（仅支持整体覆盖）。
* 极度不适合存储大量小文件（KB级别），会迅速耗尽 NameNode 的内存。
* 当前版本缺乏鉴权（Authentication）和基于用户的权限控制（ACL），假设网络环境绝对受信。