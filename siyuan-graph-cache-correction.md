# SiYuan 缓存清理与关系图边界：事实核对与结论修正

> 本文对 `siyuan-backlink-index.md`、`siyuan-backlink-index-followup.md`、`siyuan-graph-cache-boundary.md` 三份文档的关键结论进行代码级核对。所有结论明确区分为「已确认事实」（有直接代码证据）与「风险推断」（无直接代码证据，基于代码分析的合理推测）。**代码引用统一使用仓库相对路径**。

---

## 目录

- [1. 方法论：事实 vs 推断的区分标准](#1-方法论事实-vs-推断的区分标准)
- [2. ClearCache() 实际清理行为核对](#2-clearcache-实际清理行为核对)
- [3. 引用缓存清理调用链完整追踪](#3-引用缓存清理调用链完整追踪)
- [4. 关系图展示与缓存的真实边界](#4-关系图展示与缓存的真实边界)
- [5. 前后三份文档结论一致性核对](#5-前后三份文档结论一致性核对)
- [6. 修正后的清理缺口总结](#6-修正后的清理缺口总结)
- [7. 附：相对路径代码索引](#7-附相对路径代码索引)

---

## 1. 方法论：事实 vs 推断的区分标准

| 类别 | 判定标准 | 置信度 |
|------|---------|--------|
| **✓ 已确认事实** | 有直接代码证据（函数调用、变量定义、返回值类型、全仓搜索结果为 0） | 100% |
| **⚠️ 风险推断** | 代码路径存在但未直接验证；或基于架构分析的合理推测；或需运行时验证 | 60%-90% |
| **❌ 需修正结论** | 之前文档中的表述不准确、需澄清或补充 | - |

核对原则：全仓搜索函数调用时，如果结果为 0，即视为"未调用"的事实证据。如果结果 > 0，则逐个检查调用点上下文。

---

## 2. ClearCache() 实际清理行为核对

### 2.1 ✓ 事实 1：ClearCache() 只清理 ristretto 块缓存

**代码证据**：`kernel/sql/cache.go:48-50`
```go
func ClearCache() {
    blockCache.Clear()
    // 注意：此处没有任何 defIDRefsCache 相关操作
}
```

**全仓搜索确认**：`ClearCache` 共 6 个调用点，均来自 `kernel/sql/database.go`，全部只触发 blockCache.Clear()：
- `kernel/sql/database.go:79` — InitDatabase
- `kernel/sql/database.go:1036` — deleteBlocksByBoxTx
- `kernel/sql/database.go:1156` — deleteByRootID
- `kernel/sql/database.go:1203` — batchDeleteByRootIDs
- `kernel/sql/database.go:1244` — batchDeleteByPathPrefix
- `kernel/sql/database.go:1290` — batchUpdatePath
- `kernel/sql/database.go:1314` — batchUpdateHPath

### 2.2 ✓ 事实 2：go-cache 有 Flush() 方法但未被调用

**代码证据**：
- `kernel/sql/cache.go:84` — `var defIDRefsCache = gcache.New(30*time.Minute, 5*time.Minute)`
- `kernel/go.mod:58` — `github.com/patrickmn/go-cache v2.1.0+incompatible`

patrickmn/go-cache v2.1.0 确实提供 `Flush()` 方法用于清空所有缓存条目，但 SiYuan 代码中**无任何调用**。

### 2.3 ✓ 事实 3：GetRefsCacheByDefID 使用 Items() 遍历，可能返回已过期条目

**代码证据**：`kernel/sql/cache.go:86-93`
```go
func GetRefsCacheByDefID(defID string) (ret []*Ref) {
    for defBlockID, refs := range defIDRefsCache.Items() {  // ← 使用 Items()
        if defBlockID == defID {
            for _, ref := range refs.Object.(map[string]*Ref) {
                ret = append(ret, ref)
            }
        }
    }
    ...
}
```

go-cache 的 `Items()` 方法返回**所有条目，包括已逻辑过期但 janitor 尚未物理删除**的条目（janitor 每 5 分钟运行一次）。而 `Get(defID)` 方法会检查过期时间。此处用 `Items()` 而非 `Get(defID)` 意味着：在最坏情况下，已过期的引用缓存可能在 5 分钟内仍被返回。

**❌ 需修正**：之前文档提到"TTL 过期机制"时，未指出 `Items()` 方法会绕过 TTL 检查，需补充此细节。

---

## 3. 引用缓存清理调用链完整追踪

### 3.1 ✓ 事实 4：removeBlockCache(id) 只有 2 个调用点

**全仓搜索 `removeBlockCache(` 结果**：
1. `kernel/sql/database.go:975` — `deleteBlocksByIDs(tx *sql.Tx, ids []string)`
   ```go
   for _, id := range ids {
       removeBlockCache(id)
   }
   ```
2. `kernel/sql/block.go:77` — `updateRootContent(tx *sql.Tx, ...)`
   ```go
   removeBlockCache(id)
   cache.RemoveBlockIAL(id)
   ```

**❌ 需修正**：之前文档暗示 removeBlockCache 有更多调用点，实际只有上述 2 处。

### 3.2 ✓ 事实 5：removeBlockCache 级联清理 L1，但仅限"块作为 def"的方向

**代码证据**：`kernel/sql/cache.go:79-82`
```go
func removeBlockCache(id string) {
    blockCache.Del(id)
    removeRefCacheByDefID(id)   // 级联：删除块时，清理它作为被引用方(def)的 L1 缓存
}
```

**代码证据**：`kernel/sql/cache.go:117-119`
```go
func removeRefCacheByDefID(defID string) {
    defIDRefsCache.Delete(defID)
}
```

**全仓搜索 `removeRefCacheByDefID(` 结果**：只有 1 个调用点（从 removeBlockCache 级联）。

### 3.3 ✓ 事实 6：DynamicRefTexts 完全没有 Delete 调用

**全仓搜索 `DynamicRefTexts\.Delete` 结果**：**0 调用**。

**代码证据**：`kernel/treenode/node.go:453-478`
```go
var DynamicRefTexts = sync.Map{}

func SetDynamicBlockRefText(blockRef *ast.Node, refText string) {
    ...
    DynamicRefTexts.Store(blockRef.TextMarkBlockRefID, refText)
    // 注意：此处只有 Store，没有任何地方调用 Delete
}
```

### 3.4 ✓ 事实 7：L1 引用缓存（defIDRefsCache）的读路径只有 1 个入口

**全仓搜索 `GetRefsCacheByDefID(` 结果**：4 个调用点，全部在 `getRefsCacheByDefNode` 内。

**代码证据**：`kernel/model/transaction.go:1951-2001`
```go
func getRefsCacheByDefNode(updateNode *ast.Node) (ret []*sql.Ref, changedNodes []*ast.Node) {
    ret = sql.GetRefsCacheByDefID(updateNode.ID)                          // 精确匹配
    if updateNode.Parent.IsContainerBlock() && ... {
        for parent := updateNode.Parent; nil != parent; parent = parent.Parent {
            parentRefs := sql.GetRefsCacheByDefID(parent.ID)              // 向上查父容器
            ret = append(ret, parentRefs...)
        }
    }
    if updateNode.IsContainerBlock() {
        ast.Walk(updateNode, func(n *ast.Node, entering bool) ast.WalkStatus {
            childRefs := sql.GetRefsCacheByDefID(n.ID)                    // 向下查子块
            ret = append(ret, childRefs...)
        })
    }
    if ast.NodeHeading == updateNode.Type && "1" == updateNode.IALAttr("fold") {
        for _, child := range treenode.HeadingChildren(updateNode) {
            childRefs := sql.GetRefsCacheByDefID(child.ID)                // 查折叠标题子块
            ret = append(ret, childRefs...)
        }
    }
    return
}
```

**❌ 需修正**：之前文档暗示 L1 缓存被多处使用，实际上**仅用于动态锚文本级联刷新这一条路径**。关系图、反链面板等均不经过此缓存。

### 3.5 ✓ 事实 8：IAL 缓存有独立的 ClearBlocksIAL()，但 ClearCache() 不调用

**代码证据**：
- `kernel/cache/ial.go:77-79` — `ClearBlocksIAL()` 方法存在：
  ```go
  func ClearBlocksIAL() {
      blockIALCache.Clear()
  }
  ```
- 但 `kernel/sql/cache.go:48-50` 的 `ClearCache()` 中没有调用它。

**全仓搜索 `RemoveBlockIAL(` 结果**：3 个调用点：
1. `kernel/sql/block.go:78` — `updateRootContent`
2. `kernel/model/sync.go:310` — 同步更新
3. `kernel/model/sync.go:357` — 同步更新

---

## 4. 关系图展示与缓存的真实边界

### 4.1 ✓ 事实 9：关系图的所有引用查询均为纯 SQL，完全不经过 L1 缓存

**代码证据**：`kernel/model/graph.go` 中引用查询调用的函数：
1. `buildFullLinks(stmt)` → `buildDefsAndRefs(stmt)` → `sql.DefRefs(condition, Conf.Graph.MaxBlocks)`
   - `kernel/sql/block_ref_query.go:453-502` — 两轮 SQL 扫描，JOIN blocks + refs，无缓存
2. `sql.QueryDefRootBlocksByRefRootID(rootID)` — `kernel/sql/block_ref_query.go:162-175`
   - 纯 SQL：`SELECT * FROM blocks WHERE id IN (SELECT DISTINCT def_block_root_id FROM refs WHERE root_id = ?)`
3. `sql.QueryRefRootBlocksByDefRootIDs(rootIDs)` — `kernel/sql/block_ref_query.go:177-202`
   - 纯 SQL：JOIN refs + blocks 查引用方文档

**全仓搜索 `kernel/model/graph.go` 中 `GetRefsCacheByDefID` 调用**：**0 次**。

### 4.2 ✓ 事实 10：关系图的 WS 消息订阅缺口确实存在

**代码证据**：`app/src/layout/dock/Graph.ts:54-86` 的 msgCallback 仅处理 4 种 cmd：
- `mount`
- `rename`
- `closeBox` / `removeBox`
- `removeDoc`

**缺失处理的相关 WS 命令**（均由 `kernel/util/websocket.go` 定义并广播）：
- `setDefRefCount` — `kernel/util/websocket.go:358-360`
- `setRefDynamicText` — `kernel/util/websocket.go:354-356`
- `savedoc` — `kernel/util/websocket.go:336-344`
- `databaseIndexCommit` — `kernel/sql/queue.go:177`

这些命令在 `app/src/index.ts:89-116` 中被主 WS 连接处理，但仅更新编辑器 DOM，**不通知 Graph 面板**。

### 4.3 ✓ 事实 11：关系图增量分批加载算法参数与逻辑完全可复现

**代码证据**：`app/src/layout/dock/Graph.ts:666-716` 中：
- 初始批次：`i = max(ceil(nodes * 0.1), 128)`
- 节点间隔：`intervalNodeTime = max(ceil(256 / 8), 32)` = 32ms
- 边间隔：固定 256ms
- 每批大小：`batch = clamp(nodes / 256 / 2, 64, 256)`
- 初始缩放：`initialScale = max(0.03, 1 - 0.3 * floor(nodes / 128))`
- Physics：`solver = forceAtlas2Based`，`maxVelocity = clamp(nodes, 256, 1024)`

以上均为可量化的事实参数，无推断成分。

### 4.4 ⚠️ 风险推断 1：关系图刷新延迟的用户感知

由于关系图不订阅引用变更推送，且后端每次查询直查 DB，当 FlushQueue 未完成时（最长约 2 秒），用户点击刷新仍可能读到旧数据。

- **触发条件**：编辑包含引用的块后 2 秒内点击关系图刷新
- **验证方法**：在 `sql.FlushQueue` 前后打点，对比关系图返回数据
- **置信度**：90%（架构上必然存在此窗口）

### 4.5 ⚠️ 风险推断 2：关系图查询性能随库线性退化

`sql.DefRefs` 是 `O(|refs|)` 的两轮全表扫描（每次均 JOIN refs + blocks），无渐进式缓存。在 10w+ 引用的超大库中，响应时间可能超过 500ms。

- **触发条件**：引用数量 > 50,000
- **验证方法**：压测不同引用量级下的 `BuildGraph` 耗时
- **置信度**：80%（基于 SQL 复杂度推断）

---

## 5. 前后三份文档结论一致性核对

### 5.1 结论对照表

| 结论点 | `siyuan-backlink-index.md` | `siyuan-backlink-index-followup.md` | `siyuan-graph-cache-boundary.md` | 本核对文档（最终） |
|--------|----------------------------|-------------------------------------|-----------------------------------|-------------------|
| ClearCache 不清 L1 | ⚠️ 提及但未强调 | ✓ 明确指出缺口 A | ✓ 明确指出缺口 A | ✓ 已确认事实 + **补充：Items() 绕过 TTL 检查** |
| DynamicRefTexts 无 Delete | ⚠️ 隐含 | ⚠️ 隐含 | ✓ 明确指出缺口 B | ✓ 已确认事实（全仓搜索 0 调用） |
| removeBlockCache 单向清理 | ❌ 未提及 | ⚠️ 隐含 | ✓ 明确指出缺口 C | ✓ 已确认事实（只有 2 个调用点） + **补充：调用点数量纠正** |
| IAL 未批量清理 | ❌ 未提及 | ❌ 未提及 | ✓ 明确指出缺口 D | ✓ 已确认事实 |
| 关系图不经过 L1 缓存 | ✓ 隐含 | ✓ 明确 | ✓ 明确 | ✓ 已确认事实（全仓搜索 0 调用） |
| 关系图刷新缺口 | ❌ 未提及 | ❌ 未提及 | ✓ 明确 | ✓ 已确认事实 |
| GetRefsCacheByDefID 用 Items() | ❌ 未提及 | ❌ 未提及 | ❌ 未提及 | ✓ 新增发现 |
| L1 缓存仅用于动态锚文本 | ❌ 暗示多处使用 | ❌ 暗示多处使用 | ❌ 暗示多处使用 | ✓ 新增发现（仅 4 个调用点） |

### 5.2 一致性总体评价

**三份前文档的核心结论均正确**，但存在以下可改进点：

1. **事实/推断区分不足**：部分表述（如"用户几乎感知不到"）是推断，应与事实区分
2. **调用点数量不准确**：removeBlockCache 只有 2 个调用点，之前暗示更多
3. **遗漏关键细节**：GetRefsCacheByDefID 使用 Items() 而非 Get() 的潜在问题
4. **L1 缓存使用范围夸大**：之前暗示多处使用，实际上仅动态锚文本级联刷新用

---

## 6. 修正后的清理缺口总结

### 6.1 已确认事实级别的缺口（✓ 有代码证据）

| # | 缺口描述 | 位置 | 影响评估 |
|---|---------|------|---------|
| **A** | `ClearCache()` 只清 ristretto，不清 go-cache defIDRefsCache | `kernel/sql/cache.go:48-50` | 文档删除后最长 30min + 5min 内，L1 可能返回已删除文档的引用 |
| **A+** | `GetRefsCacheByDefID` 用 `Items()` 遍历，绕过 TTL 检查 | `kernel/sql/cache.go:86-93` | 逻辑过期的条目在 janitor 清理前（最长 5min）仍可能被返回 |
| **B** | `DynamicRefTexts` sync.Map 无 Delete 调用 | `kernel/treenode/node.go:453-478` | 纯内存泄漏，无功能影响 |
| **C** | `removeBlockCache(id)` 只有 2 个调用点，且仅清理"块作为 def"方向 | `kernel/sql/cache.go:79-82` | 删除块时，引用了被删块的其他文档的 L1 缓存残留 |
| **D** | `ClearCache()` 不调用 IAL 缓存的 `ClearBlocksIAL()` | `kernel/cache/ial.go:77-79` vs `kernel/sql/cache.go:48-50` | IAL 缓存残留，内存泄漏 |

### 6.2 风险推断级别的缺口（⚠️ 合理推测）

| # | 缺口描述 | 触发条件 | 置信度 |
|---|---------|---------|--------|
| **E** | 关系图刷新延迟窗口 | 编辑后 2 秒内手动刷新 | 90% |
| **F** | 关系图查询性能线性退化 | refs 表 > 50,000 行 | 80% |
| **G** | 级联刷新漏更新 | 文档删除后 35 分钟内 `refreshDynamicRefTexts0` 被调用 | 70% |

### 6.3 为什么这些缺口在实际使用中影响有限

**已确认的自修复机制**（✓ 事实）：
1. `GetRefsCacheByDefID` 的 DB 回退：L1 为空时直查 DB 并回填（`kernel/sql/cache.go:94-99`）
2. 关系图直查 DB：完全绕过 L1 缓存（第 4.1 节）
3. 30min TTL + 5min janitor：即使不清，最长 35 分钟也会全部清理
4. 7 次级联 + 用户下一次编辑：锚文本漏刷新的窗口很窄

---

## 7. 附：相对路径代码索引

### 7.1 缓存清理相关

| 相对路径 | 行范围 | 内容 |
|---------|--------|------|
| `kernel/sql/cache.go` | 48-50 | `ClearCache()` — 只清 ristretto |
| `kernel/sql/cache.go` | 79-82 | `removeBlockCache()` — 级联清 L1 |
| `kernel/sql/cache.go` | 84 | `defIDRefsCache` 定义（30min TTL，5min janitor） |
| `kernel/sql/cache.go` | 86-101 | `GetRefsCacheByDefID()` — 使用 `Items()` 遍历 |
| `kernel/sql/cache.go` | 108-115 | `putRefCache()` |
| `kernel/sql/cache.go` | 117-119 | `removeRefCacheByDefID()` |
| `kernel/sql/database.go` | 968-1018 | `deleteBlocksByIDs()` — 调用 removeBlockCache |
| `kernel/sql/database.go` | 1020-1037 | `deleteBlocksByBoxTx()` — 调用 ClearCache |
| `kernel/sql/database.go` | 1140-1158 | `deleteByRootID()` — 调用 ClearCache |
| `kernel/sql/database.go` | 1161-1206 | `batchDeleteByRootIDs()` — 调用 ClearCache |
| `kernel/sql/database.go` | 1208-1246 | `batchDeleteByPathPrefix()` — 调用 ClearCache |
| `kernel/sql/database.go` | 1248-1294 | `batchUpdatePath()` — 调用 ClearCache |
| `kernel/sql/database.go` | 1296-1318 | `batchUpdateHPath()` — 调用 ClearCache |
| `kernel/sql/block.go` | 61-80 | `updateRootContent()` — 调用 removeBlockCache + RemoveBlockIAL |
| `kernel/treenode/node.go` | 453-478 | `DynamicRefTexts` 定义 + SetDynamicBlockRefText（只有 Store） |
| `kernel/cache/ial.go` | 73-75 | `RemoveBlockIAL()` |
| `kernel/cache/ial.go` | 77-79 | `ClearBlocksIAL()` — 存在但未被 ClearCache 调用 |
| `kernel/model/transaction.go` | 1951-2001 | `getRefsCacheByDefNode()` — L1 缓存唯一读路径（4 处调用） |

### 7.2 关系图相关

| 相对路径 | 行范围 | 内容 |
|---------|--------|------|
| `app/src/layout/dock/Graph.ts` | 19-92 | Graph 类结构与 WS 连接配置 |
| `app/src/layout/dock/Graph.ts` | 54-86 | msgCallback — 仅处理 4 种 cmd |
| `app/src/layout/dock/Graph.ts` | 419-499 | `searchGraph()` — 请求构建与防重入 |
| `app/src/layout/dock/Graph.ts` | 518-781 | `onGraph()` — 渲染与增量加载 |
| `app/src/layout/dock/Graph.ts` | 666-716 | 增量分批加载算法（初始 10% + 双定时器追加） |
| `app/src/index.ts` | 89-116 | 主 WS 消息分发（不通知 Graph） |
| `kernel/api/graph.go` | 53-104 | `/api/graph/getGraph` — 后端入口 |
| `kernel/api/graph.go` | 106-163 | `/api/graph/getLocalGraph` — 后端入口 |
| `kernel/model/graph.go` | 62-165 | `BuildTreeGraph()` — 局部图构建（无 L1 缓存调用） |
| `kernel/model/graph.go` | 167-218 | `BuildGraph()` — 全局图构建（无 L1 缓存调用） |
| `kernel/model/graph.go` | 296-367 | `growLinkedNodes()` — 16 层 BFS |
| `kernel/model/backlink.go` | 934-999 | `buildFullLinks()` / `buildDefsAndRefs()` — 调用 sql.DefRefs |
| `kernel/sql/block_ref_query.go` | 162-175 | `QueryDefRootBlocksByRefRootID()` — 纯 SQL |
| `kernel/sql/block_ref_query.go` | 177-202 | `QueryRefRootBlocksByDefRootIDs()` — 纯 SQL |
| `kernel/sql/block_ref_query.go` | 453-502 | `DefRefs()` — 两轮 SQL 扫描，无缓存 |
| `kernel/util/websocket.go` | 336-360 | `PushSaveDoc` / `PushSetDefRefCount` / `PushSetRefDynamicText` — 未被 Graph 订阅 |
| `kernel/sql/queue.go` | 177 | `BroadcastByType("main", "databaseIndexCommit")` — 未被 Graph 订阅 |
| `kernel/go.mod` | 58 | go-cache v2.1.0 依赖确认 |
