# RDFS (Rust Distributed File System) 架构设计

## 1. 系统概述
RDFS 是一个基于 Rust 开发的、类似于 HDFS 的分布式文件系统。
* **目标：** 支持大文件的分布式存储、高吞吐量读取、多副本容错机制。
* **非目标：** V1.0 版本暂不支持低延迟小文件并发读写、不支持跨数据中心的数据分布、不支持追加写。

## 2. 核心架构与角色
系统采用典型的 **中心化单 Master 架构**，包含三个核心组件：

* **NameNode (元数据节点)：** * 职责：维护文件系统的目录树（Namespace）以及文件到数据块（Block）的映射关系。它不存储实际数据。
    * 状态：驻留在内存中，通过写前日志（WAL, Write-Ahead Log）和定期快照（Checkpoint）持久化到本地磁盘。
* **DataNode (数据节点)：**
    * 职责：负责实际数据块（Block）的物理存储。处理来自 Client 的读写请求，并定期向 NameNode 发送心跳和 Block 汇报。
* **Client (客户端)：**
    * 职责：提供给用户使用的 SDK 或 CLI 工具。负责将用户的文件切分为固定大小的 Block（默认 64MB），并与 NameNode 和 DataNode 交互。

## 3. 元数据管理设计 (NameNode 侧)
NameNode 的核心是内存中的文件系统目录树（Inode Tree）。

* **文件拆分规则：** 每个文件被等分为定长的 Chunk/Block（暂定 64MB）。每个 Block 拥有全局唯一的 `BlockId`（如 UUID 或单调递增的 u64）。
* **数据结构：**
    * `FileNode`: 包含文件名、创建时间、权限、以及一个 `Vec<BlockId>`。
    * `BlockMap`: 一个从 `BlockId` 映射到 `Vec<DataNodeId>` 的哈希表，记录每个块的物理位置。

## 4. 核心工作流设计
### 4.1 写入文件流程 (Write Path)
1. **申请创建：** Client 向 NameNode 发送 `CreateFile(path, size)` 请求。
2. **分配节点：** NameNode 在目录树创建节点，并为该文件分配所需的 Block，同时为每个 Block 选择 3 个存活的 DataNode，返回给 Client。
3. **流水线写入：** Client 将数据流式发送给第一个 DataNode，第一个 DataNode 边落盘边转发给第二个，依此类推（Pipeline 机制）。
4. **提交确认：** 所有 DataNode 落盘成功后，Client 向 NameNode 发送 `CompleteFile(path)`，文件对其他用户可见。

### 4.2 读取文件流程 (Read Path)
1. **获取元数据：** Client 向 NameNode 发送 `GetFileInfo(path)` 请求。
2. **返回位置：** NameNode 返回该文件的所有 `BlockId` 及其对应的 DataNode 列表（按网络距离排序）。
3. **并行读取：** Client 直接与距离最近的 DataNode 建立连接，并行拉取所有 Block 数据并在本地拼接返回给用户。

## 5. RPC 通信协议
采用 gRPC (通过 Rust 的 `tonic` 和 `prost` 库实现)。
* 端口分配：NameNode 默认 9000，DataNode 默认 9001。
