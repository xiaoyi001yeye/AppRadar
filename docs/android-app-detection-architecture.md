# 安卓应用运行探测与监控平台架构设计

## 1. 目标与范围

本方案设计一个“端 + 云 + Web 控制台”的完整系统，用于：

1. 在安卓手机端采集“当前运行/活跃应用”信息（含前台切换、近期活跃统计等）。
2. 将采集结果可靠上报至监控服务端。
3. 在监控端页面中进行实时/准实时展示与历史分析。
4. 具备可扩展性、安全性、低功耗与合规能力（用户授权、最小化采集）。

> 说明：Android 对“后台读取其它应用运行状态”限制严格，需基于合法权限与用途实现（Usage Access、Accessibility 事件、前台服务等），并符合隐私政策与应用市场规范。

---

## 2. 总体架构

```mermaid
flowchart LR
    A[Android Agent SDK/App] -->|HTTPS + JWT/mTLS| G[API Gateway]
    G --> I[Ingestion Service]
    I --> Q[(Kafka/Pulsar)]
    Q --> P[Stream Processor]
    P --> R[(Redis Cache)]
    P --> T[(TSDB/ClickHouse)]
    P --> O[(Object Storage Raw Logs)]

    T --> B[Backend Query API]
    R --> B
    B --> W[Web Monitoring Console]

    G --> AA[Auth Service]
    B --> AA
    I --> M[Device Registry Service]
    B --> M

    subgraph Ops
      X[Prometheus/Grafana]
      Y[ELK/OpenSearch]
      Z[Alertmanager]
    end

    I --> X
    P --> X
    B --> X
    W --> X
    I --> Y
    P --> Y
    B --> Y
    W --> Y
    X --> Z
```

### 2.1 逻辑分层

- **采集层（Android Agent）**：事件监听、数据聚合、离线缓存、压缩加密上报。
- **接入层（Gateway + Ingestion）**：鉴权、限流、幂等、批量写入消息队列。
- **计算层（Stream Processor）**：去重、会话化、指标聚合、异常检测。
- **存储层（热+冷）**：Redis（实时视图）、ClickHouse/TSDB（历史查询）、对象存储（原始日志归档）。
- **展示层（Web Console）**：实时列表、设备详情、时间线、Top App 排行、告警面板。
- **运维安全层**：监控、日志、告警、审计、密钥管理。

---

## 3. Android 端设计

## 3.1 采集能力组合（建议分级）

1. **基础方案（推荐默认）**：
   - `PACKAGE_USAGE_STATS`（Usage Access）
   - 周期拉取 `UsageStatsManager` + `queryEvents`
   - 获取前台切换与应用活跃时长（近实时，精度中等）

2. **增强方案（可选）**：
   - `AccessibilityService` 监听窗口变化事件（高实时）
   - 适用于企业内管控场景（需强授权提示与合规评估）

3. **补充能力**：
   - `ACTIVITY_RECOGNITION`（可选，用于人机状态判定）
   - 电量、网络、设备状态辅助字段

## 3.2 Android 组件划分

- `CollectorService`：前台服务，负责调度采集任务。
- `UsageEventReader`：读取 `UsageEvents`，转换为统一事件。
- `AccessibilityEventReader`（可选模块）：监听窗口变化实时事件。
- `SessionAggregator`：将离散事件聚合为应用会话（start/end/duration）。
- `LocalQueue`：SQLite/Room 持久化待上传数据。
- `Uploader`：批量压缩、签名、重试上传。
- `PolicyManager`：下发配置（采样率、上传间隔、黑白名单）。

## 3.3 端上数据模型

```json
{
  "event_id": "uuid-v7",
  "device_id": "hashed_android_id",
  "user_id": "optional_enterprise_uid",
  "package_name": "com.example.app",
  "app_label": "Example",
  "event_type": "APP_FOREGROUND|APP_BACKGROUND|HEARTBEAT",
  "ts": 1760000000123,
  "duration_ms": 23000,
  "network": "WIFI|CELL",
  "battery": 0.82,
  "os_version": "14",
  "sdk_int": 34,
  "source": "USAGE_STATS|ACCESSIBILITY",
  "trace_id": "..."
}
```

## 3.4 关键实现细节

- **低功耗策略**：
  - 前台服务 + 自适应轮询（屏亮/解锁时高频；熄屏低频）。
  - WorkManager 做兜底上报，不做高频实时。
