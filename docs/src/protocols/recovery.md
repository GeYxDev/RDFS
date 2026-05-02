# 容错与副本恢复机制 (Fault Tolerance)

在 RDFS 的设计哲学中，我们假设底层的硬件是极其不可靠的。为了保证文件数据的高可用性和完整性，系统必须能够在无需人工干预的情况下，自动检测故障、隔离异常并恢复数据。

---

## 1. 节点级故障检测 (Node Failure Detection)

系统的健康状态由 NameNode 通过 **心跳机制 (Heartbeat)** 来维持。

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
当被判定为 Dead 的僵尸节点网络恢复，重新连入集群并汇报自己拥有的 Block 时，NameNode 会执行严格的 **世代版本号 (GenStamp) 裁决算法**：
1. **版本号比对：** NameNode 将 DataNode 上报的块 `gen_stamp` 与自身内存中该块最新的 `gen_stamp` 期望值进行对比。
2. **低版本抛弃：** 若上报的 `gen_stamp` 低于期望值（说明这是管线断裂时残留的脏数据），无论当前健康副本数是多少，NameNode 均直接下发 `Command::DeleteBlock` 要求节点抹除该脏数据。
3. **冗余副本清理：** 若上报的 `gen_stamp` 与期望值一致（说明是完全健康的数据），此时若该块的副本数超过期望值（如变成 4 个副本），NameNode 会综合考量剩余磁盘空间，挑选出一个多余的副本下发删除指令。

---

## 3. 动态写入容错与防脑裂 (Dynamic Pipeline Recovery)

对于正在写入（处于 `UNDER_CONSTRUCTION` 状态）且未完成的 Block，如果发生节点宕机，传统的重平衡机制是不适用的。RDFS 采用 **管线降级与世代隔离** 机制，由 NameNode 授权、Client 执行恢复：

* **故障上报与授权：** 当管线中某个节点断开，Client 上报故障后，NameSystem Actor 剔除故障节点并将该 Block 的 `gen_stamp` 升级（如从 V1 变 V2），随后返回新版本号和新管线。
* **长度对齐后续传：** Client 联系所有幸存 DataNode，查询落盘长度，取最小值作为最终长度，下发截断指令并要求各节点更新 `gen_stamp` 为 V2。
* **防御旧数据复活：** 如果被剔除的节点假死恢复，它本地磁盘上残留的是 V1 版本的脏数据。当它向 NameNode 汇报 V1 块时，NameNode 对比内存中最新的 V2 版本，会立刻判定 V1 为过期垃圾，并下发删除指令，从而绝对杜绝了脏数据污染读取流。
* **读取路径双保险：** 即使 NameNode 尚未下发删除指令，客户端在读取时也不会拿到脏数据。因为 NameNode 在恢复结束后签发的所有新 Token 中都携带了最新的 `gen_stamp` V2。假死节点上的脏数据块仍是 V1，当其收到携带 V2 的 Token 时，DataNode 会在验签后执行 `gen_stamp` 比对，发现不匹配而直接拒绝读取。

### 3.1 块恢复状态保护 (Block Under Recovery)
为避免 Pipeline Recovery 期间 NameNode 后台重平衡线程与 Client 恢复操作冲突，引入 **Block Under Recovery** 状态：
* **进入条件：** NameNode 处理 `ReportFailedBlock` 或启动租约恢复时，将该 Block 标记为 `UnderRecovery`。
* **状态行为：**
  * 不参与后台副本调度（不生成 `ReplicateBlock` / `DeleteBlock` 指令）。
  * 拒绝客户端读取（`GetFileInfo` 或 `ReadBlock` 返回 `UNAVAILABLE` 错误）。
* **退出条件：** 恢复完成（`CompleteBlockRecovery` 或 `CommitBlockRecovery`）后，清除标记并更新副本列表与 `gen_stamp`。
* **超时保护：** 超过阈值（如 5 分钟）自动清除标记，触发常规副本复制。
* **设计意图：** 确保恢复期间 NameNode 不干扰手动修复，同时避免客户端读到不一致数据。

---

## 4. 租约恢复 (Lease Recovery)

当 Client 在写入过程中异常崩溃（未正常调用 `CompleteFile`），导致文件长期处于 `UNDER_CONSTRUCTION` 状态时，必须由 NameNode 触发租约恢复，以释放该文件并保证其可读。

### 4.1 触发条件
* NameNode 的 `LeaseManager` 检测到某文件的租约超过 60 秒未被续约。
* 新 Client 尝试打开一个处于 `UNDER_CONSTRUCTION` 状态的文件进行读写，主动请求恢复。

### 4.2 轮询策略
客户端调用 `RecoverLease` 后，不应假设文件立即可用。应周期性地（如每 2 秒）调用 `GetFileInfo` 检查 `file_status`：
* 若状态变为 `ACTIVE`，表示恢复成功，文件可正常读写。
* 若文件已不存在（`NOT_FOUND`），表示恢复超时失败，NameNode 已放弃该文件。
* 若仍为 `UNDER_CONSTRUCTION`，则继续等待直到超时（如 60 秒）后放弃。

