# RDFS 通信协议规范 (RPC Protocol Specification)

本规范定义了 RDFS 系统中所有组件（Client, NameNode, DataNode）之间的网络通信接口。
系统全局采用 **gRPC** 框架作为底层通信协议，使用 **Protocol Buffers (v3)** 进行数据序列化。

---

## 1. 全局约定

* **字节序与编码：** 所有字符串默认采用 UTF-8 编码。
* **流式传输 (Streaming)：** 凡是涉及实际文件数据（`bytes`）读写的接口，必须使用 gRPC 的 Stream 机制，严禁在单个一元 RPC (Unary RPC) 中打包超大字节数组。
* **默认端口：**
    * NameNode RPC 服务：`9000`
    * DataNode RPC 服务：动态分配，向 NameNode 注册时上报。

---

## 2. 通用数据结构 (Common Messages)

在不同的 RPC 接口中，会频繁复用以下核心数据结构：

### 2.1 DataNodeInfo (数据节点物理信息)
描述一个 DataNode 如何被网络中的其他节点寻址。
* `node_id` (string)：节点的全局唯一标识符（使用 UUIDv4）。
* `ip_addr` (string)：节点的 IPv4/IPv6 地址。
* `port` (uint32)：节点提供数据读写 gRPC 服务的端口号。

### 2.2 BlockInfo (数据块元数据)
描述一个独立的数据切片。
* `block_id` (uint64)：块的全局唯一 ID（由 NameNode 单调递增分配）。
* `length` (uint64)：块当前的实际大小（单位：Bytes）。
* `gen_stamp` (uint64)：块的世代版本号（解决网络脑裂和陈旧写入问题）。

### 2.3 LocatedBlock (带位置的块信息)
将逻辑上的 Block 与物理上的 DataNode 绑定，客户端拿到该结构后可知去哪里读写数据。
* `block` (BlockInfo)：数据块本身的属性。
* `locations` (List<DataNodeInfo>)：存储该数据块副本的所有可用 DataNode 列表（按网络距离或负载排序）。
* `access_token` (Token)：由 NameNode 签发的安全访问令牌，客户端凭此向 DataNode 证明读/写权限。

### 2.4 DataNodeCommand (元数据节点下发指令)
NameNode 通过心跳响应向 DataNode 下发的管理指令。
* `action` (enum)：指令类型（如 DELETE_BLOCK, REPLICATE_BLOCK 等）。
* `target_block` (BlockInfo)：需要操作的数据块。
* `dest_nodes` (List<DataNodeInfo>)：若为复制指令，此为目标节点列表。

### 2.5 Token (安全访问令牌)
由 NameNode 签发，用于 DataNode 去中心化验证客户端权限的凭证。
* `identifier` (bytes)：包含 `block_id`、`client_id`、权限位（读/写）、过期时间及 `key_id` 等要素的序列化明文。
* `signature` (bytes)：NameNode 使用 MasterKey 对 `identifier` 计算出的 HMAC-SHA256 签名结果。

---

## 3. NameNode 交互规范 (NameNode's API)

NameNode 只处理元数据，所有交互均为轻量级的一元 RPC。

### 3.1 客户端 -> NameNode (Client to NameNode)

#### 3.1.1 `CreateFile` (创建文件)
* **场景：** 客户端请求在系统中创建一个新文件。
* **Request：**
    * `path` (string)：文件的绝对路径（例如：`/data/logs/2026.log`）。
    * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
    * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否创建成功。

#### 3.1.2 `AllocateBlock` (申请数据块)
* **场景：** 客户端在写入文件时，发现当前块已写满（达到 64MB），向 NameNode 申请一个新的物理块和存储节点。
* **Request：**
    * `path` (string)：文件路径。
    * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
    * `located_block` (LocatedBlock)：返回新生成的 BlockID，以及系统分配的 3 个 DataNode 地址。

#### 3.1.3 `CompleteFile` (提交文件写入结果)
* **场景：** 客户端完成文件所有数据块的流式写入后，告知 NameNode 文件已关闭，此时 NameNode 会将文件状态从 UNDER_CONSTRUCTION 转为 ACTIVE。
* **Request：**
    * `path` (string)：文件路径。
    * `total_size` (uint64)：文件的最终总字节数。
    * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
    * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否提交成功。

#### 3.1.4 `GetFileInfo` (获取文件定位)
* **场景：** 客户端想要读取某个文件，需要先获取该文件的所有块清单及物理位置。
* **Request：**
    * `path` (string)：文件路径。
* **Response：**
    * `file_length` (uint64)：文件总大小。
    * `file_status` (enum)：文件当前状态。若客户端不支持脏读，NameNode 可直接对非 ACTIVE 文件返回 gRPC 错误。
    * `blocks` (List<LocatedBlock>)：该文件包含的所有数据块及其物理位置（按网络拓扑距离优化排序）。

#### 3.1.5 `RenewLease` (租约续约)
* **场景：** 客户端持有任何文件的写锁时，必须定期向 NameNode 发送心跳续约。
* **Request：**
    * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
    * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否续约成功。

#### 3.1.6 `ReportBadBlock` (坏块举报)
* **场景：** 客户端在流式读取 (`ReadBlock`) 时，对接收到的数据碎片进行校验。若发现实际数据与返回的校验和不匹配，客户端向 NameNode 举报该故障节点，并自行去其他副本节点读取。
* **Request：**
    * `block_id` (uint64)：损坏的数据块 ID。
    * `node_id` (string)：返回损坏数据的 DataNode ID。
    * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
    * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否举报成功。

