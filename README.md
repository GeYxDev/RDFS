# RDFS (Rust Distributed File System)

![Rust](https://img.shields.io/badge/rust-1.75%2B-orange.svg)
![Architecture](https://img.shields.io/badge/architecture-Centralized_Master-blue.svg)
![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)

RDFS 是一个使用 Rust 语言从零开发的类 HDFS（Hadoop Distributed File System）的分布式文件系统。本项目旨在利用 Rust 卓越的内存安全性和无数据竞争的并发模型，构建一个轻量级、高吞吐量的分布式存储底层基础设施。

## 🎯 设计目标

* **化繁为简：** 采用经典的中心化单 Master 架构（Single NameNode），专注于分布式存储的数据分块、流水线复制与高可用性。
* **强大性能：** 核心网络层基于 `tokio` 异步运行时和 `tonic` (gRPC)，实现高效的节点间通信与数据流传输。
* **安全可靠：** 借助 Rust 的所有权机制彻底杜绝内存泄漏，通过多副本机制保障数据不丢失。

## 🏗️ 系统架构

RDFS 包含三个核心组件：

1. **NameNode (元数据节点)：** 系统的“大脑”。负责管理全局的文件目录树（Namespace），并维护文件块（Block）到具体物理节点的映射关系。它不触碰实际数据流。
2. **DataNode (数据节点)：** 系统的“苦力”。负责实际数据块（Chunk）的磁盘读写，定期向 NameNode 发送心跳以汇报自身状态和持有的数据块。
3. **Client (客户端)：** 用户的入口。负责将大文件切分成固定大小的 Block（如 64MB），并直接与 DataNode 交互进行数据的高速上传和下载。

## 📁 项目结构 (Workspace)

本项目采用标准的 Cargo Workspace 结构组织：

```text
RDFS/
├── common/      # 公共模块：RPC Protobuf 定义、全局错误处理、通用工具函数
├── namenode/    # NameNode 核心逻辑：内存 Inode Tree、持久化机制
├── datanode/    # DataNode 核心逻辑：本地磁盘管理、心跳服务
├── client/      # 客户端 SDK 和 CLI 命令行工具
└── docs/        # 系统架构设计文档 (基于 mdBook)
```

## 📖 文档指南 (Documentation)

本项目的文档分为两部分：

1. **架构设计文档 (宏观)：** 记录了系统设计的原理、RPC 协议和工作流。
   ```bash
   cd docs
   mdbook serve --open
   ```
2. **API 参考文档 (微观)：** 记录了代码级别的结构体和函数说明。
   ```bash
   cargo doc --no-deps --open --workspace
   ```

## 🚀 快速开始 (Getting Started)

### 环境依赖
* [Rust toolchain](https://rustup.rs/) (建议 1.75+)
* Protocol Buffers 编译器 (`protoc`)

### 构建与运行 (示例)

> **注意：** 项目目前处于早期开发阶段，以下命令为规划中的运行方式。

1. **克隆项目**
   ```bash
   git clone https://github.com/GeYxDev/RDFS.git
   cd rdfs
   ```

2. **编译全量项目**
   ```bash
   cargo build --release
   ```

3. **启动集群 (本地模拟)**
   ```bash
   # 终端 1: 启动 NameNode (默认监听 9000 端口)
   cargo run --bin namenode

   # 终端 2 & 3: 启动两个 DataNode 实例
   cargo run --bin datanode -- --data-dir /tmp/dn1 --port 9001
   cargo run --bin datanode -- --data-dir /tmp/dn2 --port 9002
   ```

## 🗺️ 开发路线图 (Roadmap)

- [ ] **v0.1: 基础骨架** - 完成 gRPC 通信定义与单机内存元数据管理。
- [ ] **v0.2: 读写通路** - 实现 Client -> NameNode -> DataNode 的基础文件读写。
- [ ] **v0.3: 分布式多副本** - 实现 Pipeline 数据复制与持久化。
- [ ] **v0.4: 容错与恢复** - 实现 DataNode 宕机检测与数据块自动重均衡。
- [ ] **v0.5: 高性能化** - 优化以完成开发，提升系统并发能力与安全性。

## 📄 开源协议

本项目采用 [Apache License 2.0](./LICENSE) 开源协议。