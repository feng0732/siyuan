# SiYuan 插件加载与 API 沙箱代码分析

## 1. 概述

SiYuan（思源笔记）的插件系统采用前后端协同架构：内核（Go）负责插件元数据管理、代码加载与多端广播；前端（TypeScript/JavaScript）通过 `window.eval` 执行插件代码，依托 `Plugin` 基类与封装的 `API` 对象构建受控的运行时环境。系统的「沙箱」并非操作系统级别的进程隔离，而是基于**命名空间封装、API 白名单、模块注入、运行时包装**的轻量级隔离模型。

本文档沿着运行链路分析插件发现、初始化、权限隔离、接口调用和生命周期回收五大阶段的协作机制，并标注安全边界、错误处理、资源释放与兼容性策略。

---

## 2. 插件发现与初始化流程

### 2.1 后端发现（内核层）

#### 2.1.1 持久化存储结构

插件元数据持久化于 `data/storage/petal/petals.json`，由 `Petal` 结构体描述：

- [plugin.go:35-47](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L35-L47)

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

[plugin.go:240-294](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L240-L294) `getPetals()` 读取 `petals.json`，并校验磁盘上 `data/plugins/<name>/plugin.json` 是否存在，不存在则清理过期配置条目（同时调用 `bazaar.RemovePackageInfo` 清除包信息）。

#### 2.1.3 加载入口与过滤

