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
│  │   ├── menu/             # 移动端右侧菜单/搜索/设置面板          │
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
│      └── main/_mobile.scss # 移动端特定样式覆盖                    │
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
| 移动端入口 | [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/index.ts) | 移动端 App 类，事件绑定、启动流程 |
| 手势核心 | [touch.ts (mobile)](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts) | 滑动手势、侧栏切换、方向判定 |
| 手势共用 | [touch.ts (boot)](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/boot/globalEvent/touch.ts) | 背景图调整、iOS 长按菜单 |
| 输入法管理 | [keyboardToolbar.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/keyboardToolbar.ts) | 键盘工具栏、输入法高度检测、光标滚动 |
| 原生桥接 | [mobileAppUtil.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/mobileAppUtil.ts) | JSAndroid/JSHarmony/webkit 调用封装 |
| 框架初始化 | [initFramework.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/initFramework.ts) | 侧栏/菜单/工具栏绑定，文档打开逻辑 |
| 编辑器 | [editor.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/editor.ts) | openMobileFileById、文档切换 |
| 设备识别 | [compatibility.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/protyle/util/compatibility.ts) | 所有 isXxx 系列判断函数 |
| 运行时检测 | [functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/util/functions.ts) | isMobile()、getFrontend() |
| 菜单面板 | [menu/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/menu/index.ts) | 右侧菜单内容与事件绑定 |
| Protyle 核心 | [protyle/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/protyle/index.ts) | 编辑器核心类（双端共用） |
| 构建配置 | [webpack.mobile.js](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/webpack.mobile.js) | 移动端打包配置、宏 MOBILE=true |
| 移动端样式 | [_mobile.scss](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/assets/scss/main/_mobile.scss) | 移动端布局样式覆盖 |

---

## 2. 设备识别与响应式布局边界判定

### 2.1 多层级识别机制

SiYuan 的设备识别分为 **构建时** 和 **运行时** 两个维度，配合 **DOM 特征检测** 和 **UserAgent 检测** 完成最终判定。

#### 2.1.1 构建时分支（ifdef-loader 宏）

