# SiYuan 移动端布局与触控适配实现分析

## 1. 整体架构概览

SiYuan 采用**双入口 + 编译条件分支 + 运行时特性检测**的三层架构来实现移动端与桌面端的分离与复用。

```
┌───────────────────────────────────────────────────────────────────┐
│                        构建层 (Webpack)                            │
│  ┌───────────────────────┐     ┌───────────────────────────┐      │
│  │ webpack.desktop.js    │     │ webpack.mobile.js         │      │
│  │ 入口: src/index.ts    │     │ 入口: src/mobile/index.ts │      │
│  │ 宏定义: BROWSER       │     │ 宏定义: MOBILE, BROWSER   │      │
│  └───────────────────────┘     └───────────────────────────┘      │
└───────────────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────────────┐
│                        源代码组织层                                 │
│  app/src/                                                          │
│  ├── index.ts              # 桌面端入口 App 类                     │
│  ├── mobile/               # 移动端专属代码目录                     │
│  │   ├── index.ts          # 移动端入口 App 类                     │
│  │   ├── editor.ts         # 移动端编辑器打开逻辑                   │
│  │   ├── dock/             # 移动端侧栏面板 (Files/Outline/Tags)  │
│  │   ├── menu/             # 移动端主菜单面板                      │
│  │   ├── settings/         # 移动端设置项组件                      │
│  │   └── util/             # 移动端工具集                          │
│  │       ├── touch.ts            # 核心手势处理                    │
│  │       ├── keyboardToolbar.ts  # 键盘工具栏/输入法管理           │
│  │       ├── mobileAppUtil.ts    # 原生 App 桥接调用               │
│  │       ├── initFramework.ts    # 移动端 UI 框架初始化            │
│  │       └── closePanel.ts       # 面板关闭统一入口                │
│  ├── protyle/              # Protyle 编辑器核心（双端共用）         │
│  ├── layout/               # 桌面端布局系统                        │
│  ├── boot/globalEvent/     # 全局事件（含 touch.ts 共用部分）      │
│  └── assets/scss/          # 样式                                  │
│      ├── base.scss         # 桌面端入口样式                        │
│      ├── mobile.scss       # 移动端入口样式                        │
│      ├── main/_mobile.scss # 移动端特定样式覆盖                    │
│      └── component/_menu.scss  # 菜单组件样式（含 fullscreen 模式） │
└───────────────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────────────┐
│                        运行时特性检测层                             │
│  ┌──────────────────────┐  ┌────────────────────────────────┐     │
│  │ DOM 特征检测         │  │ UserAgent / Container 检测     │     │
│  │ isMobile() → 检测    │  │ isInMobileApp() →             │     │
│  │ #sidebar 元素是否    │  │   isInAndroid() / iOS / Harmony│     │
│  │ 存在                 │  │ isIPhone() / isIPad()          │     │
│  └──────────────────────┘  └────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────┘
```

**关键文件索引：**

| 模块 | 文件路径 | 核心职责 |
|------|----------|----------|
| 移动端入口 | `app/src/mobile/index.ts` | 移动端 App 类，事件绑定、启动流程 |
| 手势核心 | `app/src/mobile/util/touch.ts` | 滑动手势、侧栏切换、方向判定 |
| 手势共用 | `app/src/boot/globalEvent/touch.ts` | 背景图调整、iOS 长按菜单 |
| 输入法管理 | `app/src/mobile/util/keyboardToolbar.ts` | 键盘工具栏、输入法高度检测、光标滚动 |
| 原生桥接 | `app/src/mobile/util/mobileAppUtil.ts` | JSAndroid/JSHarmony/webkit 调用封装 |
| 框架初始化 | `app/src/mobile/util/initFramework.ts` | 侧栏/菜单/工具栏绑定，文档打开逻辑 |
| 编辑器 | `app/src/mobile/editor.ts` | openMobileFileById、文档切换 |
| 设备识别 | `app/src/protyle/util/compatibility.ts` | 所有 isXxx 系列判断函数 |
| 运行时检测 | `app/src/util/functions.ts` | isMobile()、getFrontend() |
| 菜单面板 | `app/src/mobile/menu/index.ts` | 主菜单内容与事件绑定 |
| Protyle 核心 | `app/src/protyle/index.ts` | 编辑器核心类（双端共用） |
| 构建配置 | `app/webpack.mobile.js` | 移动端打包配置、宏 MOBILE=true |
| 移动端样式 | `app/src/assets/scss/main/_mobile.scss` | 移动端布局样式覆盖 |
| 菜单组件样式 | `app/src/assets/scss/component/_menu.scss` | b3-menu--fullscreen 全屏菜单样式 |
| HTML 模板 | `app/src/assets/template/mobile/index.tpl` | 移动端 DOM 结构定义 |

---

## 2. 设备识别与响应式布局边界判定

### 2.1 多层级识别机制

SiYuan 的设备识别分为 **构建时** 和 **运行时** 两个维度，配合 **DOM 特征检测** 和 **UserAgent 检测** 完成最终判定。

#### 2.1.1 构建时分支（ifdef-loader 宏）

在 `app/webpack.mobile.js` 中定义编译宏：

```javascript
{
    loader: "ifdef-loader",
    options: {
        BROWSER: true,
        MOBILE: true,
    },
}
```

源代码中使用条件编译语法：

```typescript
/// #if !MOBILE
import {getInstanceById} from "../../layout/util";  // 仅桌面端编译
import {Tab} from "../../layout/Tab";
/// #endif
```

**影响：** 条件编译实现了代码级裁剪，移动端包体不包含桌面端多窗口/布局系统等重型模块。

#### 2.1.2 运行时 DOM 特征检测

在 `app/src/util/functions.ts` 中：

```typescript
export const isMobile = () => {
    return document.getElementById("sidebar") ? true : false;
};
```

**设计意图：** 通过 DOM 中是否存在 `#sidebar` 元素判定当前加载的是移动端 HTML 模板还是桌面端模板。此方法在共用模块（如 Protyle 核心、菜单系统）中广泛使用，无需传入编译宏。

在 `app/src/constants.ts` 中被用于确定工具栏/UI 尺寸：

```typescript
public static readonly SIZE_TOOLBAR_HEIGHT: number = isMobile() ? 0 : 32;
public static readonly PROTYLE_TOOLBAR: string[] = isMobile() ? [
    "block-ref", "a", "|", "text", "strong", "em", "u", "clear", "|",
    "code", "tag", "inline-math", "inline-memo",
] : [ /* 桌面端完整列表 */ ];
```

#### 2.1.3 容器/浏览器检测

在 `app/src/protyle/util/compatibility.ts` 中：

| 函数 | 判定逻辑 | 用途 |
|------|----------|------|
| `isIPhone()` | `navigator.userAgent.indexOf("iPhone") > -1` | iPhone 特殊交互（事件名选择、双击问题） |
| `isIPad()` | `navigator.userAgent.indexOf("iPad") > -1` | iPad 菜单弹出位置（坐标 vs 全屏底部） |
| `isInAndroid()` | `container === "android" && window.JSAndroid` | 原生安卓桥接调用（剪贴板/键盘/打开文件） |
| `isInIOS()` | `container === "ios" && window.webkit?.messageHandlers` | iOS WKWebView 桥接 |
| `isInHarmony()` | `container === "harmony" && window.JSHarmony` | 鸿蒙原生桥接 |
| `isInMobileApp()` | 上述三个的并集 | 是否运行在原生 App 内嵌 WebView |
| `isSafari()` | UA 含 Safari 不含 Chrome/Chromium | Safari 浏览器特殊兼容 |
| `isChromeBrowser()` | 标准 Chrome 检测 | 移动端 PWA viewport 设置 |
| `isPhablet()` | 移动端 UA 正则匹配 | 早期的通用移动设备判定 |

`window.siyuan.config.system.container` 的取值来源：**内核 `/api/system/getConf` 返回**，由后端 Go 代码根据实际运行环境设置。

### 2.2 三种响应式机制的分工与边界

SiYuan 的响应式不是单一机制，而是 **端间硬切换、局部 CSS 断点、横竖屏 JS 监听** 三层独立机制的叠加。三者各自负责不同的粒度范围，互不重叠：

