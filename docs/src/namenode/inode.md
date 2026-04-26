# 目录树与内存模型 (Inode Tree)

在 RDFS 中，NameNode 的核心职责是管理整个文件系统的命名空间（Namespace）。这个命名空间在内存中的具象化表示，就是 **Inode Tree（索引节点树）**。

## 1. 设计目标

* **纯内存操作：** 所有的路径解析、文件创建、目录遍历都必须在内存中完成，保证极低的延迟。
* **高并发安全：** NameNode 需要同时处理成百上千个 Client 的 RPC 请求，内存模型必须支持无数据竞争的并发读写。
* **低内存占用：** 尽量紧凑地设计数据结构，以便单台 NameNode 能够支撑上亿级别的文件元数据。

## 2. 核心数据结构 (Data Structures)

我们抽象出 `Inode` 来统一表示文件系统中的每一个元素（无论是文件还是目录）。

```rust
use std::collections::HashMap;

/// 定义节点类型
#[derive(Debug, Clone, PartialEq)]
pub enum FileType {
    File,
    Directory,
}

/// 索引节点：表示一个文件或目录的元数据
#[derive(Debug, Clone)]
pub struct Inode {
    pub id: u64,                  // 节点全局唯一 ID (通常递增)
    pub name: String,             // 当前节点名称 (如 "data.txt")
    pub node_type: FileType,      // 类型：文件或目录
    
    // --- 目录特有属性 ---
    // Key: 子节点名称 (如 "user"), Value: 子节点的 Inode ID
    pub children: Option<HashMap<String, u64>>, 
    
    // --- 文件特有属性 ---
    // 该文件包含的所有数据块 ID 列表
    pub blocks: Option<Vec<u64>>, 
    pub file_size: u64,           // 文件总大小 (字节)
}
```

## 3. 内存布局策略 (Rust-Specific Layout)

在 Rust 中构建树状结构有多种方案，为了平衡**开发难度**与**并发性能**，我们通常不采用传统的嵌套结构（即 `struct Node { children: Vec<Node> }`），而是采用 **“扁平化的 ID 映射表 (Arena / Map-based Tree)”** 方案。

### 为什么选择 ID 映射表？
如果你尝试用 `Arc<RwLock<Inode>>` 互相嵌套来构建传统的树：
1. **死锁风险极高：** 路径解析时（如 `/a/b/c`），你需要先锁住 `/`，再锁住 `a`，再锁住 `b`。一旦有并发操作同时从下往上遍历，瞬间就会死锁。
2. **生命周期地狱：** 节点的移动或删除涉及到复杂的生命周期转移。

### RDFS 的内存模型实现

我们将整棵树“压平”，保存在一个全局的哈希表中，通过 `Inode ID` 互相连接。

```rust
use std::sync::RwLock;
use std::collections::HashMap;

/// 整个 NameNode 的核心内存状态
pub struct NamespaceMap {
    /// 存放系统中所有的 Inode。
    /// Key: Inode ID, Value: Inode 实体
    inodes: RwLock<HashMap<u64, Inode>>,
    
    /// 用于分配全局唯一的 Inode ID 和 Block ID
    next_inode_id: RwLock<u64>,
    next_block_id: RwLock<u64>,
}

impl NamespaceMap {
    /// 初始化根目录 "/"
    pub fn new() -> Self {
        let mut inodes = HashMap::new();
        // 根目录的 ID 永远是 0
        let root_inode = Inode {
            id: 0,
            name: "/".to_string(),
            node_type: FileType::Directory,
            children: Some(HashMap::new()),
            blocks: None,
            file_size: 0,
        };
        inodes.insert(0, root_inode);
        
        Self {
            inodes: RwLock::new(inodes),
            next_inode_id: RwLock::new(1),
            next_block_id: RwLock::new(1),
        }
    }
}
```

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

## 5. 并发粒度优化探讨 (未来扩展)

当前设计中使用了一个全局的 `RwLock<HashMap>`：
* **优点：** 逻辑极其简单，代码不易出错。多读单写（Multiple-Reader, Single-Writer）。
* **缺点：** 当有大量 Client 同时进行创建文件（写操作）时，全局写锁会阻塞所有的读操作。

**V1.0 架构妥协：** 由于 V1.0 主要追求跑通全链路，我们将暂时使用上述的**全局读写锁**模型。
在未来的版本迭代中，可以引入 Rust 生态中强大的并发哈希表 [`dashmap`](https://crates.io/crates/dashmap)，将全局锁降级为分段锁（Segmented Lock），实现真正的超高并发。