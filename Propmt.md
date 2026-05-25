你是一个资深 Apple 平台架构师、Swift 工程师、AI Coding Tool 工程师。

现在需要从 0 开发一个 Apple 平台开发者工具：

项目名：
Beacon

产品定位：
一个用于监控 AI Coding Agent 状态的 Apple 原生工具。

Beacon 通过 plugin / hook 接入：

* Claude Code
* opencode

实时采集 Agent 状态，
并安全同步到 CloudKit，
在 iPhone 和 macOS 上展示 Agent 当前运行状态。

目标体验类似：

* Activity Monitor
* Docker Desktop
* OrbStack
* Find My Presence
* Dynamic Island Live Status

整个产品必须强调：

* Apple 原生体验
* 极简 UI
* 安全
* 本地优先
* 最小权限
* 实时状态感

技术要求

必须使用：

* Swift
* SwiftUI
* CloudKit
* App Group
* Keychain
* Structured Concurrency
* AsyncSequence
* SwiftData 或 SQLite
* macOS MenuBar App

允许：

* 少量 shell
* 少量 Node.js（仅 opencode plugin）

禁止：

* Electron
* React Native
* Firebase
* 第三方后端
* 上传 Prompt
* 上传代码
* 上传 diff
* 上传环境变量

系统架构

Beacon 分为：

1. Hook / Plugin Layer
2. Beacon Agent（macOS）
3. Cloud Sync Layer
4. iOS App
5. Shared Core SDK

一、Hook / Plugin Layer

Claude Code：
使用官方 hooks。

监听：

* session_start
* tool_use
* notification
* waiting_input
* permission_required
* stop
* error

opencode：
使用官方 plugin system。

监听：

* session.status
* session.idle
* tool.execute.before
* tool.execute.after
* permission.ask
* session.error

Plugin 层禁止：

* 访问 CloudKit
* 直接联网
* 上传数据
* 读取 Keychain

Plugin 层只能：

* 调用本地 CLI
* 或 localhost/unix socket

例如：

beacon emit 
–source claude-code 
–event tool_use 
–session xxx

二、Beacon Agent（macOS）

实现一个 macOS MenuBar App：

名称：
Beacon Agent

职责：

* 接收 plugin/hook 事件
* 本地状态聚合
* 项目授权
* 安全校验
* 数据脱敏
* CloudKit 同步
* 本地缓存
* 多 Session 管理
* 设备身份管理

IPC

优先：

* unix domain socket

其次：

* localhost http

必须：

* 本机限定
* token 校验
* process 校验
* rate limit

安全要求

必须实现：

* Keychain 保存 device private key
* device public/private key 机制
* payload 签名
* nonce 防重放
* timestamp 校验
* 项目路径 hash 化
* 不上传绝对路径
* 不上传 prompt
* 不上传文件内容
* 不上传 shell 命令
* 不上传环境变量

项目授权

首次发现项目：

弹出授权：

“是否允许 Beacon 监控此项目状态？”

支持：

* 永久允许
* 临时允许
* 拒绝

Hook 安全

禁止：

* eval
* 动态 shell 拼接
* 下载执行脚本
* 任意命令执行

所有 hook command 必须固定。

三、Cloud Layer

使用：
CloudKit private database

Container：

iCloud.dev.beacon

Record Type：

AgentStatus

字段：

* id
* source
* machineId
* machineName
* projectHash
* projectName
* sessionId
* status
* lastEvent
* updatedAt
* signature

同步要求：

* 增量同步
* 自动重试
* 离线缓存
* 冲突处理
* 多设备同步
* CloudKit Subscription

四、iOS App

使用：
SwiftUI

产品名：
Beacon

首页

展示：

* 当前运行中的 Agent
* Agent 来源
* 项目名称
* 当前状态
* 最后更新时间
* 当前 Mac

状态类型

统一抽象：

* running
* idle
* waiting_input
* permission_required
* syncing
* error
* stopped

UI 风格

风格参考：

* Apple Developer Tools
* Activity Monitor
* Find My
* Dynamic Island
* visionOS 风格卡片

要求：

* 深色模式优先
* 毛玻璃
* 呼吸灯状态
* 极简
* 高信息密度

MenuBar

MenuBar 需要：

* 实时状态点
* 最近 Agent
* 当前运行项目
* 快速暂停同步
* 快速查看错误

Dynamic Island / Live Activity

预留：

* Agent Running
* Waiting For Input
* Build Failed
* Claude Needs Permission

五、数据模型

请设计：

* AgentEvent
* AgentSession
* AgentStatus
* DeviceIdentity
* ProjectPermission
* SyncQueue

六、状态聚合

Claude Code 和 opencode 的状态不同。

需要统一映射。

例如：

Claude：

* Notification
* PostToolUse
* Stop

opencode：

* session.status
* session.idle

统一抽象为：

* running
* idle
* waiting
* error

七、本地存储

使用：
SwiftData 或 SQLite

需要：

* Event Queue
* Retry Queue
* Crash Recovery
* Session Cache

八、配对机制

第一版：

基于：

* 同一 Apple ID
* 同一个 CloudKit Container

自动发现设备。

第二版：

支持二维码配对：

流程：

1. iOS App 生成 pairing token
2. macOS 扫码
3. public key exchange
4. 建立 trust relationship

九、CLI

CLI 名称：

beaconctl

命令：

beacon status
beacon emit
beacon sessions
beacon devices
beacon logs

十、项目结构

请设计：

* Swift Package 拆分
* App 模块结构
* Shared Core
* IPC Layer
* Sync Layer
* Security Layer

十一、输出内容

请输出：

1. 完整系统架构
2. 数据流
3. Swift Package 结构
4. CloudKit Schema
5. IPC 设计
6. Keychain 设计
7. 签名机制
8. Hook 示例
9. opencode plugin 示例
10. SwiftUI 页面结构
11. MenuBar 架构
12. 项目目录结构
13. MVP 开发顺序
14. macOS 权限问题
15. 后台保活
16. CloudKit Subscription
17. 本地缓存与重试
18. 安全审计建议
19. 性能优化建议
20. 上架 App Store 风险

最终目标：

实现一个真正可上线、Apple 原生风格、长期可维护、安全的 AI Agent 状态系统：

Beacon。