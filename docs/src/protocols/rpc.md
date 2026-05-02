# RDFS 通信协议规范 (RPC Protocol Specification)

本规范定义了 RDFS 系统中所有组件（Client, NameNode, DataNode）之间的网络通信接口。
系统全局采用 **gRPC** 框架作为底层通信协议，使用 **Protocol Buffers (v3)** 进行数据序列化。

---

## 1. 全局约定

* **字节序与编码：** 所有字符串默认采用 UTF-8 编码。
* **流式传输 (Streaming)：** 凡是涉及实际文件数据（`bytes`）读写的接口，必须使用 gRPC 的 Stream 机制，严禁在单个一元 RPC (Unary RPC) 中打包超大字节数组。
* **默认端口：**
  * NameNode RPC 服务：`9000`
  * DataNode RPC 服务：生产环境采用动态端口分配，向 NameNode 注册时上报，本地测试和调试环境可指定固定端口。

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
* `action` (enum)：指令类型，根据不同指令携带不同参数：
  * `DELETE_BLOCK`：要求节点物理删除指定 Block。
    * `target_block` (BlockInfo)：需要操作的数据块。
  * `REPLICATE_BLOCK`：要求节点将指定 Block 复制给目标节点。
    * `target_block` (BlockInfo)：需要操作的数据块。
    * `dest_nodes` (List<DataNodeInfo>)：若为复制指令，此为目标节点列表。
  * `BLOCK_RECOVERY`：租约恢复指令，指定本节点为 Primary DN，协调副本对齐。
    * `block_id` (uint64)：需要恢复的数据块 ID。
    * `new_gen_stamp` (uint64)：NameNode 为本次恢复生成的世代版本号。
    * `replicas` (List<DataNodeInfo>)：持有该 Block 副本的所有 DataNode 列表（含 Primary 自身）。

### 2.5 Token (安全访问令牌)
由 NameNode 签发，用于 DataNode 去中心化验证客户端权限的凭证。
* `identifier` (bytes)：包含以下要素的序列化明文：
  * `block_id` (uint64)：授权操作的数据块 ID。
  * `gen_stamp` (uint64)：授权时该块当前的世代版本号（防止脑裂脏读/脏写）。
  * `client_id` (string)：申请该令牌的客户端 UUID。
  * `access_mode` (uint32)：权限位掩码（例如：`1`=READ, `2`=WRITE, `4`=COPY）。
  * `expiry_date` (int64)：令牌失效的 Unix 时间戳。
  * `key_id` (uint32)：签发该令牌所使用的 MasterKey 版本号。
* `signature` (bytes)：NameNode 使用 MasterKey 对 `identifier` 计算出的 HMAC-SHA256 签名结果。

---

## 3. NameNode 交互规范 (NameNode's API)

NameNode 只处理元数据，所有交互均为轻量级的一元 RPC。

### 3.1 客户端 -> NameNode (Client to NameNode)

#### 3.1.1 `CreateFile` (创建文件)
* **场景：** 客户端请求在系统中创建新文件。
* **Request：**
  * `path` (string)：文件的绝对路径（例如：`/data/logs/2026.log`）。
  * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否创建成功。

#### 3.1.2 `MakeDirectory` (创建目录)
* **场景：** 客户端请求在指定路径创建目录。
* **Request：**
  * `path` (string)：目录的绝对路径（例如：`/data/logs`）。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否创建成功。

#### 3.1.3 `AllocateBlock` (申请数据块)
* **场景：** 客户端在写入文件时，发现当前块已写满（达到 64MB），向 NameNode 申请一个新的物理块和存储节点。
* **Request：**
  * `path` (string)：文件路径。
  * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
  * `located_block` (LocatedBlock)：返回新生成的 BlockID，以及系统分配的 3 个 DataNode 地址。