```
┌────────────────────────────────────────────────────────────────────┐
│ Layer 1: 端间硬切换（编译入口级）                                    │
│ 粒度：桌面端 vs 移动端 = 两套完全独立的 HTML/CSS/JS 包体             │
│ 机制：webpack.desktop.js  ↔ webpack.mobile.js  +  ifdef 宏裁剪      │
│ 切换时机：发布/构建时（一次性，非运行时）                             │
│ 覆盖范围：                                                         │
│   ✓ 面板体系（Dock/Layouter  ↔  #sidebar/#menu/#model）            │
│   ✓ 窗口系统（多 Tab  ↔  单 #editor）                               │
│   ✓ 整体 UI 框架（desktop.scss  ↔  mobile.scss）                   │
│   ✗ 不处理端内的屏幕尺寸差异                                        │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ Layer 2: 局部 CSS 媒体查询断点（组件/内容级）                         │
│ 粒度：端内视口宽度变化 → 特定组件的微调整                            │
│ 机制：@media (max-width: Npx) 在双端 SCSS 中共用                     │
│ 切换时机：运行时视口 resize 触发（纯 CSS，无需 JS）                  │
│ 断点与作用范围：                                                    │
│   620px — superblock 横列 ↔ 竖排 切换（mobile/_mobile.scss）        │
│   750px — 设置面板 Tab 文字隐藏、历史面板上下分栏、卡片表单换行       │
│            （util/_responsive.scss，双端共用）                      │
│   535~1199px — PDF.js 工具栏元素分级隐藏（第三方库自带断点）          │
│   767/991/1199px — Viewer.js 分级隐藏（第三方库自带断点）            │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│ Layer 3: 横竖屏 JS 监听（状态级）                                    │
│ 粒度：屏幕方向变化 → 键盘/卡片等 JS 状态更新                         │
│ 机制：window.matchMedia("(orientation:portrait)").addEventListener  │
│ 切换时机：物理方向切换 / 软件键盘弹出引发 resize                     │
│ 作用范围：                                                          │
│   ✓ 键盘高度缓存 height1/height2 的 portait/landscape 分离          │
│   ✓ 卡片视图 card__icon 图标显隐（updateCardHV）                    │
│   ✗ 不改变 DOM 结构与布局方式（仅修改 class / 状态变量）             │
└────────────────────────────────────────────────────────────────────┘
```

#### 2.2.1 Layer 1 — 端间硬切换

桌面端与移动端之间是**编译入口级硬切换**，不存在运行时"同一套布局自适应两端"。

**硬切换的判定依据：**

1. **双入口 HTML 模板**：`app/src/assets/template/desktop/index.tpl` ↔ `app/src/assets/template/mobile/index.tpl`，DOM 结构从根级开始就不同
2. **双入口 JS 启动**：`app/src/index.ts`（App 桌面类）↔ `app/src/mobile/index.ts`（App 移动类）
3. **双入口 SCSS**：`app/src/assets/scss/base.scss` ↔ `app/src/assets/scss/mobile.scss`
4. **ifdef 条件编译**：`MOBILE` 宏控制桌面端重型模块（Dock/Layouter/Tab）不进入移动端包体

**移动端核心面板（与桌面端完全不同的 DOM 结构）：**

| 面板 | HTML class | CSS 默认 transform | 展开时 transform | 隐藏方向 |
|------|-----------|-------------------|-----------------|---------|
| `#sidebar` | `side-panel fn__flex-column` | `translateX(-100vw)` | `translateX(0px)` | 隐藏于左侧 |
| `#menu` | `b3-menu b3-menu--fullscreen` | `translateX(100vw)` | `translateX(0px)` | 隐藏于右侧 |
| `#model` | `side-panel side-panel--all fn__flex-column` | `translateY(-200vh)` | `translateY(0px)` | 隐藏于上方 |
| `#editor` | — | — | 中央编辑区 | — |

> **注意**：`#menu` 与 `#sidebar`/`#model` 使用不同的 CSS 类。`#sidebar` 和 `#model` 共用 `side-panel` 基类（`position:fixed; transform:translateX(-100vw)`），而 `#menu` 使用 `b3-menu--fullscreen`（`position:fixed; left:0; right:0; width:100%`），其 `translateX(100vw)` 由 `#menu` 专属 CSS 规则在 `_mobile.scss` 中单独定义。

#### 2.2.2 Layer 2 — 局部 CSS 媒体查询断点

**确实存在 @media 断点**，只是它们作用于**端内组件级微调**而非端间切换。所有断点清单如下（从 SCSS 文件交叉验证）：

| 断点宽度 | 文件位置 | 生效端 | 影响内容 |
|---------|---------|--------|---------|
| **620px** | `app/src/assets/scss/main/_mobile.scss` | **仅移动端** | Superblock `[data-sb-layout="col"]` 横列布局 → 强制竖排：`flex-direction: column`；子元素 `margin-right: 0` |
| **750px** | `app/src/assets/scss/util/_responsive.scss` | **双端共用** | 设置面板 Tab 栏隐藏文字（只留图标）、表单标签全宽换行、历史面板左右分栏→上下分栏（左栏固定40%高）、快捷键面板键位输入框全宽居中 |
| **535px** | `app/src/assets/scss/pdf/_pdf.scss` | **双端共用** (PDF.js) | PDF.js 缩放选择器隐藏 |
| **640px** | `app/src/assets/scss/pdf/_pdf.scss` | **双端共用** (PDF.js) | PDF.js 小型视图元素及子节点全部隐藏 |
| **700px** | `app/src/assets/scss/pdf/_pdf.scss` | **双端共用** (PDF.js) | PDF.js 中型视图元素隐藏 |
| **770px** | `app/src/assets/scss/pdf/_pdf.scss` | **双端共用** (PDF.js) | PDF.js 大型视图元素隐藏 |
| **840px** | `app/src/assets/scss/pdf/_pdf.scss` | **双端共用** (PDF.js) | PDF.js 侧栏展开时不预留左侧空间（覆盖 left 属性） |
| **767px** | `app/src/assets/scss/viewerjs/_viewer.scss` | **双端共用** (Viewer.js) | Viewer.js `hide-xs-down` 类生效 |
| **991px** | `app/src/assets/scss/viewerjs/_viewer.scss` | **双端共用** (Viewer.js) | Viewer.js `hide-sm-down` 类生效 |
| **1199px** | `app/src/assets/scss/viewerjs/_viewer.scss` | **双端共用** (Viewer.js) | Viewer.js `hide-md-down` 类生效 |

**620px 断点对 superblock 横列布局的影响（最核心的业务断点）：**

默认 superblock 列布局（`app/src/assets/scss/protyle/_wysiwyg.scss`）：
```scss
.sb[data-sb-layout="col"] {
    flex-direction: row;    // 横向排列（多列并排）
    flex-wrap: wrap;
    justify-content: space-between;
    column-gap: 1.5em;
}
```

当 `max-width: 620px` 时（移动端小屏触发，`_mobile.scss`）：
```scss
.protyle-wysiwyg [data-node-id].sb[data-sb-layout="col"] {
    flex-direction: column; // 强制纵向排列（单列竖排）
    flex-wrap: initial;
    & > div {
        margin-right: 0;    // 清除列间距
    }
}
```

**含义**：用户在桌面端创建了一个"左右并排"的 superblock 列布局，在移动端（屏幕 < 620px）浏览时会**自动变为上下堆叠**，保证窄屏可读性。这是纯 CSS 驱动的运行时响应式，不需要 JS 介入。

**750px 断点（双端通用，最广覆盖的业务断点）：**

覆盖 4 类组件：
1. **设置面板 Tab 栏**：`.config__panel > .b3-tab-bar` 的 `.b3-list-item__text` 隐藏，仅保留图标，`width: auto` → 避免标签文字挤压换行
2. **设置项表单**：`.config__item > *`（输入框/按钮/下拉/滑块）全部 `width: 100%` + `margin-top: 8px` → 从"标签-控件横排"变为"标签-控件竖排"
3. **历史面板**：`.history__panel` 从左右分栏变为上下分栏（左 Tab 栏 `height: 40%`，`width: auto`，底部加 border）
4. **快捷键定义**：`.config-keymap__key` 键位输入框 `width: 100%` + 居中对齐

**设计意图**：750px 断点作用于**弹出对话框/浮动面板**（#model 设置面板、Dialog 对话框）。这些组件在桌面端可能以较小的宽度弹出（或窗口本身就窄），在移动端则要占满屏幕。通过同一个 @media 规则覆盖双端的窄屏场景，避免重复写 CSS。

#### 2.2.3 Layer 3 — 横竖屏 JS 监听

横竖屏监听完全由 **JS 驱动**，不触发 DOM 结构变化，只修改状态变量和少量 class：

