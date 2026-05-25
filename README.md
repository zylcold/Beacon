# Beacon

Beacon 是一个 Apple 原生的 AI Coding Agent 状态监控工具：通过 Claude Code hooks 与 opencode plugin 接入，在本地聚合状态，最小化采集后同步到 CloudKit，并在 iPhone 与 macOS MenuBar 实时展示。

## 1) 完整系统架构

### 分层
1. **Hook / Plugin Layer（采集侧）**
   - Claude Code Hook（`session_start/tool_use/notification/waiting_input/permission_required/stop/error`）
   - opencode Plugin（`session.status/session.idle/tool.execute.before/tool.execute.after/permission.ask/session.error`）
   - 只允许调用本地 `beaconctl emit`。

2. **Beacon Agent（macOS MenuBar App + Daemon 能力）**
   - UDS/localhost 接收事件
   - 鉴权（token + process 校验 + rate limit）
   - 数据脱敏与标准化
   - 本地状态聚合与持久化
   - CloudKit 增量同步

3. **Shared Core SDK（跨端共享）**
   - 领域模型、状态映射、签名与校验、队列协议

4. **Cloud Sync Layer（CloudKit Private DB）**
   - Record upsert / pull / subscription
   - 冲突解决（LWW + monotonic 时间戳）

5. **iOS App**
   - SwiftUI 实时展示当前 Agent 状态
   - 按项目/设备/会话浏览

## 2) 数据流

1. Hook/Plugin 捕获事件 -> 执行固定命令：
   `beaconctl emit --source <source> --event <event> --session <id> ...`
2. `beaconctl` 将 payload 写入 UDS。
3. Beacon Agent 进行 token + nonce + timestamp 校验。
4. 通过 `EventNormalizer` 映射为统一 `AgentEvent`。
5. `StatusReducer` 聚合为 `AgentSession` 与 `AgentStatus`。
6. 入本地 `EventQueue/SyncQueue`。
7. `CloudSyncEngine` 批量上传到 CloudKit。
8. iOS/macOS 通过 CloudKit subscription 拉取变更并刷新 UI。

## 3) Swift Package 结构

```text
Packages/
  BeaconCore/                 # 模型、状态机、协议
  BeaconSecurity/             # keychain、签名、nonce/timestamp
  BeaconIPC/                  # uds/http server + client + token auth
  BeaconStorage/              # SwiftData/SQLite repository
  BeaconSync/                 # CloudKit sync engine + subscription
  BeaconHooks/                # hook/plugin payload parser
  BeaconUIComponents/         # 跨端状态卡片与视觉组件
Apps/
  BeaconAgent-macOS/
  Beacon-iOS/
Tools/
  beaconctl/
  hooks/claude-code/
  plugins/opencode/
```

## 4) CloudKit Schema

**Container**: `iCloud.dev.beacon`（Private Database）

**RecordType: `AgentStatus`**
- `id: String`（主键）
- `source: String`（claude-code/opencode）
- `machineId: String`
- `machineName: String`
- `projectHash: String`
- `projectName: String`（可选，用户授权后显示）
- `sessionId: String`
- `status: String`（running/idle/waiting_input/permission_required/syncing/error/stopped）
- `lastEvent: String`
- `updatedAt: Date`
- `signature: Bytes`

**索引建议**
- `machineId + updatedAt`
- `projectHash + updatedAt`
- `status + updatedAt`

## 5) IPC 设计

### 优先 UDS
- Socket: `/var/run/beacon-agent.sock`
- 消息：JSON Lines（每条带 length + checksum）

### 备选 localhost HTTP
- `127.0.0.1:6123`
- 仅开发模式启用

### 安全约束
- 本机限定
- 短期 token（Keychain 存储，按项目隔离）
- process 校验（pid + executable path allowlist）
- rate limit（session 与 source 双维度）

## 6) Keychain 设计

### Keychain 项目
- `beacon.device.privateKey`
- `beacon.device.publicKey`
- `beacon.ipc.sharedToken.<projectHash>`
- `beacon.sync.cursor`

### 策略
- 首次启动生成 P-256 密钥对
- 私钥不可导出
- token 轮转（24h）

