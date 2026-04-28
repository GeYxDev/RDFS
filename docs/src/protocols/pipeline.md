# 核心工作流：读取与写入 (Read/Write Pipeline)

本章节详细阐述 RDFS 中文件数据的流转过程。RDFS 的核心设计原则是 **“控制流与数据流分离”**：NameNode 仅负责轻量级的元数据调度（控制流），所有海量的文件数据传输（数据流）均由 Client 直接与 DataNode 进行交互。

---

## 1. 写入数据流 (Write Pipeline)

写入流程是分布式存储中最复杂的一环，因为它不仅要将数据切块，还要保证数据在多个节点间的副本一致性。RDFS 采用 **流水线复制 (Pipeline Replication)** 机制，并结合了严密的租约与版本控制。

### 1.1 核心步骤解析
假设 Client 要写入一个 100MB 的文件（默认 Block 大小为 64MB，因此将被分为 Block 1 [64MB] 和 Block 2 [36MB]）。
1. **获取租约与创建元数据 (Create)：**
   Client 生成全局唯一的 `client_id`，向 NameNode 发送 `CreateFile` 请求。NameNode 检查合法性后，在内存中创建处于 `UNDER_CONSTRUCTION` 状态的文件，并将该文件的写锁（租约）赋予该客户端。
2. **申请数据块与管线 (Allocate)：**
   Client 准备写入 Block 1，向 NameNode 发送 `AllocateBlock`。NameNode 生成一个全局唯一的 `block_id`、一个初始世代版本号 `gen_stamp`，根据负载均衡策略挑选 3 个健康的 DataNode（假设为 DN1, DN2, DN3），并使用当前活跃的 MasterKey 签发一个具备写入权限 (WRITE) 的 Block Token 一并返回给 Client。
3. **流式推送与管线建立 (Stream Pipeline)：**
   Client 与最近的 **DN1** 建立双向 gRPC Stream，利用 `oneof` 和双向异步特性进行高效传输。
    * **首包 (WriteHeader)：** Client 发送元数据，携带目标管线 `[DN2, DN3]`。DN1 收到后，主动与 DN2 建流，并将目标剥离为 `[DN3]` 转发；DN2 再与 DN3 建流。
    * **后续包 (WriteChunk)：** Client 将 64MB 的数据切分为 64KB 的 Chunk，附带 `checksum` 持续推入。DN1 收到 Chunk 边落盘、边向 DN2 转发，DN2 边落盘、边向 DN3 转发；DN2 在完成自身落盘并收到 DN3 落盘确认后向 DN1 返回落盘确认，DN1 在完成本地落盘并收到 DN2 确认后，立即通过该双向流向 Client 回传一个 ACK。
    * *(重要机制)：* RDFS 依赖 gRPC/HTTP2 底层的 **背压 (Backpressure)** 机制进行流控。如果 DN3 磁盘写入慢，会导致 DN2 接收缓冲填满，进而反压 DN1，最终让 Client 的发送函数自动挂起，从而避免全网内存溢出。当 64MB 全部发送完毕后，Client 发起 `CloseSend`，各级 DataNode 收到 EOF 信号后执行落盘 (`fsync`)，并向上游返回最终的写入字节数。
4. **物理层确认 (BlockReceived)：**
   当某个 DataNode 成功将该 Block 完全刷入物理磁盘后，会立即异步向 NameNode 发送 `BlockReceived` RPC，告知中心节点该块已安全落盘。
5. **循环与状态提交 (Commit)：**
   Block 1 结束后，Client 继续申请 Block 2。当所有数据写完，Client **必须调用** `CompleteFile`。NameNode 校验数据长度无误后，释放该 Client 的租约，将文件状态正式标记为 `ACTIVE`，允许全局读取。

### 1.2 异常管线恢复 (Pipeline Recovery)
如果在流水线传输中途，某个节点（如 DN2）突然宕机或断网，RDFS 依靠以下机制实现无缝断点续传：
1. **中断与暂停：** DN1 发现向 DN2 转发超时，断开连接并向上游 Client 抛出 gRPC 异常。Client 暂停发送新数据。
2. **剔除与版本升级：** Client 向 NameNode 发起重新分配管线请求，NameNode 将宕机的 DN2 从副本列表中踢出，并将该 Block 的 `gen_stamp` 升级，防止 DN2 假死恢复后其残留的旧数据污染读取。
3. **长度对齐：** Client 与幸存的 DN1、DN3 沟通，获取它们各自实际安全落盘的字节数，取**最小值**。随后 Client 再次下发截断指令，要求节点剔除超出该最小值的脏数据，并将本地磁盘块的 `gen_stamp` 更新同步至最新版本。
4. **降级续传：** Client 构建新的管线 `Client -> DN1 -> DN3`，从对齐的位置继续发送剩余的 Chunk。
5. **事后补偿：** 缺少的第三个副本将在后续 NameNode 处理心跳时，通过 `DataNodeCommand` 指挥集群后台自动补齐。

---

## 2. 读取数据流 (Read Path)

与写入相比，读取流程相对简单得多。它的核心目标是“就近读取”和“并发拉取”。

### 2.1 核心步骤解析
1. **获取块定位 (Get Locations)：**
   Client 向 NameNode 发送 `GetFileInfo` 请求。NameNode 返回该文件包含的所有 Block 列表，以及每个 Block 对应的 3 个 DataNode 地址。列表中 DataNode 的顺序通常已根据网络拓扑距离进行了优化排序。
2. **并行读取策略 (Parallel Read)：**
   Client 拿到完整的定位图后，可以按顺序读取，也可以在本地开启多线程并行连接不同的 DataNode 拉取不同的 Block，以充分利用网络带宽。
3. **流式拉取与校验 (Fetch & Validate)：**
   Client 向目标 DataNode 发起 `ReadBlock` Stream。DataNode 将数据以 Chunk 形式源源不断推回，每个 Chunk 附带一个 `checksum`。Client 在内存中实时计算接收数据的校验和进行比对。
4. **坏块举报与容错重试 (Bad Block Report & Retry)：**
   * 如果网络断开，Client 自动尝试该 Block 的第二个备用 DataNode。
   * 如果读取成功但 **校验和不匹配**（说明磁盘静默损坏），Client 会立即向 NameNode 发起 `ReportBadBlock` 请求举报该节点，丢弃脏数据，并无缝切换到其他副本节点继续读取。

---

## 3. 工程实现关键挑战

在具体用 Rust 实现上述 Pipeline 时，需要特别关注以下几点：

* **全双工 Stream 处理：** 在 Write Pipeline 中，DataNode 必须同时处理 **上游的读流** 和 **下游的写流**，可使用 `tokio::sync::mpsc` channel 做一层内存缓冲解耦。
* **高效 oneof 处理：** gRPC 传输的首包 Header 和后续 Chunk 在 Rust 中会被 Tonic 映射为 `Enum` 匹配 (`match payload`)。需要在 Stream 消费侧仔细处理状态机，确保只有第一个包触发路由逻辑。
* **内存零拷贝 (Zero-copy)：** 尽量利用 `bytes::Bytes` 类型传递数据碎片，避免在 gRPC 序列化/反序列化和本地磁盘 I/O 之间发生频繁的内存拷贝。
* **死锁与超时：** Pipeline 中任何一个节点卡死都会导致全局阻塞。所有的网络操作必须使用 `tokio::time::timeout` 进行包裹。