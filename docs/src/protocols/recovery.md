# 容错与副本恢复机制 (Fault Tolerance)

在 RDFS 的设计哲学中，我们假设底层的硬件是极其不可靠的。为了保证文件数据的高可用性和完整性，系统必须能够在无需人工干预的情况下，自动检测故障并恢复数据。

## 1. 故障检测模型 (Failure Detection)

系统的健康状态主要由 NameNode 通过**心跳机制 (Heartbeat)** 来维持。

* **心跳周期 (Heartbeat Interval)：** DataNode 启动后，每隔固定的时间（默认 3 秒）向 NameNode 发送一次 `HeartbeatRequest`。
* **超时判定 (Timeout Threshold)：** * NameNode 内存中维护着一个 `DataNodeTracker`，记录每个节点最后一次心跳的时间戳。
    * 如果 NameNode 连续 10 秒（约 3 个周期）未收到某节点的心跳，将其标记为 **"Stale" (迟钝)**，暂时不向其分配新的写入请求。
    * 如果连续 30 秒未收到心跳，将其正式标记为 **"Dead" (宕机)**。

## 2. 副本状态机与重平衡 (Replica Re-balancing)

RDFS 默认的数据块副本数（Replication Factor）为 3。当一个 DataNode 被标记为 "Dead" 时，它硬盘上存储的所有 Block 都会丢失一个副本，这些 Block 的实际副本数将降至 2 或 1，被称为 **Under-replicated Blocks（副本匮乏块）**。

### 2.1 恢复工作流 (Recovery Pipeline)

副本的恢复是由 NameNode 异步驱动的：

1. **发现匮乏：** NameNode 的后台定时任务（或宕机事件触发器）扫描整个 `BlockMap`，找出所有副本数小于 3 的 Block。
2. **加入队列：** 将这些 BlockID 加入到 `ReplicationQueue`（待复制队列）中。
3. **分配任务：** NameNode 针对每一个待复制的 Block：
    * 找出一个当前拥有该 Block 完好副本的健康 DataNode（作为 **Source**）。
    * 找出另一个磁盘空间充足且当前没有该 Block 的健康 DataNode（作为 **Target**）。
4. **下发指令：** NameNode **不会**自己去复制数据。它会在下一次 Source 节点发来心跳时，在 `HeartbeatResponse` 中携带一个 `Command::ReplicateBlock(block_id, target_ip)` 指令。
5. **节点间复制：** Source 节点收到指令后，主动与 Target 节点建立 gRPC 连结，将该 Block 的数据流式推送过去。完成后，Target 节点向 NameNode 汇报“我拥有了这个新 Block”，该 Block 的副本数恢复为 3。

## 3. 数据完整性与静默损坏 (Data Integrity & Silent Corruption)

除了节点直接宕机，磁盘由于物理原因导致个别 bit 翻转（静默损坏）同样致命。

* **写入时的 Checksum：** Client 在向 DataNode 发送 Chunk 时，必须同时计算并附带数据的 CRC32（或 SHA-256）校验和。DataNode 收到后进行二次校验，校验通过才落盘，并将校验和一同保存在磁盘的元数据文件中（如 `block_1024.meta`）。
* **读取时的 Checksum：** Client 在读取数据时，DataNode 会同时返回数据和保存的校验和。Client 在本地计算并比对。
* **损坏处理：** 如果 Client 发现校验和不匹配，会立刻抛弃该数据，尝试从该 Block 的备用 DataNode 读取，并向 NameNode 发送 `ReportBadBlock` RPC。NameNode 会立即将该损坏副本标记为无效，并触发上述的副本恢复工作流。

## 4. NameNode 单点故障 (SPOF) 的局限性

在 RDFS V1.0 的架构中，我们采用的是单 Master 架构。这意味着：
* **DataNode 宕机是完全容错的。**
* **NameNode 宕机是致命的（Single Point of Failure）。**

如果 NameNode 进程崩溃，只要它的持久化目录（FsImage 和 EditLog）还在，重启后系统可以恢复全量状态。但如果 NameNode 的宿主机硬盘彻底损坏且没有外部备份，整个文件系统的目录树将永久丢失（即便 DataNode 上存有所有数据块，我们也无法知道它们拼起来是什么文件）。

*在未来的架构演进中，可以通过引入 Standby NameNode 配合 Zookeeper 或 Raft 协议来实现 Master 的高可用 (HA)。*