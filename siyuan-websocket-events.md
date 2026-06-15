# SiYuan WebSocket 实时事件通道完整流程分析

## 1. 架构总览

SiYuan 的 WebSocket 实时事件系统采用 **Hub-Spoke（集线器-分支）架构**，通过后端统一的 `WebSocketServer` 管理所有连接，前端按功能模块建立多个独立的 WebSocket 连接，通过 `type` 参数区分不同业务通道。系统同时支持 **SSE（Server-Sent Events）** 作为备选通信协议。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端（浏览器/Electron）                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │  main    │ │ filetree │ │ protyle* │ │  tag     │ │ outline  │  ...     │
│  │  WS连接  │ │  WS连接  │ │  WS连接  │ │  WS连接  │ │  WS连接  │          │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘          │
└───────┼────────────┼────────────┼────────────┼────────────┼────────────────┘
        │            │            │            │            │
        ▼            ▼            ▼            ▼            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           后端（Go Gin + Melody）                             │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │                    WebSocketServer (melody.Melody)                     │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │   │
│  │  │  sessions: sync.Map{appId -> sync.Map{sessionId -> Session}}    │  │   │
│  │  │  authSessions: sync.Map (独立的鉴权会话)                         │  │   │
│  │  └─────────────────────────────────────────────────────────────────┘  │   │
│  │                                                                       │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐    │   │
│  │  │ PushEvent  │  │ Broadcast  │  │  ByType    │  │ ByApp/Single │    │   │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └──────┬───────┘    │   │
│  └────────┼───────────────┼───────────────┼────────────────┼────────────┘   │
│           │               │               │                │                │
│  ┌────────▼───────────────▼───────────────▼────────────────▼────────────┐   │
│  │                         内部事件源 (model/*)                           │   │
│  │  事务处理 / 文件操作 / 同步状态 / 通知消息 / 主题变更 ...                │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────┐  ┌────────────────────────────────┐    │
│  │     BroadcastChannels (独立广播)  │  │     UnifiedSSE (SSE 服务)     │    │
│  └──────────────────────────────────┘  └────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心数据结构

### 2.1 后端消息结构 Result
[result.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/result.go#L35-L46)

```go
type Result struct {
    Cmd       string         `json:"cmd"`       // 命令名，前端根据此字段分发
    ReqId     float64        `json:"reqId"`     // 请求ID，对应前端的请求
    AppId     string         `json:"app"`       // 应用实例ID
    SessionId string         `json:"sid"`       // 会话ID
    PushMode  PushMode       `json:"pushMode"`  // 推送模式
    Callback  any            `json:"callback"`  // 回调标识
    Code      int            `json:"code"`      // 状态码: 0=成功, <0=错误/提示
    Msg       string         `json:"msg"`       // 消息文本
    Data      any            `json:"data"`      // 业务数据
    Context   map[string]any `json:"context,omitempty"` // 上下文，如 rootIDs
}
```

### 2.2 前端接收数据结构 IWebSocketData
[index.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/types/index.d.ts#L742-L750)

```typescript
interface IWebSocketData {
    cmd?: string;
    callback?: string;
    data?: any;
    msg: string;
    code: number;
    sid?: string;
    context?: any;
}
```

### 2.3 推送模式 PushMode
[result.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/result.go#L24-L33)

| 模式值 | 常量名 | 说明 |
|--------|--------|------|
| 0 | `PushModeBroadcast` | **所有应用所有会话广播**（最常用） |
| 1 | `PushModeSingleSelf` | **自我应用会话单播**（只发给自己的连接） |
| 2 | `PushModeBroadcastExcludeSelf` | **非自我会话广播**（发给其他连接） |
| 4 | `PushModeBroadcastExcludeSelfApp` | **非自我应用所有会话广播**（发给其他应用实例） |
| 5 | `PushModeBroadcastApp` | **单个应用内所有会话广播** |
| 6 | `PushModeBroadcastMainExcludeSelfApp` | **非自我应用主会话广播** |

### 2.4 连接类型 TWS
[index.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/types/index.d.ts#L3)

```typescript
type TWS = "main" | "filetree" | "protyle" | "backlink" | "bookmark" | "graph" | "outline" | "tag"
```

#### 八种连接类型定位差异

| 类型 | 定位 | 实例数量 | 订阅范围 | 核心命令集 |
|------|------|----------|----------|------------|
| **main** | 全局主控通道 | 1/应用 | 全局事件、系统状态、进度通知 | 20+ 种：logoutAuth/setAppearance/reloadPlugin/progress/syncing/backgroundtask/rename/closeBox/... |
| **filetree** | 文件树导航 | 1/应用（Dock） | 笔记本和文档树的增删改查 | reloadFiletree/moveDoc/mount/createnotebook/closeBox/removeBox/create/rename/... |
| **protyle** | 编辑器通道 | N/应用（每个打开的文档） | 特定文档的编辑事务、重载指令 | transactions/reload/addLoading/unfoldHeading/readonly/heading2doc/rename/moveDoc/... |
| **backlink** | 反链面板 | N/应用 | 反链关联文档的生命周期事件 | rename/closeBox/removeBox/removeDoc（仅判断自身存在性，无反链内容实时更新） |
| **outline** | 大纲面板 | N/应用 | 大纲标题变更、文档保存 | savedoc(触发 onTransaction 更新大纲)/rename/closeBox/removeDoc |
| **bookmark** | 书签面板 | 1/应用（Dock） | 书签属性变更、文档存在性 | transactions(检测 bookmark class)/closeBox/removeBox/removeDoc/mount |
| **tag** | 标签面板 | 1/应用（Dock） | 标签属性变更、文档存在性 | transactions(检测 tag data-type)/closeBox/removeBox/removeDoc/mount |
| **graph** | 图关系视图 | N/应用 | 图关联的文档/笔记本变更 | mount/rename/closeBox/removeBox/removeDoc |

**订阅模式分类**：
- **全局级**（1个实例）：main、filetree、bookmark、tag —— 全量接收该 type 的所有广播
- **文档级**（N个实例，每文档一个）：protyle、outline、backlink、graph —— 通过 `blockId/rootId` 在前端过滤自身相关消息

> **重要边界**：后端 `BroadcastByType` 是按 `type` 全量广播，**不做文档级过滤**。文档级过滤全部在前端 `msgCallback` 中通过 `if (this.blockId === data.data.id)` 等判断完成。这意味着打开 10 个文档时，每个 protyle 连接都会收到所有文档的 transactions 推送，大部分被前端丢弃。

---

## 3. 后端实现分析

### 3.1 服务器启动与连接入口

[serve.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/server/serve.go#L697-L851) 中的 `serveWebSocket` 函数是核心入口：

#### 3.1.1 Melody 初始化
```go
util.WebSocketServer = melody.New()
util.WebSocketServer.Config.MaxMessageSize = 1024 * 1024 * 8 // 8MB 消息上限
```

#### 3.1.2 连接鉴权流程
`HandleConnect` 回调中执行多层鉴权检查：

1. **Cookie 鉴权**：检查 Cookie 中的 `siyuan` session，验证 `AccessAuthCode`
2. **JWT Token 鉴权**：解析 `X-Auth-Token` 请求头，验证角色权限（Administrator/Editor/Reader）
3. **Auth 页面特殊通道**：`/ws?app=siyuan&id=auth&type=auth` 用于授权页面保活

鉴权失败时通过 `s.CloseWithMsg([]byte("  unauthenticated"))` 关闭连接。

#### 3.1.3 发布服务标记
若 Token 为发布服务 Token，设置 `s.Set("isPublish", true)`，用于后续 `ClosePublishServiceSessions` 时的识别。

### 3.2 连接注册与会话管理

[websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/websocket.go#L111-L189)

#### 3.2.1 AddPushChan —— 连接注册
连接建立成功后，从 URL Query 参数提取关键标识并存入 session metadata：

| 参数 | 说明 | 存储位置 |
|------|------|----------|
| `app` | 应用实例 ID（前端随机生成） | `session.Set("app", appID)` |
| `id` | 会话 ID（每个 WebSocket 连接唯一） | `session.Set("id", id)` |
| `type` | 连接类型（main/filetree/protyle/...） | `session.Set("type", typ)` |

数据结构（双重 Map）：
```
sessions (sync.Map)
└── appId (string)
    └── sync.Map
        └── sessionId (string) -> *melody.Session
```

鉴权会话独立存储在 `authSessions` 中。

#### 3.2.2 RemovePushChan —— 连接注销
从 `sessions` / `authSessions` 中递归删除：
1. 先根据 appId 找到对应的子 Map
2. 从中删除 sessionId
3. 若子 Map 为空，同步删除 appId 节点（资源释放）

#### 3.2.3 ClosePushChan —— 主动关闭
前端发送 `closews` 命令时，遍历所有 app 的所有 session，找到匹配 id 的连接并关闭。

### 3.3 事件广播机制

#### 3.3.1 按类型广播 BroadcastByType
[websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/websocket.go#L81-L92)

最常用的推送方式，遍历所有 app 下的所有 session，筛选 `type` 匹配的连接发送消息。

```go
func BroadcastByType(typ, cmd string, code int, msg string, data any)
```

典型调用：
- `BroadcastByType("main", "reloadui", 0, "", nil)` —— 刷新 UI
- `BroadcastByType("protyle", "reload", 0, "", rootID)` —— 编辑器重载
- `BroadcastByType("filetree", "reloadFiletree", 0, "", nil)` —— 文件树刷新

#### 3.3.2 统一事件入口 PushEvent
[websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/websocket.go#L383-L400)

根据 `PushMode` 字段路由到具体推送策略：

```go
func PushEvent(event *Result) {
    switch event.PushMode {
    case PushModeBroadcast:                   Broadcast(msg)
    case PushModeSingleSelf:                  single(msg, event.AppId, event.SessionId)
    case PushModeBroadcastExcludeSelf:        broadcastOthers(msg, event.SessionId)
    case PushModeBroadcastExcludeSelfApp:     broadcastOtherApps(msg, event.AppId)
    case PushModeBroadcastApp:                broadcastApp(msg, event.AppId)
    case PushModeBroadcastMainExcludeSelfApp: broadcastOtherAppMains(msg, event.AppId)
    }
}
```

#### 3.3.3 事务推送 pushTransactions
[transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/api/transaction.go#L94-L124)

事务处理完成后的推送逻辑最为关键：

1. **推送模式选择**：
   - 属性视图（AttrView）相关事务使用 `PushModeBroadcast`（全量广播，需要所有连接同步）
   - 其他事务默认 `PushModeBroadcastExcludeSelf`（排除发起者自身，避免重复渲染）

2. **上下文补充**：
   - 提取所有变更事务中的 `rootIDs`（变更的文档根 ID 列表），放入 `evt.Context`
   - 前端据此判断是否需要刷新当前文档

3. **同步等待**：
   - `model.FlushTxQueue()` 确保文件写入完成
   - `tx.WaitForCommit()` 确保所有事务提交完毕后才推送

### 3.4 常用推送函数速览
[websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/websocket.go#L214-L381)

| 函数 | type | cmd | 用途 |
|------|------|-----|------|
| `ReloadUI()` | main | reloadui | 刷新整个 UI |
| `PushMsg(msg, timeout)` | main | msg | 顶部通知消息 |
| `PushErrMsg(msg, timeout)` | main | msg | 错误消息 (code=-1) |
| `PushStatusBar(msg)` | main | statusbar | 状态栏消息 |
| `PushProgress(code, current, total, msg)` | main | progress | 进度条更新 |
| `PushTxErr(msg, code, data)` | main | txerr | 事务错误 |
| `PushReloadDoc(rootID)` | main | reloaddoc | 重载指定文档 |
| `PushSaveDoc(rootID, typ, sources)` | - | savedoc | 保存文档事件（全量广播） |
| `PushReloadProtyle(rootID)` | protyle | reload | 重载编辑器实例 |
| `PushReloadFiletree()` | filetree | reloadFiletree | 刷新文件树 |
| `PushReloadTag()` | main | reloadTag | 刷新标签 |
| `PushBackgroundTask(data)` | main | backgroundtask | 后台任务 |
| `PushDownloadProgress(id, percent)` | - | downloadProgress | 下载进度（全量广播） |
| `PushProtyleLoading(rootID, msg)` | protyle | addLoading | 编辑器加载提示 |
| `PushSetRefDynamicText(...)` | main | setRefDynamicText | 引用文本动态更新 |

### 3.5 客户端命令处理

[serve.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/server/serve.go#L789-L850) 中的 `HandleMessage`：

1. **消息解析**：`gulu.JSON.UnmarshalJSON(msg, &request)`
2. **Auth 会话过滤**：鉴权连接不接受业务命令
3. **命令创建**：`cmd.NewCommand(cmdStr, cmdId, param, s)`（目前仅 `closews`、`ping`）
4. **只读模式校验**：写操作在只读模式被拒绝
5. **异步执行**：`cmd.Exec(command)` 通过 goroutine 异步执行

目前通过 WS 上行的命令只有：
- `ping` —— 心跳/保活（空操作）
- `closews` —— 关闭指定会话，调用 `ClosePushChan`

> **注意**：绝大多数业务操作仍通过 HTTP POST API 完成，WebSocket 主要用于 **服务端→客户端** 的实时推送。

### 3.6 独立广播通道 BroadcastChannels

[broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/api/broadcast.go#L42-L94)

独立于主 WebSocket 系统的广播通道，用于 API 层面的事件分发（如插件通信）：

- **路由**：`/ws/broadcast?channel=xxx`（WebSocket）和 `/es/broadcast/subscribe`（SSE）
- **消息转发**：某个客户端发送的消息会广播给该 channel 的所有其他订阅者
- **双通道支持**：同时支持 WebSocket 和 SSE 两种协议
- **通道销毁**：`DestroyBroadcastChannel` 在无订阅者时自动清理

---

## 4. 前端实现分析

### 4.1 Model 基类 —— WebSocket 客户端封装

[Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/Model.ts#L9-L107)

所有需要 WebSocket 连接的模块（App/Files/Protyle/Tag 等）都基于 `Model` 类：

#### 4.1.1 连接建立
```typescript
private connect(options: {
    id: string,          // 会话 ID（唯一）
    type?: TWS,          // 连接类型（main/filetree/protyle/...）
    callback?: () => void,
    msgCallback?: (data: IWebSocketData) => void
}) {
    const websocketURL = `${window.location.protocol === "https:" ? "wss" : "ws"}://${window.location.host}/ws`;
    const ws = new WebSocket(`${websocketURL}?app=${Constants.SIYUAN_APPID}&id=${options.id}${options.type ? "&type=" + options.type : ""}`);
}
```

关键参数：
- `Constants.SIYUAN_APPID`：前端启动时用 `Math.random().toString(36).substring(8)` 生成的应用实例 ID
- `id`：每个 Model 实例独立生成的会话 ID（通常是 `genUUID()`）

#### 4.1.2 生命周期事件

**onopen** —— 连接建立：
1. 调用用户 `callback`
2. 若存在 `errorLog` 对话框（内核中断后产生），则：
   - 触发 `reloadSync` 重新同步（仅此时触发，正常重连不触发）
   - 关闭错误对话框

**onmessage** —— 消息接收：
1. 前置检查：`window.siyuan.config` 必须存在（等待配置加载完成）
2. `processMessage()` 统一预处理
3. 交给具体业务的 `msgCallback` 分发

**onclose** —— 连接关闭：
1. 若 `ev.reason` 字符串**包含** `"unauthenticated"` 子串，**不重连**（鉴权失败）
2. 若 `ev.reason` 字符串**不包含** `"close websocket"` 子串，**3 秒后自动重连**（异常关闭）
3. 若 `ev.reason` 字符串**包含** `"close websocket"` 子串，**不重连**（服务端主动正常关闭）

> 判断逻辑使用 `String.indexOf()` 子串搜索，不是精确匹配。例如：
> - `"  unauthenticated"` → 匹配，不重连
> - `"  close websocket: publish service closed"` → 匹配 "close websocket"，不重连
> - 空字符串或网络错误码（如 "1006"）→ 不匹配，重连

**onerror** —— 连接错误：
- 仅对 URL 以 `&type=main` 结尾 且 `readyState=3` 的连接触发 `kernelError()` 提示
- `onerror` 本身不触发重连，通常随后会触发 `onclose`，由 `onclose` 逻辑处理

#### 4.1.3 send 方法
```typescript
public send(cmd: string, param: Record<string, unknown>, process = false) {
    this.reqId = process ? 0 : new Date().getTime();
    this.ws.send(JSON.stringify({
        cmd,
        reqId: this.reqId,
        param,
    }));
}
```

### 4.2 processMessage —— 消息预处理

[processMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/util/processMessage.ts#L10-L77)

在消息交给具体业务模块前，统一处理以下通用命令：

| cmd | 处理逻辑 |
|-----|----------|
| `msg` | 显示消息提示（`showMessage`），支持微软 Defender 排除项交互 |
| `cmsg` | 隐藏指定消息（`hideMessage`） |
| `cprogress` | 移除进度条 DOM |
| `reloadui` | 先 `exportLayout` 保存布局，再 `window.location.reload()` 全页刷新，`resetScroll` 时清空滚动位置 |
| `closepublishpage` | 调用 `handlePublishServiceClosed`，sessionStorage 存消息后刷新 |
| `code < 0` | 显示提示消息 (-2=info, -1=error) |
| 其他 | 返回原始 response，继续交给业务 msgCallback |

### 4.3 主连接 main

#### 4.3.1 桌面端
[index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/index.ts#L69-L214)

桌面端 main 连接处理 **20+ 种命令**，是所有连接中最复杂的：

| 命令分类 | 具体命令 |
|----------|----------|
| 系统控制 | `logoutAuth`、`exit`、`readonly`、`setConf`、`setPublish` |
| 外观主题 | `setAppearance`、`setSnippet`、`refreshtheme` |
| 插件扩展 | `reloadPlugin`、`reloadEmojiConf` |
| 文档操作 | `reloaddoc`、`rename`、`closeBox`、`removeBox`、`removeDoc`、`openFileById` |
| 进度状态 | `progress`、`statusbar`、`downloadProgress`、`txerr`、`backgroundtask` |
| 同步相关 | `syncing`、`syncMergeResult` |
| 引用/标签 | `setRefDynamicText`、`setDefRefCount`、`reloadTag` |
| 存储 | `setLocalStorageVal`、`setLocalShorthandCount` (浏览器) |

```typescript
ws: new Model({
    app: this,
    id: genUUID(),
    type: "main",
    msgCallback: (data) => {
        // 1. 插件事件总线广播（所有插件都能收到 main 消息）
        this.plugins.forEach((plugin) => {
            plugin.eventBus.emit("ws-main", data);
        });
        // 2. 内置命令分发 switch-case
        switch (data.cmd) { /* 20+ 种 */ }
    }
})
```

#### 4.3.2 移动端
[mobile/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/mobile/index.ts#L72-L82)

移动端主连接只做两件事：
1. 插件 `ws-main` 事件广播
2. 调用 `onMessage(this, data)`（移动端专用消息处理器）

移动端没有桌面端完整的 Tab 系统，因此 `rename`/`closeBox`/`removeDoc` 等命令由 `onMessage` 统一适配。

### 4.4 编辑器连接 protyle

[Protyle/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/index.ts#L127-L265)

每个编辑器实例建立独立的 `type=protyle` WebSocket 连接：

```typescript
this.protyle.ws = new Model({
    app,
    id: this.protyle.id,        // Protyle 实例 ID
    type: "protyle",
    msgCallback: (data) => {
        switch (data.cmd) {
            case "reload":
                // data.data === rootID 时才重载当前编辑器
                if (data.data === this.protyle.block.rootID) reloadProtyle(...)
                break;
            case "transactions":
                // 核心事务处理：逐 operation 应用 DOM 变更
                this.onTransaction(data);
                break;
            case "addLoading":
                if (data.data === this.protyle.block.rootID) addLoading(...)
                break;
            case "unfoldHeading":      setFoldById(...);
            case "readonly":           setReadonlyByConfig(...);
            case "heading2doc/li2doc": reloadProtyle 或重新 getDoc
            case "rename":             更新面包屑、标题、引用文本
            case "moveDoc":            更新 path 和 notebookId
            case "closeBox/removeBox": 关闭 Tab
            case "removeDoc":          关闭 Tab + 清理滚动位置
            case "refreshAttributeView": av 重新渲染
        }
    }
});
```

**onTransaction 事务处理流程**：
1. 预览模式下仅重新渲染预览
2. 遍历所有 `doOperations`
3. 反链面板中 delete/move 操作触发 refresh
4. 其他操作调用 `onTransaction(protyle, item, false)` 应用 DOM 变更
5. 内容为空时根据 action 决定是否创建空段落或重载

### 4.5 文件树连接 filetree

[Files.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/dock/Files.ts#L44-L118)

```typescript
super({
    app: options.app,
    type: "filetree",
    id: options.tab.id,
    msgCallback(data) {
        switch (data.cmd) {
            case "reloadFiletree": setNoteBook(() => this.init(false));
            case "reloadDocInfo":  this.updateDocInfo(data);
            case "moveDoc":        this.onMove(data);
            case "mount":          this.onMount(data);
            case "createnotebook": 插入新笔记本 HTML
            case "closeBox":       this.onRemove(data);
            case "removeBox":      this.onRemove(data);
            case "removeDoc":      this.onRemove(data);
            case "create":         this.selectItem 或 updateItemArrow
            case "rename":         this.onRename(data.data);
            case "renamenotebook": 更新笔记本名称文本
            case "createdailynote":选中新文档
            case "heading2doc":    选中新文档
            case "li2doc":         选中新文档
        }
    }
});
```

### 4.6 各订阅端处理差异汇总

#### 4.6.1 命令覆盖度对比

| 命令 | main | protyle | filetree | backlink | outline | bookmark | tag | graph |
|------|:----:|:-------:|:--------:|:--------:|:-------:|:--------:|:---:|:-----:|
| transactions | ✗ | ✓ 核心 | ✗ | ✗ | ✓ (savedoc触发) | ✓ 检测属性 | ✓ 检测标签 | ✗ |
| reload | ✗ | ✓ (rootID过滤) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| rename | ✓ (遍历Tab) | ✓ (path匹配) | ✓ | ✓ (rootID过滤) | ✓ | ✗ | ✗ | ✓ (box+rootID) |
| closeBox/removeBox | ✓ (遍历Tab) | ✓ | ✓ | ✓ (local型) | ✓ | ✓ 整体刷新 | ✓ 整体刷新 | ✓ |
| removeDoc | ✓ (遍历Tab) | ✓ | ✓ | ✓ (local型) | ✓ | ✓ 整体刷新 | ✓ 整体刷新 | ✓ (local型) |
| mount | ✗ | ✗ | ✓ | ✗ | ✗ | ✓ (code=1时跳过) | ✓ (code=1时跳过) | ✓ (global型) |
| progress/statusbar | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| readonly | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| reloadui | 全局(预处理) | - | - | - | - | - | - | - |

#### 4.6.2 文档级订阅的过滤模式差异

后端统一按 type 广播，前端各订阅端有三种过滤策略：

1. **全量消费**（bookmark、tag、filetree）：
   - 收到命令后直接触发 `this.update()` 整体刷新
   - 实现简单但性能较差，每次变更都全量重渲染

2. **ID 匹配消费**（protyle、backlink、outline、graph - local型）：
   ```typescript
   if (data.data === this.protyle.block.rootID) { reloadProtyle(...) }  // protyle
   if (data.data.ids.includes(this.rootId) && this.type === "local") {  // backlink
       this.parent.parent.removeTab(this.parent.id);
   }
   ```
   - 精确匹配自身关注的文档 ID，不匹配直接丢弃
   - 消息流量浪费：10 个文档时每条推送被 9 个连接丢弃

3. **全局无过滤**（main）：
   - 所有消息都处理
   - 通过遍历所有 Tab 来找到受影响的实例

### 4.7 移动端主动保活机制

#### 4.7.1 reconnectWebSocket 全局函数
[mobile/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/mobile/index.ts#L214-L220)

移动端特有、桌面端没有的主动保活机制（Issue #8441）：

```typescript
window.reconnectWebSocket = () => {
    window.siyuan.ws.send("ping", {});                  // main 连接
    window.siyuan.mobile.docks.file.send("ping", {});   // filetree 连接
    window.siyuan.mobile.editor.protyle.ws.send("ping", {});  // 当前编辑器
    window.siyuan.mobile.popEditor?.protyle.ws.send("ping", {});  // 弹窗编辑器
};
```

**特点**：
- 不是真正的"重连"，而是向所有活跃连接发送 `ping` 命令
- 由原生 App 层定期调用（通常是应用从后台切回前台时）
- 目的：**探测连接是否存活** + **触发 NAT/防火墙的会话保活**
- 若连接已断开，`send()` 会抛出异常并触发 `onerror`/`onclose`，进而启动自动重连

#### 4.7.2 后端 ping 命令
[ping.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/cmd/ping.go#L19-L32)

后端的 `ping` 命令是空操作：
```go
func (cmd *ping) Exec() {
    // 空实现，仅用于保活和连接探测
}
func (cmd *ping) IsRead() bool {
    return true
}
```

- `IsRead() = true`：只读命令，不受只读模式限制
- 无返回值，不产生任何推送
- 本质上是**客户端→服务端的单向心跳**，服务端不回复 pong

> **注意**：无自动定时器。ping 仅在移动端切前台等特定时机被动触发，桌面端完全没有应用层心跳。这是连接假死风险的核心来源。

---

## 5. 完整流程追踪（以编辑事务为例）

这是最典型也最复杂的实时事件链路。

### 5.1 总览流程

```
用户输入文字
    │
    ▼
