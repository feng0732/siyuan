# SiYuan 多窗口与标签页管理分析

## 代码引用核对说明

本文档所有代码引用均经过重新核对，确保与实际代码行号和逻辑一致。所有路径均使用仓库相对路径（相对于项目根目录）。

核对范围：
- ✅ 窗口状态管理（Wnd/Tab 类及其方法）
- ✅ 跨窗口同步（IPC 消息、WebSocket 广播）
- ✅ 关闭保存逻辑（完整链路：主进程拦截 → 渲染进程处理 → 持久化 → 销毁）

---

## 一、概述

SiYuan（思源笔记）的多窗口与标签页管理系统是其界面交互的核心基础设施，承载着窗口生命周期管理、标签页集合维护、路由导航、状态持久化和跨窗口同步等关键能力。该系统采用 **前端渲染进程 + Electron 主进程 + Go 后端** 的三层架构，通过层级化的布局模型（Layout → Wnd → Tab → Model）实现复杂的分屏和多标签页管理。

### 核心文件清单

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| 布局容器 | [Layout.ts](app/src/layout/index.ts#L10-L111) | 布局容器，管理子 Wnd/Layout，支持横向/纵向分屏 |
| 窗口管理 | [Wnd.ts](app/src/layout/Wnd.ts#L56-L1090) | 窗口（分屏单元），管理标签页集合，处理拖拽、分屏 |
| 标签页 | [Tab.ts](app/src/layout/Tab.ts#L18-L242) | 标签页实例，维护头部和面板 DOM，承载 Model |
| 模型基类 | [Model.ts](app/src/layout/Model.ts#L9-L107) | WebSocket 通信基类，各类面板模型的父类 |
| 布局工具 | [util.ts](app/src/layout/util.ts) | 布局序列化/反序列化、持久化、焦点管理 |
| 标签工具 | [tabUtil.ts](app/src/layout/tabUtil.ts) | 标签页工具函数、激活态获取、批量关闭 |
| 新窗口 | [openNewWindow.ts](app/src/window/openNewWindow.ts#L22-L112) | 打开独立 Electron 窗口 |
| 窗口关闭 | [closeWin.ts](app/src/window/closeWin.ts#L5-L13) | 独立窗口关闭前的资源清理 |
| 跨窗口通信 | [onWindowsMsg.ts](app/src/window/onWindowsMsg.ts#L13-L45) | 渲染进程间消息处理 |
| 前进后退 | [backForward.ts](app/src/util/backForward.ts) | 导航栈管理 |
| 窗口初始化 | [init.ts](app/src/window/init.ts#L21-L95) | 独立窗口初始化流程 |
| IPC 消息分发 | [onGetConfig.ts](app/src/boot/onGetConfig.ts#L114-L184) | 主进程消息分发处理（关闭保存、跨窗口消息） |
| Electron 主进程 | [main.js](app/electron/main.js) | 窗口创建、IPC 通信、关闭拦截 |
| 后端配置 | [conf.go](kernel/model/conf.go#L62-L109) | 布局配置持久化存储 |

---

## 二、核心架构与类层次

### 2.1 类继承与组合关系

```
App (全局应用实例)
 └── window.siyuan.layout (布局状态根)
      ├── layout: Layout (根布局容器)
      │    └── children: (Layout | Wnd)[]
      │         ├── Layout (可嵌套分屏)
      │         └── Wnd (分屏窗口)
      │              └── children: Tab[]
      │                   └── model: Model (Editor / Asset / Graph / ...)
      ├── leftDock: Dock (左侧停靠面板)
      ├── rightDock: Dock (右侧停靠面板)
      └── bottomDock: Dock (底部停靠面板)
```

### 2.2 布局模型层级详解

**Layout 布局容器**

[Layout](app/src/layout/index.ts#L10-L111) 是一个可嵌套的弹性布局容器，核心属性：
- `direction`: `tb`（纵向）或 `lr`（横向）
- `children`: 子元素数组，元素可以是 `Layout` 或 `Wnd`
- `type`: `center`（中心区域）、`normal`（普通）、`left/right/bottom`（停靠）
- `resize`: 是否显示拖拽调整条

**Wnd 窗口（分屏单元）**

[Wnd](app/src/layout/Wnd.ts#L56-L1090) 是标签页的容器，对应一个分屏区域：
- `children: Tab[]`: 该窗口内的标签页集合
- `headersElement`: 标签栏 DOM
- `element`: 窗口根 DOM
- 支持拖拽标签排序、拖出分屏、拖拽到边缘创建新分屏

**Tab 标签页**

[Tab](app/src/layout/Tab.ts#L18-L242) 是单个标签页：
- `headElement`: 标签头 DOM
- `panelElement`: 标签内容面板 DOM
- `model: Model`: 标签对应的内容模型（编辑器、资源、图等）
- `parent: Wnd`: 所属窗口

**Model 模型**

[Model](app/src/layout/Model.ts#L9-L107) 是所有面板的基类，维护 WebSocket 连接：
- `ws: WebSocket`: 与后端的通信通道
- `reqId`: 请求 ID
- 支持自动重连（3秒间隔）

### 2.3 Model 子类体系

| 模型类 | 用途 |
|-------|------|
| `Editor` | 文档编辑器（最主要的标签类型） |
| `Asset` | 资源查看（PDF、图片等） |
| `Graph` | 关系图 |
| `Outline` | 大纲 |
| `Backlink` | 反链 |
| `Search` | 搜索 |
| `Files` | 文件树 |
| `Bookmark` | 书签 |
| `Tag` | 标签 |
| `Custom` | 自定义（插件提供） |

---

## 三、窗口状态管理

### 3.1 窗口类型

SiYuan 存在两种层面的"窗口"概念：

1. **Electron 窗口**（操作系统级窗口）
   - 主窗口：应用启动时创建
   - 独立窗口：通过拖拽标签页或菜单创建，每个窗口有独立的渲染进程

2. **Wnd 分屏窗口**（应用内分屏单元）
   - 在同一个 Electron 窗口内通过布局系统实现的多窗格
   - 支持左右分屏（`lr`）和上下分屏（`tb`）
   - 支持任意层级嵌套

### 3.2 独立窗口创建流程

**入口**：[openNewWindow.ts](app/src/window/openNewWindow.ts#L22-L36)

```typescript
export const openNewWindow = (tab: Tab, options: windowOptions = {}) => {
    const json = {};
    layoutToJSON(tab, json);
    ipcRenderer.send(Constants.SIYUAN_OPEN_WINDOW, {
        position: options.position,
        width: options.width,
        height: options.height,
        alwaysOnTop: !!options.alwaysOnTop,
        url: `${window.location.protocol}//${window.location.host}/stage/build/app/window.html?v=${Constants.SIYUAN_VERSION}&json=${encodeURIComponent(JSON.stringify([json]))}`
    });
    tab.parent.removeTab(tab.id);  // 原窗口移除标签
};
```

完整流程：
```
用户拖拽标签页到窗口外 / 调用 openNewWindow()
  ↓
layoutToJSON(tab) 序列化标签数据
  ↓
ipcRenderer.send("siyuan-open-window", {url, position, size...})
  ↓
Electron 主进程接收 [main.js#L1133](app/electron/main.js#L1133)
  ↓
创建新 BrowserWindow
  ↓
加载 window.html?json=... （URL 携带标签数据）
  ↓
原窗口移除标签 tab.parent.removeTab(tab.id)
```

**新窗口初始化**：[init.ts](app/src/window/init.ts#L21-L95)

```
加载 window.html
  ↓
new App() → 初始化全局状态
  ↓
fetchPost("/api/system/getConf") 获取配置
  ↓
从 URL query 或 sessionStorage 读取布局 JSON
  ↓
JSONToCenter(app, layoutJSON) 反序列化构建布局
  ↓
afterLayout() → 激活标签、加载插件
```

**主进程创建窗口**：[main.js#L1133-L1178](app/electron/main.js#L1133-L1178)

```javascript
ipcMain.on("siyuan-open-window", (event, data) => {
    const mainWindow = BrowserWindow.getFocusedWindow() || BrowserWindow.getAllWindows()[0];
    const mainBounds = mainWindow.getBounds();
    const mainScreen = screen.getDisplayNearestPoint({x: mainBounds.x, y: mainBounds.y});
    const win = new BrowserWindow({/* ... */});
    // ... 设置位置、大小、置顶
    win.loadURL(data.url);
    windowNavigate(win, "window");
    win.on("close", (event) => {
        if (win && !win.isDestroyed()) {
            win.webContents.send("siyuan-save-close");
        }
        event.preventDefault();
    });
    // 跨显示器时自动占满目标屏幕
    const targetScreen = screen.getDisplayNearestPoint(screen.getCursorScreenPoint());
    if (mainScreen.id !== targetScreen.id) {
        win.setBounds(targetScreen.workArea);
    }
});
```

### 3.3 窗口焦点管理

**焦点切换核心函数**：[setPanelFocus()](app/src/layout/util.ts#L34-L66)

```typescript
export const setPanelFocus = (element: Element, isSaveLayout = true) => {
    if (element.getAttribute("data-type") === "wnd") {
        const title = element.querySelector(
            '.layout-tab-bar .item--focus[data-type="tab-header"] .item__text'
        )?.textContent || "";
        setTitle(title, title ? false : true);
        element.classList.add("layout__wnd--active");
        // 更新活动时间戳
        element.querySelector(".layout-tab-bar .item--focus")
            ?.setAttribute("data-activetime", Date.now().toString());
        if (isSaveLayout) saveLayout();
    }
    // ... Dock 面板焦点处理（移除其他激活态）
};
```

**焦点切换触发场景**：
- 点击标签头 → `Wnd.switchTab()` → `setPanelFocus()`
- 点击窗口区域 → 捕获焦点事件
- 分屏切换 → 自动设置焦点

### 3.4 分屏创建（Wnd.split）

[Wnd.split()](app/src/layout/Wnd.ts#L982-L1044) 实现分屏逻辑：

```
拖拽标签页到窗口边缘
  ↓
updateDragElement() 计算放置位置（左/右/上/下）
  ↓
drop 事件触发 → Wnd.split(direction, after)
  ↓
创建新 Wnd 实例
  ↓
根据当前布局层级决定是否需要嵌套 Layout
  ↓
将被拖拽的 Tab 移动到新 Wnd
  ↓
resizeTabs() 重新计算各面板尺寸
  ↓
saveLayout() 持久化布局
```

---

## 四、标签集合管理

### 4.1 标签页添加

[Wnd.addTab()](app/src/layout/Wnd.ts#L574-L645)

```typescript
public addTab(tab: Tab, keepCursor = false, isSaveLayout = true, activeTime?: string) {
    // 1. 找到当前聚焦标签的位置（考虑固定标签）
    let oldFocusIndex = 0;
    this.children.forEach((item, index) => {
        if (item.headElement && item.headElement.classList.contains("item--focus")) {
            oldFocusIndex = index;
            let nextElement = item.headElement.nextElementSibling;
            while (nextElement && nextElement.classList.contains("item--pin")) {
                oldFocusIndex++;  // 跳过固定标签
                nextElement = nextElement.nextElementSibling;
            }
        }
    });
    
    // 2. 在聚焦标签后插入新标签
    this.children.splice(oldFocusIndex + 1, 0, tab);
    
    // 3. DOM 插入
    if (this.headersElement.childElementCount === 0) {
        this.headersElement.append(tab.headElement);
    } else {
        this.headersElement.children[oldFocusIndex].after(tab.headElement);
    }
    
    // 4. 设置关闭按钮监听
    tab.headElement.querySelector(".item__close").addEventListener("click", (event) => {
        if (tab.headElement.classList.contains("item--pin")) {
            tab.unpin();
        } else {
            tab.parent.removeTab(tab.id);
        }
        event.stopPropagation();
        event.preventDefault();
    });
    
    // 5. 设置活动时间
    tab.headElement.setAttribute("data-activetime", activeTime || (new Date()).getTime().toString());
    
    // 6. 超过最大标签数时自动关闭最久未使用的
    if (this.children.length > window.siyuan.config.fileTree.maxOpenTabCount) {
        this.removeOverCounter(isSaveLayout);
    }
    
    // 7. 持久化
    if (isSaveLayout) {
        saveLayout();
    }
}
```

### 4.2 标签页切换

[Wnd.switchTab()](app/src/layout/Wnd.ts#L467-L572)

```typescript
public switchTab(target: HTMLElement, pushBack = false, update = true, resize = true, isSaveLayout = true) {
    let currentTab: Tab;
    let isInitActive = false;
    
    // 1. 切换焦点状态
    this.children.forEach((item) => {
        if (target === item.headElement) {
            item.headElement.classList.add("item--focus");
            if (item.headElement.getAttribute("data-init-active") === "true") {
                item.headElement.removeAttribute("data-init-active");
                isInitActive = true;
            } else {
                item.headElement.setAttribute("data-activetime", (new Date()).getTime().toString());
                // 更新文档浏览时间
                if (item.model instanceof Editor) {
                    fetchPost("/api/storage/updateRecentDocViewTime", {
                        rootID: item.model.editor.protyle.block.rootID
                    });
                }
            }
            item.panelElement.classList.remove("fn__none");
            currentTab = item;
        } else {
            item.headElement?.classList.remove("item--focus");
            item.panelElement.classList.add("fn__none");
        }
    });
    
    // 2. 设置窗口焦点（反序列化时不处理）
    if (!isInitActive) {
        setPanelFocus(this.headersElement.parentElement.parentElement, isSaveLayout);
    }
    
    // 3. 懒加载 Model（首次激活时）
    if (currentTab && currentTab.headElement) {
        const initData = currentTab.headElement.getAttribute("data-initdata");
        if (initData) {
            currentTab.addModel(newModelByInitData(this.app, currentTab, JSON.parse(initData)));
            currentTab.headElement.removeAttribute("data-initdata");
            if (isSaveLayout) saveLayout();
            return;
        }
    }
    
    // 4. 特殊模型处理（Graph / Asset 焦点设置）
    if (currentTab && currentTab.model instanceof Graph) {
        currentTab.model.onGraph(false);
    }
    
    // 5. Editor 类型：更新侧边栏、保持光标位置、全屏同步
    if (currentTab && currentTab.model instanceof Editor) {
        const keepCursorId = currentTab.headElement.getAttribute("keep-cursor");
        if (keepCursorId) {
            // 在新页签中打开但不跳转，切换时需调整滚动位置
            const nodeElement = currentTab.model.editor.protyle.wysiwyg.element
                .querySelector(`[data-node-id="${keepCursorId}"]`);
            if (nodeElement) {
                scrollCenter(currentTab.model.editor.protyle, nodeElement, "start");
            } else {
                openFileById({app: this.app, id: keepCursorId, action: [...]});
            }
            currentTab.headElement.removeAttribute("keep-cursor");
        }
        if (update) {
            updatePanelByEditor({
                protyle: currentTab.model.editor.protyle,
                focus: true,
                pushBackStack: pushBack,
                reload: false,
                resize,
            });
        }
    }
    
    // 6. 持久化
    if (isSaveLayout) saveLayout();
}
```

**懒加载机制**：
未激活的标签不初始化 Model，仅保存 `data-initdata` 属性，在首次切换到该标签时才通过 `newModelByInitData()` 创建 Model 实例，显著节省内存。

### 4.3 标签页关闭

[Wnd.removeTab()](app/src/layout/Wnd.ts#L887-L903) → [removeTabAction()](app/src/layout/Wnd.ts#L772-L885)

```typescript
// 入口：检查上传状态
public removeTab(id: string, isBatchClose = false, animate = true, isSaveLayout = true) {
    for (let index = 0; index < this.children.length; index++) {
        const item = this.children[index];
        if (item.id === id) {
            if ((item.model instanceof Editor) && item.model.editor?.protyle) {
                if (item.model.editor.protyle.upload.isUploading) {
                    showMessage(window.siyuan.languages.uploading);
                    return;  // 上传中阻止关闭
                }
            }
            this.removeTabAction(id, isBatchClose, animate, isSaveLayout);
            return;
        }
    }
}

// 实际关闭逻辑
private removeTabAction = (id: string, isBatchClose = false, animate = true, isSaveLayout = true) => {
    this.children.find((item, index) => {
        if (item.id === id) {
            // 1. 存入已关闭标签栈（最多 SIZE_UNDO = 64 个）
            if (item.headElement) {
                if (window.siyuan.closedTabs.length === Constants.SIZE_UNDO) {
                    window.siyuan.closedTabs.shift();
                }
                window.siyuan.closedTabs.push({
                    tab: item,
                    time: (new Date()).toISOString(),
                    parentId: this.parent.element.getAttribute("data-id"),
                });
            }
            
            // 2. 保存滚动位置（Editor 类型）
            if (item.model instanceof Editor && item.model.editor?.protyle) {
                saveScroll(item.model.editor.protyle);
                // 更新文档关闭时间
                fetchPost("/api/storage/updateRecentDocCloseTime", {
                    rootID: item.model.editor.protyle.block.rootID,
                    scrollTop: 0  // 已在 saveScroll 中保存
                });
            }
            
            // 3. 如果是窗口最后一个标签
            if (this.children.length === 1) {
                this.destroyModel(this.children[0].model);
                this.children = [];
                if (["bottom", "left", "right"].includes(this.parent.type)) {
                    // 停靠区域：移除整个 Wnd
                    item.panelElement.remove();
                    this.remove();
                } else {
                    // 中心区域：创建空标签
                    newCenterEmptyTab(this.parent, this.app);
                }
            } else {
                // 4. 如果关闭的是当前聚焦标签，找到最近使用的标签
                if (item.headElement?.classList.contains("item--focus")) {
                    let latestHeadElement: HTMLElement;
                    Array.from(this.headersElement.children).forEach((headItem: HTMLElement) => {
                        if (headItem.getAttribute("data-id") !== id) {
                            if (!latestHeadElement) {
                                latestHeadElement = headItem;
                            } else if (headItem.getAttribute("data-activetime") > latestHeadElement.getAttribute("data-activetime")) {
                                latestHeadElement = headItem;
                            }
                        }
                    });
                    if (latestHeadElement) {
                        this.switchTab(latestHeadElement, true, true, true, false);
                    }
                }
                
                // 5. 销毁 Model 资源
                this.destroyModel(item.model);
                this.children.splice(index, 1);
            }
            
            // 6. 动画移除（200ms 过渡）
            if (animate) {
                item.panelElement.style.width = "0";
                item.headElement.style.width = "0";
                setTimeout(() => {
                    item.headElement.remove();
                    item.panelElement.remove();
                }, Constants.TIMEOUT_TRANSITION);
            } else {
                item.headElement.remove();
                item.panelElement.remove();
            }
            
            // 7. 持久化
            if (isSaveLayout) saveLayout();
            
            // 8. 清理 Electron 缓存
            webFrame.clearCache();
            
            return true;
        }
    });
};
```

### 4.4 标签页固定（Pin/Unpin）

[Tab.pin()](app/src/layout/Tab.ts#L162-L191) 和 [Tab.unpin()](app/src/layout/Tab.ts#L212-L237)

- 固定的标签始终排在标签栏前面
- 固定标签只显示图标，隐藏标题文字
- 固定标签不会被"超过最大标签数自动关闭"逻辑移除（`removeOverCounter` 方法会跳过 `item--pin`）
- 固定标签不会被"关闭其他标签"操作关闭
- 点击固定标签的关闭按钮会先 unpin 再关闭

### 4.5 批量关闭操作

[closeTabByType()](app/src/layout/tabUtil.ts#L381-L417)

支持三种批量关闭模式：
- `closeOthers`: 关闭当前标签以外的所有非固定标签
- `closeAll`: 关闭所有非固定标签
- `other`: 关闭指定的标签集合

批量关闭后统一调用 `/api/storage/batchUpdateRecentDocCloseTime` 更新关闭时间，减少网络请求次数。

---

## 五、路由跳转与导航

### 5.1 URL 参数路由

SiYuan 不使用传统的前端路由，而是通过 URL 查询参数实现页面导航：

[pathName.ts - getIdZoomInByPath()](app/src/util/pathName.ts#L29-L54)

```typescript
export const getIdZoomInByPath = () => {
    const searchParams = new URLSearchParams(window.location.search);
    // 支持三种入口：
    // 1. PWA 协议: web+siyuan://blocks/20221031001313-rk7sd0e
    // 2. Android 协议: siyuan://blocks/...
    // 3. Web URL 参数: ?id=xxx&focus=1&fullscreen=1
    
    return { id, isZoomIn };
};
```

**在布局初始化时使用**：
[util.ts - JSONToLayout()](app/src/layout/util.ts#L436-L465)

```typescript
const idZoomIn = getIdZoomInByPath();
if (idZoomIn.id) {
    openFileById({
        app,
        id: idZoomIn.id,
        action: idZoomIn.isZoomIn ? [CB_GET_ALL, CB_GET_FOCUS] : [...],
        zoomIn: idZoomIn.isZoomIn,
    });
}
```

### 5.2 前进后退导航

[backForward.ts](app/src/util/backForward.ts)

**数据结构**：
```typescript
interface IBackStack {
    position: { start: number; end: number }; // 光标偏移
    id: string;         // 块 ID
    protyle: IProtyle;  // 编辑器实例引用
    zoomId?: string;    // 缩放到的块 ID
}

window.siyuan.backStack: IBackStack[]  // 后退栈
forwardStack: IBackStack[]             // 前进栈
```

**入栈时机**：`pushBack()` 在光标位置变化时调用

**后退流程**：`goBack()`
```
弹出 backStack 顶部元素
  ↓
focusStack() 尝试定位到该位置
  ↓
如果 protyle 元素已不存在（标签被关闭）：
  - 检查块是否还存在
  - 存在则重新打开标签
  - 不存在则继续弹出下一个
  ↓
成功则 push 到 forwardStack
```

**关键特性**：
- 栈深度限制：`Constants.SIZE_UNDO = 64`
- 相同块连续移动不重复入栈，只更新位置
- 新的导航动作会清空 forwardStack
- 支持标签页关闭后的恢复（重新打开）

### 5.3 哈希状态（Hash）

[setModelsHash()](app/src/window/setHeader.ts#L49-L70)

窗口将所有已打开文档的 rootID 以零宽空格（`ZWSP`）分隔存入 URL hash，用于：
- 页面刷新后快速识别打开的文档
- 窗口状态的轻量级标识

---

## 六、数据持久化

### 6.1 布局序列化

[layoutToJSON()](app/src/layout/util.ts#L474-L606)

将布局树递归序列化为 JSON，每个节点包含：

| 节点类型 | 序列化字段 |
|---------|-----------|
| Layout | `instance: "Layout"`, `direction`, `size`, `resize`, `type`, `children[]` |
| Wnd | `instance: "Wnd"`, `resize`, `width`, `height`, `children[]` |
| Tab | `instance: "Tab"`, `title`, `icon`, `docIcon`, `pin`, `active`, `activeTime`, `children` |
| Editor | `instance: "Editor"`, `notebookId`, `blockId`, `rootId`, `mode`, `action` |
| Asset | `instance: "Asset"`, `path`, `page` |
| 其他模型 | 各类型特定字段 |

### 6.2 布局反序列化

[JSONToCenter()](app/src/layout/util.ts#L254-L386)

递归构建布局树：
1. Layout 节点 → 创建 Layout 实例
2. Wnd 节点 → 创建 Wnd 实例
3. Tab 节点 → 创建 Tab 实例（Model 延迟初始化）
4. Model 节点 → 存入 `data-initdata` 属性，首次激活时初始化

**优化项**：
- 扁平化嵌套的单 Layout 节点（减少不必要的嵌套层级）
- 未激活的 Tab 不创建 Model，节省资源
- 插件未加载的 Custom 标签会被移除

### 6.3 持久化存储路径

| 场景 | 存储位置 | 触发时机 |
|------|---------|---------|
| 主窗口布局 | 后端 `conf/uiLayout`（通过 `/api/system/setUILayout`） | 布局变化时实时保存 |
| 独立窗口布局 | `sessionStorage.layout` | 布局变化时实时保存 |
| 已关闭标签栈 | `localStorage`（Constants.LOCAL_CLOSED_TABS） | 关闭标签时 |
| 文档滚动位置 | `localStorage`（Constants.LOCAL_FILEPOSITION） | 切换/关闭标签时 |
| 窗口状态（位置/大小） | `~/.config/siyuan/windowState.json` | 窗口关闭时 |

### 6.4 保存策略

[saveLayout()](app/src/layout/util.ts#L128-L168)

```typescript
export const saveLayout = () => {
    const breakObj = {};
    let layoutJSON: any = {};
    if (isWindow()) {
        // 独立窗口
        layoutJSON = { layout: {} };
        layoutToJSON(window.siyuan.layout.layout, layoutJSON.layout, breakObj);
    } else {
        // 主窗口
        layoutJSON = {
            hideDock: useElement.getAttribute("xlink:href") === "#iconDock",
            layout: {},
            bottom: dockToJSON(window.siyuan.layout.bottomDock),
            left: dockToJSON(window.siyuan.layout.leftDock),
            right: dockToJSON(window.siyuan.layout.rightDock),
        };
        layoutToJSON(window.siyuan.layout.layout, layoutJSON.layout, breakObj);
        window.siyuan.config.uiLayout = layoutJSON;
    }
    
    if (Object.keys(breakObj).length > 0 && saveCount < 10) {
        // 有未就绪的 Model，延迟重试（指数退避）
        saveCount++;
        setTimeout(() => saveLayout(), Constants.TIMEOUT_LOAD * saveCount);
    } else {
        saveCount = 0;
        if (isWindow()) {
            sessionStorage.setItem("layout", JSON.stringify(layoutJSON));
        } else {
            if (!window.siyuan.config.readonly) {
                fetchPost("/api/system/setUILayout", {
                    layout: layoutJSON,
                    errorExit: false
                });
            }
        }
    }
};
```

**触发保存的场景**：
- 标签切换
- 标签添加/移除
- 标签拖拽排序
- 分屏调整
- 焦点变化
- 窗口 resize

### 6.5 后端存储

[kernel/model/conf.go](kernel/model/conf.go#L62-L109)

```go
type AppConf struct {
    UILayout *conf.UILayout `json:"uiLayout"`
    // ...
}

func (conf *AppConf) SetUILayout(uiLayout *conf.UILayout) {
    conf.m.Lock()
    defer conf.m.Unlock()
    conf.UILayout = uiLayout
}
```

- 使用读写锁保证并发安全
- 通过 `Conf.Save()` 持久化到磁盘的 `conf.json` 文件

---

## 七、跨窗口同步机制

SiYuan 的跨窗口同步采用 **双路通信架构**：
1. **Electron IPC 通道**：用于窗口间控制消息（标签拖拽、锁屏、关闭同步）
2. **WebSocket 通道**：用于后端数据推送（文档重命名、删除、笔记本关闭、插件事件）

两种通道各司其职，共同实现多窗口间的状态一致性。

### 7.1 通信架构全景

```
┌─────────────────────────────────────────────────────────────┐
│                    Electron 主进程                          │
│  ┌──────────────────────────┐  ┌────────────────────────┐  │
│  │  ipcMain (siyuan-send-   │  │  powerMonitor          │  │
│  │      windows)             │  │  (lock-screen)        │  │
│  └──────────────────────────┘  └────────────────────────┘  │
│                      ▲                    ▲                  │
│                      │ IPC                │                  │
└──────────────────────┼────────────────────┼──────────────────┘
                       │                    │
┌──────────────────────┼────────────────────┼──────────────────┐
│  主窗口渲染进程        │                    │                  │
│  ┌────────────────────▼──────────┐  ┌────▼──────────────┐   │
│  │  WebSocket (后端数据推送)     │  │  ipcRenderer       │   │
│  │  (rename/removeDoc/...)       │  │  (control msgs)    │   │
│  └───────────────────────────────┘  └───────────────────┘   │
│                      ▲                                         │
│                      │ WebSocket                               │
└──────────────────────┼─────────────────────────────────────────┘
                       │
┌──────────────────────┼─────────────────────────────────────────┐
│  独立窗口渲染进程      │                                         │
│  ┌────────────────────▼──────────┐  ┌────────────────────┐   │
│  │  WebSocket (后端数据推送)     │  │  ipcRenderer       │   │
│  │  (与主窗口相同处理逻辑)       │  │  (control msgs)    │   │
│  └───────────────────────────────┘  └────────────────────┘   │
│                                                               │
└───────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Go 后端内核 (单实例)                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  WebSocket Hub + PushMode 广播机制                    │  │
│  │  - rename / removeDoc / closeBox / reloadPlugin ...  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 后端 WebSocket 推送机制（pushMode）

后端通过 `PushMode` 精确控制消息的广播范围，定义在 [result.go](kernel/util/result.go#L24-L33)：

```go
type PushMode int

const (
    PushModeBroadcast                   PushMode = 0  // 所有应用所有会话广播
    PushModeSingleSelf                  PushMode = 1  // 自我应用会话单播
    PushModeBroadcastExcludeSelf        PushMode = 2  // 非自我会话广播
    PushModeBroadcastExcludeSelfApp     PushMode = 4  // 非自我应用所有会话广播
    PushModeBroadcastApp                PushMode = 5  // 单个应用内所有会话广播
    PushModeBroadcastMainExcludeSelfApp PushMode = 6  // 非自我应用主会话广播
)
```

**推送分发逻辑**在 [websocket.go](kernel/util/websocket.go#L383-L400)：

```go
func PushEvent(event *Result) {
    msg := event.Bytes()
    mode := event.PushMode
    switch mode {
    case PushModeBroadcast:
        Broadcast(msg)                      // 推送到所有连接
    case PushModeSingleSelf:
        single(msg, event.AppId, event.SessionId)
    case PushModeBroadcastExcludeSelf:
        broadcastOthers(msg, event.SessionId)
    case PushModeBroadcastExcludeSelfApp:
        broadcastOtherApps(msg, event.AppId)
    case PushModeBroadcastApp:
        broadcastApp(msg, event.AppId)
    case PushModeBroadcastMainExcludeSelfApp:
        broadcastOtherAppMains(msg, event.AppId)
    }
}
```

**关键数据事件均使用 `PushModeBroadcast (0)`**，意味着主窗口和独立窗口会同时收到这些事件：

| 事件 | 推送位置 | 数据 |
|------|---------|------|
| `rename` | [history.go#360](kernel/model/history.go#L360) / [file.go#1724](kernel/model/file.go#L1724) | `{box, id, path, title}` |
| `removeDoc` | [heading.go#284](kernel/model/heading.go#L284) / [file.go#1635](kernel/model/file.go#L1635) | `{ids: [rootID, ...]}` |
| `closeBox` / `removeBox` | [mount.go#185-193](kernel/model/mount.go#L185-L193) | `{box: notebookID}` |
| `reloadPlugin` | 插件管理 API | `{uninstallPlugins, unloadPlugins, reloadPlugins, ...}` |

---

### 7.3 主窗口与独立窗口：WebSocket 推送处理对比

**重要发现**：主窗口和独立窗口处理 WebSocket 推送的代码是**完全相同**的！

两者都在 `App` 构造函数中注册了相同的 `msgCallback` 处理函数 [window/index.ts#L60-L163](app/src/window/index.ts#L60-L163)：

```typescript
new Model({
    msgCallback(data) {
        if (data.cmd === "error" && data.msg) {
            showMessage(data.msg, 3000, "error");
        }
        if (data.reqId === this.reqId || 0 === data.reqId) {
            switch (data.cmd) {
                case "logoutAuth":         redirectToCheckAuth(); break;
                case "setAppearance":      updateAppearance(data.data); break;
                case "rename":             this.handleRename(data); break;
                case "closeBox":
                case "removeBox":          this.handleCloseBox(data); break;
                case "removeDoc":          this.handleRemoveDoc(data); break;
                case "reloadPlugin":       reloadPlugin(this, data.data); break;
                // ... 其他 20+ 种事件
            }
        }
    }
})
```

**处理逻辑完全相同，但执行结果有差异**：

| 差异点 | 主窗口 | 独立窗口 |
|--------|-------|---------|
| 文件树刷新 | ✅ 有文件树，`setNoteBook()` 会刷新 | ❌ 无文件树，`setNoteBook()` 无效 |
| 侧边栏更新 | ✅ 有完整侧边栏，`allModels` 包含所有 Dock 面板 | ✅ 有标签面板，但可能缺少某些 Dock 面板 |
| 插件实例 | ✅ 完整的插件实例集合 | ✅ 独立的插件实例集合（相同插件代码，独立实例） |
| 窗口标题 | ✅ 会调用 `setTitle()` 更新 | ✅ 也会调用 `setTitle()` 更新 |

---

### 7.4 文档重命名事件同步流程

**触发场景**：
- 修改文档标题（标题块属性）
- 移动文档到其他目录（更新 path）

**后端推送**：
```go
evt := util.NewCmdResult("rename", 0, util.PushModeBroadcast)
evt.Data = map[string]any{
    "box":     boxID,
    "id":      tree.Root.ID,      // 文档 rootID
    "path":    tree.Path,
    "title":   tree.Root.IALAttr("title"),
}
util.PushEvent(evt)
```

**前端同步流程**（所有窗口同时执行）：

**第一步：处理未激活标签** [window/index.ts#L101-L113](app/src/window/index.ts#L101-L113)
```typescript
case "rename":
    getAllTabs().forEach((tab) => {
        if (tab.headElement) {
            const initTab = tab.headElement.getAttribute("data-initdata");
            if (initTab) {
                const initTabData = JSON.parse(initTab);
                // 只匹配 Editor 类型且 rootID 匹配的标签
                if (initTabData.instance === "Editor" && initTabData.rootId === data.data.id) {
                    tab.updateTitle(data.data.title);  // 更新标签头标题
                }
            }
        }
    });
    break;
```

**第二步：处理已激活标签**（通过 `reloadSync` 间接处理）
- 后端还会发送 `syncMergeResult` 或 `reloaddoc` 事件
- 触发 `reloadSync()` 函数 [processSystem.ts#L79-L87](app/src/dialog/processSystem.ts#L79-L87)
- 在 `reloadSync` 中遍历所有已激活的 Editor Model：
```typescript
allModels.editor.forEach(item => {
    if (data.upsertRootIDs.includes(item.editor.protyle.block.rootID)) {
        fetchPost("/api/block/getDocInfo", { id: item.editor.protyle.block.rootID },
            (response) => {
                // 1. 刷新编辑器内容
                reloadProtyle(item.editor.protyle, false, updateReadonly);
                // 2. 更新标签标题和编辑器标题栏
                updateTitle(item.editor.protyle.block.rootID, item.parent, item.editor.protyle);
            });
    }
});
```

**`updateTitle` 辅助函数** [processSystem.ts#L29-L38](app/src/dialog/processSystem.ts#L29-L38)：
```typescript
const updateTitle = (rootID: string, tab: Tab, protyle?: IProtyle) => {
    fetchPost("/api/block/getDocInfo", { id: rootID }, (response) => {
        tab.updateTitle(response.data.name);  // 更新标签头
        if (protyle && protyle.title) {
            // 更新编辑器内的标题栏
            protyle.title.setTitle(response.data.name, 
                response.data.ial[Constants.CUSTOM_SY_TITLE_EMPTY] === "true");
        }
    });
};
```

**重命名同步完整链路**：
```
用户修改文档标题
  ↓
后端事务处理 → 更新数据库
  ↓
推送 rename 事件（PushModeBroadcast）
  ↓
所有窗口（主+独立）同时接收
  │
  ├─→ 未激活标签：直接更新 tab.headElement 标题
  │    [window/index.ts#L101-L113]
  │
  └─→ 已激活标签：通过 reloadSync
       ├─ 刷新编辑器内容
       └─ 更新标签标题 + 编辑器标题栏
          [processSystem.ts#L79-L87]
```

---

### 7.5 文档删除事件同步流程

**触发场景**：
- 删除文档（heading.go）
- 删除目录（file.go，包含所有子文档）

**后端推送**：
```go
evt := util.NewCmdResult("removeDoc", 0, util.PushModeBroadcast)
evt.Data = map[string]any{
    "ids": []string{srcTree.ID},  // 支持批量删除
}
util.PushEvent(evt)
```

**前端同步流程**（所有窗口同时执行）：

**第一步：处理未激活标签** [window/index.ts#L128-L140](app/src/window/index.ts#L128-L140)
```typescript
case "removeDoc":
    getAllTabs().forEach((tab) => {
        if (tab.headElement) {
            const initTab = tab.headElement.getAttribute("data-initdata");
            if (initTab) {
                const initTabData = JSON.parse(initTab);
                if (initTabData.instance === "Editor" 
                    && data.data.ids.includes(initTabData.rootId)) {
                    tab.parent.removeTab(tab.id);  // 直接关闭标签
                }
            }
        }
    });
    break;
```

**第二步：处理已激活标签**（通过 `reloadSync`）
- 在 `reloadSync` 中遍历所有已激活的 Model：
```typescript
allModels.editor.forEach(item => {
    if (data.removeRootIDs.includes(item.editor.protyle.block.rootID)) {
        // 关闭标签
        item.parent.parent.removeTab(item.parent.id, false, false);
        // 清理滚动位置缓存
        delete window.siyuan.storage[Constants.LOCAL_FILEPOSITION]
            [item.editor.protyle.block.rootID];
        setStorageVal(Constants.LOCAL_FILEPOSITION, 
            window.siyuan.storage[Constants.LOCAL_FILEPOSITION]);
    }
});
```

**不仅处理 Editor，还处理所有相关 Model 类型**：
- Graph（本地关系图）
- Outline（本地大纲）
- Backlink（本地反链）

**删除同步完整链路**：
```
用户删除文档
  ↓
后端事务处理 → 删除文件 → 更新数据库
  ↓
推送 removeDoc 事件（PushModeBroadcast）
  ↓
所有窗口（主+独立）同时接收
  │
  ├─→ 未激活标签：直接 removeTab 关闭
  │    [window/index.ts#L128-L140]
  │
  └─→ 已激活标签：通过 reloadSync
       ├─ Editor：关闭标签 + 清理滚动位置
       ├─ Graph（本地）：关闭标签
       ├─ Outline（本地）：关闭标签
       ├─ Backlink（本地）：关闭标签
       └─ 其他类型：刷新数据
          [processSystem.ts#L88-L128]
```

---

### 7.6 笔记本关闭/删除事件同步流程

**触发场景**：
- 卸载笔记本（`closeBox`）
- 删除笔记本（`removeBox`）
- 新手引导笔记本关闭（特殊逻辑）

**后端推送** [mount.go#L185-L193](kernel/model/mount.go#L185-L193)：
```go
cmdName := "closeBox"
if IsUserGuide(boxID) {
    if err := RemoveBox(boxID); err == nil {
        cmdName = "removeBox"  // 新手引导笔记本自动删除
    }
}
evt := util.NewCmdResult(cmdName, 0, util.PushModeBroadcast)
evt.Data = map[string]any{"box": boxID}
util.PushEvent(evt)
```

**前端同步流程**（所有窗口同时执行）：

**第一步：处理未激活标签** [window/index.ts#L114-L127](app/src/window/index.ts#L114-L127)
```typescript
case "closeBox":
case "removeBox":
    getAllTabs().forEach((tab) => {
        if (tab.headElement) {
            const initTab = tab.headElement.getAttribute("data-initdata");
            if (initTab) {
                const initTabData = JSON.parse(initTab);
                if (initTabData.instance === "Editor" 
                    && data.data.box === initTabData.notebookId) {
                    tab.parent.removeTab(tab.id);  // 关闭该笔记本的所有标签
                }
            }
        }
    });
    break;
```

**第二步：处理已激活标签**（同样通过 `reloadSync`）
- 当笔记本关闭/删除时，会触发 `syncMergeResult` 事件
- `reloadSync` 中检查 `removeRootIDs` 包含的文档并关闭标签

**特殊说明**：
- `closeBox` 和 `removeBox` 在前端处理逻辑完全相同
- 区别仅在后端：`closeBox` 只卸载，`removeBox` 会删除物理文件
- 前端只需关闭该笔记本下的所有标签即可

---

### 7.7 插件事件跨窗口同步

**触发场景**：
- 插件安装、卸载、启用、禁用
- 插件代码变更（热重载）
- 插件存储数据变更

**后端推送**：
```go
evt := util.NewCmdResult("reloadPlugin", 0, util.PushModeBroadcast)
evt.Data = map[string]any{
    "uninstallPlugins":  [...],  // 已卸载的插件列表
    "unloadPlugins":     [...],  // 已禁用的插件列表
    "reloadPlugins":     [...],  // 需重载的插件列表
    "dataChangePlugins": [...],  // 数据变更的插件列表
}
util.PushEvent(evt)
```

**前端同步流程**：

**主窗口和独立窗口各自独立处理** [window/index.ts#L76-L78](app/src/window/index.ts#L76-L78)：
```typescript
case "reloadPlugin":
    reloadPlugin(this, data.data);
    break;
```

**`reloadPlugin` 完整处理逻辑** [loader.ts#L225-L299](app/src/plugin/loader.ts#L225-L299)：
```typescript
export const reloadPlugin = async (app: App, data: {
    uninstallPlugins?: string[],
    unloadPlugins?: string[],
    reloadPlugins?: string[],
    dataChangePlugins?: string[],
} = {}) => {
    const {uninstallPlugins, unloadPlugins, reloadPlugins, dataChangePlugins} = data;
    
    // 1. 禁用插件
    unloadPlugins.forEach((item) => {
        uninstall(app, item, true);  // 调用插件 onunload()
    });
    
    // 2. 卸载插件
    uninstallPlugins.forEach((item) => {
        uninstall(app, item, false);  // 完全移除
    });
    
    // 3. 重载插件（启用或代码变更）
    for (const item of reloadPlugins) {
        await loadPlugin(app, item);  // 重新加载插件代码
    }
    
    // 4. 插件数据变更回调
    dataChangePlugins.forEach((item) => {
        const pluginInstance = app.plugins.find(p => p.name === item);
        if (pluginInstance) {
            fetchPost("/api/plugin/getSetting", { name: item }, 
                (response) => {
                    pluginInstance.data = response.data;  // 更新设置
                    if (pluginInstance.onSetting) {
                        pluginInstance.onSetting();  // 触发回调
                    }
                });
        }
    });
};
```

**插件事件同步的关键特性**：

1. **独立实例，独立处理**：
   - 主窗口和独立窗口有各自的插件实例集合
   - 每个窗口独立执行 `reloadPlugin`，互不影响
   - 相同插件代码，但状态（`pluginInstance.data`）是隔离的

2. **Custom 标签的自动处理**：
   - 插件卸载时，类型为 `Custom` 且属于该插件的标签会被移除
   - 这是在 `JSONToLayout` 时检查的，插件不存在则跳过创建

3. **插件卸载顺序**：
   - 先调用 `onunload()` 清理资源
   - 然后移除 DOM 元素（Dock 面板、顶层块）
   - 最后从 `app.plugins` 数组中移除

---

### 7.8 Electron IPC 通道同步（控制消息）

除了 WebSocket 数据推送，窗口间还通过 Electron IPC 传递控制消息。

**IPC 消息分发入口** [onGetConfig.ts#L175-L184](app/src/boot/onGetConfig.ts#L175-L184)：
```typescript
ipcRenderer.on(Constants.SIYUAN_SAVE_CLOSE, (event, close) => {
    if (isWindow()) {
        closeWindow(app);    // 独立窗口关闭
    } else {
        winOnClose(close);   // 主窗口关闭
    }
});

ipcRenderer.on(Constants.SIYUAN_SEND_WINDOWS, (e, ipcData: IWebSocketData) => {
    onWindowsMsg(ipcData, app);  // 跨窗口广播消息
});
```

**主进程广播机制** [main.js#L1301-L1305](app/electron/main.js#L1301-L1305)：
```javascript
ipcMain.on("siyuan-send-windows", (event, data) => {
    BrowserWindow.getAllWindows().forEach(item => {
        item.webContents.send("siyuan-send-windows", data);
    });
});
```

**渲染进程消息处理** [onWindowsMsg.ts#L13-L45](app/src/window/onWindowsMsg.ts#L13-L45)：

| 消息类型 | 用途 | 主窗口处理 | 独立窗口处理 |
|---------|------|-----------|-------------|
| `closetab` | 标签跨窗口拖拽后，关闭原窗口标签 | ✅ 移除指定标签 | ✅ 移除指定标签 |
| `resetTabsStyle` | 拖拽时的样式同步（添加/移除拖拽区域样式） | ✅ 移除拖拽样式 | ✅ 移除拖拽样式 + 调整拖拽区域 |
| `lockscreenByMode` | 系统锁屏事件同步 | ✅ 根据配置锁屏 | ✅ 根据配置锁屏 |

---

### 7.9 系统锁屏同步

**触发**：操作系统锁屏事件

**流程** [main.js#L1416-L1420](app/electron/main.js#L1416-L1420)：
```javascript
powerMonitor.on("lock-screen", () => {
    writeLog("system lock-screen");
    BrowserWindow.getAllWindows().forEach(item => {
        item.webContents.send("siyuan-send-windows", {cmd: "lockscreenByMode"});
    });
});
```

**各窗口独立判断** [onWindowsMsg.ts#L32-L38](app/src/window/onWindowsMsg.ts#L32-L38)：
```typescript
case "lockscreenByMode":
    if (window.siyuan.config.system.lockScreenMode === 1) {
        lockScreen(app);  // 仅配置为 1 时才锁屏
    }
    break;
```

---

### 7.10 标签跨窗口拖拽同步

```
用户在窗口 A 拖拽标签到窗口 B
  ↓
dragstart [Tab.ts#L84]: 记录 tab ID 和布局数据
  ↓
dragover: 窗口 B 边缘高亮，显示放置预览
  ↓
drop: 窗口 B 接收数据 → JSONToCenter() 创建标签
  ↓
窗口 B 发送 IPC 广播: 
  ipcRenderer.send("siyuan-send-windows", {cmd: "closetab", data: tabId})
  ↓
主进程广播到所有窗口 [main.js#L1301-L1305]
  ↓
窗口 A 收到 closetab 命令 [onWindowsMsg.ts#L15-L16]
  ↓
tab.parent.removeTab(ipcData.data) → 移除原标签
```

---

### 7.11 同步机制总结

| 同步类型 | 通道 | 推送范围 | 主窗口处理 | 独立窗口处理 |
|---------|------|---------|-----------|-------------|
| 文档重命名 | WebSocket | 所有应用所有会话 | ✅ 更新未激活标签标题 + reloadSync 处理已激活 | ✅ 相同逻辑 |
| 文档删除 | WebSocket | 所有应用所有会话 | ✅ 关闭未激活标签 + reloadSync 关闭已激活 | ✅ 相同逻辑 |
| 笔记本关闭 | WebSocket | 所有应用所有会话 | ✅ 关闭该笔记本所有标签 | ✅ 相同逻辑 |
| 插件事件 | WebSocket | 所有应用所有会话 | ✅ 独立执行 reloadPlugin | ✅ 独立执行 reloadPlugin（独立插件实例） |
| 标签拖拽关闭 | IPC | 所有窗口 | ✅ 移除对应标签 | ✅ 移除对应标签 |
| 系统锁屏 | IPC | 所有窗口 | ✅ 根据配置锁屏 | ✅ 根据配置锁屏 |
| 拖拽样式同步 | IPC | 所有窗口 | ✅ 更新样式 | ✅ 更新样式 + 调整拖拽区域 |

**关键设计原则**：
1. **数据同步走 WebSocket，控制同步走 IPC**
2. **主窗口和独立窗口使用相同的处理代码**，确保行为一致
3. **后端使用 PushMode 精确控制广播范围**，避免不必要的消息
4. **未激活标签和已激活标签分开处理**，前者轻量更新，后者完整刷新
5. **插件实例隔离**，各窗口独立管理自己的插件生命周期

---

## 八、资源管理

### 8.1 Model 销毁

[Wnd.destroyModel()](app/src/layout/Wnd.ts#L741-L770)

```typescript
private destroyModel(model: Model) {
    if (!model) return;
    
    if (model instanceof Editor) {
        // 1. 销毁相关浮动面板
        window.siyuan.blockPanels.forEach(item => {
            if (model.editor.protyle.wysiwyg.element.contains(item.element)) {
                item.destroy();
            }
        });
        // 2. 销毁编辑器
        model.editor.destroy();
    } else if (model instanceof Search) {
        model.editors.edit.destroy();
        model.editors.unRefEdit.destroy();
    } else if (model instanceof Asset) {
        if (model.pdfObject?.pdfLoadingTask) {
            model.pdfObject.pdfLoadingTask.destroy();
        }
    } else if (model instanceof Custom) {
        if (model.destroy) model.destroy();  // 插件模型自定义销毁
    }
    
    // 3. 发送关闭 WebSocket 消息，通知后端清理
    model.send("closews", {});
}
```

### 8.2 WebSocket 连接管理

每个 Model 实例维护独立的 WebSocket 连接：
- 连接断开后自动重连（3秒间隔）
- 认证失败不重连
- 模型销毁时发送 `closews` 命令通知后端清理

**重连机制**：[Model.ts - ws.onclose](app/src/layout/Model.ts#L65-L80)

### 8.3 内存管理策略

1. **标签懒加载**：未激活的标签不初始化 Model，只保存初始化数据（`data-initdata`）
2. **最大标签数限制**：超过 `maxOpenTabCount` 时自动关闭最久未使用的（`removeOverCounter`）
3. **窗口关闭时清理**：调用 `destroyModel()` 释放编辑器、WebSocket 等资源
4. **Electron 缓存清理**：关闭标签后调用 `webFrame.clearCache()`
5. **编辑器 Range 维护**：`moveTab()` 时重新计算 Range，避免 DOM 移动后引用失效

---

## 九、窗口关闭与资源释放

> **重要修正**：主窗口和独立窗口的关闭流程是**完全不同的两条路径**，不能混为一谈。
> - 主窗口关闭：走 `winOnClose → exportLayout → exitSiYuan` 路径，核心是**保存布局+退出应用**
> - 独立窗口关闭：走 `closeWindow → 卸载插件 → destroy` 路径，核心是**清理插件+销毁窗口**

### 9.1 统一入口：IPC 消息分发

所有关闭请求的统一入口在 [onGetConfig.ts#L175-L181](app/src/boot/onGetConfig.ts#L175-L181)：

```typescript
ipcRenderer.on(Constants.SIYUAN_SAVE_CLOSE, (event, close) => {
    if (isWindow()) {
        closeWindow(app);    // 独立窗口 → 走独立窗口关闭流程
    } else {
        winOnClose(close);   // 主窗口 → 走主窗口关闭流程
    }
});
```

根据 `isWindow()` 判断当前是主窗口还是独立窗口，分发到不同的处理函数。

---

### 9.2 主窗口关闭流程（winOnClose）

**触发时机**：
- 用户点击主窗口关闭按钮
- 系统退出命令（`before-quit` 事件，`close=true`）
- 菜单退出操作

**处理函数**：[onGetConfig.ts#L114-L132](app/src/boot/onGetConfig.ts#L114-L132)

```typescript
const winOnClose = (close = false) => {
    exportLayout({
        cb() {
            if (window.siyuan.config.appearance.closeButtonBehavior === 1 && !close) {
                // 最小化到托盘（仅关闭按钮触发时）
                if ("windows" === window.siyuan.config.system.os) {
                    ipcRenderer.send(Constants.SIYUAN_CONFIG_TRAY, {
                        languages: window.siyuan.languages["_trayMenu"],
                    });
                } else {
                    ipcRenderer.send(Constants.SIYUAN_CMD, "closeButtonBehavior");
                }
            } else {
                exitSiYuan();  // 真正退出应用
            }
        },
        errorExit: true
    });
};
```

**完整流程**：

```
用户点击主窗口关闭按钮
  │
  ├─→ 主进程 close 事件拦截 [main.js#L513-L518]
  │    send("siyuan-save-close", false)
  │    event.preventDefault()  // 阻止默认关闭
  │
  ├─→ winOnClose(close=false)
  │    │
  │    └─→ exportLayout() 保存布局 [util.ts#L170-L209]
  │         │
  │         ├─ 1. 保存所有编辑器滚动位置 (saveScroll)
  │         ├─ 2. 序列化布局 JSON (layoutToJSON)
  │         └─ 3. 调用 /api/system/setUILayout 保存到后端
  │
  ├─→ 保存完成后回调 cb()
  │    │
  │    ├─→ 如果配置了最小化到托盘且 close=false：
  │    │    └─ 最小化到托盘（不退出应用）
  │    │
  │    └─→ 否则：exitSiYuan() 退出应用 [processSystem.ts#L291]
  │         │
  │         ├─ 调用 /api/system/exit 让后端退出
  │         └─ 后端退出后发送 siyuan-quit 让主进程退出
  │
  └─→ 主进程接收 quit 命令，退出整个应用
```

**主窗口关闭的关键特征**：

| 特征 | 说明 |
|------|------|
| 布局保存 | ✅ 调用 `exportLayout()`，保存到后端配置 |
| 滚动位置 | ✅ 保存所有编辑器的滚动位置 |
| 插件卸载 | ❌ **不卸载插件**（应用退出后进程销毁自动回收） |
| 标签资源释放 | ❌ **不主动调用 destroyModel**（进程退出自动回收） |
| 应用是否退出 | 取决于配置：可能只是最小化，也可能完全退出 |
| destroy 命令 | ❌ 不发送 destroy，由 quit 命令退出应用 |

---

### 9.3 独立窗口关闭流程（closeWindow）

**触发时机**：
- 用户点击独立窗口关闭按钮
- 拖拽标签合并回主窗口后关闭独立窗口

**处理函数**：[closeWin.ts#L5-L13](app/src/window/closeWin.ts#L5-L13)

```typescript
export const closeWindow = async (app: App) => {
    // 1. 卸载所有插件
    for (let i = 0; i < app.plugins.length; i++) {
        try {
            await app.plugins[i].onunload();
        } catch (e) {
            console.error(e);
        }
    }
    // 2. 发送销毁命令
    ipcRenderer.send(Constants.SIYUAN_CMD, "destroy");
};
```

**完整流程**：

```
用户点击独立窗口关闭按钮
  │
  ├─→ 主进程 close 事件拦截 [main.js#L1169-L1174]
  │    send("siyuan-save-close")
  │    event.preventDefault()  // 阻止默认关闭
  │
  ├─→ closeWindow(app)
  │    │
  │    ├─ 1. 遍历所有插件，调用 onunload() 卸载
  │    │    （独立窗口有独立的插件实例，需要显式卸载）
  │    │
  │    └─ 2. 发送 "destroy" 命令给主进程
  │
  └─→ 主进程 destroy 命令处理 [main.js#L1046-L1051]
       currentWindow.destroy()
```

**独立窗口关闭的关键特征**：

| 特征 | 说明 |
|------|------|
| 布局保存 | ❌ **不调用 exportLayout()** |
| 滚动位置 | ❌ **不主动保存**（依赖实时 saveLayout，但 saveLayout 不 saveScroll） |
| 插件卸载 | ✅ 显式调用每个插件的 `onunload()` 方法 |
| 标签资源释放 | ❌ **不主动调用 destroyModel**（窗口销毁后 GC 回收） |
| 应用是否退出 | ❌ 只销毁当前窗口，不影响主窗口和后端 |
| destroy 命令 | ✅ 发送 destroy，仅销毁当前 BrowserWindow |

---

### 9.4 主窗口 vs 独立窗口：关闭流程对比

| 对比项 | 主窗口 (winOnClose) | 独立窗口 (closeWindow) |
|-------|---------------------|----------------------|
| **入口函数** | `winOnClose(close)` | `closeWindow(app)` |
| **入口位置** | [onGetConfig.ts#L114](app/src/boot/onGetConfig.ts#L114) | [closeWin.ts#L5](app/src/window/closeWin.ts#L5) |
| **布局保存** | ✅ exportLayout() → 后端 | ❌ 不主动保存（依赖实时 saveLayout） |
| **滚动位置保存** | ✅ saveScroll 所有编辑器 | ❌ 不主动保存 |
| **插件卸载** | ❌ 不卸载（进程销毁自动回收） | ✅ 遍历调用 onunload() |
| **标签 Model 销毁** | ❌ 不主动调用 destroyModel | ❌ 不主动调用 destroyModel |
| **WebSocket 断开** | 依赖进程退出 | 依赖窗口销毁 |
| **最终动作** | 最小化托盘 / exitSiYuan() 退出应用 | send("destroy") 销毁窗口 |
| **后端影响** | 退出整个后端内核 | 无影响（后端是单实例共享） |
| **errorExit 参数** | ✅ 传 true（保存失败仍退出） | 无此概念 |

---

### 9.5 布局保存机制辨析

独立窗口虽然关闭时不调用 `exportLayout()`，但布局数据并非完全不保存：

**1. 实时保存（saveLayout）**
- 每次布局变化（标签切换、添加、移除、分屏调整）都会触发 `saveLayout()`
- 独立窗口的 `saveLayout()` 将布局序列化为 JSON 存入 `sessionStorage`
- 但 `saveLayout()` **不保存滚动位置**（不调用 saveScroll）

[util.ts#L157-L158](app/src/layout/util.ts#L157-L158)：
```typescript
if (isWindow()) {
    sessionStorage.setItem("layout", JSON.stringify(layoutJSON));
}
```

**2. 关闭时最终保存（exportLayout）**
- 主窗口关闭时调用 `exportLayout()`，会先 `saveScroll` 再保存布局
- 独立窗口关闭时**不调用** `exportLayout()`，因此最新的滚动位置可能丢失

**3. 布局恢复**
- 主窗口：从后端 `conf/uiLayout` 恢复
- 独立窗口：从 `sessionStorage.layout` 恢复（窗口刷新时），或从 URL 参数恢复（新建时）

> **注意**：独立窗口销毁后 `sessionStorage` 也随之消失，因此关闭独立窗口后再重新打开，无法恢复之前的布局状态，只能从 URL 参数创建新的布局。

---

### 9.6 标签资源释放机制

**两种窗口都不主动释放标签 Model 资源**，原因不同：

**主窗口**：
- 整个应用都要退出，进程销毁会自动回收所有内存
- 显式逐个释放反而可能拖慢退出速度
- WebSocket 连接会随进程退出而断开，后端也会退出

**独立窗口**：
- 窗口销毁后，渲染进程的 JavaScript 上下文随之销毁
- 理论上 GC 会回收所有对象
- 但存在一些潜在问题：
  - WebSocket 连接可能没有优雅关闭（没有发送 `closews` 消息）
  - 插件如果有外部资源引用，可能泄漏（已通过 onunload 处理）
  - 后端的会话清理可能依赖连接断开检测

---

### 9.7 异常场景处理

**1. 正在上传时关闭标签**

[Wnd.removeTab()](app/src/layout/Wnd.ts#L891-L895)

```typescript
if ((item.model instanceof Editor) && item.model.editor?.protyle) {
    if (item.model.editor.protyle.upload.isUploading) {
        showMessage(window.siyuan.languages.uploading);
        return;  // 上传中阻止关闭
    }
}
```

**2. 内核中断恢复**

[Model.ts - ws.onopen](app/src/layout/Model.ts#L45-L56)

WebSocket 重连成功后，会重新同步数据和刷新界面。

**3. 文档被删除时标签处理**

后端通过 WebSocket 推送 `removeDoc` 事件，前端遍历所有标签，移除被删除文档的标签。

**4. 插件卸载时标签清理**

布局加载时检查 Custom 类型标签对应的插件是否存在，不存在则移除该标签。

[util.ts - JSONToLayout()](app/src/layout/util.ts#L412-L434)

**5. 启动时不恢复标签**

如果配置了 `closeTabsOnStart`，启动时只保留固定的标签。

**6. 关闭按钮行为配置**

[onGetConfig.ts#L117-L128](app/src/boot/onGetConfig.ts#L117-L128)

- `closeButtonBehavior === 1`：点击关闭按钮最小化到托盘（仅未设置 `close=true` 时）
- 其他值：直接退出应用

---

## 十、主要协作流程图

### 10.1 应用启动布局恢复流程

```
应用启动
  │
  ├─→ fetchPost("/api/system/getConf") 获取配置
  │
  ├─→ 加载插件
  │
  ├─→ JSONToLayout(app, uiLayout.layout)
  │    │
  │    ├─→ 递归构建 Layout/Wnd/Tab 树
  │    │    （Tab 的 Model 不初始化，存 data-initdata）
  │    │
  │    ├─→ JSONToDock() 构建停靠面板
  │    │
  │    ├─→ 处理 closeTabsOnStart（可选移除非固定标签）
  │    │
  │    ├─→ 移除缺失插件的 Custom 标签
  │    │
  │    └─→ 激活上次的活动标签
  │         (tab.parent.switchTab())
  │
  ├─→ getIdZoomInByPath() 处理 URL 参数
  │
  ├─→ saveLayout() 保存初始布局
  │
  └─→ 插件 afterLoad
```

### 10.2 打开新文档流程

```
用户点击文档树 / 搜索结果
  │
  ├─→ openFileById()
  │
  ├─→ 检查是否已有该文档的标签
  │    ├─ 有 → 切换到该标签
  │    └─ 无 → 创建新标签
  │
  ├─→ 如果 openFilesUseCurrentTab：
  │    └─ 替换当前"未更新"标签
  │
  ├─→ wnd.addTab(newTab)
  │    ├─ 插入到当前聚焦标签后
  │    ├─ 设置关闭按钮监听
  │    └─ 超过最大标签数时关闭最旧的
  │
  ├─→ 创建 Editor Model
  │    ├─ 建立 WebSocket 连接
  │    └─ 请求文档数据
  │
  └─→ saveLayout()
```

### 10.3 拖拽创建新窗口流程

```
用户拖拽标签头
  │
  ├─→ dragstart 事件 [Tab.ts#L84]
  │    ├─ 序列化 tab 数据（layoutToJSON）
  │    ├─ 设置 dataTransfer
  │    └─ 设置拖拽元素样式（opacity: 0.38）
  │
  ├─→ 拖出窗口边界
  │
  ├─→ dragend 事件（检测到在窗口外）
  │    └─→ openNewWindow(tab) [openNewWindow.ts#L22]
  │         ├─ 序列化 tab
  │         ├─ 发送 siyuan-open-window 到主进程
  │         └─ 移除当前窗口的标签
  │
  └─→ 新窗口加载
       ├─ 从 URL 参数解析标签 JSON
       ├─ JSONToCenter 构建布局
       └─ 激活标签、初始化 Model
```

### 10.4 窗口关闭流程（两条独立路径）

> **修正说明**：主窗口和独立窗口的关闭是**两条完全独立的路径**，没有统一的"关闭保存流程"。

**10.4.1 主窗口关闭流程**

```
用户点击主窗口关闭按钮
  │
  ├─→ 主进程 close 事件 [main.js#L513-L518]
  │    send("siyuan-save-close", false)
  │    event.preventDefault()
  │
  ├─→ 渲染进程分发 [onGetConfig.ts#L175-L181]
  │    isWindow() === false → winOnClose(false)
  │
  └─→ winOnClose()  [onGetConfig.ts#L114-L132]
       │
       └─→ exportLayout()  [util.ts#L170-L209]
            │
            ├─ 1. saveScroll 所有编辑器滚动位置
            ├─ 2. layoutToJSON 序列化布局
            └─ 3. fetchPost("/api/system/setUILayout") 保存到后端
                 │
                 └─ 保存完成回调 cb()
                      │
                      ├─ 配置最小化托盘 且 close=false → 最小化
                      └─ 否则 → exitSiYuan() 退出应用 [processSystem.ts#L291]
                           │
                           ├─ 调用 /api/system/exit 退出后端
                           └─ 发送 siyuan-quit 退出主进程
```

**10.4.2 独立窗口关闭流程**

```
用户点击独立窗口关闭按钮
  │
  ├─→ 主进程 close 事件 [main.js#L1169-L1174]
  │    send("siyuan-save-close")
  │    event.preventDefault()
  │
  ├─→ 渲染进程分发 [onGetConfig.ts#L175-L181]
  │    isWindow() === true → closeWindow(app)
  │
  └─→ closeWindow()  [closeWin.ts#L5-L13]
       │
       ├─ 1. 遍历插件，调用 onunload() 卸载
       │    （独立窗口有独立插件实例，必须显式卸载）
       │
       └─ 2. 发送 "destroy" 命令
            │
            └─ 主进程 destroy 命令 [main.js#L1046-L1051]
                 currentWindow.destroy()
```

**10.4.3 两条路径的关键差异**

| 环节 | 主窗口 | 独立窗口 |
|------|-------|---------|
| 入口函数 | `winOnClose(close)` | `closeWindow(app)` |
| 布局保存 | `exportLayout()` → 后端 | 不主动调用，依赖 `saveLayout()` 实时保存 |
| 滚动位置 | `saveScroll()` 所有编辑器 | 不主动保存 |
| 插件处理 | 不卸载（进程销毁自动回收） | 显式调用 `onunload()` |
| 标签资源 | 不主动释放 | 不主动释放 |
| 最终动作 | 最小化 / 退出整个应用 | 仅销毁当前 BrowserWindow |
| 后端影响 | 退出内核 | 无影响 |

---

## 十一、关键协作细节

### 11.1 标签激活时间（activeTime）

- 每个标签头有 `data-activetime` 属性，记录时间戳
- 切换标签、窗口获得焦点时更新
- 用于：
  - 关闭当前标签时，选择最近活动的标签作为下一个
  - 超过最大标签数时，关闭最久未活动的
  - 布局持久化中保存，用于恢复时排序

### 11.2 焦点与窗口标题

[setPanelFocus()](app/src/layout/util.ts#L34-L38)

```typescript
if (element.getAttribute("data-type") === "wnd") {
    const title = element.querySelector(
        '.layout-tab-bar .item--focus[data-type="tab-header"] .item__text'
    )?.textContent || "";
    setTitle(title, title ? false : true);
}
```

窗口焦点变化时同步更新 Electron 窗口标题。

### 11.3 窗口拖拽区域

[setTabPosition()](app/src/window/setHeader.ts#L8-L46)

独立窗口的标签栏右侧空白区域作为拖拽区域（`-webkit-app-region: drag`），当标签栏滚动时动态调整，确保始终有可拖拽区域。

### 11.4 保持光标位置

当在新标签中打开文档但不切换时（`keepCursor = true`），会记录 `keep-cursor` 属性，后续切换到该标签时自动滚动到对应位置。

[Wnd.switchTab()](app/src/layout/Wnd.ts#L524-L551)

### 11.5 布局扁平化优化

[JSONToCenter()](app/src/layout/util.ts#L261-L265)

反序列化时，将连续的单孩子 Layout 节点扁平化，减少不必要的嵌套层级，提升渲染性能。

### 11.6 Linux 粘贴拦截

[Wnd.ts#L109-L122](app/src/layout/Wnd.ts#L109-L122)

Linux 系统下使用剪贴板管理特殊的粘贴事件拦截机制，通过 `#preventPast` 私有方法阻止默认粘贴行为。

---

## 十二、潜在风险与问题

### 12.1 状态一致性风险

1. **跨窗口标签拖拽竞态**：拖拽过程中原窗口和新窗口的标签状态同步依赖 IPC 消息时序，极端情况下可能出现状态不一致
2. **持久化延迟**：`saveLayout()` 有重试机制（最多 10 次），如果 Model 一直未就绪，可能导致布局保存不完整
3. **WebSocket 重连期间**：重连期间的数据变更可能丢失，需依赖重连后的全量同步
4. **独立窗口滚动位置丢失**：独立窗口关闭时**不调用 `saveScroll()`**，最新的滚动位置可能未保存到 localStorage，下次打开时滚动位置是上次 saveLayout 时的状态
5. **独立窗口崩溃**：独立窗口崩溃时，sessionStorage 中的布局状态丢失，下次打开无法恢复
6. **独立窗口布局不可恢复**：独立窗口销毁后 sessionStorage 清空，重新打开时只能从 URL 参数创建新布局，无法恢复之前的多标签/分屏状态

### 12.2 资源泄漏风险

1. **独立窗口 WebSocket 未优雅关闭**：独立窗口关闭时**不调用 `destroyModel()`**，WebSocket 连接没有发送 `closews` 消息，依赖连接断开检测，可能导致后端会话清理延迟
2. **独立窗口标签资源未显式释放**：独立窗口关闭时所有 Model 实例（Editor、Graph、Asset 等）都不会调用 `destroy()`，依赖 GC 回收，大文档场景可能内存释放不及时
3. **Custom 模型资源**：插件提供的自定义模型如果未正确实现 `destroy()` 方法，可能导致内存泄漏（独立窗口更严重，因为窗口关闭时不主动调用）
4. **关闭标签动画期间**：200ms 的关闭动画期间标签 DOM 仍存在，可能被误操作
5. **WebSocket 连接数**：每个标签一个 WebSocket 连接，独立窗口越多连接数越多，对后端造成压力
6. **插件卸载时序**：独立窗口关闭时插件 `onunload` 是异步的，如果窗口销毁太快可能导致插件清理不完整

### 12.3 异常处理风险

1. **上传中断**：正在上传时窗口被强制关闭（任务管理器结束进程），上传状态不一致
2. **网络异常**：exportLayout 的 fetchPost 失败可能导致布局未保存就关闭
3. **并发关闭**：before-quit 事件遍历所有工作区发送 save-close，可能存在时序问题
4. **事件时序竞态**：rename 事件和 syncMergeResult 事件到达顺序不确定，可能导致已激活标签标题重复更新
5. **跨窗口状态不一致**：WebSocket 推送延迟可能导致不同窗口标签状态短暂不一致
6. **插件事件同步失败**：独立窗口插件实例在 reloadPlugin 时出错，可能导致与主窗口插件状态不一致

### 12.4 性能风险

1. **全量序列化**：每次布局变化都完整序列化整个布局树，标签数量多时可能有性能影响
2. **懒加载切换开销**：首次切换到未激活标签时需要初始化 Model，可能有明显延迟
3. **全局查询**：`getAllModels()`、`getAllTabs()` 等函数使用递归遍历，布局复杂时开销较大
4. **关闭保存阻塞**：exportLayout 需等待所有编辑器 saveScroll 完成，大文档可能延迟关闭

### 12.5 可维护性风险

1. **状态分散**：窗口状态分布在 DOM 属性（data-id、data-activetime）、JS 对象（children 数组）和存储中，维护成本高
2. **类型安全**：布局 JSON 序列化/反序列化没有强类型约束，字段变更容易出错
3. **副作用链长**：一个简单的标签切换会触发 saveLayout、updatePanelByEditor、setTitle 等多个副作用
4. **消息分发分散**：IPC 消息处理分布在 onGetConfig.ts、window/index.ts 等多处，缺乏统一管理

---

## 十三、后续验证方向

### 13.1 功能验证

- [ ] 打开 100+ 标签后系统稳定性（内存、响应速度）
- [ ] 跨显示器拖拽标签窗口的行为正确性
- [ ] 网络异常时 WebSocket 重连与数据恢复
- [ ] 快速连续关闭标签的状态一致性
- [ ] 窗口最大化/最小化/还原时布局恢复
- [ ] 关闭按钮设置为"最小化到托盘"的行为正确性
- [ ] 独立窗口刷新后布局恢复（sessionStorage）
- [ ] 独立窗口关闭再重新打开后布局状态（预期：不可恢复，需重新创建）
- [ ] 独立窗口多标签滚动位置保存与恢复
- [ ] 独立窗口关闭时插件卸载的完整性
- [ ] 文档重命名后所有窗口标签标题同步更新
- [ ] 文档删除后所有窗口对应标签同步关闭
- [ ] 笔记本关闭/删除后所有窗口相关标签同步关闭
- [ ] 插件重载后主窗口和独立窗口插件状态一致性
- [ ] 快速连续修改文档标题时各窗口状态一致性
- [ ] 多窗口同时打开同一文档时，其中一个窗口删除文档的同步处理

### 13.2 性能验证

- [ ] 布局序列化性能（不同标签数量级）
- [ ] 批量关闭标签的耗时
- [ ] WebSocket 连接数对后端的影响
- [ ] 大文档标签切换的内存变化
- [ ] 关闭保存流程的总耗时（exportLayout 执行时间）

### 13.3 异常场景验证

- [ ] 进程崩溃后的数据恢复程度
- [ ] 插件加载失败时的布局降级
- [ ] 上传中断后的状态处理
- [ ] 多窗口同时编辑同一文档的冲突处理
- [ ] 极低内存下的标签卸载策略
- [ ] 网络异常时关闭保存的失败处理
- [ ] 多工作区同时关闭的时序正确性
- [ ] 独立窗口强制关闭（任务管理器）后的后端会话清理
- [ ] 独立窗口 WebSocket 连接异常断开后的后端状态
- [ ] 独立窗口插件异步卸载未完成时窗口销毁的影响
- [ ] 独立窗口大量标签关闭后的内存释放速度

### 13.4 安全验证

- [ ] 布局 JSON 注入风险（反序列化时的 XSS 可能性）
- [ ] WebSocket 连接的认证安全性
- [ ] 跨窗口消息的来源校验
- [ ] sessionStorage 布局数据的敏感信息泄露

---

## 十四、总结

SiYuan 的多窗口与标签页管理系统设计体现了桌面级应用的复杂度：

### 核心架构亮点

1. **层次化布局模型**：Layout → Wnd → Tab → Model 的四层结构灵活支持分屏和多标签
2. **懒加载优化**：未激活标签不初始化 Model，通过 `data-initdata` 延迟加载，平衡了功能与性能
3. **多维度持久化**：后端配置 + sessionStorage + localStorage 三层存储，兼顾主窗口和独立窗口
4. **双路通信**：Electron IPC 用于窗口间控制（关闭、拖拽样式同步），WebSocket 用于数据同步
5. **细粒度标签资源管理**：针对不同 Model 类型有专门的销毁逻辑，插件可自定义 `destroy()` 方法
6. **双轨关闭机制**：主窗口走"保存布局+退出应用"路径，独立窗口走"卸载插件+销毁窗口"路径，各司其职

### 三层协作模式

| 层级 | 职责 | 关键技术 |
|------|------|---------|
| Electron 主进程 | 窗口生命周期管理、IPC 中继、关闭拦截 | BrowserWindow、ipcMain、powerMonitor |
| 渲染进程 | 布局渲染、标签管理、状态同步 | Layout/Wnd/Tab/Model、WebSocket |
| Go 后端 | 配置持久化、数据广播 | conf.json、WebSocket pushMode |

### 改进空间

该系统在功能完整性上表现出色，但在以下方面仍有改进空间：

1. **独立窗口关闭流程完善**：独立窗口关闭时应调用 `exportLayout()` 保存滚动位置，或在 `saveLayout` 中集成滚动位置保存
2. **独立窗口资源释放**：独立窗口关闭时应主动遍历标签调用 `destroyModel()`，确保 WebSocket 优雅关闭和资源及时释放
3. **状态一致性**：跨窗口拖拽的竞态条件、持久化重试机制
4. **异常处理**：网络异常时的关闭保存失败回退、插件异步卸载的等待机制
5. **性能优化**：增量序列化、连接池管理 WebSocket
6. **可维护性**：统一的消息分发中心、类型安全的序列化协议
7. **独立窗口布局持久化**：可考虑将独立窗口布局也保存到后端配置，支持跨会话恢复

特别是在标签数量极大（>50）和多窗口密集交互的场景下，需要重点关注性能、状态一致性和资源释放问题。

---

## 代码核对记录

### 窗口与标签管理

| 核对项 | 状态 | 位置 |
|-------|------|------|
| Wnd 类定义 | ✅ | [Wnd.ts#L56](app/src/layout/Wnd.ts#L56) |
| Wnd.switchTab | ✅ | [Wnd.ts#L467-L572](app/src/layout/Wnd.ts#L467-L572) |
| Wnd.addTab | ✅ | [Wnd.ts#L574-L645](app/src/layout/Wnd.ts#L574-L645) |
| Wnd.removeTab | ✅ | [Wnd.ts#L887-L903](app/src/layout/Wnd.ts#L887-L903) |
| Wnd.removeTabAction | ✅ | [Wnd.ts#L772-L885](app/src/layout/Wnd.ts#L772-L885) |
| Wnd.destroyModel | ✅ | [Wnd.ts#L741-L770](app/src/layout/Wnd.ts#L741-L770) |
| Wnd.split | ✅ | [Wnd.ts#L982](app/src/layout/Wnd.ts#L982) |
| Tab.dragstart / dragend | ✅ | [Tab.ts#L84](app/src/layout/Tab.ts#L84) / [L106](app/src/layout/Tab.ts#L106) |

### 布局持久化

| 核对项 | 状态 | 位置 |
|-------|------|------|
| saveLayout | ✅ | [util.ts#L128-L168](app/src/layout/util.ts#L128-L168) |
| exportLayout | ✅ | [util.ts#L170-L209](app/src/layout/util.ts#L170-L209) |
| setPanelFocus | ✅ | [util.ts#L34-L66](app/src/layout/util.ts#L34-L66) |

### 关闭流程（两条独立路径）

| 核对项 | 状态 | 位置 |
|-------|------|------|
| siyuan-save-close 分发入口 | ✅ | [onGetConfig.ts#L175-L181](app/src/boot/onGetConfig.ts#L175-L181) |
| 主窗口关闭 winOnClose | ✅ | [onGetConfig.ts#L114-L132](app/src/boot/onGetConfig.ts#L114-L132) |
| 独立窗口关闭 closeWindow | ✅ | [closeWin.ts#L5-L13](app/src/window/closeWin.ts#L5-L13) |
| 主窗口 close 事件拦截 | ✅ | [main.js#L513-L518](app/electron/main.js#L513-L518) |
| 独立窗口 close 事件拦截 | ✅ | [main.js#L1169-L1174](app/electron/main.js#L1169-L1174) |
| 主进程 destroy 命令 | ✅ | [main.js#L1046-L1051](app/electron/main.js#L1046-L1051) |
| before-quit 事件处理 | ✅ | [main.js#L1519-L1526](app/electron/main.js#L1519-L1526) |
| exitSiYuan 退出应用 | ✅ | [processSystem.ts#L291](app/src/dialog/processSystem.ts#L291) |
| 独立窗口初始化 init | ✅ | [init.ts#L21-L85](app/src/window/init.ts#L21-L85) |

### 跨窗口同步

| 核对项 | 状态 | 位置 |
|-------|------|------|
| onWindowsMsg 跨窗口消息处理 | ✅ | [onWindowsMsg.ts#L13-L45](app/src/window/onWindowsMsg.ts#L13-L45) |
| 主进程 siyuan-send-windows 广播 | ✅ | [main.js#L1301-L1305](app/electron/main.js#L1301-L1305) |
| 锁屏事件广播 | ✅ | [main.js#L1416-L1420](app/electron/main.js#L1416-L1420) |

### 关键发现

| 发现 | 说明 |
|------|------|
| 主窗口和独立窗口关闭是两条独立路径 | ❌ 之前错误地认为是统一流程，现已纠正 |
| 独立窗口关闭不调用 exportLayout | ✅ 确认：仅卸载插件后直接 destroy |
| 独立窗口关闭不主动调用 destroyModel | ✅ 确认：依赖窗口销毁后 GC 回收 |
| 主窗口关闭不卸载插件 | ✅ 确认：依赖进程退出自动回收 |
| 独立窗口布局存 sessionStorage，不可跨会话恢复 | ✅ 确认：关闭后再打开无法恢复布局 |
