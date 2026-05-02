# 心跳与状态汇报 (Heartbeat)

DataNode 采用被动设计，NameNode 永远不会主动发起连接去询问 DataNode 的状态（为了避免复杂的网络穿透和连接保持问题）。因此，DataNode 必须主动、周期性地向 NameNode 汇报自身的情况。

这个汇报机制分为三个核心阶段：**注册 (Register)**、**周期心跳 (Heartbeat)** 和 **全量块汇报 (Block Report)**。

---

## 1. 核心工作流

这个汇报机制在 RDFS 中分为四个核心阶段，它们共同构成了数据节点与中心节点的生命周期纽带与安全信任链：

### 1.1 节点注册 (Registration / Handshake)
* **触发时机：** DataNode 进程刚启动时。
* **行为：** DataNode 向 NameNode 发起握手，发送自己的基础物理信息，包括 `node_id`（首次启动生成并持久化在本地）、`ip`、`port`，以及本地磁盘的 `capacity`（总容量）。
* **目的：** 让 NameNode 在内存中为该节点建立身份档案，准备接收流量。

### 1.2 周期心跳 (Periodic Heartbeat)
* **触发时机：** 注册成功后，每隔固定时间（默认 3 秒）触发一次。
* **上报内容：** 当前的磁盘真实使用量（`used_capacity`）。
* **接收指令与安全凭证：** NameNode 对心跳的 `Response` 充当了 **指令下发通道**：
  * **管理指令 (`List<DataNodeCommand>`)：**
    * `ReplicateBlock`：指示该节点将本地某个块复制给其他目标节点（触发副本恢复）。
    * `DeleteBlock`：指示该节点物理删除本地的某个废弃块。
    * `BlockRecoveryCommand`：租约恢复指令，本节点被指定为 Primary DN，需携带 `block_id`、`new_gen_stamp` 及所有副本地址列表，协调完成 Block Recovery。
  * **安全密钥 (`MasterKey`)：**
    * 下发 NameNode 当前正在使用的 Token 签发密钥（包含 Active 和 Standby）。DataNode 收到后立即更新本地内存中的 `SecretManager`，确保对后续 Client 的流式读写请求能够进行正确的 HMAC-SHA256 验签拦截。

### 1.3 增量块汇报 (Incremental Block Report / BlockReceived)
* **触发时机：** 在 Pipeline 写入流程中，一旦某个 Block 安全刷入本地磁盘（`fsync` 成功），立刻触发。
* **行为：** 异步发送 `BlockReceived(BlockInfo)` 单条 RPC 请求给 NameNode。
* **目的：** 这是物理层落盘的实时反馈。NameNode 依赖此汇报来确认文件写入进度，并允许 Client 顺利释放写锁租约。

### 1.4 全量块汇报 (Full Block Report)
* **触发时机：** 注册成功后的第一时间，以及随后每隔较长时间（如 1 小时）触发一次。
* **行为：** DataNode 深度扫描本地存储目录，提取出所有 Block 的完整元数据：`[block_id, length, gen_stamp]`，打包发给 NameNode 进行全网对账。
* **目的：** 应对网络分区、磁盘静默截断、节点假死等极端异常。NameNode 对比上报的 `gen_stamp` 与内存期望值执行 **世代版本号冲突裁决** 决定是否下发删除指令，是 **系统最终一致性的核心兜底机制**。

---

## 2. 并发模型设计 (Concurrency Design)

在 DataNode 的代码架构中，心跳机制 **绝对不能阻塞主线程和互相阻塞**。

### 2.1 心跳与指令执行解耦 (Decoupling)
如果在心跳循环里直接 `await` 执行指令（比如花 10 秒去复制一个数据块），会导致心跳超时，NameNode 会误判节点死亡。必须使用 `tokio::sync::mpsc` (通道) 将“接收指令”和“执行指令”彻底解耦。