**监听点 1 — index.ts 的 orientation:portrait 监听**：
```javascript
window.matchMedia("(orientation:portrait)").addEventListener("change", () => {
    updateCardHV();   // 卡片模式图标显隐
    activeBlur();     // 方向变化时收起键盘
});
```

`updateCardHV()` 实现（`app/src/card/util.ts`）：
- 竖屏：移除 `.card__action .card__icon` 的 `fn__none` → 卡片操作图标正常显示
- 横屏：添加 `fn__none` → 卡片操作图标全部隐藏（为 PDF/文档阅读留出更多横向空间）

**监听点 2 — keyboardToolbar.ts 的 resize 监听**：
```javascript
window.addEventListener("resize", () => {
    window.siyuan.mobile.size.isLandscape = matchMedia("(orientation: landscape)").matches;
    if (isLandscape) {
        if (!size.landscape) size.landscape = {height1, height2};
        // 更新 landscape.height1/height2
    } else {
        if (!size.portrait) size.portrait = {height1, height2};
        // 更新 portrait.height1/height2
    }
    // 高度差 -100px 判定键盘弹起/收起
});
```

这里 `orientation: landscape` 的作用是**将键盘高度缓存分为两套**，因为横屏和竖屏的 `window.innerHeight` 基准完全不同。同一个"弹出键盘"动作，竖屏时视口减少 ~320px，横屏时可能只减少 ~180px（键盘更矮更宽）。如果不分两套缓存，方向切换后的键盘高度估算会完全失效。

#### 2.2.4 三种响应式机制的对比总结

| 维度 | Layer 1: 端间硬切换 | Layer 2: CSS @media 断点 | Layer 3: 横竖屏 JS 监听 |
|------|-------------------|-------------------------|------------------------|
| **切换粒度** | 桌面端 ↔ 移动端 | 端内视口宽度变化 | 屏幕方向变化 |
| **实现机制** | 双 webpack 入口 + ifdef 宏 + 双 HTML 模板 | `@media (max-width: Npx)` CSS 规则 | `matchMedia("orientation")` + JS 状态/样式修改 |
| **切换时机** | 构建发布时（一次性） | 运行时 viewport resize（纯 CSS） | 物理方向旋转/键盘弹起时 resize（JS 驱动） |
| **改变范围** | DOM 根结构、包体内容、模块裁剪 | 特定组件的 CSS 属性（flex-direction / width / display） | 状态变量 + 少量 class（键盘缓存、卡片图标） |
| **核心断点** | N/A（是/否移动端二选一） | 620px / 750px（业务）；535~1199px（第三方库） | portrait ↔ landscape（无中间态） |
| **影响 superblock 列布局** | 间接（移动端才加载 `_mobile.scss` 中的 620px 规则） | **直接**：620px 时横列→竖排 | 无直接影响 |
| **影响键盘工具栏** | 直接（工具栏只有移动端才有） | 无直接 CSS 影响 | **直接**：height1/height2 缓存分离 |
| **桌面端可用** | N/A（属于桌面端侧） | 是（750px 断点桌面端窗口缩窄时也生效） | 否（`updateCardHV()` 被 `/// #if MOBILE` 包裹） |

**相关代码：**
- 侧栏/模型面板样式：`app/src/assets/scss/main/_mobile.scss` `.side-panel` 规则
- 菜单全屏样式：`app/src/assets/scss/component/_menu.scss` `.b3-menu--fullscreen` 规则
- `#menu` 专属偏移：`app/src/assets/scss/main/_mobile.scss` `#menu { transform: translateX(100vw); top: 0; }`
- superblock 620px 断点：`app/src/assets/scss/main/_mobile.scss` 第 510~518 行
- superblock 默认列布局：`app/src/assets/scss/protyle/_wysiwyg.scss` 第 277~282 行
- 750px 通用断点：`app/src/assets/scss/util/_responsive.scss` 全文
- 卡片横竖屏切换：`app/src/card/util.ts` `updateCardHV()`
- 键盘高度横竖屏分离缓存：`app/src/mobile/util/keyboardToolbar.ts` resize 监听

---

## 3. 布局切换与桌面端功能复用

### 3.1 移动端面板切换状态机

三种核心面板通过 CSS `transform` 实现位移切换，配合手势拖拽实现物理跟随效果。

```
                    ┌──────────────────────────────────┐
                    │         初始/默认状态              │
                    │  #sidebar: translateX(-100vw)     │
                    │            隐藏于屏幕左侧         │
                    │  #menu:    translateX(100vw)      │
                    │            隐藏于屏幕右侧         │
                    │  #model:   translateY(-200vh)     │
                    │            隐藏于屏幕上方         │
                    └─────────────┬────────────────────┘
                                  │
        ┌─────────────────────────┼──────────────────────┐
        ↓                         ↓                      ↓
  右滑手势 /                左滑手势 /              设置面板被调用
  toolbarFile 点击         toolbarMore 点击         openModel()
  (iconMenu)               (iconSettings)
        │                         │                      │
        ↓                         ↓                      ↓
┌───────────────┐       ┌───────────────┐      ┌──────────────────┐
│ #sidebar 展开  │       │  #menu 展开   │      │  #model 展开      │
│ translateX(0) │       │ translateX(0) │      │ translateY(0)     │
│ 左侧显示      │       │ 右侧显示      │      │ 从顶部滑入        │
│ 文件树/大纲等  │       │ 主菜单项      │      │ 模态遮罩 + 内容   │
└───────┬───────┘       └───────┬───────┘      └────────┬─────────┘
        │                       │                         │
        │ 左滑 / 遮罩点击       │ 右滑 / 遮罩点击         │ 关闭按钮点击
        │ (推向左侧关闭)        │ (推向右侧关闭)          │ / #model右滑
        └───────────────────────┼─────────────────────────┘
                                ↓
                    ┌──────────────────┐
                    │  closePanel()    │
                    │  统一将 inline   │
                    │  transform 重置  │
                    │  为空字符串      │
                    │  (还原CSS默认)   │
                    │  + 隐藏遮罩     │
                    └──────────────────┘
```

**关键函数协作：**

| 函数 | 位置 | 职责 |
|------|------|------|
| `closePanel()` | `app/src/mobile/util/closePanel.ts` | 将 `#menu`/`#sidebar`/`#model` 的 inline transform 重置为空（还原 CSS 默认值），延时隐藏遮罩 `.side-mask` |
| `closeModel()` | `app/src/mobile/util/closePanel.ts` | 先调用 `activeBlur()` 收起键盘，再关闭 #model |
| `popMenu()` | `app/src/mobile/menu/index.ts` | 先 `activeBlur()` 再将 #menu 设为 `translateX(0px)` |
| `popSide(render?)` | `app/src/mobile/util/touch.ts` | `render=true` 时模拟 `toolbarFile` 点击；`render=false` 时直接设 `sidebar.style.transform = "translateX(0px)"` |

**`closePanel()` 的还原机制：** 调用 `element.style.transform = ""` 移除 inline 样式，元素回归 CSS 默认值——sidebar 回到 `translateX(-100vw)`（左侧），menu 回到 `translateX(100vw)`（右侧），model 回到 `translateY(-200vh)`（上方）。

### 3.2 桌面端功能复用策略

SiYuan 通过以下方式实现桌面端核心功能在移动端的复用：

#### 3.2.1 Protyle 编辑器核心 100% 共用

`app/src/protyle/` 目录下的所有代码**完全共用**，包括：
- 文档渲染与编辑（WYSIWYG、渲染管线、Lute 集成）
- 光标/选区处理、撤销重做、输入法合成事件
- 工具栏（Toolbar）、滚动加载（Scroll）
- 数据库视图（AV）、代码块、表格、数学公式渲染
- 标题（Title）、题头图（Background）、面包屑（Breadcrumb）
- 行级提示（Hint）、块级菜单（Gutter）

**移动端差异化通过以下机制实现：**

1. **构造参数差异** — `app/src/mobile/editor.ts` 传入移动端特有 ProtyleOptions：

```typescript
const protyleOptions: IProtyleOptions = {
    render: {
        scroll: true,
        title: true,
        titleShowTop: true,
        background: true,
        gutter: true,
    },
    typewriterMode: true,
};
```

2. **`isMobile()` 运行时分支** — Protyle 内部大量调用 `isMobile()` 做行为微调：
   - `app/src/constants.ts` 调整工具栏按钮（移动端精简）
   - Gutter 菜单的弹出方式（移动端走 `fullscreen("bottom")`）
   - 滚动加载阈值、动画速度的微调整

