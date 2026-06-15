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
   - 触发 `reloadSync` 重新同步
   - 关闭错误对话框

**onmessage** —— 消息接收：
1. 前置检查：`window.siyuan.config` 必须存在（等待配置加载完成）
2. `processMessage()` 统一预处理
3. 交给具体业务的 `msgCallback` 分发

**onclose** —— 连接关闭：
1. 若原因为 `unauthenticated`，不重连（鉴权失败）
2. 若原因为 `close websocket`，不重连（服务端主动正常关闭）
3. **否则 3 秒后自动重连**：重新调用 `connect()`，使用相同的 id/type/msgCallback

**onerror** —— 连接错误：
- 仅对 `type=main` 的连接（连接状态 `readyState=3`）触发 `kernelError()` 提示

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

```typescript
ws: new Model({
    app: this,
    id: genUUID(),
    type: "main",
    msgCallback: (data) => {
        // 1. 插件事件总线广播
        this.plugins.forEach((plugin) => {
            plugin.eventBus.emit("ws-main", data);
        });
        // 2. 内置命令分发 switch-case
        switch (data.cmd) {
            case "logoutAuth":         redirectToCheckAuth();
            case "setAppearance":      updateAppearance(data.data);
            case "reloadPlugin":       reloadPlugin(this, data.data);
            case "reloaddoc":          reloadSync(...);
            case "progress":           progressLoading(data);
            case "statusbar":          progressStatus(data);
            case "downloadProgress":   downloadProgress(data.data);
            case "txerr":              transactionError(data.msg);
            case "syncing":            processSync(data, this.plugins);
            case "backgroundtask":     progressBackgroundTask(data.data.tasks);
            case "rename":             遍历 Tab 更新标题;
            case "closeBox":           遍历 Tab 关闭笔记本;
            case "removeDoc":          遍历 Tab 关闭文档;
            // ... 共 20+ 种命令
        }
    }
})
```