- **离线可靠性**：
  - Room 队列按批次确认 ACK 删除。
  - 指数退避重试（1s/2s/4s...上限 15m）。
- **幂等与去重**：
  - `event_id` 全局唯一；服务端按 `(device_id,event_id)` 幂等。
- **时间准确性**：
  - 记录 `device_ts` 与 `server_received_ts`，便于时钟漂移校正。

### 3.4.1 降低用户感知的端侧设计（新增）

> 目标：在不牺牲合规与数据质量的前提下，最大限度降低用户对 Agent 持续运行的体感（打扰、耗电、卡顿、网络突发）。

1. **权限最小化与渐进授权**
   - 默认仅启用 `Usage Access`，`AccessibilityService` 仅在明确业务需要时再引导开启。
   - 首次仅申请“当前功能必需权限”，高级能力按需二次授权，避免一次性高敏权限带来的强感知。

2. **通知最小打扰（Foreground Service）**
   - 前台服务通知使用低优先级频道（`IMPORTANCE_LOW`），常驻但不响铃、不震动、不弹 Heads-up。
   - 通知文案简洁中性（如“服务运行中”），避免频繁变更文案导致状态栏闪动。
   - 非关键状态不主动发额外通知，错误与重试优先静默恢复。

3. **采集与上传“机会化”调度**
   - 事件采集以系统事件驱动为主，轮询兜底为辅，减少无效唤醒。
   - 上传采用批量 + 延迟容忍：优先在 `充电/Wi-Fi/设备空闲` 窗口发送。
   - 屏亮与交互高峰期降频上传，避免与前台应用争抢 CPU/网络。

4. **性能与功耗预算护栏**
   - 设定端侧预算：CPU 占用、单日耗电占比、后台网络流量上限，超阈值自动降采样。
   - 引入自适应策略：低电量（如 <20%）时仅保留关键事件，暂停非关键增强采集。

5. **网络与存储感知优化**
   - 数据先聚合再压缩上报（如按会话合并），减少小包高频传输。
   - 本地队列采用上限与淘汰策略（TTL + 容量水位），避免异常情况下占用过多存储。

6. **可配置“低感知模式”策略包（建议默认）**
   - 策略参数：`sample_interval`、`upload_batch_size`、`upload_window`、`max_daily_data_mb`。
   - 默认开启“平衡模式”：轻度实时 + 强约束功耗；控制台可远程切换“高实时/低感知”档位。

7. **透明可控以减少主观反感**
   - 设置页展示“为何采集、采什么、不采什么、如何关闭”，降低用户不确定性。
   - 提供一键暂停（例如 2/8/24 小时）与彻底退出开关，提升可控感并减少投诉。

8. **效果评估（低感知 KPI）**
   - `fg_notification_complaint_rate`：前台通知相关投诉率。
   - `agent_daily_battery_pct_p50/p95`：Agent 日耗电分位数。
   - `agent_bg_cpu_ms_per_hour`：后台每小时 CPU 时长。
   - `user_pause_rate`：用户主动暂停率（过高说明体感仍强）。

## 3.5 Agent 端可发现性设计（跨网络快速发现与数据接收）

> 目标：让 Agent 在 Wi-Fi、蜂窝网络、同网段局域网、NAT/防火墙隔离等不同网络环境下，都可以被“接收端”快速发现，并尽快建立稳定数据通道。

### 3.5.1 设计原则

1. **多通道发现并存**：本地广播发现 + 云端目录发现 + 反向长连接发现。
2. **先发现、后鉴权、再收数**：发现不等于可信，接收数据前必须完成身份校验。
3. **按网络环境自适应**：自动识别网络类型，选择最低成本、最高成功率的发现路径。
4. **秒级可达与退化可用**：理想情况下秒级发现；异常时可退化为轮询/目录模式。

### 3.5.2 三层发现架构

```mermaid
flowchart TD
    subgraph LocalNet[同网段局域网]
      A1[mDNS/Bonjour 广播]
      A2[UDP Beacon]
      A3[被动监听 + 主动探测]
    end

    subgraph Cloud[跨网/异网]
      B1[Device Registry 心跳注册]
      B2[能力目录查询 API]
      B3[推送在线状态变更 Webhook/SSE]
    end

    subgraph Tunnel[NAT/受限网络]
      C1[Agent -> Relay 的 WebSocket/MQTT 长连接]
      C2[接收端通过 Relay 反向寻址]
    end

    R[Receiver] --> A3
    R --> B2
    R --> C2
```