3. **条件编译 `/// #if !MOBILE`** — 在 Protyle 中排除桌面端面板联动：

```typescript
// protyle/index.ts 中典型模式
/// #if !MOBILE
import {updatePanelByEditor} from "../editor/util";
import {setPanelFocus} from "../layout/util";
/// #endif
```

#### 3.2.2 菜单/对话框系统的适配层

桌面端 `app/src/menus/` 下的菜单定义逻辑大量复用，但移动端使用不同的 **挂载方式和弹出位置**：

| 桌面端 | 移动端适配 | 所在文件 |
|--------|------------|----------|
| 鼠标坐标 popup 菜单 | `menu.fullscreen("bottom")` 从底部铺满弹出 | `app/src/boot/globalEvent/touch.ts` |
| 系统级 Dialog 窗口 | 统一使用 `#model` 面板承载（设置面板） | `app/src/mobile/menu/index.ts` |
| Dock 面板体系（可拖拽/多窗口） | `app/src/mobile/dock/` 下封装 MobileXxx 适配层，内部复用核心逻辑 | `MobileOutline`、`MobileFiles` 等 |

#### 3.2.3 设置系统的 MobileXxx 包装

`app/src/mobile/settings/` 下的每个设置文件都是桌面端 `app/src/config/` 对应模块的**移动端适配层**：

- `settings/appearance.ts` → 复用 `config/appearance.ts` 的数据结构，只改 UI 承载
- `settings/editor.ts` → 复用 `config/editor.ts` 的配置项定义
- 实际保存操作统一调用 `/api/setting/set` 内核接口

---

## 4. 手势操作与触控事件处理

### 4.1 全局触控事件管线

移动端在 `app/src/mobile/index.ts` 绑定了文档级触控监听：

```typescript
document.addEventListener("touchstart", handleTouchStart, false);
document.addEventListener("touchmove", handleTouchMove, false);
document.addEventListener("touchend", (event) => {
    handleTouchEnd(event, siyuanApp);
}, false);
```

处理流程是 **串联式管道**，每个阶段都有提前退出（return）的条件判定。

**xDiff 方向约定**（关键，贯穿所有手势判定）：

```
xDiff = Math.floor(clientX - currentX)   // 起点 X - 当前 X

xDiff > 0  →  当前位置 < 起点  →  手指向左移动  →  "toLeft"
xDiff < 0  →  当前位置 > 起点  →  手指向右移动  →  "toRight"
```

### 4.1.1 touchstart 处理流程

```
touchstart → handleTouchStart
  ├─→ globalTouchStart() ← 共用逻辑（背景图拖拽检测）
  │     ├─ 命中：背景图拖拽模式启动，返回 true，阻止后续
  │     └─ 未命中：返回 false，继续
  ├─ 记录触摸起点 (clientX, clientY, time)
  ├─ iOS 边缘屏蔽（非 iPhone 且 x < 8 或 x > width-8 → clientX = null）
  └─ 重置状态变量（firstDirection/firstXY/lastClientX/scrollBlock）
```

### 4.1.2 touchmove 处理流程

```
touchmove → handleTouchMove
  ├─ 提前退出检查：
  │   ├─ clientX/clientY 未设置（边缘屏蔽后）→ return
  │   ├─ 目标是 AUDIO/对话框/键盘工具栏/PDF查看器/子菜单 → return
  │   ├─ firstXY === "y"（已锁定纵向）→ return
  │   ├─ 键盘工具栏正在显示 → return（编辑中禁止手势）
  │   └─ 编辑器内有选中文本 → return（选中扩选禁止）
  │
  ├─ 计算 xDiff / yDiff
  ├─ 确定首次方向 firstDirection（xDiff > 0 → toLeft, 否则 toRight）
  ├─ 确定首次主导轴 firstXY：
  │   ├─ |xDiff| > |yDiff| → firstXY = "x"（横向为主）
  │   └─ |xDiff| ≤ |yDiff| → firstXY = "y"（纵向为主，后续 move 直接 return）
  │
  ├─ 面板内"同向保持"降级修正：
  │   ├─ 在 #menu 中且 firstDirection === "toLeft" → firstXY = "y"
  │   │  （在菜单上左滑是"保持打开"方向，应让位给内容纵向滚动）
  │   └─ 在 #sidebar 中且 firstDirection === "toRight" → firstXY = "y"
  │      （在侧栏上右滑是"保持打开"方向，应让位给内容纵向滚动）
  │
  ├─ 反向位移检测（lastClientX）：
  │   ├─ toRight 过程中检测到反向（previousClientX > currentX）→ 记录 lastClientX
  │   └─ toLeft 过程中检测到反向（previousClientX < currentX）→ 记录 lastClientX
  │
  ├─ 横向移动处理（|xDiff| > |yDiff|）：
  │   ├─ #model 内 → return（不处理）
  │   ├─ 内部横滚元素白名单检测 → scrollBlock = true → return
  │   ├─ z-index 提升（首次移动时）
  │   ├─ 根据触摸目标容器实时 transform：
  │   │   ├─ 在 #menu 上：
  │   │   │   ├─ xDiff < 0（右滑 = 关闭方向）→ menu.transform = translateX(−xDiff)
  │   │   │   │   −xDiff 为正，menu 从 translateX(0) 向正方向推（向右关闭）
  │   │   │   └─ xDiff ≥ 0（左滑 = 保持方向）→ menu.transform = translateX(0px)
  │   │   ├─ 在 #sidebar 上：
  │   │   │   ├─ xDiff > 0（左滑 = 关闭方向）→ sidebar.transform = translateX(−xDiff)
  │   │   │   │   −xDiff 为负，sidebar 从 translateX(0) 向负方向推（向左关闭）
  │   │   │   └─ xDiff ≤ 0（右滑 = 保持方向）→ sidebar.transform = translateX(0px)
  │   │   └─ 在编辑区/其他：
  │   │       ├─ firstDirection === "toRight" → sidebar 从左侧跟手拉出：
  │   │       │   sidebar.transform = translateX(min(−xDiff − windowWidth, 0))
  │   │       │   （xDiff < 0 → −xDiff 为正，从 −windowWidth 趋向 0）
  │   │       └─ firstDirection === "toLeft" → menu 从右侧跟手拉出：
  │   │           menu.transform = translateX(max(windowWidth − xDiff, 0))
  │   │           （xDiff > 0，从 windowWidth 趋向 0）
  │   └─ activeBlur() + 编辑器 overflow = "hidden"
  └─ 遮罩透明度更新 transformMask(...)
```

### 4.1.3 touchend 处理流程

```
touchend → handleTouchEnd
  ├─→ globalTouchEnd() ← iOS 长按菜单判定（900ms 长按）
  ├─ 提前退出检查（AUDIO/对话框/子菜单/PDF/键盘工具栏）→ return
  ├─ 还原编辑器 overflow = ""
  ├─ scrollBlock === true → closePanel(), return
  ├─ 有效性检查：(time < 1000ms) OR (|xDiff| > innerWidth/3)
  │     └─ 无效 → closePanel()，面板回弹
  │
  ├─ isXScroll = |xDiff| > |yDiff|
  │
  ├─ 触摸目标在 #model 内：
  │   └─ isXScroll && toRight && 无反向 → closeModel()
  │
  ├─ 触摸目标在 #menu 内：
  │   ├─ isXScroll && toRight（右滑 = 关闭方向）：
  │   │   ├─ 有反向 lastClientX → popMenu()（取消关闭，保持打开）
  │   │   └─ 无反向 → closePanel()（确认关闭，menu 滑回右侧 +100vw）
  │   ├─ isXScroll && toLeft（左滑 = 保持方向）：
  │   │   ├─ 有反向 lastClientX → closePanel()（取消保持，执行关闭）
  │   │   └─ 无反向 → popMenu()（确认保持打开）
  │   └─ 非横向 → popMenu()（保持打开）
  │
  ├─ 触摸目标在 #sidebar 内：
  │   ├─ isXScroll && toLeft（左滑 = 关闭方向）：
  │   │   ├─ 有反向 lastClientX → popSide(false)（取消关闭，保持打开）
  │   │   └─ 无反向 → closePanel()（确认关闭，sidebar 滑回左侧 -100vw）
  │   ├─ isXScroll && toRight（右滑 = 保持方向）：
  │   │   ├─ 有反向 lastClientX → closePanel()（取消保持，执行关闭）
  │   │   └─ 无反向 → popSide(false)（确认保持打开）
  │   └─ 非横向 → popSide(false)（保持打开）
  │
  └─ 触摸目标在编辑区/其他：
      ├─ xDiff > 0（左滑）→ popMenu()（从右侧打开菜单）
      ├─ xDiff < 0（右滑）→ popSide()（从左侧打开侧栏）
      └─ 有反向 lastClientX 时一律 closePanel()
```

