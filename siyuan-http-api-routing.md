# SiYuan 内核 HTTP API 路由层代码结构分析

## 1. 概述

SiYuan（思源笔记）内核采用 Go 语言编写，基于 [Gin Web 框架](https://gin-gonic.com/) 构建 HTTP API 服务。本文档深入分析其路由层代码结构，涵盖请求接收、参数验证、业务分发、结果封装、错误处理、权限控制、兼容性处理、长操作处理及前端交互规范等核心环节。

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

#### 3.3.4 典型 Handler 参数验证流程

```go
func transferBlockRef(c *gin.Context) {
    ret := gulu.Ret.NewResult()
    defer c.JSON(http.StatusOK, ret)

    arg, ok := util.JsonArg(c, ret)  // 1. 解析 JSON
    if !ok { return }

    fromID := arg["fromID"].(string)
    if util.InvalidIDPattern(fromID, ret) { return }  // 2. ID 格式校验
    // ... 继续处理
}
```

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
    ReqId     float64        `json:"reqId"`     // 请求 ID（用于竞态处理）
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

#### 3.5.3 Panic 恢复

全局 `Recover` 中间件通过 `logging.Recover()` 捕获 panic 并记录日志，避免进程崩溃。

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

#### 3.6.3 只读模式检查

[CheckReadonly()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/session.go#L195-L205) 在以下情况下阻止写操作：
- 全局 `util.ReadOnly` 标志为 true
- 当前用户角色为 `RoleReader` 或 `RoleVisitor`

---

### 3.7 兼容性处理（API 废弃机制）

#### 3.7.1 废弃 API 标记模式

路由注册时使用 `deprecated` 中间件标记废弃端点：

```go
ginServer.Handle("POST", "/api/system/reloadUI",
    model.CheckAuth, model.CheckAdminRole, model.CheckReadonly,
    reloadUI,
    deprecated)  // 废弃标记中间件
// TODO 请使用 /api/ui/reloadUI，该端点计划于 2026 年 6 月 30 日后删除
// https://github.com/siyuan-note/siyuan/issues/15308#issuecomment-3077675356
```

[deprecated()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/router.go#L541-L544) 仅记录 WARN 级别日志，不阻止请求执行，确保平滑过渡。

#### 3.7.2 前端兼容层

前端 [compatibility.ts](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/protyle/util/compatibility.ts) 提供运行时兼容处理，如：
- 旧 API 返回数据结构适配
- 块格式版本迁移
- 存储键名迁移（`setStorageVal`）

---

### 3.8 长操作与异步任务处理

对于耗时操作（索引重建、OCR、同步等），SiYuan 采用**任务队列 + WebSocket 进度推送**模式。

#### 3.8.1 任务队列架构

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

**常用任务类型：**

| Action | 模式 | 说明 |
|--------|------|------|
| `DatabaseIndexFull` | 同步/唯一 | 数据库全量重建索引 |
| `ReloadProtyle` | 异步/200ms延迟/唯一 | 编辑器刷新 |
| `ReloadFiletree` | 异步/200ms延迟/唯一 | 文件树刷新 |
| `OCRImage` | 同步/唯一 | 图片 OCR |
| `PushMsg` | 异步 | 消息推送 |

#### 3.8.2 进度推送机制

长时间操作通过 WebSocket 推送进度至前端：

```go
// 有明确进度
util.PushProgress(util.PushProgressCodeProgressed, current, total, "正在处理...")

// 无明确进度（无限转圈）
util.PushEndlessProgress("正在索引，请稍候...")

// 完成/清除
util.ClearPushProgress(100)
util.PushClearProgress()
```

前端通过监听 `progress` cmd 更新进度条遮罩层。

---

### 3.9 前端交互规范

#### 3.9.1 HTTP API 调用规范

前端通过 [fetchPost()](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/app/src/util/fetch.ts#L8-L117) 统一调用内核 API：

**调用约定：**
- 所有 API 请求方法为 **POST**（除少数静态资源 GET）
- 请求体为 JSON（`Content-Type: application/json`）或 FormData（文件上传）
- 响应始终返回 200 HTTP 状态码，通过 `code` 字段区分业务状态
- 401 时前端自动刷新页面跳转登录

**请求竞态处理：**
```typescript
// 搜索等高频操作添加 reqId 防止旧响应覆盖新结果
if (["/api/search/searchRefBlock", "/api/graph/getGraph", ...].includes(url)) {
    window.siyuan.reqIds[url] = new Date().getTime();
    data.reqId = window.siyuan.reqIds[url];
}
// 响应时比较 reqId，丢弃过期响应
if (response.data.reqId && window.siyuan.reqIds[url] 
    && window.siyuan.reqIds[url] > response.data.reqId) {
    return;  // 丢弃
}
```

#### 3.9.2 WebSocket 实时通信

内核与前端通过 `/ws` 建立长连接，会话按 `app`（应用实例）+ `id`（窗口/标签页）+ `type`（会话类型）分类。

**主要会话类型：**

| type | 用途 | 推送 cmd 示例 |
|------|------|---------------|
| `main` | 主窗口 | `msg`, `progress`, `reloadui`, `statusbar` |
| `protyle` | 编辑器实例 | `reload`, `refreshAttributeView` |
| `filetree` | 文件树面板 | `reloadFiletree`, `reloadDocInfo` |
| `auth` | 授权页（特殊免操作） | - |

#### 3.9.3 广播通道机制

[broadcast.go](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/broadcast.go) 提供插件/扩展用的通用广播系统：

- **WebSocket 接入**：`GET /ws/broadcast?channel=<name>`
- **SSE 接入**：`GET /es/broadcast/subscribe?channel=<name>&retry=1000`
- **发送消息**：`POST /api/broadcast/postMessage`、`POST /api/broadcast/publish`（支持 multipart 二进制）

---

## 4. 各环节关联关系图

```
           ┌───────────────────────────────────────────────────────┐
           │                    前端 (TypeScript)                   │
           │  fetchPost() / fetchSyncPost() / WebSocket / SSE      │
           └───────────────────────────┬───────────────────────────┘
                                       │
               ┌───────────────────────┼───────────────────────┐
               ▼                       ▼                       ▼
    ┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐
    │   HTTP POST        │   │   WebSocket /ws    │   │  SSE /es/broadcast │
    │   /api/**          │   │   (cmd/reqId/param)│   │  (EventSource)     │
    └─────────┬──────────┘   └─────────┬──────────┘   └─────────┬──────────┘
              │                        │                        │
              ▼                        ▼                        ▼
    ┌──────────────────────────────────────────────────────────────────────┐
    │                     Gin Engine + 全局中间件                           │
    │  ControlConcurrency → Timing → Recover → CORS → JWT → Gzip → Session │
    └─────────────────────────────────────┬────────────────────────────────┘
                                          │
                        ┌─────────────────┴─────────────────┐
                        ▼                                   ▼
             ┌────────────────────┐              ┌────────────────────┐
             │  无需鉴权 (白名单)  │              │  CheckAuth 中间件   │
             │  /version, /boot...│              │  JWT/Token/Cookie/  │
             └─────────┬──────────┘              │  BasicAuth/本地免密 │
                       │                         └─────────┬──────────┘
                       │                                   │
                       │                                   ▼
                       │                         ┌────────────────────┐
                       │                         │ CheckAdminRole     │─── 403 Forbidden
                       │                         │ CheckReadonly      │─── code=-1 只读提示
                       │                         └─────────┬──────────┘
                       │                                   │
                       └───────────────────┬───────────────┘
                                           ▼
                              ┌────────────────────────────┐
                              │     API Handler 函数        │
                              │  ┌──────────────────────┐  │
                              │  │ 1. JsonArg 解析参数   │  │
                              │  │ 2. ParseJsonArg 校验 │  │
                              │  │ 3. InvalidIDPattern  │  │
                              │  └──────────┬───────────┘  │
                              │             ▼              │
                              │  ┌──────────────────────┐  │
                              │  │ 4. 业务逻辑调用       │  │
                              │  │    model.Xxx()        │  │
                              │  └───────┬───────┬───────┘  │
                              │          │       │          │
                              └──────────┼───────┼──────────┘
                                         │       │
                    ┌────────────────────┘       └────────────────────┐
                    ▼                                                   ▼
         ┌────────────────────┐                              ┌────────────────────┐
         │   同步响应          │                              │  异步推送           │
         │   c.JSON(200, ret)  │                              │  task.Append*Task() │
         │   {code,msg,data}   │                              │  util.Broadcast*()  │
         └─────────┬──────────┘                              │  util.Push*()       │
                   │                                         └─────────┬──────────┘
                   │                                                   │
                   └───────────────────────┬───────────────────────────┘
                                           ▼
                              ┌────────────────────────────┐
                              │   前端 processMessage()     │
                              │  code<0 → showMessage()    │
                              │  cmd=msg/progress/...      │
                              │  → 对应 UI 组件更新         │
                              └────────────────────────────┘
```

---

## 5. 潜在风险与追踪问题

### 5.1 架构级风险

| # | 风险点 | 严重程度 | 描述 | 相关代码 |
|---|--------|----------|------|----------|
| R1 | **并发控制粒度过粗** | 中高 | `ControlConcurrency` 按请求路径加锁，不同文档的写操作也会串行，可能成为性能瓶颈。虽对读操作做了前缀白名单，但非标准命名的读接口会被误锁。 | [session.go#L456-L507](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/session.go#L456-L507) |
| R2 | **参数类型断言 Panic 风险** | 中 | 大量 Handler 使用 `arg["key"].(string)` 直接类型断言，若参数缺失或类型错误会导致 Panic。虽有 `Recover` 中间件，但会中断请求处理链。应推广使用 `ParseJsonArg[T]()`。 | [block.go#L81](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/block.go#L81) |
| R3 | **Session 存储未加密** | 中 | Cookie Session 使用 `cookie.NewStore` 仅签名未加密，`AccessAuthCode` 明文存储于客户端 Cookie。若攻击者获取 Cookie，可直接使用。 | [serve.go#L147-L154](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/server/serve.go#L147-L154) |
| R4 | **JWT 密钥每次启动随机生成** | 低 | `InitJWT()` 使用 `rand.Read(jwtKey)` 生成随机密钥，内核重启后所有已有 JWT 失效。发布服务场景需每次重新鉴权。 | [auth.go#L99-L125](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/model/auth.go#L99-L125) |
| R5 | **API Token 明文配置** | 低 | API Token 存储在 `conf.json` 中且可通过 `/api/system/getConf` 获取，管理员角色即可查看。 | [system.go#L96-L110](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/system.go#L96-L110) |

### 5.2 可维护性问题

| # | 问题 | 影响 | 建议 |
|---|------|------|------|
| M1 | **router.go 单文件过大** | 500+ 行路由注册，可读性差，新增接口易出错 | 按模块拆分为 `router_*.go`（如 `router_block.go`、`router_system.go`） |
| M2 | **路由注册未使用 Group** | 相同前缀的权限配置重复，如所有 `/api/block/*` 都重复写 `CheckAuth, CheckAdminRole, CheckReadonly` | 使用 `ginServer.Group("/api/block", model.CheckAuth, ...)` 分组 |
| M3 | **废弃 API 依赖 TODO 注释** | 无自动化检测，到期可能忘记删除 | 建议添加版本号 + CI 检查机制 |
| M4 | **错误码不统一** | 同时使用 `code=-1/-2` 和 HTTP 状态码取负（`code=-401`），前端判断逻辑复杂 | 统一为单一错误码体系 |

### 5.3 前端交互相关风险

| # | 问题 | 影响 |
|---|------|------|
| F1 | **HTTP 响应始终 200** | 无法通过 HTTP 状态码做负载均衡健康检查、网关层面的错误率统计 |
| F2 | **reqId 竞态只覆盖 5 个接口** | 其他高频操作（如 `getDoc`、搜索建议）仍存在旧响应覆盖问题 |
| F3 | **WebSocket 消息无 ACK 机制** | 服务端推送后无法确认客户端是否收到，网络波动可能导致 UI 状态不一致 |

---

## 6. 追踪问题清单

以下是代码中标记的待追踪事项（来源于 TODO 注释）：

| 追踪链接 | 涉及端点 | 说明 | 计划移除日期 |
|----------|----------|------|--------------|
| [issue#15308](https://github.com/siyuan-note/siyuan/issues/15308#issuecomment-3077675356) | `/api/system/reloadUI` | 迁移至 `/api/ui/reloadUI` | 2026-06-30 |
| [issue#16664](https://github.com/siyuan-note/siyuan/issues/16664#issuecomment-3694774305) | `/api/storage/setLocalStorage` | 迁移至 `/api/storage/setLocalStorageVal` | 2026-06-30 |
| [issue#15663](https://github.com/siyuan-note/siyuan/issues/15663#issuecomment-3219296189) | `/api/filetree/refreshFiletree` | 迁移至 `/api/system/rebuildDataIndex` | 2026-06-30 |
| [PR#17027](https://github.com/siyuan-note/siyuan/pull/17027) | `/api/attr/resetBlockAttrs` | 迁移至 `/api/attr/setBlockAttrs` | 2026-06-30 |
| [issue#15727](https://github.com/siyuan-note/siyuan/issues/15727) | `/api/av/searchAttributeViewNonRelationKey` | 直接废弃 | 2026-06-30 |
| [broadcast.go#L953](file:///d:/fz/0601/solo-dogfeeding/code/303-siyuan/kernel/api/broadcast.go#L953) | CardDAV PROPFIND | Thunderbird 的 `<current-user-privilege-set/>` 属性处理待完善 | - |

---

## 7. 总结

SiYuan 内核 HTTP API 路由层设计体现了以下特点：

**优势：**
- 中间件链层次清晰，全局横切关注点（并发、安全、日志）统一处理
- 鉴权方式多样，适配桌面端、移动端、发布服务、WebDAV 等多种场景
- WebSocket + SSE 双通道实时推送，支持插件扩展的广播机制
- 任务队列实现了异步/同步双模式 + 防抖去重，保障 UI 响应性
- 前后端通过约定的 `{code, msg, data}` 结构形成稳定契约

**改进空间：**
- 路由注册模块化程度可提升（拆文件 + 用 Group）
- 参数验证应全面推广泛型 `ParseJsonArg`，消除类型断言 Panic 风险
- 并发控制可从路径级细化到文档/笔记本级，提升并行度
- 错误码与 HTTP 状态码使用需统一规范

该架构在单机桌面应用场景下表现优秀，为思源笔记的稳定性和扩展性提供了坚实基础。
