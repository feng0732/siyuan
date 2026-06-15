# SiYuan 工作空间与配置持久化机制深度分析

> 版本：v3.6.5 | 分析日期：2026-06-15

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [工作空间初始化与启动加载流程](#2-工作空间初始化与启动加载流程)
3. [配置分层模型与存储结构](#3-配置分层模型与存储结构)
4. [配置变更、运行时缓存及落盘保存](#4-配置变更运行时缓存及落盘保存)
5. [多窗口状态管理机制](#5-多窗口状态管理机制)
6. [默认值合并策略](#6-默认值合并策略)
7. [损坏配置的恢复机制](#7-损坏配置的恢复机制)
8. [跨端数据一致性](#8-跨端数据一致性)
9. [风险评估](#9-风险评估)
10. [可进一步验证的问题点](#10-可进一步验证的问题点)
11. [关键协作模块索引](#11-关键协作模块索引)

---

## 1. 整体架构概览

SiYuan 采用 **前后端分离** 的单体架构：

- **前端 (app/)**：TypeScript + Electron（桌面端）/ 浏览器（Web端）/ 移动端（iOS/Android/Harmony），负责 UI 渲染、布局管理、用户交互
- **后端 (kernel/)**：Go 语言实现的内核进程，负责数据持久化、HTTP/WebSocket 服务、业务逻辑、同步引擎
- **通信层**：HTTP（API 请求）+ WebSocket（实时推送）+ Electron IPC（桌面端窗口间通信）

### 关键流程数据流

```
┌─────────────┐     HTTP API      ┌─────────────┐    读写文件     ┌──────────────┐
│  Frontend   │ ───────────────>  │   Kernel    │ ─────────────> │  File System │
│  (TS/Elec)  │ <───────────────  │   (Go)      │ <───────────── │ (Workspace)  │
└──────┬──────┘   WS Broadcast    └──────┬──────┘   filelock     └──────────────┘
       │                                  │
       │         Electron IPC             │
       └──────────────────────────────────┘
```

---

## 2. 工作空间初始化与启动加载流程

### 2.1 Kernel 启动全链路

启动入口在 [kernel/main.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/main.go#L30-L59)，执行顺序严格依赖：

| 步骤 | 函数 | 核心职责 | 关键代码位置 |
|------|------|----------|-------------|
| 1 | `util.Boot()` | 解析命令行参数、初始化工作空间路径、加锁 | [working.go:Boot](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L84-L172) |
| 2 | `model.InitConf()` | 加载 conf.json、合并默认值、语言初始化 | [conf.go:InitConf](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L123-L625) |
| 3 | `server.Serve()` | 启动 HTTP+WebSocket 服务、写入 port.json | [serve.go:Serve](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/server/serve.go#L133-L273) |
| 4 | `model.InitAppearance()` | 主题/图标初始化 | - |
| 5 | `sql.Init*Database()` | SQLite 数据库初始化（主库/历史/资源内容/块树） | - |
| 6 | `model.BootSyncData()` | 启动数据同步引擎 | - |
| 7 | `model.InitBoxes()` | 加载所有笔记本（Box）配置 | [conf.go:InitBoxes](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L1001-L1013) |
| 8 | `util.SetBooted()` | 标记启动完成，清理进度遮罩 | - |

### 2.2 工作空间路径决策

路径决策逻辑在 [working.go:initWorkspaceDir](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L246-L322)，优先级如下：

1. **命令行参数** `--workspace` > 环境变量 `SIYUAN_WORKSPACE_PATH`
2. **workspace.json** 最后一条记录（最近使用），位于 `~/.config/siyuan/workspace.json`
3. **平台默认路径**：
   - Windows: `%USERPROFILE%/SiYuan`
   - macOS: `~/Library/Application Support/SiYuan`
   - Linux/其他: `~/SiYuan`

### 2.3 工作空间独占锁机制

通过 `flock` 文件锁确保同一工作空间仅被一个内核实例占用：

```go
// working.go:tryLockWorkspace
WorkspaceLock = flock.New(filepath.Join(WorkspaceDir, ".lock"))
ok, err := WorkspaceLock.TryLock()   // 非阻塞尝试加锁
if !ok { os.Exit(ExitCodeWorkspaceLocked) }
```

退出时通过 [UnlockWorkspace](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L537-L551) 释放锁并删除 `.lock` 文件。

### 2.4 前端启动与配置获取

前端在 [app/src/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/index.ts#L217-L242) 中按以下链路初始化：

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
| **全局应用级** | `{workspace}/conf/` | `conf.json` | [conf.go:AppConf](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L52-L89) | Save() 调用 + 启动结束时 |
| **笔记本级** | `{workspace}/data/{boxID}/.siyuan/` | `conf.json` | [box.go:BoxConf](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/conf/box.go#L22-L34) | 笔记本设置变更 API |
| **用户家目录级** | `~/.config/siyuan/` | `workspace.json`、`port.json`、`cookie.key` | - | 工作空间切换、端口分配、首次启动 |

### 3.2 AppConf 核心字段

`AppConf` 包含 20+ 个子配置模块，主要模块：

| 字段 | 类型 | 说明 |
|------|------|------|
| `Appearance` | `*conf.Appearance` | 主题、图标、模式（明/暗）、状态栏 |
| `Editor` | `*conf.Editor` | 字体大小、Markdown 选项、历史保留 |
| `FileTree` | `*conf.FileTree` | 文档面板、最大打开标签数、排序 |
| `UILayout` | `*conf.UILayout` | 整个界面布局 JSON（map 结构） |
| `Sync` | `*conf.Sync` | 同步模式（云端/S3/WebDAV/本地）、间隔 |
| `System` | `*conf.System` | 设备 ID、内核版本、网络代理、工作空间路径 |
| `Keymap` | `*conf.Keymap` | 快捷键映射（嵌套 map） |
| `AI` / `Bazaar` / `Repo` | - | AI 配置、集市、数据仓库 |
| `m` / `userLock` | `*sync.RWMutex` | 并发读写锁（**不序列化**） |

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
├── repo/                  # 数据仓库索引
└── temp/                  # 临时目录（每次启动重建）
    ├── siyuan.db          # 主 SQLite 数据库
    ├── history.db         # 历史数据库
    ├── blocktree.db       # 块树数据库
    └── asset_content.db   # 资源内容数据库
```

---

## 4. 配置变更、运行时缓存及落盘保存

### 4.1 配置变更的标准流程

所有配置变更遵循 **「内存修改 → 条件落盘 → 广播通知」** 三步曲。以设置编辑器只读状态为例，参见 [setting.go:setEditorReadOnly](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/setting.go#L33-L52)：

```
1. 修改内存：   model.Conf.Editor.ReadOnly = readOnly
2. 落盘保存：   model.Conf.Save()
3. 广播通知：   util.BroadcastByType("protyle", "readonly", ...)
                util.BroadcastByType("main", "readonly", ...)
```

### 4.2 Save() 保存机制的原子性

[conf.go:Save](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L870-L891) 实现了 **数据对比 + 文件锁写入**：

```go
func (conf *AppConf) Save() {
    Conf.m.Lock()                     // 1. 写锁保护
    defer Conf.m.Unlock()

    newData := MarshalIndentJSON(Conf) // 2. 序列化新数据
    oldData := filelock.ReadFile(...)  // 3. 读取旧数据
    if bytes.Equal(newData, oldData) { // 4. 内容未变更则跳过写入
        return
    }
    conf.save0(newData)                // 5. filelock.WriteFile 原子写入
}
```

**关键优化**：保存前进行字节级对比，无差异时跳过磁盘 I/O。

### 4.3 filelock 原子写入

使用 `github.com/siyuan-note/filelock` 包（在多个场景中出现），确保写入过程：
1. 先写入 `.tmp` 临时文件
2. 执行 `rename` 原子替换
3. 避免并发写入导致的半写损坏

### 4.4 UILayout 布局保存的特殊路径

**主窗口布局** 保存路径与普通配置不同：
- 前端主动触发：[layout/util.ts:saveLayout](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/layout/util.ts#L128-L168)
- 调用 API：`POST /api/system/setUILayout`
- 后端处理：[system.go:setUILayout](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/system.go#L570-L599) → `model.Conf.SetUILayout()` → `Save()`

**子窗口布局** 保存在 `sessionStorage`（仅当前窗口生命周期），不落盘。

### 4.5 退出时的保存屏障

[conf.go:Close](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L738-L832) 在进程退出时执行：
1. `FlushTxQueue()` — 冲刷事务队列
2. `syncData()` — 执行同步（非强制退出）
3. `Conf.Close()` → `Conf.Save()` — 最终保存配置
4. `sql.FlushQueue()` — 数据库写入
5. `UnlockWorkspace()` — 释放锁

---

## 5. 多窗口状态管理机制

### 5.1 WebSocket 会话分组

内核通过两级 `sync.Map` 管理所有前端连接，定义于 [websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/websocket.go#L30-L36)：

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

### 5.2 六种广播模式

[websocket.go:PushEvent](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/websocket.go#L383-L400) 支持六种推送范围：

| 模式 | 语义 | 场景 |
|------|------|------|
| `Broadcast` | 全量广播 | 文档保存、数据变更 |
| `SingleSelf` | 仅回传给请求者 | API 响应 |
| `BroadcastExcludeSelf` | 除自己外全部 | 本地光标位置同步 |
| `BroadcastExcludeSelfApp` | 跨窗口广播 | 多窗口数据同步 |
| `BroadcastApp` | 同一 App 内全部 | 单窗口内多面板同步 |
| `BroadcastMainExcludeSelfApp` | 除自己外所有 main 频道 | 布局变更通知 |

### 5.3 BroadcastChannel 频道机制

除主 WebSocket 外，还有独立的 [broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/broadcast.go) 通道系统，支持：
- WebSocket + SSE（Server-Sent Events）双协议
- 按频道名隔离订阅者
- 引用计数自动销毁无订阅者通道

### 5.4 Electron 多窗口间的 IPC 桥

桌面端额外通过 Electron IPC 实现窗口间通信：
- **主窗口 → 子窗口**：`Constants.SIYUAN_SEND_WINDOWS` → [onWindowsMsg.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/window/onWindowsMsg.ts)
- **子窗口创建**：[openNewWindow.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/window/openNewWindow.ts) 将当前 Tab 的布局 JSON 通过 URL 参数传给新窗口
- 子窗口仅在 `sessionStorage` 中保存自己的局部布局

### 5.5 前端消息分发链路

所有 WebSocket 消息到达前端后经过两级分发：

1. **[processMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/util/processMessage.ts#L10-L77)**：处理通用消息（msg、进度、reloadui）
2. **[index.ts:ws-main 回调](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/index.ts#L73-L211)**：处理 30+ 种业务消息（setConf、reloaddoc、readonly、syncing 等）

---

## 6. 默认值合并策略

### 6.1 防御性默认值填充

`InitConf()` 中对 **每个可能为 nil 的字段** 进行显式的 nil 检查和默认值填充，而非依赖 JSON 反序列化的零值。这是 SiYuan 配置系统最核心的设计特点。

### 6.2 指针区分「未设置」与「设置为零」

对于需要区分「用户未设置该字段」和「用户显式设置为 0」的场景，使用指针类型。参见 [conf.go:InitConf](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L257-L300)：

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
| `Editor.FontSize` | [9, 72] | conf.go:274 |
| `FileTree.MaxOpenTabCount` | [8, 32] | conf.go:222-227 |
| `Sync.Interval` | [30s, 12h] | conf.go:413-418 |
| `Editor.HistoryRetentionDays` | [30, 3650] | conf.go:289-294 |
| `AI.OpenAI.APITemperature` | (0, 2] | conf.go:560-562 |
| `Flashcard.Weights` | 长度=19 且均为合法数字 | conf.go:514-540 |

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

多语言配置文件采用「目标语言 + en_US 合并」策略，参见 [serve.go:serveAppearance](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/server/serve.go#L468-L506)：
- 加载目标语言 JSON
- 遍历 en_US.json 的所有 key
- 目标语言缺失的 key 自动用 en_US 值填充

---

## 7. 损坏配置的恢复机制

### 7.1 全局 conf.json 损坏

在 [conf.go:InitConf](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L127-L138)：

```go
confPath := filepath.Join(util.ConfDir, "conf.json")
if gulu.File.IsExist(confPath) {
    data, err := os.ReadFile(confPath)      // 读文件失败：记录日志，使用空配置
    if err != nil { logging.LogErrorf(...) }
    else {
        err = gulu.JSON.UnmarshalJSON(data, Conf) // JSON 解析失败：记录日志
        if err != nil { logging.LogErrorf(...) }  // Conf 保持为 NewAppConf() 默认值
    }
}
```

**恢复逻辑**：读/解析失败 → 跳过 → 使用 `NewAppConf()` 初始化 → `InitConf()` 尾端填充所有默认值 → `Save()` 写回一份合法配置。**用户数据不会丢失，但自定义配置会被重置为默认值**。

### 7.2 笔记本 conf.json 损坏

在 [box.go:ListNotebooks](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/box.go#L110-L132)：

```go
boxConfPath := filepath.Join(boxDirPath, ".siyuan", "conf.json")
if !isExistConf {
    logging.LogWarnf("found a corrupted box [%s]", boxDirPath) // 记录警告
} else {
    data, readErr := filelock.ReadFile(boxConfPath)
    if readErr != nil { continue }                             // 读失败：跳过该笔记本
    if parseErr := UnmarshalJSON(data, boxConf); parseErr != nil {
        logging.LogErrorf("parse box conf ... failed")
        filelock.Remove(boxConfPath)                            // 删除坏文件
        continue                                                 // 下次启动时将用默认 BoxConf
    }
}
```

**恢复策略**：读取失败 → 记录日志跳过；解析失败 → 删除损坏的 conf.json → 下次启动使用默认 `NewBoxConf()`。

### 7.3 用户指南笔记本自动清理

[closeUserGuide](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L1200-L1273) 在退出时执行：
- 检测用户指南笔记本的 conf.json 是否损坏
- 损坏时自动删除整个笔记本目录
- 为新用户启动时重新生成

### 7.4 FSRS 闪卡权重损坏

对 FSRS 算法权重的双重校验（[conf.go:514-540](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go#L514-L540)）：
1. 长度校验：必须恰好 19 个逗号分隔值
2. 数字校验：每个值必须是合法浮点数
3. 任一校验失败 → 重置为默认权重 + 推送用户通知

### 7.5 DataIndexState 索引恢复

启动时检测 `DataIndexState == 1`（上次索引未完成）：
- 推送通知提示用户重新索引
- 重置状态为 0 → 后续触发全量重建

---

## 8. 跨端数据一致性

### 8.1 容器类型与路径适配

SiYuan 支持五种容器类型，定义于 [working.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L395-L404)：

| 容器 | 常量值 | 主要差异 |
|------|--------|---------|
| `std` | 桌面端（Windows/macOS/Linux） | 随机端口、完整功能 |
| `docker` | Docker 容器部署 | 固定端口、强制访问授权码、禁用感知同步 |
| `android` / `ios` / `harmony` | 移动端 | 固定端口 6806、工作空间沙箱路径转换 |

### 8.2 iOS 沙箱路径适配

[ReadWorkspacePaths](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go#L324-L361) 对 iOS 进行特殊处理：

```go
if ContainerIOS == Container && strings.Contains(d, "/Documents/") {
    // iOS 端沙箱路径会变化，需要转换为相对路径再拼接当前沙箱中的工作空间基路径
    d = d[strings.Index(d, "/Documents/")+len("/Documents/"):]
    d = filepath.Join(workspaceBaseDir, d)
}
```

### 8.3 同步引擎的多端一致性

`Sync` 配置支持四种 Provider，每次退出时与启动后均触发 `syncData()`：
1. SiYuan 官方云（需订阅 Pro）
2. S3 兼容对象存储
3. WebDAV 协议
4. 本地目录路径（局域网/NAS 场景）

同步冲突解决策略：以文件最后修改时间为基础的覆盖式合并，配合快照索引（`Repo`）回溯。

### 8.4 固定端口代理服务

桌面端反代 6806 端口至内核随机端口，保证：
- 移动端固定连接 6806
- 浏览器书签无需更新
- TLS 证书绑定固定端口

实现于 [proxy](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/server/proxy/) 包。

---

## 9. 风险评估

### 9.1 高风险项

| 风险 | 等级 | 说明 | 影响范围 |
|------|------|------|----------|
| **conf.json 损坏无备份恢复** | ⚠️ 中高 | 损坏后仅使用默认值，无版本历史或备份文件机制 | 用户所有全局自定义配置丢失 |
| **Save() 竞态窗口** | ⚠️ 中 | `bytes.Equal` 与 `save0` 之间存在 TOCTOU 窗口，极端并发下可能丢失写 | 极少概率的配置丢失 |
| **工作空间锁崩溃残留** | ⚠️ 中 | 进程崩溃后 `.lock` 文件可能残留，需用户手动删除 | 下次启动失败 |
| **UILayout 超大 JSON** | ⚠️ 中 | 布局配置无大小限制，大量 Tab/Dock 可能导致 conf.json 膨胀 | 启动变慢、序列化内存压力 |
| **跨端路径大小写敏感** | ⚠️ 中 | Windows 不区分大小写，macOS 默认不区分，Linux 区分 | 跨设备同步可能出现重复文件 |

### 9.2 中风险项

| 风险 | 说明 |
|------|------|
| FSRS 权重默认值静默重置 | 用户未感知时权重被还原，闪卡学习曲线异常 |
| BroadcastChannel 泄漏 | 短时间大量创建/销毁频道可能存在 goroutine 泄漏 |
| 移动端沙箱路径迁移 | iOS 升级导致沙箱路径变动，工作空间关联丢失（已部分修复） |
| Docker 强制 auth code 环境变量绕过 | `SIYUAN_ACCESS_AUTH_CODE_BYPASS=true` 可跳过安全检查 |

### 9.3 低风险项

| 风险 | 说明 |
|------|------|
| CookieKey 明文存储 | `~/.config/siyuan/cookie.key` 明文，但仅用于本地 Cookie 加密 |
| DataIndexState 无超时 | 中断后不会自动触发重建，需等下次启动 |
| 语言文件合并时的 map 引用 | 无性能影响 |

---

## 10. 可进一步验证的问题点

以下问题在静态代码分析中无法完全确定，建议通过运行时实验或日志审计验证：

### 10.1 并发与一致性

1. **Q1：多窗口同时保存布局时是否存在覆盖？**
   - 两个窗口同时调用 `/api/system/setUILayout`
   - 后端是否存在「最后写入者胜」导致一个窗口布局丢失？
   - **建议测试**：双窗口同时拖拽 Dock，观察是否稳定

2. **Q2：Save() 数据对比的线程安全？**
   - `Conf.m.Lock()` 持有期间执行序列化 + 读旧文件 + 写新文件
   - 若 `filelock.ReadFile` 被外部进程阻塞，是否会导致全局配置读被锁？
   - **建议测试**：模拟慢速磁盘，观察 UI 响应性

3. **Q3：WebSocket 广播风暴？**
   - 高频操作（如持续拖拽文档）是否会产生大量 broadcast？
   - `BroadcastByType` 内部是否存在节流机制？
   - **建议测试**：观察大量编辑时的 WS 帧频率

### 10.2 数据恢复与迁移

4. **Q4：conf.json 损坏后用户配置是否真的不可逆？**
   - 是否存在 `conf.json.bak` 或 `conf.json.tmp` 残留？
   - 历史版本目录中是否保存过配置快照？
   - **建议验证**：故意损坏 conf.json，观察恢复后数据丢失程度

5. **Q5：版本升级时的字段裁剪？**
   - 新版本 AppConf 新增字段时正常（nil → 默认值）
   - 新版本删除旧字段时，JSON 反序列化是否会保留未知字段？
   - 老用户降级回旧版本后，新字段值会被静默丢弃？
   - **建议验证**：在新版设置 → 降级 → 再升级，观察配置完整性

6. **Q6：笔记本 conf.json 删除后重建逻辑？**
   - `ListNotebooks` 检测到 parse 失败时 `filelock.Remove(boxConfPath)`
   - 但没有立即调用 save 写入新的默认 conf.json
   - 是否会导致每次启动都看到 "corrupted box" 警告？
   - **建议验证**：删除某笔记本 conf.json 后多次重启观察日志

### 10.3 跨端与同步

7. **Q7：iOS 沙箱路径转换的健壮性？**
   - 当前逻辑硬编码判断 `/Documents/` 子串
   - 用户自定义路径包含该字符串时是否会误触发？
   - 如工作空间恰好名为 `Documents`？

8. **Q8：跨端配置值不兼容？**
   - `System.OS` / `System.OSPlatform` 与平台强耦合
   - 同步时若将一台设备的 System 配置同步到另一台，是否覆盖平台特有值？
   - **建议验证**：检查 Sync 模块是否同步 conf.json 或仅同步 data/

9. **Q9：Docker 部署下感知同步被禁用的影响？**
   - `if ContainerDocker == util.Container { Conf.Sync.Perception = false }`
   - 文件感知同步禁用后是否仅依赖定时轮询？
   - 是否存在文件变更延迟的窗口？

### 10.4 性能与资源

10. **Q11：布局 JSON 序列化的性能拐点？**
    - 50+ 个打开 Tab 时 `layoutToJSON` 的耗时？
    - 是否存在防抖/节流？`saveLayout` 中有 `Constants.TIMEOUT_LOAD * saveCount` 的递增重试，但无节流
    - **建议验证**：打开大量文档后测量切换标签时 saveLayout 调用频率

11. **Q12：BroadcastChannel 的资源清理？**
    - 订阅者为 0 时 `Destroy(false)` 不会立即清理，需等下次触发
    - 是否存在长时间存在空 channel 的内存泄漏？

---

## 11. 关键协作模块索引

### 后端（kernel/）

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| 全局配置管理 | [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/conf.go) | InitConf/Save/Close/GetMaskedConf |
| 工作空间路径 | [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/working.go) | initWorkspaceDir、加解锁、workspace.json 读写 |
| HTTP/WS 服务 | [server/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/server/serve.go) | 路由注册、port.json、CORS、鉴权中间件 |
| WebSocket 广播 | [util/websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/websocket.go) | 6 种 Push 模式、会话管理 |
| 广播通道 | [api/broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/broadcast.go) | BroadcastChannel、SSE、订阅管理 |
| 设置 API | [api/setting.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/setting.go) | 各子模块配置 setter |
| 系统 API | [api/system.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/system.go) | getConf/setUILayout/getEmojiConf |
| 工作空间 API | [api/workspace.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/workspace.go) | 工作空间增删、切换 |
| 笔记本 API | [api/notebook.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/api/notebook.go) | setNotebookConf/lsNotebooks |
| 笔记本模型 | [model/box.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/model/box.go) | ListNotebooks、损坏笔记本检测 |
| 配置结构定义 | [conf/*.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/conf/) | 20+ 子配置结构体、默认值构造函数 |
| 启动入口 | [main.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/main.go) | 启动全流程编排 |
| WS 消息结构 | [util/result.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/util/result.go) | Result 结构体、Cmd/Push 模式 |
| CMD 命令框架 | [cmd/cmd.go](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/kernel/cmd/cmd.go) | Cmd 接口、异步 Exec goroutine |

### 前端（app/src/）

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| 应用入口 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/index.ts) | getConf、ws-main 消息分发 |
| 配置获取与应用 | [boot/onGetConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/boot/onGetConfig.ts) | JSONToLayout、外观应用、窗口事件初始化 |
| 布局持久化 | [layout/util.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/layout/util.ts) | saveLayout/exportLayout/resetLayout |
| 通用消息处理 | [util/processMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/util/processMessage.ts) | msg、reloadui、进度条 |
| 新窗口创建 | [window/openNewWindow.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/window/openNewWindow.ts) | Tab→新窗口、IPC 通信 |
| 窗口间消息 | [window/onWindowsMsg.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/window/onWindowsMsg.ts) | Electron IPC 消息处理 |
| 配置项模块 | [config/*.ts](file:///d:/fz/0601/solo-dogfeeding/code/299-siyuan/app/src/config/) | 各子配置面板的前端逻辑 |

---

## 结语

SiYuan 的配置与工作空间体系设计体现了 **「多层防御、渐进恢复、中心化广播」** 的工程思想：通过细粒度默认值合并对抗版本演进，通过文件锁+原子写入对抗崩溃损坏，通过 WebSocket 的频道化广播支撑多窗口实时协同。其主要改进空间集中在 **配置的版本化备份**、**布局的增量序列化** 以及 **跨端配置值的平台隔离** 三个方向。
