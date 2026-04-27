# 目录树与内存模型 (Inode Tree)

在 RDFS 中，NameNode 的核心职责是管理整个文件系统的命名空间（Namespace）。这个命名空间在内存中的具象化表示，就是 **Inode Tree（索引节点树）**。

---

## 1. 设计目标

* **纯内存操作：** 所有的路径解析、文件创建、目录遍历都必须在内存中完成，保证极低的延迟。
* **高并发安全：** NameNode 需要同时处理成百上千个 Client 的 RPC 请求，内存模型必须支持无数据竞争的并发读写。
* **低内存占用：** 尽量紧凑地设计数据结构，以便单台 NameNode 能够支撑上亿级别的文件元数据。

---

## 2. 核心数据结构 (Data Structures)

HDFS 的 NameNode 是极度受限于内存容量的组件（Memory-Bound）。为了将上亿个文件的元数据塞进有限的内存，必须利用 Rust 的**带载荷枚举 (Data-carrying Enum)** 来消除结构体中的冗余字段，确保文件节点绝不承担目录节点的内存开销。

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

## 3. 内存布局策略 (Rust-Specific Layout)

在 Rust 中构建树状结构有多种方案，为了平衡**开发难度**与**并发性能**，前期开发将不采用传统的嵌套指针结构（如 `Box` 或 `Arc<RwLock<Inode>>` 互相嵌套），而是采用 **“扁平化的 ID 映射表 (Arena / Map-based Tree)”** 方案。

### 为什么选择 ID 映射表？
如果你尝试用锁嵌套来构建传统的树：
1. **死锁灾难：** 路径解析时（如 `/a/b/c`），需要先锁 `/`，再锁 `a`，再锁 `b`。如果有跨目录的移动操作（`mv /a/x /b/y`），由于加锁顺序不一致，瞬间就会引发死锁。
2. **所有权地狱：** 节点的移动或删除涉及到复杂的生命周期和所有权转移，很容易与借用检查器（Borrow Checker）发生冲突。

### RDFS 的无死锁内存模型实现
将整棵树“压平”，所有 Inode 平铺保存在全局的哈希表中，通过唯一的 `u64 ID` 作为逻辑指针互相连接。

```rust
use std::sync::RwLock;
use std::collections::HashMap;

/// 整个 NameNode 的核心内存状态
pub struct NamespaceMap {
    /// 存放系统中所有的 Inode (扁平化存储)
    /// Key: Inode ID, Value: Inode 实体
    inodes: RwLock<HashMap<u64, Inode>>,
    
    /// ID 生成器
    next_inode_id: RwLock<u64>,
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
            inodes: RwLock::new(inodes),
            next_inode_id: RwLock::new(1),
        }
    }
}
```

---

## 4. 核心工作流：路径解析 (Path Resolution)

当 Client 请求读取 `/user/data/log.txt` 时，NameNode 需要在内存中进行以下操作：

1. **切分路径：** 将路径按 `/` 切分为 `["user", "data", "log.txt"]`。
2. **遍历 ID：**
    * 获取全局读锁：`let map = inodes.read().unwrap();`
    * 从 `ID = 0` (根目录) 开始。
    * 在 `ID=0` 的 `children` 字典中查找 `user`，得到 `ID = 10`。
    * 在 `ID=10` 的 `children` 字典中查找 `data`，得到 `ID = 25`。
    * 在 `ID=25` 的 `children` 字典中查找 `log.txt`，得到 `ID = 108`。
3. **返回结果：** 检查 `ID=108` 的类型。如果是文件，则返回其 `blocks` 列表。

---

## 5. 并发粒度优化探讨 (未来扩展)

当前设计中使用了一个全局的 `RwLock<HashMap>`：
* **优点：** 逻辑极其简单，代码不易出错。多读单写（Multiple-Reader, Single-Writer）。
* **缺点：** 当有大量 Client 同时进行创建文件（写操作）时，全局写锁会阻塞所有的读操作。

**V1.0 架构妥协：** 由于 V1.0 主要追求跑通全链路，我们将暂时使用上述的**全局读写锁**模型。
在未来的版本迭代中，可以引入 Rust 生态中强大的并发哈希表 [`dashmap`](https://crates.io/crates/dashmap)，将全局锁降级为分段锁（Segmented Lock），实现真正的超高并发。