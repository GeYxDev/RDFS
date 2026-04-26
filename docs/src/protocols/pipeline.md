# 核心工作流：读取与写入 (Read/Write Pipeline)

本章节详细阐述 RDFS 中文件数据的流转过程。RDFS 的核心设计原则是 **“控制流与数据流分离”**：NameNode 仅负责轻量级的元数据调度（控制流），所有海量的文件数据传输（数据流）均由 Client 直接与 DataNode 进行交互。

---

## 1. 写入数据流 (Write Pipeline)

写入流程是分布式存储中最复杂的一环，因为它不仅要将数据切块，还要保证数据在多个节点间的副本一致性。RDFS 采用**流水线复制 (Pipeline Replication)** 机制。

### 1.1 核心步骤解析

假设 Client 要写入一个 100MB 的文件（默认 Block 大小为 64MB，因此将被分为 Block 1 [64MB] 和 Block 2 [36MB]）。

1. **创建元数据 (Create):** Client 向 NameNode 发送 `CreateFile` 请求。NameNode 检查路径是否合法且无重名后，在内存的 Inode Tree 中创建一个空文件节点。
2. **申请数据块 (Allocate):**
   Client 准备写入 Block 1，向 NameNode 发送 `AllocateBlock`。NameNode 生成一个全局唯一的 `BlockId`，并根据负载均衡策略挑选 3 个健康的 DataNode（假设为 DN1, DN2, DN3），将这 3 个节点的地址构造成一个管道列表返回给 Client。
3. **建立数据流水线 (Setup Pipeline):**
    * Client 与最近的 **DN1** 建立 gRPC Stream 连接。
    * **DN1** 收到连接后，作为代理，主动与 **DN2** 建立连接。
    * **DN2** 收到连接后，再与 **DN3** 建立连接。
      至此，一条 `Client -> DN1 -> DN2 -> DN3` 的单向数据通道建立完毕。
4. **流式推送数据 (Stream Data):**
   Client 将 64MB 的 Block 1 切分为多个 64KB 的 Chunk（数据包），源源不断地推入流水线。
    * DN1 收到 Chunk 1，**边将其写入本地磁盘，边将其转发** 给 DN2。
    * DN2 收到 Chunk 1，边落盘边转发给 DN3。
    * （这种边写边发的机制，极大降低了 3 副本写入的总体延迟）。
5. **确认回传 (Ack Propagation):**
   当 DN3 成功落盘一个 Chunk 后，向上游 DN2 发送 Ack（确认包）；DN2 收到 Ack 后加上自己的 Ack 发给 DN1；DN1 最终将确认发给 Client。
6. **循环与关流:**
   Block 1 写完后，Client 关闭当前 Pipeline，然后为 Block 2 重新向 NameNode 申请新的 `AllocateBlock`，重复上述步骤。

---

## 2. 读取数据流 (Read Path)

与写入相比，读取流程相对简单得多。它的核心目标是“就近读取”和“并发拉取”。

### 2.1 核心步骤解析

1. **获取块定位 (Get Locations):**
   Client 向 NameNode 发送 `GetFileInfo` 请求。NameNode 返回该文件包含的所有 Block 列表，以及每个 Block 对应的 3 个 DataNode 地址。
    * *优化策略：* NameNode 在返回 DataNode 列表时，通常会根据拓扑距离（或随机打乱）进行排序，把最合适的节点放在列表首位。
2. **并行读取策略:**
   Client 拿到完整的定位图后，可以按顺序挨个 Block 读，也可以开启多线程（在 Rust 中直接 `tokio::spawn`）并行去不同的 DataNode 拉取不同的 Block。
3. **流式拉取数据 (Fetch Data):**
   针对某一个具体的 Block，Client 直接与该 Block 的第一个 DataNode 建立 gRPC Stream 链接，发送 `ReadBlock(block_id, offset, length)` 请求。DataNode 将磁盘上的数据按 Chunk 源源不断推回给 Client。
4. **容错重试 (Read Failure Handling):**
   如果在拉取过程中与当前的 DataNode 断开连接，或者校验和（Checksum）不匹配，Client 会立刻丢弃该节点的连接，自动尝试该 Block 的第二个备用 DataNode，对用户侧完全透明。

---

## 3. Rust 实现上的关键挑战

在具体用 Rust 实现上述 Pipeline 时，需要特别关注以下几点：

* **全双工 Stream 处理：** 在 Write Pipeline 中，DataNode 必须同时处理**上游的读流**和**下游的写流**，建议使用 `tokio::sync::mpsc` channel 做一层内存缓冲解耦。
* **内存零拷贝 (Zero-copy)：** 尽量利用 `bytes::Bytes` 类型传递数据碎片，避免在 gRPC 序列化/反序列化和本地磁盘 I/O 之间发生频繁的内存拷贝。
* **死锁与超时：** Pipeline 中任何一个节点卡死都会导致全局阻塞。所有的网络操作必须使用 `tokio::time::timeout` 进行包裹。