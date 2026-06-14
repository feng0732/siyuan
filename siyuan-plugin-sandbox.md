# SiYuan 插件加载与 API 沙箱代码分析

## 1. 概述

SiYuan（思源笔记）的插件系统采用**前后端协同架构，内核（Go）负责插件元数据管理、代码加载与多端广播，前端（TypeScript/JavaScript）通过 `window.eval` 执行插件代码，依托 `Plugin` 基类与封装的 `API` 对象作为受控的运行时环境。系统的「沙箱」并非操作系统级别的进程隔离，而是基于**命名空间封装、API 白名单、模块注入、运行时包装**的轻量级隔离模型。

本文档沿着运行链路分析插件发现、初始化、权限隔离、接口调用和生命周期回收五大阶段的协作机制，并标注安全边界、错误处理、资源释放与兼容性策略。

---

## 2. 插件发现与初始化流程

### 2.1 后端发现（内核层）

#### 2.1.1 持久化存储结构

插件元数据持久化于 `data/storage/petal/petals.json`，由 `Petal` 结构体描述：

- [plugin.go:35-47](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L35-L47) 定义了 `Petal` 核心字段：

```go
type Petal struct {
    Name              string         // 插件名（唯一标识）
    DisplayName       string         // 展示名
    Enabled           bool           // 是否启用
    Incompatible      bool           // 是否不兼容
    DisabledInPublish bool           // 发布模式是否禁用
    DisallowInstall   bool           // 是否不允许安装
    JS   string                      // JS 源码
    CSS  string                      // CSS 源码
    I18n map[string]any             // 国际化文本
}
```

#### 2.1.2 插件目录扫描

[plugin.go:240-294](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L240-L294) `getPetals()` 读取 `petals.json`，并校验磁盘上 `data/plugins/<name>/plugin.json` 是否存在，不存在则清理过期配置。

#### 2.1.3 加载入口

