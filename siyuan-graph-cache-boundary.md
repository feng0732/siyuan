# SiYuan 关系图前端展示链路与引用缓存失效边界分析

> 本文为 `siyuan-backlink-index.md` 与 `siyuan-backlink-index-followup.md` 的补充，聚焦关系图面板端到端展示链路（请求/渲染/增量加载/跟随刷新）和引用缓存旧引用清理完整性核对。代码引用使用可点击绝对路径格式。

---

## 目录

- [1. 关系图面板前端架构总览](#1-关系图面板前端架构总览)
- [2. 请求链路：从 UI 交互到 API 响应](#2-请求链路从-ui-交互到-api-响应)
- [3. 渲染与增量加载流程](#3-渲染与增量加载流程)
- [4. 跟随编辑器刷新：事件驱动机制](#4-跟随编辑器刷新事件驱动机制)
- [5. 引用缓存失效入口全追踪](#5-引用缓存失效入口全追踪)
- [6. 旧引用清理完整性核对](#6-旧引用清理完整性核对)
- [7. 附：关键绝对路径索引](#7-附关键绝对路径索引)

---

## 1. 关系图面板前端架构总览

### 1.1 组件分层

关系图面板实现位于 [Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts)，基于三层继承架构：

```
Model (抽象基类) [layout/Model.ts]
  └─ Graph (业务实现) [layout/dock/Graph.ts]
        ├─ 面板控件 (18+ 个配置项 input/checkbox/slider)
        ├─ vis.Network (第三方库 vis-network 9.1.13)
        └─ BlockPanel / openFileById (点击节点交互)
```

Model 基类 [Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/Model.ts#L33-L87) 负责 WebSocket 连接管理：连接到 `/ws` 端点并携带 `app&id&type` 参数，实现断线 3 秒自动重连。

### 1.2 三种图类型

| 类型 | `this.type` | API 接口 | 必填参数 |
|------|------------|---------|---------|
| **全局关系图** | `"global"` | `/api/graph/getGraph` | 无 |
| **局部关系图（当前文档）** | `"local"` | `/api/graph/getLocalGraph` | `blockId`, `rootId` |
| **Pin 关系图（聚焦块）** | `"pin"` | `/api/graph/getLocalGraph` | `blockId` |

### 1.3 面板配置项矩阵

`Graph.ts` 构造函数中动态生成 18+ 个配置控件，local 和 global 仅在 `minRefs` 滑块上有差异：

| 配置项 | global | local | 对应后端字段 |
|-------|:------:|:-----:|-------------|
| heading/list/listItem/blockquote | ✅ | ✅ | `conf.type.*` |
| callout/super/table/math/code | ✅ | ✅ | `conf.type.*` |
| paragraph/tag/dailyNote | ✅ | ✅ | `conf.type.tag` + `conf.dailyNote` |
| arrow | ✅ | ✅ | `conf.d3.arrow` |
| minRefs (滑块 0-16) | ✅ | ❌ | `conf.minRefs` |
| nodeSize / linkWidth / lineOpacity | ✅ | ✅ | `conf.d3.*` |
| centerStrength / collideRadius | ✅ | ✅ | `conf.d3.*` |
| collideStrength / linkDistance | ✅ | ✅ | `conf.d3.*` |

---

## 2. 请求链路：从 UI 交互到 API 响应

### 2.1 触发搜索的 11 个入口

[Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts#L54-L417) 中 `searchGraph()` 被以下场景触发：

| 触发源 | 位置 | 说明 |
|--------|------|------|
| **构造完成** | L376 | 面板初始化时首次请求，`focus=options.type!=="global"` |
| **ws mount 事件** | L57-L61 | 全局图收到 `cmd="mount"` 且 `code!=1` 时刷新 |
| **ws rename 事件** | L62-L72 | 文档重命名时 local/global 均刷新 |
| **搜索框输入** | L352-L364 | `input` + `compositionend` 事件，中文输入法防抖 |
| **刷新按钮** | L328-L330 | 用户手动点击，`refresh=true` |
| **类型切换 checkbox** | L371-L375 | 任何 block 类型开关变更 |
| **滑块变更** | L365-L370 | 所有 D3 力导向参数滑块 |
| **reset 按钮** | L416 | `reset()` 方法末尾调用 |
| **配置变更** | L302-L312 | 重置配置 API 返回后调用 |
| **防重入保护** | L421-L423 | 刷新 icon 在旋转中（请求未完成）时直接 return |

### 2.2 防重复请求与 Pin 图聚焦判定

[Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts#L419-L499) 的 `searchGraph` 方法设计了两层保护：

```typescript
// 第一层：请求期间刷新 icon 旋转态，防止重复点击
const element = this.element.querySelector('.block__icon[data-type="refresh"] svg');
if (element.classList.contains("fn__rotate") && !id) {
    return;
}
element.classList.add("fn__rotate");

// 第二层：Pin 图只有当该文档块当前被聚焦的标签页打开时才渲染
if (!refresh && this.type === "pin" && this.blockId) {
    const isActive = Array.from(document.querySelectorAll(".fn__flex > .layout-tab-bar > .item--focus")).find(activeElement => {
        const tab = getInstanceById(activeElement.getAttribute("data-id"));
        if (tab instanceof Tab && tab.model instanceof Editor) {
            // 匹配 rootID / parentID / blockID 任一
            if (tab.model.editor.protyle.block.rootID === this.blockId ||
                tab.model.editor.protyle.block.parentID === this.blockId ||
                tab.model.editor.protyle.block.id === this.blockId) {
                return true;
            }
        }
    });
    if (!isActive) { return; }  // 不渲染，也不移除旋转态
}
```

### 2.3 请求参数与响应结构

**全局图请求** ([Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts#L448-L463))：
```typescript
fetchPost("/api/graph/getGraph", {
    k: this.inputElement.value,
    conf: { type, d3, dailyNote, minRefs }
}, response => {
    this.graphData = response.data;   // { nodes, links, conf, box, reqId }
    window.siyuan.config.graph.global = response.data.conf;
    this.onGraph(false);
    element.classList.remove("fn__rotate");
});
```

**局部图请求** ([Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts#L464-L498))：
```typescript
fetchPost("/api/graph/getLocalGraph", {
    type: this.type,  // "local" | "pin" — 防刷新重复处理 hack
    k: this.inputElement.value,
    id: id || this.blockId,
    conf: { type, d3, dailyNote },
});
```

响应结构统一由后端 [graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/api/graph.go#L96-L103) 定义：
```typescript
{
    nodes: GraphNode[],  // { id, box, path, size, title?, label, type, refs, defs }
    links: GraphLink[],  // { from, to, ref, arrows? }
    conf:  IGraphConfig, // 后端合并后的实际配置
    box:   string,       // 笔记本 ID
    reqId: number        // 请求追踪 ID
}
```

后端入口还支持 200-500ms 的 `RandomSleep` 防止请求风暴 ([graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/api/graph.go#L103-L104))。

---

## 3. 渲染与增量加载流程

### 3.1 onGraph 渲染主流程

[Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts#L518-L781) `onGraph(hl: boolean)` 方法完整流程：

```
onGraph(hl)
  │
  ├─ 0. 容器可见性检查 — clientHeight===0 时直接 return（面板关闭态不渲染）
  │
  ├─ 1. 销毁旧 network 实例
  │     this.network?.destroy()
  │
  ├─ 2. 空数据短路返回
  │
  ├─ 3. 节点着色：13 种块类型映射 CSS 变量色
  │     ├─ NodeDocument     → --b3-graph-doc-point
  │     ├─ NodeHeading      → --b3-graph-heading-point
  │     ├─ NodeParagraph    → --b3-graph-p-point
  │     ├─ tag/textmark tag → --b3-graph-tag-point
  │     └─ 其他 9 种        → 各有专用 CSS 变量
  │
  ├─ 4. 边着色：
  │     ├─ item.ref=true  → --b3-graph-ref-line  (引用边)
  │     └─ item.ref=false → --b3-graph-line      (父子层级边)
  │
  ├─ 5. 动态加载 vis-network 脚本 (CDN)
  │     addScript("vis-network.min.js?v=9.1.13")
  │
  ├─ 6. 构建 Physics 参数（按节点数自适应）：
  │     ├─ timestep:        节点>32 ? 0.1 : 0.5
  │     ├─ maxVelocity:     clamp(节点数, 256, 1024)
  │     ├─ minVelocity:     clamp(节点数, 8, 64)
  │     ├─ solver:          forceAtlas2Based
  │     └─ stabilization:   64 iterations, fit=true
  │
  ├─ 7. 初始缩放比例 initialScale
  │     Math.max(0.03, 1 - 0.3 * floor(nodes/128))
  │
  ├─ 8. 增量分批加载节点与边（核心！）
  │     ├─ initial batch:   max(ceil(n*0.1), 128) 个节点和边立即加入
  │     │     nodes = new vis.DataSet(this.graphData.nodes.slice(0, i))
  │     │     edges = new vis.DataSet(this.graphData.links.slice(0, j))
  │     │
  │     ├─ 节点定时器 intervalNode (间隔 32-256ms, 每批 64-256 个)
  │     │     network.body.data.nodes.add(nodesAdded)
  │     │
  │     └─ 边定时器 intervalEdge (固定 256ms, 每批同大小)
  │           network.body.data.edges.add(edgesAdded)
  │           └─ 全部加载完后 network.fit() 自动居中
  │
  └─ 9. 注册事件回调
        ├─ stabilizationIterationsDone → stopSimulation + hlNode(hl)
        ├─ dragEnd → 3s 后 stopSimulation（防抖）
        └─ click → 根据修饰键执行不同导航
```

### 3.2 增量分批加载算法详解

[Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts#L666-L716) 的分批策略是性能关键：

```typescript
// 计算参数
let i = Math.max(Math.ceil(this.graphData.nodes.length * 0.1), 128);  // 初始 10% 节点，至少 128
let j = Math.max(Math.ceil(this.graphData.links.length * 0.1), 128);

// 节点批量大小：time=256ms, 计算每批数量
const time = 256;
const intervalNodeTime = Math.max(Math.ceil(time / 8), 32);  // 32~256ms
let batch = this.graphData.nodes.length / time / 2;
batch = Math.max(Math.min(batch, 256), 64);  // clamp 64~256

// 两个定时器并行添加
const intervalNode = setInterval(() => {
    const nodesAdded = this.graphData.nodes.slice(i, i + batch);
    if (nodesAdded.length === 0) { clearInterval(intervalNode); return; }
    network.body.data.nodes.add(nodesAdded);
    i += batch;
}, intervalNodeTime);

const intervalEdge = setInterval(() => {
    const edgesAdded = this.graphData.links.slice(j, j + batch);
    if (edgesAdded.length === 0) {
        clearInterval(intervalEdge);
        network.fit({ animation: true });   // 全部就绪后自适应布局
        return;
    }
    network.body.data.edges.add(edgesAdded);
    j += batch;
}, time);
```

**设计要点：**
- **初始 10% + 分批追加**：避免 2000+ 节点一次渲染导致的 2-5s 主线程阻塞
- **节点比边更快**：节点定时器间隔 32-256ms（节点间隔 8 倍压缩），边固定 256ms，保证"先有节点再连线"的视觉顺序
- **批次大小自适应**：小图（<32K 节点）每批 64 个，大图每批最多 256 个，平衡流畅度与总耗时
- **自适应居中**：边全部加载完后统一 `network.fit()`，避免中途频繁重算布局

### 3.3 节点点击交互

[Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts#L729-L779) 根据修饰键执行 5 种导航：

| 按键 | 动作 |
|------|------|
| **无修饰** | `openFileById` 默认位置打开文档 |
| **Shift** | `openFileById` `position="bottom"` 分屏下方 |
| **Alt** | `openFileById` `position="right"` 分屏右侧 |
| **Ctrl** | `new BlockPanel` 浮窗展示块面板 |
| **标签节点** | `openGlobalSearch` 用 `#tag#` 全局搜索 |

所有打开操作前都会经过 `checkFold()` 判断是否需先展开折叠标题。

---

## 4. 跟随编辑器刷新：事件驱动机制

### 4.1 关系图 WebSocket 消息处理矩阵

[Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts#L54-L86) `msgCallback` 处理 4 种 cmd：

| `data.cmd` | 触发条件 | 处理逻辑 |
|------------|---------|---------|
| **`mount`** | 内核 WS 连接建立时的通知 | 全局图：`code!==1` 时 `searchGraph(false)` 刷新 |
| **`rename`** | 文档重命名（标题变更） | Local：`data.box === graphData.box && data.data.id === rootId` → 刷新 + 更新标题<br>Global：无条件刷新 |
| **`closeBox`** / **`removeBox`** | 笔记本关闭/移除 | Local：`graphData.box === data.data.box` → 移除整个 Tab |
| **`removeDoc`** | 文档被删除 | Local：`rootId ∈ data.data.ids` → 移除整个 Tab |

### 4.2 关系图刷新的缺口：缺少实时引用变更推送

**关键发现**：关系图的 msgCallback **没有订阅以下与引用变更直接相关的 WS 消息**：

| 相关推送命令 | 定义位置 | 关系图是否处理 |
|-------------|---------|:-------------:|
| `setDefRefCount` | [websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/util/websocket.go#L358-L360) | ❌ 未处理 |
| `setRefDynamicText` | [websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/util/websocket.go#L354-L356) | ❌ 未处理 |
| `savedoc` | [websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/util/websocket.go#L336-L344) | ❌ 未处理（Outline 面板处理了） |
| `databaseIndexCommit` | [queue.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/queue.go#L177) | ❌ 未处理（后端仅 BroadcastByType "main"） |

这些命令由主 WS 连接（type=main）统一处理，处理入口在 [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/index.ts#L89-L116)：

```typescript
case "setDefRefCount":
    setDefRefCount(data.data);       // 只更新编辑器内的角标，不通知关系图
    break;
case "setRefDynamicText":
    setRefDynamicText(data.data);    // 只更新编辑器内的锚文本 DOM
    break;
case "reloaddoc":
    reloadSync(/* 仅影响文档树 */);
    break;
```

[processSystem.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/dialog/processSystem.ts#L154-L195) 的 `setDefRefCount` 仅遍历 `getAllEditor()` 查询并更新 DOM，**不通过事件总线通知任何 Dock 面板**：

```typescript
export const setDefRefCount = (data: {...}) => {
    getAllEditor().forEach(editor => {
        // 仅更新 protyle 标题上的 .protyle-attr--refcount 角标
        // 和块上的 protyle-attr--refcount
        // 不触发 Graph.searchGraph()
    });
};
```

### 4.3 实际刷新行为总结

| 场景 | 关系图是否自动刷新 | 用户感知 |
|------|:-----------------:|---------|
| 文档重命名（标题变） | ✅ 自动（rename cmd） | 无需操作 |
| 笔记本/文档被删除 | ✅ 自动（关闭 Tab） | 正常关闭 |
| 编辑块内容（引用的锚文本变） | ❌ 不自动 | 需点刷新按钮或重开关面板 |
| 新增/删除引用关系 | ❌ 不自动 | 同上 |
| 引用计数变化（加了一个引用） | ❌ 不自动 | 同上 |
| 打开面板瞬间（mount） | ✅ 自动 | 正常 |

**结论：关系图是"按需刷新"而非"实时同步"。** 引用变更仅在用户主动触发 searchGraph（按钮/输入/配置变更）时才从后端重新拉取最新数据。由于后端每次 `BuildTreeGraph/BuildGraph` 均直接查 `refs` 表（不经过 go-cache），只要 SQL FlushQueue 已完成，看到的就是最新的。

---

## 5. 引用缓存失效入口全追踪

### 5.1 缓存层级回顾

| 层级 | 变量 | 类型 | Key | 关联失效 |
|------|------|------|-----|---------|
| **L1 引用缓存** | `defIDRefsCache` | go-cache (30min TTL) | `defBlockID` → `map[refBlockID]*Ref` | `removeRefCacheByDefID` / `ClearCache` |
| **L2 动态锚文本** | `DynamicRefTexts` | sync.Map | `defBlockID` → `refText` | 进程重启即失效（无显式删除） |
| **L3 块缓存** | `blockCache` | ristretto LRU | `blockID` → `Block` | `removeBlockCache` → 级联触发 L1 失效 |
| **L4 IAL 缓存** | `PutBlockIAL` | sync.Map (cache 包) | `blockID` → `map[string]string` | `RemoveBlockIAL` |

### 5.2 显式失效入口总表

通过全仓搜索 `removeBlockCache`、`removeRefCacheByDefID` 和 `ClearCache`，定位以下 **9 个入口**：

#### 5.2.1 `removeBlockCache(id)` → 级联清理 L1+L3

定义位置：[cache.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/cache.go#L79-L82)
```go
func removeBlockCache(id string) {
    blockCache.Del(id)
    removeRefCacheByDefID(id)   // 级联：删除块缓存必须同时清理它作为 def 的引用缓存
}
```

**调用者 1：deleteBlocksByIDs**
[database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go#L968-L1018)
- 场景：按块 ID 数组批量删除（列表项断开等局部删除）
- 清理：遍历每个 ID 单独调 `removeBlockCache(id)`
- **问题**：仅清理了被删块自身的 L1+L3，未清理「引用了被删块的其他文档」的引用缓存（这部分引用已失效，但缓存仍保留）

**调用者 2：updateRootContent**
[block.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/block.go#L61-L80)
- 场景：文档根块内容更新（rename 操作导致标题变更）
- 清理：`removeBlockCache(id)` + `cache.RemoveBlockIAL(id)`

#### 5.2.2 `ClearCache()` → 全量清理 L1+L3

定义位置：[cache.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/cache.go#L48-L50)
```go
func ClearCache() {
    blockCache.Clear()
    // 注意：不清理 defIDRefsCache — 它另有 TTL 过期机制
}
```

**注意**：`ClearCache` **只清 L3 块缓存，不清 L1 引用缓存！** L1 仅依赖 `removeRefCacheByDefID` 显式删除或 30min TTL。

**调用者 3：InitDatabase**
[database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go#L78-L81) 启动时清理

**调用者 4：deleteBlocksByBoxTx**
[database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go#L1020-L1037) 整个笔记本删除后清理

**调用者 5：deleteByRootID**
[database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go#L1140-L1158) 单文档删除

**调用者 6：batchDeleteByRootIDs**
[database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go#L1161-L1206) 批量文档删除

**调用者 7：batchDeleteByPathPrefix**
[database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go#L1208-L1246) 按路径前缀批量删除（目录删除）

**调用者 8：batchUpdatePath**
[database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go#L1248-L1294) 文档移动（路径更新）

**调用者 9：batchUpdateHPath**
[database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go#L1296-L1318) 文档重命名（层级路径更新）

#### 5.2.3 FlushQueue 中的大批量临时禁用

[queue.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/queue.go#L125-L129) 当单次 Flush 操作数 > 512 时：
```go
if 512 < len(ops) {
    disableCache()     // cacheDisabled = true，所有 put/get 直接短路
    defer enableCache()
}
```
这是性能优化：批量操作时绕过缓存写穿透开销，但**不主动清理**。

### 5.3 L2 `DynamicRefTexts` (sync.Map) 的缺失

通过代码全仓搜索，**未发现任何对 `DynamicRefTexts.Delete()` 的调用**。该 Map 只在 `SetDynamicBlockRefText` 中写入，永不显式删除。失效机制为：
- 进程重启：自然清空
- 新写入覆盖：同 defID 有新锚文本时，Store 覆盖旧值
- 死条目残留：被删块的 defID 永久留在内存中（直到进程重启），但 `doUpdate` 中判断 `ok && ""!=dRefText` 时才使用，因此无功能性错误，仅少量内存泄漏

---

## 6. 旧引用清理完整性核对

### 6.1 场景矩阵：14 种变更场景的清理核对

对每个引用变更场景，从「DB refs 表」「L1 go-cache」「L2 DynamicRefTexts」「L3 ristretto」「L4 IAL Map」五个维度评估完整性：

| # | 变更场景 | DB refs 表 | L1 go-cache | L2 sync.Map | L3 ristretto | L4 IAL Map | 评估 |
|---|---------|:---------:|:----------:|:-----------:|:-----------:|:----------:|:----:|
| 1 | **doUpdate 编辑块内容（内含引用）** | ✅ (upsertRefs 先删后插) | ⚠️ 仅写入新 def，不删除旧 def | ✅ 新值 Store 覆盖 | ✅ 不涉及块删除 | ✅ PutBlockIAL | ⚠️ L1 残留旧 defID |
| 2 | **doUpdate 移除了某个引用** | ✅ (同上) | ❌ 被移除 defID 的 L1 条目未清 | ❌ 不移除 | ✅ | ✅ | ❌ L1/L2 残留 |
| 3 | **doInsert 插入含引用的块** | ✅ (commit 后 upsertRefs) | ❌ 不写 L1 | ❌ 不写 | ✅ | ✅ | ❌ L1 缺失（需 miss 回填） |
| 4 | **doDelete 删除含引用的块** | ✅ (deleteBlocksByIDs → upsertTree) | ⚠️ 仅清被删块自身 id=defID | ❌ | ✅ | ✅ | ⚠️ 引用了被删块的 L1 未清 |
| 5 | **整个文档被删 (deleteByRootID)** | ✅ DELETE FROM refs WHERE root_id=? | ❌ 只清 L3，不清 L1 | ❌ | ✅ (ClearCache) | ❌ | ❌ L1/L2 全量残留 |
| 6 | **批量删文档 (batchDeleteByRootIDs)** | ✅ WHERE root_id IN (...) | ❌ 同上 | ❌ | ✅ (ClearCache) | ❌ | ❌ L1/L2 全量残留 |
| 7 | **目录批量删除** | ✅ WHERE path LIKE prefix% | ❌ 同上 | ❌ | ✅ (ClearCache) | ❌ | ❌ L1/L2 全量残留 |
| 8 | **删整个笔记本 (deleteByBoxTx)** | ✅ DELETE refs WHERE box=? | ❌ 同上 | ❌ | ✅ (ClearCache) | ❌ | ❌ L1/L2 全量残留 |
| 9 | **文档重命名 (rename)** | ✅ UPDATE refs SET def_block_path=? | ❌ | ❌ | ✅ (ClearCache) + removeBlockCache(rootID) | ✅ | ⚠️ L1 未动（路径变更不影响 key=defID） |
| 10 | **文档移动 (move)** | ✅ UPDATE refs SET box/path=? + def_block_path=? | ❌ | ❌ | ✅ (ClearCache) | ❌ | ⚠️ L1 未动（key 仍是正确的） |
| 11 | **动态锚文本级联刷新** | ✅ UpdateRefsTreeQueue → upsertRefs | ✅ CacheRef 写入（若 doUpdate 路径） | ✅ Store 新值 | ✅ putBlockCache | ❌ | ✅ 写入路径完整 |
| 12 | **手动 reindex 全量重建** | ✅ 全表删除重建 | ✅ 30min TTL 或 miss 覆盖 | ❌ | ❌（未显式调用 ClearCache？需看调用链） | ❌ | ⚠️ 部分缓存不清 |
| 13 | **引用计数刷新** | 只读 | ✅ FlushQueue 保证读正确 | ❌ | ❌ | ❌ | 只读不写 |
| 14 | **关系图查询 (BuildGraph)** | 只读 | ❌（不查 L1，直查 DB refs 表） | ❌ | ❌ | ❌ | ✅ 不依赖缓存故正确 |

### 6.2 清理缺口汇总

#### 缺口 A：`ClearCache()` 不清理 `defIDRefsCache`（L1）

这是最大的缺口。[cache.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/cache.go#L48-L50) 的 ClearCache 只清 `blockCache`（ristretto），`defIDRefsCache` 只能：
1. 等 30 分钟 TTL 自然过期
2. 等 `removeRefCacheByDefID(defID)` 精确删除（仅 `removeBlockCache(id)` 会触发，适用面窄）
3. 等 `GetRefsCacheByDefID` miss 后从 DB 回填（但旧条目仍占内存直到 TTL）

**影响**：场景 5/6/7/8（文档/目录/笔记本删除）后，大量失效的引用条目残留在 L1 中，最长可达 30 分钟。若恰好有查询 `GetRefsCacheByDefID` 命中这些残留，会返回「已被删除文档」的旧引用列表。

#### 缺口 B：`DynamicRefTexts`（L2）永不清零

`SetDynamicBlockRefText` 只有 Store 没有 Delete。场景 2/4/5/6/7/8 都会产生死键（def 已不存在，但 Map 仍保留）。

**影响**：纯内存泄漏，每百万引用约消耗几 MB，但对正确性无影响（doUpdate 中使用前判断 `ok`，且键是 blockID 不会冲突）。

#### 缺口 C：`removeBlockCache(id)` 仅单向清理

[cache.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/cache.go#L79-L82) 只清理「该块作为 def」的 L1 条目，但不清理「该块作为 ref」——即引用了被删块的那些文档（它们在 L1 中作为 key=otherDefID 的 map value 存在残留）。

**影响**：场景 4（块被删）后，查询其他 defID 仍可能读到包含不存在 blockID 的 Ref 条目，导致关系图或反链面板显示"悬垂引用"（不过 Ref 结构体中存了 Content，视觉上仍能看到文本，只是点击跳转会报错）。

#### 缺口 D：IAL 缓存（L4）在批量操作中未清理

场景 5/6/7（删除文档）和 10（移动）调用的 `ClearCache` 不涉及 `cache.RemoveBlockIAL`，IAL sync.Map 留死键。类似缺口 B，内存泄漏为主，正确性影响较小。

### 6.3 风险严重度评估

| 缺口 | 严重度 | 触发频率 | 对用户可见影响 | 建议修复方向 |
|------|:------:|:-------:|--------------|-------------|
| A — ClearCache 不清 L1 | **中** | 高（删文档常见） | 反链面板最多 30 分钟显示已删文档的过时引用 | `ClearCache` 追加 `defIDRefsCache.Flush()` |
| B — DynamicRefTexts 永不清理 | **低** | 持续积累 | 仅内存增长，用户不可见 | 定期遍历 + DB 校验清理，或接入文档删除事件 |
| C — removeBlockCache 单向 | **中高** | 中（删块偶尔） | 悬垂引用（跳转报错但文本残留） | 删除块时遍历 refs 表反查被引用方 defID 并清理 |
| D — IAL 未批量清理 | **低** | 同 A | 内存增长 + 极端情况下返回已删块的旧属性 | `ClearCache` 追加 IAL 的整体清理接口 |

### 6.4 为什么用户几乎感知不到这些缺口？

1. **后端查询的回退策略**：`GetRefsCacheByDefID` 的 miss 会直查 DB 并回填，即使 L1 部分失效也能被下次正确覆盖
2. **关系图直查 DB**：`BuildGraph`/`BuildTreeGraph` 使用 `sql.DefRefs`/`QueryRefRootBlocksByDefRootIDs` 全走 SQL，不经过 L1 缓存（参考 [graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/model/graph.go#L176-L197)）
3. **30 分钟 TTL + 定期 Flush**：即使残留，最长 30 分钟也会自然失效
4. **反链面板的 DB 兜底**：`GetBacklink2` 主要依赖实时 SQL 查询，缓存仅用于计数优化

**真正受影响的只有 `refreshDynamicRefTexts0` 中调用 `getRefsCacheByDefNode` → `GetRefsCacheByDefID` 这条链**：在缺口 A 的窗口内可能漏掉部分引用而无法正确触发锚文本级联刷新。但 7 次级联 + 用户下一次编辑 + miss 回填三重保障下，问题几乎会自修复。

---

## 7. 附：关键绝对路径索引

### 7.1 关系图前端

| 绝对路径 | 行范围 | 内容摘要 |
|---------|--------|---------|
| [Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts) | L19-L92 | Graph 类结构：三种 type (local/pin/global)、构造函数、ws 回调 |
| [Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts) | L93-L291 | 18+ 配置控件 HTML 动态生成、事件绑定 |
| [Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts) | L419-L499 | `searchGraph()`：请求构建、防重入、Pin 图聚焦判定、fetchPost 调用 |
| [Graph.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/dock/Graph.ts) | L518-L781 | `onGraph()`：着色、vis-network 初始化、**增量分批加载算法**、交互回调 |
| [Model.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/layout/Model.ts) | L33-L87 | WebSocket 连接管理：onopen/onmessage/onclose (3s 重连) |
| [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/index.ts) | L89-L116 | 主 WS 消息分发：setDefRefCount / setRefDynamicText / reloaddoc |
| [processSystem.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/dialog/processSystem.ts) | L154-L195 | `setRefDynamicText()` / `setDefRefCount()`：仅更新编辑器 DOM，不触发图刷新 |
| [processMessage.ts](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/app/src/util/processMessage.ts) | L10-L77 | 全局消息预处理：msg/cmsg/cprogress/reloadui 等 |

### 7.2 关系图后端

| 绝对路径 | 行范围 | 内容摘要 |
|---------|--------|---------|
| [api/graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/api/graph.go) | L53-L104 | HTTP `/api/graph/getGraph`：临时覆盖配置 + BuildGraph + 发布过滤 |
| [api/graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/api/graph.go) | L106-L163 | HTTP `/api/graph/getLocalGraph`：BuildTreeGraph + RandomSleep(200-500ms) |
| [model/graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/model/graph.go) | L35-L61 | GraphNode / GraphLink 结构体定义（含 Refs/Defs 统计字段） |
| [model/graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/model/graph.go) | L62-L165 | `BuildTreeGraph`：局部关系图 9 步构建流程 |
| [model/graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/model/graph.go) | L167-L218 | `BuildGraph`：全局关系图构建流程（含文档块关联） |
| [model/graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/model/graph.go) | L296-L367 | `growLinkedNodes`：16 层正反向 BFS 扩展算法 |
| [model/graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/model/graph.go) | L378-L456 | `genTreeNodes` / `buildLinks` / `markLinkedNodes`：节点大小 log2(Defs) 计算 |
| [model/graph.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/model/graph.go) | L471-L511 | `pruneUnref`：MinRefs 剪枝 + MaxBlocks 上限 + 悬挂边清理 |
| [model/backlink.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/model/backlink.go) | L934-L999 | `buildFullLinks` / `buildDefsAndRefs`：共享的 def↔ref 双向聚合 |
| [sql/block_ref_query.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/block_ref_query.go) | L162-L202 | `QueryDefRootBlocksByRefRootID` / `QueryRefRootBlocksByDefRootIDs`：文档级引用 SQL |
| [sql/block_ref_query.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/block_ref_query.go) | L453-L502 | `DefRefs`：两轮 SQL 扫描 + `block_id@def_block_id` 键拼接 |

### 7.3 缓存失效与清理

| 绝对路径 | 行范围 | 内容摘要 |
|---------|--------|---------|
| [sql/cache.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/cache.go) | L32-L50 | `cacheDisabled` 开关 + ristretto blockCache 配置 + `ClearCache`（只清 L3） |
| [sql/cache.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/cache.go) | L52-L82 | blockCache 的 put/get/remove（remove 级联触发 L1 清理） |
| [sql/cache.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/cache.go) | L84-L119 | `defIDRefsCache` (30min TTL)：GetRefsCacheByDefID / CacheRef / putRefCache / removeRefCacheByDefID |
| [sql/database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go) | L968-L1018 | `deleteBlocksByIDs`：按 ID 逐块 `removeBlockCache` |
| [sql/database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go) | L1020-L1037 | `deleteBlocksByBoxTx` → `ClearCache` |
| [sql/database.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/database.go) | L1140-L1318 | `deleteByRootID` / `batchDeleteByRootIDs` / `batchDeleteByPathPrefix` / `batchUpdatePath` / `batchUpdateHPath` → 均调用 `ClearCache` |
| [sql/block.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/block.go) | L61-L80 | `updateRootContent` → `removeBlockCache` + `RemoveBlockIAL` |
| [sql/block.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/block.go) | L82-L104 | `updateBlockContent` → `putBlockCache` 覆写 |
| [sql/queue.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/queue.go) | L102-L180 | `FlushQueue`：op>512 时 `disableCache()` + 完后 `EvtSQLIndexFlushed` + `databaseIndexCommit` 广播 |
| [sql/queue.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/sql/queue.go) | L182-L223 | `execOp`：14 种队列 action 分发 |
| [treenode/node.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/treenode/node.go) | L453-L478 | `DynamicRefTexts` sync.Map 定义 + `SetDynamicBlockRefText`（只有 Store，无 Delete） |
| [util/websocket.go](file:///d:/fz/0601/solo-dogfeeding/code/284-siyuan/kernel/util/websocket.go) | L336-L360 | `PushSaveDoc` / `PushReloadProtyle` / `PushSetDefRefCount` / `PushSetRefDynamicText`：各推送命令定义 |