### 4.2 核心手势状态变量

在 `app/src/mobile/util/touch.ts` 中定义：

| 变量 | 类型 | 作用 |
|------|------|------|
| `clientX/Y` | number | 触摸起点坐标（边缘区域置 null 以禁用手势） |
| `xDiff/yDiff` | number | 位移差值（起点 - 当前点），xDiff > 0 为左滑，< 0 为右滑 |
| `time` | number | 起点时间戳，用于判定快滑（< 1000ms 有效） |
| `firstDirection` | "toLeft"/"toRight" | 首次横向位移方向，用于后续一致性检测 |
| `firstXY` | "x"/"y" | 首次主导方向轴，"y" 锁定后后续 move 直接 return |
| `lastClientX` | number | 反向位移检测：与 firstDirection 不一致时记录最后一次 clientX |
| `scrollBlock` | boolean | 内部可横滚元素命中标记，让位给原生滚动 |
| `isFirstMove` | boolean | 首次横向有效移动标记（用于提升 z-index） |
| `previousClientX` | number | 上一帧 clientX，用于逐帧检测方向反转 |

### 4.3 手势判定的冲突解决策略

#### 4.3.1 横向手势 vs 纵向滚动（优先级：原生滚动）

在 `app/src/mobile/util/touch.ts` 的 handleTouchMove 中：

```typescript
if (!firstXY) {
    if (Math.abs(xDiff) > Math.abs(yDiff)) {
        firstXY = "x";  // 横向为主 → 拦截进入面板手势
    } else {
        firstXY = "y";  // 纵向为主 → 交给系统，后续 move 直接 return
    }
    // 面板内"同向保持"降级：在已打开的面板上向打开方向滑动视为面板内纵向滚动
    if (firstXY === "x") {
        if ((hasClosestByAttribute(target, "id", "menu") && firstDirection === "toLeft") ||
            (hasClosestByAttribute(target, "id", "sidebar") && firstDirection === "toRight")) {
            firstXY = "y";  // 降级为纵向，允许滚动浏览面板内容
        }
    }
}
```

**设计意图**：当用户在已打开的 `#menu` 上左滑（toLeft，即"保持打开"方向），或已打开的 `#sidebar` 上右滑（toRight，即"保持打开"方向），这更可能是想**纵向滚动面板内容**而非执行面板手势，因此降级为纵向处理。只有"关闭方向"的滑动（menu 上右滑、sidebar 上左滑）才保留为横向面板手势。

#### 4.3.2 面板手势 vs 内部元素水平滚动（优先级：元素内部）

在 `app/src/mobile/util/touch.ts` 的 handleTouchMove 中：遍历父链查找以下元素，如果存在且仍有可滚动余量，设置 `scrollBlock = true` 让出手势：

| 元素类型 | 检测方式 | 滚动容器定位 |
|----------|----------|-------------|
| 代码块 | `data-type="NodeCodeBlock"` | `.code-block` → 第二个子元素（代码区域） |
| 数据库 | `data-type="NodeAttributeView"` | `.layout-tab-bar` / `.av__scroll` / `.av__kanban` |
| 数学公式块 | `data-type="NodeMathBlock"` | 递归查找第一个 `scrollWidth > clientWidth` 的子元素 |
| 表格 | `data-type="NodeTable"` | `.table` → 第一个子元素 |
| 列表 | `.list` 类 | 直接使用（缩进层级溢出） |
| 面包屑 | `.protyle-breadcrumb__bar--nowrap` | 直接使用 |

滚动余量判定：
```typescript
// 向右滑（xDiff < 0）：检查是否还能向左滚
xDiff < 0 && scrollElement.scrollLeft > 0
// 向左滑（xDiff > 0）：检查是否还能向右滚
xDiff > 0 && Math.ceil(scrollElement.clientWidth + scrollElement.scrollLeft) < scrollElement.scrollWidth
```

#### 4.3.3 iOS 边缘手势屏蔽

```typescript
if (isIPhone() ||
    (event.touches[0].clientX > 8 && event.touches[0].clientX < window.innerWidth - 8)) {
    clientX = event.touches[0].clientX;
    clientY = event.touches[0].clientY;
} else {
    clientX = null;  // 非 iPhone 且在 8px 边缘内 → 禁用手势
}
```

> **注意**：此逻辑在 iPhone 上**始终记录起点**（因为 `isIPhone()` 为 true 时短路 OR），只在非 iPhone 设备且起点在 8px 边缘内时才屏蔽手势。

### 4.4 iOS 长按菜单（globalTouchEnd）

在 `app/src/boot/globalEvent/touch.ts` 中，条件：`yDiff === undefined`（未产生移动）且 `duration > 900ms`：

- 文档树节点 → `initNavigationMenu()` / `initFileMenu()`（复用桌面端菜单生成逻辑）
- 行级元素（ref/tag/a/math/memo 等）→ 对应 `xxxMenu(protyle, target)`（全部复用桌面端）
- iPad 特判：使用精确坐标弹出 `popup({x, y})`，iPhone 一律 `fullscreen("bottom")`

### 4.5 背景图拖拽手势（globalTouchStart）

在 `app/src/boot/globalEvent/touch.ts` 中：如果 touchstart 命中文档题头图 `.protyle-background img`，则：

1. 临时设置 `contentElement.style.overflow = "hidden"` 阻止页面滚动
2. 绑定 `document.ontouchmove` 实时调整 `objectPosition` 百分比
3. `touchend` 时解绑所有临时事件，恢复 overflow

**此模块在桌面端/移动端共用，通过 ontouch* 属性实现。**

---

## 5. 输入法（键盘）对布局的影响与编辑状态

### 5.1 输入法触发的检测与管理

移动端的键盘处理是整个系统中 **最复杂的模块**，因为 iOS/Android/鸿蒙 的 WebView 对键盘事件的支持不一致。

#### 5.1.1 键盘显示触发链

```
用户点击编辑区
  │
  ├─→ click 事件（优先使用 click 而非 touchstart，避免键盘不收起问题）
  │     ├─ 滚动输入/textarea 到视口中心
  │     └─ canInput(element) → true → callMobileAppShowKeyboard()
  │
  └─→ 焦点拦截（HTMLElement.prototype.focus 覆写）
        所有 focus() 调用 → canInput() 检测 → callMobileAppShowKeyboard()
```

`canInput()` 判定逻辑在 `app/src/mobile/util/mobileAppUtil.ts`：
- INPUT/TEXTAREA 且非 readonly
- contenteditable="true" 且所在 `.protyle-wysiwyg[data-readonly="false"]`

#### 5.1.2 键盘锁定机制（防闪烁）

在 `app/src/mobile/util/mobileAppUtil.ts` 中引入了**主动唤起后的 500ms 锁定窗口**：

```typescript
export let keyboardLockUntil = 0;

export const callMobileAppShowKeyboard = () => {
    keyboardLockUntil = Date.now() + 500;
    window.JSAndroid?.showKeyboard();      // 或 JSHarmony / iOS
};

export const activeBlur = () => {
    if (Date.now() < keyboardLockUntil) {
        console.warn(`activeBlur blocked by lock ...`);
        return;  // ← 防止某些机型（如鸿蒙 Pura X）键盘弹起后立即 blur 的死循环
    }
    window.JSAndroid?.hideKeyboard();
    hideKeyboardToolbar();
    (document.activeElement as HTMLElement).blur();
};
```

### 5.2 键盘高度检测与布局调整

由于移动端浏览器没有直接获取键盘高度的 DOM API，SiYuan 采用了 **`resize` 事件监听 + 高度差缓存** 的方案，见 `app/src/mobile/util/keyboardToolbar.ts`：

```
初始化/横竖屏切换：
  height1 = window.innerHeight   # 记录"无键盘时"的初始高度
  height2 = window.innerHeight   # 初始同 height1

window resize 事件触发：
  if (innerHeight < height1 - 100)   # 骤降 > 100px → 判定为键盘弹起
      height2 = innerHeight          # 缓存键盘状态高度
  if (innerHeight > height1)         # 新高 > 原值 → 更新基准（应对小窗/横屏切换）
      height1 = innerHeight

实际键盘高度估算 = height1 - height2
```