### 4.3 恢复流程
1. **NameNode 发起恢复：** NameSystem Actor 统一处理租约超时与 `RecoverLease` 请求，仅将租约恢复任务放入后台队列后即刻返回，不阻塞等待任务执行；租约超时事件也采用相同异步后台处理逻辑。
   * 查询该文件最后一个 Block 的所有 DataNode 位置（从 `BlockMap` 获取）。
   * 将该 Block 标记为 `UnderRecovery` 状态，防止后台重平衡指令干扰。
   * 将该 Block 的 `gen_stamp` 升级。
   * 从幸存副本中选出一个 Primary DataNode。
   * 生成 `BlockRecoveryCommand`，通过 Primary DN 的下一次心跳响应下发。
2. **Primary DN 协调副本：** Primary DN 收到指令后。
   * 停止本地对该 Block 的任何残留写入。
   * 并行向所有其他副本 DN 发送 `InterDNRecoverBlock` RPC，携带 `block_id` 和 `new_gen_stamp`。
   * 收集各副本返回的 `final_length` 和 `is_committed` 标志。
   * 确定最终长度，若存在任一副本标记为 `COMMITTED`（Client 曾明确告知块已完成），则以这些 COMMITTED 副本的长度为准，否则取所有返回长度中的最小值。
   * 要求所有副本将 Block 截断至最终长度，并将本地 `gen_stamp` 更新为 `new_gen_stamp`。
   * 通过 `CommitBlockRecovery` RPC 将最终结果告知 NameNode。
3. **NameNode 闭环：** NameSystem Actor 收到 `CommitBlockRecovery` 后。
   * 更新内存中该 Block 的 `gen_stamp` 和 `final_length`。
   * 将文件状态从 `UNDER_CONSTRUCTION` 转为 `ACTIVE`。
   * 后续通过心跳清理未参与恢复的旧 gen_stamp 副本。
4. **放弃未恢复文件：** 若 Primary DN 在超时内未完成恢复（如所有副本均不可达），NameNode 执行 `AbandonFile`。
   * 删除该文件在目录树中的 Inode。
   * 将该文件的所有 Block 加入待删除列表，通过心跳下发给任意可达 DataNode 清理残留数据。

---

## 5. 磁盘级数据完整性防御 (Data Integrity & Bit Rot)

除了节点宕机，磁盘物理介质导致的个别比特翻转（静默损坏）同样致命。RDFS 采用“被动+主动”双重校验机制。

### 5.1 被动校验 (Client-Side Validation)
* **写入校验：** Client 推送 Chunk 时必须附带 Checksum 校验和。DataNode 收到后进行二次校验，与 Client 传来的期望值比对，只有校验通过后才落盘，并将校验和存入独立的元数据文件（如 `block_1024.meta`）。若校验不通过则 DataNode 拒绝写入并向上游返回异常。
* **读取拦截：** Client 拉取数据时，DataNode 会同时返回数据和 `.meta` 中的 Checksum。Client 比对失败后，抛弃该数据，向 NameNode 调用 `ReportBadBlock`，并无缝切换至其他副本。

### 5.2 主动防御 (DataNode Block Scanner)
* **后台巡检：** 为了防止冷数据长期不被读取而损坏，DataNode 内部运行将一个低优先级的异步任务 `Block Scanner`（块扫描器）。
* **工作流：** 扫描器在节点 I/O 空闲时，循环读取本地磁盘上的每一个 Block，重新计算 CRC32 并与 `.meta` 文件比对。一旦发现损坏，DataNode 主动向 NameNode 宣告该块失效，NameNode 会提前启动 `ReplicateBlock` 流程，在损坏蔓延前完成自我修复。

---

## 6. NameNode 容错与演进 (SPOF Constraints)

在 RDFS 的架构基线中，控制流高度集中：

* **DataNode 宕机是完全容错的。**
* **NameNode 宕机是致命的单点故障 (Single Point of Failure)。**

如果 NameNode 进程崩溃，只要其挂载的持久化存储（`FsImage` 目录快照 和 `EditLog` 增量操作日志）安全，重启后可在几分钟内恢复全量内存目录树。

### 6.1 重启时序与安全模式 (Safe Mode)
NameNode 崩溃重启后，绝对不能立即恢复对外服务。因为根据持久化铁律，本地磁盘上 **不包含** Block 的物理节点位置信息。此时必须依赖 **安全模式 (Safe Mode)** 来统筹全网 DataNode 的注册时序：
1. **本地状态重建：** NameNode 进程启动，优先加载本地的 `FsImage` 和 `EditLog`，在内存中重建并发目录树、恢复 ID 发号器并加载 MasterKey。
2. **进入安全模式：** 内存树恢复后，NameNode 启动 RPC 服务，但强制进入并锁定为安全模式。在此期间，**严禁任何 Client 发起元数据写操作（如 Create, Delete, AllocateBlock）**，所有写请求直接返回 `Unavailable` 错误。
3. **接受节点汇报：** NameNode 被动等待全网幸存的 DataNode 重新建立连接。DataNode 注册并发送全量 `BlockReport`。NameNode 据此在内存中重新拼接出“逻辑 Block -> 物理 DataNode”的路由表，并执行世代版本号 (GenStamp) 的冲突裁决。
4. **退出安全模式：** 当 NameNode 统计到全网绝大多数的 Block 都已至少被一个健康的 DataNode 认领后，自动解除安全模式锁定，集群全面恢复读写服务。

*(注：工业级的最终演进方案，是引入 Standby NameNode 配合基于 Raft 协议的 Quorum Journal Manager (QJM) 来实现高可用 HA 切换。)*