# 数据块存储与布局 (Block Storage)

DataNode 的核心职责是物理磁盘管理。不管上层的流式网络处理多么精妙，数据最终都要落入硬盘。本节定义了 RDFS 中数据块在单台物理机上的存储结构、磁盘 I/O 策略以及数据完整性保障机制。

## 1. 物理磁盘布局 (Physical Disk Layout)

在底层文件系统（如 ext4, XFS）中，如果将数以万计的文件放在同一个目录下，会导致极差的查找性能（Inode 扫描瓶颈）。因此，RDFS 采用**多级哈希目录 (Directory Sharding)** 机制来打散数据块。

假设 DataNode 的基础存储目录配置为 `/data/rdfs_dn/`，其内部结构如下：

```text
/data/rdfs_dn/
├── current/                  # 当前处于稳定状态的数据块
│   ├── subdir_00/            # 第一级子目录 (基于 Block ID 取模或哈希)
│   │   ├── blk_1024.data     # 真实的物理数据块文件 (最大 64MB)
│   │   ├── blk_1024.meta     # 该数据块的元数据与校验和文件
│   │   ├── blk_2048.data
│   │   └── blk_2048.meta
│   ├── subdir_01/
│   └── ...
│   └── subdir_FF/            # 默认 256 个子目录 (0x00 ~ 0xFF)
├── tmp/                      # 正在接收但尚未完成的临时数据块
│   └── blk_1025.tmp          # Pipeline 写入期间产生的文件
└── node_id                   # 记录本节点的持久化 UUID，防止重启后身份丢失
```

**寻址规则：**
对于给定的 `block_id = 1024`，我们可以通过简单的取模或位运算确定其物理路径：
`subdir = block_id % 256` -> 对应路径 `/current/subdir_00/blk_1024.data`。

## 2. 数据与元数据分离 (Data & Meta Files)

为了保证高效的流式读取和数据完整性，每一个逻辑上的 Block 在磁盘上都对应**两个**物理文件：

### 2.1 `.data` 文件
纯粹的用户数据流。为了支持零拷贝或最高效的 `tokio::fs::File::read`，里面不包含任何额外的系统信息。如果 Block 没写满，文件的实际大小就是当前写入的大小。

### 2.2 `.meta` 文件
与 `.data` 文件强绑定，记录该块的属性以及**校验和 (Checksum)**。
在 Rust 中，这通常被序列化为一段极小的二进制或 JSON 数据。

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
pub struct BlockMetadata {
    /// 块的版本戳 (Generation Stamp)，用于解决旧副本冲突
    pub generation_stamp: u64,
    /// 块的实际字节数
    pub length: u64,
    /// 每个 Chunk (例如 64KB) 的 CRC32 校验和数组
    /// 客户端读取时按 Chunk 匹配，一旦发现某段数据损坏，立即中断
    pub chunk_checksums: Vec<u32>, 
}
```

## 3. 块的生命周期与状态转换 (Block Lifecycle)

为了防止读取到写了一半的“脏数据”，Block 在本地存储层有严格的生命周期管理：

1. **Receiving (接收中):** Client 开始向 DataNode 写入数据。此时文件被创建在 `/tmp/blk_<id>.tmp` 目录下。该状态下，数据**对读请求不可见**。
2. **Finalizing (提交中):**
   Client 宣布写入完成。DataNode 执行 `tokio::fs::File::sync_all()`（强行刷盘），计算所有 Checksum 并生成 `.meta` 文件。
3. **Finalized (已完成):**
   DataNode 将 `.tmp` 文件原子重命名（Atomic Rename）移入 `/current/subdir_xx/blk_<id>.data`。此时该块正式处于稳定状态，可以响应外部的 Read 请求。
4. **Deleted (已删除):**
   收到 NameNode 下发的删除指令后，将对应的 `.data` 和 `.meta` 文件一并移除。

## 4. 异步 I/O 与性能优化 (Rust 侧实现要求)

在 DataNode 的代码编写中，I/O 必须遵循以下铁律：

* **严禁阻塞 (No Blocking):** 绝对不能使用 `std::fs`。所有的磁盘读写必须使用 `tokio::fs::File`、`tokio::io::AsyncReadExt` 和 `tokio::io::AsyncWriteExt`。
* **缓冲读写 (Buffered I/O):** 由于每次 `tokio::fs` 调用都会陷入内核态，建议在底层使用 `tokio::io::BufReader` 和 `tokio::io::BufWriter`，将小块的读写合并为较大块（如 4MB 批量落盘）。
* **Direct I/O (进阶优化):** 对于这种纯粹的底层存储，操作系统的 Page Cache 往往是浪费的（数据通常只写一次，且客户端已有缓存）。在未来的架构升级中，可以通过 Linux 的 `O_DIRECT` 标志绕过内核缓存，实现真正的 Zero-copy 传输。

## 5. 数据校验机制 (Data Integrity)

RDFS 必须容忍磁盘发生静默错误（Silent Data Corruption，即位翻转）。

* **写入时计算：** Client 在发起 Write Pipeline 时，本身会以 64KB 为一个 Chunk 发送数据。DataNode 每收到一个 Chunk，立刻计算一遍 CRC32C，将其追加到内存的 checksum 数组中。
* **读取时比对：** 响应 Read 请求时，DataNode 会在内存中加载 `.meta` 文件，一边从磁盘读 `.data`，一边验证算出的 CRC32 与 `.meta` 中的记录是否一致。
    * 如果一致：将其打包为 gRPC Stream 发送给客户端。
    * 如果不一致：立即向 NameNode 上报 `CorruptBlock` 异常，中断与客户端的传输。