[前端 Protyle WYSIWYG]
    │ input 事件触发事务收集
    │ transaction.ts → transaction()
    │ window.siyuan.transactions.push(...)
    │ transactionsTimeout → 512ms 防抖延时后批量提交
    ▼
[HTTP POST /api/transactions]
    │ 携带参数: transactions, reqId, app, session
    │
    ▼
[后端 performTransactions] ── transaction.go
    │ 1. JSON 解析 + 参数校验
    │ 2. model.PerformTransactions(&transactions)
    │    ├── 执行每个 Operation（insert/update/delete/...）
    │    ├── 写入文件系统（FlushTxQueue）
    │    └── 更新 SQL 索引
    │ 3. ret.Data = transactions （带 RetData 返回值）
    │ 4. pushTransactions(app, session, transactions)
    │    ├── 判断是否 AttrView 事务决定 PushMode
    │    ├── 提取 rootIDs 放入 evt.Context
    │    ├── WaitForCommit 等待提交完成
    │    └── util.PushEvent(evt) → 路由到 BroadcastOthers 等
    ▼
[后端 PushEvent] ── websocket.go
    │ 根据 PushMode 遍历 sessions sync.Map
    │ 对 type=protyle 的每个匹配 session:
    │     session.Write(event.Bytes())
    ▼
