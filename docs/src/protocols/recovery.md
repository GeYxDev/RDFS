# 容错与副本恢复机制 (Fault Tolerance)

在 RDFS 的设计哲学中，我们假设底层的硬件是极其不可靠的。为了保证文件数据的高可用性和完整性，系统必须能够在无需人工干预的情况下，自动检测故障、隔离异常并恢复数据。

---

## 1. 节点级故障检测 (Node Failure Detection)

系统的健康状态由 NameNode 通过**心跳机制 (Heartbeat)** 来维持。

* **心跳周期 (Heartbeat Interval)：** DataNode 启动后，默认每隔 3 秒向 NameNode 发送一次 `Heartbeat`。
* **超时状态机 (Timeout State Machine)：** NameNode 内存中维护着一个 `DataNodeTracker`，追踪时间戳：
    * **Stale (迟钝)：** 连续 10 秒（约 3 个周期）未收到心跳。NameNode 暂时将其从读写流量的候选列表中剔除，避免拖慢 Client 请求。
    * **Dead (宕机)：** 连续 30 秒未收到心跳。NameNode 正式判定节点死亡，触发全局范围的副本重平衡。

---

## 2. 稳态副本重平衡 (Replica Re-balancing)

RDFS 默认的数据块副本数（Replication Factor）为 3。NameNode 的后台线程会持续监控 `BlockMap` 中的实际副本数与期望副本数的差异。

### 2.1 匮乏块恢复 (Under-replicated Blocks)
当 DataNode 宕机导致某些 Block 副本数降至 1 或 2 时：
1. **加入队列：** NameNode 将这些 BlockID 加入 `ReplicationQueue`。
2. **分配任务：** NameNode 挑选一个拥有健康副本的 DataNode (Source)，以及一个空间充足的 DataNode (Target)。
3. **异步指令：** 在 Source 节点下一次发来心跳时，NameNode 通过返回值下发 `Command::ReplicateBlock` 指令。
4. **节点间拷贝：** Source 节点主动与 Target 节点建立 gRPC 连结，将数据流式推送。Target 落盘后发送 `BlockReceived` 汇报，副本数恢复为 3。

### 2.2 冗余块清理 (Over-replicated Blocks)
* **僵尸节点复活场景：** 如果被判定为 Dead 的节点网络恢复，重新连入集群，并汇报自己拥有的 Block。这会导致部分 Block 的副本数变成 4。
* **清理机制：** NameNode 发现副本超载后，会根据机架感知或磁盘剩余空间策略，挑选出一个多余的副本，向其所在的 DataNode 发送 `Command::DeleteBlock`。DataNode 收到后直接在物理磁盘上 `unlink` 抹除数据。

---

## 3. 动态写入容错与防脑裂 (Dynamic Pipeline Recovery)

对于正在写入（处于 `UNDER_CONSTRUCTION` 状态）且未完成的 Block，如果发生节点宕机，传统的重平衡机制是不适用的。RDFS 采用**管线降级与世代隔离**机制：

* **世代版本号 (gen_stamp) 隔离：** 当管线中某个节点断开，Client 向 NameNode 申请剔除该节点时，NameNode 必须将该 Block 的 `gen_stamp` 升级（如从 V1 变 V2）。
* **防御旧数据复活：** 如果被剔除的节点假死恢复，它本地磁盘上残留的是 V1 版本的脏数据。当它向 NameNode 汇报 V1 块时，NameNode 对比内存中最新的 V2 版本，会立刻判定 V1 为过期垃圾，并下发删除指令，从而绝对杜绝了脏数据污染读取流。

---

## 4. 磁盘级数据完整性防御 (Data Integrity & Bit Rot)

除了节点宕机，磁盘物理介质导致的个别比特翻转（静默损坏）同样致命。RDFS 采用“被动+主动”双重校验机制。

### 4.1 被动校验 (Client-Side Validation)
* **写入校验：** Client 推送 Chunk 时必须附带 CRC32 校验和。DataNode 收到后进行二次校验，校验通过才落盘，并将校验和存入独立的元数据文件（如 `block_1024.meta`）。
* **读取拦截：** Client 拉取数据时，DataNode 会同时返回数据和 Checksum。Client 比对失败后，抛弃该数据，向 NameNode 调用 `ReportBadBlock`，并无缝切换至其他副本。

### 4.2 主动防御 (DataNode Block Scanner)
* **后台巡检：** 为了防止冷数据长期不被读取而损坏，DataNode 内部运行将一个低优先级的异步任务 `Block Scanner`（块扫描器）。
* **工作流：** 扫描器在节点 I/O 空闲时，循环读取本地磁盘上的每一个 Block，重新计算 CRC32 并与 `.meta` 文件比对。一旦发现损坏，DataNode 主动向 NameNode 宣告该块失效，NameNode 会提前启动 `ReplicateBlock` 流程，在损坏蔓延前完成自我修复。

---

## 5. NameNode 容错与演进 (SPOF Constraints)

在 RDFS 的架构基线中，控制流高度集中：
* **DataNode 宕机是完全容错的。**
* **NameNode 宕机是致命的单点故障 (Single Point of Failure)。**

如果 NameNode 进程崩溃，只要其挂载的持久化存储（`FsImage` 目录快照 和 `EditLog` 增量操作日志）安全，重启后可在几分钟内恢复全量内存目录树。但若宿主机磁盘彻底损坏且无异地备份，整个文件系统将永久丢失（即便 DataNode 上数据都在，也无法解析文件名和目录结构）。

*(注：工业级的最终演进方案，是引入 Standby NameNode 配合基于 Raft 协议的 Quorum Journal Manager (QJM) 来实现高可用 HA 切换。)*