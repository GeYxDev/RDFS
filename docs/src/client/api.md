# API 接口设计 (API Design)

本章节定义了 `rdfs-client` 包对外暴露的 Rust API 规范。设计的核心原则是 **符合 Rust 异步生态直觉 (Idiomatic Async Rust)**，尽量抹平本地文件系统与分布式文件系统之间的认知差异。

使用者无需关心“Block 切片”、“网络流水线”、“节点重试”、“Token 鉴权”等底层细节，只需调用符合 `tokio::io` 标准特征 (Traits) 的方法即可完成流式读写。

---

## 1. 核心客户端入口 (`RdfsClient`)

`RdfsClient` 是用户与整个 RDFS 集群交互的唯一上下文。它内部不仅持有了与 NameNode 通信的连接池，还管理着当前进程的全局身份（`client_id`）以及后台的租约保活任务。
```rust
pub struct RdfsClient {
    pub client_id: String, // 客户端全局唯一标识 (UUIDv4)
    // 内部封装的 NameNode gRPC Client
    // 内部持有的 Lease Renewer (租约续约) 后台任务句柄
}

impl RdfsClient {
    /// 连接到 RDFS 集群并完成身份初始化
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

    /// 获取单个文件或目录的详细属性，并获取对应数据块的 Access Token
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
    // 内部维护了当前读取到的 BlockId、Offset、Access Token，以及与 DataNode 的 Stream 状态
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
        // 1. 发起请求：向 DataNode 发送 Read 请求，并出示该 Block 的 Token 凭证进行鉴权。
        // 2. 拉取与校验：如果当前 Block 没读完，从 Stream 中拉取数据填充 buf，并实时比对 Checksum 校验和。
        // 3. 容错与举报：如果遇到 Checksum 错误或连接断开，静默向 NameNode 举报坏块，并尝试备用节点。
        // 4. 跨块切换：如果当前 Block 读完了，静默向 NameNode 获取下一个 Block 的物理位置和新 Token，无缝切换。
    }
}

// 支持跳跃读取 (Seek)
#[tokio::async_trait]
impl tokio::io::AsyncSeek for RdfsReader { ... }
```

---

## 4. 流式写入接口 (`RdfsWriter`)

写入是 RDFS 中最复杂的操作。除了基础的覆盖写 (`create`)，我们还需要支持大数据的灵魂操作——追加写 (`append`)。
```rust
/// 文件创建时的配置选项
#[derive(Debug, Clone)]
pub struct CreateOptions {
    pub replication: u16,   // 副本数，默认 3
    pub block_size: u64,    // 块大小，默认 64MB
    pub overwrite: bool,    // 如果存在是否覆盖，默认 false
}

impl Default for CreateOptions { ... }

impl RdfsClient {
    /// 使用默认配置创建新文件
    pub async fn create(&mut self, path: &str) -> Result<RdfsWriter, RdfsError>;

    /// 使用自定义配置创建新文件
    pub async fn create_with(&mut self, path: &str, options: CreateOptions) -> Result<RdfsWriter, RdfsError>;

    /// 追加写入已存在的文件 (HDFS 核心特性)
    pub async fn append(&mut self, path: &str) -> Result<RdfsWriter, RdfsError>;
}

/// RDFS 分布式文件写入器
pub struct RdfsWriter {
    // ⚠️ Rust 异步工程铁律：
    // 由于 AsyncWrite 的 poll_write 是同步上下文，无法直接执行 gRPC 的 .await 操作。
    // RdfsWriter 内部实际上是一个 mpsc::Sender，它将数据切片推入通道；
    // 真正的大文件切块、向 NameNode 申请 AllocateBlock (附带 Token)、
    // 建立双向流 (Bidi-Stream)、处理逆流 ACK、以及 Pipeline Recovery 等复杂状态机，
    // 均由底层一个被 spawn 出来的专属异步 Worker 任务 (Receiver) 处理。
}

#[tokio::async_trait]
impl tokio::io::AsyncWrite for RdfsWriter {
    fn poll_write(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &[u8],
    ) -> Poll<Result<usize, std::io::Error>> {
        // 将 buf 拷贝/转移并推入内部的 mpsc 缓冲通道。
        // 若通道满（底层背压），则向 cx 注册 Waker 并返回 Poll::Pending。
    }

    fn poll_flush(...) -> Poll<Result<(), std::io::Error>> { ... }

    /// 极其重要！关流与安全退出逻辑
    fn poll_shutdown(...) -> Poll<Result<(), std::io::Error>> {
        // 1. 发送 EOF 信号给底层 Worker，等待缓冲刷入 DataNode。
        // ⚠️ 必须设置超时机制 (Timeout)：防止 Worker 在尝试无望的 Pipeline Recovery 时陷入死锁，导致上层应用被永久挂起。
        //    - 默认超时值为 heartbeat_interval * 3（如 3s * 3 = 9 秒，可配置）。
        //    - 若超时，Worker 任务应立即放弃未完成的数据，返回 io::ErrorKind::TimedOut。
        //      Worker 任务在被 Drop 时必须关闭与 DataNode 的所有 Stream，释放 mpsc 通道并清理临时资源。
        //    - 上层 SDK 将该错误转换为 RdfsError::WriteTimeout，用户可据此决定是否重试。
        // 2. 触发 NameNode 的 CompleteFile RPC 调用。
        // 3. 将文件状态转为 ACTIVE 并释放当前文件的写锁 (Lease)。
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

    #[error("Authentication failed: invalid or expired block token")]
    Unauthenticated(String),
    
    #[error("No available data nodes to replicate block")]
    NoAvailableDataNodes,

    #[error("Data corruption detected: checksum mismatch at block {block_id}")]
    ChecksumMismatch { block_id: u64 },

    #[error("RPC communication failed: {0}")]
    RpcError(#[from] tonic::Status),
    
    #[error("IO error: {0}")]
    IoError(#[from] std::io::Error),

    #[error("Lease expired or revoked: {0}")]
    LeaseExpired(String),
}
```