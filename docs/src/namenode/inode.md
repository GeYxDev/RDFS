# 目录树与内存模型 (Inode Tree)

在 RDFS 中，NameNode 的核心职责是管理整个文件系统的命名空间（Namespace）。这个命名空间在内存中的具象化表示，就是 **Inode Tree（索引节点树）**。

---

## 1. 设计目标

* **纯内存操作：** 所有的路径解析、文件创建、目录遍历都必须在内存中完成，保证极低的延迟。
* **高并发安全：** NameNode 需要同时处理成百上千个 Client 的 RPC 请求，内存模型必须读分片无锁并发、写操作单线程串行执行，彻底规避死锁并保障并发安全。
* **低内存占用：** 尽量紧凑地设计数据结构，以便单台 NameNode 能够支撑上亿级别的文件元数据。

---

## 2. 核心数据结构 (Data Structures)

HDFS 的 NameNode 是极度受限于内存容量的组件（Memory-Bound）。为了将上亿个文件的元数据塞进有限的内存，必须利用 Rust 的 **带载荷枚举 (Data-carrying Enum)** 来消除结构体中的冗余字段，确保文件节点绝不承担目录节点的内存开销。

### 2.1 文件状态机定义
呼应 RDFS 的租约机制，文件必须具备状态标识，以防并发修改：
```rust
#[derive(Debug, Clone, PartialEq)]
pub enum FileStatus {
    /// 文件已关闭，所有 Block 落盘完毕，可供全局并发读取
    Active,
    /// 文件正在被写入/追加，携带当前持有写锁（租约）的 client_id
    UnderConstruction(String), 
}
```

### 2.2 载荷隔离：目录与文件的特有属性
通过 Enum 将属性物理隔离，实现极致的内存紧凑：
```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub enum InodeState {
    Directory {
        // Key: 子节点名称 (如 "data"), Value: 子节点的全局 Inode ID
        children: HashMap<String, u64>,
    },
    File {
        // 文件包含的数据块 ID 列表 (有序)
        blocks: Vec<u64>,
        file_size: u64,
        replication: u16,        // HDFS特有：该文件期望的副本数 (默认 3)
        status: FileStatus,      // 当前写入状态与租约归属
    },
}
```

### 2.3 Inode 实体定义
将通用属性（时间、名称）与隔离载荷合并，构成完整的节点：
```rust
#[derive(Debug, Clone)]
pub struct Inode {
    pub id: u64,                  // 节点全局唯一 ID
    pub name: String,             // 节点名称 (如 "log.txt")
    pub parent_id: u64,           // 父节点 ID（用于快速向上回溯或删除）
    pub mtime: u64,               // 最后修改时间戳 (毫秒)
    
    pub state: InodeState,        // 节点特有状态 (File 或 Directory)
}
```

---

## 3. 高并发内存布局 (Sharded Arena Tree)

在 Rust 中构建树状结构，如果使用传统的嵌套指针（如 `Arc<RwLock<Inode>>` 互相嵌套），在处理路径解析或跨目录移动时极易触发 **隐式锁排序死锁**。

为了实现极致并发，RDFS 采用了 **扁平化并发 ID 映射表 (Sharded Arena Tree)** 方案，将整棵树“压平”，利用 **Actor 状态机组合 DashMap 的读写分离架构** 将全局锁竞争拆解为安全的局部并发。
```rust
use dashmap::DashMap;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;

/// 整个 NameNode 的核心内存状态
/// 在读写分离架构中，该结构通常被 Arc 包裹。
/// - gRPC Handler 拥有只读访问权（直接查询 DashMap）
/// - NameSystem Actor 拥有唯一的写权限（负责修改 DashMap）
pub struct NamespaceMap {
    /// 存放系统中所有的 Inode (扁平化存储)
    /// Key: Inode ID, Value: Inode 实体
    inodes: DashMap<u64, Inode>,
    
    /// 全局 ID 生成器 (完全无锁的原子操作)
    next_inode_id: AtomicU64,
}

impl NamespaceMap {
    /// 初始化系统，并自动创建根目录 "/"
    pub fn new() -> Arc<Self> {
        let mut inodes = DashMap::new();
        // 根目录的 ID 永远是 0，它的父节点指向自己
        let root_inode = Inode {
            id: 0,
            name: "/".to_string(),
            parent_id: 0, 
            mtime: 0, // 实际应为系统当前时间
            state: InodeState::Directory {
                children: HashMap::new(),
            },
        };
        inodes.insert(0, root_inode);

        Arc::new(Self {
            inodes,
            next_inode_id: AtomicU64::new(1),
        })
    }

    /// 获取新 Inode ID (无锁调用，不阻塞目录树)
    pub fn allocate_id(&self) -> u64 {
        self.next_inode_id.fetch_add(1, Ordering::Relaxed)
    }
}
```

