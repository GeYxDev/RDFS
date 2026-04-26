# RDFS 通信协议规范 (RPC Protocol Specification)

本规范定义了 RDFS 系统中所有组件（Client, NameNode, DataNode）之间的网络通信接口。
系统全局采用 **gRPC** 框架作为底层通信协议，使用 **Protocol Buffers (v3)** 进行数据序列化。

## 1. 全局约定
* **字节序与编码：** 所有字符串默认采用 UTF-8 编码。
* **流式传输 (Streaming)：** 凡是涉及实际文件数据（`bytes`）读写的接口，必须使用 gRPC 的 Stream 机制，严禁在单个一元 RPC (Unary RPC) 中打包超大字节数组。
* **默认端口：** * NameNode RPC 服务: `9000`
    * DataNode RPC 服务: 动态分配，向 NameNode 注册时上报。

---

## 2. 通用数据结构 (Common Messages)

在不同的 RPC 接口中，会频繁复用以下核心数据结构：

### 2.1 DataNodeInfo (数据节点物理信息)
描述一个 DataNode 如何被网络中的其他节点寻址。
* `node_id` (string): 节点的全局唯一标识符（建议使用 UUIDv4）。
* `ip_addr` (string): 节点的 IPv4/IPv6 地址。
* `port` (uint32): 节点提供数据读写 gRPC 服务的端口号。

### 2.2 BlockInfo (数据块元数据)
描述一个独立的数据切片。
* `block_id` (uint64): 块的全局唯一 ID（由 NameNode 单调递增分配）。
* `length` (uint64): 块当前的实际大小（单位：Bytes）。

### 2.3 LocatedBlock (带位置的块信息)
**核心结构**：将逻辑上的 Block 与物理上的 DataNode 绑定。客户端正是拿到这个结构后，才知道去哪里读写数据。
* `block` (BlockInfo): 数据块本身的属性。
* `locations` (List<DataNodeInfo>): 存储该数据块副本的所有可用 DataNode 列表（通常按网络距离或负载排序）。

---

## 3. NameNode 交互规范 (大脑的 API)

NameNode 只处理元数据，所有交互均为轻量级的一元 RPC。

### 3.1 客户端 -> NameNode (Client to NameNode)

#### 3.1.1 `CreateFile` (创建文件)
* **场景：** 客户端请求在系统中创建一个新文件。
* **Request:**
    * `path` (string): 文件的绝对路径（例如：`/data/logs/2026.log`）。
* **Response:**
    * `success` (bool): 是否创建成功。
    * `error_msg` (string): 若失败，返回原因（如“文件已存在”或“路径非法”）。

#### 3.1.2 `AllocateBlock` (申请数据块)
* **场景：** 客户端在写入文件时，发现当前块已写满（达到 64MB），向 NameNode 申请一个新的物理块和存储节点。
* **Request:**
    * `path` (string): 文件路径。
* **Response:**
    * `located_block` (LocatedBlock): 返回新生成的 BlockID，以及系统分配的 3 个 DataNode 地址。

#### 3.1.3 `GetFileInfo` (获取文件定位)
* **场景：** 客户端想要读取某个文件，需要先获取该文件的所有块清单及物理位置。
* **Request:**
    * `path` (string): 文件路径。
* **Response:**
    * `file_length` (uint64): 文件总大小。
    * `blocks` (List<LocatedBlock>): 该文件包含的所有数据块及其物理位置（按 block_id 或文件内偏移量顺序排列）。

### 3.2 DataNode -> NameNode (DataNode to NameNode)

#### 3.2.1 `Heartbeat` (心跳包)
* **场景：** DataNode 启动后，每隔固定时间（如 3 秒）向 NameNode 报活。NameNode 若连续 10 秒未收到心跳，则判定该节点宕机。
* **Request:**
    * `node_info` (DataNodeInfo): 汇报自己的 IP 和端口。
    * `capacity` (uint64): 节点配置的总存储容量。
    * `used` (uint64): 节点当前已使用的容量。
* **Response:**
    * `commands` (List<Command>): NameNode 返回的异步指令（例如：要求该节点删除某个废弃的 Block，或将某个 Block 复制给其他节点。V1.0 版本可暂为空列表）。

#### 3.2.2 `BlockReport` (块状态汇报)
* **场景：** DataNode 启动时（或每隔 1 小时），向 NameNode 汇报自己磁盘上真实存在的所有 BlockID 列表，用于 NameNode 校验内存状态与物理状态的最终一致性。
* **Request:**
    * `node_id` (string): 发起汇报的节点 ID。
    * `blocks` (List<uint64>): 本地磁盘上所有完好的 Block ID 集合。
* **Response:**
    * `success` (bool): 接收确认。

---

## 4. DataNode 交互规范 (苦力的 API)

专门用于处理海量底层数据的传输，严格使用 **gRPC Stream**。

### 4.1 客户端 -> DataNode (Client to DataNode)

#### 4.1.1 `WriteBlock` (流式写入)
* **场景：** 客户端拿到 `AllocateBlock` 分配的地址后，向指定的 DataNode 流式推送数据。
* **Request (Stream):** 客户端将连续发送多个数据包。
    * `block_id` (uint64): 目标块 ID。
    * `chunk_data` (bytes): 数据碎片。建议客户端以 64KB 为一个 chunk 进行流式切片发送，避免内存溢出。
* **Response (Unary):** * `success` (bool): 整个 Block 是否成功落盘。
    * `bytes_written` (uint64): 实际接收并写入磁盘的字节数。

#### 4.1.2 `ReadBlock` (流式读取)
* **场景：** 客户端请求读取某个具体的物理数据块。
* **Request (Unary):**
    * `block_id` (uint64): 目标块 ID。
    * `offset` (uint64): 起始读取位置。支持断点续传和随机读取。
    * `length` (uint64): 期望读取的数据长度。
* **Response (Stream):** DataNode 会源源不断地返回数据碎片。
    * `chunk_data` (bytes): 实际的数据碎片（如 64KB/包）。