横竖屏各自维护独立缓存：
```typescript
window.siyuan.mobile.size = {
    isLandscape: boolean,
    landscape: { height1, height2 },
    portrait:  { height1, height2 },
};
```

### 5.3 键盘工具栏的显示与布局联动

`#keyboardToolbar` 是移动端的核心编辑辅助条，位于屏幕**底部（键盘之上）**，包含两种形态：

| 形态 | 触发条件 | 内容 | 高度 |
|------|----------|------|------|
| 紧凑模式（默认） | `selectionchange` 事件后判定可以输入 | 缩进/添加/BIU切换/撤销重做/上下移动等一行按钮 | 48px |
| 扩展模式 | 点击"+"（斜杠插入）或"A"（文字样式）按钮 | 斜杠菜单 / 颜色字号选择面板 | 键盘高度 + 工具栏高度 |

#### 5.3.1 工具栏显示/隐藏状态机

```
document.selectionchange（防抖 620ms）
  │
  └→ renderKeyboardToolbar()
       ├─ canInput(activeElement) === false → hideKeyboardToolbar() return
       ├─ showUtil === false → hideKeyboardToolbarUtil()（收起扩展区）
       ├─ showKeyboardToolbar() → 显示紧凑模式
       │     ├─ #keyboardToolbar.fn__none → 移除
       │     ├─ protyle.parent.paddingBottom = 48px ← 预留底部空间
       │     ├─ 若光标位置 < 顶部 或 > (innerHeight - 42) → smooth scroll
       │     └─ 插件事件：emit("mobile-keyboard-show")
       │
       └─ 动态按钮检测：
            无选中文本 → 显示"块操作栏"（缩进/+/块类型/BIU/撤销/重做/移动）
            有选中文本 → 显示"行内样式栏"（返回/引用/链接/加粗/斜体/下划线/删除线/...）
```

#### 5.3.2 扩展模式（斜杠/样式菜单）对布局的影响

调用 `showKeyboardToolbarUtil(oldScrollTop)` 时：

```typescript
keyboardHeight = height1 - height2 + toolHeight;
editor.protyle.element.parent.paddingBottom = keyboardHeight + "px";
editor.protyle.contentElement.scrollTop = oldScrollTop;  // 还原滚动位置
setTimeout(() => {
    toolbarElement.style.height = keyboardHeight + "px";
}, 300);
```

**核心设计要点：**
- `paddingBottom` 增加 → 文档底部产生空距，最后一块可滚动到光标可见位置
- `oldScrollTop` 预存 & 还原 → 切换扩展面板时文档不会"跳"
- `TIMEOUT_TRANSITION (300ms)` 延迟设高度 → 配合 CSS transition 产生平滑动画
- 设 `showUtil = true` + 1 秒后自动复原 → 防止 selectionchange 触发立即重绘把扩展面板冲掉

### 5.4 编辑状态与手势/滚动的互斥

为避免编辑中误触发侧栏滑动，在 `handleTouchMove` 中设置了多重互斥条件：

```
handleTouchMove 提前退出条件（OR）：
  ├─ #keyboardToolbar 正在显示（.fn__none 不包含） → 编辑中禁止
  ├─ 用户有选中文本（range.toString() !== ""）且选区在编辑器内 → 选中扩选禁止
  ├─ 目标在对话框 / PDF 查看器 / 子菜单面板 / AUDIO 中 → 禁止
  └─ firstXY === "y"（纵向滚动主导）→ 禁止横滑手势
```

**编辑中滚动保护：** 当首次判定为横向手势后，立即设置编辑器 `overflow = "hidden"` 防止页面产生垂直位移抖动：

```typescript
// handleTouchMove 中
if (window.siyuan.mobile.editor) {
    window.siyuan.mobile.editor.protyle.contentElement.style.overflow = "hidden";
}
// handleTouchEnd 中还原
if (window.siyuan.mobile.editor) {
    window.siyuan.mobile.editor.protyle.contentElement.style.overflow = "";
}
```

### 5.5 光标位置与选区管理

移动端 `touchstart` 时额外保存 `touchRange`，用于后续修正被键盘顶出视口的光标：

```typescript
// handleTouchStart 中
if (选区存在 && 点击在编辑器下半屏 && 工具栏隐藏) {
    window.siyuan.mobile.touchRange = getRangeByPoint(event.touches[0].clientX, event.touches[0].clientY);
}
```

`showKeyboardToolbar()` 中如果发现光标已跳出视口（`cursorTop < 0`），则使用 `touchRange` 进行修正：

```typescript
if (cursorTop < 0 && window.siyuan.mobile.touchRange) {
    focusBlock(rangeBlockElement) 或 focusByRange(touchRange);
    cursorTop = 重新计算后的位置;
}
// 如果仍不在可视区 → contentElement.scroll({top: ..., behavior: smooth})
```

---

## 6. 关键流程时序分析

### 6.1 文档打开流程（openMobileFileById）

涉及模块：`app/src/mobile/editor.ts` + `Protyle` 核心 + `closePanel.ts`

```
用户点击文档（文档树/最近文档/搜索结果）
  │
  └→ openMobileFileById(app, id, action[])
       │
       ├─ 情况 A：已有编辑器 && 目标块在当前文档
       │     ├─ saveScroll(protyle) 保存滚动位置
       │     ├─ pushBack() 压入返回栈
       │     ├─ highlightById / scrollCenter 定位
       │     ├─ closePanel() 收起面板
       │     └─ /api/storage/updateRecentDocViewTime 更新浏览时间
       │
       ├─ 情况 B：已有编辑器 && 目标块不在当前文档
       │     ├─ pushBack() 压入返回栈
       │     ├─ 同 rootID：仅更新浏览时间
       │     ├─ 不同 rootID：wysiwyg.innerHTML = "" 清空 + 更新打开时间
       │     ├─ action 含 CB_GET_SCROLL → getDocByScroll() 按历史滚动打开
       │     │   否则 → /api/filetree/getDoc + onGet() 渲染
       │     └─ undo.clear() 清空撤销栈
       │
       ├─ 情况 C：无编辑器（首次启动）
       │     └─ new Protyle(app, #editor, protyleOptions) 创建编辑器实例
       │
       ├─ setEditor() → 更新工具栏 title 输入框绑定
       └─ closePanel() → 收起所有面板露出编辑区
```

### 6.2 横滑手势完整流程（从触摸到面板切换）

以下以"从编辑区右滑打开侧栏"为例，追踪完整时序：

```
touchstart 触发（起点在编辑区中央）
  │
  ├─ globalTouchStart → 非背景图 → 继续
  ├─ clientX = 300, clientY = 400, time = T0
  └─ 重置 firstDirection/firstXY/lastClientX/scrollBlock

touchmove 第1帧（手指向右移动到 x=350）
  │
  ├─ 键盘未显示 / 无选中文本 / 非对话框 → 继续
  ├─ xDiff = 300 - 350 = -50, yDiff = 0
  ├─ firstDirection = xDiff < 0 ? "toRight"
  ├─ firstXY = |−50| > |0| → "x"（横向为主）
  ├─ 不在 #menu / #sidebar 上 → 无需降级
  ├─ 无内部横滚元素命中 → scrollBlock = false
  ├─ isFirstMove → 提升 sidebar/menu/mask 的 z-index
  ├─ firstDirection === "toRight" → sidebar 跟手：
  │   sidebar.transform = translateX(min(−(−50) − 400, 0)) = translateX(−350)
  │   （sidebar 从 -100vw 位置向 0 方向移动了 50px）
  └─ mask.opacity = min(1 − (400 + (−50))/400, 0.68) = 0.125

touchmove 第N帧（手指继续右移到 x=500）
  │
  ├─ xDiff = 300 − 500 = −200
  ├─ sidebar.transform = translateX(min(200 − 400, 0)) = translateX(−200)
  │   （sidebar 已拉出 200px）
  └─ mask.opacity = min(1 − 200/400, 0.68) = 0.5

touchend 触发
  │
  ├─ scrollBlock === false → 继续
  ├─ time 检查：< 1000ms → scrollEnable = true
  ├─ isXScroll = |−200| > |0| → true
  ├─ 不在 #model / #menu / #sidebar 内 → 进入编辑区逻辑
  ├─ xDiff = −200 < 0 → 右滑
  │   ├─ 无反向 lastClientX → popSide()
  │   └─ sidebar.transform = "translateX(0px)" → 完全展开
  └─ 遮罩显示，侧栏内容可浏览
```

