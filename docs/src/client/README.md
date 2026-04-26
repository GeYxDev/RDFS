# 客户端 SDK (Client)

![Role](https://img.shields.io/badge/Role-User_Interface-blue.svg)
![API](https://img.shields.io/badge/API-Async%20Rust-orange.svg)

Client 是 RDFS (Rust Distributed File System) 与外部世界交互的唯一大门。它的核心使命是 **“封装复杂性”**：将底层的 NameNode 调度、DataNode 数据流、Block 切片以及容错重试机制，全部隐藏在一个极其简单、易用的 API 背后。

对于使用者来说，操作 RDFS 应该像操作本地的 `tokio::fs` 一样自然。

## 🎯 核心职责

1. **元数据交互 (Metadata Operations):** 向 NameNode 发起 RPC 请求，执行目录创建、文件定位、块申请等控制流操作。
2. **大文件切片 (File Chunking):** 将用户传入的大文件（如 10GB 的日志），自动在内存中切分为固定大小的 Block（默认 64MB），并进一步切分为 64KB 的 Chunk 进行流式网络传输。
3. **流水线读写 (Pipeline I/O):** 根据 NameNode 返回的路由表，直接与 DataNode 建立 gRPC Stream，实现高吞吐量的数据读写。
4. **透明容错 (Transparent Failover):** 在读取数据时，如果当前连接的 DataNode 宕机或返回了错误的 Checksum，Client 必须能够静默地切换到备用副本节点，对上层业务完全透明。

## 📦 形态与接口定义 (API Design)

Client 模块将提供两种使用形态：**Rust SDK** 和 **CLI 命令行工具**。

### 1. 核心 Rust SDK (`src/lib.rs`)
我们将提供一个类似标准库的纯异步 API：

```rust
use rdfs_client::RdfsClient;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

// 1. 连接到集群 (只需指定 NameNode 地址)
let mut client = RdfsClient::connect("[http://127.0.0.1:9000](http://127.0.0.1:9000)").await?;

// 2. 写入文件
let mut writer = client.create("/user/data.txt").await?;
writer.write_all(b"hello distributed world!").await?;
writer.close().await?; // 触发底层 Pipeline 的 Finalize

// 3. 读取文件
let mut reader = client.open("/user/data.txt").await?;
let mut buffer = Vec::new();
reader.read_to_end(&mut buffer).await?;
```

### 2. CLI 工具 (`src/bin/rdfs.rs`)
为了方便运维和测试，Client 包还会编译出一个二进制可执行文件，对标 Hadoop 的文件系统命令：
* `rdfs ls /user/`
* `rdfs put local_file.txt /user/data.txt`
* `rdfs cat /user/data.txt`
* `rdfs rm /user/data.txt`

## 📂 模块划分建议 (`src/`)

```text
src/
├── lib.rs               # SDK 的公开 API 入口 (RdfsClient)
├── bin/
│   └── rdfs.rs          # 命令行工具 (CLI) 的入口，基于 clap 解析参数
├── namenode_rpc.rs      # 封装与 NameNode 的所有 gRPC 交互逻辑
├── datanode_rpc.rs      # 封装与 DataNode 的 gRPC Stream 读写逻辑
└── io/                  # 核心 I/O 抽象引擎
    ├── mod.rs
    ├── reader.rs        # 实现 AsyncRead，内部自动处理跨 Block 读取和节点切换
    └── writer.rs        # 实现 AsyncWrite，内部自动处理 64MB 切块和 Pipeline 建立
```

## 🚀 开发里程碑 (Local Milestones)

- [ ] **Phase 1: 基础基建**
    - 引入 `common` 包的 gRPC 客户端代码。
    - 实现 `namenode_rpc.rs`，跑通简单的 `CreateFile` 和 `GetFileInfo`。
- [ ] **Phase 2: CLI 骨架搭建**
    - 引入 `clap` 库，搭建 `rdfs` 命令行的路由结构（ls, mkdir 等只涉及元数据的操作）。
- [ ] **Phase 3: 写入引擎 (Writer)**
    - 实现 `RdfsWriter`，跑通：写满 64MB -> 找 NameNode 要新 Block -> 连 DataNode 发送数据的全流程。
- [ ] **Phase 4: 读取引擎 (Reader)**
    - 实现 `RdfsReader`，跑通根据 Offset 智能路由到对应 Block 和可用 DataNode 的逻辑。