[前端 Protyle Model.onmessage]
    │ JSON.parse(event.data)
    │ processMessage(response) （预处理，通常无匹配直接返回）
    ▼
[Protyle msgCallback → case "transactions"]
    │ this.onTransaction(data)
    │   ├── 预览模式: this.protyle.preview.render()
    │   └── 编辑模式: 对每个 doOperation 执行 onTransaction()
    │       ├── insert  → DOM 节点插入
    │       ├── update  → DOM 属性/内容更新
    │       ├── delete  → DOM 节点移除
    │       ├── move    → DOM 节点移动
    │       └── ... 其他 50+ 种操作类型
    ▼
用户看到实时更新
```

### 5.2 前端防抖合并详细时序

[transaction.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/wysiwyg/transaction.ts#L1364-L1460)

涉及两个独立的时间参数，需严格区分：

| 参数名 | 值 | 用途 | 代码位置 |
|--------|-----|------|----------|
| `TIMEOUT_INPUT` | 256ms | **合并判断窗口**：两次输入时间间隔小于此值才可能合并 | L1387: `protyle.transactionTime - time < Constants.TIMEOUT_INPUT` |
| `TIMEOUT_INPUT * 2` | 512ms | **防抖提交延时**：最后一次输入后等待多久提交 HTTP | L1449-1451: `setTimeout(promiseTransaction, Constants.TIMEOUT_INPUT * 2)` |

#### 5.2.1 合并判断条件（256ms 窗口）

```typescript
public static readonly TIMEOUT_INPUT = 256;  // constants.ts L302

