# 数据块存储与布局 (Block Storage)

DataNode 的核心职责是物理磁盘管理。不管上层的流式网络处理多么精妙，数据最终都要落入硬盘。本节定义了 RDFS 中数据块在单台物理机上的存储结构、磁盘 I/O 策略以及数据完整性保障机制。

---

## 1. 物理磁盘布局 (Physical Disk Layout)

为了避免单目录下文件过多导致 Inode 扫描瓶颈，RDFS 采用 **多级哈希目录 (Directory Sharding)** 机制打散数据块。

假设基础存储目录为 `/data/rdfs_dn/`，其内部结构如下：

```text
/data/rdfs_dn/
├── current/                  # 稳定状态的数据块 (Finalized)
│   ├── subdir_00/            # 第一级子目录 (基于 Block ID 取模：block_id % 256)
│   │   ├── blk_1024.data          # 物理数据块文件
│   │   ├── blk_1024_1001.meta     # 元数据文件 (后缀 1001 为 gen_stamp)
│   │   ├── blk_2048.data
│   │   └── blk_2048_1005.meta
│   ├── subdir_01/
│   └── ...
├── rbw/                      # 正在接收/写入的副本 (Replica Being Written)
│   ├── blk_1025.data
│   └── blk_1025_1002.meta
└── node_id                   # 记录本节点的持久化 UUID，防止重启后身份丢失
```

---

## 2. 数据与元数据分离 (Data & Meta Files)

每个逻辑 Block 在磁盘上对应 **两个** 物理文件。世代版本号 (`gen_stamp`) 必须体现在 `.meta` 的文件名中，使得 DataNode 在重启全盘扫描做 `BlockReport` 时，只需读取目录项，无需发起任何昂贵的文件 Open/Read 操作即可凑齐上报信息。

### 2.1 `.data` 文件 (`blk_<id>.data`)
纯粹的用户数据流，不包含任何额外系统信息，支持直接的 `read/write`。如果块未写满，文件实际大小即为当前长度。

### 2.2 `.meta` 文件 (`blk_<id>_<gen_stamp>.meta`)
记录该块所有的 **校验和 (Checksum)** 数组。通过序列化工具（如 `bincode` 或 JSON）保存。
```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
pub struct BlockMetadata {
    /// 每个 Chunk (例如 64KB) 的 CRC32 校验和数组
    pub chunk_checksums: Vec<u32>,
}
```

---

## 3. 块的生命周期与状态转换 (Block Lifecycle)

1. **RBW (Replica Being Written - 写入中)：**
   Client 开始向 DataNode 推送流数据。文件被创建在 `/rbw/` 目录下。该状态下，数据正处于 Pipeline 的接收流中。
2. **Finalizing (提交中)：**
   Client 发送 `CloseSend` (EOF)。DataNode 执行 `tokio::fs::File::sync_all()` 强行刷盘，并将内存中累积的 Checksum 数组序列化写入 `.meta` 文件。
3. **Finalized (已完成)：**
   DataNode 将 `.data` 和 `.meta` 文件从 `/rbw/` 移入 `/current/subdir_xx/` 目录。此时该块正式处于稳定状态。
   ⚠️ 重命名铁律 (Rename Order)：必须先重命名 `.meta` 文件，再重命名 `.data` 文件。由于两个独立的 `rename` 系统调用无法做到绝对原子化，如果在此期间发生掉电，重启后 DataNode 扫描 `/current/` 目录时可能遇到两种孤立文件：
   * **正向残留：** 先重命名 `.data` 成功，重命名 `.meta` 前掉电 -> 存在孤立的 `.data` 文件。**处理：** 直接删除该 `.data`（无元数据校验和，无法保证完整性），并上报 NameNode 该块失效。
   * **逆向残留：** 先重命名 `.meta` 成功，重命名 `.data` 前掉电 -> 存在孤立的 `.meta` 文件。**处理：** 删除该 `.meta` 文件，并上报 NameNode 该块失效。没有数据块的元数据无意义，NameNode 会通过副本复制机制补齐缺损副本。
4. **Deleted (已删除)：**
   收到 NameNode 指令后，使用 `tokio::fs::remove_file` 同步清理对应的 data 和 meta 文件。

---

## 4. 异步 I/O 与性能优化 (Async I/O & Performance Tuning)

在 DataNode 的代码编写中，I/O 必须遵循以下铁律：

* **严禁阻塞 (No Blocking)：** 绝对不能使用 `std::fs`。所有的磁盘读写必须使用 `tokio::fs::File`、`tokio::io::AsyncReadExt` 和 `tokio::io::AsyncWriteExt`。
* **缓冲读写 (Buffered I/O)：** 由于每次 `tokio::fs` 调用都会陷入内核态，建议在底层使用 `tokio::io::BufReader` 和 `tokio::io::BufWriter`，将小块的读写合并为较大块（如 4MB 批量落盘）。
* **进阶优化 (Direct I/O)：** 对于这种纯粹的底层存储，操作系统的 Page Cache 往往是浪费的（数据通常只写一次，且客户端已有缓存）。在未来的架构升级中，可以通过 Linux 的 `O_DIRECT` 标志绕过内核缓存，实现真正的 Zero-copy 传输。

---

## 5. 数据校验机制 (Data Integrity)

数据在通过网卡、内核协议栈流转时，存在极小概率发生内存比特翻转。因此，RDFS 强制执行严格的端到端 Checksum 校验防线：

* **写入流 (Write Pipeline)：** Client 负责计算每个 64KB Chunk 的 Checksum 并发给 DataNode。DataNode 收到 Chunk 数据后，必须在落盘前消耗少量 CPU 周期重新计算 Checksum，并与 Client 传来的期望值进行严格比对。只有比对一致，才允许落入 `.data` 文件并将校验和存入 `.meta` 文件。若不一致，直接断开 Stream 并向上游抛出异常。
* **读取流 (Read Pipeline)：** DataNode 响应 Read 请求时，只需无脑地将 `.data` 和 `.meta` 里的校验和打包成 gRPC Stream 发给 Client。**由 Client 在内存中比对 Checksum**。若 Client 发现不一致，它会主动废弃数据并向 NameNode 举报坏块。
* **主动防御 (Block Scanner)：** DataNode 的 CPU 只有在系统空闲时，才会通过后台运行的 `Block Scanner` 主动去读磁盘算 Checksum，进行静默损坏的排查。