### 2.2 防背压死锁设计 (Anti-Backpressure Deadlock)
如果在异常情况下（例如由于磁盘极慢，导致指令执行队列堵塞），`cmd_tx.send(cmd).await` 可能会挂起，进而卡死整个心跳循环，这也是绝对不允许的。**必须使用 `try_send` 或带有超时的发送**。
```rust
use tokio::sync::mpsc;
use tracing::{warn, info};

// 伪代码示例：DataNode 的心跳与指令处理架构
pub async fn start_heartbeat_system(mut client: NameNodeServiceClient, secret_mgr: Arc<SecretManager>) {
  // 创建一个容量为 1000 的指令通道
  let (cmd_tx, mut cmd_rx) = mpsc::channel::<DataNodeCommand>(1000);

  // 任务 1：后台心跳循环 (负责发心跳、收指令、同步密钥，绝不阻塞)
  tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(3));
    loop {
      interval.tick().await;

      let req = HeartbeatRequest { /* ... */ };
      if let Ok(response) = client.heartbeat(req).await {
        let resp = response.into_inner();

        // 1. 同步安全密钥：非阻塞更新本地验签规则
        if let Some(keys) = resp.master_keys {
          secret_mgr.update_keys(keys);
        }

        // 2. 下发管理指令：使用 try_send 防止队列满时卡死心跳循环
        for cmd in resp.commands {
          if let Err(e) = cmd_tx.try_send(cmd) {
            warn!("Instruction execution queue is full, discarding NameNode instruction: {:?}", e);
          }
        }
      }
    }
  });

  // 任务 2：专属指令执行池 (后台慢慢干活，不影响心跳)
  tokio::spawn(async move {
    while let Some(cmd) = cmd_rx.recv().await {
      match cmd.action {
        Action::DeleteBlock => storage::delete_local_block(cmd.target_block).await,
        Action::ReplicateBlock => network::replicate_to_target(cmd).await,
      }
    }
  });
}
```

### 2.3 关键指令保序 (Critical Command Ordering)
为防止背压导致关键容错指令（`ReplicateBlock`、`DeleteBlock`、`BlockRecoveryCommand`）丢失，在心跳指令分发环节增加以下机制：
* **高优通道：** 为关键指令单独建立容量更大的 `mpsc` 通道，心跳循环优先将关键指令插入高优队列。
* **超时发送：** 对高优通道使用 `send_timeout(Duration::from_millis(100))` 替代 `try_send`，在通道满时短暂等待而非直接丢弃。
* **指标监控：** 记录关键指令的丢弃次数和通道拥塞时长，暴露为 Prometheus 指标，便于运维发现副本修复延迟。

### 2.4 验签失败率监控 (SigVerify Failure Monitor)
为防止网络分区或系统时钟偏移导致本地密钥全部失效，DataNode 需运行一个后台监控任务：
* 每 10 秒检查最近 100 次 Token 验签的失败率。
* 若失败率超过 5%，则在下一次心跳请求中将 `request_full_keys` 置为 true，NameNode 将无视常规下发策略，直接在响应中返回完整的 Active 和 Standby 密钥。
* 此机制确保即便错过多次心跳导致密钥过期滞后，DataNode 也能快速恢复验签能力，避免 Client 请求被大量拒绝。

---

## 3. 容错与重试机制 (Resilience)

底层网络是不可靠的，DataNode 的心跳模块必须具备极强的韧性：

* **NameNode 闪断重连：** 如果 `client.heartbeat()` 失败，DataNode 不应该崩溃，而是应该打印警告日志，并在下一个 3 秒周期继续尝试。
* **指数退避 (Exponential Backoff)：** (高级特性) 如果 NameNode 长时间宕机，DataNode 的重试间隔可以从 3 秒逐渐退避到 30 秒，避免 NameNode 刚重启时遭遇“心跳风暴（Heartbeat Storm）”而雪崩。
* **僵尸块处理：** 如果在执行 `BlockReport` 时，发现本地有一个 Block，但 NameNode 返回说“我不认识这个块（可能文件已被删除）”，DataNode 应该将其放入待删除队列，延期物理回收。