#### 4.3.2 移动端
[mobile/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/mobile/index.ts#L72-L82)

移动端主连接只做两件事：
1. 插件 `ws-main` 事件广播
2. 调用 `onMessage(this, data)`（移动端专用消息处理器）

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

---

## 5. 完整流程追踪（以编辑事务为例）

这是最典型也最复杂的实时事件链路：

```
用户输入文字
    │
    ▼
[前端 Protyle WYSIWYG]
    │ input 事件触发事务收集
    │ transaction.ts → transaction()
    │ window.siyuan.transactions.push(...)
    │ transactionsTimeout → 256ms 防抖后批量提交
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

**消息顺序保证**：
1. HTTP 请求层面由 `model.ControlConcurrency` 中间件保证 **串行化**执行
2. 同一事务的 `WaitForCommit()` 确保文件 IO 完成才推送
3. 前端 `transactionsTimeout` 防抖确保批量提交的顺序性
4. WebSocket 层使用 TCP 保证消息按发送顺序到达

---

## 6. 连接生命周期管理

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
         ├─ reason 含 "unauthenticated"  ──► 停止
         ├─ reason 含 "close websocket"  ──► 停止（正常关闭）
         └─ 其他原因                           │
               │ 3000ms setTimeout             │
               └──────► connect() 重连 ◄──────┘
                         （新 WebSocket 对象）
```

### 6.2 主动关闭流程

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

### 6.3 发布服务特殊关闭

[websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/websocket.go#L509-L550) 中的 `ClosePublishServiceSessions`：

1. 遍历所有 session，收集 `isPublish=true` 的连接
2. 先发送 `closepublishpage` 通知客户端
3. `time.Sleep(500ms)` 等待消息发送 + 客户端页面刷新准备
4. 用 `"  close websocket: publish service closed"` 作为关闭消息，客户端停止重连
5. 调用 `RemovePushChan` 清理

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
1. **onerror 处理**：仅 main 连接错误时展示 `kernelError()` 对话框
2. **消息过滤**：`processMessage` 对 `code<0` 统一转提示消息
3. **重连恢复**：非主动关闭时 3 秒后自动重连，重连成功时触发 `reloadSync` 数据重新同步

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

| 风险类别 | 具体描述 | 影响范围 | 建议 |
|----------|----------|----------|------|
| **连接风暴** | 每个编辑器建立独立 WS 连接，打开 30+ Tab 时连接数剧增；重连时同时建立大量连接 | 内存、文件描述符耗尽 | 考虑多路复用（单一连接，消息路由时增加 type 字段判断） |
| **消息顺序** | 重连期间丢失的推送不会重放；`reloadSync` 仅部分恢复；多个 HTTP 请求并发（尽管 ControlConcurrency 已大部分避免） | 数据不一致/UI 陈旧 | 增加消息序号，重连后 catch-up 拉取缺失事件 |
| **内存泄漏** | sessions sync.Map 中若 HandleDisconnect 未及时触发（网络异常无 TCP FIN），僵尸连接堆积；BroadcastChannels 未正确清理 | 内存缓慢增长 | 增加心跳超时检测，周期性清理无响应 session |
| **广播放大** | `Broadcast(msg)` 全量遍历 sessions → N² 复杂度，高并发时 CPU 飙升 | 延迟、CPU 占用 | 按 type 建立索引 Map（`map[type][]session`），避免全量遍历 |
| **消息大小** | 事务推送包含完整 `doOperations`，大块文档编辑时单条消息可达数 MB，超出 melody 的 8MB MaxMessageSize（上行），下行不受限但带宽压力大 | 推送失败、丢包 | 增量 diff 传输；大事务拆分为多次推送 |
| **重连数据同步** | `reloadSync` 仅执行 upsert/remove rootIDs，重连期间的非事务消息（statusbar/msg/progress）永久丢失 | UI 状态不一致 | 消息持久化 + 重放机制；按时间戳拉取近期事件队列 |
| **权限绕过** | WebSocket 鉴权仅在 HandleConnect 时执行，长连接期间权限变更（角色降级/注销）不会失效 | 已降级用户仍能接收推送 | 敏感推送增加二次校验；支持服务端主动踢人下线 |
| **鉴权重连** | reason 含 "unauthenticated" 时不重连，但 Cookie 过期后页面刷新才能重新登录 | 用户体验差 | 检测到 unauthenticated 时自动跳转 `/check-auth` 页面 |
| **ping/pong 缺失** | melody 有 Pong 回调但无应用层心跳；仅前端 `reconnectWebSocket` 全局函数手动发送 ping，无定时器自动执行 | 连接假死（NAT/防火墙丢弃空闲连接） | 应用层 30s 定时 ping/pong，超时主动关闭触发重连 |
| **发布服务竞态** | `ClosePublishServiceSessions` 中 `time.Sleep(500ms)` 是硬编码等待，高负载下消息可能未发出就关闭连接 | 部分客户端收不到关闭通知 | 使用 Write 回调 + WaitGroup 替代 sleep |
| **transactionsTimeout** | 256ms 防抖期间若 WebSocket 断开，本地事务未提交但可能已有推送基于旧状态 | 数据丢失 | 提交失败时事务回滚 + 本地重做队列 |
| **BroadcastByType 原子性** | 遍历 sync.Map 过程中若新 session 加入/离开，遍历快照可能不包含该 session | 消息漏发/重复 | 使用不可变快照 + 版本号确认 |

---

## 10. 进一步研究方向

### 10.1 架构优化方向

1. **连接多路复用（Multiplexing）**：
   - 单一 WebSocket 连接承载所有 type 的消息，通过消息头增加 `targetTypes: ["main", "protyle:xxx"]` 字段路由
   - 预期收益：连接数减少 70%+，降低服务端和浏览器资源消耗

2. **事件溯源（Event Sourcing）**：
   - 建立事件日志（Append-Only Log），每条消息分配单调递增序号
   - 重连时携带 `lastSeenSeq`，服务端按序号重放缺失事件
   - 解决重连期间事件丢失问题

3. **增量事务传输**：
   - 目前 transactions 推送包含完整 `doOperations` 数组
   - 研究基于 CRDT 的增量同步，或基于 OT（Operational Transform）的操作变换
   - 对多人实时协作至关重要

4. **消息优先级队列**：
   - 按重要性分级：`transactions`/`reloadui` 高优先级，`statusbar`/`progress` 低优先级
   - 高负载时低优先级消息可合并或丢弃

### 10.2 可靠性增强方向

5. **应用层心跳 + 连接健康度监控**：
   - 前后端双向 30s ping/pong（非 TCP level，而是业务层命令）
   - 连续 3 次无响应判定为死链，主动关闭 + 重连
   - 导出 Prometheus 指标：连接数、消息速率、平均延迟

6. **背压（Backpressure）机制**：
   - 前端处理缓慢（长时间事务渲染）时通知后端降低推送频率
   - 服务端发送缓冲区水位监控，超过阈值时触发 `BroadcastByType` 合并推送

7. **消息持久化 + 离线补偿**：
   - 关键事件（文档创建/删除/重命名）写入 SQLite 事件表
   - 浏览器恢复网络时按时间窗口补齐

### 10.3 安全增强方向

8. **消息签名/防篡改**：
   - 敏感命令（如 `closeBox`/`removeDoc`）增加 HMAC 签名校验
   - 防止 XSS 成功后伪造 WebSocket 消息执行破坏性操作

9. **会话绑定与踢人**：
   - WebSocket 绑定 JWT 的 sessionId，JWT 失效时主动断开对应连接
   - 用户修改密码/注销账号时遍历 sessions Map 踢掉所有关联连接

### 10.4 可观测性方向

10. **链路追踪**：
    - 请求入站时生成 TraceId，通过 HTTP Header → 事务 → WebSocket 消息 → 前端全链路透传
    - 便于排查 "用户 A 的修改为什么用户 B 没看到" 类问题

11. **前端 WebSocket DevTools**：
    - 开发模式下可视化展示：连接状态、消息队列、推送 cmd 统计、重连次数、平均延迟
    - 辅助插件开发者调试

---

## 11. 关键文件索引

| 模块 | 文件 | 关键职责 |
|------|------|----------|
| 后端服务入口 | [serve.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/server/serve.go#L697-L851) | WebSocket 服务器初始化、鉴权、消息接收、命令分发 |
| 后端连接管理 | [websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/websocket.go) | 会话注册/注销、多种广播策略、30+ 种推送函数 |
| 后端数据结构 | [result.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/util/result.go) | Result 消息结构、6 种 PushMode 常量 |
| 后端事务推送 | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/api/transaction.go) | 事务执行、pushTransactions 推送模式选择、rootIDs 提取 |
| 后端独立广播 | [broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/api/broadcast.go) | BroadcastChannels、SSE 支持、广播 API |
| 后端命令执行 | [cmd/cmd.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/cmd/cmd.go) | Cmd 接口定义、命令工厂、异步执行与 Recover |
| 后端 closews 命令 | [cmd/closews.go](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/kernel/cmd/closews.go) | 主动关闭 WebSocket 连接 |
| 前端 Model 基类 | [Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/Model.ts) | WebSocket 连接封装、重连逻辑、消息回调、send 方法 |
| 前端消息预处理 | [processMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/util/processMessage.ts) | msg/cmsg/cprogress/reloadui/closepublishpage 通用处理 |
| 前端桌面主连接 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/index.ts#L69-L214) | 桌面端 main 连接，20+ 种命令分发 |
| 前端移动端主连接 | [mobile/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/mobile/index.ts#L72-L82) | 移动端 main 连接，插件事件广播 |
| 前端编辑器连接 | [protyle/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/index.ts#L127-L350) | Protyle 连接，transactions/reload 等编辑器命令 |
| 前端文件树连接 | [layout/dock/Files.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/layout/dock/Files.ts#L44-L118) | filetree 连接，文档树/笔记本变更处理 |
| 前端类型定义 | [types/index.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/types/index.d.ts#L3-L3) | TWS 类型、IWebSocketData 接口定义 |
| 前端常量 | [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/constants.ts#L20-L20) | SIYUAN_APPID 随机生成逻辑 |
| 前端事务提交 | [protyle/wysiwyg/transaction.ts](file:///d:/fz/0601/solo-dogfeeding/code/297-siyuan/app/src/protyle/wysiwyg/transaction.ts) | 本地事务收集、防抖批量提交 |