let needDebounce = false;
if (lastTransaction && 
    lastTransaction.doOperations.length === 1 && 
    lastTransaction.doOperations[0].action === "update" &&
    doOperations.length === 1 && 
    doOperations[0].action === "update" &&
    lastTransaction.doOperations[0].id === doOperations[0].id &&
    protyle.transactionTime - time < Constants.TIMEOUT_INPUT  // < 256ms
) {
    needDebounce = true;
}
```

**5 个条件必须同时满足**才能合并：
1. 上一个事务存在
2. 上一个事务只有 1 个 operation 且是 `update` 类型
3. 当前事务也只有 1 个 operation 且是 `update` 类型
4. 两次操作的 **块 ID 相同**（同一块连续编辑）
5. 时间间隔 < **256ms**

满足条件时，**原地替换** `window.siyuan.transactions` 数组中最后一个元素的 `doOperations`，不新增条目。

#### 5.2.2 防抖提交流程（512ms 延时）

```
t=0ms   用户输入字符 'a'
        → transaction() 被调用
        → lastTransaction 不存在，needDebounce=false
        → transactions.push({doOperations: [{action:'update', id:'xxx', data:'a'}]})
        → protyle.transactionTime = time
        → transactionsTimeout = setTimeout(promiseTransaction, 512ms)

