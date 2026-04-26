# Summary

- [简介与目标 (Introduction)](README.md)
- [系统整体架构 (Architecture Overview)](architecture.md)

# 核心组件设计 (Core Components)

- [元数据节点 (NameNode)](namenode/README.md)
    - [目录树与内存模型 (Inode Tree)](namenode/inode.md)
    - [持久化机制 (FsImage & EditLog)](namenode/persistence.md)
- [数据节点 (DataNode)](datanode/README.md)
    - [数据块存储与布局 (Block Storage)](datanode/storage.md)
    - [心跳与状态汇报 (Heartbeat)](datanode/heartbeat.md)
- [客户端 SDK (Client)](client/README.md)
    - [API 接口设计 (API Design)](client/api.md)

# 数据流与协议 (Data Flow & Protocols)

- [RPC 通信协议规范 (RPC Protocol)](protocols/rpc.md)
- [核心工作流：读取与写入 (Read/Write Pipeline)](protocols/pipeline.md)
- [容错与副本恢复机制 (Fault Tolerance)](protocols/recovery.md)

# 开发与部署 (Development)

- [开发环境与构建指南 (Developer Guide)](development/guide.md)
- [里程碑与未来计划 (Milestones)](development/milestones.md)
- [开源协议 (License)](license.md)