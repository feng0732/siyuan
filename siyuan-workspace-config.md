# SiYuan 工作空间与配置持久化机制深度分析

> 版本：v3.6.5 | 分析日期：2026-06-15 | 源码根：`kernel/` 与 `app/`

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [工作空间初始化与启动加载流程](#2-工作空间初始化与启动加载流程)
3. [配置分层模型与存储结构](#3-配置分层模型与存储结构)
4. [配置变更、运行时缓存及落盘保存](#4-配置变更运行时缓存及落盘保存)
5. [多窗口状态管理机制](#5-多窗口状态管理机制)
6. [默认值合并策略](#6-默认值合并策略)
7. [损坏配置的恢复机制](#7-损坏配置的恢复机制)
8. [跨端数据一致性与同步冲突策略](#8-跨端数据一致性与同步冲突策略)
9. [风险评估](#9-风险评估)
10. [可进一步验证的问题点](#10-可进一步验证的问题点)
11. [关键协作模块索引](#11-关键协作模块索引)

---

## 1. 整体架构概览

SiYuan 采用 **前后端分离** 的单体架构：

- **前端 (`app/`)**：TypeScript + Electron（桌面端）/ 浏览器（Web端）/ 移动端（iOS/Android/Harmony），负责 UI 渲染、布局管理、用户交互
- **后端 (`kernel/`)**：Go 语言实现的内核进程，负责数据持久化、HTTP/WebSocket 服务、业务逻辑、同步引擎（依赖外部库 `siyuan-note/dejavu`）
- **通信层**：HTTP（API 请求）+ WebSocket（实时推送）+ Electron IPC（桌面端窗口间通信）

### 关键流程数据流

```
┌─────────────┐     HTTP API      ┌─────────────┐    读写文件     ┌──────────────┐
│  Frontend   │ ───────────────>  │   Kernel    │ ─────────────> │  File System │
│ (TS/Elec)   │ <───────────────  │   (Go)      │ <───────────── │ (Workspace)  │
└──────┬──────┘   WS Broadcast    └──────┬──────┘   filelock     └──────────────┘
       │                                  │
       │         Electron IPC             │
       └──────────────────────────────────┘
```

---

## 2. 工作空间初始化与启动加载流程

### 2.1 Kernel 启动全链路

启动入口在 [kernel/main.go#L30-L59](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/main.go#L30-L59)，执行顺序严格依赖：

| 步骤 | 函数 | 核心职责 | 关键代码位置 |
|------|------|----------|-------------|
| 1 | `util.Boot()` | 解析命令行参数、初始化工作空间路径、加锁 | [util/working.go#L84-L172](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L84-L172) |
| 2 | `model.InitConf()` | 加载 conf.json、合并默认值、语言初始化 | [model/conf.go#L123-L625](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L123-L625) |
| 3 | `server.Serve()` | 启动 HTTP+WebSocket 服务、写入 port.json | [server/serve.go#L133-L273](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/server/serve.go#L133-L273) |
| 4 | `model.InitAppearance()` | 主题/图标初始化 | — |
| 5 | `sql.Init*Database()` | SQLite 数据库初始化（主库/历史/资源内容/块树） | — |
| 6 | `model.BootSyncData()` | 启动数据同步引擎（先获取云端增量→本地索引→再落盘） | [model/sync.go#L124-L158](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/sync.go#L124-L158) |
| 7 | `model.InitBoxes()` | 加载所有笔记本（Box）配置 | [model/conf.go#L1001-L1013](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L1001-L1013) |
| 8 | `util.SetBooted()` | 标记启动完成，清理进度遮罩 | — |

### 2.2 工作空间路径决策

路径决策逻辑在 [util/working.go#L246-L322](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L246-L322)，优先级如下：

1. **命令行参数** `--workspace` > 环境变量 `SIYUAN_WORKSPACE_PATH`
2. **workspace.json** 最后一条记录（最近使用），位于 `~/.config/siyuan/workspace.json`
3. **平台默认路径**：
   - Windows: `%USERPROFILE%/SiYuan`
   - macOS: `~/Library/Application Support/SiYuan`
   - Linux/其他: `~/SiYuan`

### 2.3 工作空间独占锁机制

通过 `flock` 文件锁确保同一工作空间仅被一个内核实例占用：

```go
// util/working.go : tryLockWorkspace
WorkspaceLock = flock.New(filepath.Join(WorkspaceDir, ".lock"))
ok, err := WorkspaceLock.TryLock()   // 非阻塞尝试加锁
if !ok { os.Exit(ExitCodeWorkspaceLocked) }
```

退出时通过 [util/working.go#L537-L551](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L537-L551) `UnlockWorkspace` 释放锁并删除 `.lock` 文件。

### 2.4 前端启动与配置获取

前端在 [app/src/index.ts#L217-L242](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/index.ts#L217-L242) 中按以下链路初始化：

```
fetchPost(/api/system/getConf)
   └→ 加载插件 loadPlugins()
      └→ 读取 localStorage
         └→ GET /appearance/langs/{lang}.json
            └→ 获取云用户 getCloudUser
               └→ onGetConfig() → 布局反序列化、外观应用、资源初始化
```

---

## 3. 配置分层模型与存储结构

### 3.1 三级配置体系

| 层级 | 位置 | 文件名 | 结构定义 | 保存时机 |
|------|------|--------|----------|---------|
| **全局应用级** | `{workspace}/conf/` | `conf.json` | [model/conf.go#L52-L89](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L52-L89) `AppConf` | `Save()` 调用 + `InitConf()` 合并默认值后 |
| **笔记本级** | `{workspace}/data/{boxID}/.siyuan/` | `conf.json` | [conf/box.go#L22-L34](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/conf/box.go#L22-L34) `BoxConf` | 笔记本设置变更 API |
| **用户家目录级** | `~/.config/siyuan/` | `workspace.json`/`port.json`/`cookie.key` | — | 工作空间切换、端口分配、首次启动 |

### 3.2 AppConf 核心字段

`AppConf` 包含 20+ 个子配置模块，主要模块：

| 字段 | 类型 | 说明 |
|------|------|------|
| `Appearance` | `*conf.Appearance` | 主题、图标、模式（明/暗）、状态栏 |
| `Editor` | `*conf.Editor` | 字体大小、Markdown 选项、历史保留 |
| `FileTree` | `*conf.FileTree` | 文档面板、最大打开标签数、排序 |
| `UILayout` | `*conf.UILayout` | 整个界面布局 JSON（map 结构） |
| `Sync` | `*conf.Sync` | 同步模式（云端/S3/WebDAV/本地）、间隔、GenerateConflictDoc 开关 |
| `System` | `*conf.System` | 设备 ID、内核版本、网络代理、工作空间路径 |
| `Keymap` | `*conf.Keymap` | 快捷键映射（嵌套 map） |
| `AI` / `Bazaar` / `Repo` | - | AI 配置、集市、数据仓库 |
| `m` / `userLock` | `*sync.RWMutex` | 并发读写锁（**不序列化**，json 反序列化后被替换） |

### 3.3 工作空间目录结构

```
{WorkspaceDir}/
├── .lock                  # flock 独占锁文件
├── conf/
│   ├── conf.json          # 全局配置
│   └── appearance/        # 用户主题、图标
├── data/
│   ├── {boxID}/
│   │   ├── .siyuan/
│   │   │   └── conf.json  # 笔记本配置
│   │   └── {path}/**.sy   # 文档数据（JSON）
│   ├── assets/            # 图片、附件等
│   ├── templates/         # 模板
│   ├── widgets/           # 挂件
│   ├── plugins/           # 插件
│   ├── snippets/          # 代码片段
│   └── public/            # 公开访问
├── history/               # 历史版本（自动生成）
├── repo/                  # 数据仓库索引（dejavu 快照树）
└── temp/                  # 临时目录（每次启动重建）
    ├── siyuan.db          # 主 SQLite 数据库
    ├── history.db         # 历史数据库
    ├── blocktree.db       # 块树数据库
    └── asset_content.db   # 资源内容数据库
```

---

## 4. 配置变更、运行时缓存及落盘保存

### 4.1 配置变更的标准流程

所有配置变更遵循 **「内存修改 → 条件落盘 → 广播通知」** 三步曲。以设置编辑器只读状态为例，参见 [api/setting.go#L33-L52](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/setting.go#L33-L52)：

```
1. 修改内存：   model.Conf.Editor.ReadOnly = readOnly
2. 落盘保存：   model.Conf.Save()
3. 广播通知：   util.BroadcastByType("protyle", "readonly", ...)
                util.BroadcastByType("main", "readonly", ...)
```

### 4.2 Save() 保存机制的原子性与字节对比

[model/conf.go#L870-L891](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L870-L891) 实现了 **读-对比-写** 流程：

```go
func (conf *AppConf) Save() {
    if util.ReadOnly { return }
    Conf.m.Lock()                          // 1. 写锁保护
    defer Conf.m.Unlock()

    newData, _ := gulu.JSON.MarshalIndentJSON(Conf, "", "  ")  // 2. 序列化新数据
    oldData, err := filelock.ReadFile(confPath)                 // 3. 读取旧文件
    if err != nil {                    // 4a. 读失败（文件不存在/损坏）→ 直接写
        conf.save0(newData)
        return
    }
    if bytes.Equal(newData, oldData) {  // 4b. 内容未变更 → 跳过写入
        return
    }
    conf.save0(newData)                 // 4c. 有差异 → 原子写入
}
```

**关键优化**：保存前进行字节级对比，无差异时跳过磁盘 I/O。

**风险点**（TOCTOU）：`bytes.Equal` 与 `save0` 之间没有额外锁保护文件系统层面的并发写入（仅保护内存）。极端并发下若有外部进程同时写 conf.json，存在窗口。

### 4.3 filelock 原子写入

使用 `github.com/siyuan-note/filelock` 包（`save0` → `filelock.WriteFile`），写入过程为经典的两阶段：
1. 先写入 `conf.json.tmp` 临时文件
2. 执行 `rename` 原子替换原文件
3. 避免并发写入导致的半写/截断损坏

### 4.4 UILayout 布局保存的特殊路径

**主窗口布局** 保存路径与普通配置不同：
- 前端主动触发：[app/src/layout/util.ts#L128-L168](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/layout/util.ts#L128-L168) `saveLayout()`
- 调用 API：`POST /api/system/setUILayout`
- 后端处理：[api/system.go#L570-L599](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/system.go#L570-L599) → `model.Conf.SetUILayout()` → `Save()`

**子窗口布局** 保存在 `sessionStorage`（仅当前窗口生命周期），不落盘。

### 4.5 退出时的屏障顺序（已核实）

[model/conf.go#L738-L832](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L738-L832) `Close(force, setCurrentWorkspace, execInstallPkg)` 执行顺序为 **15 步严格编排**：

| 序号 | 代码位置 | 动作 | 说明 |
|------|---------|------|------|
| 1 | L744 | `FlushTxQueue()` | 冲刷文档事务队列（所有待保存的文档先写入 .sy） |
| 2 | L749 | `syncData(true, false)` | 退出前同步（非 force） |
| 3 | L758 | `closeUserGuide()` | 清理用户指南笔记本损坏状态 |
| 4 | **L761** | **`sql.FlushQueue()`** | **先将 SQLite 的写队列全部 flush 到磁盘** |
| 5 | L780 | `Conf.Close()` → `Conf.Save()` | **再落盘 conf.json**（确保已包含 Sync.Synced 等同步后更新的字段） |
| 6 | L781 | `sql.CloseDatabase()` | 关闭所有 SQLite 连接 |
| 7 | L782 | `util.SaveAssetsTexts()` | 将 OCR 文本等资源内容刷盘 |
| 8 | L783 | `clearWorkspaceTemp()` | 清理 temp/ 下的临时文件 |
| 9 | L784 | `clearCorruptedNotebooks()` | 删除检测到的损坏笔记本 |
| 10 | L785 | `clearPortJSON()` | 清除端口映射 |
| 11 | L787-L797 | 更新 workspace.json | 将当前工作空间移到列表末尾 |
| 12 | L800 | `BroadcastByType("main", "exit")` | 通知前端退出 |
| 13 | L801 | `util.UnlockWorkspace()` | 释放工作空间锁 |
| 14 | L803-L810 | `time.Sleep(500ms~30s)` | 等待 WS 消息发出、Windows 更新安装程序启动 |
| 15 | L812-L830 | `closeSyncWebSocket` + 关闭 HTTP/WS server | 关闭网络服务，`os.Exit(0)` |

**设计含义**：第 4 步（sql.FlushQueue）先于第 5 步（Conf.Save）确保——若同步过程中写入了新的数据库记录（如同步时间戳、闪卡状态），这些记录先落盘；然后 Conf.Save() 再写入包含最新 `Sync.Synced`、`Sync.Stat` 的配置，保证配置中记录的状态与数据库实际状态一致。

---

## 5. 多窗口状态管理机制

### 5.1 WebSocket 会话分组

内核通过两级 `sync.Map` 管理所有前端连接，定义于 [util/websocket.go#L30-L36](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/websocket.go#L30-L36)：

```go
sessions     = sync.Map{}  // {appId, {sessionId, *melody.Session}}
authSessions = sync.Map{}  // 授权页单独存储
```

每个 WebSocket 连接在 URL query 中携带三个标识符：

| 参数 | 含义 | 用途 |
|------|------|------|
| `app` | 应用实例 ID（每个窗口独立） | 区分不同窗口/设备 |
| `id` | 会话 ID（窗口内主/子面板） | 区分 main / protyle / filetree 等频道 |
| `type` | 频道类型 | 广播消息过滤 |

### 5.2 六种广播模式（已核实）

[util/result.go#L27-L32](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/result.go#L27-L32) 定义了六种 PushMode 常量：

| 常量 | 值 | 语义 | 分发函数 | 典型场景 |
|------|---|------|---------|---------|
| `PushModeBroadcast` | 0 | 全量广播（所有 App 所有会话） | `Broadcast()` | 文档保存、数据变更 |
| `PushModeSingleSelf` | 1 | 仅回传给请求者（appId + sessionId） | `single()` | API 请求的响应 |
| `PushModeBroadcastExcludeSelf` | 2 | 除当前会话外全部广播 | `broadcastOthers()` | 本地光标位置同步给同窗口的其他面板 |
| `PushModeBroadcastExcludeSelfApp` | 4 | 排除当前 App 的所有会话（跨 App 广播） | `broadcastOtherApps()` | 多窗口数据同步 |
| `PushModeBroadcastApp` | 5 | 同一 App 内所有会话 | `broadcastApp()` | 单窗口内多面板同步（如 main→protyle） |
| `PushModeBroadcastMainExcludeSelfApp` | 6 | 排除当前 App 外所有 main 频道 | `broadcastOtherAppMains()` | 跨窗口布局变更通知 |

六种模式在 [util/websocket.go#L383-L400](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/websocket.go#L383-L400) `PushEvent()` 中通过 `switch mode` 分发。

**注意**：`PushMode` 值 **3 不存在**（0/1/2/4/5/6），是历史兼容跳过的编号。

### 5.3 BroadcastChannel 频道机制

除主 WebSocket 外，还有独立的 [api/broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/broadcast.go) 通道系统，支持：
- WebSocket + SSE（Server-Sent Events）双协议
- 按频道名隔离订阅者（`sync.Map` 引用计数）
- `Destroy(force)`：无订阅者 + 非强制时延迟销毁

### 5.4 Electron 多窗口间的 IPC 桥

桌面端额外通过 Electron IPC 实现窗口间通信：
- **主窗口 → 子窗口**：`Constants.SIYUAN_SEND_WINDOWS` → [app/src/window/onWindowsMsg.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/window/onWindowsMsg.ts)
- **子窗口创建**：[app/src/window/openNewWindow.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/window/openNewWindow.ts) 将当前 Tab 的布局 JSON 通过 URL 参数 `?layout=` 传给新窗口
- 子窗口仅在 `sessionStorage` 中保存自己的局部布局（`saveLayout` → `window.sessionStorage.layout`），不调用后端 API

### 5.5 前端消息分发链路

所有 WebSocket 消息到达前端后经过两级分发：

1. **[app/src/util/processMessage.ts#L10-L77](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/util/processMessage.ts#L10-L77)**：处理通用消息（msg、进度条、reloadui）
2. **[app/src/index.ts#L73-L211](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/index.ts#L73-L211)** ws-main 回调：处理 30+ 种业务消息（setConf、reloaddoc、readonly、syncing、exit 等）

---

## 6. 默认值合并策略

### 6.1 防御性默认值填充

`InitConf()` 中对 **每个可能为 nil 的字段** 进行显式的 nil 检查和默认值填充，而非依赖 JSON 反序列化的零值。这是 SiYuan 配置系统最核心的设计特点。

### 6.2 指针区分「未设置」与「设置为零」

对于需要区分「用户未设置该字段」和「用户显式设置为 0」的场景，使用指针类型。参见 [model/conf.go#L257-L300](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L257-L300)：

```go
// 新增字段的默认值，使用指针类型来区分字段不存在（nil）和用户设置为 0（非 nil）
if nil == Conf.Editor.BacklinkSort {
    Conf.Editor.BacklinkSort = defaultEditor.BacklinkSort
}
if nil == Conf.Editor.FloatWindowDelay {
    v := 620
    Conf.Editor.FloatWindowDelay = &v
} else {
    *Conf.Editor.FloatWindowDelay = max(0, min(2000, *Conf.Editor.FloatWindowDelay))
}
```

### 6.3 边界范围校准

对数值型配置进行 **最小/最大边界校准**，防止异常配置导致崩溃：

| 配置项 | 范围 | 代码位置 |
|--------|------|---------|
| `Editor.FontSize` | [9, 72] | [model/conf.go#L274](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L274) |
| `FileTree.MaxOpenTabCount` | [8, 32] | [model/conf.go#L222-L227](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L222-L227) |
| `Sync.Interval` | [30s, 12h] | [model/sync.go#L394-L404](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/sync.go#L394-L404) |
| `Editor.HistoryRetentionDays` | [30, 3650] | [model/conf.go#L289-L294](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L289-L294) |
| `AI.OpenAI.APITemperature` | (0, 2] | [model/conf.go#L560-L562](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L560-L562) |
| `Flashcard.Weights` | 长度=19 且均为合法数字 | [model/conf.go#L514-L540](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L514-L540) |

### 6.4 废弃字段自动迁移

当配置语义变更时，自动执行值转换：
```go
// 导出 BlockRefMode: 0/1/5 → 4（脚注+锚点哈希）
if 0 == Conf.Export.BlockRefMode || 1 == ... || 5 == ... {
    Conf.Export.BlockRefMode = 4
}
// DocxTemplate: 旧字段注入到 PandocParams 后置空
if "" != Conf.Export.DocxTemplate {
    Conf.Export.PandocParams += " --reference-doc " + ...
    Conf.Export.DocxTemplate = ""
    Conf.Save()
}
```

### 6.5 语言缺失项兜底

多语言配置文件采用「目标语言 + en_US 合并」策略，参见 [server/serve.go#L468-L506](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/server/serve.go#L468-L506)：
- 加载目标语言 JSON
- 遍历 en_US.json 的所有 key
- 目标语言缺失的 key 自动用 en_US 值填充

---

## 7. 损坏配置的恢复机制

### 7.1 全局 conf.json 损坏

在 [model/conf.go#L127-L138](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L127-L138)：

```go
confPath := filepath.Join(util.ConfDir, "conf.json")
if gulu.File.IsExist(confPath) {
    data, err := os.ReadFile(confPath)     // 读文件失败：记录日志，使用空配置
    if err != nil { logging.LogErrorf(...) }
    else {
        err = gulu.JSON.UnmarshalJSON(data, Conf) // JSON 解析失败：记录日志
        if err != nil { logging.LogErrorf(...) }  // Conf 保持为 NewAppConf() 默认值
    }
}
```

**恢复逻辑**：读/解析失败 → 跳过 → 使用 `NewAppConf()` 初始化 → `InitConf()` 尾端填充所有默认值 → `Save()` 写回一份合法配置。**后果：用户自定义配置全部重置为默认值，无版本回溯能力**。

### 7.2 笔记本 conf.json 损坏

在 [model/box.go#L110-L132](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/box.go#L110-L132) `ListNotebooks()` 中：

```go
boxConfPath := filepath.Join(boxDirPath, ".siyuan", "conf.json")
if !isExistConf {
    logging.LogWarnf("found a corrupted box [%s]", boxDirPath) // 记录警告
} else {
    data, readErr := filelock.ReadFile(boxConfPath)
    if readErr != nil { continue }                              // 读失败：跳过该笔记本
    if parseErr := UnmarshalJSON(data, boxConf); parseErr != nil {
        logging.LogErrorf("parse box conf ... failed")
        filelock.Remove(boxConfPath)                             // 删除坏文件
        continue                                                  // 下次启动时将用默认 BoxConf
    }
}
```

**恢复策略**：读取失败 → 记录日志跳过；解析失败 → 删除损坏的 conf.json → 下次启动通过 `NewBoxConf()` 生成默认值。

**注意**：检测到 parse 失败时仅删除坏文件但未立即写回新 conf.json，意味着下次启动前每次 `ListNotebooks()` 都会触发一次删除日志。

### 7.3 用户指南笔记本自动清理

[model/conf.go#L1200-L1273](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L1200-L1273) `closeUserGuide()` 在退出时执行：
- 检测用户指南笔记本的 conf.json 是否损坏
- 损坏时自动删除整个笔记本目录
- 新用户启动时通过 `NewUserGuide()` 重新生成

### 7.4 FSRS 闪卡权重损坏

对 FSRS 算法权重的双重校验（[model/conf.go#L514-L540](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L514-L540)）：
1. 长度校验：必须恰好 19 个逗号分隔值
2. 数字校验：每个值必须是合法浮点数
3. 任一校验失败 → 重置为默认权重 + `BroadcastByType("main", "msg", ...)` 推送用户通知

### 7.5 DataIndexState 索引恢复

启动时检测 `DataIndexState == 1`（上次索引未完成）：
- 推送通知提示用户重新索引
- 重置状态为 0 → 触发全量重建索引

---

## 8. 跨端数据一致性与同步冲突策略

### 8.1 容器类型与路径适配

SiYuan 支持五种容器类型，定义于 [util/working.go#L395-L404](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L395-L404)：

| 容器 | 常量值 | 主要差异 |
|------|--------|---------|
| `std` | 桌面端（Windows/macOS/Linux） | 随机端口、完整功能 |
| `docker` | Docker 容器部署 | 固定端口、强制访问授权码、`SetSyncPerception` 强制设为 false |
| `android` / `ios` / `harmony` | 移动端 | 固定端口 6806、工作空间沙箱路径转换 |

### 8.2 iOS 沙箱路径适配

[util/working.go#L324-L361](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L324-L361) `ReadWorkspacePaths()` 对 iOS 进行特殊处理：

```go
if ContainerIOS == Container && strings.Contains(d, "/Documents/") {
    // iOS 端沙箱路径会变化，需要转换为相对路径再拼接当前沙箱中的工作空间基路径
    d = d[strings.Index(d, "/Documents/")+len("/Documents/"):]
    d = filepath.Join(workspaceBaseDir, d)
}
```

### 8.3 同步 Provider 与四种触发时机

`Sync` 配置支持四种 Provider（[conf/sync.go#L73-L78](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/conf/sync.go#L73-L78)）：
1. **`ProviderSiYuan = 0`**：SiYuan 官方云（需订阅 Pro）
2. **`ProviderS3 = 2`**：S3 兼容对象存储
3. **`ProviderWebDAV = 3`**：WebDAV 协议
4. **`ProviderLocal = 4`**：本地目录路径（局域网/NAS 场景）

四种同步触发时机由 `Sync.Mode` 控制（[conf/sync.go#L23](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/conf/sync.go#L23)）：
- `Mode = 0`：兼容旧配置，`InitConf` 中自动转换为 1
- **`Mode = 1` 自动**：启动、退出、`SyncInterval` 定时、感知 WS 通知、手动触发
- **`Mode = 2` 手动（启动+退出）**：仅启动和退出时同步
- **`Mode = 3` 完全手动**：仅用户点击「同步」按钮时同步

此外连续 8 次自动同步失败 → 推迟 64 分钟再同步（[model/sync.go#L265-L270](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/sync.go#L265-L270)）。

### 8.4 同步主流程（syncRepo）

三种同步入口都汇到 dejavu 库的 API：

| 入口 | 调用的 dejavu API | 适用场景 |
|------|------------------|---------|
| `SyncDataDownload()` | `repo.SyncDownload()` | 仅下载（菜单操作） |
| `SyncDataUpload()` | `repo.SyncUpload()` | 仅上传（菜单操作） |
| `BootSyncData()` | `bootSyncRepo()` → `repo.GetCloudLatest()` + `repo.GetSyncCloudFiles()` + 本地索引 | 启动同步（并行执行索引与拉取元数据） |
| **`SyncData()` / `syncData()`** | **`repo.Sync(syncContext)`** | **常规同步与退出前同步：先 Download 再 Upload 的统一流程** |

常规同步核心流程位于 [model/repository.go#L1510-L1606](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/repository.go#L1510-L1606) `syncRepo()`：

```
1. indexRepoBeforeCloudSync(repo)   // 本地构建快照索引（beforeIndex, afterIndex）
2. repo.Sync(syncContext)          // dejavu 库执行：云端索引→下载差异→合并→上传差异
   └→ 返回 mergeResult, trafficStat, err
3. processSyncMergeResult(...)     // 处理冲突/新增/删除文件，重建索引
4. checkIndex() + autoPurgeRepo()  // 异步：索引订正 + 仓库清理（非 exit 场景）
```

`dataChanged` 的判定条件为三个 OR：
```go
dataChanged = nil == beforeIndex || beforeIndex.ID != afterIndex.ID || mergeResult.DataChanged()
```
即只要本地索引变了或 dejavu 认为发生了数据变更，就会通过 WS 通知其他设备。

### 8.5 冲突处理策略（已核实代码证据）

冲突处理的核心在 [model/repository.go#L1632-L1681](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/repository.go#L1632-L1681) `processSyncMergeResult()`：

**dejavu 库的 MergeResult 包含三类文件集合：**
```go
mergeResult.Conflicts   // 两端都有修改的冲突文件列表
mergeResult.Upserts     // 需要插入/更新的文件列表（云端新增、本地新增）
mergeResult.Removes     // 需要删除的文件列表
```

**实际处理逻辑（两种模式）：**

1. **默认模式**（`Sync.GenerateConflictDoc = false`，[conf/sync.go#L40](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/conf/sync.go#L40)）：
   - `mergeResult.Conflicts` 中的文件 **直接接受 dejavu 库内部的合并结果**（具体策略由 dejavu 外部库决定，内核源码中无法直接查看）
   - 不生成任何冲突副本
   - 用户无法感知发生了冲突
   - ⚠️ **本地对同一文件的修改可能被静默覆盖**

2. **冲突副本模式**（`GenerateConflictDoc = true`）：
   - 遍历 `mergeResult.Conflicts` 中每个 `.sy` 文档
   - 从 `temp/repo/sync/conflicts/{timestamp}/` 读取冲突版本文件（dejavu 暂存的副本）
   - 调用 `resetTree(tree, "Conflicted", true)` 复制一份副本
   - 原文件仍使用 dejavu 合并后的结果
   - `needReloadFiletree = true`，前端刷新文档树以展示「Conflicted」后缀的新文档
   - 同时写入历史快照目录 `history/{timestamp}-sync/`

**结论**：之前关于「以文件最后修改时间为基础的覆盖式合并」的表述不准确。实际策略是：

> **冲突检测在 dejavu 外部库中完成，内核仅负责暴露可选的「冲突副本保留」机制。默认情况下（GenerateConflictDoc=false），冲突文件的最终版本完全由 dejavu 的内部算法决定（未知且不可配置）。**

### 8.6 感知同步（WebSocket 触发）

当 `Sync.Perception = true` 时（Docker 强制为 false），内核维护一个额外的 WebSocket 连接 `webSocketConn` 到 SiYuan 云端：
- 其他设备同步完成 → 云端推送 `{"cmd":"synced","synced":...}` → 本地触发 `SyncData(false)`
- 本地同步完成且数据有变更 → 向云端发送 `synced` 消息 → 云端广播到其他在线设备

### 8.7 固定端口代理服务

桌面端反代 6806 端口至内核随机端口，保证：
- 移动端固定连接 6806
- 浏览器书签无需更新
- TLS 证书绑定固定端口

实现于 [server/proxy/](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/server/proxy/) 包。

---

## 9. 风险评估

### 9.1 高风险项

| 风险 | 等级 | 说明 | 证据位置 |
|------|------|------|---------|
| **conf.json 损坏不可逆** | ⚠️ 中高 | 损坏后仅 `NewAppConf()` + 默认值覆盖，**无版本历史或备份文件机制**，用户所有全局自定义配置（主题、快捷键、AI 密钥等）完全丢失 | [model/conf.go#L127-L138](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L127-L138) |
| **同步冲突静默丢失** | ⚠️ 中高 | `GenerateConflictDoc=false`（默认）时，两端同时修改同一 `.sy` 文件 → dejavu 算法判定的冲突文件版本直接覆盖本地副本，**无用户通知、无历史对比** | [model/repository.go#L1642-L1681](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/repository.go#L1642-L1681) + [conf/sync.go#L40](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/conf/sync.go#L40) |
| **Save() 的 TOCTOU 窗口** | ⚠️ 中 | `Conf.m.Lock()` 仅保护内存，`filelock.ReadFile` → `bytes.Equal` → `filelock.WriteFile` 之间没有文件级锁，极端并发（多内核进程/外部脚本写 conf.json）下可能丢失写 | [model/conf.go#L870-L891](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L870-L891) |
| **工作空间锁崩溃残留** | ⚠️ 中 | 进程崩溃（蓝屏、`kill -9`、断电）后 `.lock` 文件残留，`flock.TryLock()` 直接拒绝启动，需用户手动删除 | [util/working.go:tryLockWorkspace](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go) |

### 9.2 中风险项

| 风险 | 说明 | 证据位置 |
|------|------|---------|
| **UILayout 超大 JSON** | 布局配置无大小限制，大量 Tab/Dock 打开时 conf.json 膨胀到数 MB，启动反序列化变慢 | 无显式大小限制，任意 SetUILayout 直接写入 |
| **笔记本 conf.json 删除后重建延迟** | 检测到 parse 失败时 `filelock.Remove()` 但未写回新值，每次 `ListNotebooks()` 都会重走删除路径 | [model/box.go#L110-L132](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/box.go#L110-L132) |
| **跨端路径大小写敏感** | Windows 不区分大小写，macOS 默认 APFS 可配置，Linux 区分。跨设备同步可能产生同名重复文件 | 无显式大小写归一化逻辑 |
| **FSRS 权重默认值静默重置** | 格式校验失败时直接替换为 19 个默认值，可能导致闪卡学习曲线突变 | [model/conf.go#L514-L540](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L514-L540) |
| **iOS 沙箱路径误判** | 硬编码 `strings.Contains(d, "/Documents/")`，若工作空间目录名恰好为 `Documents` 可能误切 | [util/working.go#L324-L361](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L324-L361) |
| **Docker 强制 bypass 环境变量** | `SIYUAN_ACCESS_AUTH_CODE_BYPASS=true` 可跳过访问授权码检查，部署时需警惕 | 路由层 `checkAuth` 中间件 |
| **BroadcastChannel 引用计数清理延迟** | `Destroy(false)` 无订阅者时不立即清理，高频创建/销毁场景可能有短时 goroutine 堆积 | [api/broadcast.go:Destroy](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/broadcast.go) |
| **SaveAssetsTexts 在 sql.CloseDatabase 之后** | `SaveAssetsTexts()`（L782）执行于 `sql.CloseDatabase()`（L781）**之后**，若 OCR 文本仍有 DB 写入则会失败 | [model/conf.go#L780-L783](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L780-L783) |

### 9.3 低风险项

| 风险 | 说明 |
|------|------|
| CookieKey 明文存储 | `~/.config/siyuan/cookie.key` 明文，但仅用于本地 Cookie 加密，无远程泄露风险 |
| DataIndexState 无超时 | 中断后不会自动触发重建，需等下次启动 |
| 语言文件合并时的 map 引用 | 无性能影响 |
| PushMode 值 3 缺失 | 历史遗留，不影响逻辑 |

---

## 10. 可进一步验证的问题点

以下问题在静态代码分析中无法完全确定，建议通过运行时实验或日志审计验证：

### 10.1 并发与一致性

1. **Q1：多窗口同时 setUILayout 是否存在覆盖？**
   - 两个窗口同时拖拽 Dock 并触发 `POST /api/system/setUILayout`
   - 后端 `Conf.SetUILayout()` → `Save()` 没有基于版本号的 CAS
   - **预期表现**：后写者胜，第一个窗口的布局修改丢失
   - **建议测试**：双窗口同时拖拽不同的 Dock 面板，10 次观察稳定性

2. **Q2：Save() 中 filelock.ReadFile 被阻塞的影响？**
   - `Conf.m.Lock()` 持有期间执行「读旧文件 → 序列化 → 写新文件」
   - 若磁盘极慢（网络盘/NAS），所有读配置的 goroutine（`Conf.m.RLock()`）被阻塞
   - **建议测试**：挂载慢速网络盘作为工作空间，观察 UI 响应性与超时

3. **Q3：同步完成后的 WS 广播风暴？**
   - `processSyncMergeResult` 中对每个 upsert/remove 的 `.sy` 都会触发对应广播（reloaddoc、reloadtree）
   - 大量文档同步（数百篇）时前端是否合并刷新还是逐条刷新
   - **建议测试**：启动时同步 500 篇文档，观察 WS 帧数与 UI 卡顿情况

### 10.2 同步冲突策略

4. **Q4：dejavu 内部的冲突决策具体是什么？**
   - 源码中仅能看到 `mergeResult.Conflicts` 作为输入
   - 决策是「最后修改时间戳优先」、「云端优先」、还是「内容哈希对比」？
   - **建议验证**：两台设备分别修改同一文档的不同段落，观察默认模式下的最终版本
   - 此问题需查阅 `siyuan-note/dejavu` 仓库的 `Sync()` / `SyncDownload()` 实现

5. **Q5：conf/ 目录是否纳入同步范围？**
   - `conf/conf.json` 中含 `System.ID`（设备唯一标识）、`System.WorkspaceDir`（绝对路径）
   - 若同步这些字段会覆盖另一台设备的本地值
   - **建议验证**：检查 dejavu 的 SyncIgnore 规则，确认 `conf/`、`repo/`、`temp/` 是否在忽略列表
   - 从代码中看有 `getSyncIgnoreLines()` → `repo.SetIgnore()` 的调用链但未看到具体内容

6. **Q6：GenerateConflictDoc 与 Upserts 的重复判定？**
   - 冲突文件既在 `Conflicts` 中，是否也会出现在 `Upserts` 中？
   - 若是，`incReindex(upserts, removes)` 中对同一文档是否被索引两次

### 10.3 退出顺序与数据完整性

7. **Q7：SaveAssetsTexts 位于 sql.CloseDatabase 之后是否安全？**
   - L781 `sql.CloseDatabase()` → L782 `util.SaveAssetsTexts()`
   - 若 `SaveAssetsTexts()` 内部有数据库写入（如更新资产内容表），此时连接已关闭
   - **建议验证**：搜索 `SaveAssetsTexts` 的实现，确认是否有数据库操作

8. **Q8：ExitSyncSucc 非零直接 return 是否跳过了 Conf.Save()？**
   - L750-L753：`if 0 != ExitSyncSucc { exitCode = 1; return }`
   - 直接 return 后 L780 的 `Conf.Close()` 被跳过
   - 这意味着 `Sync.Synced`、`Sync.Stat` 可能未写入 conf.json（虽然 syncData 内部已经 Save() 过）
   - **需验证**：syncData 内部是否已调用足够的 Conf.Save() 覆盖所有可能的修改

9. **Q9：退出时 Broadcast "exit" 后 UnlockWorkspace 的 500ms 够吗？**
   - L800 Broadcast → L801 UnlockWorkspace → L803 Sleep(500ms) → L818 WebSocketServer.Close()
   - 若前端处理 "exit" 需要 >500ms（如大布局序列化），WS 已关闭导致消息丢失
   - 多窗口场景下 500ms 是否足够所有窗口响应

### 10.4 恢复与迁移

10. **Q10：版本降级后新字段的静默裁剪？**
    - 新版本增加字段 A → 保存到 conf.json
    - 降级到旧版本（AppConf 结构体无字段 A）→ json.Unmarshal 默认忽略未知字段？还是报错？
    - Go 标准库 encoding/json 默认是**忽略未知字段**（不报错），但 gulu.JSON 是否为标准库包装？
    - 若后续再升级回来，字段 A 的值在中间降级过程中丢失了

11. **Q11：笔记本 conf.json 删除后何时重建？**
    - `ListNotebooks()` 检测到 parseErr → `filelock.Remove(boxConfPath)`
    - 删除后没有立即调用 `NewBoxConf()` + `filelock.WriteFile()` 写回
    - 后续流程中 `UpdateNotebookConf()` / `Close()` 是否会触发重写？
    - **建议验证**：删除某笔记本 conf.json → 重启 → 不修改该笔记本设置 → 再重启 → 观察是否每次都有 "parse box conf failed" 日志

### 10.5 性能与资源

12. **Q12：saveLayout 的递增重试是否有上限？**
    - `saveLayout()` 中 `for (Constants.TIMEOUT_LOAD * saveCount) > time...`
    - `saveCount` 若持续递增（例如 setUILayout API 持续超时）会导致等待指数级变长
    - 是否有最大重试次数与 reset 机制

13. **Q13：syncSameCount 的指数退避是否溢出？**
    - `delay := time.Minute * 2^syncSameCount`，当 syncSameCount > 63 时整型溢出
    - 虽有 `syncSameCount.Store(5)` 的上限截断，但只在 `> 10` 时触发（此时 2^10 = 1024 分钟 = ~17 小时，已远超 fixSyncInterval = 5 分钟的兜底）

---

## 11. 关键协作模块索引

### 后端（kernel/）

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| **全局配置管理** | [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go) | InitConf / Save / Close（15 步编排）/ GetMaskedConf / SetUILayout |
| **工作空间路径与锁** | [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go) | initWorkspaceDir、flock 加解锁、workspace.json 读写、iOS 沙箱转换 |
| **HTTP/WS 服务** | [server/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/server/serve.go) | 路由注册、port.json、CORS、`checkAuth` 中间件、语言文件合并 |
| **WS 广播分发** | [util/websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/websocket.go) | 6 种 PushMode 分发、Broadcast\* 系列工具函数、会话两级 sync.Map 管理 |
| **WS 消息结构** | [util/result.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/result.go) | Result 结构体、6 个 PushMode 常量、NewCmdResult |
| **广播通道（WS+SSE）** | [api/broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/broadcast.go) | BroadcastChannel 结构体、Subscribe/Destroy、SSE、引用计数 |
| **子设置 API** | [api/setting.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/setting.go) | 20+ 个字段 setter（ReadOnly / Keymap / Appearance 等），每 setter 三步曲 |
| **系统 API** | [api/system.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/system.go) | getConf / setUILayout / getEmojiConf / bootProgress |
| **工作空间 API** | [api/workspace.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/workspace.go) | switchWorkspace / getWorkspacesPath |
| **笔记本 API 与模型** | [api/notebook.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/notebook.go) + [model/box.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/box.go) | setNotebookConf / ListNotebooks（含损坏笔记本检测） |
| **同步主流程** | [model/sync.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/sync.go) | SyncData / BootSyncData / syncData / checkSync / SetSyncProvider\* |
| **同步冲突处理与仓库** | [model/repository.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/repository.go) | syncRepo / bootSyncRepo / processSyncMergeResult / newRepository / syncRepoDownload / syncRepoUpload |
| **配置结构体定义** | [conf/\*.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/conf/) | 20+ 子配置结构体 + New\*() 默认值构造函数（含 Sync.BoxConf.Layout 等） |
| **CMD 异步命令框架** | [cmd/cmd.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/cmd/cmd.go) | Cmd 接口、`Exec()` goroutine 异步执行框架 |
| **启动入口** | [main.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/main.go) | 8 步启动全流程编排（util.Boot → util.SetBooted） |

### 前端（app/src/）

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| **应用入口与 ws-main 分发** | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/index.ts) | fetchPost getConf、ws-main 消息 30+ 种业务 case 分发 |
| **配置获取后初始化** | [boot/onGetConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/boot/onGetConfig.ts) | JSONToLayout、外观应用、窗口事件初始化、WS 连接建立 |
| **布局持久化** | [layout/util.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/layout/util.ts) | saveLayout / exportLayout / resetLayout / layoutToJSON |
| **通用 WS 消息处理** | [util/processMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/util/processMessage.ts) | msg、reloadui、进度条、窗口滚动重置 |
| **Electron 新窗口** | [window/openNewWindow.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/window/openNewWindow.ts) | Tab→layout→URL 参数、Electron BrowserWindow 创建、IPC |
| **Electron 窗口间 IPC** | [window/onWindowsMsg.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/window/onWindowsMsg.ts) | 监听 SIYUAN_SEND_WINDOWS 消息、跨窗口 focus |
| **前端配置面板** | [config/\*.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/config/) | 各子设置面板的表单逻辑与 API 调用 |

---

## 结语

SiYuan 的配置与工作空间体系设计体现了 **「多层防御、渐进恢复、中心化广播」** 的工程思想：

- **防御性默认值合并** 对抗版本演进（nil 检查 + 指针区分语义 + 边界校准）
- **文件锁 + 原子写入 + 字节对比** 对抗崩溃损坏与无效 I/O
- **6 种 WS 广播模式 + Electron IPC + BroadcastChannel** 支撑多窗口实时协同
- **退出 15 步编排** 确保「文档 → 数据库 → 配置 → 同步状态」按正确依赖顺序落盘

其主要改进空间集中在三个方向：
1. **配置版本化备份**：conf.json 损坏前自动备份为 `conf.json.bak.{timestamp}`，提升恢复能力
2. **同步冲突策略透明化**：GenerateConflictDoc 默认开启，或至少在 UI 中高亮最近冲突
3. **跨端配置隔离**：`System.ID`、`System.WorkspaceDir`、`System.OS` 等平台强绑定字段不应参与 dejavu 同步；UILayout 按平台分别持久化（桌面端大布局 ≠ 移动端窄屏布局）