### 3.2 DataNode -> NameNode (DataNode to NameNode)

#### 3.2.1 `Heartbeat` (心跳包)
* **场景：** DataNode 启动后，每隔固定时间（如 3 秒）向 NameNode 报活。NameNode 若连续 10 秒未收到心跳，则判定该节点宕机。
* **Request：**
    * `node_info` (DataNodeInfo)：汇报自己的 IP 和端口。
    * `capacity` (uint64)：节点配置的总存储容量。
    * `used` (uint64)：节点当前已使用的容量。
* **Response：**
    * `commands` (List<DataNodeCommand>)：NameNode 返回的异步指令（例如：要求该节点删除某个废弃的 Block，或将某个 Block 复制给其他节点）。
    * `master_keys` (List<bytes>)：NameNode 下发的当前有效密钥（Active Key 与 Standby Key），供 DataNode 缓存在本地用于 Token 验签验证。

#### 3.2.2 `BlockReport` (块状态汇报)
* **场景：** DataNode 启动时（或每隔 1 小时），向 NameNode 汇报自己磁盘上真实存在的所有 BlockID 列表，用于 NameNode 校验内存状态与物理状态的最终一致性。
* **Request：**
    * `node_id` (string)：发起汇报的节点 ID。
    * `blocks` (List<BlockInfo>)：本地磁盘上所有完好的 Block ID 集合。
* **Response：**
    * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否汇报成功。

#### 3.2.3 `BlockReceived` (增量块汇报)
* **场景：** DataNode 成功接收并落盘一个新块后，立即异步通知 NameNode，无需等待全量汇报。
* **Request：**
    * `node_id` (string)：发起汇报的节点 ID。
    * `block` (BlockInfo)：已确认的块信息。
* **Response：**
    * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否汇报成功。

---

## 4. DataNode 交互规范 (DataNode's API)

专门用于处理海量底层数据的传输，严格使用 **gRPC Stream**。

### 4.1 客户端 -> DataNode (Client to DataNode)

#### 4.1.1 `WriteBlock` (流式写入)
* **场景：** 客户端拿到 `AllocateBlock` 分配的地址后，向指定的 DataNode 建立双向流。为极致优化网络带宽，采用“首包传元数据，后续包传纯数据”的机制，并由 DataNode 自动向底层推进流水线 (Pipeline) 复制。客户端推送数据的同时异步接收来自 Pipeline 的确认（ACK）。
* **Request (Stream)：** 客户端将连续发送多个数据包。包的载荷 (Payload) 须根据发送顺序，在以下两种结构中二选一：
  * **[首包结构]** `WriteHeader` 仅作为流的第一个包发送：
    * `block_id` (uint64)：目标块 ID。
    * `pipeline_targets` (List<DataNodeInfo>)：流水线下游目标节点列表。例如，客户端传入 `[DN2, DN3]`；当 DN1 向 DN2 转发时，该列表应剥离自身，更新为 `[DN3]`。
    * `checksum_type` (uint32)：数据校验算法类型（例如定义 `1` 为 CRC32）。
    * `access_token` (Token)：客户端提交的写入权限凭证。DataNode 收到首包后必须立即验签，失败则直接断开 Stream。
  * **[后续包结构]** `WriteChunk` 从第二个包开始循环发送，直至流结束：
    * `chunk_data` (bytes)：数据碎片。默认客户端以 64KB 为一个 chunk 进行切片发送，避免内存溢出。
    * `checksum` (uint32)：针对当前这一包 chunk 数据的校验和。
* **Response (Stream)：** DataNode 异步返回确认包 `WriteChunkResponse`。
    * `offset` (uint64)：已成功写入并确认的偏移量。
    * `status` (enum)：写入状态（如 SUCCESS, CHECKSUM_ERROR, PIPELINE_ERROR）。
* **ACK 返回时机：** 为了保证多副本的一致性与高性能，采用“逆流确认”机制。
    * 逐级转发：DN1 收到 Chunk 后，在异步写入本地磁盘的同时，立即将其转发给 DN2。
    * 下游优先确认：DN1 只有在收到来自下游（DN2）针对该 Chunk 的成功 ACK 后，且自身本地写入也完成后，才会向上游（Client）发送该 Chunk 的 `WriteChunkResponse`。
    * 最终一致性：只有 Client 收到 ACK，才代表该 Chunk 已在 Pipeline 中所有指定的 DataNode 上安全落盘。

#### 4.1.2 `ReadBlock` (流式读取)
* **场景：** 客户端请求读取某个具体的物理数据块。
* **Request (Unary)：**
    * `block_id` (uint64)：目标块 ID。
    * `offset` (uint64)：起始读取位置。支持断点续传和随机读取。
    * `length` (uint64)：期望读取的数据长度。
    * `access_token` (Token)：客户端提交的读取权限凭证。DataNode 验签通过后才允许建立返回数据的 Stream。
* **Response (Stream)：** DataNode 连续返回数据碎片。
    * `chunk_data` (bytes)：实际的数据碎片（如 64KB/包）。
    * `checksum` (uint32)：针对该碎片的校验和。