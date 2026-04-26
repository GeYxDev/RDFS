# 心跳与状态汇报 (Heartbeat)

DataNode 是一个被动的“打工人”，NameNode 永远不会主动发起连接去询问 DataNode 的状态（为了避免复杂的网络穿透和连接保持问题）。因此，DataNode 必须主动、周期性地向 NameNode 汇报自身的情况。

这个汇报机制分为三个核心阶段：**注册 (Register)**、**周期心跳 (Heartbeat)** 和 **全量块汇报 (Block Report)**。

---

## 1. 核心工作流

### 1.1 节点注册 (Registration)
* **触发时机：** DataNode 进程刚启动时。
* **行为：** DataNode 向 NameNode 发送自己的基础物理信息，包括 `node_id`（通常在第一次启动时生成并持久化在本地）、提供 gRPC 服务的 `ip` 和 `port`，以及本地磁盘的 `capacity`（总容量）。
* **目的：** 让 NameNode 在内存中为该节点建立档案（`DataNodeInfo`），准备接收后续的心跳。

### 1.2 周期心跳 (Periodic Heartbeat)
* **触发时机：** 注册成功后，每隔固定时间（默认 3 秒）触发一次。
* **上报内容：** 当前的磁盘使用量（`used_capacity`）。
* **接收指令 (重要)：** 心跳的返回值 `HeartbeatResponse` 并不是简单的 `ACK`，它可能包含 NameNode 下发的异步指令。
  * `ReplicateBlock(block_id, target_ip)`: 指示该节点将某个块复制给别人。
  * `DeleteBlock(block_id)`: 指示该节点删除本地的某个废弃块（例如 Client 删除了文件，NameNode 就会通过心跳让 DataNode 物理删除数据）。

### 1.3 全量块汇报 (Block Report)
* **触发时机：** 注册成功后的第一时间，以及随后每隔较长时间（如 1 小时）触发一次。
* **行为：** DataNode 扫描本地存储目录，提取出所有完好无损的 `block_id` 列表，打包发给 NameNode。
* **目的：** NameNode 内存中的 Block 映射表可能会因为网络分区、进程重启等原因与物理真实情况脱节。全量块汇报是**系统最终一致性的兜底机制**。

---

## 2. Rust 并发模型设计

在 DataNode 的代码架构中，心跳机制不能阻塞主线程（主线程需要监听 gRPC 端口处理客户端的读写请求）。因此，心跳和汇报必须作为独立的后台异步任务（Background Tasks）运行。

### 2.1 后台任务抽象
我们通常在 DataNode 启动时，使用 `tokio::spawn` 剥离出专门的汇报循环：

```rust
// 伪代码示例：DataNode 的心跳后台任务
pub async fn start_heartbeat_loop(
    mut client: NameNodeServiceClient<tonic::transport::Channel>,
    node_info: DataNodeInfo,
) {
    let mut interval = tokio::time::interval(std::time::Duration::from_secs(3));
    
    loop {
        interval.tick().await; // 等待 3 秒
        
        let req = tonic::Request::new(HeartbeatRequest {
            node_id: node_info.node_id.clone(),
            // ... 获取当前磁盘状态 ...
        });

        match client.heartbeat(req).await {
            Ok(response) => {
                let commands = response.into_inner().commands;
                // 将指令推入本地的命令执行队列
                process_namenode_commands(commands).await;
            }
            Err(e) => {
                // NameNode 暂时连不上？记录日志，继续下一个循环等待重试
                tracing::warn!("Failed to send heartbeat to NameNode: {}", e);
            }
        }
    }
}
```

## 3. 容错与重试机制 (Resilience)

底层网络是不可靠的，DataNode 的心跳模块必须具备极强的韧性：

1. **NameNode 闪断重连：** 如果 `client.heartbeat()` 失败，DataNode 不应该崩溃，而是应该打印警告日志，并在下一个 3 秒周期继续尝试。
2. **指数退避 (Exponential Backoff)：** (高级特性) 如果 NameNode 长时间宕机，DataNode 的重试间隔可以从 3 秒逐渐退避到 30 秒，避免 NameNode 刚重启时遭遇“心跳风暴（Heartbeat Storm）”而雪崩。
3. **僵尸块处理：** 如果在执行 `BlockReport` 时，发现本地有一个 Block，但 NameNode 返回说“我不认识这个块（可能文件已被删除）”，DataNode 应该将其放入待删除队列，延期物理回收。