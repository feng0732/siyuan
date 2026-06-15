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

## 七、跨窗口同步

### 7.1 通信架构

```
┌─────────────────┐     ipcMain     ┌─────────────────┐
│  主窗口渲染进程  │◄───────────────►│                 │
│ (BrowserWindow) │  siyuan-cmd     │  Electron 主进程 │
└─────────────────┘                 │   (main.js)     │
         ▲                           │                 │
         │ siyuan-send-windows       │                 │
         ▼                           │                 │
┌─────────────────┐     ipcMain     │                 │
│  独立窗口渲染进程│◄───────────────►│                 │
│ (BrowserWindow) │  siyuan-cmd     │                 │
└─────────────────┘                 └─────────────────┘
         │
         │ WebSocket
         ▼
┌─────────────────┐
│   Go 后端内核    │
│  (单实例共享)    │
└─────────────────┘
```

### 7.2 IPC 消息分发入口

所有 IPC 消息的分发入口在 [onGetConfig.ts](app/src/boot/onGetConfig.ts#L175-L184)：

```typescript
// 关闭保存消息
ipcRenderer.on(Constants.SIYUAN_SAVE_CLOSE, (event, close) => {
    if (isWindow()) {
        closeWindow(app);    // 独立窗口
    } else {
        winOnClose(close);   // 主窗口
    }
});

// 跨窗口广播消息
ipcRenderer.on(Constants.SIYUAN_SEND_WINDOWS, (e, ipcData: IWebSocketData) => {
    onWindowsMsg(ipcData, app);
});
```

### 7.3 主进程广播机制

[main.js - siyuan-send-windows](app/electron/main.js#L1301-L1305)

```javascript
ipcMain.on("siyuan-send-windows", (event, data) => {
    BrowserWindow.getAllWindows().forEach(item => {
        item.webContents.send("siyuan-send-windows", data);
    });
});
```

主进程作为消息中继，将一个渲染进程的消息广播给所有窗口。

### 7.4 渲染进程消息处理

[onWindowsMsg.ts](app/src/window/onWindowsMsg.ts#L13-L45)

```typescript
export const onWindowsMsg = (ipcData: IWebSocketData, app: App) => {
    switch (ipcData.cmd) {
        case "closetab":
            // 从其他窗口拖走标签后，关闭原窗口对应标签
            const tab = getInstanceById(ipcData.data);
            if (tab && tab instanceof Tab) {
                tab.parent.removeTab(ipcData.data);
            }
            break;
        case "resetTabsStyle":
            // 拖拽时的样式同步
            if (ipcData.data === "rmDragStyle") {
                // 移除拖拽样式
                document.querySelectorAll(".layout-tab-bars--drag").forEach(...);
                document.querySelectorAll(".layout-tab-bar li[data-clone='true']").forEach(...);
            } else if (isWindow()) {
                // 独立窗口拖拽区域调整
                document.querySelectorAll(".layout-tab-bar--readonly .fn__flex-1").forEach((item: HTMLElement) => {
                    if (item.getBoundingClientRect().top <= 0) {
                        (item.style as CSSStyleDeclarationElectron).WebkitAppRegion = 
                            ipcData.data === "addRegionStyle" ? "drag" : "";
                    }
                });
            }
            break;
        case "lockscreenByMode":
            // 系统锁屏事件同步
            if (window.siyuan.config.system.lockScreenMode === 1) {
                lockScreen(app);
            }
            break;
    }
};
```

### 7.5 系统锁屏同步

[main.js - lock-screen](app/electron/main.js#L1416-L1420)

```javascript
powerMonitor.on("lock-screen", () => {
    writeLog("system lock-screen");
    BrowserWindow.getAllWindows().forEach(item => {
        item.webContents.send("siyuan-send-windows", {cmd: "lockscreenByMode"});
    });
});
```

系统锁屏事件通过 `powerMonitor` 监听，然后广播给所有窗口，由各窗口根据配置决定是否锁屏。

### 7.6 WebSocket 广播

后端通过 WebSocket 的 `pushMode` 机制实现跨会话同步：

[Model.ts - send()](app/src/layout/Model.ts#L89-L106)

```typescript
// pushMode 说明：
// 0: 所有应用所有会话广播
// 1: 自我应用会话单播
// 2: 非自我会话广播
// 4: 非自我应用所有会话广播
// 5: 单个应用内所有会话广播
// 6: 非自我应用主会话广播
```

所有窗口共享同一个后端内核，通过 WebSocket 推送实现数据实时同步：
- 文档重命名 → 所有窗口标签标题更新
- 文档删除 → 所有窗口对应标签关闭
- 笔记本关闭 → 所有窗口相关标签关闭

### 7.7 标签跨窗口拖拽

```
从窗口 A 拖拽标签到窗口 B
  ↓
dragstart [Tab.ts#L84]: 设置 dataTransfer 数据，记录 tab ID
  ↓
dragover: 窗口 B 显示放置预览
  ↓
drop: 窗口 B 接收 JSONToCenter() 创建标签
  ↓
窗口 B 发送 ipcRenderer.send("siyuan-send-windows", {cmd: "closetab", data: tabId})
  ↓
主进程广播到所有窗口 [main.js#L1301-L1305]
  ↓
窗口 A 收到 closetab 命令 [onWindowsMsg.ts#L15-L16]，移除对应标签
```

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

## 九、异常情况下的窗口关闭处理

### 9.1 关闭保存完整链路

关闭保存的完整流程涉及三层协作：

```
用户点击关闭按钮 / 系统关闭命令
  │
  ├─→ Electron 主进程拦截 close 事件
  │    │
  │    ├─→ 主窗口 [main.js#L513-L518]
  │    │    currentWindow.on("close", (event) => {
  │    │        if (!isDestroyed()) {
  │    │            send("siyuan-save-close", false);
  │    │        }
  │    │        event.preventDefault();  // 阻止默认关闭
  │    │    })
  │    │
  │    ├─→ 独立窗口 [main.js#L1169-L1174]
  │    │    win.on("close", (event) => {
  │    │        if (!isDestroyed()) {
  │    │            send("siyuan-save-close");
  │    │        }
  │    │        event.preventDefault();
  │    │    })
  │    │
  │    └─→ 应用退出 [main.js#L1519-L1526]
  │         app.on("before-quit", (event) => {
  │             workspaces.forEach(item => {
  │                 event.preventDefault();
  │                 send("siyuan-save-close", true);
  │             });
  │         })
  │
  ├─→ 渲染进程接收 [onGetConfig.ts#L175-L181]
  │    ipcRenderer.on("siyuan-save-close", (event, close) => {
  │        if (isWindow()) {
  │            closeWindow(app);    // 独立窗口
  │        } else {
  │            winOnClose(close);   // 主窗口
  │        }
  │    })
  │
  ├─→ 主窗口关闭处理 [onGetConfig.ts#L114-L132]
  │    const winOnClose = (close = false) => {
  │        exportLayout({
  │            cb() {
  │                if (closeButtonBehavior === 1 && !close) {
  │                    // 最小化到托盘
  │                } else {
  │                    exitSiYuan();  // 真正退出
  │                }
  │            },
  │            errorExit: true
  │        });
  │    }
  │
  ├─→ 独立窗口关闭处理 [closeWin.ts#L5-L13]
  │    export const closeWindow = async (app: App) => {
  │        // 卸载插件
  │        for (let i = 0; i < app.plugins.length; i++) {
  │            try {
  │                await app.plugins[i].onunload();
  │            } catch (e) { console.error(e); }
  │        }
  │        // 发送销毁命令
  │        ipcRenderer.send(Constants.SIYUAN_CMD, "destroy");
  │    }
  │
  ├─→ 布局持久化 [util.ts#L170-L209]
  │    export const exportLayout = async (options) => {
  │        // 1. 保存所有编辑器滚动位置
  │        const editors = getAllModels().editor;
  │        for (let i = 0; i < editors.length; i++) {
  │            await saveScroll(editors[i].editor.protyle);
  │        }
  │        // 2. 序列化布局
  │        layoutToJSON(window.siyuan.layout.layout, layoutJSON.layout);
  │        // 3. 保存到后端或 sessionStorage
  │        if (isWindow()) {
  │            sessionStorage.setItem("layout", JSON.stringify(layoutJSON));
  │            options.cb();
  │        } else {
  │            fetchPost("/api/system/setUILayout", ..., () => options.cb());
  │        }
  │    }
  │
  └─→ 主进程真正销毁 [main.js#L1046-L1051]
       case "destroy":
           if (!currentWindow.isDestroyed()) {
               currentWindow.destroy();
           }
           break;
```

### 9.2 异常场景处理

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

### 10.4 窗口关闭保存完整流程

```
用户触发关闭
  │
  ├─→ 主进程 close 事件 [main.js#L513]
  │    └─ send("siyuan-save-close")
  │
  ├─→ 渲染进程接收 [onGetConfig.ts#L175]
  │    ├─ 独立窗口：closeWindow(app)
  │    └─ 主窗口：winOnClose(close)
  │
  ├─→ exportLayout() 保存布局 [util.ts#L170]
  │    ├─ 保存所有编辑器滚动位置
  │    ├─ 序列化布局 JSON
  │    └─ 保存到后端 / sessionStorage
  │
  ├─→ 回调 cb() 执行后续动作
  │    ├─ 最小化到托盘 / exitSiYuan()
  │    └─ 独立窗口：卸载插件
  │
  └─→ ipcRenderer.send("siyuan-cmd", "destroy")
       └─ 主进程 destroy 命令 [main.js#L1046]
            └─ currentWindow.destroy()
```

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
4. **独立窗口崩溃**：独立窗口崩溃时，sessionStorage 中的布局状态丢失，下次打开无法恢复

### 12.2 资源泄漏风险

1. **Custom 模型资源**：插件提供的自定义模型如果未正确实现 `destroy()` 方法，可能导致内存泄漏
2. **关闭标签动画期间**：200ms 的关闭动画期间标签 DOM 仍存在，可能被误操作
3. **WebSocket 连接数**：每个标签一个 WebSocket 连接，标签数量多时连接数较多，对后端造成压力
4. **插件卸载时机**：窗口关闭时插件 `onunload` 是异步的，如果窗口销毁太快可能导致清理不完整

### 12.3 异常处理风险

1. **上传中断**：正在上传时窗口被强制关闭（任务管理器结束进程），上传状态不一致
2. **网络异常**：exportLayout 的 fetchPost 失败可能导致布局未保存就关闭
3. **并发关闭**：before-quit 事件遍历所有工作区发送 save-close，可能存在时序问题

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
5. **细粒度资源管理**：针对不同 Model 类型有专门的销毁逻辑，插件可自定义 `destroy()` 方法
6. **完整的关闭保存链路**：主进程拦截 → 渲染进程处理 → 持久化 → 回调销毁，确保数据安全

### 三层协作模式

| 层级 | 职责 | 关键技术 |
|------|------|---------|
| Electron 主进程 | 窗口生命周期管理、IPC 中继、关闭拦截 | BrowserWindow、ipcMain、powerMonitor |
| 渲染进程 | 布局渲染、标签管理、状态同步 | Layout/Wnd/Tab/Model、WebSocket |
| Go 后端 | 配置持久化、数据广播 | conf.json、WebSocket pushMode |

### 改进空间

该系统在功能完整性上表现出色，但在以下方面仍有改进空间：

1. **状态一致性**：跨窗口拖拽的竞态条件、持久化重试机制
2. **异常处理**：网络异常时的关闭保存失败回退、插件异步卸载的等待机制
3. **性能优化**：增量序列化、连接池管理 WebSocket
4. **可维护性**：统一的消息分发中心、类型安全的序列化协议

特别是在标签数量极大（>50）和多窗口密集交互的场景下，需要重点关注性能和状态一致性问题。

---

## 代码核对记录

| 核对项 | 状态 | 备注 |
|-------|------|------|
| Wnd 类定义行号 | ✅ | [Wnd.ts#L56](app/src/layout/Wnd.ts#L56) |
| Wnd.switchTab 行号 | ✅ | [Wnd.ts#L467-L572](app/src/layout/Wnd.ts#L467-L572) |
| Wnd.addTab 行号 | ✅ | [Wnd.ts#L574-L645](app/src/layout/Wnd.ts#L574-L645) |
| Wnd.removeTab 行号 | ✅ | [Wnd.ts#L887-L903](app/src/layout/Wnd.ts#L887-L903) |
| Wnd.removeTabAction 行号 | ✅ | [Wnd.ts#L772-L885](app/src/layout/Wnd.ts#L772-L885) |
| Wnd.destroyModel 行号 | ✅ | [Wnd.ts#L741-L770](app/src/layout/Wnd.ts#L741-L770) |
| Wnd.split 行号 | ✅ | [Wnd.ts#L982](app/src/layout/Wnd.ts#L982) |
| Tab.dragstart/dragend 行号 | ✅ | [Tab.ts#L84](app/src/layout/Tab.ts#L84) / [L106](app/src/layout/Tab.ts#L106) |
| util.saveLayout 行号 | ✅ | [util.ts#L128-L168](app/src/layout/util.ts#L128-L168) |
| util.exportLayout 行号 | ✅ | [util.ts#L170-L209](app/src/layout/util.ts#L170-L209) |
| util.setPanelFocus 行号 | ✅ | [util.ts#L34-L66](app/src/layout/util.ts#L34-L66) |
| siyuan-save-close 分发入口 | ✅ | [onGetConfig.ts#L175-L181](app/src/boot/onGetConfig.ts#L175-L181) |
| winOnClose 主窗口关闭逻辑 | ✅ | [onGetConfig.ts#L114-L132](app/src/boot/onGetConfig.ts#L114-L132) |
| closeWindow 独立窗口关闭 | ✅ | [closeWin.ts#L5-L13](app/src/window/closeWin.ts#L5-L13) |
| onWindowsMsg 跨窗口消息 | ✅ | [onWindowsMsg.ts#L13-L45](app/src/window/onWindowsMsg.ts#L13-L45) |
| 主进程 siyuan-send-windows | ✅ | [main.js#L1301-L1305](app/electron/main.js#L1301-L1305) |
| 主进程 destroy 命令 | ✅ | [main.js#L1046-L1051](app/electron/main.js#L1046-L1051) |
| 主窗口 close 事件拦截 | ✅ | [main.js#L513-L518](app/electron/main.js#L513-L518) |
| 独立窗口 close 事件拦截 | ✅ | [main.js#L1169-L1174](app/electron/main.js#L1169-L1174) |
| before-quit 事件处理 | ✅ | [main.js#L1519-L1526](app/electron/main.js#L1519-L1526) |
| 锁屏事件广播 | ✅ | [main.js#L1416-L1420](app/electron/main.js#L1416-L1420) |