---

## 4. 核心工作流：路径解析 (Path Resolution)

得益于 `DashMap`，当 Client 请求读取 `/user/data/log.txt` 时，NameNode 不需要获取任何全局大锁，即可进行细粒度的解析：

1. **切分路径：** 将路径按 `/` 切分为 `["user", "data", "log.txt"]`。
2. **逐层寻址：**
    * 从 `ID=0` (根目录) 开始，获取该分片的局部读引用 `inodes.get(&0)`。
    * 在 `ID=0` 的 `children` 字典中查找 `user`，得到 `ID=10`，并释放 `ID=0` 的引用。
    * 重新定位至 `inodes.get(&10)`，查找 `data`，得到 `ID=25`。
    * 重新定位至 `inodes.get(&25)`，查找 `log.txt`，得到 `ID=108`。
3. **返回结果：** 检查 `ID=108` 的类型。如果是文件，则返回其 `blocks` 列表。

---

## 5. 并发控制与安全保证 (Concurrency & Security)

如果直接使用 `DashMap` 的分片锁来执行跨目录的写操作（如 `rename`），极易触发隐式的死锁问题。为了实现最高级别的系统稳定性与写操作的强一致性，RDFS 摒弃了复杂的细粒度锁排序机制，转而采用 **读写分离的 Actor 模型**。

### 5.1 最终一致性读模型
* 对于跨多个 Inode 的写操作（如 `rename`、跨目录移动、文件创建），由于 DashMap 各分片独立更新且读操作不加全局锁，外部读者 **可能短暂观察到中间状态**（例如文件在源目录已消失但目标目录尚未出现）。
* 这一窗口极短（仅相当于单线程 Actor 内两次 DashMap 写入之间），在正常的 RPC 超时重试机制下，客户端通过指数退避重试即可最终获得一致结果。
* NameNode **不对外提供跨操作的事务性快照读**，这是与 HDFS 全局读写锁模型的一个重要设计取舍，目的是消除 NameNode 的读锁瓶颈，支撑更高并发。

> **一致性边界：** RDFS 不提供单调读（Monotonic Read）或“读己之写”（Read-Your-Writes）保证。例如，rename 操作后短时间内 list 目录可能看不到新路径，或读到部分更新后的中间状态。客户端应使用指数退避重试（如初始 10ms，退避因子 2，最大 1s）直到获得预期结果。

### 5.2 读操作：无锁并发
对于文件系统的绝大多数操作（如 `GetFileInfo`, `GetBlockLocations` 以及 Client 的心跳寻址），并发度极高：
* gRPC Handler 线程直接通过 `inodes.get(&id)` 访问 `DashMap`。
* 仅在提取数据的极短时间内持有该分片的底层读锁，提取完毕后立刻 Drop 释放。
* 读操作完全不阻塞，也不会被系统的其他写操作长时间阻塞，达到了极致的读吞吐量。

### 5.3 写操作：绝对串行
任何试图改变命名空间状态的操作（如 `CreateFile`, `Mkdir`, `Rename`），**绝对禁止** 直接修改 `DashMap`。
1. **指令封装：** gRPC Handler 接收到写请求后，将其封装为 `Command`，通过 Tokio 的 `mpsc::channel` 发送给后台唯一的 NameSystem Actor，并等待 oneshot 回调。
2. **串行执行：** NameSystem Actor 以单线程事件循环的方式，依次从管道中拉取指令。
3. **两阶段提交：**
    * **落盘：** Actor 首先将该操作序列化为 `EditLogOp`，随后通过 `spawn_blocking` 执行同步 `fsync` 并等待结果。
    * **写内存：** 日志落盘后，Actor 作为 **全系统唯一的写入者**，直接修改 `DashMap`（此时绝无其他线程与其竞争写锁，绝对不会发生死锁）。
4. **返回结果：** 内存更新完毕后，通过 oneshot channel 唤醒 gRPC Handler 响应 Client。

通过这一架构，RDFS 实现了 EditLog 与内存状态的 100% 严格一致，彻底移除了并发修改目录树的心智负担与死锁风险。