### 6.3 方向-面板-动作对照表

**从编辑区滑动的方向与打开面板对应关系：**

| xDiff | 方向 | 打开面板 | 面板来源方向 | CSS 默认 → 目标 |
|-------|------|----------|-------------|----------------|
| xDiff < 0 | 右滑 (toRight) | `#sidebar` (popSide) | 从左侧滑入 | `translateX(-100vw)` → `translateX(0px)` |
| xDiff > 0 | 左滑 (toLeft) | `#menu` (popMenu) | 从右侧滑入 | `translateX(100vw)` → `translateX(0px)` |

**在面板上滑动的方向与关闭/保持对应关系：**

| 当前面板 | 关闭方向 | 关闭 transform 变化 | 保持方向 | 保持状态 |
|----------|---------|--------------------|---------|---------| 
| `#sidebar` | 左滑 (toLeft, xDiff > 0) | `0px` → `translateX(−xDiff)` 推向左侧 | 右滑 (toRight, xDiff < 0) | `translateX(0px)` |
| `#menu` | 右滑 (toRight, xDiff < 0) | `0px` → `translateX(−xDiff)` 推向右侧 | 左滑 (toLeft, xDiff ≥ 0) | `translateX(0px)` |
| `#model` | 右滑 (toRight) | closeModel() | — | — |

---

## 7. 协作模块依赖关系图

```
┌────────────────────────────────────────────────────────────────────┐
│                     window.siyuan（全局状态）                       │
│  .mobile = {                                                        │
│    size: {isLandscape, landscape{h1,h2}, portrait{h1,h2}},         │
│    docks: {outline, file, bookmark, tag, backlink, inbox, ...},    │
│    editor: Protyle,      // 主编辑器                                │
│    popEditor: Protyle,   // 弹窗编辑器（可选）                       │
│    touchRange: Range,    // 键盘修正用                               │
│  }                                                                  │
│  .menus.menu / .zIndex / .backStack / .storage / .config / ...    │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
┌──────────────────────┼──────────────────────────────────────────────┐
│  移动端 App 入口     │         核心模块协作                          │
│  mobile/index.ts     │                                               │
│                      │                                               │
│  ┌────────────────┐  │   ┌───────────────────┐   ┌────────────────┐ │
│  │ 全局事件绑定    │──┼──▶│ touch.ts (mobile) │◀──│ click/keydown  │ │
│  │ (touch/click/  │  │   │ 手势处理核心       │   │ (index.ts)     │ │
│  │  key/resize)   │  │   └─────────┬─────────┘   └────────────────┘ │
│  └────────────────┘  │             │                                │
│                      │             ▼                                │
│  ┌────────────────┐  │   ┌───────────────────┐   ┌────────────────┐ │
│  │ 初始化管线     │──┼──▶│ initFramework.ts  │◀──│ Protyle 核心   │ │
│  │ getConf→lang  │  │   │ UI 框架/工具栏绑定 │   │ 双端共用       │ │
│  │ →plugins→启动 │  │   └─────────┬─────────┘   └────────────────┘ │
│  └────────────────┘  │             │                                │
│                      │             ▼                                │
│  ┌────────────────┐  │   ┌───────────────────┐   ┌────────────────┐ │
│  │ 文档打开入口   │──┼──▶│ editor.ts         │◀──│ closePanel.ts  │ │
│  │ openMobile*    │  │   │ 编辑器/切换管理   │   │ 面板统一关闭   │ │
│  └────────────────┘  │   └─────────┬─────────┘   └────────────────┘ │
│                      │             │                                │
│                      │             ▼                                │
│                      │   ┌───────────────────┐   ┌────────────────┐ │
│                      │   │ keyboardToolbar   │◀──│ mobileAppUtil  │ │
│                      │   │ 键盘工具栏/输入法 │   │ 原生桥接       │ │
│                      │   │ 高度检测/光标修正 │   │ JSAndroid/Harmony│
│                      │   └─────────┬─────────┘   └────────────────┘ │
│                      │             │                                │
│                      │             ▼                                │
│                      │   ┌───────────────────┐   ┌────────────────┐ │
│                      │   │ menu/index.ts     │◀──│ onMessage.ts   │ │
│                      │   │ 主菜单/设置面板   │   │ WebSocket 消息 │ │
│                      │   │ + 面板打开辅助    │   │ 远端事件驱动   │ │
│                      │   └───────────────────┘   └────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 8. 潜在风险点分析

### 8.1 设备识别与边界

| 风险 | 场景 | 影响 | 相关代码 |
|------|------|------|----------|
| **UA 伪装导致误判** | 第三方浏览器/工具伪装 iPhone UA 或者 iPadOS 13+ 上报 Macintosh UA | `isIPad()` 失效 → 菜单弹出位置错误；交互行为与实际不匹配 | `app/src/protyle/util/compatibility.ts` |
| **容器检测依赖全局对象** | `window.JSAndroid` / `JSHarmony` / `webkit` 注入时机晚于初始化脚本 | `isInMobileApp()` 返回 false → 原生功能不可用；剪贴板/键盘异常 | `app/src/protyle/util/compatibility.ts` |
| **`isMobile()` 依赖 DOM 就绪时机** | 在 DOM 未构建完时调用（constants.ts 模块加载阶段）→ `#sidebar` 不存在 | 常量计算错误，如 `SIZE_TOOLBAR_HEIGHT = 32`（应为 0） | `app/src/util/functions.ts` |
| **Edge 浏览器不做 resize 键盘检测** | `!isInEdge()` 时跳过 resize 监听 → Edge 中键盘高度无法感知 | 工具栏被键盘遮挡 / 光标不可见 | `app/src/mobile/util/keyboardToolbar.ts` |

### 8.2 手势与触控

| 风险 | 场景 | 影响 | 相关代码 |
|------|------|------|----------|
| **手势状态变量为模块级全局** | 多指触控 / 同时操作两个触点时，状态被覆盖 | 手势中途状态错乱，面板卡在半开位置 | `app/src/mobile/util/touch.ts` |
| **内部横滚元素检测不完整** | 新增可横滚组件（如 Timeline / Graph 迷你图）未列入白名单 | 触发面板手势而不是内部滚动，用户体验割裂 | `app/src/mobile/util/touch.ts` |
| **纵向/横向首次判定后无法切换** | 用户开始纵向，之后转为大幅度横向滑动 | firstXY="y" 锁定 → 横向手势永远不会生效 | `app/src/mobile/util/touch.ts` |
| **8px 边缘屏蔽逻辑对非 iPhone 设备** | 非 iPhone 设备在 8px 边缘内触摸 → clientX=null → 手势完全禁用 | Android 全面屏手势区附近无法触发面板滑动 | `app/src/mobile/util/touch.ts` |
| **编辑器 overflow 恢复丢失** | touchmove 中设 `overflow:hidden`，但 touchend 中因某个 return 条件提前退出 | 编辑器永久无法滚动，需刷新页面 | `app/src/mobile/util/touch.ts`（move 设 hidden / end 设 ""） |

### 8.3 输入法与编辑状态

| 风险 | 场景 | 影响 | 相关代码 |
|------|------|------|----------|
| **键盘锁定 500ms 硬编码** | 低端机型键盘动画 > 500ms；或者切换输入法/选词期间触发 blur | 锁定失效，键盘被意外收起；或锁定时间过长无法手动关闭 | `app/src/mobile/util/mobileAppUtil.ts` |
| **`selectionchange` 620ms 防抖** | 快速光标移动 → 工具栏显示滞后；用户连续输入中途工具栏才刷新 | 工具栏与光标位置不同步；点击工具栏按钮作用到错误选区 | `app/src/mobile/util/keyboardToolbar.ts` |
| **resize 差值 -100px 阈值** | 折叠屏展开时视口变化可能 > 100px 但非键盘事件；横屏键盘较矮 | 误判为键盘弹起 → 布局错乱 / 键盘高度被低估 | `app/src/mobile/util/keyboardToolbar.ts` |
| **`touchRange` 单实例覆盖** | A 区域 touchstart → B 区域 touchstart（多指/快速连点） | 键盘修正时使用了过期的 range，光标跳到错误位置 | `app/src/mobile/util/touch.ts` |
| **paddingBottom 未在销毁时清理** | 从输入态跳转到其他页面（PDF/搜索）未调用 hideKeyboardToolbar | 后续页面底部永久留白 48px | `app/src/mobile/util/keyboardToolbar.ts` |
| **`HTMLElement.prototype.focus` 全局覆写** | 第三方插件/代码依赖原生 focus 行为（如带 options 参数） | 兼容性问题；focus 参数丢失 | `app/src/mobile/index.ts` |

