# SiYuan 内核 HTTP API 路由层代码结构分析

## 1. 概述

SiYuan（思源笔记）内核采用 Go 语言编写，基于 [Gin Web 框架](https://gin-gonic.com/) 构建 HTTP API 服务。本文档从代码实际实现出发，深入分析其路由层代码结构，**明确区分各机制的独立性与真实调用关系**，涵盖请求接收、参数验证、业务分发、结果封装、错误处理、权限控制、兼容性处理、长操作处理及前端交互规范等核心环节。

> **重要说明**：本文档所有结论均基于实际代码调用关系验证，不做功能层面的推断。

核心代码分布：
- 路由注册：[router.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/router.go)
- 服务器初始化：[serve.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/server/serve.go)
- 权限控制：[session.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/session.go)、[auth.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/auth.go)、[role.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/role.go)
- 结果封装：[result.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/result.go)
- 参数验证：[net.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/net.go)
- 前端推送：[websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/websocket.go)
- 任务队列：[queue.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/task/queue.go)
- 广播通道：[broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/broadcast.go)
- 前端调用：[fetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/util/fetch.ts)、[processMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/util/processMessage.ts)
- 属性视图模型：[av.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/av/av.go)、[layout.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/av/layout.go)

---

## 2. 整体架构流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        HTTP Request / WebSocket                              │
└──────────────────────────────────────────┬──────────────────────────────────┘
                                           │
                                           ▼
                        ┌──────────────────────────────────┐
                        │         Gin Server 初始化         │
                        │  Serve() @ kernel/server/serve.go │
                        └────────────────────┬─────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    ▼                      ▼                      ▼
        ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
        │  全局中间件链        │ │  静态资源路由        │ │  WebSocket/SSE       │
        │  (按执行顺序)        │ │  /assets, /stage,   │ │  /ws, /es/broadcast │
        │                     │ │  /widgets, /plugins │ │  /ws/broadcast      │
        │  1. ControlConc...  │ │  /appearance, etc.  │ └──────────┬──────────┘
        │  2. Timing          │ └──────────┬──────────┘            │
        │  3. Recover         │            │                       │
        │  4. corsMiddleware  │            │                       │
        │  5. jwtMiddleware   │            │                       │
        │  6. gzip            │            │                       │
        │  7. Sessions        │            │                       │
        └──────────┬──────────┘            │                       │
                   │                        │                       │
                   ▼                        ▼                       ▼
        ┌──────────────────────────────────────────────────────────────────┐
        │                      api.ServeAPI(ginServer)                      │
        │              路由注册入口 @ kernel/api/router.go                   │
        └─────────────────────────────────────┬────────────────────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    ▼                         ▼                         ▼
        ┌────────────────────┐     ┌────────────────────┐     ┌────────────────────┐
        │  无需鉴权接口       │     │  需要鉴权接口       │     │  写操作额外检查     │
        │  (少数公开端点)     │     │  model.CheckAuth   │     │  model.CheckAdmin..│
        │  /bootProgress     │     └─────────┬──────────┘     │  model.CheckRead...│
        │  /version          │               │                └─────────┬──────────┘
        │  /loginAuth        │               │                          │
        │  /getCaptcha       │               ▼                          ▼
        └────────────────────┘     ┌────────────────────┐     ┌────────────────────┐
                                   │   API Handler       │     │   API Handler       │
                                   │   (参数验证)         │     │   (执行业务逻辑)    │
                                   └─────────┬──────────┘     └─────────┬──────────┘
                                             │                          │
                                             ▼                          ▼
                                   ┌────────────────────┐     ┌────────────────────┐
                                   │ util.JsonArg()      │     │  model 层业务函数   │
                                   │ ParseJsonArg[T]()   │     │  e.g. InsertBlock   │
                                   │ InvalidIDPattern()  │     └─────────┬──────────┘
                                   └─────────┬──────────┘               │
                                             │                          │
                                             │            ┌─────────────┼─────────────┐
                                             │            ▼             ▼             ▼
                                             │  ┌──────────────┐ ┌─────────────┐ ┌──────────┐
                                             │  │ 同步返回      │ │ 异步任务队列 │ │ WS 推送  │
                                             │  │ c.JSON()     │ │ task.*      │ │ util.*   │
                                             │  └──────┬───────┘ └──────┬──────┘ └────┬─────┘
                                             │         │                │             │
                                             └─────────┴────────────────┴─────────────┘
                                                               │
                                                               ▼
                                                  ┌──────────────────────────┐
                                                  │     响应结果统一格式       │
                                                  │  {code, msg, data}       │
                                                  └──────────────────────────┘
```

---

## 3. 模块功能详细描述

### 3.1 请求接收层：服务器初始化与中间件链

服务器初始化入口位于 [Serve()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/server/serve.go#L133-L273) 函数，构建完整的 Gin Engine 实例并注册全局中间件。

#### 3.1.1 全局中间件执行顺序

| 序号 | 中间件 | 源码位置 | 功能说明 |
|------|--------|----------|----------|
| 1 | `ControlConcurrency` | [session.go#L456-L507](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/session.go#L456-L507) | 请求串行化控制，防止写操作并发冲突 |
| 2 | `Timing` | [session.go#L421-L444](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/session.go#L421-L444) | API 性能监控，超时时推送警告 |
| 3 | `Recover` | [session.go#L446-L449](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/session.go#L446-L449) | Panic 恢复，防止进程崩溃 |
| 4 | `corsMiddleware` | [serve.go#L974-L1015](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/server/serve.go#L974-L1015) | CORS 跨域支持，WebDAV/CalDAV 特殊处理 |
| 5 | `jwtMiddleware` | [serve.go#L1019-L1032](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/server/serve.go#L1019-L1032) | 解析 JWT Token，设置用户角色上下文 |
| 6 | `gzip.Gzip` | - | 响应压缩，排除音视频/PDF 等二进制格式 |
| 7 | `sessions.Sessions` | - | Cookie Session 管理 |

#### 3.1.2 并发控制机制 `ControlConcurrency`

该中间件实现了**路径级别的互斥锁**，关键逻辑：

```go
// 读操作前缀白名单 - 直接放行
if strings.HasPrefix(reqPath, "/stage/") ||
   strings.HasPrefix(reqPath, "/assets/") ||
   strings.HasPrefix(function, "get") ||      // 函数名前缀判断
   strings.HasPrefix(function, "list") ||
   strings.HasPrefix(function, "search") || ... {
    c.Next()
    return
}

// 写操作 - 按路径加锁串行执行
mutex := requesting[reqPath]
mutex.Lock()
defer mutex.Unlock()
c.Next()
```

这种设计保障了数据一致性，但对非前缀匹配的读操作可能存在过度串行化问题。

---

### 3.2 路由注册与业务分发

路由统一在 [ServeAPI()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/router.go#L25-L539) 函数中注册，采用**扁平结构**而非路由组。

#### 3.2.1 接口分类概览

| 模块前缀 | 接口数量 | 主要功能 |
|----------|----------|----------|
| `/api/system/*` | ~40 | 系统配置、版本、工作区、网络、日志 |
| `/api/block/*` | ~50+ | 块 CRUD、引用、标题操作、任务列表 |
| `/api/filetree/*` | ~30 | 文档树、文档创建/删除/移动/重命名 |
| `/api/notebook/*` | ~12 | 笔记本管理 |
| `/api/attr/*` | ~5 | 块属性读写 |
| `/api/search/*` | ~15 | 全文搜索、块搜索、资源搜索 |
| `/api/export/*` | ~30 | Markdown/HTML/PDF/Docx 等多格式导出 |
| `/api/av/*` | ~30+ | 属性视图（数据库）操作 |
| `/api/sync/*` | ~20 | 云同步配置与执行 |
| `/api/ui/*` | ~7 | UI 重载（Protyle/Tag/Filetree/主题等） |
| `/api/broadcast/*` | ~5 | 广播通道管理（WebSocket/SSE） |
| 其他 | ~50 | 标签、书签、历史、资源、 bazaar、AI 等 |

#### 3.2.2 路由处理链模式

每个路由注册遵循以下模式：

```go
ginServer.Handle("POST", "/api/block/insertBlock",
    model.CheckAuth,       // 中间件1: 鉴权
    model.CheckAdminRole,  // 中间件2: 管理员角色
    model.CheckReadonly,   // 中间件3: 只读模式检查
    insertBlock)           // Handler: 实际业务处理
```

Handler 签名统一为 `func(c *gin.Context)`，使用 Gin 的 `c.JSON()` 返回结果。

---

### 3.3 参数验证层

参数验证通过 `kernel/util/net.go` 中提供的工具函数完成，形成了三层验证机制。

#### 3.3.1 JSON 参数解析：`JsonArg()`

[JsonArg()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/net.go#L205-L221) 负责将请求体绑定为 `map[string]any`：

```go
func JsonArg(c *gin.Context, result *gulu.Result) (arg map[string]any, ok bool) {
    arg = map[string]any{}
    if err := c.ShouldBindJSON(&arg); err != nil {
        result.Code = -1
        result.Msg = fmt.Sprintf("Parses request [%s] failed: %s", c.Request.URL.Path, detail)
        return
    }
    ok = true
    return
}
```

#### 3.3.2 类型安全的泛型提取：`ParseJsonArg[T]()`

[ParseJsonArg[T]()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/net.go#L228-L285) 是 Go 1.18+ 泛型实现的参数提取器，支持：
- **required**：必填检查
- **rejectEmpty**：非空检查（自动 trim 字符串）
- **类型校验**：不匹配时返回友好的 JSON 类型名提示

批量提取使用 `ParseJsonArgs()` + `BindJsonArg[T]()`：

```go
var message, channelName string
if !util.ParseJsonArgs(arg, ret,
    util.BindJsonArg("message", &message, true, true),
    util.BindJsonArg("channel", &channelName, true, true),
) {
    return  // 任一参数失败时 ret 已填充错误信息
}
```

#### 3.3.3 业务级校验：`InvalidIDPattern()`

[InvalidIDPattern()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/net.go#L314-L322) 校验块 ID 是否符合 SiYuan 的 ID 格式规范（基于时间戳的 23 位 ID）。

---

### 3.4 结果封装与响应格式

#### 3.4.1 统一响应结构

SiYuan 使用 `gulu.Ret.NewResult()` 创建统一响应对象，实际结构为：

```go
type Result struct {
    Code int    `json:"code"`  // 0=成功, -1=错误, 1=需要验证码等特殊状态
    Msg  string `json:"msg"`   // 消息内容（错误时为错误信息）
    Data any    `json:"data"`  // 业务数据
}
```

WebSocket 推送使用扩展的 [Result](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/result.go#L35-L46) 结构：

```go
type Result struct {
    Cmd       string         `json:"cmd"`       // 推送命令名
    ReqId     float64        `json:"reqId"`     // 请求 ID（透传字段，用于前端竞态）
    AppId     string         `json:"app"`       // 应用 ID
    SessionId string         `json:"sid"`       // 会话 ID
    PushMode  PushMode       `json:"pushMode"`  // 推送模式（广播/单播等）
    Callback  any            `json:"callback"`
    Code      int            `json:"code"`
    Msg       string         `json:"msg"`
    Data      any            `json:"data"`
    Context   map[string]any `json:"context,omitempty"`
}
```

#### 3.4.2 推送模式枚举

| PushMode | 值 | 说明 |
|----------|-----|------|
| `PushModeBroadcast` | 0 | 所有应用所有会话广播 |
| `PushModeSingleSelf` | 1 | 自我应用会话单播 |
| `PushModeBroadcastExcludeSelf` | 2 | 非自我会话广播 |
| `PushModeBroadcastExcludeSelfApp` | 4 | 非自我应用所有会话广播 |
| `PushModeBroadcastApp` | 5 | 单个应用内所有会话广播 |
| `PushModeBroadcastMainExcludeSelfApp` | 6 | 非自我应用主会话广播 |

---

### 3.5 错误处理机制

#### 3.5.1 错误码约定

| Code | 含义 | 前端行为 |
|------|------|----------|
| `0` | 成功 | 正常处理回调 |
| `-1` | 错误 | 弹出 error 级别消息 |
| `-2` | 提示 | 弹出 info 级别消息 |
| `1` | 特殊状态（如需要验证码） | 渲染验证码 UI |
| `-401/-403/-404` | HTTP 状态码取负 | 鉴权失败时刷新页面/报错 |

#### 3.5.2 错误数据扩展

错误响应的 `Data` 字段通常包含：

```go
ret.Data = map[string]any{
    "closeTimeout": 7000,  // 消息自动关闭时间（毫秒），0=不自动关闭
    "id": msgId,           // 消息 ID，用于后续 hideMessage
}
```

---

### 3.6 权限控制体系

权限控制采用**多层次联合校验**，由 [CheckAuth()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/session.go#L207-L384) 中间件统一处理。

#### 3.6.1 用户角色定义

[role.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/role.go#L21-L32) 定义了 4 种角色：

| 角色 | 值 | 权限 |
|------|-----|------|
| `RoleAdministrator` | 0 | 管理员（完整读写权限） |
| `RoleEditor` | 1 | 编辑者（读写权限） |
| `RoleReader` | 2 | 读者（只读权限） |
| `RoleVisitor` | 3 | 匿名访问者 |

#### 3.6.2 鉴权方式优先级

`CheckAuth` 按以下顺序尝试鉴权，任一成功即放行：

1. **JWT Token 认证**（已在 `jwtMiddleware` 中解析）
   - Header: `X-Auth-Token: <JWT>`
   - 用于发布服务反向代理场景

2. **API Token 认证**
   - Header: `Authorization: Token <token>` / `Bearer <token>`
   - Query: `?token=<token>`
   - 通过 `Conf.Api.Token` 比对

3. **本地访问免密**（需同时满足）
   - 未设置 `AccessAuthCode`
   - 请求来源为 `127.0.0.1` / `localhost`
   - Host / Origin / X-Forwarded-Host 均为本地地址
   - 特殊跳过 Chrome 扩展来源（`chrome-extension://`）

4. **Cookie Session 认证**
   - 浏览器登录后写入 Cookie 的 `siyuan` session
   - Session 中存储 `AccessAuthCode` 与配置比对

5. **HTTP Basic Auth**
   - 用户名 = 工作区名称
   - 密码 = 访问授权码
   - 主要用于 WebDAV / CalDAV / CardDAV

---

### 3.7 长操作与异步任务处理

对于耗时操作（索引重建、OCR、同步等），SiYuan 采用**任务队列 + WebSocket 进度推送**模式。

#### 3.7.1 任务队列架构

[queue.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/task/queue.go) 实现了双模式任务队列：

```
┌─────────────────────────────────────────────────────────────┐
│                        Task Queue                            │
├──────────────────────┬──────────────────────────────────────┤
│   同步任务 (Async=false)  │   异步任务 (Async=true)          │
│   - 串行执行，按顺序出队   │   - 满足 Delay 后并发 goroutine  │
│   - 阻塞后续同步任务      │   - 不阻塞同步任务                │
│   - 互斥去重 (unique*)    │   - 互斥去重 (unique*)           │
└──────────────────────┴──────────────────────────────────────┘
```

**关键特性：**

1. **唯一性去重**：`uniqueActions` 列表中的任务（如 `DatabaseIndexFull`、`ReloadProtyle`）在队列中只能存在一个
2. **延迟执行**：通过 `Delay` 字段实现防抖（如 UI 刷新延迟 200ms 合并多次请求）
3. **超时控制**：每个任务有独立的 `Timeout`（默认 24 小时）
4. **退出保护**：`IsExiting` 标志为 true 时拒绝新任务

**唯一性去重实现** ([queue.go#L73-L78](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/task/queue.go#L73-L78))：

```go
if gulu.Str.Contains(action, uniqueActions) {
    if currentTasks := getCurrentTasks(); containTask(task, currentTasks) {
        return  // 队列中已存在相同 Action + Args 的任务，直接丢弃
    }
}
```

**注意**：`containTask()` 通过比较 `task.Action` + 逐个比较 `task.Args` 判断是否重复，与前端 `reqId` 机制无任何关联。

---

## 4. 兼容性处理机制分析

### 4.1 废弃接口兼容处理

SiYuan 采用**三层废弃兼容策略**，确保 API 演进过程中前端和插件的平滑过渡。

#### 4.1.1 路由层废弃标记

路由注册时通过 `deprecated` 中间件标记废弃端点，该中间件定义于 [router.go#L541-L544](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/router.go#L541-L544)：

```go
func deprecated(c *gin.Context) {
    logging.LogWarnf("[%s] is deprecated, it will be removed at [%s], visit [https://github.com/siyuan-note/siyuan/issues/15727] for details",
        c.Request.RequestURI, "2026-06-30")
}
```

**真实行为（代码验证）：**
- ✅ 仅执行 `logging.LogWarnf()` 记录 WARN 级别日志
- ❌ **不**调用 `c.Abort()`，不中断后续中间件和 Handler 执行
- ❌ **不**修改 `ret.Code` 或 `ret.Msg`，不改变响应结果
- ❌ **不**触发错误处理流程

> **结论**：`deprecated` 中间件与错误处理是**完全独立**的两套机制。废弃接口调用的日志告警仅记录在服务端日志中，不影响请求执行结果，也不会向前端推送任何消息。

**当前标记废弃的接口清单：**

| 废弃端点 | 替代端点 | 追踪链接 | 计划移除 |
|----------|----------|----------|----------|
| `/api/system/reloadUI` | `/api/ui/reloadUI` | [issue#15308](https://github.com/siyuan-note/siyuan/issues/15308) | 2026-06-30 |
| `/api/storage/setLocalStorage` | `/api/storage/setLocalStorageVal` | [issue#16664](https://github.com/siyuan-note/siyuan/issues/16664) | 2026-06-30 |
| `/api/filetree/refreshFiletree ` | `/api/system/rebuildDataIndex` | [issue#15663](https://github.com/siyuan-note/siyuan/issues/15663) | 2026-06-30 |
| `/api/attr/resetBlockAttrs` | `/api/attr/setBlockAttrs` | [PR#17027](https://github.com/siyuan-note/siyuan/pull/17027) | 2026-06-30 |
| `/api/av/searchAttributeViewNonRelationKey` | 无（直接废弃） | [issue#15727](https://github.com/siyuan-note/siyuan/issues/15727) | 2026-06-30 |

#### 4.1.2 参数级废弃兼容

部分接口在参数层面做了兼容处理，典型实现于 [attribute_view.go#L5007-L5012](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/attribute_view.go#L5007-L5012)：

```go
if _, ok := v["itemID"]; ok {
    itemID = v["itemID"].(string)
} else if _, ok := v["rowID"]; ok {
    // TODO 计划于 2026 年 6 月 30 日后删除
    itemID = v["rowID"].(string)
    logging.LogWarnf("[%s] parameter [%s] is deprecated, it will be removed at [%s], ...",
        "/api/av/batchSetAttributeViewBlockAttrs", "rowID", "2026-06-30")
}
```

**兼容模式：**
- **新参数优先**：`itemID` 存在时使用新参数
- **旧参数回退**：`itemID` 不存在时尝试旧参数 `rowID`
- **日志告警**：使用旧参数时记录 WARN 日志

> **注意**：参数级废弃同样只记录日志，不设置 `ret.Code = -1`，不进入错误处理流程。

#### 4.1.3 数据结构级废弃兼容：属性视图字段分析

此处需要特别澄清：**`View` 结构体与 `BaseLayout` 结构体的字段是两套独立定义，废弃标记仅存在于 `BaseLayout` 中**。

**1. View 结构体（实际使用的业务模型）**

定义于 [av.go#L204-L229](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/av/av.go#L204-L229)：

```go
type View struct {
    ID               string         `json:"id"`
    Icon             string         `json:"icon"`
    Name             string         `json:"name"`
    HideAttrViewName bool           `json:"hideAttrViewName"`
    Desc             string         `json:"desc"`
    Filters          []*ViewFilter  `json:"filters,omitempty"` // ✅ 正常使用，无废弃标记
    Sorts            []*ViewSort    `json:"sorts,omitempty"`   // ✅ 正常使用，无废弃标记
    PageSize         int            `json:"pageSize"`          // ✅ 正常使用，无废弃标记
    LayoutType       LayoutType     `json:"type"`
    Table            *LayoutTable   `json:"table,omitempty"`
    Gallery          *LayoutGallery `json:"gallery,omitempty"`
    Kanban           *LayoutKanban  `json:"kanban,omitempty"`
    ItemIDs          []string       `json:"itemIds,omitempty"`
    // ... 分组相关字段
}
```

`View.Filters` / `View.Sorts` / `View.PageSize` 在代码中被正常读写使用，例如：
- [attribute_view.go#L2548](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/attribute_view.go#L2548)：`view.Filters = append(view.Filters[:i], view.Filters[i+1:]...)`
- [attribute_view.go#L2914](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/attribute_view.go#L2914)：`view.Filters = append(view.Filters, &av.ViewFilter{...})`

**2. BaseLayout 结构体（嵌入到 LayoutTable/LayoutGallery/LayoutKanban）**

定义于 [layout.go#L27-L34](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/av/layout.go#L27-L34)：

```go
type BaseLayout struct {
    Spec int    `json:"spec"`
    ID   string `json:"id"`
    ShowIcon  bool `json:"showIcon"`
    WrapField bool `json:"wrapField"`

    // TODO 以下三个字段已经废弃，计划于 2026 年 6 月 30 日后删除
    //Deprecated
    Filters []*ViewFilter `json:"filters,omitempty"` // ⚠️ 仅此处标记废弃
    //Deprecated
    Sorts []*ViewSort `json:"sorts,omitempty"`        // ⚠️ 仅此处标记废弃
    //Deprecated
    PageSize int `json:"pageSize,omitempty"`          // ⚠️ 仅此处标记废弃
}
```

`LayoutTable` / `LayoutGallery` / `LayoutKanban` 通过嵌入 `*BaseLayout` 获得这些字段：

- [layout_table.go#L29-L31](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/av/layout_table.go#L29-L31)：`RowIDs []string` 同样被 `//Deprecated` 标记
- [layout_gallery.go#L36-L38](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/av/layout_gallery.go#L36-L38)：`CardIDs []string` 被 `//Deprecated` 标记

**真实状态（代码验证）：**

| 结构体 | Filters/Sorts/PageSize 状态 | 代码中是否被读取 | 是否存在迁移逻辑 |
|--------|-----------------------------|------------------|------------------|
| `View` | ✅ 正常字段，无废弃标记 | ✅ 大量读写操作 | ❌ 不需要迁移 |
| `BaseLayout` | ⚠️ 被 `//Deprecated` 标记 | ❌ 未发现实际读取 | ❌ 未发现迁移代码 |
| `BaseInstance` | ✅ 正常字段，无废弃标记 | ✅ 被方法访问 | ❌ 不需要迁移 |

> **修正结论**：
> 1. 废弃标记仅存在于 `BaseLayout` 结构体的 3 个字段 + `LayoutTable.RowIDs` + `LayoutGallery.CardIDs`
> 2. `View` 结构体中的同名字段是正常的业务字段，**未被废弃**
> 3. 代码中**未发现**从废弃字段到新字段（如 `filterGroups`/`sortGroups`）的迁移逻辑
> 4. 废弃字段的作用是**保留 JSON tag 以解析历史数据**，实际业务逻辑使用的是 `View` 结构体中的同名字段

---

### 4.2 本地存储历史数据兼容机制

本地存储（Local Storage）是前端状态持久化的核心途径，SiYuan 实现了完整的历史数据兼容体系。

#### 4.2.1 后端存储结构

后端存储管理于 [model/storage.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/storage.go)，核心数据文件包括：

| 存储文件 | 内容 | 访问 API |
|----------|------|----------|
| `storage/local.json` | 全局本地存储（搜索配置、UI 布局、导出设置等） | `getLocalStorage`/`setLocalStorageVal` |
| `storage/recent-doc.json` | 最近文档列表（浏览/打开/关闭时间） | `getRecentDocs`/`updateRecentDoc*` |
| `storage/outline.json` | 大纲展开状态（按文档 ID 存储） | `getOutlineStorage`/`setOutlineStorage` |
| `storage/criteria.json` | 高级搜索条件保存 | `getCriteria`/`setCriterion` |

**后端容错处理** ([storage.go#L692-L711](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/storage.go#L692-L711))：

```go
// 当 local.json 损坏时，清空文件避免无法进入主界面
// https://github.com/siyuan-note/siyuan/issues/7911
func getLocalStorage() (ret map[string]any) {
    ret = map[string]any{}
    lsPath := filepath.Join(util.DataDir, "storage/local.json")
    if !filelock.IsExist(lsPath) { return }
    data, err := filelock.ReadFile(lsPath)
    if err != nil { return }
    if err = gulu.JSON.UnmarshalJSON(data, &ret); err != nil {
        logging.LogErrorf("unmarshal storage [local] failed: %s", err)
        return  // 解析失败返回空 map，由前端填充默认值
    }
    return
}
```

#### 4.2.2 前端数据迁移

前端在 [compatibility.ts#getLocalStorage()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/protyle/util/compatibility.ts#L425-L566) 中实现了完整的数据迁移逻辑：

**迁移流程：**

```
1. 调用 /api/storage/getLocalStorage 获取原始数据
2. 构建 defaultStorage 默认配置（27 个存储键的默认值）
3. 对每个键执行兼容处理：
   ├─ 若值为 string → 尝试 JSON.parse 解析
   ├─ 若解析成功且为 number → 直接赋值
   ├─ 若解析成功且为 object → Object.assign 与默认值合并
   ├─ 若解析失败或值为 undefined → 使用默认值
4. 针对特定键做额外兼容（如 replaceTypes）
```

**典型兼容代码：**

```typescript
[Constants.LOCAL_EXPORTIMG, Constants.LOCAL_SEARCHKEYS, ...].forEach((key) => {
    if (typeof response.data[key] === "string") {
        try {
            const parseData = JSON.parse(response.data[key]);
            if (typeof parseData === "number") {
                // https://github.com/siyuan-note/siyuan/issues/8852
                // Object.assign 会导致 number to Number，特殊处理
                window.siyuan.storage[key] = parseData;
            } else {
                window.siyuan.storage[key] = Object.assign(defaultStorage[key], parseData);
            }
        } catch (e) {
            window.siyuan.storage[key] = defaultStorage[key];  // 解析失败用默认值
        }
    } else if (typeof response.data[key] === "undefined") {
        window.siyuan.storage[key] = defaultStorage[key];      // 新键用默认值
    }
});
```

#### 4.2.3 存储变更与 WebSocket 推送

`setLocalStorageVal` 和 `removeLocalStorageVals` 在完成存储操作后会通过 WebSocket 推送事件通知其他窗口同步，这是一个**真实存在的调用关系**：

**后端推送** ([storage.go#L158-L162](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/storage.go#L158-L162))：

```go
app := arg["app"].(string)
evt := util.NewCmdResult("setLocalStorageVal", 0, util.PushModeBroadcastMainExcludeSelfApp)
evt.AppId = app
evt.Data = map[string]any{"key": key, "val": val}
util.PushEvent(evt)  // ← 调用推送通道
```

**前端接收** ([index.ts#L139-L140](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/index.ts#L139-L140))：

```typescript
case "setLocalStorageVal":
    window.siyuan.storage[data.data.key] = data.data.val;
    break;
```

> **结论**：本地存储变更与 WebSocket 推送之间存在**明确的调用关系**（存储操作完成 → 构造事件 → PushEvent）。这是多窗口数据同步机制，与"兼容性处理"本身无直接关系，是存储 API 的标准跨端同步行为。

---

## 5. 请求竞态限制机制

在高频操作场景下（搜索、关系图渲染），SiYuan 通过 `reqId` 时间戳机制防止旧响应覆盖新结果。

### 5.1 前端请求侧实现

竞态处理实现于 [fetch.ts#L18-L30](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/util/fetch.ts#L18-L30)：

```typescript
if (["/api/search/searchRefBlock", "/api/graph/getGraph", "/api/graph/getLocalGraph",
    "/api/block/getRecentUpdatedBlocks", "/api/search/fullTextSearchBlock"].includes(url)) {
    window.siyuan.reqIds[url] = new Date().getTime();  // 记录最新请求时间戳
    if (data.type === "local" && url === "/api/graph/getLocalGraph") {
        // 特例：打开文档A的关系图后刷新，不添加 reqId 避免无法渲染
    } else {
        data.reqId = window.siyuan.reqIds[url];         // 请求携带 reqId
    }
}
// 并发导出后端接受顺序不一致，同样需要 reqId
if (url === "/api/transactions") {
    data.reqId = new Date().getTime();
}
```

### 5.2 后端对 reqId 的处理：**仅透传**

以 `searchRefBlock` 为例，[search.go#L356-L385](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/search.go#L356-L385)：

```go
reqId := arg["reqId"]                           // 1. 从请求参数中取出
ret.Data = map[string]any{"reqId": reqId}       // 2. 先暂存到返回数据
// ... 执行业务逻辑 SearchRefBlock ...
ret.Data = map[string]any{
    "blocks": blocks,
    "newDoc": newDoc,
    "k":      util.EscapeHTML(keyword),
    "reqId":  arg["reqId"],                     // 3. 业务结果中原样放回去
}
```

**后端对 reqId 的行为（代码验证）：**

| 检查项 | 结果 |
|--------|------|
| 用 reqId 做逻辑判断 | ❌ 不存在 |
| 用 reqId 查询/写入任务队列 | ❌ 不存在 |
| 用 reqId 做缓存 key | ❌ 不存在 |
| 用 reqId 做请求去重 | ❌ 不存在 |
| 从 arg 读取后原样塞入 ret.Data | ✅ 所有相关接口均如此 |

在 [task/queue.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/task/queue.go) 中 Grep `reqId` / `ReqId` 结果为 **0 匹配**。

### 5.3 前端响应侧处理

响应处理于 [fetch.ts#L82-L87](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/util/fetch.ts#L82-L87)：

```typescript
if (["/api/search/searchRefBlock", "/api/graph/getGraph", "/api/graph/getLocalGraph",
    "/api/block/getRecentUpdatedBlocks", "/api/search/fullTextSearchBlock"].includes(url)) {
    if (response.data.reqId && window.siyuan.reqIds[url] 
        && window.siyuan.reqIds[url] > response.data.reqId) {
        return;  // 本地记录的 reqId > 响应 reqId，说明是过期响应，丢弃
    }
}
```

### 5.4 请求竞态与任务队列的关系：**完全独立**

| 对比维度 | 前端 reqId 竞态机制 | 后端任务队列去重机制 |
|----------|---------------------|----------------------|
| 生效位置 | 浏览器端（TypeScript） | 内核端（Go） |
| 作用范围 | fetchPost → 响应回调之间 | task.AppendTask → task.ExecTask 之间 |
| 判断依据 | `window.siyuan.reqIds[url] > response.data.reqId` 时间戳比较 | `containTask()` 比较 Action + Args 是否完全一致 |
| 效果 | 丢弃过期响应，不影响后端执行 | 拒绝重复入队，后端只执行一次 |
| 代码依赖 | 无外部依赖，纯前端逻辑 | 依赖 uniqueActions 列表 + reflect.DeepEqual |
| 接口覆盖 | 5 个搜索/关系图接口 + transactions | 18 种 uniqueActions（索引/UI 刷新/OCR 等） |
| 存在调用关系 | ⚠️ **无任何调用关系** | |

> **修正结论**：前端 `reqId` 竞态机制与后端任务队列去重是**两套完全独立的机制**，运行在不同层级、不同进程中，没有任何代码层面的调用关系。两者在效果上对"重复请求"场景有互补作用，但这是架构设计上的巧合，不是有意识的"双重保护"协作。

---

## 6. 错误消息展示与推送消息处理

SiYuan 构建了**双通道错误与消息系统**：HTTP 响应携带错误码 + WebSocket 主动推送消息，两者共享前端 `processMessage()` 入口。

### 6.1 统一响应结构与错误码

后端使用 `{code, msg, data}` 统一结构，错误码约定于 [result.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/result.go)：

| code | 类型 | 前端处理 | 示例 |
|------|------|----------|------|
| `0` | 成功 | 执行回调 | 操作正常完成 |
| `-1` | 错误 | `showMessage(..., "error")` | 参数错误、权限不足 |
| `-2` | 提示 | `showMessage(..., "info")` | 操作提示、确认信息 |
| `-401` | 未授权 | 3秒后自动刷新页面 | Token 过期 |
| `-403` | 禁止访问 | 显示错误消息 | 角色权限不足 |
| `-404` | 资源不存在 | 显示错误消息 | 文档已删除 |

**后端错误设置示例** [workspace.go#L48-L51](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/workspace.go#L48-L51)：

```go
if util.IsPartitionRootPath(path) {
    ret.Code = -1
    ret.Msg = model.Conf.Language(273)          // 多语言消息
    ret.Data = map[string]any{"closeTimeout": 7000}  // 消息展示时长
    return
}
```

### 6.2 前端消息处理：统一入口 `processMessage()`

[processMessage()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/util/processMessage.ts#L10-L77) 是所有消息的统一入口，被两处调用：

**调用入口 1：HTTP 响应**
[fetch.ts#L88-L91](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/util/fetch.ts#L88-L91)：
```typescript
if (typeof response === "object" && typeof response.msg === "string" && typeof response.code === "number") {
    if (processMessage(response) && cb) {
        cb(response);
    }
}
```

**调用入口 2：WebSocket 消息**
[Model.ts#L57-L62](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/layout/Model.ts#L57-L62)：
```typescript
ws.onmessage = (event) => {
    if (options.msgCallback && window.siyuan.config) {
        const data = processMessage(JSON.parse(event.data));
        options.msgCallback.call(this, data);
    }
};
```

**processMessage 内部处理流程：**

```
Response / WebSocket Message
        │
        ▼
    processMessage()
        │
        ├─ cmd == "msg" → showMessage() 弹出普通消息
        │   ├─ 支持消息 ID 追踪
        │   ├─ 支持自定义 closeTimeout
        │   └─ 支持嵌入按钮（如 Microsoft Defender 排除）
        │
        ├─ cmd == "cmsg" → hideMessage() 关闭指定消息
        │
        ├─ cmd == "cprogress" → 移除进度条遮罩
        │
        ├─ cmd == "reloadui" → 保存布局后刷新页面
        │   └─ resetScroll=true 时清空滚动位置存储
        │
        ├─ cmd == "closepublishpage" → 发布服务关闭处理
        │
        └─ code < 0 → 通用错误/提示处理
            ├─ code == -1 → error 级别消息
            └─ code == -2 → info 级别消息
```

> **结论**：HTTP 错误响应与 WebSocket 推送消息**共享同一个 `processMessage()` 处理入口**，这是真实存在的调用关系。这确保了无论错误通过哪个通道传递，用户看到的 UI 表现完全一致。

### 6.3 WebSocket 推送通道

后端通过 [websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/util/websocket.go) 提供多种推送函数：

| 函数 | 用途 | 推送 cmd |
|------|------|----------|
| `PushMsg(msg, timeout)` | 普通消息推送 | `msg` |
| `PushErrMsg(msg, timeout)` | 错误消息推送 | `msg` (code=-1) |
| `PushProgress(code, current, total, msg)` | 进度推送 | `progress` |
| `PushEndlessProgress(msg)` | 无明确进度推送 | `progress` (code=1) |
| `ClearPushProgress(total)` | 清除进度 | `progress` (code=2) |
| `PushClearMsg(msgId)` | 清除指定消息 | `cmsg` |
| `PushClearProgress()` | 清除进度遮罩 | `cprogress` |
| `PushStatusBar(msg)` | 状态栏消息 | `statusbar` |
| `ReloadUI()` | 重载 UI | `reloadui` |

---

## 7. 各机制独立性与调用关系汇总

### 7.1 独立关系矩阵

下表明确标记各机制之间是否存在直接的代码调用关系：

| 机制 A ↓ / 机制 B → | 废弃兼容 | 本地存储兼容 | 请求竞态 (reqId) | 任务队列去重 | 错误处理 | WebSocket 推送 | processMessage |
|---|---|---|---|---|---|---|---|
| **废弃兼容** | - | ❌ 独立 | ❌ 独立 | ❌ 独立 | ❌ 独立¹ | ❌ 独立 | ❌ 独立 |
| **本地存储兼容** | ❌ 独立 | - | ❌ 独立 | ❌ 独立 | ❌ 独立 | ✅ 调用² | ❌ 间接³ |
| **请求竞态 (reqId)** | ❌ 独立 | ❌ 独立 | - | ❌ 独立⁴ | ❌ 独立 | ❌ 独立 | ❌ 独立 |
| **任务队列去重** | ❌ 独立 | ❌ 独立 | ❌ 独立⁴ | - | ❌ 独立 | ✅ 调用⁵ | ✅ 间接⁶ |
| **错误处理 (ret.Code=-1)** | ❌ 独立¹ | ❌ 独立 | ❌ 独立 | ❌ 独立 | - | ❌ 独立⁷ | ✅ 调用⁸ |

**注释说明：**

1. **废弃兼容 vs 错误处理**：`deprecated()` 中间件只执行 `logging.LogWarnf()`，不修改 `ret.Code`、不调用 `c.Abort()`、不改变响应结果。两者是完全独立的机制。

2. **本地存储兼容 → WebSocket 推送**：`setLocalStorageVal` / `removeLocalStorageVals` 完成存储后调用 `util.PushEvent()` 推送同步事件，这是真实的直接调用关系。

3. **本地存储兼容 → processMessage（间接）**：推送事件经 WebSocket 到达前端后，会进入 `processMessage()`，但不是通过 `cmd=msg`/`code<0` 分支，而是通过 `case "setLocalStorageVal"` 自定义分支处理。

4. **请求竞态 vs 任务队列去重**：无任何调用关系。前端 `reqId` 在浏览器中比较时间戳，后端 `uniqueActions` 在 Go 中比较 Action+Args，两者互不知晓。

5. **任务队列 → WebSocket 推送**：`StatusJob()` 调用 `util.PushBackgroundTask()` 推送任务状态，部分任务 handler 内部也会调用 Push* 函数。

6. **任务队列 → processMessage（间接）**：任务状态推送 cmd = `backgroundtask`，经 processMessage() 后转交给 msgCallback 处理。

7. **错误处理 → WebSocket 推送**：绝大多数情况下，错误处理通过 HTTP 响应的 `code=-1` 返回给前端，不主动触发 WebSocket 推送。但业务层可以同时设置 `ret.Code=-1` 并调用 `PushErrMsg()`，这是业务选择而非机制绑定。

8. **错误处理 → processMessage**：HTTP 响应到达前端后，`fetchPost()` 调用 `processMessage()` 处理 `code<0` 的情况，这是直接调用关系。

### 7.2 真实调用链路图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        机制调用关系全景图（仅包含代码证实的真实关系）                     │
└─────────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────┐            ┌──────────────────┐
  │  废弃兼容层   │            │  请求竞态 (reqId)  │
  │ deprecated() │            │   fetch.ts 前端    │
  │  仅打日志     │            │   时间戳比较       │
  └──────┬───────┘            └────────┬──────────┘
         │ 完全独立                    │ 完全独立
         ▼                             ▼
  (不影响任何其他机制)           (不影响任何后端机制)


  ┌──────────────────────────────┐
  │      HTTP API Handler         │
  │  ┌─────────────────────────┐  │
  │  │ JsonArg + 参数校验       │  │
  │  └───────────┬─────────────┘  │
  │              │                │
  │    ┌─────────┴──────────┐     │
  │    ▼                    ▼     │
  │  成功路径              失败路径 │
  │  ret.Data = 结果       ret.Code=-1 │
  │  ret.Code = 0          ret.Msg = 错误│
  │    │                    │     │
  └────┼────────────────────┼─────┘
       │                    │
       │ c.JSON() 响应      │ c.JSON() 响应
       ▼                    ▼
  前端 fetchPost()     前端 fetchPost()
       │                    │
       └──────────┬─────────┘
                  ▼
         processMessage()      ◄────── 直接调用关系 ✅
         (code<0 分支处理)


  ┌──────────────────────────────┐
  │     setLocalStorageVal       │
  │     removeLocalStorageVals   │
  │  ┌────────────────────────┐  │
  │  │ 1. 写入 local.json     │  │
  │  └────────────┬───────────┘  │
  │               ▼              │
  │  ┌────────────────────────┐  │
  │  │ 2. PushEvent() 推送    │──┼── 直接调用关系 ✅
  │  └────────────────────────┘  │
  └──────────────────────────────┘
               │
               ▼ WebSocket
     前端 ws.onmessage
               │
               ▼
         processMessage()      ◄────── 间接关系 (自定义 case)
         (setLocalStorageVal 分支)


  ┌──────────────────────────────┐
  │        任务队列 task.*        │
  │  AppendTask() 入队            │
  │  ├─ uniqueActions 去重       │  ← 与 reqId 无任何关系
  │  └─ ExecTaskJob() 执行       │
  │       └─ 部分任务内调用 Push* │── 直接调用关系 ✅
  └──────────────────────────────┘
               │
               ▼
         util.Push* 函数
               │
               ▼ WebSocket
     前端 ws.onmessage
               │
               ▼
         processMessage()      ◄────── 间接关系 (progress/msg 等分支)
```

---

## 8. 修正后的关键结论

基于代码实际实现，对之前文档中的三处结论修正如下：

### 修正 1：属性视图废弃字段的实际范围

**❌ 原结论**："属性视图中 Filters/Sorts/PageSize 字段已废弃，正在迁移到 filterGroups/sortGroups/pagination"

**✅ 修正结论**：
- 废弃标记**仅存在于 `BaseLayout` 结构体**（`layout.go#L27-L34`）的 3 个字段
- **`View` 结构体**（`av.go#L204-L229`）中的同名字段 `Filters`/`Sorts`/`PageSize` 是**正常业务字段**，未被废弃，在代码中被大量读写
- 未发现从废弃字段到新字段的迁移逻辑
- `LayoutTable.RowIDs` / `LayoutGallery.CardIDs` 同样仅作历史 JSON 解析保留用

### 修正 2：请求竞态与任务队列去重的关系

**❌ 原结论**："请求竞态机制与任务队列的防抖去重形成双重保护，两者相互协作"

**✅ 修正结论**：
- 两套机制**完全独立**，无任何代码层面的调用关系
- 前端 `reqId`：运行在浏览器进程中，通过时间戳比较决定是否丢弃响应，仅覆盖 5+1 个接口
- 后端 `uniqueActions`：运行在内核进程中，通过 Action+Args 比较决定是否入队，覆盖 18 种任务类型
- 两者在"防止重复操作"场景下有效果上的互补，但这是架构设计上的分层隔离，不是有意识的协作

### 修正 3：废弃兼容与错误处理的边界

**❌ 原结论**："废弃接口的处理与错误处理机制共享相同的日志基础设施，存在关联"

**✅ 修正结论**：
- 两套机制**完全独立**
- `deprecated()` 中间件**只做一件事**：`logging.LogWarnf()` 输出 WARN 日志，不中断请求、不修改响应、不向前端传递任何信号
- 错误处理通过设置 `ret.Code = -1` + `ret.Msg` 生效，与废弃标记互不干扰
- 两者唯一共享的是 Go `logging` 包这个基础设施（与 `fmt.Println` 类似的通用库），不存在业务层面的关联

---

## 9. 潜在风险与改进建议

| # | 风险点 | 严重程度 | 描述与代码位置 |
|---|--------|----------|---------------|
| R1 | **并发控制粒度过粗** | 中高 | `ControlConcurrency` 按请求路径加锁，不同文档的写操作也会串行，可能成为性能瓶颈。 [session.go#L456-L507](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/session.go#L456-L507) |
| R2 | **参数类型断言 Panic 风险** | 中 | 大量 Handler 使用 `arg["key"].(string)` 直接类型断言，若参数缺失或类型错误会导致 Panic。应推广使用 `ParseJsonArg[T]()`。 |
| R3 | **reqId 覆盖接口有限** | 中 | 仅 6 个高频接口启用竞态保护，其他高频操作（如 `getDoc`、搜索建议）仍存在旧响应覆盖风险。 [fetch.ts#L18-L30](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/util/fetch.ts#L18-L30) |
| R4 | **废弃接口依赖人工检查** | 低 | 所有废弃标记依赖 TODO 注释，无自动化到期提醒，临近 2026-06-30 可能遗漏删除。 [router.go#L541-L544](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/router.go#L541-L544) |
| R5 | **HTTP 始终返回 200** | 低 | 所有响应始终返回 200 HTTP 状态码，无法通过网关/负载均衡做错误率统计。 |
| R6 | **BaseLayout 废弃字段无迁移校验** | 低 | 虽然字段被标记废弃，但无运行时检测历史数据是否仍在使用旧字段，移除时可能有兼容性风险。 [layout.go#L27-L34](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/av/layout.go#L27-L34) |

---

## 10. 追踪问题清单

以下是代码中标记的待追踪事项（来源于 TODO 注释）：

| 追踪链接 | 涉及端点/字段 | 说明 | 计划移除日期 |
|----------|--------------|------|--------------|
| [issue#15308](https://github.com/siyuan-note/siyuan/issues/15308#issuecomment-3077675356) | `/api/system/reloadUI` | 迁移至 `/api/ui/reloadUI` | 2026-06-30 |
| [issue#16664](https://github.com/siyuan-note/siyuan/issues/16664#issuecomment-3694774305) | `/api/storage/setLocalStorage` | 迁移至 `/api/storage/setLocalStorageVal` | 2026-06-30 |
| [issue#15663](https://github.com/siyuan-note/siyuan/issues/15663#issuecomment-3219296189) | `/api/filetree/refreshFiletree` | 迁移至 `/api/system/rebuildDataIndex` | 2026-06-30 |
| [PR#17027](https://github.com/siyuan-note/siyuan/pull/17027) | `/api/attr/resetBlockAttrs` | 迁移至 `/api/attr/setBlockAttrs` | 2026-06-30 |
| [issue#15727](https://github.com/siyuan-note/siyuan/issues/15727) | `/api/av/searchAttributeViewNonRelationKey` | 直接废弃 | 2026-06-30 |
| [issue#15162](https://github.com/siyuan-note/siyuan/issues/15162) | `BaseLayout.Filters/Sorts/PageSize` | 历史数据兼容字段 | 2026-06-30 |
| [issue#15194](https://github.com/siyuan-note/siyuan/issues/15194) | `LayoutTable.RowIDs` / `LayoutGallery.CardIDs` | 历史数据兼容字段 | 2026-06-30 |
| [issue#15708](https://github.com/siyuan-note/siyuan/issues/15708#issuecomment-3239694546) | 参数 `rowID` → `itemID` | 属性视图参数重命名 | 2026-06-30 |

---

## 11. 总结

SiYuan 内核 HTTP API 路由层体现了**清晰的分层隔离设计**，各机制之间的独立性大于关联性：

### 独立性特征（已代码验证）

1. **废弃兼容层完全隔离**：仅通过日志记录提醒开发者，不影响请求执行结果、不触发错误处理、不推送前端消息，确保移除 deprecated 中间件不会导致任何功能变化。

2. **请求竞态与后端无耦合**：`reqId` 机制完全在前端实现，后端的唯一参与是"接过来、传回去"的透明透传。这种设计允许前端单独调整竞态策略而无需修改内核代码。

3. **任务队列与 HTTP 层解耦**：任务去重基于 Action+Args 语义比较，不依赖请求标识。即使前端发出带不同 reqId 的重复请求，后端基于 uniqueActions 规则仍可正确去重。

### 存在调用关系的机制（已代码验证）

1. **错误处理 → processMessage**：通过 HTTP 响应的 `code<0` 字段绑定。这是前端与后端约定的契约，确保所有业务错误有一致的 UI 表现。

2. **存储变更 → WebSocket 推送**：`setLocalStorageVal` 等 API 在完成写入后主动推送，实现多窗口状态同步。这是**业务逻辑的一部分**，不是兼容性机制的特性。

3. **任务执行 → WebSocket 推送**：长任务通过推送 cmd=progress/backgroundtask 反馈状态，保持 UI 响应性。

### 设计哲学

SiYuan 的兼容性和前端交互设计遵循以下原则：

| 原则 | 体现 |
|------|------|
| **最小影响原则** | 废弃标记只打日志，不改变任何行为；升级到新版本不会因 API 废弃导致功能中断 |
| **分层独立原则** | 前端竞态、后端去重、错误处理各自运行在独立层级，便于单独演进和测试 |
| **统一入口原则** | processMessage() 作为 HTTP+WebSocket 的共同消息处理入口，保证用户体验一致 |
| **向后兼容原则** | 废弃字段保留 JSON tag 以解析历史数据，迁移时间窗口统一设定为 1 年（2026-06-30） |

这种架构在桌面应用场景下表现优秀，通过明确的机制边界降低了系统复杂度，也为后续演进提供了清晰的修改面。
