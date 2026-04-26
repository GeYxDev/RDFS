# 持久化机制 (FsImage &amp; EditLog)

NameNode 的所有元数据（目录结构、文件属性、Block 列表）在系统运行时都驻留在内存中。为了保证系统重启或崩溃后数据不丢失，RDFS 采用了两步走的安全策略：**EditLog（操作日志）** 与 **FsImage（命名空间快照）**。

## 1. 核心概念

### 1.1 EditLog (写前日志 / WAL)
* **本质：** 一个不断追加写入（Append-only）的二进制日志文件。
* **原理：** 任何修改元数据的操作（如创建文件、删除目录、分配 Block），在应用到内存 Inode Tree **之前**，必须先序列化为一条记录并 `fsync` 刷入本地磁盘的 EditLog 文件中。
* **优势：** 磁盘的顺序追加写速度极快，不会成为高并发请求的性能瓶颈。
* **劣势：** 随着系统运行，文件会无限增大。如果系统运行了半年后重启，从头回放半年的日志将耗费数小时。

### 1.2 FsImage (元数据快照)
* **本质：** NameNode 内存中 Inode Tree 在某一特定时刻的完整二进制 Dump（序列化快照）。
* **原理：** 将整个树状结构直接写入磁盘的一个文件中。
* **优势：** 加载速度极快，启动时只需反序列化即可瞬间恢复状态。
* **劣势：** 生成快照非常耗时且消耗 CPU，不能每次发生写操作时都生成。

---

## 2. 系统启动与恢复流程 (Recovery Path)

当 NameNode 节点启动时，它会严格按照以下顺序恢复状态：

1. **加载快照：** 读取本地磁盘上最新的 `fsimage_latest.bin` 文件，反序列化到内存，重建基础的 Inode Tree。
2. **回放日志：** 找到快照生成时刻之后的 `editlog_xxx.log` 文件，按顺序在内存中重放（Replay）所有的操作记录。
3. **接受注册：** 内存目录树恢复完毕后，进入“安全模式”（Safe Mode），等待各个 DataNode 连接并汇报它们手中实际存在的 Block 列表。
4. **对外服务：** 当绝大多数 Block 都有 DataNode 认领后，退出安全模式，开始接受 Client 的读写请求。

---

## 3. Checkpoint (快照合并机制)

为了防止 EditLog 无限膨胀导致启动极慢，系统需要定期进行 Checkpoint（合并）。

**触发条件：**
* 定时触发：例如每隔 1 小时。
* 定量触发：例如 EditLog 中积累了超过 100 万条记录。

**执行流程 (在 RDFS V1.0 单节点架构下)：**
1. NameNode 冻结当前的 EditLog（将其重命名为只读），并创建一个新的 EditLog 供后续请求继续写入。
2. NameNode 开启一个后台异步任务（或使用写时复制/读写锁），将内存中的全量 Inode Tree 序列化并写入临时文件 `fsimage.tmp`。
3. 写入完成后，将 `fsimage.tmp` 原子重命名为新的 `fsimage_latest.bin`。
4. 安全删除旧的、已经被合并的 EditLog 文件。

---

## 4. Rust 数据结构建模设计

在 Rust 中，我们可以极具表达力地定义 EditLog 的操作类型。建议使用 `serde` 配合 `bincode` 进行高效的二进制序列化。

```rust
use serde::{Serialize, Deserialize};

/// 定义所有会改变 NameNode 状态的操作
#[derive(Serialize, Deserialize, Debug)]
pub enum EditLogOp {
    /// 事务 1: 创建新文件
    CreateFile {
        path: String,
        inode_id: u64,
        timestamp: u64,
    },
    
    /// 事务 2: 为文件追加一个新的数据块
    AllocateBlock {
        inode_id: u64,
        block_id: u64,
    },
    
    /// 事务 3: 删除文件或目录
    DeleteNode {
        path: String,
    },
    
    /// 事务 4: 重命名
    Rename {
        old_path: String,
        new_path: String,
    }
}
```

## 5. RDFS 技术选型建议

在实现持久化模块时，我们有两种技术路径：

* **路径 A (经典硬核派)：** 完全基于 `std::fs::File`，自己控制文件流、缓冲(`BufWriter`) 和手动刷盘(`file.sync_all()`)。这是学习分布式存储底层的最佳途径。
* **路径 B (现代实用派)：** 嵌入一个纯 Rust 编写的 KV 存储引擎（如 [`sled`](https://github.com/spacejam/sled) 或 `rocksdb`）。将 `path` 作为 Key，`Inode` 序列化作为 Value，由存储引擎代为管理 WAL 和快照。

**V1.0 决策：** 为了最大化学习收益并保持最纯正的 DFS 风味，我们推荐采用 **路径 A**。自己定义 FsImage 的二进制魔数（Magic Number）、版本号和校验和（Checksum）。