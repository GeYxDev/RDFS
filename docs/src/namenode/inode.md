# 目录树与内存模型 (Inode Tree)

在 RDFS 中，NameNode 的核心职责是管理整个文件系统的命名空间（Namespace）。这个命名空间在内存中的具象化表示，就是 **Inode Tree（索引节点树）**。

---

## 1. 设计目标

* **纯内存操作：** 所有的路径解析、文件创建、目录遍历都必须在内存中完成，保证极低的延迟。
* **高并发安全：** NameNode 需要同时处理成百上千个 Client 的 RPC 请求，内存模型必须摒弃全局大锁，采用分片机制实现无阻塞的并发读写。
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

为了实现极致并发，RDFS 采用了 **扁平化并发 ID 映射表 (Sharded Arena Tree)** 方案，将整棵树“压平”，利用 `DashMap`（分段锁并发字典）将全局锁竞争拆解为局部并发。
```rust
use std::sync::RwLock;
use std::collections::HashMap;
use std::sync::atomic::{AtomicU64, Ordering};

/// 整个 NameNode 的核心内存状态
pub struct NamespaceMap {
    /// 存放系统中所有的 Inode (扁平化存储)
    /// Key: Inode ID, Value: Inode 实体
    inodes: DashMap<u64, Inode>,
    
    /// 全局 ID 生成器 (完全无锁的原子操作)
    next_inode_id: AtomicU64,
}

impl NamespaceMap {
    /// 初始化系统，并自动创建根目录 "/"
    pub fn new() -> Self {
        let mut inodes = HashMap::new();
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
        
        Self {
            inodes,
            next_inode_id: AtomicU64::new(1),
        }
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

## 5. 并发控制与防死锁规范 (Deadlock Prevention)

虽然 `DashMap` 解决了不同目录并行写入的全局锁问题，但在执行 **跨分片原子操作**（如 `mv /dir1/a /dir2/b` 需要同时修改 `dir1` 和 `dir2` 的元数据）时，如果不加规范，会产生严重的 **AB-BA 死锁**。

### 工程规约：强制锁排序 (Lock Ordering by ID)
在 RDFS 的内部实现中，任何需要同时获取多个 Inode 引用的操作，**必须严格遵守按照 `Inode ID` 从小到大的顺序获取锁**。
* **错误示例：** 执行 `mv` 时，先获取源目录的写锁，再获取目标目录的写锁。如果线程 1 移动 A 到 B，线程 2 移动 B 到 A，必定死锁。
* **正确规范：** 先比较源目录 ID 和目标目录 ID。始终先 `get_mut()` 较小的 ID，再 `get_mut()` 较大的 ID。

通过这一强制规约，配合 `AtomicU64` 消除的发号器全局锁，RDFS 能够在异步环境中实现工业级的并发安全性与极高的元数据吞吐量。