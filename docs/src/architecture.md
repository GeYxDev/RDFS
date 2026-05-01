# 系统整体架构 (Architecture Overview)

## 1. 系统概述

RDFS 是一个基于 Rust 与 Tokio 异步生态构建的、深度借鉴 HDFS 设计哲学的工业级分布式文件系统。

* **核心目标：** 支撑海量大文件的高吞吐量分布式存储、提供多副本容错与静默数据损坏自愈能力。
* **关键特性：** 支持流式管线写入 (Pipeline)、支持文件追加写 (Append)、基于世代版本号 (Generation Stamp) 彻底杜绝分布式脑裂，和基于 HMAC 的 Block Token 机制实现数据节点级的细粒度访问控制。
* **非目标：** 不支持低延迟、高并发的小文件随机修改操作（仅支持追加与覆盖）。

## 2. 核心架构与物理角色

系统采用 **高内聚的中心化 Master 架构**，控制流与数据流严格物理隔离。包含四大核心组件：

* **NameNode (元数据主节点)：**
  * **职责：** 系统的“大脑”。全内存管理命名空间（目录树）、处理文件到数据块（Block）的逻辑映射、统筹租约分配与节点状态机。同时作为集群的“发证机关”，负责维护 MasterKey 并签发带有签名的 Block Token。
  * **铁律：** 绝对不触碰实际文件数据流。
* **Secondary NameNode (元数据合并节点)：**
  * **职责：** 系统的“打杂节点”。专门在后台从主节点拉取 EditLog (写前日志) 和 FsImage (内存快照) 进行合并，防止主节点 OOM 并加速集群重启。
* **DataNode (数据存储节点)：**
  * **职责：** 系统的“苦力”。负责数据块（默认 64MB）在本地磁盘的物理存储，执行 NameNode 下发的容错指令（删除/复制），并在后台主动扫描坏块。在数据读写前，负责在本地校验 Client 携带的 Block Token 的合法性，拒绝未授权访问。
* **Client (客户端 SDK / CLI)：**
  * **职责：** 系统的“大门”。将底层的 RPC 交互、大文件智能切片、端到端 Checksum 校验、复杂的 **异常管线恢复 (Pipeline Recovery)** 状态机、以及 Token 自动透传，封装为极简的异步流式 API 暴露给用户。

## 3. 元数据与内存模型 (NameNode Side)

NameNode 的核心是为应对海量高并发设计的内存模型。

* **扁平化目录树 (Arena Tree)：** 抛弃容易死锁的嵌套指针锁，使用 `DashMap` 实现，通过将 Inode 映射到多个独立加锁的分片（Shards）中，大幅降低高并发写入时的锁竞争（Lock Contention），并利用“自顶向下”的加锁顺序机制预防死锁。
* **内存零浪费：** 采用 Rust 的带载荷枚举 (`Enum`)，将文件状态（如 `Active`, `UnderConstruction`）和目录属性物理隔离。
* **防脑裂引擎：** 每个 Block 不仅有 `block_id`，还强制绑定一个单调递增的 **世代版本号 (`gen_stamp`)**。

## 4. 核心工作流设计

### 4.1 写入数据流 (Write Pipeline)
1. **获取租约：** Client 向 NameNode 申请 `CreateFile` 或 `Append`，获得文件的独占写锁（Lease），文件进入构建中状态。
2. **分配管线：** Client 发起 `AllocateBlock`，NameNode 分配全新的 `block_id`、`gen_stamp`，使用当前 MasterKey 生成包含写入权限的 Block Token，并按策略返回 3 个健康的 DataNode 地址及该 Token。
3. **流水线推送与背压：** Client 将数据切分为 64KB 的 Chunk，与首个 DataNode 建立双向 gRPC Stream。建立连接的首包（Metadata）中必须携带上述 Block Token，DataNode 校验通过后才允许建立流水线。数据如同流水线般 `Client -> DN1 -> DN2 -> DN3` 边写边转。全程依赖 HTTP/2 **背压 (Backpressure)** 进行零开销流控。
4. **容错与状态提交：** DataNode 落盘前必须强制进行 Checksum 二次校验。若中途节点宕机，Client 自动剔除坏节点、升级版本号并降级续传。写完后调用 `CompleteFile` 正式闭环。

### 4.2 读取数据流 (Read Path)
1. **获取元数据：** Client 发起 `GetFileInfo` 请求，获取该文件所有 Block 的位置清单及安全访问 Token。
2. **并行拉取：** Client 在本地开启多任务，就近连接对应的 DataNode 建立 Stream 流式拉取数据。请求首包必须提交 Token，DataNode 在本地计算 HMAC 验证过期时间及读取权限后，开始流式下发数据。
3. **端到端校验：** DataNode 返回数据的同时返回 Checksum，由 Client 的 CPU 实时进行校验比对。一旦发现损坏，即刻抛弃脏数据，向 NameNode 举报 (`ReportBadBlock`) 并静默切换备用节点。

### 4.3 安全控制机制 (Access Token)
1. **结构设计：** 一个有效的 Block Token 包含以下核心要素：`block_id`（限定操作对象）、`expiry_date`（过期时间）、`access_mode`（读/写/复制权限）。
2. **签发流程：** NameNode 内部维护一个周期性轮转的 `MasterKey`。当 Client 申请分配或查询 Block 时，NameNode 将 Payload 与 `MasterKey` 进行 HMAC 签名，并将组合后的 Token 字符串返回给 Client。
3. **去中心化校验：** 为了避免每次读写都向 NameNode 发起鉴权请求（避免 RPC 风暴），DataNode 必须具备本地校验能力：
    * **密钥同步：** NameNode 通过定期的心跳回执 (Heartbeat Response)，将当前的 `MasterKey` 下发给所有存活的 DataNode。
    * **本地验签：** DataNode 收到 Client 的流式读写请求时，解析 Token Payload，使用内存中缓存的 `MasterKey` 重新计算 HMAC。若签名匹配且时间未过期，则放行 I/O 操作。

## 5. 网络通信协议 (RPC)

全栈采用 **gRPC** (基于 `tonic` 和 `prost`)。
* **控制平面 (NameNode 交互)：** 极度精简的一元 RPC (Unary RPC)，用于心跳、状态汇报和元数据查询。
* **数据平面 (DataNode 交互)：** 严格使用流式 RPC (Streaming RPC)，配合 `oneof` 首包元数据机制，最大化榨干网络带宽。