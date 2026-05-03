# 客户端 SDK (Client)

![Role](https://img.shields.io/badge/Role-User_Interface-blue.svg)
![API](https://img.shields.io/badge/API-Async%20Rust-orange.svg)
![Security](https://img.shields.io/badge/Security-Token_Bearer-green.svg)

Client 是 RDFS (Rust Distributed File System) 与外部世界交互的唯一大门。它的核心使命是 **“封装复杂性”**：将底层的 NameNode 调度、DataNode 数据流、Block 切片、Token 鉴权透传以及容错重试机制，全部隐藏在一个极其简单、易用的 API 背后。

对于使用者来说，操作 RDFS 应该像操作本地的 `tokio::fs` 一样自然。

## 🎯 核心职责

1. **元数据与租约交互 (Metadata & Lease)：** 向 NameNode 发起 RPC 请求执行控制流操作。在进行写入时，必须在后台静默运行 **租约保活 (Lease Renewal)** 任务，防止 NameNode 误判客户端宕机而收回写锁。
2. **鉴权令牌透传 (Token Bearer)：** 客户端从 NameNode 申请元数据时会获得 **Block Token**。客户端在与 DataNode 建立读写 Stream 时，必须主动在首包中提交该 Token，并在其过期或受损时处理相应的鉴权异常。
3. **大文件切片与端到端校验 (Chunking & Checksum)：** 将大文件自动切分为 64MB 的 Block 和 64KB 的 Chunk 进行网络传输。在写入时生成 Checksum 校验和；**在读取时必须在本地实时比对校验和**，保障端到端的数据绝对完整。
4. **双向流水线读写与 ACK (Bidi-Pipeline I/O)：** 根据路由表，与首个 DataNode 建立 **双向 gRPC Stream**，客户端必须同时运行“发送任务”（推数据）和“接收任务”（接收底层传回的 `WriteChunkResponse` ACK），以此实现无等待的滑动窗口高吞吐。
5. **透明容错与管线恢复 (Transparent Failover & Recovery)：**
    * **读取容错 (Read Tolerance)：** 遇到宕机或 Checksum 报错，静默切换至备用节点，并向 NameNode 举报坏块 (`ReportBadBlock`)。
    * **写入容错 (Pipeline Recovery)：** 写入中途遭遇节点宕机（或长时间收不到 ACK）时，静默执行“踢出坏节点 -> 升级 `gen_stamp` -> 长度对齐 -> 降级续传”的管线恢复复杂状态机，对上层 `AsyncWrite` 调用者完全透明。
    * **恢复互斥 (Recovery Exclusion)：** 若 Client 在管线恢复时收到 `RESOURCE_EXHAUSTED` 错误，表示该 Block 正在被租约恢复或其他管线恢复操作处理，应等待随机退避（如 1-3 秒）后重试 `ReportFailedBlock`。
    * **提交重试 (Commit Retry)：** 在 `close()` 调用 `CompleteFile` 时，若 NameNode 因末块副本数不足而拒绝提交（竞态窗口），Client 必须自动重试（间隔 6 秒，最多 10 次），直到提交成功或超时失败。此过程对上层用户透明。
    * **一致补偿 (Consistency Compensation)：** 若 `GetFileInfo` 返回 `UNDER_CONSTRUCTION` 状态，Client SDK 应自动在短时间内（最多 50ms）重试 2-3 次，因为文件可能已完成 `CompleteFile` 但 DashMap 尚未传播。若重试后仍为 `UNDER_CONSTRUCTION`，则向用户返回 `FileNotClosed` 错误。

## 📦 形态与接口定义 (API Design)

Client 模块将提供两种使用形态：**Rust SDK** 和 **CLI 命令行工具**。

### 1. 核心 Rust SDK (`src/lib.rs`)
提供一个类似标准库的纯异步 API：
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
├── namenode_rpc.rs      # 封装与 NameNode 的 gRPC 交互 (获取元数据及 Token)
├── datanode_rpc.rs      # 封装与 DataNode 的 gRPC Stream 通信 (支持双向流握手)
├── lease_renewer.rs     # 后台守护任务：专门负责定期向 NameNode 发送 RenewLease 续约
└── io/                  # 核心 I/O 抽象引擎
    ├── mod.rs
    ├── reader.rs        # 实现 AsyncRead，内置 Token 鉴权、Checksum 实时校验、坏块举报与平滑节点切换
    └── writer.rs        # 实现 AsyncWrite，内置双向流 ACK 监听、64MB 切块、EOF 提交及 Pipeline Recovery
```

## 🚀 开发里程碑

- [ ] **Phase 1：基础基建与元数据骨架**
    - 引入 `common` 包，实现 `namenode_rpc.rs`，并妥善缓存获取到的 Block Token。
    - 实现 `rdfs_client::create` 和 `open`，跑通文件状态交互。
    - 实现 `lease_renewer.rs` 并在 `create` 时自动唤醒后台续约任务。
- [ ] **Phase 2：命令行 CLI 工具**
    - 引入 `clap` 搭建 `rdfs ls`, `mkdir`, `rm` 等控制流指令。
- [ ] **Phase 3：极致读取引擎 (Reader)**
    - 实现 `RdfsReader` (`AsyncRead`)，跑通单块流式读取（确保首包发送 Token）。
    - 加入跨 Block 自动无缝切换逻辑。
    - 加入流式 Checksum 校验机制，并在失败时调用 `ReportBadBlock`。
- [ ] **Phase 4：极致写入引擎 (Writer) 与异常恢复**
    - 实现 `RdfsWriter` (`AsyncWrite`)，建立全双工 `mpsc` 模型，并发处理 Send 和 ACK Recv。
    - 跑通完整链路：Token 透传 -> 64KB Chunk 发送 -> 接收 ACK -> 64MB 切块 -> 申请新 Block。
    - 在文件 `close()` 时拦截处理，调用 `CompleteFile` 正式提交。
    - 实现 Pipeline Recovery 状态机（捕获 gRPC 写入异常，经 NameNode 授权后由 Client 执行流水线恢复）。