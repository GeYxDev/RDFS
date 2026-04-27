# RDFS (Rust Distributed File System) 🦀

![Rust](https://img.shields.io/badge/rust-1.75%2B-orange.svg)
![Architecture](https://img.shields.io/badge/architecture-Centralized_Master-blue.svg)
![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)

RDFS 是一个使用 Rust 语言从零开发的、深度借鉴 HDFS 核心架构思想的工业级分布式文件系统。本项目旨在利用 Rust 卓越的内存安全特性和无数据竞争的并发模型，构建一个**轻量、高吞吐、强容错**的分布式存储底层基础设施。

## 🎯 设计目标

* **极致并发与安全：** 利用 Rust 的所有权机制和 `tokio` 异步运行时，实现无锁化/分段锁的高吞吐元数据管理，彻底杜绝内存泄漏和数据竞争。
* **零开销数据流：** 引入基于 gRPC (HTTP/2) 的 **背压流控 (Backpressure)**，摒弃冗余的业务层 ACK，实现 Client 与 DataNode 之间的零拷贝流水线 (Pipeline) 复制。
* **坚不可摧的容错：** 引入 **世代版本号** 隔离机制杜绝分布式脑裂，并在客户端内置极其复杂的 **管线异常恢复 (Pipeline Recovery)** 状态机，实现故障对上层业务的完全透明。

## 🏗️ 系统架构

RDFS 包含四个核心逻辑组件：

1. **NameNode (元数据主节点)：** 系统的“大脑”。负责管理全局扁平化目录树（Namespace）与租约（Lease），维护文件块到物理节点的逻辑映射。**绝对不触碰实际数据流。**
2. **Secondary NameNode (第二名称节点)：** 系统的“打杂节点”。专职在后台拉取并合并 NameNode 的 EditLog (写前日志) 和 FsImage (内存快照)，防止主节点 OOM 并加速重启。
3. **DataNode (数据存储节点)：** 系统的“苦力”。负责数据块（Chunk）在本地磁盘的流式读写，执行 NameNode 下发的容错指令，并在后台运行块扫描器主动防御静默损坏。
4. **Client (重客户端)：** 用户的“大门”。将底层的 gRPC 交互、大文件切片、端到端 Checksum 实时校验、以及节点宕机时的断点降级续传，封装为极简的 `AsyncRead / AsyncWrite` 接口。

## 📁 项目结构

本项目采用标准的 Cargo Workspace 结构组织。为了逻辑紧凑，**Secondary NameNode** 与 **NameNode** 的逻辑共同托管于 `namenode` 模块中：

```text
RDFS/
├── common/              # 公共模块：RPC Protobuf 定义、全局错误处理、通用工具函数
├── namenode/            # NameNode 核心：包含网络服务、内存 Inode Tree、持久化机制
├── datanode/            # DataNode 核心：本地磁盘管理、心跳服务
├── client/              # 客户端 SDK 和 CLI 命令行工具
└── docs/                # 系统全套架构设计图纸
```

## 📖 文档指南

在编写任何代码之前，我们已经完成了对系统每一处细节的推演。如果你是新加入的开发者，请务必按照以下脉络阅读 `docs/` 目录下的文档，它们是本系统的“真理之源”：

* **[宏观白皮书]** `architecture.md`
* **[三大核心规范]** 通信协议 (`rpc.md`) | 读写流水线 (`pipeline.md`) | 容错防脑裂 (`recovery.md`)
* **[组件微观设计]** 内存模型 (`inode.md`) | 持久化 (`persistence.md`) | 磁盘布局 (`storage.md`) | 异步心跳 (`heartbeat.md`)
* **[工程落地指南]** 客户端 API (`api.md`) | 详尽里程碑 (`milestones.md`) | 本地启动指南 (`guide.md`)

## 🚀 快速开始

### 环境依赖
* [Rust toolchain](https://rustup.rs/) (建议 1.75+)
* Protocol Buffers 编译器 (`protoc`)

### 构建与运行本地伪分布式集群

> 详细的调试技巧与 IDE 配置请参阅 `docs/guide.md`。

1. **编译全量项目**
   ```bash
   cargo build
   ```

2. **启动集群 (多终端运行)**
   ```bash
   # 1. 启动主脑 NameNode (持久化到 /tmp/rdfs/name)
   cargo run --bin namenode -- --name-dir /tmp/rdfs/name
   
   # 2. 启动打杂节点 Secondary NameNode (逻辑同属于 namenode crate)
   cargo run --bin secondary_namenode -- --checkpoint-dir /tmp/rdfs/snn
   
   # 3. 启动 3 个 DataNode (构成标准 3 副本集群)
   cargo run --bin datanode -- --data-dir /tmp/rdfs/dn1 --port 9001
   cargo run --bin datanode -- --data-dir /tmp/rdfs/dn2 --port 9002
   cargo run --bin datanode -- --data-dir /tmp/rdfs/dn3 --port 9003
   ```

3. **使用 CLI 客户端体验**
   ```bash
   cargo run --bin rdfs -- mkdir /user/data
   cargo run --bin rdfs -- put ./Cargo.toml /user/data/Cargo.toml
   cargo run --bin rdfs -- ls /user/data
   ```

## 📄 开源协议

本项目采用 [Apache License 2.0](./LICENSE) 开源协议。