### 8.4 性能压力

| 风险点 | 触发条件 | 性能表现 |
|--------|----------|----------|
| **文档级 touchmove 监听无节流** | 快速滑动时每帧触发 DOM 查询 + transform 读写 | 低端 Android 机型跟手延迟；可配合 `requestAnimationFrame` 优化 |
| **`selectionchange` 620ms 内高频触发** | 光标在字符间快速移动 | 防抖队列堆积；连续输入结束后面板频繁重建 |
| **`hasClosestBlock` 遍历父链** | 每次 touchmove/end 都可能多次调用 | 深层嵌套文档时产生显著 CPU 开销 |
| **长文档中键盘工具栏展开** | `paddingBottom = keyboardHeight` 触发大规模重排 | 含大量图片/数学公式的文档会出现明显跳帧 |
| **`getSelectionPosition()` 布局抖动** | 键盘弹出后读取光标位置 → 强制 layout | 在长文档中每次 show 工具栏都会触发一次同步 layout |

### 8.5 功能复用与条件编译

| 风险 | 场景 | 影响 |
|------|------|------|
| **`/// #if !MOBILE` 遗漏清理** | 桌面端重构时修改了条件块内部变量名 | 移动端编译通过但运行时报变量 undefined（ifdef 静默） |
| **`isMobile()` 分支行为漂移** | 桌面端迭代时只测桌面，移动端分支被忽略 | 移动端表现与设计不一致（如 PROTYLE_TOOLBAR 按钮行为不一致） |
| **MobileDock 适配层不同步** | 桌面端 Dock 新增功能但移动端对应 MobileXxx 类未更新 | 功能缺失或报错 |
| **Protyle 内部假设 Tab/Layout 存在** | 新增功能时使用了桌面端 `getInstanceById`/`Tab` 但未加条件编译 | 移动端运行时 ReferenceError |

---

## 9. 需要验证的问题清单

### P0 — 核心稳定性（必须验证）

1. **键盘锁定 500ms 在全机型上是否足够？**
   - 测试：鸿蒙 Pura X / 中低端 Android / 旧款 iPhone SE
   - 关注：快速点击输入框 → 再点击空白 → 键盘是否闪烁/卡死

2. **resize 键盘检测在折叠屏展开/折叠时是否误判？**
   - 测试：Galaxy Z Fold / Mate X 系列，折叠态输入 → 中途展开
   - 关注：height1 基准是否被污染，导致后续键盘高度完全算错

3. **编辑中多点触控的手势状态是否冲突？**
   - 测试：左手滚动的同时右手点侧栏（多指并发）
   - 关注：clientX/lastClientX/firstDirection 等全局变量是否错乱

4. **Protyle 编辑器 overflow:hidden 还原的可靠性？**
   - 测试：横滑手势中途被来电/通知打断 → touchend 是否到达？
   - 关注：回来后编辑器是否无法滚动

5. **`focus` 原型方法覆写的兼容性？**
   - 测试：Protyle 内所有 focus 调用 + 常见插件使用 focus 的场景
   - 关注：是否丢失 focus options、是否有无限递归、是否影响 iframe

### P1 — 体验一致性（强烈建议验证）

6. **firstXY="y" 锁定后大幅横滑的可用性？**
   - 测试：先稍微向下滑 → 再明显横滑打开侧栏
   - 关注：用户是否需要"重新抬手指再滑"才能打开侧栏，体验是否自然

7. **内部横滚元素白名单的完整度？**
   - 测试：属性视图的看板布局、画廊视图、数据库的 tab 横向滚动、ECharts 图
   - 关注：能否正常内部滚动、不会误触发面板手势

8. **非 iPhone 设备 8px 边缘屏蔽的影响？**
   - 测试：Android 全面屏设备左右边缘触摸
   - 关注：左滑打开菜单、右滑打开侧栏的可达性

9. **selectionchange 防抖 620ms 的合理性？**
   - 测试：使用系统输入法快速选词、切词、删词
   - 关注：工具栏按钮状态与实际选区是否"跟得上"

10. **横竖屏切换 + 键盘弹出的双重布局变化？**
    - 测试：竖屏输入中（键盘已弹）→ 切横屏 → 再切回来
    - 关注：size.portrait/landscape 的 height1/height2 是否正确重置、paddingBottom 是否匹配

### P2 — 功能复用地平线（版本迭代前验证）

11. **ifdef 条件编译的桌面端/移动端双向测试？**
    - 测试：每轮桌面端 Tab/Layout 重构后跑一次移动端 smoke test
    - 关注：`ReferenceError` / 条件块内的依赖是否在移动端可解析

12. **常量 isMobile() 的初始化时序？**
    - 测试：在 HTML 中 DOMContentLoaded 前提前 import constants.ts
    - 关注：SIZE_TOOLBAR_HEIGHT、PROTYLE_TOOLBAR 是否为"桌面端值"

13. **插件 `switch-protyle`、`mobile-keyboard-show/hide` 事件的可靠性？**
    - 测试：多个插件同时监听这些事件，面板反复开关
    - 关注：事件触发时机、频率、参数是否稳定

14. **长文档（>1000 块）下键盘工具栏展开的重排成本？**
    - 测试：在超长文档中点击"+"展开斜杠菜单
    - 关注：FPS、是否跳帧、是否卡顿

15. **Edge/Opera/QQ/UC 等非主流移动浏览器的可用性？**
    - 测试：UA 中不包含标准 Chrome/Edg 标识的浏览器
    - 关注：viewport、resize 事件、键盘高度检测是否工作

---

## 10. 总结

SiYuan 移动端的设计在**工程可维护性**和**双端复用率**之间做出了明确取舍，其响应式体系呈现清晰的**三层叠加结构**：

- **高复用：** Protyle 编辑器核心、菜单/对话框生成逻辑、设置配置项定义、Protyle 插件机制 100% 共用，保证功能一致性
- **重适配：** UI 布局、手势交互、键盘处理、面板承载完全独立实现，以 `mobile/` 目录为适配层，通过条件编译和运行时检测与桌面端切割
- **强耦合：** 手势、键盘、编辑器滚动、面板切换通过全局 `window.siyuan.mobile` 共享状态紧密协作，效率高但可测试性/可调试性较弱
- **三层响应式而非单一机制：**
  - **Layer 1 端间硬切换（编译入口级）**：双 HTML 模板 + 双 JS/SCSS 入口 + ifdef 宏裁剪，实现桌面端/移动端包体级分离
  - **Layer 2 局部 CSS @media 断点（组件级）**：620px 控制 superblock 横列→竖排（移动端独有）、750px 控制设置/历史/快捷键面板的窄屏适配（双端共用），以及 535~1199px 第三方库（PDF.js/Viewer.js）自带断点
  - **Layer 3 横竖屏 JS 监听（状态级）**：`matchMedia("orientation")` 驱动键盘高度缓存分离（portrait/landscape 各存一套）和卡片图标显隐，不触发布局结构性变化
- **非对称面板布局：** `#sidebar`（左侧，`side-panel`）与 `#menu`（右侧，`b3-menu--fullscreen`）使用不同的 CSS 类和隐藏方向（`-100vw` vs `+100vw`），手势处理代码需要针对两个面板分别计算 transform
- **superblock 620px 断点是关键业务响应式**：用户在桌面端创建的多列并列 superblock，在移动端窄屏（<620px）时纯 CSS 自动变为上下堆叠，保证跨端内容可读性，此机制独立于端间硬切换，是"移动/桌面同内容不同展示"的核心手段

对于后续迭代，**最值得投入的改进方向**是：将 `touch.ts` 中的模块级全局变量封装为 TouchState 类、引入 `requestAnimationFrame` 节流、统一 `#sidebar` 与 `#menu` 的 CSS 基类以减少手势代码中的分支、建立"折叠屏/平板中间布局模式"以覆盖越来越多的混合形态设备，并探索将 620px/750px 断点整合为统一的设计 tokens（如 `--b3-breakpoint-xs: 620px` / `--b3-breakpoint-sm: 750px`）以便于响应式规则的一致性演进。