#### 3.1.4 `CompleteFile` (提交文件写入结果)
* **场景：** 客户端完成文件所有数据块的流式写入后，告知 NameNode 文件已关闭，此时 NameNode 会将文件状态从 UNDER_CONSTRUCTION 转为 ACTIVE。
* **Request：**
  * `path` (string)：文件路径。
  * `total_size` (uint64)：文件的最终总字节数。
  * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否提交成功（若文件最后一个 Block 已确认副本数不足，NameNode 将拒绝提交并返回 FAILED_PRECONDITION，Client 应重试或上报异常）。

#### 3.1.5 `GetFileInfo` (获取文件定位)
* **场景：** 客户端想要读取某个文件，需要先获取该文件的所有块清单及物理位置。
* **Request：**
  * `path` (string)：文件路径。
* **Response：**
  * `file_length` (uint64)：文件总大小。
  * `file_status` (enum)：文件当前状态。若客户端不支持脏读，NameNode 可直接对非 ACTIVE 文件返回 gRPC 错误。
  * `blocks` (List<LocatedBlock>)：该文件包含的所有数据块及其物理位置（按网络拓扑距离优化排序）。
* **错误码：**
  * `NOT_FOUND`：路径不存在。
  * `PERMISSION_DENIED`：无权访问。
  * `FAILED_PRECONDITION`：文件处于 `UNDER_CONSTRUCTION` 状态且客户端要求强一致读。
  * `UNAVAILABLE`：请求的某个 Block 正处于 `UnderRecovery` 状态，客户端应稍后重试或读取其他副本。

#### 3.1.6 `RenewLease` (租约续约)
* **场景：** 客户端持有任何文件的写锁时，必须定期向 NameNode 发送心跳续约。
* **Request：**
  * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否续约成功。

#### 3.1.7 `ReportBadBlock` (坏块举报)
* **场景：** 客户端在流式读取 (`ReadBlock`) 时，对接收到的数据碎片进行校验。若发现实际数据与返回的校验和不匹配，客户端向 NameNode 举报该故障节点，并自行去其他副本节点读取。
* **Request：**
  * `block_id` (uint64)：损坏的数据块 ID。
  * `node_id` (string)：返回损坏数据的 DataNode ID。
  * `client_id` (string)：当前发起写入的客户端的全局唯一标识符（使用 UUID）。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否举报成功。

#### 3.1.8 `ReportFailedBlock` (管线故障上报)
* **场景：** 写入过程中 Pipeline 内某 DataNode 宕机，Client 暂停写入，向 NameNode 报告故障节点并请求恢复授权。
* **Request：**
  * `block_id` (uint64)：发生故障的数据块 ID。
  * `failed_node_id` (string)：宕机或不可达的 DataNode ID。
  * `client_id` (string)：当前客户端的唯一标识。
* **Response：**
  * `new_gen_stamp` (uint64)：NameNode 为隔离陈旧数据而升级的世代版本号。
  * `new_pipeline` (List<DataNodeInfo>)：剔除故障节点后的新管线列表。
  * `access_token` (Token)：携带新 `gen_stamp` 的写入权限 Token。

#### 3.1.9 `CompleteBlockRecovery` (块恢复完成上报)
* **场景：** Client 完成长度对齐、截断并更新幸存节点的 `gen_stamp` 后，通知 NameNode 最终状态，以便 NameNode 落盘并更新内存元数据。
* **Request：**
  * `block_id` (uint64)：恢复完成的块 ID。
  * `new_gen_stamp` (uint64)：对齐后生效的世代版本号。
  * `final_length` (uint64)：对齐后的块最终大小。
  * `client_id` (string)：当前客户端的唯一标识。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否更新成功。

#### 3.1.10 `RecoverLease` (租约恢复请求)
* **场景：** 新 Client 尝试打开一个处于 UNDER_CONSTRUCTION 状态的文件时，主动请求 NameNode 触发租约恢复。
* **语义：** 异步触发。NameNode 收到请求后立即返回（不等待恢复完成），在后台启动租约恢复流程。客户端应通过后续调用 `GetFileInfo` 轮询文件状态，直到 `file_status` 变为 ACTIVE，或收到文件不存在的错误（恢复失败被放弃）。
* **Request：**
  * `path` (string)：需要恢复的文件路径。
  * `client_id` (string)：发起请求的客户端标识。