### 3.5.3 场景化策略矩阵

| 网络场景 | 首选发现方式 | 备选方式 | 典型时延 |
|---|---|---|---|
| 同 Wi-Fi / 同子网 | mDNS 服务发现 | UDP Beacon + 端口探测 | 0.5~2s |
| 不同子网（企业内网） | 云端 Registry 查询 | SSE 推送在线状态 | 1~5s |
| 蜂窝网络 / CGNAT | 反向长连接（WS/MQTT） | 短轮询目录服务 | 2~8s |
| 严格防火墙环境 | 443 端口 TLS 长连接 | 代理网关转发 | 3~10s |

### 3.5.4 Agent 广播与注册协议

1. **局域网广播（可选启用）**
   - Agent 周期发布 `service=_appradar-agent._tcp.local`（mDNS TXT 含 `device_id_hash`,`ver`,`cap`）。
   - 同时每 2~5 秒发送轻量 UDP Beacon（仅含匿名标识与能力摘要）。

2. **云端注册（默认启用）**
   - Agent 启动后向 `Device Registry` 注册并维持心跳（例如 15s/次）。
   - 注册信息：`device_id_hash`、`network_type`、`reachable_mode(lan|relay|both)`、`last_seen`、`capabilities`。

3. **反向可达（受限网络关键）**
   - Agent 主动连 `Relay`（WSS:443），保持低频心跳。
   - 接收端只需知道 `device_id`，即可通过 Relay 路由到在线 Agent。

### 3.5.5 Receiver 快速发现流程（建议）

1. **并行启动三路发现**：`LAN` 扫描、`Registry` 查询、`Relay` 在线探测。
2. **首个命中即建立通道**：优先级 `LAN 直连 > Relay > 云中转拉取`。
3. **短期缓存发现结果**：缓存 30~120 秒，避免重复探测。
4. **链路健康检查**：若 RTT/丢包升高，自动切换到次优路径。

### 3.5.6 统一发现数据结构（示例）

```json
{
  "device_id": "hashed_android_id",
  "display_name": "Pixel-01",
  "online": true,
  "reachable_mode": "lan|relay|both",
  "lan_endpoint": "192.168.1.10:46001",
  "relay_endpoint": "wss://relay.example.com/session/abc",
  "capabilities": ["events.push", "batch.upload", "delta.sync"],
  "network_type": "WIFI|CELL",
  "last_seen": 1760000000123,
  "priority": 10
}
```

### 3.5.7 安全控制（发现链路）

- 局域网广播包不放敏感信息，仅放匿名设备标识与版本能力。
- 发现后建立连接时执行双向认证（JWT + 设备签名，企业场景可加 mTLS）。
- 每次会话下发短时令牌（1~5 分钟）用于接收数据，过期即失效。
- 对发现请求与连接建立执行频率限制，防止扫描与放大攻击。

### 3.5.8 可观测性指标（Discovery SLI/SLO）

- `discovery_success_rate`（5 分钟窗口）
- `discovery_p95_latency_ms`
- `first_packet_after_discovery_ms`
- `path_switch_count`（LAN/Relay 切换次数）
- `relay_online_ratio`

建议目标：`discovery_success_rate >= 99%`，`p95_latency <= 5s`（企业网络可按区域分层定义）。

---

## 4. 服务端设计

## 4.1 API 设计

### 采集上报接口

- `POST /api/v1/ingest/events`
- Headers: `Authorization: Bearer <token>`, `X-Device-Id`, `X-Signature`
- Body: gzip + json array（建议 100~500 条/批）

返回：

```json
{
  "accepted": 490,
  "duplicated": 10,
  "rejected": 0,
  "server_time": 1760000000999
}
```

### 监控查询接口

- `GET /api/v1/devices/{id}/timeline?from=&to=`
- `GET /api/v1/apps/top?scope=device|group&window=1h`
- `GET /api/v1/realtime/active?group_id=`（可走 WebSocket/SSE）

## 4.2 数据流处理

1. Ingestion 校验签名、token、字段合法性。
2. 写入 Kafka Topic（按 `device_id` 分区，保持设备内有序）。
3. 流处理作业：
   - 去重（状态表/布隆过滤器 + 短期 Redis）
   - 会话拼接（foreground/background）
   - 指标聚合（1min/5min）
4. 输出：
   - Redis：最新活动 app（秒级查询）
   - ClickHouse：历史明细+聚合
   - 对象存储：原始归档（审计/回放）