t=100ms 用户输入字符 'b'
        → transaction() 被调用
        → 5个条件都满足（间隔 100ms < 256ms），needDebounce=true
        → 原地替换：transactions[last].doOperations = [{action:'update', id:'xxx', data:'ab'}]
        → protyle.transactionTime = time
        → clearTimeout(transactionsTimeout)
        → transactionsTimeout = setTimeout(promiseTransaction, 512ms)
        (防抖重置：再等 512ms)

t=612ms 定时器触发（t=100ms + 512ms）
        → promiseTransaction() 执行
        → 取出 transactions[0] 发送 HTTP 请求
        → transactions.splice(0, 1)
```

**关键点**：
- 两次输入的合并窗口是 **256ms**（判断是否为同一块的连续编辑）
- 每次输入后重置的防抖延时是 **512ms**（等待多久确认不再输入才提交）
- `transactions` 数组是 **FIFO 队列**，`promiseTransaction` 按顺序逐个提交

#### 5.2.3 promiseTransaction 串行提交

[transaction.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/wysiwyg/transaction.ts#L64-L140)

```typescript
const promiseTransaction = () => {
    if (window.siyuan.transactions.length === 0) return;
    
    // 取出队首事务
    const protyle = window.siyuan.transactions[0].protyle;
    const doOperations = window.siyuan.transactions[0].doOperations;
    
    // 立即从队列中移除（不等 HTTP 返回）
    // 重要：不能放入回调中，否则防抖合并会出问题
    window.siyuan.transactions.splice(0, 1);
    
    fetchPost("/api/transactions", { ... }, (response) => {
        // 回调中如果队列还有事务，继续提交下一个
        if (window.siyuan.transactions.length === 0) {
            countBlockWord([], protyle.block.rootID, true);
        } else {
            promiseTransaction();  // 递归调用，串行提交
        }
        
        // 本地 DOM 优化处理（fold/update/delete/append 等）
        response.data[0].doOperations.forEach(...);
    });
};
```

**串行保证**：
- 同一时间只有一个 HTTP 请求在途
- 当前请求返回后才发送下一个
- 队列中积压的事务依次顺序提交

#### 5.2.4 旁路：直接提交（不防抖）

以下操作 **跳过防抖**，直接发送 HTTP 请求：

```typescript
// 折叠、属性视图设置等操作需要即时响应
if (doOperations.length === 1 && (
    doOperations[0].action === "unfoldHeading" || 
    doOperations[0].action === "setAttrViewBlockView" ||
    (doOperations[0].action === "setAttrs" && doOperations[0].data.startsWith('{"fold":'))
) || (doOperations.length === 2 && doOperations[0].action === "insertAttrViewBlock")) {
    protyle.transactionTime = time + Constants.TIMEOUT_INPUT * 2;  // 阻止合并
    fetchPost("/api/transactions", ...);
    return;
}
```

### 5.3 后端串行化保证

#### 5.3.1 ControlConcurrency 中间件

[session.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/model/session.go#L456-L507)

```go
func ControlConcurrency(c *gin.Context) {
    // WebSocket 升级请求不控制
    if websocket.IsWebSocketUpgrade(c.Request) {
        c.Next()
        return
    }
    
    // 静态资源、读接口直接放行
    if strings.HasPrefix(reqPath, "/stage/") || ... ||
       strings.HasPrefix(function, "get") || strings.HasPrefix(function, "list") ||
       strings.HasPrefix(function, "search") || strings.HasPrefix(function, "render") {
        c.Next()
        return
    }
    
    // 写接口：按路径粒度加锁
    requestingLock.Lock()
    mutex := requesting[reqPath]
    if nil == mutex {
        mutex = &sync.Mutex{}
        requesting[reqPath] = mutex
    }
    requestingLock.Unlock()
    
    mutex.Lock()       // 同一 API 路径串行执行
    defer mutex.Unlock()
    c.Next()
}
```

**锁粒度**：**按 API 路径（reqPath）** 加互斥锁，不是全局锁。

- `/api/transactions` 是同一个路径，所以所有事务请求串行执行
- 不同 API 路径（如 `/api/filetree/createDoc` 和 `/api/transactions`）可以并行
- 读接口（get/list/search/render 开头）完全不加锁

#### 5.3.2 事务内顺序保证

[transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/api/transaction.go#L94-L124)

```go
func pushTransactions(app, session string, transactions []*model.Transaction) {
    model.FlushTxQueue()   // 刷新文件写入队列
    tx.WaitForCommit()     // 等待事务提交完成
    
    // 然后才推送
    util.PushEvent(evt)
}
```

- `FlushTxQueue()`：确保文件系统写入完成
- `WaitForCommit()`：确保所有事务操作提交完毕
- 推送发生在**所有持久化完成之后**，避免推送了但文件没写入的竞态

### 5.4 消息顺序的多层保证总结

| 层级 | 机制 | 保证范围 | 代码依据 |
|------|------|----------|----------|
| 前端防抖层 | `transactions` FIFO 队列 + `promiseTransaction` 递归回调串行提交 | 同一编辑器的操作按顺序提交（在途仅一个 HTTP） | [transaction.ts L64-L86](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/wysiwyg/transaction.ts#L64-L86) |
| HTTP 传输层 | TCP 保证顺序 | 单个请求内的字节流顺序 | TCP 协议 |
| 后端中间件 | `ControlConcurrency` 按 API 路径加互斥锁 | `/api/transactions` 的所有请求串行执行（同一 API 路径仅一个 goroutine 在处理） | [session.go L456-L507](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/model/session.go#L456-L507) |
| 事务层 | `WaitForCommit` + `FlushTxQueue` | 推送发生在所有持久化完成之后，避免推送了但文件未写入 | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/api/transaction.go) |
| WebSocket 层 | TCP 保证顺序 + melody 单连接串行 `session.Write` | 单个 WebSocket 连接内消息按发送顺序到达 | TCP 协议 + melody 内部 Write 序列化 |

**无法保证的范围**：
- 不同 `type` 连接之间（如 main 与 protyle）的消息到达顺序，因为是不同的 TCP 连接
- 重连期间丢失的消息，因为没有重放机制
- 后端 `BroadcastByType` 遍历过程中新加入/离开的连接，遍历快照可能不含该连接

---

## 6. 连接生命周期与断线恢复

### 6.1 完整生命周期状态图

```
       前端启动
         │
         ▼
   new Model(...)
   构造函数调用 connect()
         │
         ▼
  new WebSocket(url)  ───────┐
         │                   │
         ▼                   │ 失败 / 3秒超时
      onopen                 │ (onerror)
         │                   │
         ├── callback()      │
         └── 关闭错误对话框  │
         │                   │
         ▼                   │
  ◄───── 正常使用 ───────────►│
  │  onmessage 处理推送      │
  │  send(closews/ping)      │
         │                   │
         ▼                   │
      onclose ◄──────────────┘
         │
         ├─ reason 包含 "unauthenticated"  ──► 永久停止（鉴权失败）
         ├─ reason 包含 "close websocket"  ──► 永久停止（正常主动关闭）
         └─ 其他（不含上述两串）                │
               │ 3000ms setTimeout             │
               └──────► connect() 重连 ◄──────┘
                         （新 WebSocket 对象）
