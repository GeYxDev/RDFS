# 开发环境与构建指南 (Developer Guide)

欢迎参与 RDFS 的开发！本指南将帮助你从零配置本地开发环境，并了解项目的构建、测试与运行流程。

---

## 1. 环境依赖 (Prerequisites)

RDFS 是一个纯 Rust 项目，但在网络层重度依赖 gRPC。因此，你需要安装 Rust 工具链和 Protobuf 编译器。

### 1.1 安装 Rust 工具链
我们推荐使用 `rustup` 来管理 Rust 版本。RDFS 要求 **Rust 1.75 或更高版本**。

在终端中运行以下命令安装：
```bash
curl --proto '=https' --tlsv1.2 -sSf [https://sh.rustup.rs](https://sh.rustup.rs) | sh
```

安装完成后，确保将 Cargo 的 `bin` 目录加入到你的 `PATH` 中，并验证安装：
```bash
rustc --version
cargo --version
```

### 1.2 安装 Protobuf 编译器 (`protoc`)
项目中使用的 `tonic-build` 依赖系统的 `protoc` 将 `.proto` 文件编译为 Rust 代码。
* **macOS (使用 Homebrew)：**
  ```bash
  brew install protobuf
  ```
* **Ubuntu / Debian：**
  ```bash
  sudo apt update
  sudo apt install -y protobuf-compiler libprotobuf-dev
  ```
* **Windows：**
  请前往 [Protobuf Releases](https://github.com/protocolbuffers/protobuf/releases) 页面下载预编译的 `protoc-xxx-win64.zip`，解压并将 `bin` 目录添加到系统环境变量 `PATH` 中。

验证安装：
```bash
protoc --version
```

---

## 2. 项目构建与检查 (Build & Check)

克隆项目到本地后，你可以使用标准的 Cargo 命令来管理工作区。

### 2.1 编译项目
第一次编译时，`cargo` 会自动调用 `common/build.rs` 生成 gRPC 代码，随后编译所有子包。
```bash
# Debug 模式编译（编译快，运行慢，适合开发）
cargo build

# Release 模式编译（编译慢，运行极快，适合性能测试）
cargo build --release
```

### 2.2 代码规范与静态检查
在提交代码之前，**必须** 保证通过格式化和静态分析检查：
```bash
# 自动格式化代码
cargo fmt --all

# 运行 Clippy 静态检查（严格模式）
cargo clippy --all-targets --all-features -- -D warnings
```

### 2.3 运行单元测试
RDFS 的核心逻辑（如 Inode Tree 的路径解析、DataNode 的磁盘映射）必须有完善的单元测试覆盖：
```bash
cargo test --workspace
```

---

## 3. 本地集群运行 (Running Locally)

在开发阶段，我们通常在同一台物理机上启动多个进程来模拟分布式集群。为了方便测试，建议将所有数据存储在 `/tmp/rdfs/` 目录下。

### 3.1 启动主节点 (NameNode)
打开第一个终端面板，启动系统的“大脑”，并指定元数据的持久化目录：
```bash
# NameNode 默认监听 0.0.0.0:9000
RUST_LOG=info cargo run --bin namenode -- \
  --name-dir /tmp/rdfs/name
```

### 3.2 启动第二名称节点 (Secondary NameNode)
打开第二个终端面板，启动负责在后台合并快照与日志的“打杂节点”：
```bash
RUST_LOG=info cargo run --bin secondary_namenode -- \
  --namenode-addr [http://127.0.0.1:9000](http://127.0.0.1:9000) \
  --checkpoint-dir /tmp/rdfs/namesecondary
```

### 3.3 启动数据节点 (DataNode)
打开多个新的终端面板，启动多个 DataNode（构成默认的 3 副本集群）。你需要为它们指定不同的物理存储目录和 RPC 端口。
```bash
# 终端 3: 启动 DataNode 1
RUST_LOG=info cargo run --bin datanode -- \
  --data-dir /tmp/rdfs/dn1 \
  --port 9001 \
  --namenode-addr [http://127.0.0.1:9000](http://127.0.0.1:9000)

# 终端 4: 启动 DataNode 2
RUST_LOG=info cargo run --bin datanode -- \
  --data-dir /tmp/rdfs/dn2 \
  --port 9002 \
  --namenode-addr [http://127.0.0.1:9000](http://127.0.0.1:9000)

# 终端 5: 启动 DataNode 3
RUST_LOG=info cargo run --bin datanode -- \
  --data-dir /tmp/rdfs/dn3 \
  --port 9003 \
  --namenode-addr [http://127.0.0.1:9000](http://127.0.0.1:9000)
```

### 3.4 使用 CLI 客户端测试 (Client)
打开第六个终端，使用编译好的 CLI 工具对集群进行读写操作：
```bash
# 创建目录
cargo run --bin rdfs -- mkdir /user/data

# 上传本地文件到 RDFS (默认 3 副本)
cargo run --bin rdfs -- put ./Cargo.toml /user/data/Cargo.toml

# 查看目录状态
cargo run --bin rdfs -- ls /user/data
```

---

## 4. IDE 配置与并发调试 (IDE & Debugging)

为了获得最佳的 Rust 开发体验，强烈推荐使用 **Visual Studio Code** 或 **IntelliJ Rust (RustRover)**。

### 4.1 VS Code 插件清单
* **rust-analyzer：** 绝对的核心插件，提供代码补全、跳转、重构和内联提示。
* **CodeLLDB：** 强大的 Debugger，支持在代码中打断点调试异步任务。
* **crates：** 帮助你在 `Cargo.toml` 中管理依赖版本。
* **Even Better TOML：** 完善的 `.toml` 文件高亮与校验。

### 4.2 `tokio` 异步与背压死锁调试指南
RDFS 引入了极度复杂的 **双向流 (Bidi-Stream) Pipeline** 和 **mpsc 通道解耦**。如果你发现数据写着写着突然卡死（吞吐量掉到 0），大概率是遇到了通道满了引发的背压环形死锁。

强烈建议在开发 DataNode 流水线时启用 `tokio-console`：
1. 在 `Cargo.toml` 中加入依赖并开启 `tokio_unstable` 标志。
2. 运行 `tokio-console` 终端工具。
3. 它能像“任务管理器”一样，实时展示每一个被 `tokio::spawn` 出来的 Worker 状态。你可以清晰地看到哪个 `send().await` 被挂起了，从而快速定位并打破死锁循环。

> **编码提醒：** 为避免环形死锁，拆分 Pipeline 任务时，所有跨任务发送必须使用 `try_send` 或 `send_timeout`，禁止无超时的 `.await`；接收任务在通道满时应丢弃数据或断开连接。