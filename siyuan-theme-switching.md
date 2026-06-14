# SiYuan 主题与外观切换机制分析

## 一、整体架构概述

SiYuan 采用前后端分离架构，主题与外观系统由**后端内核（Go）**负责配置管理、资源扫描与持久化，**前端界面（TypeScript）**负责资源加载、样式切换与界面刷新，二者通过 **HTTP API** 和 **WebSocket 广播** 进行协作。

### 架构分层

| 层级 | 职责 | 核心模块 |
|------|------|----------|
| 表现层 | 主题/图标选择 UI、设置面板、动态样式渲染 | `appearance.ts`、`assets.ts`、移动端 `appearance.ts` |
| 业务层 | 主题加载、图标加载、外观设置、代码片段 | `loadAssets()`、`InitAppearance()`、`LoadThemes()` |
| 数据层 | 配置持久化、资源文件管理、文件锁 | `conf.json`、`filelock`、`Appearance` 结构体 |
| 通信层 | HTTP 请求、WebSocket 广播 | `fetchPost`、`BroadcastByType` |

---

## 二、主题资源管理

### 2.1 资源目录结构

主题资源分布在两个位置：

- **内置资源**：`app/appearance/`（随应用发布）
  - `themes/daylight/` - 明亮主题
  - `themes/midnight/` - 暗黑主题
  - `icons/` - 图标集
  - `langs/` - 多语言包
  - `fonts/` - 字体资源

- **用户配置目录**：`~/.siyuan/appearance/`（运行时拷贝 + 用户扩展）
  - `themes/` - 所有可用主题
  - `icons/` - 所有可用图标
  - `emojis/` - 自定义表情

