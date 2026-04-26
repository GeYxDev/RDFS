# API 接口设计 (API Design)

本章节定义了 `rdfs-client` 包对外暴露的 Rust API 规范。设计的核心原则是**符合 Rust 异步生态直觉 (Idiomatic Async Rust)**，尽量抹平本地文件系统与分布式文件系统之间的认知差异。

使用者无需关心“Block 切片”、“网络流水线”、“节点重试”等底层细节，只需调用符合 `tokio::io` 标准特征 (Traits) 的方法即可完成流式读写。

---

## 1. 核心客户端入口 (`RdfsClient`)

`RdfsClient` 是用户与整个 RDFS 集群交互的唯一上下文。它内部持有了与 NameNode 通信的 gRPC Channel 连接池。

```rust
pub struct RdfsClient {
    // 内部对 NameNode gRPC Client 的封装
}

impl RdfsClient {
    /// 连接到 RDFS 集群 (通常只需要提供 NameNode 的地址)
    /// 
    /// # Examples
    /// ```rust
    /// let client = RdfsClient::connect("http://127.0.0.1:9000").await?;
    /// ```
    pub async fn connect(namenode_addr: &str) -> Result<Self, RdfsError>;
}
```

---

## 2. 元数据操作接口 (Namespace API)

这些接口主要负责目录结构的变更和文件属性的查询。它们本质上是对 NameNode 的轻量级 RPC 封装。

```rust
impl RdfsClient {
    /// 创建单级或多级目录
    pub async fn mkdir(&mut self, path: &str) -> Result<(), RdfsError>;

    /// 删除文件或目录
    /// * `recursive`: 如果为 true，则连同子目录一并删除 (类似于 rm -r)
    pub async fn delete(&mut self, path: &str, recursive: bool) -> Result<bool, RdfsError>;

    /// 列出指定目录下的所有文件和子目录状态
    pub async fn list_status(&mut self, path: &str) -> Result<Vec<FileStatus>, RdfsError>;

    /// 获取单个文件或目录的详细属性
    pub async fn get_file_info(&mut self, path: &str) -> Result<FileStatus, RdfsError>;
}

/// 文件/目录的通用状态描述
#[derive(Debug, Clone)]
pub struct FileStatus {
    pub path: String,
    pub is_dir: bool,
    pub length: u64,         // 如果是文件，表示总字节数
    pub block_count: usize,  // 占据了多少个数据块
}
```

---

## 3. 流式读取接口 (`RdfsReader`)

对于文件读取，我们提供 `open` 方法，返回一个实现了 `tokio::io::AsyncRead` 的定制化 Reader。这是实现极高吞吐量的关键。

```rust
impl RdfsClient {
    /// 打开一个存在的文件以供读取
    pub async fn open(&mut self, path: &str) -> Result<RdfsReader, RdfsError>;
}

/// RDFS 分布式文件读取器
pub struct RdfsReader {
    // 内部维护了当前读取到的 BlockId、Offset、以及与 DataNode 的 Stream 状态
}

// 极其关键的设计：实现 Tokio 的标准异步读 Trait
// 这意味着所有的上层生态（如 hyper, axum 等）都可以把 RDFS 文件当成普通的流来处理。
#[tokio::async_trait]
impl tokio::io::AsyncRead for RdfsReader {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut ReadBuf<'_>,
    ) -> Poll<std::io::Result<()>> {
        // 内部逻辑：
        // 1. 如果当前 Block 没读完，从当前的 DataNode Stream 中拉取数据填充 buf。
        // 2. 如果当前 Block 读完了，静默向 NameNode 获取下一个 Block 的物理位置，无缝切换到下一个 DataNode 连结。
        // 3. 如果遇到 Checksum 错误或连接断开，静默尝试备用节点。
    }
}

// 支持跳跃读取 (Seek)
#[tokio::async_trait]
impl tokio::io::AsyncSeek for RdfsReader { ... }
```

---

## 4. 流式写入接口 (`RdfsWriter`)

写入接口类似，返回一个实现了 `tokio::io::AsyncWrite` 的 Writer。

```rust
impl RdfsClient {
    /// 创建一个新文件并返回写入器。如果文件已存在，返回覆盖错误。
    pub async fn create(&mut self, path: &str) -> Result<RdfsWriter, RdfsError>;
}

/// RDFS 分布式文件写入器
pub struct RdfsWriter {
    // 内部维护了本地的 64KB Chunk Buffer，以及 Write Pipeline 状态
}

#[tokio::async_trait]
impl tokio::io::AsyncWrite for RdfsWriter {
    fn poll_write(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &[u8],
    ) -> Poll<Result<usize, std::io::Error>> {
        // 内部逻辑：
        // 1. 接收数据，写入本地内存缓存。
        // 2. 满 64KB 后，发送给当前的 DataNode Pipeline。
        // 3. 累计发送满 64MB 后，向 NameNode 申请新的 Block，建立新的 Pipeline。
    }

    fn poll_flush(...) -> Poll<Result<(), std::io::Error>> { ... }

    /// 极其重要！关流逻辑
    fn poll_shutdown(...) -> Poll<Result<(), std::io::Error>> {
        // 1. 强制将剩余缓存刷入 Pipeline。
        // 2. 告知 NameNode 写入完成 (CompleteFile)，更改元数据状态，释放锁。
    }
}
```

---

## 5. 错误处理 (`RdfsError`)

统一包装分布式系统中的各类异常，方便用户通过 `match` 表达式进行精确的错误处理。

```rust
#[derive(Debug, thiserror::Error)]
pub enum RdfsError {
    #[error("File or directory not found: {0}")]
    NotFound(String),

    #[error("Permission denied for path: {0}")]
    PermissionDenied(String),

    #[error("No available data nodes to replicate block")]
    NoAvailableDataNodes,

    #[error("Data corruption detected: checksum mismatch at block {block_id}")]
    ChecksumMismatch { block_id: u64 },

    #[error("RPC communication failed: {0}")]
    RpcError(#[from] tonic::Status),
    
    #[error("IO error: {0}")]
    IoError(#[from] std::io::Error),
}
```