## 4.3 存储模型建议

### ClickHouse 表（示例）

- `app_events_raw`
  - 主键：`(event_date, device_id, ts)`
  - 分区：`toYYYYMM(event_date)`
- `app_sessions`
  - 字段：`device_id, package_name, start_ts, end_ts, duration_ms`
- `agg_app_usage_1m`
  - 字段：`minute_bucket, package_name, active_devices, total_duration_ms`

### Redis Key（示例）

- `device:{id}:current_app` -> `{"pkg":"...","ts":...}` TTL 10m
- `group:{id}:active_devices` -> Sorted Set

---

## 5. Web 监控端设计

## 5.1 页面结构

1. **实时总览**
   - 在线设备数、活跃应用 TopN、事件吞吐、延迟分位数。
2. **设备详情**
   - 当前应用卡片、最近切换时间线、最近 24h 使用分布图。
3. **应用画像**
   - 按应用查看活跃设备趋势、平均停留时长、峰值时段。
4. **告警中心**
   - 规则：异常应用出现、黑名单命中、静默设备超时。

## 5.2 前端技术建议

- React + TypeScript + Ant Design（或 Vue3 + Element Plus）
- 图表：ECharts
- 实时通道：SSE（简单）或 WebSocket（双向）
- 状态管理：Redux Toolkit/Pinia

## 5.3 查询性能

- 列表接口默认分页 + 时间窗口限制。
- 高并发榜单走预聚合表（1m/5m）而非实时扫明细。
- 热点设备详情可加 API 缓存（5~15s）。

---

## 6. 安全与合规

- **最小化采集原则**：仅采包名/时长/时间戳，不采内容数据。
- **传输安全**：TLS1.2+，可升级 mTLS（企业设备）。
- **设备身份**：首次注册发放 device token，周期轮换。
- **隐私合规**：首次启动明确告知用途与权限，支持用户撤回。
- **访问控制**：RBAC（组织/项目/设备分级授权）。
- **审计**：所有查询与导出行为写审计日志。

---

## 7. 可靠性与可运维性

- SLO 示例：
  - 事件接收成功率 >= 99.9%
  - P95 端到端展示延迟 <= 10s
- 灰度发布：
  - 按设备分组灰度新采样策略
- 灾备：
  - Kafka 多副本、ClickHouse 副本分片、对象存储跨区
- 监控指标：
  - ingest_qps, ingest_error_rate, queue_lag, ws_connections, dashboard_p95

---

## 8. MVP 分阶段落地

## Phase 1（2~4 周）

- Android：UsageStats 采集 + 本地队列 + 批量上传
- 服务端：Ingestion + ClickHouse 落库 + 基础查询 API
- Web：设备列表 + 设备时间线 + Top 应用

## Phase 2（4~8 周）

- 实时通道（SSE/WebSocket）
- Redis 热缓存 + 聚合作业
- 告警规则引擎（简单阈值）

## Phase 3（8 周+）

- Accessibility 增强采集（可配置）
- 多租户/RBAC 完整权限体系
- 智能分析（异常行为模型）

---

## 9. 风险与应对

1. **Android 权限限制导致采集不稳定**
   - 多源融合（Usage + Accessibility 可选）
   - 明确引导权限开启与健康检查页
2. **电量消耗增加**
   - 动态采样 + 批量上传 + 熄屏降频
3. **数据量突增**
   - 网关限流 + 消息队列削峰 + 预聚合
4. **合规风险**
   - 文档化授权、数据脱敏、留存周期管理

---

## 10. 建议的代码仓结构（Monorepo）

```text
AppRadar/
  android-agent/
    app/
    collector/
    uploader/
  backend/
    gateway/
    ingestion/
    query-api/
    stream-jobs/
  web-console/
  infra/
    k8s/
    helm/
    terraform/
  docs/
    android-app-detection-architecture.md
```

---

## 11. 样例时序（设备上报到页面展示）

```mermaid
sequenceDiagram
    participant D as Android Device
    participant G as API Gateway
    participant I as Ingestion
    participant K as Kafka
    participant S as Stream Job
    participant R as Redis
    participant C as ClickHouse
    participant W as Web Console

    D->>G: POST /ingest/events (batch)
    G->>I: forward + auth context
    I->>K: publish(device_id partition)
    S->>K: consume
    S->>R: upsert current_app
    S->>C: insert session/event
    W->>R: query realtime current_app
    W->>C: query historical timeline
```