> 开发模式下直接使用工作目录下的资源，生产模式从配置目录加载。
> 参考：[working.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/util/working.go#L158-L165)

### 2.2 主题包结构

每个主题是一个独立目录，包含：

| 文件 | 作用 | 可选性 |
|------|------|--------|
| `theme.json` | 主题元数据（名称、版本、作者、适用模式） | 必需 |
| `theme.css` | 主题样式表 | 必需 |
| `theme.js` | 主题脚本（动态行为） | 可选 |

**主题元数据示例**（[daylight/theme.json](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/appearance/themes/daylight/theme.json)）：

```json
{
  "name": "daylight",
  "author": "Vanessa",
  "url": "https://github.com/Vanessa219",
  "version": "1.1.1",
  "modes": ["light"]
}
```

### 2.3 主题扫描与加载

后端 `InitAppearance()` 是初始化入口，执行以下操作：

1. **创建外观目录**：确保配置目录存在
2. **拷贝内置资源**：将 `app/appearance/` 拷贝到用户配置目录
3. **加载主题列表**：扫描 `themes/` 目录，解析每个主题的 `theme.json`
4. **加载图标列表**：扫描 `icons/` 目录
5. **默认值回退**：检查当前配置是否有效，否则回退到内置主题

参考：[appearance.go - InitAppearance()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/appearance.go#L34-L69)

**主题扫描逻辑**（[LoadThemes()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/appearance.go#L129-L212)）：

- 遍历主题目录，跳过非目录项
- 解析 `theme.json`，失败则跳过该主题
- 根据 `modes` 字段将主题分为 `lightThemes` 和 `darkThemes`
- 内置主题（daylight/midnight）始终排在列表首位
- 记录当前使用主题的版本号和是否有 JS 脚本

---

## 三、配置状态管理

### 3.1 配置数据结构

#### 后端 Go 结构体

[appearance.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/conf/appearance.go#L21-L39) 定义了 `Appearance` 结构体：

| 字段 | 类型 | 说明 |
|------|------|------|
| `Mode` | int | 模式：0=明亮，1=暗黑 |
| `ModeOS` | bool | 是否跟随系统主题 |
| `ThemeLight` | string | 明亮模式主题名 |
| `ThemeDark` | string | 暗黑模式主题名 |
| `ThemeVer` | string | 当前主题版本 |
| `ThemeJS` | bool | 是否启用主题 JS |
| `Icon` | string | 当前图标集 |
| `IconVer` | string | 图标版本 |
| `CodeBlockThemeLight` | string | 明亮模式代码主题 |
| `CodeBlockThemeDark` | string | 暗黑模式代码主题 |
| `Lang` | string | 界面语言 |

#### 前端 TypeScript 接口

[config.d.ts - IAppearance](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/types/config.d.ts#L194-L265) 定义了对应的接口。

### 3.2 配置持久化

配置保存在 `~/.siyuan/conf.json`，通过 `AppConf.Save()` 方法持久化。

**持久化机制**（[conf.go - Save()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/conf.go#L870-L900)）：

```
1. 加读写锁（m.Lock），防止并发写入
2. 序列化为 JSON（缩进格式）
3. 读取旧配置，对比是否有变化（无变化则跳过写入）
4. 使用 filelock 原子写入，避免文件损坏
```

**文件锁保证**：使用 `filelock.WriteFile()` 确保写入过程的原子性，防止进程崩溃导致配置文件损坏。

### 3.3 配置初始化与默认值

`InitConf()` 函数负责配置初始化：

1. 从 `conf.json` 加载已有配置
2. 对缺失字段设置默认值（防御性编程）
3. 对无效值进行修正（如语言不存在则回退到 `en_US`）
4. 首次运行时填充所有默认值
5. 保存修正后的配置

参考：[conf.go - InitConf()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/conf.go#L123-L625)

**默认值**（[NewAppearance()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/conf/appearance.go#L41-L55)）：

- 模式：明亮（Mode=0）
- 跟随系统：开启（ModeOS=true）
- 明亮主题：daylight
- 暗黑主题：midnight
- 图标：material
- 语言：en_US

---

## 四、界面刷新与动态切换流程

### 4.1 切换触发方式

主题切换有三种触发方式：

| 触发方式 | 入口 | 说明 |
|----------|------|------|
| 用户手动切换 | 设置面板选择主题/模式 | 完整的设置保存流程 |
| 系统主题变化 | `prefers-color-scheme` 监听 | 仅在跟随系统模式下触发 |
| 主题文件变化 | 文件监听 + `refreshtheme` 广播 | 仅刷新样式，不改变配置 |

### 4.2 完整切换流程（用户手动切换）

```
┌─────────────┐     HTTP POST     ┌──────────────┐     Save     ┌──────────┐
│  前端 UI     │ ────────────────► │  setAppearance │ ───────────► │ conf.json │
│ appearance.ts│   /api/setting/   │  (setting.go) │              └──────────┘
└─────────────┘                    └──────────────┘
       ▲                                    │
       │                                    │ BroadcastByType
       │                                    ▼
       │                            ┌──────────────┐
       │                            │  WebSocket   │
       │                            │   广播通道    │
       │                            └──────────────┘
       │                                    │
       │  updateAppearance()                │
       └────────────────────────────────────┘
                    loadAssets()
                    ├── 切换默认主题 CSS
                    ├── 切换自定义主题 CSS
                    ├── 切换主题 JS（如有）
                    ├── 切换图标集
                    ├── 设置代码高亮主题
                    └── 更新 HTML data 属性
```

**步骤详解**：

#### 第 1 步：前端提交设置
用户在设置面板修改主题后，`appearance._send()` 收集所有外观配置，通过 `fetchPost("/api/setting/setAppearance", ...)` 发送到后端。

参考：[appearance.ts - _send()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/config/appearance.ts#L184-L212)

#### 第 2 步：后端保存配置
`setAppearance()` 接口：
1. 反序列化参数到 `Appearance` 结构体
2. 更新内存中的配置（`model.Conf.Appearance`）
3. 调用 `model.Conf.Save()` 持久化到磁盘
4. 调用 `model.InitAppearance()` 重新扫描主题
5. 通过 WebSocket 广播 `setAppearance` 事件

参考：[setting.go - setAppearance()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/api/setting.go#L530-L562)

#### 第 3 步：前端接收广播并刷新

**桌面端**：`updateAppearance()` 函数处理广播消息：
1. 若启用了主题 JS 且主题发生变化，尝试调用 `destroyTheme()` 清理
2. 若无 `destroyTheme` 函数，则导出布局后刷新页面
3. 更新状态栏显示状态
4. 调用 `appearance.onSetAppearance()` → `loadAssets()` 加载资源

参考：[updateAppearance.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/config/util/updateAppearance.ts)

**移动端**：直接 `window.location.reload()` 刷新页面

参考：[onMessage.ts - setAppearance](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/mobile/util/onMessage.ts#L39-L41)

### 4.3 资源加载核心逻辑

`loadAssets()` 是前端主题加载的核心函数，执行以下操作：

#### (1) HTML 元数据设置
设置 `<html>` 标签的 `data-*` 属性，供 CSS 选择器使用：
- `data-theme-mode`：当前模式（light/dark）
- `data-light-theme`：明亮主题名
- `data-dark-theme`：暗黑主题名
- `data-frontend` / `data-backend`：前端/后端类型

参考：[assets.ts - loadAssets()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/util/assets.ts#L22-L160)

#### (2) 系统主题同步
若开启了 `modeOS`，检查系统主题是否与当前模式一致，不一致则调用后端 API 同步。

#### (3) 样式加载（双层级）

| 样式层级 | 元素 ID | 作用 |
|----------|---------|------|
| 默认主题 | `themeDefaultStyle` | 内置主题（daylight/midnight），始终加载 |
| 自定义主题 | `themeStyle` | 用户选择的第三方主题，叠加在默认主题之上 |

**加载策略**：
- 内置主题：始终加载，确保基础样式可用
- 自定义主题：仅在选择了非内置主题时加载
- 切换时使用 **先加载后移除** 策略，避免白屏闪烁

```javascript
// 默认主题切换：新样式加载完成后再移除旧样式
new Promise((resolve) => {
    newStyleElement.onload = resolve;
    defaultStyleElement.parentNode.insertBefore(newStyleElement, defaultStyleElement);
}).then(() => {
    defaultStyleElement.remove();
    newStyleElement.id = "themeDefaultStyle";
});
```

#### (4) 主题脚本加载
检查主题目录下是否存在 `theme.js`，通过 `addScript()` 动态加载。
主题脚本可通过定义 `window.destroyTheme()` 函数提供清理能力。

#### (5) 图标集加载
采用 **默认 + 扩展** 模式：
- 默认图标集（material 或 ant）始终加载，作为基础保障
- 第三方图标集作为扩展加载，覆盖默认图标

#### (6) 代码高亮主题
根据当前模式选择对应的代码高亮主题，从 CDN 加载 highlight.js 样式。

参考：[util.ts - setCodeTheme()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/protyle/render/util.ts#L52-L75)

### 4.4 系统主题跟随机制

`initAssets()` 中注册了 `prefers-color-scheme` 监听器：

```javascript
window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", event => {
    if (!window.siyuan.config.appearance.modeOS) return;
    // 调用后端 API 更新模式
    fetchPost("/api/system/setAppearanceMode", {mode: ...}, response => {
        window.siyuan.config.appearance = response.data.appearance;
        loadAssets(response.data.appearance);
    });
});
```

后端 `setAppearanceMode` 仅更新 `Mode` 和 `ThemeJS` 字段，并保存配置，但**不广播** `setAppearance` 事件（由前端直接处理刷新）。

参考：[system.go - setAppearanceMode()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/api/system.go#L704-L725)

### 4.5 热刷新机制（主题文件变化）

后端通过文件监听检测主题 CSS 变化，触发 `broadcastRefreshThemeIfCurrent()`：
- 仅处理当前使用的主题
- 发送 `refreshtheme` 广播，附带带时间戳的 CSS URL
- 前端直接更新 `<link>` 的 `href` 属性，浏览器自动重新加载

参考：[appearance.go - broadcastRefreshThemeIfCurrent()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/appearance.go#L259-L276)

---

## 五、默认回退机制

### 5.1 主题回退

在 `InitAppearance()` 中进行有效性检查：

```go
if !containTheme(Conf.Appearance.ThemeDark, Conf.Appearance.DarkThemes) {
    Conf.Appearance.ThemeDark = "midnight"  // 回退到内置暗黑主题
    Conf.Appearance.ThemeJS = false
}
if !containTheme(Conf.Appearance.ThemeLight, Conf.Appearance.LightThemes) {
    Conf.Appearance.ThemeLight = "daylight" // 回退到内置明亮主题
    Conf.Appearance.ThemeJS = false
}
if !gulu.Str.Contains(Conf.Appearance.Icon, Conf.Appearance.Icons) {
    Conf.Appearance.Icon = "material"       // 回退到 material 图标
}
```

**回退场景**：
- 用户手动删除了当前使用的主题文件夹
- 主题文件损坏无法解析
- 升级后主题不再兼容
- 工作空间迁移后主题不存在

### 5.2 代码高亮主题回退

前端 `setCodeTheme()` 中检查配置的代码主题是否在白名单中：

```javascript
if (!Constants.SIYUAN_CONFIG_APPEARANCE_LIGHT_CODE.includes(css)) {
    css = "default"     // 明亮模式回退
}
if (!Constants.SIYUAN_CONFIG_APPEARANCE_DARK_CODE.includes(css)) {
    css = "github-dark" // 暗黑模式回退
}
```

### 5.3 语言回退

后端语言选择采用三级回退：
1. 用户指定语言
2. 系统检测语言（匹配最近似的支持语言）
3. 默认语言 `en_US`

参考：[conf.go - InitConf()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/conf.go#L156-L182)

---

## 六、移动端适配

### 6.1 移动端外观设置

移动端有独立的设置页面 [mobile/settings/appearance.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/mobile/settings/appearance.ts)，功能与桌面端一致但 UI 不同。

### 6.2 状态栏适配

`updateMobileTheme()` 函数根据主题颜色更新移动端状态栏：

- **iOS**：通过 `window.webkit.messageHandlers.changeStatusBar.postMessage()`
- **Android**：通过 `window.JSAndroid.changeStatusBarColor()`
- **HarmonyOS**：通过 `window.JSHarmony.changeStatusBarColor()`

延迟 500ms 执行，确保样式加载完成后能获取正确的背景色。

参考：[assets.ts - updateMobileTheme()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/util/assets.ts#L372-L393)

### 6.3 移动端切换策略

移动端收到 `setAppearance` 广播后**直接刷新页面**，而非桌面端的动态替换：

```javascript
case "setAppearance":
    window.location.reload();
    break;
```

**原因**：移动端主题切换涉及原生状态栏等复杂交互，全量刷新更可靠。

### 6.4 浏览器端主题色

在浏览器环境中，设置 `theme-color` meta 标签，使浏览器地址栏与主题色一致。

---

## 七、异常资源处理

### 7.1 文件系统错误

**严重错误处理**：使用 `util.ReportFileSysFatalError()` 报告致命错误，常见场景：
- 无法创建外观目录
- 无法拷贝内置资源
- 无法读取主题/图标目录

参考：[appearance.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/appearance.go#L36-L46)

### 7.2 主题解析失败

`LoadThemes()` 中对每个主题单独解析，解析失败则跳过该主题，不影响整体加载：

```go
themeConf, parseErr := bazaar.ParsePackageJSON(...)
if nil != parseErr || nil == themeConf {
    continue // 静默跳过无效主题
}
```

### 7.3 主题脚本错误

- `destroyTheme()` 调用被 `try-catch` 包裹，失败不影响后续流程
- 主题 JS 加载失败由浏览器自动处理，仅样式降级可用

### 7.4 资源加载失败

- CSS 加载失败：浏览器自动降级，默认主题样式仍在
- JS 加载失败：仅失去动态交互能力，样式不受影响
- 图标加载失败：默认图标集作为兜底

---

## 八、代码片段（Snippets）机制

代码片段是主题系统的重要补充，允许用户注入自定义 CSS/JS。

### 8.1 存储与加载

- 后端持久化：[snippet.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/api/snippet.go)
- 前端渲染：[snippets.ts - renderSnippet()](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/config/util/snippets.ts#L7-L42)

### 8.2 应用时机

- 启动时：`onGetConfig()` → `renderSnippet()`
- 修改后：收到 `setSnippet` 广播后重新渲染

### 8.3 与主题的优先级关系

代码片段通过 `<style>` 标签注入，优先级高于主题 CSS（因为在 DOM 中位置更靠后），可用来覆盖主题样式。

---

## 九、模块职责总结

| 模块 | 文件 | 核心职责 |
|------|------|----------|
| 后端配置定义 | [conf/appearance.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/conf/appearance.go) | 外观配置的数据结构和默认值 |
| 后端外观业务 | [model/appearance.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/appearance.go) | 主题扫描、图标加载、默认回退、热刷新 |
| 后端设置 API | [api/setting.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/api/setting.go) | setAppearance / setTheme / setIcon 接口 |
| 后端系统 API | [api/system.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/api/system.go) | setAppearanceMode（跟随系统模式切换） |
| 后端配置管理 | [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/model/conf.go) | 配置加载、保存、默认值补全 |
| 后端通信 | [util/websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/kernel/util/websocket.go) | BroadcastByType 广播机制 |
| 前端资源加载 | [util/assets.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/util/assets.ts) | loadAssets / initAssets / 主题图标加载 |
| 前端设置面板 | [config/appearance.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/config/appearance.ts) | 桌面端外观设置 UI 与交互 |
| 前端设置更新 | [config/util/updateAppearance.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/config/util/updateAppearance.ts) | 接收广播后更新外观 |
| 前端代码片段 | [config/util/snippets.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/config/util/snippets.ts) | 代码片段管理与渲染 |
| 前端样式工具 | [protyle/util/addStyle.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/protyle/util/addStyle.ts) | 动态添加样式表 |
| 前端脚本工具 | [protyle/util/addScript.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/protyle/util/addScript.ts) | 动态添加脚本 |
| 前端启动流程 | [boot/onGetConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/boot/onGetConfig.ts) | 初始化时调用外观相关函数 |
| 移动端设置 | [mobile/settings/appearance.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/mobile/settings/appearance.ts) | 移动端外观设置 |
| 移动端消息 | [mobile/util/onMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/288-siyuan/app/src/mobile/util/onMessage.ts) | 移动端消息处理（含 setAppearance） |

---

## 十、潜在风险

### 10.1 主题 JS 安全风险
- 主题脚本运行在主进程上下文中，权限较高
- 第三方主题可能包含恶意代码
- 建议：对集市下载的主题进行安全审计

### 10.2 并发配置写入
- 虽然有 `m.Lock()` 保护，但跨进程场景下仅靠文件锁可能不足
- 多窗口同时修改设置可能存在竞态
- 建议：检查 filelock 的跨进程语义

### 10.3 主题切换性能
- 主题 JS 存在时，可能需要刷新整个页面
- 移动端切换必刷新，体验不如桌面端
- 建议：优化主题 JS 的销毁重建机制

### 10.4 主题文件损坏
- `theme.json` 解析失败时静默跳过，用户可能困惑主题为何消失
- 建议：添加错误提示或日志反馈

### 10.5 资源缓存问题
- 主题 CSS 通过版本号 `?v=` 控制缓存，但热刷新使用时间戳
- 版本号更新不及时可能导致缓存的旧样式生效
- 建议：确保主题版本号机制的一致性

### 10.6 代码片段与主题冲突
- 用户 CSS 片段可能与主题样式冲突
- 无命名空间隔离，调试困难
- 建议：提供代码片段的开关和调试工具

---

## 十一、后续检查点

### 功能验证
- [ ] 切换明亮/暗黑模式是否平滑无闪烁
- [ ] 跟随系统主题在系统切换时是否及时响应
- [ ] 自定义主题安装/卸载后列表是否正确更新
- [ ] 图标集切换是否所有图标正确刷新
- [ ] 代码高亮主题与外观模式是否同步切换
- [ ] 移动端状态栏颜色是否与主题匹配
- [ ] 代码片段启用/禁用是否即时生效

### 异常场景
- [ ] 删除当前使用的主题后是否正确回退
- [ ] 损坏的主题文件是否被优雅跳过
- [ ] 配置文件损坏时是否能恢复默认值
- [ ] 主题 JS 报错时是否不影响基础功能
- [ ] 无网络时代码高亮主题是否降级可用

### 性能与质量
- [ ] 主题切换耗时是否在可接受范围
- [ ] 页面刷新场景是否可优化为动态替换
- [ ] 主题 CSS 是否存在未使用的冗余规则
- [ ] 图标 SVG 是否存在内存泄漏风险

### 安全审查
- [ ] 第三方主题 JS 的安全边界
- [ ] 主题 CSS 中的 expression/behavior 等风险属性
- [ ] 代码片段 XSS 注入可能性
- [ ] 配置文件注入攻击面

---

## 十二、核心调用链路

### 启动初始化链路

```
InitConf()
  └── conf.json 加载 + 默认值补全
InitAppearance()
  ├── 创建/拷贝外观资源
  ├── LoadThemes()  ── 扫描主题目录 → 解析 theme.json → 分类亮/暗
  ├── LoadIcons()   ── 扫描图标目录 → 解析 icon.json
  ├── 默认值回退检查
  └── Conf.Save()

前端 onGetConfig()
  ├── appearance.onSetAppearance()
  ├── initAssets()  ── 注册系统主题监听
  ├── setInlineStyle()  ── 字体/编辑器等内联样式
  └── renderSnippet()  ── 代码片段注入
```

### 用户切换主题链路

```
用户选择主题
  ↓
appearance._send()  ── 收集配置
  ↓
POST /api/setting/setAppearance
  ↓
setAppearance()  ── 更新内存配置 + Save() + InitAppearance()
  ↓
BroadcastByType("main", "setAppearance", ...)
  ↓
前端 updateAppearance(data)
  ├── 主题 JS 清理（destroyTheme）
  ├── 状态栏显示切换
  └── appearance.onSetAppearance(data)
        ├── 更新内存配置
        ├── 更新设置面板 UI
        └── loadAssets(data)
              ├── HTML data 属性更新
              ├── 默认主题 CSS 切换
              ├── 自定义主题 CSS 切换
              ├── 主题 JS 加载
              ├── 图标集加载
              ├── 代码高亮主题切换
              └── 移动端状态栏更新
```