[petal.go:29-45](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/api/petal.go#L29-L45) `loadPetals` API 接收前端请求，调用 `model.LoadPetals(frontend, isPublish)`。

[plugin.go:92-139](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L92-L139) `LoadPetals` 使用 `singleflight.Group` 合并并发请求，防止多前端实例同时触发时的重复加载。

核心过滤逻辑（6 层）：

```
Conf.Bazaar.PetalDisabled → 总开关关闭则跳过所有
Conf.Bazaar.Trust        → 桌面端/Docker 需用户确认信任
petal.Enabled            → 用户手动启用
!petal.Incompatible      → 平台兼容检查通过
!(isPublish && DisabledInPublish) → 发布模式过滤
!petal.DisallowInstall   → 版本要求满足
```

#### 2.1.4 源码加载

[plugin.go:141-212](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L141-L212) `loadCode(petal)` 从磁盘读取：

- `data/plugins/<name>/index.js` → `petal.JS`（必须存在）
- `data/plugins/<name>/index.css` → `petal.CSS`（可选）
- `data/plugins/<name>/i18n/<lang>.json` → `petal.I18n`（优先匹配 `Conf.Lang`，回退 `en_US` → `zh_CN` → 第一个文件）

---

### 2.2 兼容性检查（发现阶段的一部分）

#### 2.2.1 平台兼容性

[plugin.go:64-80](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/plugin.go#L64-L80) `IsIncompatiblePlugin` 检查：

- **Backends 数组**：当前 backend 为 `runtime.GOOS`（桌面端）或 `util.Container`（docker/android/ios），需在数组中或数组为空/含 "all"
- **Frontends 数组**：当前 frontend 为 `desktop-mobile`/`desktop-browser`/`mobile`，需在数组中或数组为空/含 "all"
- `frontend` 参数为空字符串时跳过兼容性检查（视为兼容）

[plugin.go:96-107](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/plugin.go#L96-L107) `isTargetSupported` 实现逻辑：

```go
func isTargetSupported(platforms []string, target string) bool {
    if len(platforms) == 0 { return true }  // 缺省视为 all
    for _, v := range platforms {
        if v == target || v == "all" { return true }
    }
    return false
}
```

#### 2.2.2 最低版本比较（isBelowRequiredAppVersion）

[installed.go:103-114](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/installed.go#L103-L114) 完整实现：

```go
func isBelowRequiredAppVersion(pkg *Package) bool {
    // 如果包没有指定 minAppVersion，则允许安装
    if "" == pkg.MinAppVersion {
        return false
    }
    // 如果包要求的 minAppVersion 大于当前版本，则不允许安装
    if 0 < semver.Compare("v"+pkg.MinAppVersion, "v"+util.Ver) {
        return true
    }
    return false
}
```

**关键细节**：

1. **依赖库**：使用 `golang.org/x/mod/semver` 进行语义化版本比较，确保符合 [SemVer 2.0](https://semver.org/) 规范
2. **前缀 "v"**：`semver.Compare` 要求版本号带 "v" 前缀（如 `v3.1.0`），因此代码在 `MinAppVersion` 和 `util.Ver` 前均拼接了 `"v"`
3. **缺省放行**：`MinAppVersion` 为空字符串时返回 `false`（允许安装），保证旧格式插件无此字段时不被误拦截
4. **比较方向**：`semver.Compare(A, B)` 返回 >0 表示 A > B。当 `v+MinAppVersion > v+当前版本` 时返回 `true`，即「要求的版本高于当前版本 → 不允许安装」
5. **调用点**：在 [installed.go:66](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/installed.go#L66) `SetInstalledPackageMetadata` 中被调用，同时设置 `DisallowInstall` 和 `DisallowUpdate`；在 [bazaar.go:70-72](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/bazaar.go#L70-L72) 在线集市包展示时也会调用

---

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
if (window.require instanceof Function) {
    requireFunc.__proto__ = window.require
}

const runCode = (code: string, sourceURL: string) => {
    return window.eval(
        "(function anonymous(require, module, exports){"
        + code + "\n})\n//# sourceURL=" + sourceURL + "\n"
    )
}
```

关键点：

- 插件代码被包裹在 IIFE 中，通过形参 `require`、`module`、`exports` 注入沙箱内
- `require("siyuan")` 被劫持到内部封装的 `API` 对象，而非真实 node `require`
- `sourceURL` 使用 `plugin:<encodedName>` 便于调试器识别来源
- 在 Electron 环境中 `requireFunc.__proto__ = window.require`，使 `requireFunc` 原型链指向 Electron 的原生 `require`，插件可通过 `Object.getPrototypeOf(requireFunc)` 绕过白名单访问 Node 原生模块

#### 2.3.3 插件实例化与校验

[loader.ts:56-77](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L56-L77)

```typescript
const pluginClass = (moduleObj.exports || exportsObj).default || moduleObj.exports
// 校验1: typeof pluginClass !== "function" → 报错返回
// 校验2: !(pluginClass.prototype instanceof Plugin) → 报错返回
const plugin = new pluginClass({app, displayName, name, i18n})
app.plugins.push(plugin)            // ← push 在 onload 之前
try {
    await plugin.onload()           // ← onload 异常不会回滚 push
} catch (e) { console.error(...) }
```

[plugin/index.ts:66-110](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L66-L110) `Plugin` 构造函数中：

- 通过 `Object.defineProperty(this, "name", { writable: false })` 锁定 `name` 只读（issue #9943）
- 初始化 `EventBus` 独立实例
- 注册快捷键配置到 `window.siyuan.config.keymap.plugin`

---

## 3. API 沙箱权限隔离机制

### 3.1 沙箱的本质：模块注入 + 受控 API

SiYuan 的"沙箱"不是 VM 级隔离，而是受控命名空间模式。核心隔离点如下：

| 隔离维度 | 实现方式 | 代码位置 | 实际强度 |
|---|---|---|---|
| **代码作用域** | IIFE 包裹，局部 `require`/`module`/`exports` 形参遮蔽全局 | [loader.ts:26-28](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L26-L28) | 弱：插件可自由访问 `window`/`document` |
| **模块白名单** | `require("siyuan")` 仅暴露 `API` 对象，其余走 `window.require`（桌面端可用） | [loader.ts:14-24](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L14-L24) | 弱：原型链泄漏可绕过 |
| **数据路径隔离** | `loadData/saveData/removeData` 路径固定 `data/storage/petal/<name>/` | [plugin/index.ts:263-335](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L263-L335) | 中：需配合 `normalizeStoragePath` 防穿越 |
| **事件总线隔离** | 每个插件独立 `EventBus`（`document.appendChild(comment)`） | [EventBus.ts:3-25](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus.ts#L3-L25) | 中：事件命名空间隔离但可被遍历 |
| **前端只读模式** | `window.siyuan.config.readonly` 或 `window.siyuan.isPublish` 时存储操作 Reject 403 | [plugin/index.ts:280-286](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L280-L286) | 中：仅 Plugin 基类方法有守卫，直接调 fetch 无限制 |

### 3.2 存储路径规范化（normalizeStoragePath）

[pathName.ts:730-742](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/util/pathName.ts#L730-L742) 完整实现：

```typescript
export const normalizeStoragePath = (storageName: string): string | null => {
    const parts = storageName.replace(/\\/g, "/").split("/");
    const resolved: string[] = [];
    for (const part of parts) {
        if (part === "..") {
            if (resolved.length > 0) {
                resolved.pop();       // 遇到 ".." 弹出上一级
            }
        } else if (part && part !== ".") {
            resolved.push(part);      // 正常路径段入栈
        }
        // 空字符串和 "." 被静默忽略
    }
    return resolved.length > 0
        ? resolved.join("/")           // 分支 A：有路径段，保留斜线
        : storageName.replace(/[\/\\]+/g, "");  // 分支 B：无路径段，去掉所有斜线
};
```

**核心执行流程分析**：

| 步骤 | 操作 | 说明 |
|---|---|---|
| 1 | `replace(/\\/g, "/")` | Windows 反斜杠统一转为正斜杠 |
| 2 | `split("/")` | 按正斜杠分割成数组，连续/前导/尾部斜线产生空字符串 |
| 3 | 遍历 `parts` | 栈算法解析路径：<br>- `part === ".."` 且 `resolved.length > 0` → `pop`<br>- `part === ".."` 且 `resolved.length === 0` → 忽略（不报错）<br>- `part` 非空且 `!== "."` → `push`<br>- 空字符串或 `"."` → 静默忽略 |
| 4 | 返回分支判断 | `resolved.length > 0` → `resolved.join("/")`<br>`resolved.length === 0` → `storageName.replace(/[\/\\]+/g, "")` |

**关键分支说明**：

- **分支 A（保留斜线）**：当解析后还有路径段时，用 `/` 连接，路径中的斜线被完整保留
- **分支 B（去斜线）**：当所有段都被 `..` 或忽略后 `resolved` 为空时，返回**原始输入去掉所有 `\` 和 `/`** 的结果（不是保留原字符，而是完全删除斜线）

**31 个输入场景的真实执行结果**（通过 Node.js 运行代码 + 对照 `/data/storage/petal/${name}/${normalizeStoragePath(storageName)}` 模板插值验证）：

| storageName 输入 | normalize 输出 | 拼接完整路径 | 是否逃逸 |
|---|---|---|---|
| `"../../../etc/passwd"` | `"etc/passwd"` | `/data/storage/petal/<name>/etc/passwd` | ❌ |
| `"../config/siyuan.json"` | `"config/siyuan.json"` | `/data/storage/petal/<name>/config/siyuan.json` | ❌ |
| `"subdir/../file.txt"` | `"file.txt"` | `/data/storage/petal/<name>/file.txt` | ❌ |
| `"a/b/c/../../d/e/f"` | `"a/d/e/f"` | `/data/storage/petal/<name>/a/d/e/f` | ❌ |
| `"a/../b/../../c"` | `"c"` | `/data/storage/petal/<name>/c` | ❌ |
| `"./data/file.json"` | `"data/file.json"` | `/data/storage/petal/<name>/data/file.json` | ❌ |
| `"/../../etc/passwd"` | `"etc/passwd"` | `/data/storage/petal/<name>/etc/passwd` | ❌ |
| `"./data/./file.json"` | `"data/file.json"` | `/data/storage/petal/<name>/data/file.json` | ❌ |
| `"data//file.txt"` | `"data/file.txt"` | `/data/storage/petal/<name>/data/file.txt` | ❌ |
| `"data/.hidden/config"` | `"data/.hidden/config"` | `/data/storage/petal/<name>/data/.hidden/config` | ❌ |
| `"data/.hidden/../file"` | `"data/file"` | `/data/storage/petal/<name>/data/file` | ❌ |
| `"\\..\\win\\path"` | `"win/path"` | `/data/storage/petal/<name>/win/path` | ❌ |
| `"data\\windows\\path"` | `"data/windows/path"` | `/data/storage/petal/<name>/data/windows/path` | ❌ |
| `"config/siyuan.json"` | `"config/siyuan.json"` | `/data/storage/petal/<name>/config/siyuan.json` | ❌ |
| `"a/"` | `"a"` | `/data/storage/petal/<name>/a` | ❌ |
| `"/a/"` | `"a"` | `/data/storage/petal/<name>/a` | ❌ |
| `"//a//"` | `"a"` | `/data/storage/petal/<name>/a` | ❌ |
| `"a/b/c/../../d"` | `"a/d"` | `/data/storage/petal/<name>/a/d` | ❌ |
| `"data/file//name.txt"` | `"data/file/name.txt"` | `/data/storage/petal/<name>/data/file/name.txt` | ❌ |
| `"..."` | `"..."` | `/data/storage/petal/<name>/...` | ❌ |
| `".config"` | `".config"` | `/data/storage/petal/<name>/.config` | ❌ |
| `"."` | `"."` | `/data/storage/petal/<name>/.` | ❌ |
| `"./"` | `"."` | `/data/storage/petal/<name>/.` | ❌ |
| `"/."` | `"."` | `/data/storage/petal/<name>/.` | ❌ |
| `"./././"` | `"..."` | `/data/storage/petal/<name>/...` | ❌ |
| `"/././."` | `"..."` | `/data/storage/petal/<name>/...` | ❌ |
| `""` | `""` | `/data/storage/petal/<name>/` | ❌ |
| `"/"` | `""` | `/data/storage/petal/<name>/` | ❌ |
| `"//"` | `""` | `/data/storage/petal/<name>/` | ❌ |
| `"///"` | `""` | `/data/storage/petal/<name>/` | ❌ |
| `"\\\\"` | `""` | `/data/storage/petal/<name>/` | ❌ |
| `"../.."` | `"...."` | `/data/storage/petal/<name>/....` | ❌ |
| `"../../.."` | `"......"` | `/data/storage/petal/<name>/......` | ❌ |
| `"data/../"` | `"data.."` | `/data/storage/petal/<name>/data..` | ❌ |
| `"/data/../"` | `"data.."` | `/data/storage/petal/<name>/data..` | ❌ |
| **`".."`** | **`".."`** | **`/data/storage/petal/<name>/..`** | ✅ 上逃逸一级 |
| **`"/../"`** | **`".."`** | **`/data/storage/petal/<name>/..`** | ✅ 上逃逸一级 |
| **`"//..//"`** | **`".."`** | **`/data/storage/petal/<name>/..`** | ✅ 上逃逸一级 |

**关键边界场景分类说明**：

| 场景 | 典型输入 | 分支 | 说明 |
|---|---|---|---|
| **空路径类** | `""`、`"/"`、`"//"`、`"///"`、`"\\\\"` | B → 空串 | 拼接后多一个尾斜线 `<name>/` |
| **当前目录类** | `"."`、`"./"`、`"/."` | B → `"."` | 拼接后 `<name>/.`（指向插件目录自身） |
| **多级点类** | `"./././"`、`"/././."`、`"..."`、`".config"` | B 或 A | 作为文件名处理，不影响目录层级 |
| **尾斜线类** | `"a/"`、`"/a/"`、`"//a//"` | A | 尾斜线被 split 吃掉，路径无尾斜 |
| **逃逸类** | `".."`、`"/../"`、`"//..//"` | B → `".."` | 唯一可逃逸一级的输入 |
| **多级 `..` 类** | `"../.."`、`"../../.."`、`"data/../"` | B | 斜线被完全删除，作为文件名处理，**不会**多级逃逸 |

**安全分析**：

1. **防穿越机制**：对 `".."` 做栈式处理，大多数穿越路径被有效压平，如 `"../../../etc/passwd"` → `"etc/passwd"`（**保留斜线**，不是之前误以为的 `"etcpasswd"`）
2. **分支 B 回退策略的真正风险**：当 `resolved.length === 0` 时，执行 `storageName.replace(/[\/\\]+/g, "")`，将所有斜线（`/` 和 `\`）**完全删除**（不是保留原字符）。因此：
   - `"../.."` → 去掉 `/` → `".." + ".."` = `"...."`（4 个点，不逃逸，作为文件名处理）
   - `"../../.."` → 去掉 `/` → `"......"`（6 个点，不逃逸）
   - `"data/../"` → 去掉 `/` → `"data" + ".."` = `"data.."`（不逃逸，作为文件名处理）
   - 只有 **`".."`、`"/../"`、`"//..//"` 等去掉斜线后恰好等于 `".."`** 的输入，才会产生最终路径 `"/data/storage/petal/<name>/.."`，可向上逃逸到 `"/data/storage/petal"` 目录
3. **不抛异常**：穿越路径被静默压平而非报错，调用方无法区分合法路径和被压平的攻击路径
4. **后端二次校验**：内核的 `/api/file/getFile` 和 `/api/file/putFile` 会校验路径合法性（基于工作空间根目录），即使前端规范化有缺陷，后端也应拦截逃逸路径

**路径处理细节验证**：

- 输入 `"../../../etc/passwd"`：
  - `parts = ["..", "..", "..", "etc", "passwd"]`
  - 3 个 `".."` 因 `resolved` 为空被忽略 → `resolved = ["etc", "passwd"]`
  - `resolved.length > 0` → 分支 A → `return "etc/passwd"` ✔️ 保留斜线
- 输入 `".."`：
  - `parts = [".."]`
  - `".."` 因 `resolved` 为空被忽略 → `resolved = []`
  - `resolved.length === 0` → 分支 B → `"..".replace(/[\/\\]+/g, "")` → `".."` ✔️ 斜线已不存在，返回原值
- 输入 `"../.."`：
  - `parts = ["..", ".."]`
  - 两个 `".."` 都被忽略 → `resolved = []`
  - 分支 B → `"../..".replace(/[\/\\]+/g, "")` → `"...."` ✔️ 斜线被完全删除，两个 `".."` 连在一起

### 3.3 暴露的 API 对象

[API.ts:340-377](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/API.ts#L340-L377) `API` 对象导出以下能力：

| 分类 | API | 说明 |
|---|---|---|
| **HTTP 通信** | `fetchPost`, `fetchSyncPost`, `fetchGet` | 直接与内核 HTTP API 交互 |
| **UI 操作** | `openTab`, `openWindow`, `openMobileFileById` | 打开文档/窗口 |
| **系统操作** | `lockScreen`, `exitSiYuan` | 锁屏/退出 |
| **编辑器** | `Protyle`, `ProtyleMethod`, `getAllEditor`, `getActiveEditor` | 编辑器实例与方法 |
| **布局** | `getAllModels`, `getAllTabs`, `getActiveTab`, `saveLayout` | 布局访问与保存 |
| **UI 组件** | `Dialog`, `Menu`, `Setting`, `Plugin` | 可实例化的 UI 类 |
| **平台** | `platformUtils`, `getFrontend`, `getBackend` | 平台检测 |
| **消息** | `showMessage`, `hideMessage`, `confirm`, `Constants` | 用户提示 |
| **其他** | `openSetting`, `openAttributePanel`, `globalCommand`, `expandDocTree`, `openEmoji` | 设置/属性/命令/表情 |

注意：
1. **HTTP API 无中间鉴权层**：插件可调用所有 `/api/*` HTTP 接口，仅依赖内核的 Session 鉴权
2. **`Plugin` 类暴露**：插件可创建其他 Plugin 实例（理论可能）
3. **`fetchPost` 无调用方标识**：内核侧无法区分请求来自插件还是用户操作

### 3.4 EventBus 隔离

[EventBus.ts:3-25](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus.ts#L3-L25)

```typescript
export class EventBus<DetailType = any> {
    private eventTarget: EventTarget;
    constructor(name = "") {
        this.eventTarget = document.appendChild(document.createComment(name));
    }
    on(type, listener)  { this.eventTarget.addEventListener(type, listener); }
    once(type, listener){ this.eventTarget.addEventListener(type, listener, {once: true}); }
    off(type, listener) { this.eventTarget.removeEventListener(type, listener); }
    emit(type, detail)  { return this.eventTarget.dispatchEvent(new CustomEvent(type, {detail, cancelable: true})); }
}
```

每个插件的 `eventTarget` 是一个独立的 DOM Comment 节点（`<!-- pluginName -->`），挂载到 `document` 下。事件通过 DOM `addEventListener/dispatchEvent` 机制隔离。

---

## 4. 接口调用与重载事件分发链路

### 4.1 插件 → 前端内部 API

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

### 4.3 内核 → 前端（WebSocket 广播机制）

#### 4.3.1 内核广播架构

[websocket.go:30-36](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/util/websocket.go#L30-L36) 内核使用 `melody` WebSocket 库，会话按 **appId → sessionId** 两级 `sync.Map` 组织：

```go
sessions     = sync.Map{}  // {appId, {sessionId, session}}
authSessions = sync.Map{}
```

每个 WebSocket 连接通过 URL 查询参数 `app`、`id`、`type` 注册到 `AddPushChan`。

#### 4.3.2 PushReloadPlugin 广播

[push_reload.go:46-90](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/push_reload.go#L46-L90) 负责广播到所有前端实例：

```go
// 优先级去重：卸载 > 禁用 > 启用/重载 > 数据变更
orderedSets := []*hashset.Set{uninstallPluginNameSet, unloadPluginNameSet, reloadPluginSet, dataChangePluginSet}
// 同一插件只保留在优先级最高的集合中
// 空 excludeApp 时: BroadcastByType("main", "reloadPlugin", ...)
// 有 excludeApp 时: BroadcastByTypeAndExcludeApp(excludeApp, "main", "reloadPlugin", ...)
```

[petal.go:74-80](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/api/petal.go#L74-L80) 当用户通过前端启用/禁用插件时，`setPetalEnabled` API 会将发起操作的 `app` ID 作为 `excludeApp` 传入，使发起端通过 HTTP 响应直接获取数据加载插件，其他端通过 WebSocket 广播重载：

```go
if enabled {
    reloadPluginSet := hashset.New(packageName)
    model.PushReloadPlugin(nil, nil, reloadPluginSet, nil, app)  // app 为发起端
} else {
    unloadPluginSet := hashset.New(packageName)
    model.PushReloadPlugin(nil, unloadPluginSet, nil, nil, app)
}
```

#### 4.3.3 BroadcastByType 实现

[websocket.go:82-92](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/util/websocket.go#L82-L92) `BroadcastByType` 遍历所有 appId 下所有 session，筛选 `type` 匹配的会话写入消息：

```go
func BroadcastByType(typ, cmd string, code int, msg string, data any) {
    typeSessions := SessionsByType(typ)
    for _, sess := range typeSessions {
        event := NewResult()
        event.Cmd = cmd; event.Code = code; event.Msg = msg; event.Data = data
        sess.Write(event.Bytes())
    }
}
```

[websocket.go:38-59](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/util/websocket.go#L38-L59) `BroadcastByTypeAndExcludeApp` 跳过指定 appId 的所有会话。

### 4.4 桌面端主窗口接收链路

[Model.ts:39-63](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/layout/Model.ts#L39-L63) WebSocket 连接 URL：

```
ws://{host}/ws?app={SIYUAN_APPID}&id={uuid}&type=main
```

[index.ts:69-76](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/index.ts#L69-L76) 桌面端主窗口 App 构造函数中 `msgCallback` 做两次分发：

**第一层：广播给所有插件**

```typescript
this.plugins.forEach((plugin) => {
    plugin.eventBus.emit("ws-main", data);
});
```

**第二层：根据 `data.cmd` 分发到具体处理函数**

[loader.ts:105-107](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/index.ts#L105-L107)（桌面端主窗口）：

```typescript
case "reloadPlugin":
    reloadPlugin(this, data.data);
    break;
```

完整桌面端主窗口 `reloadPlugin` 处理流程：

```
内核 PushReloadPlugin
  └─ WebSocket "reloadPlugin" cmd
      └─ 桌面端 App.ws.msgCallback
          ├─ 1) 所有插件 eventBus.emit("ws-main", data)  ← 插件感知原始事件
          └─ 2) switch(cmd="reloadPlugin") → reloadPlugin(app, data.data)
                ├─ unloadPlugins → uninstall(app, name, true)
                ├─ uninstallPlugins → uninstall(app, name, false)
                ├─ reloadPlugins → uninstall + loadPlugins + afterLoadPlugin
                └─ dataChangePlugins → plugin.onDataChanged()
```

### 4.5 独立窗口（Window）接收链路

[window/index.ts:32-190](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/window/index.ts#L32-L190) 独立窗口拥有自己的 `App` 类和 WebSocket 连接（`type: "main"`），`msgCallback` 逻辑与主窗口几乎一致：

```typescript
// window/index.ts:54-58
this.plugins.forEach((plugin) => {
    plugin.eventBus.emit("ws-main", data);
});
// window/index.ts:76-78
case "reloadPlugin":
    reloadPlugin(this, data.data);
    break;
```

关键区别：
- 独立窗口有独立的 `app.plugins` 数组，重载事件独立执行
- 独立窗口通过 `window.require` 的 Electron IPC 接收额外的窗口管理命令（关闭 Tab、重置样式、锁屏），见 [onWindowsMsg.ts:13-44](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/window/onWindowsMsg.ts#L13-L44)，但 **IPC 通道不处理插件重载**，插件重载只走 WebSocket
- 独立窗口加载插件代码与主窗口完全相同：`await loadPlugins(this)`

### 4.6 移动端接收链路

[mobile/index.ts:42-83](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/mobile/index.ts#L42-L83) 移动端也有独立的 `App` 类：

```typescript
// mobile/index.ts:77-80
this.plugins.forEach((plugin) => {
    plugin.eventBus.emit("ws-main", data);
});
// mobile/index.ts:80
onMessage(this, data);
```

[mobile/util/onMessage.ts:20-109](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/mobile/util/onMessage.ts#L20-L109) 移动端将 cmd 分发逻辑抽取到独立函数 `onMessage`：

```typescript
// onMessage.ts:58-59
case "reloadPlugin":
    reloadPlugin(app, data.data);
    break;
```

移动端与桌面端的区别：
- 移动端 `App` 没有 `layout`、`closedTabs` 等桌面端属性
- 移动端没有 IPC 通道，所有消息只通过 WebSocket
- 移动端的 dock 体系不同（`window.siyuan.mobile.docks`）
- 移动端信任确认默认通过（无需 `Conf.Bazaar.Trust` 检查）

### 4.7 三端分发链路对比

| 维度 | 桌面端主窗口 | 独立窗口 | 移动端 |
|---|---|---|---|
| WebSocket 连接 | `type=main` | `type=main` | `type=main` |
| `app.plugins` | 独立数组 | 独立数组 | 独立数组 |
| `msgCallback` | 内联 switch | 内联 switch | `onMessage()` 函数 |
| 插件 eventBus 广播 | ✅ `ws-main` | ✅ `ws-main` | ✅ `ws-main` |
| `reloadPlugin` 处理 | ✅ | ✅ | ✅ |
| IPC 消息 | ❌（主窗口不收 IPC） | ✅ 窗口管理命令 | ❌ 无 Electron |
| 布局恢复 | ✅ `saveLayout` | ✅ `saveLayout` | ❌ 无桌面布局 |
| `afterLoadPlugin` Dock | ✅ 三侧 Dock | ❌ `isWindow()` 跳过 | 部分支持 |

### 4.8 插件间通信

插件间没有直接的官方通信机制，只能通过：

1. 各自监听 `ws-main` 事件（WebSocket 消息的全量广播）
2. 访问 `window.siyuan` 全局对象（非隔离）
3. 自定义 DOM 事件（通过 `document.dispatchEvent`）

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
| 1 | `plugin.onunload()` | try/catch | [uninstall.ts:14-18](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L14-L18) |
| 2 | `plugin.uninstall()`（仅非 reload） | try/catch | [uninstall.ts:19-27](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L19-L27) |
| 3 | 清理 dock 配置持久化 | 保存位置/尺寸/显示状态 | [uninstall.ts:56-101](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L56-L101) |
| 4 | 关闭自定义 Tab（非 reload 时移除，reload 时 update()） | `custom.parent.parent.removeTab()` | [uninstall.ts:28-42](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L28-L42) |
| 5 | 移除顶栏图标 | `item.remove()` | [uninstall.ts:43-49](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L43-L49) |
| 6 | 移除状态栏图标 | `item.remove()` | [uninstall.ts:51-55](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L51-L55) |
| 7 | 移除 dock 按钮 | `leftDock/rightDock/bottomDock.remove(key)` | [uninstall.ts:79-97](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L79-L97) |
| 8 | 移除 EventBus Comment 节点 | `comment.remove()` | [uninstall.ts:103-109](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L103-L109) |
| 9 | 从 `app.plugins` 数组移除 | `splice(index, 1)` | [uninstall.ts:110-112](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L110-L112) |
| 10 | 移除自定义 SVG icons | `svg[data-name].remove()` | [uninstall.ts:112-114](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L112-L114) |
| 11 | 更新编辑器工具栏 | `protyle.toolbar.update()` | [uninstall.ts:114-118](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L114-L118) |
| 12 | 移除内联 style | `#pluginsStyle<name>.remove()` | [uninstall.ts:118-120](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L118-L120) |

### 5.3 错误处理策略

所有插件回调均包裹在 `try/catch` 中，仅输出 `console.error`，不中断主流程：

| 位置 | 错误点 |
|---|---|
| [loader.ts:51-55](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L51-L55) | `runCode` JS 语法/运行时错误 |
| [loader.ts:72-77](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L72-L77) | `onload()` 错误 |
| [loader.ts:136-140](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L136-L140) | `onLayoutReady()` 错误 |
| [uninstall.ts:14-18](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L14-L18) | `onunload()` 错误 |
| [uninstall.ts:20-24](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts#L20-L24) | `uninstall()` 错误 |
| [loader.ts:255-259](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L255-L259) | `onDataChanged()` 错误 |

缺陷：插件错误仅记录到 `console`，无用户级提示。

### 5.4 onDataChanged 特殊处理

[plugin/index.ts:124-134](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L124-L134)

```typescript
public onDataChanged() {
    // 兼容 3.4.1 以前同步数据使用重载插件的问题
    uninstall(this.app, this.name, true);
    loadPlugins(this.app, [this.name], false).then(() => {
        afterLoadPlugin(this);
        getAllEditor().forEach(editor => {
            editor.protyle.toolbar.update(editor.protyle);
        });
    });
}
```

默认行为是重载自身。插件可覆写此方法实现更细粒度的数据同步感知。

---

## 6. 兼容性策略

### 6.1 平台兼容性矩阵

[plugin.go:64-80](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/plugin.go#L64-L80) `plugin.json` 的 `frontends`/`backends` 数组声明支持的平台，缺省视为 "all"。

| 字段 | 可选值 | 检查时机 |
|---|---|---|
| `backends` | `windows`, `darwin`, `linux`, `docker`, `android`, `ios`, `all` | `LoadPetals` + `ParseInstalledPlugin` |
| `frontends` | `desktop-mobile`, `desktop-browser`, `mobile`, `all` | `LoadPetals` + `ParseInstalledPlugin` |

### 6.2 最低版本要求

`minAppVersion` 字段通过 `isBelowRequiredAppVersion` 使用 `golang.org/x/mod/semver` 语义化版本比较（详见 2.2.2 节）。低于要求则 `DisallowInstall=true`，在 `LoadPetals` 中自动禁用并持久化。

### 6.3 发布模式禁用

`DisabledInPublish` 字段：内核 `isPublish` 加载时自动跳过该插件。`IsReadOnlyRoleContext` 判断来自 [petal.go:42](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/api/petal.go#L42)。

### 6.4 信任确认机制

[conf/bazaar.go:19-28](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/conf/bazaar.go#L19-L28) 配置结构：

```go
type Bazaar struct {
    Trust         bool `json:"trust"`         // 是否确认信任
    PetalDisabled bool `json:"petalDisabled"`  // 是否禁用插件
}
```

[plugin.go:109-118](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L109-L118) 信任检查逻辑：

- `PetalDisabled=true`：完全禁止加载
- `Trust=false`：桌面端和 Docker 不加载；移动端（无集市模块）默认放行
- `Trust=true` + `PetalDisabled=false`：正常加载

前端启用插件时 [bazaar.ts:712-720](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/config/bazaar.ts#L712-L720) 会弹出确认对话框，首次需用户确认信任。

### 6.5 前端平台条件编译

大量使用编译条件编译（webpack DefinePlugin）：

```typescript
/// #if !MOBILE      // 桌面端专属代码
/// #else            // 移动端专属代码
/// #if BROWSER      // 浏览器端专属代码
/// #if !BROWSER     // Electron 专属代码
/// #endif
```

根据目标构建不同产物（desktop/mobile/browser），同一份 API 在不同平台有不同实现。

### 6.6 API 兼容性包装

[Menu.ts:39-60](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/Menu.ts#L39-L60) `addSeparator` 中显式兼容 3.1.24 之前版本的不同参数签名，通过 `typeof options === "object"` 和 `typeof options === "number"` 判断新旧调用方式。

### 6.7 setPetalEnabled 排除发起端

[petal.go:47-80](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/api/petal.go#L47-L80) 启用/禁用插件时，前端传入 `app: Constants.SIYUAN_APPID`，内核广播重载消息时排除发起端（通过 `BroadcastByTypeAndExcludeApp`），发起端通过 HTTP 响应直接获取 Petal 数据并调用 `loadPlugin(app, response.data)` 加载插件，避免重复加载。

---

## 7. 关键协作关系图

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Kernel (Go)                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  bazaar.ParseInstalledPlugin()                                │  │
│  │  ├─ plugin.json 解析 → Package                               │  │
│  │  ├─ IsIncompatiblePlugin(frontends/backends)                  │  │
│  │  └─ isBelowRequiredAppVersion(semver.Compare)                 │  │
│  │                                                               │  │
│  │  model.LoadPetals()                                           │  │
│  │  ├─ singleflight 去重                                         │  │
│  │  ├─ 6 层过滤 (Trust/Enabled/Compatible/Version/...)          │  │
│  │  ├─ loadCode() 读 JS/CSS/I18n                                │  │
│  │  └─ 返回 []*Petal                                            │  │
│  │                                                               │  │
│  │  model.PushReloadPlugin()                                     │  │
│  │  ├─ 优先级去重 (uninstall>unload>reload>dataChange)          │  │
│  │  ├─ BroadcastByType("main", "reloadPlugin", ...)              │  │
│  │  └─ BroadcastByTypeAndExcludeApp(发起端, "main", ...)        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                         │  HTTP / WebSocket                          │
└─────────────────────────┼─────────────────────────────────────────┘
                          │
       ┌──────────────────┼──────────────────────┐
       │                  │                      │
┌──────▼──────┐  ┌────────▼───────┐  ┌──────────▼───────┐
│  桌面端主窗口  │  │   独立窗口      │  │     移动端        │
│  App (index) │  │ App (window)   │  │ App (mobile)     │
│              │  │                │  │                  │
│ ws.msgCallback│  │ ws.msgCallback │  │ ws.msgCallback   │
│ ├─ws-main广播│  │ ├─ws-main广播  │  │ ├─ws-main广播    │
│ └─switch(cmd)│  │ └─switch(cmd)  │  │ └─onMessage(cmd) │
│   reloadPlugin│  │   reloadPlugin │  │   reloadPlugin   │
│              │  │                │  │                  │
│ IPC: 无插件  │  │ IPC: 窗口管理  │  │ 无 Electron      │
│   相关命令   │  │   (closeTab等) │  │                  │
│              │  │                │  │                  │
│ app.plugins[]│  │ app.plugins[]  │  │ app.plugins[]    │
│ 独立管理     │  │ 独立管理        │  │ 独立管理          │
└──────────────┘  └────────────────┘  └──────────────────┘
       │                  │                      │
       └──────────────────┼──────────────────────┘
                          │
┌─────────────────────────▼─────────────────────────────────────────┐
│                    共享插件加载/卸载逻辑                             │
│  loader.ts                                                        │
│  ├─ loadPlugins() → fetch /api/petal/loadPetals                   │
│  ├─ loadPluginJS()                                                │
│  │   ├─ runCode() → window.eval(IIFE + require注入)              │
│  │   ├─ requireFunc(siyuan → API)                                 │
│  │   ├─ validate extends Plugin                                   │
│  │   └─ plugin.onload()                                           │
│  ├─ afterLoadPlugin() → onLayoutReady()                           │
│  └─ reloadPlugin() → uninstall + load + afterLoad                 │
│                                                                    │
│  Plugin (基类)                                                    │
│  ├─ EventBus (DOM Comment 节点隔离)                               │
│  ├─ loadData/saveData/removeData (路径: petal/<name>/)           │
│  ├─ normalizeStoragePath (防穿越)                                  │
│  ├─ addTopBar/addStatusBar/addDock/addTab                         │
│  └─ protyleSlash / customBlockRenders                            │
│                                                                    │
│  uninstall.ts (12 步资源清理)                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## 8. 安全边界与潜在风险

### 8.1 已有的安全措施

| 措施 | 说明 | 代码位置 |
|---|---|---|
| 插件名只读锁定 | `Object.defineProperty(name, writable: false)` | [plugin/index.ts:78-81](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L78-L81) |
| 存储路径规范化 | `normalizeStoragePath` 消除 `..` 穿越 | [pathName.ts:730-742](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/util/pathName.ts#L730-L742) |
| 存储路径前缀固定 | `/data/storage/petal/<name>/` | [plugin/index.ts:268-269](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L268-L269) |
| 只读/发布模式守卫 | `saveData/removeData` 拒绝写操作 | [plugin/index.ts:280-286](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts#L280-L286) |
| 单飞并发合并 | `singleflight.Group` 防重复加载 | [plugin.go:92-104](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L92-L104) |
| 持久化一致性清理 | 磁盘插件不存在时清理 petals.json | [plugin.go:276-294](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L276-L294) |
| XSS 转义 | bazaar 展示字段 `html.EscapeString` | [package.go:134-145](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/package.go#L134-L145) |
| 版本要求拦截 | `isBelowRequiredAppVersion` + `DisallowInstall` | [installed.go:103-114](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/installed.go#L103-L114) |
| 信任确认 | 桌面端/Docker 需手动确认 `Conf.Bazaar.Trust` | [plugin.go:109-118](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L109-L118) |
| 插件继承校验 | `pluginClass.prototype instanceof Plugin` | [loader.ts:61-63](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L61-L63) |

### 8.2 潜在风险点

#### 风险 1：`window.eval` 无代码签名校验

[loader.ts:26-28](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L26-L28) 缺乏代码来源校验和签名验证。恶意或被篡改的插件可执行任意 JS。

影响：插件可访问 `window`/`document` 全局对象，可发起任意同源 HTTP 请求，桌面端可通过 `window.require` 访问 Node 能力。

#### 风险 2：`require` 原型链泄漏

[loader.ts:22-24](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L22-L24)

```typescript
requireFunc.__proto__ = window.require
```

插件可通过 `Object.getPrototypeOf(requireFunc)("child_process")` 直接调用 Electron 原生 `require`，绕过白名单访问任意 Node 模块。

#### 风险 3：`normalizeStoragePath` 回退策略的一级逃逸漏洞

[pathName.ts:730-742](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/util/pathName.ts#L730-L742) 路径穿越防护存在以下真实边界：

- 大多数穿越路径被有效压平：`"../../../etc/passwd"` → `"etc/passwd"`（保留斜线，不逃逸）
- 分支 B 去斜线导致的特殊风险：只有 `".."`、`"/../"`、`"//..//"` 等**去掉所有斜线后恰好等于 `".."`** 的输入，才能产生最终路径 `"/data/storage/petal/<name>/.."`，可向上逃逸一级到 `"/data/storage/petal"` 目录
- 多级 `..` 不会被放大逃逸：`"../.."` → `"...."`（4 个点，作为文件名），`"../../.."` → `"......"`（6 个点，作为文件名），均不逃逸
- 后端 `/api/file/getFile` 应有路径校验作为第二道防线

#### 风险 4：`window.siyuan` 全局对象完全暴露

插件可读写 `window.siyuan.config`、`window.siyuan.storage`、`window.siyuan.notebooks` 等所有全局状态。

#### 风险 5：`fetchPost` 无调用方标识

插件通过 `siyuan.fetchPost` 与内核通信，内核侧仅依赖 Session，无法区分请求来自插件还是用户操作。

#### 风险 6：`onload` 异常后插件实例残留

[loader.ts:71-77](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L71-L77)

```typescript
app.plugins.push(plugin);  // push 在 try 之前
try {
    await plugin.onload();  // 抛错后不回滚
} catch (e) { console.error(...) }  // 仅 log
```

`onload` 抛异常时，插件实例已注册到 `app.plugins` 但未完成初始化，可能导致后续 `afterLoadPlugin`、`reloadPlugin` 等操作在半初始化的插件上执行。

#### 风险 7：onunload 清理不全

若插件注册了全局监听（`window.addEventListener`、`document.addEventListener`）、定时器（`setInterval`）或动态创建未通过 Plugin API 管理的 DOM 节点，`uninstall.ts` 不会清理这些资源。

#### 风险 8：CSS 注入无隔离

[loader.ts:90-98](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts#L90-L98) CSS 直接以 `<style>` 插入 `<head>`，无作用域隔离，可覆盖全局样式。

#### 风险 9：EventBus Comment 节点可被遍历

[EventBus.ts:7](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus.ts#L7) Comment 节点挂载在 `document` 下，可通过 `document.childNodes` 遍历所有插件的 EventBus 载体。

---

## 9. 待验证问题

1. **后端文件路径校验的完整性**：内核 `/api/file/getFile` 和 `/api/file/putFile` 是否对 `path` 参数做了工作空间根目录限制？如果后端也有路径校验，则 `normalizeStoragePath` 的缺陷影响较小。

2. **多窗口同时重载的竞态**：多前端实例同时收到 `reloadPlugin` 广播时，各自独立执行 `loadPlugins` + `afterLoadPlugin`。由于每个 `App` 有独立的 `plugins` 数组，不构成跨窗口竞态；但同一窗口内 `init=true` 时 `loadPluginJS` 不 await，多插件并行初始化是否竞态访问 `app.plugins` 需要验证。

3. **独立窗口的插件生命周期**：独立窗口有独立的 `app.plugins`，但 `loadPlugins` 从同一后端 `/api/petal/loadPetals` 获取数据。如果主窗口禁用了某插件，独立窗口需要等 WebSocket 广播才能感知，存在时间窗口。

4. **`ws-main` 广播的性能影响**：所有 WebSocket 消息广播到所有插件的 `eventBus.emit("ws-main", data)`，当插件数量多且消息频繁时（如编辑事务消息），是否造成性能瓶颈。

5. **`requireFunc.__proto__` 是否可被利用**：在 Electron 环境下，插件通过 `Object.getPrototypeOf(requireFunc)` 获取原生 `require` 的完整路径需要验证。如果 Electron 启用了 `contextIsolation` 或 `nodeIntegration: false`，则 `window.require` 不可用。

6. **`normalizeStoragePath` 一级逃逸后的后端拦截**：通过 Node.js 真实运行代码已确认，输入 `".."` → `parts = [".."]` → `".."` 因 `resolved` 为空被静默忽略（**不执行 pop**） → `resolved = []` → 进入分支 B → 返回 `"..".replace(/[\/\\]+/g, "")` = `".."`。最终路径为 `"/data/storage/petal/<name>/.."`，可逃逸到 `"/data/storage/petal"` 目录。需验证后端 `/api/file/getFile` 和 `/api/file/putFile` 是否对 `..` 做了工作空间根目录限制。

7. **`DisallowInstall` 的自动禁用时机**：[plugin.go:125-128](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go#L125-L128) 当检测到 `DisallowInstall` 时会调用 `SetPetalEnabled(name, false)` 自动禁用，但这是在 `LoadPetals` 的只读加载路径中触发的副作用，是否会导致意外持久化修改？

8. **发布模式下插件 CSS/JS 是否仍被加载**：发布模式通过 `isPublish` 过滤 `LoadPetals` 返回结果，不满足条件的插件不会返回 JS/CSS 到前端，因此发布模式下不兼容/被禁用的插件代码不会执行。

---

## 10. 关键文件索引

| 文件 | 职责 |
|---|---|
| [kernel/model/plugin.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/plugin.go) | Petal 模型、持久化、代码加载、singleflight 去重 |
| [kernel/api/petal.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/api/petal.go) | HTTP API：loadPetals / setPetalEnabled（排除发起端广播） |
| [kernel/bazaar/plugin.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/plugin.go) | 已安装插件解析、平台兼容性判断 |
| [kernel/bazaar/installed.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/installed.go) | isBelowRequiredAppVersion 版本比较（semver）、集市包元数据 |
| [kernel/bazaar/package.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/bazaar/package.go) | Package 元数据结构、JSON 解析、XSS 转义 |
| [kernel/conf/bazaar.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/conf/bazaar.go) | Bazaar 配置（Trust / PetalDisabled） |
| [kernel/model/push_reload.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/model/push_reload.go) | PushReloadPlugin 广播与优先级去重 |
| [kernel/util/websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/kernel/util/websocket.go) | WebSocket 广播实现（BroadcastByType / ExcludeApp） |
| [app/src/plugin/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/index.ts) | Plugin 基类（生命周期、存储、UI 注册） |
| [app/src/plugin/loader.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/loader.ts) | 插件代码加载器（eval 执行、重载） |
| [app/src/plugin/API.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/API.ts) | 暴露给插件的 API 对象 |
| [app/src/plugin/EventBus.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/EventBus.ts) | 基于 DOM Comment 的事件总线 |
| [app/src/plugin/uninstall.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/uninstall.ts) | 12 步资源清理卸载流程 |
| [app/src/plugin/Setting.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/Setting.ts) | 插件设置面板封装 |
| [app/src/plugin/Menu.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/Menu.ts) | 菜单封装（含版本兼容） |
| [app/src/plugin/platformUtils.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/plugin/platformUtils.ts) | 平台工具函数与通知 API |
| [app/src/util/pathName.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/util/pathName.ts) | normalizeStoragePath 路径规范化 |
| [app/src/util/fetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/util/fetch.ts) | HTTP 请求封装（鉴权失效处理） |
| [app/src/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/index.ts) | 桌面端主窗口 App 类与 WebSocket 消息分发 |
| [app/src/window/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/window/index.ts) | 独立窗口 App 类与消息分发 |
| [app/src/window/onWindowsMsg.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/window/onWindowsMsg.ts) | 独立窗口 IPC 消息处理 |
| [app/src/mobile/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/mobile/index.ts) | 移动端 App 类 |
| [app/src/mobile/util/onMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/mobile/util/onMessage.ts) | 移动端 WebSocket 消息分发 |
| [app/src/layout/Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/layout/Model.ts) | WebSocket 连接管理（连接/重连/消息处理） |
| [app/src/config/bazaar.ts](file:///d:/fz/0601/solo-dogfeeding/code/287-siyuan/app/src/config/bazaar.ts) | 集市界面（含 setPetalEnabled 调用） |
