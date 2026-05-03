# 安全与访问控制 (Security & Access Control)

在 RDFS 的流式管线架构中，NameNode 与 DataNode 的数据流是严格物理隔离的。为了防止恶意客户端绕过 NameNode，直接通过伪造或盲猜 `block_id` 向 DataNode 发起非法的 `ReadBlock` 或 `WriteBlock` 请求，系统引入了基于 **HMAC-SHA256** 的 **Block Token (Access Token)** 机制。
该机制允许 DataNode 在不向 NameNode 发起反向 RPC 查询的情况下，利用本地缓存的同步密钥，对数据流的首包进行“零开销”的去中心化验签。

---

## 1. 令牌结构 (Token Structure)

一个完整的 Block Token 由 **Payload (载荷)** 和 **Signature (签名)** 两部分组成。在网络传输时，通常将序列化并签名后的 Token 以 `bytes` 或 Base64 编码的 `string` 形式包裹在 gRPC 请求中。

### 1.1 核心载荷
载荷中的字段命名严格对齐全局 RPC 规范：
* **`block_id`** (uint64)：该令牌授权操作的全局唯一数据块 ID。
* **`gen_stamp`** (uint64)：签发该令牌时 NameNode 内存中该 Block 当前的世代版本号。
* **`client_id`** (string)：申请该令牌的客户端 UUID。
* **`access_mode`** (uint32)：权限位掩码（例如：`1`=READ, `2`=WRITE, `4`=COPY）。
* **`expiry_date`** (int64)：令牌失效的 Unix 时间戳。
* **`key_id`** (uint32)：用于标识 NameNode 签发该令牌时所使用的 MasterKey 版本号。

### 1.2 签名计算
签名是对 Payload 进行哈希运算的结果，确保客户端无法篡改载荷中的权限或过期时间：

```text
Signature = HMAC-SHA256(MasterKey_id, Payload)
```

### 1.3 使用次数
* **写入令牌：** 单次有效，每个写入 Token 与特定的 `gen_stamp` 绑定，仅限当次写入管线的建立使用。一旦管线中断或文件关闭，该 Token 立即失效，Client 必须从 NameNode 重新申请。
* **读取令牌：** 重复使用，只要在 `expiry_date` 之前且 `block_id` 未发生变更，Client 可多次使用同一个读取 Token 向不同 DataNode 发起读取请求，无需重复申请。

---

## 2. 核心工作流

### 2.1 签发流程 (NameNode -> Client)
NameNode 充当系统的“发证机关”：
* **读取场景 (`GetFileInfo`)：**
  当客户端请求读取文件时，NameNode 在返回的 `List<LocatedBlock>` 中，为每一个 `LocatedBlock` 附带一个具有 READ 权限的 Token。
* **写入场景 (`AllocateBlock`)：**
  当客户端申请新块时，NameNode 生成 `block_id` 和 `gen_stamp` 的同时，签发一个具有 WRITE 权限的 Token，打包在 `LocatedBlock` 中返回。

### 2.2 验证流程 (Client -> DataNode)
DataNode 作为“安检员”，拦截并校验所有的 I/O 请求：
* **写入安检 (`WriteBlock` Stream)：**
  客户端在发起 `WriteBlock` 流式 RPC 时，流的第一个包为 `WriteHeader`。客户端必须将 Token 放入 `WriteHeader` 中。DataNode 收到首包后执行以下校验，若任一步骤失败，直接中断 Stream 并返回 `UNAUTHENTICATED` 或 `PERMISSION_DENIED` 状态码。
  * 解析 Token 的 `identifier`，检查签名 HMAC 是否与本地缓存的 MasterKey(`key_id`) 一致。
  * 校验 `block_id` 是否与请求写入的目标块匹配。
  * 校验 `gen_stamp` 是否等于该 DataNode 本地存储的该 Block 的 `gen_stamp`（若本地尚无该 Block，则直接通过，因为这是全新的写入）。
  * 检查权限位是否包含 WRITE，以及是否在有效期内。
* **读取安检 (`ReadBlock` Unary Request)：**
  客户端在发起 `ReadBlock` 一元请求时，直接在 Request 参数中携带 Token。DataNode 执行类似校验，验签通过后，才开始通过 Stream 返回 `chunk_data`。
  * 验签。
  * 校验 `block_id` 匹配。
  * 校验 `gen_stamp` 必须与本地块 `gen_stamp` 完全一致（防止读到脑裂残留的旧版本数据）。
  * 检查权限位包含 READ 且未过期。

> **注意：** 若 DataNode 本地不存在该 `block_id`，读取请求直接拒绝；写入请求则允许（因为是首次创建）。

---

## 3. 密钥同步与轮转机制 (Key Rolling)

DataNode 需要知道 NameNode 的 `MasterKey` 才能进行验签。为避免性能瓶颈，RDFS 采用心跳通道进行密钥异步下发。

### 3.1 密钥下发
DataNode 定期发送 `Heartbeat`。NameNode 会将当前的 `MasterKey` 封装为一种特殊的 `DataNodeCommand`，放在心跳的 Response 中下发给 DataNode，DataNode 收到后，将其缓存在本地内存的 `SecretManager` 中。

### 3.2 密钥轮转状态机
为了降低密钥泄露的风险，NameNode 默认每 24 小时轮转一次 MasterKey。系统同时保留两个版本的密钥以保证平滑过渡：
* **Active Key (当前主钥)：** NameNode 用于为新的 `AllocateBlock` / `GetFileInfo` 请求签发新 Token。
* **Standby Key (备用密钥)：** 不再签发新 Token，但 DataNode 内存中依然保留，用于验证那些使用老密钥签发、且尚未过期的在途 Token（确保跨越轮转周期的大文件长连接下载不会被中断）。

### 3.3 密钥确认与平滑切换
为避免新签发的 Token 被未收到新密钥的 DataNode 拒绝，实现平滑轮转：
* **两阶段切换：**
  1. NameNode 生成新密钥 V2 后，标记为 `PENDING_ACTIVE`。在下一次心跳响应中，将 V2 连同现有的 Active Key 一起下发给 DataNode。
  2. NameNode 等待 **所有健康 DataNode** 都在心跳中汇报已收到 V2（通过 `request_full_keys` 或新增的 `key_ack` 字段）。
  3. 全数确认后，NameNode 将 V2 提升为 Active Key，用于签发新 Token。旧的 Active Key 降级为 Standby，并在下一轮轮转中最终淘汰。
* **过渡期保护：** 若部分 DataNode 长时间（如 10 分钟）未确认，NameNode 应发出告警，并继续用旧 Active Key 签发 Token，直到这些节点恢复或人工介入。