* **Response：**
  * *(Empty)*：请求体为空。若文件已存在且未处于 UNDER_CONSTRUCTION，则直接返回成功（无需恢复）；若文件不存在或路径为目录，返回 NOT_FOUND；若 NameNode 成功将恢复任务加入后台队列，返回 OK；若系统繁忙无法接受新恢复任务，返回 RESOURCE_EXHAUSTED。

#### 3.1.11 `SetReplication` (修改副本因子)
* **场景：** 客户端修改指定文件的期望副本数。
* **Request：**
  * `path` (string)：目标文件路径。
  * `replication` (uint16)：新的副本数。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否修改成功。

### 3.2 DataNode -> NameNode (DataNode to NameNode)

#### 3.2.1 `Heartbeat` (心跳包)
* **场景：** DataNode 启动后，每隔固定时间（如 3 秒）向 NameNode 报活。NameNode 若连续 10 秒未收到心跳，则判定该节点宕机。
* **Request：**
  * `node_info` (DataNodeInfo)：汇报自己的 IP 和端口。
  * `capacity` (uint64)：节点配置的总存储容量。
  * `used` (uint64)：节点当前已使用的容量。
  * `request_full_keys` (bool)：可选字段，当 DataNode 检测到本地 Token 验签失败率超过阈值时可设置此字段，NameNode 将在本次心跳响应中强制下发完整的 Active/Standby 密钥列表。
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

#### 3.2.4 `CommitBlockRecovery` (块恢复完成上报)
* **场景：** Primary DataNode 完成某一 Block 的租约恢复（长度对齐、`gen_stamp` 更新）后，向 NameNode 上报最终结果。
* **Request：**
  * `block_id` (uint64)：恢复完成的块 ID。
  * `new_gen_stamp` (uint64)：恢复后生效的世代版本号。
  * `final_length` (uint64)：对齐后的块最终大小。
  * `primary_node_id` (string)：执行恢复的 Primary DataNode ID。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否上报成功。

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

#### 4.1.3 `GetBlockLength` (查询块落盘长度)
* **场景：** Pipeline Recovery 期间，Client 向幸存 DataNode 查询某 Block 的实际落盘字节数。
* **Request：**
  * `block_id` (uint64)：目标块 ID。
  * `gen_stamp` (uint64)：新的世代版本号。
* **Response：**
  * `length` (uint64)：该节点实际落盘的字节数。

#### 4.1.4 `TruncateBlock` (截断并升级版本)
* **场景：** Pipeline Recovery 期间，Client 要求 DataNode 将 Block 截断至指定长度并更新 `gen_stamp`。
* **Request：**
  * `block_id` (uint64)：目标块 ID。
  * `new_length` (uint64)：Block 对齐后的长度。
  * `new_gen_stamp` (uint64)：新的世代版本号。
* **Response：**
  * *(Empty)*：请求体为空，通过 gRPC 状态码确认是否处理成功。

### 4.2 DataNode -> DataNode (DataNode to DataNode)

#### 4.2.1 `InterDNRecoverBlock` (副本恢复协调)
* **场景：** 租约恢复期间，Primary DataNode 向同一 Block 的其他副本 DataNode 查询落盘长度及提交状态。
* **Request：**
  * `block_id` (uint64)：目标块 ID。
  * `new_gen_stamp` (uint64)：NameNode 为本次恢复指定的新世代版本号。
* **Response：**
  * `final_length` (uint64)：该节点实际落盘的字节数。
  * `is_committed` (bool)：该副本是否曾收到 Client 的 COMMITTED 标记（即对应块已确认完成）。