# 简介与目标 (Introduction)

欢迎来到 **RDFS (Rust Distributed File System)** 的核心架构设计文档库。

这份文档不仅是 RDFS 项目的“说明书”，更是我们开发过程中的“架构指南针”。在编写任何一行复杂的分布式核心逻辑之前，我们都会先在这里明确数据流向、RPC 接口以及内存状态。这份文档会随着代码的迭代而持续更新，始终保持与源码的同步。

## 1. 项目背景与初衷

分布式文件系统（DFS）是现代大数据和云原生架构的基石。HDFS（Hadoop Distributed File System）作为业界的经典标杆，其设计思想影响了无数后来的存储系统。

本项目（RDFS）旨在用 Rust 语言从零复刻一个简化版的 HDFS。选择 Rust 的原因在于：
* **内存安全：** 在底层存储系统中，指针错误或内存泄漏是致命的。Rust 的所有权机制能在编译期杜绝这些问题。
* **无数据竞争的并发：** 元数据节点需要处理海量的并发请求，Rust 的 `Send` 和 `Sync` 特型让多线程编程变得前所未有的安全。
* **极致性能：** 极低的资源占用和无 GC（垃圾回收）带来的稳定延迟，非常适合数据密集型应用。

## 2. 核心设计目标 (Goals)

在 RDFS 的 V1.0 架构设计中，我们追求以下几个核心目标：

1. **架构极简主义与职责隔离 (Simplicity & Isolation)**
   * 采用 **中心化单 Master 架构**（单一 NameNode 负责控制流，配合 Secondary NameNode 卸载合并压力）。
   * 初期不引入 Paxos/Raft 等复杂的分布式共识算法，将精力集中在最核心的分布式文件切片、**双向流水线（Bidi-Stream Pipeline）** 传输上。
2. **极致并发与高吞吐量读写 (High Throughput)**
   * 专为 **大文件**（GB/TB级别）存储设计，采用大文件块（默认 64MB）进行存储，以榨干网络带宽并减少元数据开销。
   * 支持客户端与数据节点（DataNode）之间的直接数据流传输，NameNode 仅负责元数据调度，绝对不参与数据流转。配合底层的 **背压流控（Backpressure）** 与滑动窗口，实现无阻塞的高吞吐写入。
   * 引入基于 **Actor 状态机组合 DashMap 的读写分离架构**，构建读并发、写串行的目录树，实现极高的并发写入吞吐。
3. **高可用与无缝容错 (Fault Tolerance)**
   * 秉持“硬件总是不可靠”的假设。支持 3 副本机制与后台主动坏块扫描。
   * 具备强大的 **异常管线恢复 (Pipeline Recovery)** 与宕机自动重平衡机制，对上层业务完全透明。
   * 强制推行 **端到端 Checksum 实时校验**，彻底防御网络层的静默数据损坏。
4. **工业级数据安全底座 (Security & Access Control)**
   * 引入基于 HMAC-SHA256 的 **Block Token 鉴权机制**。
   * 杜绝恶意客户端绕过 NameNode 直连窃取数据，实现数据平面的绝对安全。

## 3. 明确不做的特性 (Non-Goals)

为了防止功能蔓延，明确系统的边界同样重要。以下功能在 RDFS V1.0 中 **不予支持**：

* **不支持小文件优化：** 大量 KB 级别的小文件会迅速耗尽 NameNode 的内存，这违背了本系统为大数据存储设计的初衷。
* **不支持文件的随机修改：** 文件不支持在中间任意位置进行随机写入或部分覆盖，仅支持末尾追加（Append）和创建时整体覆盖（Overwrite）。
* **不支持跨数据中心/机架感知：** 初期假设所有的节点都在同一个局域网的高速网络内。

## 4. RDFS 架构图纸导航

RDFS 拥有一套极其严密的架构设计规范。如果你是新加入的开发者，请务必按照以下脉络阅读，它们构成了整个系统的“真理之源”：

### 🗺️ 第一阶段：宏观蓝图
* **[核心架构白皮书 (architecture.md)](architecture.md)：** 了解四大物理角色（NN, SNN, DN, Client）与整体流转。

### 📡 第二阶段：三大核心规范
* **[通信协议规范 (rpc.md)](./protocols/rpc.md)：** 了解全局 gRPC 接口、增量汇报与流式调用的控制平面。
* **[核心工作流：读取与写入 (pipeline.md)](./protocols/pipeline.md)：** 了解最硬核的背压流控、oneof 切片与断点续传。
* **[容错与副本恢复机制 (recovery.md)](./protocols/recovery.md)：** 了解超时状态机、世代版本号防脑裂与主动防御。
* **[安全与访问控制 (auth.md)](./protocols/auth.md)：** 了解 Block Token 签发、HMAC 验签与 MasterKey 动态轮转机制。

### ⚙️ 第三阶段：组件内部深度设计
* **[NameNode 内存模型 (inode.md)](./namenode/inode.md)：** 了解如何利用 Rust 的扁平化哈希表与带载荷枚举充分利用内存。
* **[NameNode 持久化 (persistence.md)](./namenode/persistence.md)：** 了解 TxID 与 Secondary NameNode 的 Checkpoint 机制。
* **[DataNode 本地存储 (storage.md)](./datanode/storage.md)：** 了解极致的零拷贝物理磁盘布局与 O(1) 启动校验。
* **[DataNode 心跳机制 (heartbeat.md)](./datanode/heartbeat.md)：** 了解基于 mpsc 通道的非阻塞状态机。

### 🚀 第四阶段：工程落地与开发
* **[客户端 SDK 接口规范 (api.md)](./client/api.md)：** 了解如何将复杂的分布式管线封装为极简的 API。
* **[开发环境与构建指南 (guide.md)](./development/guide.md)：** 从零配置环境并运行本地伪分布式集群。
* **[里程碑与未来计划 (milestones.md)](./development/milestones.md)：** 了解 V1.0 的冲刺任务与未来的高阶演进方向。