[petal.go:29-45](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/api/petal.go#L29-L45) `loadPetals` API 接收前端请求，调用 `model.LoadPetals(frontend, isPublish)`。

[plugin.go:92-139](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L92-L139) `LoadPetals` 使用 `singleflight.Group` 合并并发请求，防止多前端实例同时触发时的重复加载。

核心过滤逻辑：

```
Conf.Bazaar.PetalDisabled → 总开关
Conf.Bazaar.Trust → 信任确认（桌面端/Docker需手动确认）
petal.Enabled → 用户启用
!petal.Incompatible → 兼容
!(isPublish && petal.DisabledInPublish) → 发布模式
!petal.DisallowInstall → 不被禁止
```

#### 2.1.4 源码加载

[plugin.go:141-212](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L141-L212) `loadCode(petal)` 从磁盘读取：

- `data/plugins/<name>/index.js` → `petal.JS`
- `data/plugins/<name>/index.css` → `petal.CSS`
- `data/plugins/<name>/i18n/<lang>.json` → `petal.I18n`（优先匹配 `Conf.Lang`，回退 `en_US` → `zh_CN` → 第一个文件）

---

### 2.2 兼容性检查（发现阶段的一部分）

[plugin.go:63-107](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/plugin.go#L63-L107) `IsIncompatiblePlugin` 检查：

```go
// 检查 Backends 数组包含当前 backend（windows/darwin/linux/docker/android/ios）
// 检查 Frontends 数组包含当前 frontend（desktop-mobile/desktop-browser/mobile）
// 缺省字段视为 "all"
```

`isBelowRequiredAppVersion` 检查 `plugin.json` 中 `minAppVersion` 与当前内核版本。

### 2.3 前端初始化

#### 2.3.1 调用链总览

```
前端启动
  └─ App 构造函数
      └─ loadPlugins(app)            [loader.ts:30-45]
          ├─ fetchSyncPost("/api/petal/loadPetals")
          ├─ for each petal:
          │   ├─ loadPluginJS(app, item)   [loader.ts:47-78]
          │   │   ├─ runCode(js, sourceURL)   [loader.ts:26-28]
          │   │   ├─ 校验 extends Plugin
          │   │   ├─ new pluginClass(...)
          │   │   ├─ app.plugins.push()
          │   │   └─ await plugin.onload()
          │   └─ insertPluginCSS(item)     [loader.ts:90-98]
```

#### 2.3.2 代码执行封装

[loader.ts:14-28](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L14-L28)

```typescript
const requireFunc = (key: string) => {
    const modules = { siyuan: API }
    return modules[key] ?? window.require?.(key)
}

const runCode = (code: string, sourceURL: string) => {
    return window.eval(
        "(function anonymous(require, module, exports){"
        + code + "\n})\n//# sourceURL=" + sourceURL + "\n"
    )
}
```

**关键点**：
- 插件代码被包裹在 IIFE（立即执行函数）中，通过形参 `require`、`module`、`exports` 注入沙箱内
- `require("siyuan")` 被劫持到内部封装的 `API` 对象，而非真实 node `require`
- `sourceURL` 使用 `plugin:<encodedName>` 便于调试器识别
- 在 Electron 环境中 `requireFunc.__proto__ = window.require`，保留对其他模块的访问权（后门）

#### 2.3.3 插件实例化与校验

[loader.ts:56-77](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L56-L77)

```typescript
const pluginClass = (moduleObj.exports || exportsObj).default || moduleObj.exports
// 校验1: 必须是函数
// 校验2: 必须继承自 Plugin 基类
const plugin = new pluginClass({app, displayName, name, i18n})
app.plugins.push(plugin)
await plugin.onload()
```

[plugin/index.ts:66-110](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L66-L110) `Plugin` 构造函数中：
- 通过 `Object.defineProperty(this, "name", { writable: false })` 锁定 `name` 只读（issue #9943）
- 初始化 `EventBus` 独立实例
- 注册快捷键配置到 `window.siyuan.config.keymap.plugin`

---

## 3. API 沙箱权限隔离机制

### 3.1 "沙箱"的本质：模块注入 + 受控 API

SiYuan 的"沙箱"不是 VM 级隔离，而是**受控命名空间模式。核心隔离点如下：

| 隔离维度 | 实现方式 | 代码位置 |
|---|---|---|
| **代码作用域** | IIFE 包裹，局部 `require`/`module`/`exports` 形参遮蔽全局 | [loader.ts:26-28](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L26-L28) |
| **模块白名单** | `require("siyuan") 仅暴露 `API` 对象，其余走 `window.require`（桌面端可用） | [loader.ts:14-24](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L14-L24) |
| **数据路径隔离** | `loadData/saveData/removeData` 路径固定 `data/storage/petal/<name>/` | [plugin/index.ts:263-335](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L263-L335) |
| **事件总线隔离** | 每个插件独立 `EventBus`（`document.appendChild(comment)`） | [EventBus.ts:3-25](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus.ts#L3-L25) |
| **前端只读模式** | `window.siyuan.config.readonly` 或 `window.siyuan.isPublish` 时存储操作 Reject 403 | [plugin/index.ts:280-286](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L280-L286) |

### 3.2 暴露的 API 对象

[API.ts:340-377](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/API.ts#L340-L377) `API` 对象导出以下能力：

```typescript
export const API = {
    adaptHotkey, confirm, Constants, showMessage, hideMessage,
    fetchPost, fetchSyncPost, fetchGet,          // HTTP 请求封装
    getFrontend, getBackend,
    getModelByDockType,
    openTab, openWindow, openMobileFileById,      // UI 打开操作
    lockScreen, exitSiYuan,                       // 系统操作
    Protyle, ProtyleMethod, Plugin, Dialog, Menu, Setting,  // 类引用
    getAllEditor, getActiveTab, getAllModels, getAllTabs, getActiveEditor,
    platformUtils, openSetting, openAttributePanel,
    saveLayout, globalCommand, expandDocTree, openEmoji
}
```

**注意**：
1. **HTTP API 全部经由前端直接暴露给插件，无中间鉴权层。插件可调用所有 `/api/*` HTTP 接口（取决于内核的 HTTP Cookie/Session 鉴权）。
2. `Plugin` 类本身也被暴露，插件可以创建其他插件实例（理论上），虽然实际很少这样做）。
3. `fetchPost` 等通过 [fetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/util/fetch.ts) 使用浏览器原生 `fetch`，遵循同源策略。

### 3.3 EventBus 隔离

[EventBus.ts:3-25](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus.ts#L3-L25)

```typescript
export class EventBus<DetailType = any> {
    private eventTarget: EventTarget;
    constructor(name = "") {
        this.eventTarget = document.appendChild(document.createComment(name));
    }
    on(type, listener)  { this.eventTarget.addEventListener(type, listener); }
    once(type, listener){ ... }
    off(type, listener) { ... }
    emit(type, detail) { ... }
}
```

每个插件的 `eventTarget` 是一个独立的 DOM Comment 节点（`<!-- pluginName -->`），挂载到 `document` 下。事件不会与其他插件或核心代码冲突。

### 3.4 存储路径隔离

插件数据访问均通过 `Plugin` 基类方法，固定路径前缀 `data/storage/petal/<pluginName>/...`

```typescript
loadData(storageName) {
    fetchPost("/api/file/getFile", {
        path: `/data/storage/petal/${this.name}/${normalizeStoragePath(storageName)}
    ...
}
```

[plugin/index.ts:21](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/util/pathName.ts) 的 `normalizeStoragePath` 应做路径规范化，防止路径穿越。

---

## 4. 接口调用链路

### 4.1 插件 → 前端内部 API（siyuan.* 调用链

```
插件代码
  └─ import / const siyuan = require("siyuan")
        └─ siyuan.fetchPost(url, data)          API.ts → fetchPost()
              └─ fetch(url, init)               fetch.ts:8-117
                    ├─ 401 → 3s 后 location.reload()
                    ├─ 403/404 → 封装错误对象
                    ├─ 202 (getFile 特殊) → failCallback
                    └─ processMessage(response) → 系统消息处理
```

### 4.2 前端 → 内核（HTTP API）

插件通过 `fetchPost("/api/...")` 直接与内核通信。鉴权依赖浏览器同源策略与内核 `gin` 路由 Session。

内核路由在 [router.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/api/router.go) 中注册。插件可调用内核公开的所有 API。

### 4.3 内核 → 前端（WebSocket 广播）

[push_reload.go:46-90](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/push_reload.go#L46-L90) `PushReloadPlugin` 负责广播到所有前端实例：

```go
// 优先级去重：卸载 > 禁用 > 启用/重载 > 数据变更
orderedSets := [uninstall, unload, reload, dataChange]
// 后续集合中移除已出现在前序集合中的项
util.BroadcastByType("main", "reloadPlugin", 0, "", payload)
```

前端接收 [index.ts:73-76](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/index.ts#L73-L76) 通过 WebSocket `msgCallback` 做两次分发：

1. **全局广播给所有插件：
```typescript
this.plugins.forEach((plugin) => {
    plugin.eventBus.emit("ws-main", data);
});
```

2. **根据 `data.cmd` 分发到具体处理函数**（如 `reloadPlugin` → `reloadPlugin(this, data.data)`）

### 4.4 插件间通信

插件间没有直接的官方通信机制，只能通过：
1. 各自监听全局 `window` 对象（非官方、非隔离）
2. 通过 `siyuan.*` 全局对象上的 `eventBus`（`ws-main` 事件是唯一的广播信道）

---

## 5. 生命周期回收、错误处理与资源释放

### 5.1 卸载流程总览

```
触发源（WebSocket cmd: reloadPlugin）
  └─ reloadPlugin(app, data)              [loader.ts:225-265]
        ├─ unloadPlugins → uninstall(app, name, true)    isReload=true
        ├─ uninstallPlugins → uninstall(app, name, false)    isReload=false
        ├─ reloadPlugins → uninstall + loadPlugins + afterLoadPlugin
        └─ dataChangePlugins → plugin.onDataChanged()
```

### 5.2 uninstall 函数

[uninstall.ts:11-123](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L11-L123) 完整的资源释放步骤（按顺序）：

| 步骤 | 操作 | 代码 |
|---|---|---|
| 1 | `plugin.onunload()` | 生命周期回调，try/catch | [uninstall.ts:14-18](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L14-L18) |
| 2 | `plugin.uninstall()`（仅非 reload） | 用户彻底卸载时调用 | [uninstall.ts:19-27](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L19-L27) |
| 3 | 清理 dock 配置持久化 | 保存位置、尺寸、显示状态 | [uninstall.ts:56-101](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L56-L101) |
| 4 | 关闭自定义 Tab（非 reload）或 update() | `custom.parent.parent.removeTab()` | [uninstall.ts:28-42](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L28-L42) |
| 5 | 移除顶栏图标 | `item.remove()` | [uninstall.ts:43-49](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L43-L49) |
| 6 | 移除状态栏图标 | `item.remove()` | [uninstall.ts:51-55](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L51-L55) |
| 7 | 移除 dock 按钮 | `leftDock/rightDock/bottomDock.remove(key) | [uninstall.ts:79-97](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L79-L97) |
| 8 | 移除 EventBus Comment 节点 | `document.removeChild(comment) | [uninstall.ts:103-109](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L103-L109) |
| 9 | 从 `app.plugins` 数组移除 | `splice(index, 1) | [uninstall.ts:110-112](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L110-L112) |
| 10 | 移除自定义 SVG icons | `svg[data-name="pluginName"]`.remove() | [uninstall.ts:112-114](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L112-L114) |
| 11 | 更新编辑器工具栏 | `protyle.toolbar.update()` | [uninstall.ts:114-118](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L114-L118) |
| 12 | 移除内联 style | `#pluginsStyle<name>`.remove() | [uninstall.ts:118-120](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L118-L120) |

### 5.3 错误处理策略

所有插件回调均包裹在 `try/catch` 中，仅输出 `console.error`，**不中断主流程**：

| 位置 | 错误点 |
|---|---|
| [loader.ts:51-55](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L51-L55) | `runCode` JS 语法/运行时错误 |
| [loader.ts:72-77](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L72-L77) | `onload()` 错误 |
| [loader.ts:136-140](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L136-L140) | `onLayoutReady()` 错误 |
| [uninstall.ts:14-18](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L14-L18) | `onunload()` 错误 |
| [uninstall.ts:20-24](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L20-L24) | `uninstall()` 错误 |
| [loader.ts:255-259](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L255-L259) | `onDataChanged()` 错误 |

**缺陷**：插件错误仅记录到 `console`，无用户级提示，除非插件自身弹出 UI。

### 5.4 onDataChanged 特殊处理

[plugin/index.ts:124-134](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L124-L134)

```typescript
public onDataChanged() {
    // 兼容 3.4.1 以前同步数据使用重载插件的问题
    uninstall(this.app, this.name, true);     // 先卸载
    loadPlugins(this.app, [this.name], false).then(() => {
        afterLoadPlugin(this);               // 再加载
        // 刷新工具栏
    });
}
```

默认行为是**重载自身。插件可覆写此方法实现更细粒度的数据同步感知。

---

## 6. 兼容性策略

### 6.1 平台兼容性矩阵

后端层面：

- `plugin.json → `frontends` / `backends` 数组声明支持的平台，缺省视为 all。

[plugin.go:64-80](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/plugin.go#L64-L80)

### 6.2 最低版本要求

`minAppVersion` 字段声明最低内核版本，低于则 `DisallowInstall`。

### 6.3 发布模式禁用

`DisabledInPublish` 字段：内核 `isPublish` 加载时自动跳过。

### 6.4 前端层面的平台分支

大量使用编译条件编译（webpack DefinePlugin：

```typescript
/// #if !MOBILE
/// #else
/// #if BROWSER
/// #endif
```

根据目标构建不同产物（desktop/mobile/browser）。

### 6.5 API 兼容性包装类自身的兼容

[Menu.ts:39-60](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/Menu.ts#L39-L60) `addSeparator` 中显式兼容 3.1.24 之前版本的不同参数签名。

### 6.6 信任确认机制（桌面端/Docker端

[plugin.go:113-118](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L113-L118)

```go
if !Conf.Bazaar.Trust {
    // 移动端没有集市模块默认开启
    if Container || Docker {
        return // 桌面端/Docker 需要用户手动确认过信任后才能开启
    }
}
```

---

## 7. 关键协作关系图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Kernel (Go)                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  bazaar.ParseInstalledPlugin()                     │    │
│  │  ├─ plugin.json 解析 → Package                   │    │
│  │  └─ 兼容性检查 (frontends/backends/minVersion)│    │
│  │  model.LoadPetals()                                 │    │
│  │  ├─ singleflight 去重                           │    │
│  │  ├─ loadCode() 读 JS/CSS/I18n                │    │
│  │  └─ 返回 []*Petal                              │    │
│  │  model.PushReloadPlugin()                          │    │
│  │  └─ WebSocket 广播 "reloadPlugin               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                         │  HTTP / WebSocket                        │
└─────────────────────────┼──────────────────────────────────────┘
                          │
┌─────────────────────────┼──────────────────────────────────────┐
│                    Frontend (TS/JS)                          │
│  ┌─────────────────────┴─────────────────────────────────┐    │
│  │  App.plugins[]                                   │    │
│  │  ├─ ws msgCallback → plugin.eventBus.emit()│    │
│  │  └─ ws cmd:reloadPlugin → reloadPlugin()     │    │
│  │                                                    │    │
│  │  loader.ts                                        │    │
│  │  ├─ loadPlugins() → fetch /api/petal/loadPetals  │    │
│  │  ├─ loadPluginJS()                                │    │
│  │  │   ├─ runCode() → window.eval(IIFE)       │    │
│  │  │   ├─ requireFunc(siyuan → API)              │    │
│  │  │   ├─ validate extends Plugin              │    │
│  │  │   └─ plugin.onload()                      │    │
│  │  └─ afterLoadPlugin() → onLayoutReady()       │    │
│  │                                                    │    │
│  │  Plugin (基类)                                  │    │
│  │  ├─ EventBus (document Comment 节点)            │    │
│  │  ├─ loadData/saveData/removeData               │    │
│  │  ├─ addTopBar/addStatusBar/addDock/addTab      │    │
│  │  └─ protyleSlash / customBlockRenders         │    │
│  │                                                    │    │
│  │  uninstall.ts                                     │    │
│  │  └─ 12 步资源清理                           │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

---

## 8. 安全边界与潜在风险

### 8.1 已有的安全措施

| 措施 | 说明 |
|---|---|
| 插件名只读锁定 | `Object.defineProperty(name, writable: false) 防止覆盖 |
| 存储路径前缀固定 | `/data/storage/petal/<name>/  |
| 只读/发布模式下写操作 Reject | 403  |
| 单飞并发合并 singleflight | 防止多实例并发加载去重 |
| `petals.json 一致性清理 | 磁盘插件不存在时清理持久化 |
| XSS 转义 | bazaar 展示字段 HTML Escape |

### 8.2 潜在风险点

#### 风险 1：`window.eval` 直接执行

[loader.ts:26-28](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L26-L28) 缺乏代码来源文件系统，无代码签名校验。恶意插件可执行任意 JS：

```typescript
window.eval("(function anonymous(require, module, exports){" + code + "}")
```

**影响**：插件可访问 `window`、`document` 全对象，可读取全局任意 DOM，可发起任意同源 HTTP（同源），可访问 Electron `window.require`（桌面端）→ Node 能力）。

#### 风险 2：`require` 的原型链泄漏

[loader.ts:22-24](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus Comment 节点泄漏

[EventBus.ts:7](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus.ts#L7) `document.appendChild(...)`：EventBus 虽然每个插件各自独立，但挂载到 document，但仍可通过 `document.childNodes` 遍历所有 Comment。

#### 风险 4：`window.siyuan

全局对象完全暴露

`window.siyuan` 全局对象可被插件读写。

#### 风险 5：fetchPost 无二次鉴权

插件通过 `siyuan.fetchPost` 与 `fetch` 与内核通信，内核侧仅依赖 Session，无法区分来自插件或来自用户操作。

#### 风险 6：onunload 清理不全

若插件注册全局监听但未在 `onunload` 中显式移除全局事件监听/DOM 修改，`uninstall.ts 只清理已知资源注册表的入口，未清理：

- 全局 `window/document` 监听
- 全局定时器 `setInterval`
- 动态创建 DOM 节点未通过 `Plugin` API

#### 风险 7：CSS 注入无隔离

[loader.ts:90-98](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L90-L98) CSS 直接 `textContent` 直接插入 `<style>` 无沙箱，可覆盖全局样式。

---

## 9. 待验证问题

1. **`normalizeStoragePath` 的路径穿越防护是否足够？

验证 `normalizeStoragePath(storageName)` 实现是否阻止 `../` 穿越？

2. **`isBelowRequiredAppVersion` 的版本比较算法实现？

在 `bazaar` 包中未找到实现，需确认 `DisallowInstall` 的准确判断逻辑。

3. **WebSocket 广播插件重载竞态：多窗口同时重载时 `loadPlugins` 是否需要锁？

前端 `loadPlugins` 非串行部分 `init=true` 时 `loadPluginJS` 不 await，多插件并行初始化是否竞态访问 `app.plugins`。

4. **插件间循环引用插件 A onload 抛异常后部分资源未清理：

`loadPluginJS` 抛异常后，已经 `app.plugins.push` 是否在 try/catch 之前还是之后？代码是 push 后 onload 抛错后未回滚？

查看 [loader.ts:71-77](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L71-L77):

```typescript
app.plugins.push(plugin);  // L71
try {
    await plugin.onload();  // L73 可能抛错
} catch (e) { ... }   // 仅 log，未移除
```

→ **结论**：onload 失败插件实例残留。

5. **EventBus 离线事件监听是否在卸载时 removeEventListener？

`uninstall.ts` 移除了 Comment 节点本身会自动移除所有监听器（DOM 节点移除会 detach）。

6. **`ws-main` 事件的性能问题**

所有 WebSocket 消息广播到所有插件 eventBus，插件数量多时放大。

7. **发布模式下 CSS/JS

发布模式下 CSS 和 JS 是由前端实现是什么？

8. **`requireFunc` 中 Electron 原型链

```typescript
requireFunc.__proto__ = window.require
```

→ 原型链设置给插件可通过 `Object.getPrototypeOf(requireFunc)(...)` 直接调用原生 require。

---

## 10. 关键文件索引

| 文件 | 职责 |
|---|---|
| [kernel/model/plugin.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go) | Petal 模型、持久化、代码加载 |
| [kernel/api/petal.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/api/petal.go) | HTTP API：loadPetals / setPetalEnabled |
| [kernel/bazaar/plugin.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/plugin.go) | 已安装插件解析、兼容性判断 |
| [kernel/bazaar/package.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/package.go) | Package 元数据结构、JSON 解析、XSS 转义 |
| [kernel/model/push_reload.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/push_reload.go) | PushReloadPlugin 广播与优先级去重 |
| [app/src/plugin/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts) | Plugin 基类（生命周期、存储、UI 注册 |
| [app/src/plugin/loader.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts) | 插件代码加载器（eval 执行重载 |
| [app/src/plugin/API.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/API.ts) | 暴露给插件的 API 对象 |
| [app/src/plugin/EventBus.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus.ts) | 基于 DOM Comment 的事件总线 |
| [app/src/plugin/uninstall.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts) | 12 步资源清理卸载流程 |
| [app/src/plugin/Setting.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/Setting.ts) | 插件设置面板封装 |
| [app/src/plugin/Menu.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/Menu.ts) | 菜单封装（含版本兼容） |
| [app/src/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/index.ts) | App 类与 WebSocket 消息分发 |
| [app/src/util/fetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/util/fetch.ts) | HTTP 请求封装（鉴权失效处理 |
| [app/src/plugin/platformUtils.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/platformUtils.ts) | 平台工具函数与通知 API |
