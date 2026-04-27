# 持久化机制 (FsImage &amp; EditLog)

NameNode 的所有元数据（目录结构、文件属性、Block 列表）在系统运行时都驻留在内存中。为了保证系统重启或崩溃后数据不丢失，RDFS 采用了两步走的安全策略：**EditLog（操作日志）** 与 **FsImage（命名空间快照）**。

---

## 1. 核心概念

### 1.1 EditLog (写前日志 / WAL)
* **本质：** 一个不断追加写入（Append-only）的二进制日志文件。
* **原理：** 任何修改元数据的操作（如创建文件、删除目录、分配 Block），在应用到内存 Inode Tree **之前**，必须先序列化为一条记录并 `fsync` 刷入本地磁盘的 EditLog 文件中。
* **优势：** 磁盘的顺序追加写速度极快，不会成为高并发请求的性能瓶颈。
* **劣势：** 随着系统运行，文件会无限增大。如果系统运行了半年后重启，从头回放半年的日志将耗费数小时。
* ⚠️ **HDFS 持久化铁律：** FsImage 中**只保存“文件 -> Block ID 列表”的逻辑映射，绝对不保存 Block 实际存储在哪个 DataNode 的物理位置信息！** 物理位置信息随时在变，只能在系统启动时通过 DataNode 的 `BlockReport` 实时在内存中重建。

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
NameNode 内存中往往有上千万个对象，如果在主节点上直接进行全量序列化合并，会引发极其严重的性能抖动甚至内存溢出（OOM）。因此，RDFS 遵循 HDFS 的经典设计，引入了一个独立的物理角色：**Secondary NameNode (第二名称节点)**。

**Secondary NameNode 不是 NameNode 的热备（HA），它的唯一职责就是专门负责在后台合并 EditLog 和 FsImage。**

**执行流程 (Checkpoint 流转机制)：**
1. **触发：** 当距离上次合并达到 1 小时，或主节点的 EditLog 积累了 100 万个事务（TxID）时，Secondary NameNode 发起 Checkpoint 请求。
2. **主节点日志滚动：** 主 NameNode 暂停当前 EditLog 的写入，滚动生成一个新的 EditLog 文件（如 `edits_inprogress_1001`）接收新请求。旧的日志变为只读（如 `edits_1_1000`）。
3. **拉取数据：** Secondary NameNode 通过 HTTP/gRPC 从主节点拉取最新的 `fsimage` 和所有未合并的 `edits` 文件。
4. **本地合并：** Secondary NameNode 将拉取到的 FsImage 加载到**自己的内存**中，并逐条回放 EditLog。这消耗的是 Secondary NameNode 的 CPU 和内存，完全不影响主节点对外服务。
5. **生成新快照：** 回放完毕后，Secondary NameNode 将内存状态序列化为新的快照文件（如 `fsimage_1000`）。

---

## 4. Rust 数据结构建模设计

在 Rust 中，EditLog 的操作类型可以通过 `enum` 完美表达。核心原则是：**所有操作必须携带全局递增的事务 ID (TxID)**。

```rust
use serde::{Serialize, Deserialize};

/// 定义所有会改变 NameNode 状态的操作日志
#[derive(Serialize, Deserialize, Debug)]
pub struct EditLogRecord {
    pub txid: u64,          // 全局严格单调递增的事务 ID
    pub timestamp: u64,     // 操作发生的时间戳
    pub op: EditLogOp,      // 具体的事务负载
}

#[derive(Serialize, Deserialize, Debug)]
pub enum EditLogOp {
    /// 事务 1: 创建新文件 (处于 UnderConstruction 状态)
    CreateFile {
        path: String,
        client_id: String,
    },

    /// 事务 2: 为文件追加一个新的数据块
    AllocateBlock {
        path: String,
        block_id: u64,
        gen_stamp: u64,
    },

    /// 事务 3: 客户端写完文件，关闭租约 (UnderConstruction -> Active)
    CompleteFile {
        path: String,
        total_size: u64,
    },

    /// 事务 4: 删除文件或目录
    DeleteNode {
        path: String,
    },

    /// 事务 5: 重命名
    Rename {
        old_path: String,
        new_path: String,
    }
}
```

---

## 5. RDFS 技术选型建议

在实现持久化模块时，我们有两种技术路径：

* **路径 A (经典硬核派)：** 完全基于 `std::fs::File`，自己控制文件流、缓冲(`BufWriter`) 和手动刷盘(`file.sync_all()`)。这是学习分布式存储底层的最佳途径。
* **路径 B (现代实用派)：** 嵌入一个纯 Rust 编写的 KV 存储引擎（如 [`sled`](https://github.com/spacejam/sled) 或 `rocksdb`）。将 `path` 作为 Key，`Inode` 序列化作为 Value，由存储引擎代为管理 WAL 和快照。

**V1.0 决策：** 为了最大化学习收益并保持最纯正的 DFS 风味，我们推荐采用 **路径 A**。自己定义 FsImage 的二进制魔数（Magic Number）、版本号和校验和（Checksum）。