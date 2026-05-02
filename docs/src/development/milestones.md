# 里程碑与未来计划 (Milestones)

构建一个分布式文件系统是一项浩大的工程。为了保持开发节奏并确保每个阶段都有可验证的成果，我们将整个 RDFS 的开发周期拆分为多个渐进式的里程碑（Milestones）。

本路线图展示了通往 V1.0（最小可行性产品 MVP）的详细路径，以及对未来架构演进的展望。

---

## 1. 通往 V1.0 的演进路径 (The Road to V1.0)

### 📌 v0.1：高并发内存单机版 (Sharded In-Memory Skeleton)
*目标：在不涉及网络通信的情况下，构建系统的“大脑”。*
- [x] 搭建 Cargo Workspace 目录结构。
- [x] 完成核心架构文档（Design Docs）编写。
- [ ] 实现 `Inode Tree` 的读写分离模型：写操作通过 `NameSystemActor` 单线程串行处理，读操作直接并发查询 `DashMap`。
- [ ] 编写覆盖率大于 80% 的单元测试，确保路径解析、哈希表操作无死锁。

### 📌 v0.2：通信基建与安全鉴权 (RPC, Token & Distribution)
*目标：引入 gRPC 与安全机制，让节点能够跨进程安全通信。*
- [ ] 在 `common` 模块中使用 `tonic` 编译 `hdfs.proto`。
- [ ] NameNode 实现 `SecretManager`，跑通 MasterKey 的生成与 Block Token 的 HMAC 签发。
- [ ] DataNode 实现注册与周期性 `Heartbeat`，并通过心跳响应在本地同步 MasterKey。
- [ ] 跑通 Client -> NameNode 的基础元数据交互（附带 Token 分发）。

### 📌 v0.3：双向流核心数据流 (Bidi-Pipeline & ACK)
*目标：实现分布式存储的灵魂——“鉴权、流水线复制与逆流确认”。*
- [ ] DataNode 实现本地存储引擎与 Checksum 强制二次校验。
- [ ] DataNode 实现 gRPC 拦截器，对所有读写流首包进行 Token 本地验签。
- [ ] 实现 Client -> DN1 -> DN2 -> DN3 的双向流 (Bidi-Stream) 并发写入。
- [ ] 跑通 Pipeline 的逆流确认机制（下游落盘 -> 上游收到 ACK -> 滑动窗口推进）。

### 📌 v0.4：容错与恢复 (Resilience & Recovery)
*目标：剪断网线，拔掉硬盘，系统依然可用。*
- [ ] Client 实现后台 `LeaseRenewer` (租约保活)，NameNode 实现租约超时检测及 Lease Recovery 协调逻辑。
- [ ] Client 实现 Pipeline Recovery 状态机（断点续传、踢出坏节点、触发 NameNode 升级 `gen_stamp`）。
- [ ] DataNode 实现后台 `Block Scanner` 主动发现静默坏块；Client 实现 `ReportBadBlock` 被动举报。
- [ ] NameNode 基于 `gen_stamp` 冲突裁决机制，通过心跳下发指令，完成块的重平衡 (Re-balancing)。

### 📌 v0.5 (V1.0 MVP)：记忆持久化与可用性 (Persistence)
*目标：重启不丢数据，并提供友好的用户交互。*
- [ ] `NameNode` 实现基于严格事务 ID (TxID) 的 `EditLog` (WAL) 追加写。
- [ ] 引入独立的 **Secondary NameNode** 进程，实现后台定期的 `FsImage` 快照拉取与合并。
- [ ] 完善 `rdfs-cli` 命令行工具（支持 `put`, `get`, `ls`, `rm`, `append`）。

---

## 2. 未来演进展望 (Beyond V1.0 Advanced Explorations)

当 V1.0 完成后，RDFS 将具备作为一个基础分布式存储引擎的所有特质。以下是未来可能探索的高级架构升级方向：

### 🔬 高可用大脑 (High Availability NameNode)
当前的单节点 NameNode 是系统的单点故障（SPOF）。未来可以引入 **Standby NameNode**。
* 引入 `Raft` 或 `Zookeeper` 协议实现主从选举。
* 使用分布式日志系统（如 `JournalNode` 集群）实现 Active 和 Standby 节点之间的元数据实时同步。

### 🔬 纠删码存储 (Erasure Coding)
目前采用的 3 副本策略会造成 200% 的存储空间浪费。未来可以在冷数据存储上引入纠删码（如 Reed-Solomon 算法）。
* 将 1.5 倍的存储开销（例如 6 个数据块 + 3 个校验块）转化为极高的数据冗余容错能力，大幅降低硬件成本。

### 🔬 联邦架构 (HDFS Federation)
单台 NameNode 的内存最终会遭遇物理瓶颈（上限通常在几亿个文件）。
* 引入 Federation 机制，允许集群中存在多个 NameNode 实例，通过挂载点（Mount Table）将不同的目录树映射到不同的 NameNode 上，实现横向无限扩展（Scale-out）。

---

## 3. 已知限制 (Known Limitations)

任何系统都有其边界，请在开发和使用过程中牢记 RDFS V1.0 的以下限制：

* 不支持文件内容的随机修改（仅支持整体覆盖与末尾追加）。
* 极度不适合存储大量小文件（KB级别），会迅速耗尽 NameNode 的内存。
* 虽然系统底层通过 Block Token 实现了坚固的数据平面隔离防线，但尚不包含类似 POSIX 的复杂用户/用户组 (User/Group ACLs) 元数据权限映射。