在 [webpack.mobile.js](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/webpack.mobile.js#L62-L68) 中定义编译宏：

```javascript
// 移动端编译时注入
{
    loader: "ifdef-loader",
    options: {
        BROWSER: true,
        MOBILE: true,    // 移动端特有宏
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

在 [functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/util/functions.ts#L7-L9) 中：

```typescript
export const isMobile = () => {
    return document.getElementById("sidebar") ? true : false;
};
```

**设计意图：** 通过 DOM 中是否存在 `#sidebar` 元素判定当前加载的是移动端 HTML 模板还是桌面端模板。此方法在共用模块（如 Protyle 核心、菜单系统）中广泛使用，无需传入编译宏。

在 [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/constants.ts#L76) 和 [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/constants.ts#L844-L877) 中被用于确定工具栏/UI 尺寸：

```typescript
public static readonly SIZE_TOOLBAR_HEIGHT: number = isMobile() ? 0 : 32;
public static readonly PROTYLE_TOOLBAR: string[] = isMobile() ? [
    "block-ref", "a", "|", "text", "strong", "em", "u", "clear", "|",
    "code", "tag", "inline-math", "inline-memo",   // 移动端：删除 s/mark/sup/sub/kbd
] : [ /* 桌面端完整列表 */ ];
```

#### 2.1.3 容器/浏览器检测

在 [compatibility.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/protyle/util/compatibility.ts#L307-L374) 中：

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

### 2.2 响应式布局的"硬边界"

SiYuan 的移动端与桌面端之间采用的是**编译入口硬切换**，而非 CSS 媒体查询的软响应式。

**边界判定特征：**

1. **无 CSS 断点**：`_mobile.scss` 中不使用 `@media` 查询做端间切换，仅使用 `100vw`、`100vh` 做屏幕自适应
2. **固定 UI 结构**：移动端 HTML 模板只包含三个固定面板：
   - `#menu`：左滑出的主菜单（`transform: translateX(0/-100vw)`）
   - `#sidebar`：右滑出的侧栏（文件树/大纲/标签等）
   - `#editor`：中央编辑区
   - `#model`：设置等模态弹窗（从底部滑入）
3. **横竖屏检测**：使用 `window.matchMedia("(orientation: portrait/landscape)")` 仅用于更新卡片尺寸和键盘高度缓存，不触发布局重排

**相关代码：**
- 横竖屏监听在 [index.ts#L134-L138](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/index.ts#L134-L138)
- 面板样式在 [_mobile.scss#L155-L179](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/assets/scss/main/_mobile.scss#L155-L179)

---

## 3. 布局切换与桌面端功能复用

### 3.1 移动端面板切换状态机

三种核心面板通过 CSS `transform` 实现位移切换，配合手势拖拽实现物理跟随效果。

```
                    ┌──────────────────────┐
                    │    初始/默认状态     │
                    │  #menu: translateX(  │
                    │   -100vw) 隐藏       │
                    │  #sidebar: translateX│
                    │   (-100vw) 隐藏      │
                    │  #model: translateY  │
                    │   (-200vh) 隐藏      │
                    └─────────┬────────────┘
                              │
        ┌─────────────────────┼──────────────────────┐
        ↓                     ↓                      ↓
  右滑手势 /            左滑手势 /            设置面板被调用
  toolbarFile 点击     toolbarMore 点击       openModel()
        │                     │                      │
        ↓                     ↓                      ↓
┌───────────────┐   ┌───────────────┐      ┌──────────────────┐
│  #sidebar 展开 │   │   #menu 展开  │      │   #model 展开     │
│ translateX(0) │   │ translateX(0) │      │ translateY(0)     │
│ 显示文件树等   │   │ 显示主菜单项  │      │ 模态遮罩 + 内容   │
└───────┬───────┘   └───────┬───────┘      └────────┬─────────┘
        │                   │                         │
        │ 左滑 / 遮罩点击    │ 右滑 / 遮罩点击         │ 关闭按钮点击
        │                   │                         │ / 左滑
        └───────────────────┼─────────────────────────┘
                            ↓
                    ┌──────────────────┐
                    │  closePanel()    │
                    │  统一重置所有    │
                    │  transform +     │
                    │  隐藏遮罩        │
                    └──────────────────┘
```

**关键函数协作：**

| 函数 | 位置 | 职责 |
|------|------|------|
| `closePanel()` | [closePanel.ts#L4-L14](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/closePanel.ts#L4-L14) | 重置 `#menu`/`#sidebar`/`#model` 的 transform，延时隐藏遮罩 `.side-mask` |
| `closeModel()` | [closePanel.ts#L16-L19](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/closePanel.ts#L16-L19) | 先调用 `activeBlur()` 收起键盘，再关闭 #model |
| `popMenu()` | [menu/index.ts#L34-L37](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/menu/index.ts#L34-L37) | 先 `activeBlur()` 再将 #menu 设为 `translateX(0)` |
| `popSide()` | [touch.ts#L27-L34](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L27-L34) | 侧栏显示/重置辅助函数 |

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

1. **构造参数差异** — [editor.ts#L61-L77](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/editor.ts#L61-L77) 传入移动端特有 ProtyleOptions：

```typescript
const protyleOptions: IProtyleOptions = {
    render: {
        scroll: true,
        title: true,
        titleShowTop: true,      // 移动端标题显示在顶部工具栏
        background: true,
        gutter: true,            // 启用块操作手柄（移动端通过长按显示）
    },
    typewriterMode: true,        // 强制打字机模式（优化光标可见性）
};
```

2. **`isMobile()` 运行时分支** — Protyle 内部大量调用 `isMobile()` 做行为微调：
   - [constants.ts#L844-L877](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/constants.ts#L844-L877) 调整工具栏按钮（移动端精简）
   - Gutter 菜单的弹出方式（移动端走 `fullscreen("bottom")`）
   - 滚动加载阈值、动画速度的微调整

3. **条件编译 `/// #if !MOBILE`** — 在 Protyle 中排除桌面端面板联动：

```typescript
// protyle/index.ts 中典型模式
/// #if !MOBILE
import {updatePanelByEditor} from "../editor/util";   // 桌面端侧栏联动
import {setPanelFocus} from "../layout/util";
/// #endif
```

#### 3.2.2 菜单/对话框系统的适配层

桌面端 `app/src/menus/` 下的菜单定义逻辑大量复用，但移动端使用不同的 **挂载方式和弹出位置**：

| 桌面端 | 移动端适配 | 所在文件 |
|--------|------------|----------|
| 鼠标坐标 popup 菜单 | `menu.fullscreen("bottom")` 从底部铺满弹出 | [touch.ts (boot)](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/boot/globalEvent/touch.ts#L72-L91) |
| 系统级 Dialog 窗口 | 统一使用 `#model` 面板承载（设置面板） | [menu/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/menu/index.ts#L261-L285) |
| Dock 面板体系（可拖拽/多窗口） | `app/src/mobile/dock/` 下封装 MobileXxx 适配层，内部复用核心逻辑 | `MobileOutline`、`MobileFiles` 等 |

#### 3.2.3 设置系统的 MobileXxx 包装

`app/src/mobile/settings/` 下的每个设置文件都是桌面端 `app/src/config/` 对应模块的**移动端适配层**：

- `settings/appearance.ts` → 复用 `config/appearance.ts` 的数据结构，只改 UI 承载
- `settings/editor.ts` → 复用 `config/editor.ts` 的配置项定义
- 实际保存操作统一调用 `/api/setting/set` 内核接口

---

## 4. 手势操作与触控事件处理

### 4.1 全局触控事件管线

移动端在 [index.ts#L175-L179](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/index.ts#L175-L179) 绑定了文档级触控监听：

```typescript
document.addEventListener("touchstart", handleTouchStart, false);
document.addEventListener("touchmove", handleTouchMove, false);
document.addEventListener("touchend", (event) => {
    handleTouchEnd(event, siyuanApp);
}, false);
```

处理流程是 **串联式管道**，每个阶段都有提前退出（return）的条件判定：

```
touchstart → handleTouchStart
  ├─→ globalTouchStart() ← 共用逻辑（背景图拖拽检测）
  │     ├─ 命中：背景图拖拽模式启动，返回 true，阻止后续
  │     └─ 未命中：返回 false，继续
  ├─ 记录触摸起点 (clientX, clientY, time)
  ├─ iOS 边缘屏蔽 (clientX < 8 || clientX > width - 8)
  └─ 重置状态变量（方向、差值等）

touchmove → handleTouchMove
  ├─ 检查白名单/黑名单（编辑中/对话框/选中文字/PDF 查看器等 → return）
  ├─ 计算 xDiff / yDiff，锁定 firstXY（区分横向/纵向滚动）
  ├─ firstXY === "y"：交给系统垂直滚动，return
  ├─ firstXY === "x"：进入横向手势处理
  │     ├─ 检测内部可横向滚动元素（表格/代码块/数据库/面包屑等）
  │     │    └─ scrollBlock = true，让位给元素内部滚动
  │     ├─ 遮罩层 z-index 提升
  │     └─ 根据滑动起点所在容器，实时 transform 面板跟随
  └─ activeBlur()，禁用编辑器 overflow

touchend → handleTouchEnd
  ├─→ globalTouchEnd() ← iOS 长按菜单判定（900ms 长按）
  ├─ 条件判定提前退出（输入框/对话框/键盘显示中/PDF 中）
  ├─ scrollBlock 为真 → closePanel()，return
  ├─ 判定手势有效性：
  │     时间 < 1000ms  OR  |xDiff| > window.innerWidth / 3
  ├─ 根据起点容器（#model/#menu/#sidebar/编辑器）和方向判定目标状态
  ├─ 反向位移检测（lastClientX）：中途反向则取消手势
  └─ 最终：popMenu() / popSide() / closePanel() / closeModel()
```

### 4.2 核心手势状态变量

在 [touch.ts (mobile)](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L16-L25) 中定义：

| 变量 | 类型 | 作用 |
|------|------|------|
| `clientX/Y` | number | 触摸起点坐标（边缘区域置 null 以禁用手势） |
| `xDiff/yDiff` | number | 位移差值（起点 - 当前点） |
| `time` | number | 起点时间戳，用于判定快滑 |
| `firstDirection` | toLeft/toRight | 首次横向位移方向，用于后续一致性检测 |
| `firstXY` | "x"/"y" | 首次主导方向，纵向滚动则锁定禁用横向手势 |
| `lastClientX` | number | 反向位移检测用，记录方向反转时的 X 坐标 |
| `scrollBlock` | boolean | 内部可横滚元素命中标记，让位给原生滚动 |
| `isFirstMove` | boolean | 首次横向有效移动标记（用于提升 z-index） |

### 4.3 手势判定的冲突解决策略

**横向手势 vs 纵向滚动（优先级：原生滚动）**

在 [touch.ts#L231-L245](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L231-L245) 中：

```typescript
if (!firstXY) {
    if (Math.abs(xDiff) > Math.abs(yDiff)) {
        firstXY = "x";  // 横向为主 → 拦截进入面板手势
    } else {
        firstXY = "y";  // 纵向为主 → 交给系统，后续 move 直接 return
    }
    // 特殊修正：面板内的"反向"滑动视为面板内滚动而非关闭
    if (firstXY === "x") {
        if ((在 #menu 中且向左滑) || (在 #sidebar 中且向右滑)) {
            firstXY = "y";  // 降级为纵向，允许垂直滚动浏览菜单项
        }
    }
}
```

**面板手势 vs 内部元素水平滚动（优先级：元素内部）**

在 [touch.ts#L266-L304](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L266-L304) 中：遍历父链查找以下元素，如果存在且仍有可滚动余量，设置 `scrollBlock = true` 让出手势：

| 元素类型 | 检测方式 |
|----------|----------|
| 代码块 | `data-type="NodeCodeBlock"` + `.code-block` 类 |
| 数据库 | `data-type="NodeAttributeView"` → tab-bar / av__scroll / av__kanban |
| 数学公式块 | `data-type="NodeMathBlock"` → 递归查找有 scrollWidth 溢出的子元素 |
| 表格 | `data-type="NodeTable"` → 第一个子元素滚动容器 |
| 列表 | `.list` 类（处理缩进层级过多导致的宽度溢出） |
| 面包屑 | `.protyle-breadcrumb__bar--nowrap` 类 |

**iOS 边缘手势屏蔽：**

```typescript
// 起点距离屏幕边缘 < 8px 视为系统边缘返回手势 → 不记录起点
if (isIPhone() || (event.touches[0].clientX > 8 && event.touches[0].clientX < window.innerWidth - 8))
```

### 4.4 iOS 长按菜单（globalTouchEnd）

在 [touch.ts (boot)](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/boot/globalEvent/touch.ts#L62-L146) 中，条件：`yDiff === undefined`（未产生移动）且 `duration > 900ms`：

- 文档树节点 → `initNavigationMenu()` / `initFileMenu()`（复用桌面端菜单生成逻辑）
- 行级元素（ref/tag/a/math/memo 等）→ 对应 `xxxMenu(protyle, target)`（全部复用桌面端）
- iPad 特判：使用精确坐标弹出 `popup({x, y})`，iPhone 一律 `fullscreen("bottom")`

### 4.5 背景图拖拽手势（globalTouchStart）

在 [touch.ts (boot)](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/boot/globalEvent/touch.ts#L20-L60) 中：如果 touchstart 命中文档题头图 `.protyle-background img`，则：

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
  │     ├─ [index.ts#L97-L103](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/index.ts#L97-L103)
  │     │    滚动输入/textarea 到视口中心
  │     └─ [index.ts#L104-L108](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/index.ts#L104-L108)
  │          canInput(element) → true → callMobileAppShowKeyboard()
  │
  └─→ 焦点拦截（HTMLElement.prototype.focus 覆写）
        [index.ts#L113-L127](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/index.ts#L113-L127)
        所有 focus() 调用 → canInput() 检测 → callMobileAppShowKeyboard()
```

`canInput()` 判定逻辑在 [mobileAppUtil.ts#L18-L33](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/mobileAppUtil.ts#L18-L33)：
- INPUT/TEXTAREA 且非 readonly
- contenteditable="true" 且所在 `.protyle-wysiwyg[data-readonly="false"]`

#### 5.1.2 键盘锁定机制（防闪烁）

在 [mobileAppUtil.ts#L3-L15](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/mobileAppUtil.ts#L3-L15) 中引入了**主动唤起后的 500ms 锁定窗口**：

```typescript
export let keyboardLockUntil = 0;

export const callMobileAppShowKeyboard = () => {
    keyboardLockUntil = Date.now() + 500;  // 锁定 500ms
    window.JSAndroid?.showKeyboard();      // 或 JSHarmony / iOS
};

export const activeBlur = () => {
    if (Date.now() < keyboardLockUntil) {  // 锁定期禁止 blur
        console.warn(`activeBlur blocked by lock ...`);
        return;  // ← 防止某些机型（如鸿蒙 Pura X）"弹起键盘后立即触发 blur → 键盘又被关闭"的死循环
    }
    window.JSAndroid?.hideKeyboard();
    hideKeyboardToolbar();
    (document.activeElement as HTMLElement).blur();
};
```

### 5.2 键盘高度检测与布局调整

由于移动端浏览器没有直接获取键盘高度的 DOM API，SiYuan 采用了 **`resize` 事件监听 + 高度差缓存** 的方案，见 [keyboardToolbar.ts#L528-L568](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/keyboardToolbar.ts#L528-L568)：

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
       │     ├─ protyle.parent.paddingBottom = 48px ← 预留底部空间，避免被遮挡
       │     ├─ 若光标位置 < 顶部 或 > (innerHeight - 42) → smooth scroll 调整滚动位置
       │     └─ 插件事件：emit("mobile-keyboard-show")
       │
       └─ 动态按钮检测：
            无选中文本 → 显示"块操作栏"（缩进/+/块类型/BIU/撤销/重做/移动）
            有选中文本 → 显示"行内样式栏"（返回/引用/链接/加粗/斜体/下划线/删除线/...）
```

#### 5.3.2 扩展模式（斜杠/样式菜单）对布局的影响

调用 `showKeyboardToolbarUtil(oldScrollTop)` 时：

```typescript
// [keyboardToolbar.ts#L288-L314](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/keyboardToolbar.ts#L288-L314)
keyboardHeight = height1 - height2 + toolHeight;   // 推算总可用高度
editor.protyle.element.parent.paddingBottom = keyboardHeight + "px";
editor.protyle.contentElement.scrollTop = oldScrollTop;  // 还原滚动位置
setTimeout(() => {  // 等待过渡动画结束后再设高度，防抖动
    toolbarElement.style.height = keyboardHeight + "px";
}, 300);
```

**核心设计要点：**
- `paddingBottom` 增加 → 文档底部产生空距，最后一块可滚动到光标可见位置
- `oldScrollTop` 预存 & 还原 → 切换扩展面板时文档不会"跳"
- `TIMEOUT_TRANSITION (300ms)` 延迟设高度 → 配合 CSS transition 产生平滑动画
- 设 `showUtil = true` + 1 秒后自动复原 → 防止 selectionchange 触发立即重绘把扩展面板冲掉

### 5.4 编辑状态与手势/滚动的互斥

为避免编辑中误触发侧栏滑动，在 `handleTouchMove` 中设置了多重互斥条件，见 [touch.ts#L202-L224](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L202-L224)：

```
handleTouchMove 提前退出条件（OR）：
  ├─ #keyboardToolbar 正在显示（.fn__none 不包含） → 编辑中禁止
  ├─ 用户有选中文本（range.toString() !== ""）且选区在编辑器内 → 选中扩选禁止
  ├─ 目标在对话框 / PDF 查看器 / 子菜单面板中 → 禁止
  └─ firstXY === "y"（纵向滚动主导）→ 禁止横滑手势
```

**编辑中滚动保护：** 当首次判定为横向手势后，立即设置编辑器 `overflow = "hidden"` 防止页面产生垂直位移抖动：

```typescript
// touch.ts handleTouchMove#L345-L347
if (window.siyuan.mobile.editor) {
    window.siyuan.mobile.editor.protyle.contentElement.style.overflow = "hidden";
}
// touchend 时还原：contentElement.style.overflow = ""
```

### 5.5 光标位置与选区管理

移动端 `touchstart` 时额外保存 `touchRange`，用于后续修正被键盘顶出视口的光标：

```typescript
// touch.ts handleTouchStart#L170-L176
if (选区存在 && 点击在编辑器下半屏 && 工具栏隐藏) {
    window.siyuan.mobile.touchRange = getRangeByPoint(x, y);
}
```

`showKeyboardToolbar()` 中如果发现光标已跳出视口（`cursorTop < 0`），则使用 `touchRange` 进行修正：

```typescript
// keyboardToolbar.ts#L444-L454
if (cursorTop < 0 && window.siyuan.mobile.touchRange) {
    // 用 touchstart 时保存的 range 来重新聚焦
    focusBlock(rangeBlockElement) 或 focusByRange(touchRange);
    cursorTop = 重新计算后的位置;
}
// 如果仍不在可视区 → contentElement.scroll({top: ..., behavior: smooth})
```

---

## 6. 关键流程时序分析

### 6.1 文档打开流程（openMobileFileById）

涉及模块：`editor.ts` + `Protyle` 核心 + `setEmpty.ts`

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

```
touchstart 触发
  │
  ├─ globalTouchStart → 背景图检测 → 命中 return
  ├─ 记录 clientX/Y/time
  ├─ iPhone 边缘 8px 过滤
  └─ 重置 firstDirection/firstXY/lastClientX

touchmove 首次触发
  │
  ├─ 编辑/选中/对话框/PDF → return
  ├─ 计算 xDiff(23px) / yDiff(5px)
  ├─ firstDirection = xDiff > 0 ? toLeft : toRight
  ├─ firstXY = |xDiff| > |yDiff| ? "x" : "y"   → 此处判定为 "x"
  ├─ 扫描内部横滚元素 → 未命中 scrollBlock = false
  ├─ isFirstMove：提升侧栏/菜单/遮罩 z-index
  ├─ 计算面板实时 transform（起点在编辑区 → 目标面板跟手移动）
  └─ 遮罩透明度 = transformMask(...)

touchmove 后续触发（持续更新 transform/opacity）
  ↓
touchend 触发
  │
  ├─ scrollBlock === true → closePanel return
  ├─ 有效性检查：(now - time < 1000ms) || (|xDiff| > width/3)
  │     ├─ 无效：closePanel()，面板回弹到关闭位
  │     └─ 有效：根据 firstDirection 和 lastClientX 判定
  │
  ├─ lastClientX 检测（中途反向滑动过）：
  │     toRight 过程中有过反向 → lastClientX != undefined
  │       → 用户犹豫 → 执行 closePanel()（取消）
  │
  └─ 最终动作：
       toRight + 起点在编辑区 → popMenu()（打开主菜单）
       toLeft  + 起点在编辑区 → popSide() （打开侧栏）
       toLeft  + 起点在 #menu → closePanel()（关闭主菜单）
       toRight + 起点在 #sidebar → closePanel()（关闭侧栏）
```

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
│  [mobile/index.ts]   │                                               │
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
│                      │   │ 右侧菜单/设置面板 │   │ WebSocket 消息 │ │
│                      │   │ + 面板打开辅助    │   │ 远端事件驱动   │ │
│                      │   └───────────────────┘   └────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 8. 潜在风险点分析

### 8.1 设备识别与边界

| 风险 | 场景 | 影响 | 相关代码 |
|------|------|------|----------|
| **UA 伪装导致误判** | 第三方浏览器/工具伪装 iPhone UA 或者 iPadOS 13+ 上报 Macintosh UA | `isIPad()` 失效 → 菜单弹出位置错误；交互行为与实际不匹配 | [compatibility.ts#L307-L318](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/protyle/util/compatibility.ts#L307-L318) |
| **容器检测依赖全局对象** | `window.JSAndroid` / `JSHarmony` / `webkit` 注入时机晚于初始化脚本 | `isInMobileApp()` 返回 false → 原生功能不可用；剪贴板/键盘异常 | [compatibility.ts#L350-L367](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/protyle/util/compatibility.ts#L350-L367) |
| **`isMobile()` 依赖 DOM 就绪时机** | 在 DOM 未构建完时调用（constants.ts 模块加载阶段）→ `#sidebar` 不存在 | 常量计算错误，如 `SIZE_TOOLBAR_HEIGHT = 32`（应为 0） | [functions.ts#L7-L9](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/util/functions.ts#L7-L9) |
| **Edge 浏览器不做 resize 键盘检测** | `!isInEdge()` 时跳过 resize 监听 → Edge 中键盘高度无法感知 | 工具栏被键盘遮挡 / 光标不可见 | [keyboardToolbar.ts#L527-L569](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/keyboardToolbar.ts#L527-L569) |

### 8.2 手势与触控

| 风险 | 场景 | 影响 | 相关代码 |
|------|------|------|----------|
| **手势状态变量为模块级全局** | 多指触控 / 同时操作两个触点时，状态被覆盖 | 手势中途状态错乱，面板卡在半开位置 | [touch.ts#L16-L25](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L16-L25) |
| **内部横滚元素检测不完整** | 新增可横滚组件（如 Timeline / Graph 迷你图）未列入白名单 | 触发面板手势而不是内部滚动，用户体验割裂 | [touch.ts#L266-L304](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L266-L304) |
| **纵向/横向首次判定后无法切换** | 用户开始纵向，之后转为大幅度横向滑动 | firstXY="y" 锁定 → 横向手势永远不会生效 | [touch.ts#L231-L245](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L231-L245) |
| **8px 边缘屏蔽的一刀切** | iPhone 全面屏的 Home Indicator 区域 / Android 全面屏手势区 | 左侧菜单"滑入打开"不灵敏，需重复尝试 | [touch.ts#L183-L193](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L183-L193) |
| **编辑器 overflow 恢复丢失** | touchmove 中设 `overflow:hidden`，但 touchend 中因某个 return 条件提前退出 | 编辑器永久无法滚动，需刷新页面 | [touch.ts#L345-L347](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L345-L347) 对比 [touch.ts#L69-L71](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L69-L71) |

### 8.3 输入法与编辑状态

| 风险 | 场景 | 影响 | 相关代码 |
|------|------|------|----------|
| **键盘锁定 500ms 硬编码** | 低端机型键盘动画 > 500ms；或者切换输入法/选词期间触发 blur | 锁定失效，键盘被意外收起；或锁定时间过长无法手动关闭 | [mobileAppUtil.ts#L3-L15](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/mobileAppUtil.ts#L3-L15) |
| **`selectionchange` 620ms 防抖** | 快速光标移动 → 工具栏显示滞后；用户连续输入中途工具栏才刷新 | 工具栏与光标位置不同步；点击工具栏按钮作用到错误选区 | [keyboardToolbar.ts#L328-L415](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/keyboardToolbar.ts#L328-L415) |
| **resize 差值 -100px 阈值** | 折叠屏展开时视口变化可能 > 100px 但非键盘事件；横屏键盘较矮 | 误判为键盘弹起 → 布局错乱 / 键盘高度被低估 | [keyboardToolbar.ts#L538-L541](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/keyboardToolbar.ts#L538-L541) |
| **`touchRange` 单实例覆盖** | A 区域 touchstart → B 区域 touchstart（多指/快速连点） | 键盘修正时使用了过期的 range，光标跳到错误位置 | [touch.ts#L170-L176](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/touch.ts#L170-L176) |
| **paddingBottom 未在销毁时清理** | 从输入态跳转到其他页面（PDF/搜索）未调用 hideKeyboardToolbar | 后续页面底部永久留白 48px | [keyboardToolbar.ts#L469-L490](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/util/keyboardToolbar.ts#L469-L490) |
| **`HTMLElement.prototype.focus` 全局覆写** | 第三方插件/代码依赖原生 focus 行为（如带 options 参数） | 兼容性问题；focus 参数丢失 | [index.ts#L113-L127](file:///d:/fz/0601/solo-dogfeeding/code/302-siyuan/app/src/mobile/index.ts#L113-L127) |

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

8. **iPhone 边缘 8px 屏蔽的左右手体验？**
   - 测试：左手持机、右手持机、左右手联合操作
   - 关注：左滑菜单、右滑侧栏的成功率和灵敏度

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

SiYuan 移动端的设计在**工程可维护性**和**双端复用率**之间做出了明确取舍：

- **高复用：** Protyle 编辑器核心、菜单/对话框生成逻辑、设置配置项定义、Protyle 插件机制 100% 共用，保证功能一致性
- **重适配：** UI 布局、手势交互、键盘处理、面板承载完全独立实现，以 `mobile/` 目录为适配层，通过条件编译和运行时检测与桌面端切割
- **强耦合：** 手势、键盘、编辑器滚动、面板切换通过全局 `window.siyuan.mobile` 共享状态紧密协作，效率高但可测试性/可调试性较弱
- **硬边界：** 双端入口级分离而非 CSS 响应式断点，避免了"一套布局适配所有设备"的复杂度，但也损失了折叠屏/平板形态下的中间态可能性

对于后续迭代，**最值得投入的改进方向**是：将 `touch.ts` 中的模块级全局变量封装为 TouchState 类、引入 `requestAnimationFrame` 节流、并建立"折叠屏/平板中间布局模式"以覆盖越来越多的混合形态设备。