## 7) 签名机制

- `payloadDigest = SHA256(canonicalPayload)`
- `signature = P256.Signing.PrivateKey.sign(payloadDigest)`
- 上传字段包含：`nonce`、`timestamp`、`signature`、`keyId`
- 服务端（CloudKit 消费侧）与本地回放校验：
  - timestamp 窗口（例如 ±120 秒）
  - nonce 去重（本地 cache + ttl）

## 8) Claude Code Hook 示例

```bash
#!/usr/bin/env bash
set -euo pipefail
# 固定命令，不拼接任意 shell
/usr/local/bin/beaconctl emit \
  --source claude-code \
  --event tool_use \
  --session "$CLAUDE_SESSION_ID" \
  --project "$PWD" \
  --ts "$(date -u +%s)"
```

## 9) opencode plugin 示例

```js
export default {
  name: 'beacon-plugin',
  onEvent(event, ctx) {
    const map = {
      'session.status': 'running',
      'session.idle': 'idle',
      'permission.ask': 'permission_required',
      'session.error': 'error'
    }
    const mapped = map[event.type]
    if (!mapped) return
    // 固定执行 beaconctl，不上传 prompt/代码
    ctx.execFile('/usr/local/bin/beaconctl', [
      'emit', '--source', 'opencode', '--event', event.type,
      '--session', event.sessionId, '--status', mapped
    ])
  }
}
```

## 10) SwiftUI 页面结构

- `DashboardView`
  - Running Agent 卡片
  - 最近事件时间线
  - 设备状态条
- `SessionListView`
  - 按项目分组
- `SessionDetailView`
  - 当前状态 + 最近 N 条安全事件
- `SettingsView`
  - 授权项目管理、同步开关、设备信息

## 11) MenuBar 架构

- `MenuBarExtra` 展示当前全局状态点（绿/黄/红）
- 快捷菜单：
  - 当前运行项目
  - 最近错误
  - 暂停/恢复同步
  - 打开详细窗口

## 12) 项目目录结构

```text
Beacon/
  Apps/
    BeaconAgent-macOS/
    Beacon-iOS/
  Packages/
    BeaconCore/
    BeaconSecurity/
    BeaconIPC/
    BeaconStorage/
    BeaconSync/
    BeaconHooks/
    BeaconUIComponents/
  Tools/
    beaconctl/
    hooks/claude-code/
    plugins/opencode/
  Docs/
    Architecture.md
    ThreatModel.md
    CloudKitSchema.md
```

## 13) MVP 开发顺序

1. 定义 `BeaconCore` 模型与状态映射
2. 实现 `beaconctl emit` + UDS
3. 实现 macOS Agent 接收与本地聚合
4. 接入 SwiftData/SQLite（事件队列 + 重试队列）
5. 接 CloudKit 上传/拉取
6. iOS Dashboard 与 Session 列表
7. MenuBar 实时状态与错误提示
8. 项目授权流与权限管理

## 14) macOS 权限问题

- MenuBar App 常驻前台代理进程
- 文件系统仅访问用户授权项目路径
- 若需扫码配对（二期）再引入 Camera 权限
- 避免 Full Disk Access 依赖

## 15) 后台保活

- 通过 MenuBar app 生命周期常驻
- 使用 `LaunchAgent` 保证异常退出后自恢复
- 队列持久化确保崩溃可恢复

## 16) CloudKit Subscription

- 对 `AgentStatus` 建立 `CKQuerySubscription`
- 订阅条件：`updatedAt > lastCursor`
- 通知到达后增量拉取并更新本地 cache
- 网络不可用时指数退避重试

## 17) 本地缓存与重试

- `EventQueue`: 原始事件持久化，防止 IPC 短断丢失
- `SyncQueue`: 待同步记录（带 retryCount/nextRetryAt）
- 重试策略：指数退避 + 抖动 + 上限熔断
- 冲突策略：同 session 使用 `updatedAt` 最新写入 + 事件序号兜底

---

## 安全边界总结

严格不采集：Prompt、代码内容、diff、环境变量、完整命令行。

仅采集：来源、会话 ID、状态枚举、哈希化项目标识、时间戳与签名。