```

### 6.2 重连触发边界

[Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/Model.ts#L65-L85)

**不重连的两种情况**：
```typescript
if (0 <= ev.reason.indexOf("unauthenticated")) {
    return;  // 鉴权失败，不重连
}
if (0 > ev.reason.indexOf("close websocket")) {
    // 原因中不包含 "close websocket" → 异常关闭 → 重连
    setTimeout(() => { this.connect(...) }, 3000);
}
// 原因中包含 "close websocket" → 主动正常关闭 → 不重连
```

**重连判定边界**：
- ✅ **自动重连**：网络断开、超时、服务器重启、NAT 会话过期
- ❌ **不重连**：鉴权失败（unauthenticated）、主动关闭（close websocket）
- ⚠️ **灰区**：`onerror` 不直接触发重连，但错误通常会导致 `onclose`，由 onclose 逻辑决定

### 6.3 重连恢复的边界与局限

#### 6.3.1 重连后的恢复动作

[Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/Model.ts#L41-L56)

```typescript
ws.onopen = () => {
    if (options.callback) {
        options.callback.call(this);
    }
    // 如果有 errorLog 对话框（内核中断后产生）
    const logElement = document.getElementById("errorLog");
    if (logElement) {
        reloadSync(this.app, {upsertRootIDs: [], removeRootIDs: []});
        // 关闭错误对话框
        window.siyuan.dialogs.find(item => { ... }).destroy();
    }
};
```

**恢复动作仅在有 errorLog 对话框时触发**：
- `reloadSync()` 重新同步文档数据
- 关闭错误对话框

**正常重连（无 errorLog）时**：仅调用 `callback()`，**没有任何数据同步动作**。

> 这意味着：大多数情况下的短时间断开重连，重连期间丢失的推送消息**不会被补偿**，UI 可能处于陈旧状态。

#### 6.3.2 消息延迟消费边界

[Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/Model.ts#L57-L64)

```typescript
ws.onmessage = (event) => {
    if (options.msgCallback &&
        // 等待 config 加载完成才接受推送
        window.siyuan.config) {
        const data = processMessage(JSON.parse(event.data));
        options.msgCallback.call(this, data);
    }
};
```

**消息丢弃条件**：`window.siyuan.config` 未加载时，所有消息直接丢弃。

对应 Issue #17508：应用启动初期 WebSocket 可能先于配置建立，此时消息会被丢弃，等待配置加载完成后才开始消费。

#### 6.3.3 重连期间丢失的消息类型

| 消息类型 | 丢失后的影响 | 是否有恢复机制 |
|----------|-------------|---------------|
| transactions | 编辑器内容与后端不一致 | ❌ 无（只能靠 reloadSync 全量同步当前打开的文档） |
| rename | Tab 标题、面包屑陈旧 | ❌ 无 |
| closeBox/removeDoc | 已删除的文档仍显示在 Tab 中 | ❌ 无（用户点击时才会发现不存在） |
| progress/statusbar | 进度条状态错误 | ❌ 无（瞬时状态，丢失就丢失了） |
| msg/cmsg | 通知消息丢失 | ❌ 无（瞬时消息） |
| reloadui | UI 未刷新 | ❌ 无（下一次操作会触发） |

### 6.4 主动关闭流程

1. **前端销毁 Protyle**：[destroy.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/util/destroy.ts#L26-L29)
   ```typescript
   protyle.ws.send("closews", {});
   ```

2. **后端 closews 命令**：[closews.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/cmd/closews.go#L27-L31)
   ```go
   func (cmd *closews) Exec() {
       id, _ := cmd.session.Get("id")
       util.ClosePushChan(id.(string))  // 遍历关闭匹配 id 的连接
       cmd.Push()
   }
   ```

3. **HandleDisconnect 回调**：`util.RemovePushChan(s)` 清理 sessions Map

### 6.5 发布服务特殊关闭

[websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/websocket.go#L509-L550) 中的 `ClosePublishServiceSessions`：

1. 遍历所有 session，收集 `isPublish=true` 的连接
2. 先发送 `closepublishpage` 通知客户端
3. `time.Sleep(500ms)` 等待消息发送 + 客户端页面刷新准备
4. 用 `"  close websocket: publish service closed"` 作为关闭消息，客户端停止重连
5. 调用 `RemovePushChan` 清理

### 6.6 多连接订阅的边界条件

#### 6.6.1 连接创建的时机

| 连接类型 | 创建时机 | 销毁时机 |
|----------|----------|----------|
| main | App 构造时（应用启动） | 应用关闭 |
| filetree | Dock Files 初始化时 | 应用关闭（Dock 不销毁） |
| bookmark | Dock Bookmark 初始化时 | 应用关闭 |
| tag | Dock Tag 初始化时 | 应用关闭 |
| protyle | Protyle 实例化时（打开文档） | Protyle destroy 时（关闭文档） |
| outline | Outline Tab 打开时 | Tab 关闭时 |
| backlink | Backlink Tab 打开时 | Tab 关闭时 |
| graph | Graph Tab 打开时 | Tab 关闭时 |

#### 6.6.2 连接隔离性

- **每个连接独立鉴权**：各自在 URL 中携带 app/id/type，后端分别注册到 sessions Map
- **每个连接独立生命周期**：一个连接断开不影响其他连接
- **每个连接独立重连**：各连接的 onclose 独立触发 3 秒重连定时器，不同步

#### 6.6.3 消息重复与竞态边界

**同一事件多次投递**：
- 一个 `rename` 事件会同时推送给 main、protyle、filetree、outline、backlink、graph 等多种 type
- 前端不同模块独立处理，可能产生重复的 UI 更新（如标题更新同时被 main 和 protyle 处理）

**跨连接顺序不一致**：
- 后端 `BroadcastByType` 按 type 分别遍历发送
- 不同 type 的连接是不同的 TCP 连接，到达顺序无保证
- 例如：`transactions`（推给 protyle）和 `savedoc`（推给 outline）之间没有全局顺序

---

## 7. 错误处理与资源释放

### 7.1 错误处理机制

#### 后端层面：
1. **ServeAPI Recover 中间件**：`model.Recover` 捕获 panic，避免服务崩溃
2. **Command Exec Recover**：[cmd.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/cmd/cmd.go#L77-L82) 中 `defer logging.Recover()` 保护命令执行
3. **广播异常**：`session.Write()` 错误时 melody 内部会自动关闭连接，触发 HandleDisconnect 清理
4. **事务错误**：`TxErr` 结构体返回错误码（BlockNotFound/Locked/PushMsg），通过 `PushTxErr`/`PushMsg` 通知前端
5. **JSON 解析错误**：直接返回 `code=-1, msg="Bad Request"` 给发送方

#### 前端层面：
1. **onerror 处理**：仅 `type=main` 连接 `readyState=3` 时展示 `kernelError()` 对话框
2. **消息过滤**：`processMessage` 对 `code<0` 统一转提示消息
3. **重连恢复**：非主动关闭时 3 秒后自动重连；**仅当存在 errorLog 对话框时**，重连成功才触发 `reloadSync` 数据重新同步；正常重连不触发数据同步

### 7.2 资源释放

#### 连接资源：
1. **sessions Map 层级清理**：RemovePushChan 不仅删除 sessionId，若 appId 下无 session 也删除 appId 节点
2. **authSessions 独立清理**：鉴权会话独立管理，避免混入业务 sessions
3. **BroadcastChannels 自动清理**：
   - `PruneBroadcastChannels()` 定期清理无订阅者的通道
   - 通道关闭时 `broadcastChannel.Destroy(true)` 调用 `melody.Close()` 和 SSE close 事件

#### 前端资源：
1. **Protyle 销毁**：`destroy()` 方法发送 `closews`，通知后端释放 session
2. **错误日志对话框**：重连成功时自动关闭并清理 UI

---

## 8. 协作机制

### 8.1 多端协作（多应用实例）

当多个浏览器窗口/设备同时连接时：

1. **编辑事务**：使用 `PushModeBroadcastExcludeSelf`（模式 2），排除发起者自身
   - 用户 A 在窗口 1 编辑 → 窗口 2/3 收到 transactions 推送 → 自动 DOM 同步
   - 发起者窗口不重复推送，避免本地操作被覆盖

2. **AttrView 事务**：使用 `PushModeBroadcast`（模式 0），全量广播
   - 属性视图的数据模型复杂，需要所有端（包括发起者）统一重新渲染

3. **AppId 隔离**：
   - `BroadcastOtherApps` / `PushModeBroadcastExcludeSelfApp`（模式 4）可排除整个应用实例
   - `PushModeBroadcastApp`（模式 5）只推给指定应用内的所有连接
   - `PushModeBroadcastMainExcludeSelfApp`（模式 6）跨应用仅推送主连接

### 8.2 插件协作

插件通过两种方式参与 WebSocket：

1. **监听主连接**：`plugin.eventBus.emit("ws-main", data)` —— 主连接的每条消息都会广播给插件事件总线
2. **独立广播通道**：通过 `/ws/broadcast?channel=plugin-xxx` + `/api/broadcast/postMessage` 建立插件间通信

---

## 9. 潜在风险点

| # | 风险类别 | 具体描述 | 影响范围 | 严重程度 | 建议 |
|---|----------|----------|----------|----------|------|
| 1 | **连接风暴** | 每个编辑器建立独立 WS 连接，打开 30+ Tab 时连接数剧增；重连时同时建立大量连接 | 内存、文件描述符耗尽 | 高 | 考虑多路复用（单一连接，消息路由时增加 type 字段判断） |
| 2 | **消息顺序 - 跨连接** | 不同 type 的连接之间（main/protyle/filetree）没有全局顺序保证，同一事件多端推送可能乱序到达 | UI 状态不一致、闪烁 | 中 | 增加全局单调递增序号，前端按序号排队消费 |
| 3 | **消息顺序 - 重连丢失** | 重连期间丢失的推送不会重放；`reloadSync` 仅部分恢复；多个 HTTP 请求并发（尽管 ControlConcurrency 已大部分避免） | 数据不一致/UI 陈旧 | 高 | 增加消息序号，重连后 catch-up 拉取缺失事件 |
| 4 | **内存泄漏 - 僵尸连接** | sessions sync.Map 中若 HandleDisconnect 未及时触发（网络异常无 TCP FIN），僵尸连接堆积；BroadcastChannels 未正确清理 | 内存缓慢增长 | 中 | 增加心跳超时检测，周期性清理无响应 session |
| 5 | **广播放大 - 全量遍历** | `BroadcastByType` 每次都全量遍历所有 app 的所有 session，O(N) 复杂度，高并发时 CPU 飙升 | 延迟、CPU 占用 | 中 | 按 type 建立索引 Map（`map[type][]session`），避免全量遍历 |
| 6 | **广播放大 - 文档级浪费** | 后端按 type 全量广播，前端 90%+ 消息因 ID 不匹配被丢弃（打开 10 个文档时 protyle 连接浪费 90% 流量） | 带宽、前端 CPU | 中 | 后端支持按文档 ID 订阅（如 `/ws?type=protyle&rootId=xxx`） |
| 7 | **消息大小** | 事务推送包含完整 `doOperations`，大块文档编辑时单条消息可达数 MB，超出 melody 的 8MB MaxMessageSize（上行），下行不受限但带宽压力大 | 推送失败、丢包 | 中 | 增量 diff 传输；大事务拆分为多次推送 |
| 8 | **重连数据同步不足** | 正常重连（无 errorLog 对话框）时不触发 `reloadSync`，重连期间丢失的所有消息永久丢失 | UI 状态不一致 | 高 | 重连成功后统一触发全量同步；或引入事件溯源重放 |
| 9 | **启动期消息丢弃** | `window.siyuan.config` 未加载时，所有 WebSocket 消息直接丢弃（Issue #17508） | 初始化状态不一致 | 低 | 消息缓存队列，config 加载后重放缓存消息 |
| 10 | **权限绕过** | WebSocket 鉴权仅在 HandleConnect 时执行，长连接期间权限变更（角色降级/注销）不会失效 | 已降级用户仍能接收推送 | 中 | 敏感推送增加二次校验；支持服务端主动踢人下线 |
| 11 | **鉴权重连体验差** | reason 含 "unauthenticated" 时不重连，但 Cookie 过期后页面刷新才能重新登录 | 用户体验差 | 低 | 检测到 unauthenticated 时自动跳转 `/check-auth` 页面 |
| 12 | **心跳缺失 - 桌面端** | 桌面端完全没有应用层心跳，NAT/防火墙空闲超时会导致连接假死，只有用户操作触发 send 时才发现断开 | 连接假死、消息延迟 | 高 | 应用层 30s 定时 ping/pong，超时主动关闭触发重连 |
| 13 | **心跳缺失 - 移动端** | 移动端仅被动触发 `reconnectWebSocket`（切前台时），无后台定时器；ping 是单向的，服务端不回复 pong | 后台时连接易断 | 中 | 增加定时 ping 定时器；服务端回复 pong 用于 RTT 计算 |
| 14 | **发布服务竞态** | `ClosePublishServiceSessions` 中 `time.Sleep(500ms)` 是硬编码等待，高负载下消息可能未发出就关闭连接 | 部分客户端收不到关闭通知 | 低 | 使用 Write 回调 + WaitGroup 替代 sleep |
| 15 | **事务防抖竞态** | 512ms 防抖期间若 WebSocket 断开，本地事务未提交但可能已有推送基于旧状态；`transactions` 队列在页面刷新时丢失 | 数据丢失 | 中 | 提交失败时事务回滚 + 本地重做队列；localStorage 持久化待提交事务 |
| 16 | **BroadcastByType 原子性** | 遍历 sync.Map 过程中若新 session 加入/离开，遍历快照可能不包含该 session | 消息漏发/重复 | 低 | 使用不可变快照 + 版本号确认 |
| 17 | **ControlConcurrency 死锁风险** | 按 API 路径粒度加锁，若处理函数内部调用另一个加锁 API（嵌套请求）会产生死锁 | 服务挂起 | 低 | 增加可重入检测；或明确禁止嵌套写请求 |
| 18 | **多连接重复 UI 更新** | 同一事件（如 rename）同时推送给多个 type，前端多模块独立处理，可能产生重复渲染/闪烁 | UI 体验差 | 低 | 事件去重；或统一事件总线再分发 |

---

## 10. 进一步研究方向

### 10.1 架构优化方向

1. **连接多路复用（Multiplexing）**：
   - 单一 WebSocket 连接承载所有 type 的消息，通过消息头增加 `targetTypes: ["main", "protyle:xxx"]` 字段路由
   - 预期收益：连接数减少 70%+，降低服务端和浏览器资源消耗
   - 特别利好移动端，减少蜂窝网络下的连接开销

2. **后端按文档 ID 订阅**：
   - 目前 protyle/outline/backlink 等文档级连接的消息 90%+ 被前端丢弃
   - 研究支持 `rootId` 参数的订阅：`/ws?type=protyle&rootId=xxx`
   - 后端按 rootId 建立索引，只推送相关文档的事务

3. **事件溯源（Event Sourcing）**：
   - 建立事件日志（Append-Only Log），每条消息分配单调递增全局序号
   - 重连时携带 `lastSeenSeq`，服务端按序号重放缺失事件
   - 彻底解决重连期间事件丢失问题

4. **增量事务传输**：
   - 目前 transactions 推送包含完整 `doOperations` 数组
   - 研究基于 CRDT 的增量同步，或基于 OT（Operational Transform）的操作变换
   - 对多人实时协作至关重要

5. **统一事件总线**：
   - 目前同一事件（如 rename）被分别推送到多个 type，前端多模块独立处理
   - 建立全局事件总线，消息一次到达后分发给各订阅者
   - 减少重复解析和重复渲染

### 10.2 可靠性增强方向

6. **应用层心跳 + 连接健康度监控**：
   - 前后端双向 30s ping/pong（非 TCP level，而是业务层命令）
   - 服务端回复 pong 携带时间戳，用于计算 RTT
   - 连续 3 次无响应判定为死链，主动关闭 + 重连
   - 导出指标：连接数、消息速率、平均延迟、重连次数

7. **重连数据补偿机制**：
   - 重连成功后统一触发 `reloadSync`，而不仅是有 errorLog 时才触发
   - 对非事务类消息（statusbar/progress）：丢失即丢失，不必补偿
   - 对状态类消息（rename/closeBox/removeDoc）：通过全量同步补偿

8. **背压（Backpressure）机制**：
   - 前端处理缓慢（长时间事务渲染）时通知后端降低推送频率
   - 服务端发送缓冲区水位监控，超过阈值时触发合并推送

9. **消息持久化 + 离线补偿**：
   - 关键事件（文档创建/删除/重命名）写入 SQLite 事件表
   - 浏览器恢复网络时按时间窗口补齐

### 10.3 安全增强方向

10. **消息签名/防篡改**：
    - 敏感命令（如 `closeBox`/`removeDoc`）增加 HMAC 签名校验
    - 防止 XSS 成功后伪造 WebSocket 消息执行破坏性操作

11. **会话绑定与踢人**：
    - WebSocket 绑定 JWT 的 sessionId，JWT 失效时主动断开对应连接
    - 用户修改密码/注销账号时遍历 sessions Map 踢掉所有关联连接

### 10.4 可观测性方向

12. **链路追踪**：
    - 请求入站时生成 TraceId，通过 HTTP Header → 事务 → WebSocket 消息 → 前端全链路透传
    - 便于排查 "用户 A 的修改为什么用户 B 没看到" 类问题

13. **前端 WebSocket DevTools**：
    - 开发模式下可视化展示：连接状态、消息队列、推送 cmd 统计、重连次数、平均延迟
    - 辅助插件开发者调试

### 10.5 移动端专项优化

14. **移动端后台连接管理**：
    - 应用退到后台时主动维持一个最低限度的连接（仅 main）
    - 关闭 protyle/filetree 等非关键连接，节省电量和流量
    - 切回前台时批量恢复连接 + 同步数据

15. **弱网适应**：
    - 2G/3G 弱网下降低事务推送频率，合并为更大的批量
    - 增加消息超时重传机制

---

## 11. 关键文件索引

### 11.1 后端核心文件

| 模块 | 文件 | 关键职责 |
|------|------|----------|
| 服务入口 | [serve.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/server/serve.go#L697-L851) | WebSocket 服务器初始化、鉴权、消息接收、命令分发 |
| 连接管理 | [websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/websocket.go) | 会话注册/注销、多种广播策略、30+ 种推送函数 |
| 消息结构 | [result.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/result.go) | Result 消息结构、6 种 PushMode 常量 |
| 事务推送 | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/api/transaction.go) | 事务执行、pushTransactions 推送模式选择、rootIDs 提取 |
| 并发控制 | [session.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/model/session.go#L456-L507) | ControlConcurrency 中间件、按 API 路径加锁 |
| 独立广播 | [broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/api/broadcast.go) | BroadcastChannels、SSE 支持、广播 API |
| 命令框架 | [cmd/cmd.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/cmd/cmd.go) | Cmd 接口定义、命令工厂、异步执行与 Recover |
| 关闭命令 | [cmd/closews.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/cmd/closews.go) | 主动关闭 WebSocket 连接 |
| 心跳命令 | [cmd/ping.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/cmd/ping.go) | ping 保活命令（空操作） |

### 11.2 前端核心文件

| 模块 | 文件 | 关键职责 |
|------|------|----------|
| 连接基类 | [Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/Model.ts) | WebSocket 连接封装、重连逻辑、消息回调、send 方法 |
| 消息预处理 | [processMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/util/processMessage.ts) | msg/cmsg/cprogress/reloadui/closepublishpage 通用处理 |
| 桌面主连接 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/index.ts#L69-L214) | 桌面端 main 连接，20+ 种命令分发 |
| 移动主连接 | [mobile/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/mobile/index.ts#L72-L220) | 移动端 main 连接、reconnectWebSocket 保活函数 |
| 编辑器连接 | [protyle/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/index.ts#L127-L350) | Protyle 连接，transactions/reload 等编辑器命令 |
| 文件树连接 | [layout/dock/Files.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/dock/Files.ts#L44-L118) | filetree 连接，文档树/笔记本变更处理 |
| 大纲连接 | [layout/dock/Outline.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/dock/Outline.ts) | outline 连接，大纲标题变更处理 |
| 反链连接 | [layout/dock/Backlink.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/dock/Backlink.ts) | backlink 连接，反链面板生命周期事件 |
| 书签连接 | [layout/dock/Bookmark.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/dock/Bookmark.ts) | bookmark 连接，书签属性变更检测 |
| 标签连接 | [layout/dock/Tag.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/dock/Tag.ts) | tag 连接，标签属性变更检测 |
| 图连接 | [layout/dock/Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/dock/Graph.ts) | graph 连接，图视图变更处理 |
| 事务提交 | [protyle/wysiwyg/transaction.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/wysiwyg/transaction.ts) | 本地事务收集、512ms 防抖、promiseTransaction 串行提交 |
| 类型定义 | [types/index.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/types/index.d.ts#L3-L3) | TWS 类型、IWebSocketData 接口定义 |
| 常量定义 | [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/constants.ts#L20-L302) | SIYUAN_APPID、TIMEOUT_INPUT=256 等常量 |
