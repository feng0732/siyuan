# SiYuan 双向链接深度分析 Follow-Up

> 本文为 `siyuan-backlink-index.md` 的补充，聚焦三个具体流程的代码级追踪及与索引维护、缓存一致性的关联核对。代码引用统一使用仓库相对路径格式。

---

## 目录

- [1. 编辑事务中的引用缓存写入](#1-编辑事务中的引用缓存写入)
- [2. 动态锚文本同步与级联刷新](#2-动态锚文本同步与级联刷新)
- [3. 关系图使用引用查询结果的展示链路](#3-关系图使用引用查询结果的展示链路)
- [4. 与索引维护、缓存一致性的关联核对](#4-与索引维护缓存一致性的关联核对)
- [5. 附：关键相对路径索引](#5-附关键相对路径索引)

---

## 1. 编辑事务中的引用缓存写入

### 1.1 事务执行总览

SiYuan 使用串行化事务队列（`txQueue` channel 容量 7）保证编辑操作的原子性和顺序性。核心入口与执行流：

| 阶段 | 位置 | 关键操作 |
|------|------|---------|
| 入队 | `kernel/model/transaction.go` L127-L132 | `PerformTransactions` 将 tx 推入 `txQueue` |
| 消费 | `kernel/model/transaction.go` L74-L81 | `flushQueue` 协程持续消费，串行调用 `flushTx` |
| 加锁 | `kernel/model/transaction.go` L83-L125 | `flushTx` 用 `flushLock` 保证全局串行 |
| 执行 | `kernel/model/transaction.go` L148 | `performTx` 调度具体 op：`doCreate/doUpdate/doInsert/doDelete/doMove` 等 |

### 1.2 doUpdate 中的引用缓存写入

`doUpdate` 是引用缓存写入最集中的路径，发生在 AST 后序遍历扫描每个 `TextMark` 节点时。

**调用路径：**
```
doUpdate (L1410)
  └─ ast.Walk(subTree.Root, ...)  (L1437)
       └─ if n.IsTextMarkType("block-ref")
            ├─ sql.CacheRef(subTree, n)                       ← 写入 go-cache 引用缓存
            └─ if "d" == n.TextMarkBlockRefSubtype
                 └─ treenode.DynamicRefTexts.Load(...)        ← 从 sync.Map 读取动态锚文本覆盖
```

**代码片段（`kernel/model/transaction.go` L1448-L1459）：**
```go
} else if n.IsTextMarkType("block-ref") {
    sql.CacheRef(subTree, n)
    if "d" == n.TextMarkBlockRefSubtype {
        if dRefText, ok := treenode.DynamicRefTexts.Load(n.TextMarkBlockRefID); ok && "" != dRefText {
            n.TextMarkTextContent = dRefText.(string)
        }
    }
    newDefIDs = append(newDefIDs, n.TextMarkBlockRefID)
}
```

#### 1.2.1 sql.CacheRef 的实现

`kernel/sql/cache.go` L103-L115：

```go
func CacheRef(tree *parse.Tree, refNode *ast.Node) {
    ref := buildRef(tree, refNode)   // 根据 AST 节点和 tree 上下文构造 sql.Ref 结构体
    putRefCache(ref)
}

func putRefCache(ref *Ref) {
    defBlockRefs, ok := defIDRefsCache.Get(ref.DefBlockID)
    if !ok {
        defBlockRefs = map[string]*Ref{}
    }
    defBlockRefs.(map[string]*Ref)[ref.BlockID] = ref
    defIDRefsCache.SetDefault(ref.DefBlockID, defBlockRefs)
}
```

缓存结构：
- **`defIDRefsCache`**：`go-cache` TTL 缓存（30 分钟 TTL，5 分钟清理周期），Key 为 `defBlockID`，Value 为 `map[refBlockID]*Ref`
- **二级 Map**：每个 def 对应一个 ref 的 Map，天然实现同 refBlockID 的幂等覆盖

#### 1.2.2 新旧引用 ID 对比与计数刷新

`kernel/model/transaction.go` L1432-L1478：

```go
oldDefIDs := getRefDefIDs(oldNode)       // 扫描旧节点收集引用 ID
newDefIDs := ...                         // 遍历新 subtree 过程中收集
refDefIDs = gulu.Str.RemoveDuplicatedElem(refDefIDs)

if !slices.Equal(oldDefIDs, newDefIDs) {
    refDefIDs = append(refDefIDs, newDefIDs...)
    for _, defID := range refDefIDs {
        // 异步延迟刷新引用计数，uniqueActions 保证任务唯一
        task.AppendAsyncTaskWithDelay(task.SetDefRefCount, util.SQLFlushInterval, refreshRefCount, defID)
    }
}
```

`getRefDefIDs` 的实现（`kernel/model/transaction.go` L1619-L1635）会递归扫描块下所有 `block-ref` 和 `embed-block-ref`：

```go
func getRefDefIDs(node *ast.Node) (refDefIDs []string) {
    ast.Walk(node, func(n *ast.Node, entering bool) ast.WalkStatus {
        if !entering { return ast.WalkContinue }
        if treenode.IsBlockRef(n) {
            refDefIDs = append(refDefIDs, n.TextMarkBlockRefID)
        } else if treenode.IsEmbedBlockRef(n) {
            refDefIDs = append(refDefIDs, treenode.GetEmbedBlockRef(n))
        }
        return ast.WalkContinue
    })
    refDefIDs = gulu.Str.RemoveDuplicatedElem(refDefIDs)
    return
}
```

### 1.3 doInsert 与 doDelete 中的引用处理

| 操作 | 位置 | 引用缓存动作 |
|------|------|-------------|
| **doInsert** | `kernel/model/transaction.go` L1320-L1324 | 调用 `getRefDefIDs` 收集引用 ID，触发异步 `SetDefRefCount` |
| **doDelete / doDelete0** | `kernel/model/transaction.go` L968-L972 | 同样调用 `getRefDefIDs` + 异步 `SetDefRefCount` |
| **doCreate / doMove / doAppend** | - | 通过写树 → commit → refreshDynamicRefTexts 间接触发 |

> **注意**：`doInsert` 和 `doDelete` **不直接调用 `sql.CacheRef`**，它们仅负责收集引用变更并触发计数刷新。引用缓存的写入依赖后续 `tx.commit()` 中的 `writeTreeUpsertQueue` → `sql.UpsertTreeQueue` → `upsertRefs` 以及 `refreshDynamicRefTexts` 中 `sql.UpdateRefsTreeQueue`。

---

## 2. 动态锚文本同步与级联刷新

### 2.1 触发时机

动态锚文本（Dynamic Anchor Text）是当被引用块（def）内容变化时，引用处（ref）自动同步显示新文本的机制。触发入口有三：

| 入口 | 位置 | 触发条件 |
|------|------|---------|
| **事务提交** | `kernel/model/transaction.go` L1877 | `tx.commit()` 在所有 op 完成、树入队后执行 |
| **单节点刷新** | `kernel/model/push_reload.go` L251-L257 | `refreshDynamicRefText` 外部 API 调用 |
| **文档重命名** | `kernel/model/transaction.go` L2006-L2025 | `updateRefTextRenameDoc` → `FlushUpdateRefTextRenameDocJob` |

### 2.2 事务 commit 中的调用链

```
tx.commit()                                       (transaction.go L1865)
  ├─ writeTreeUpsertQueue(tree)                   ← 树入 SQL 异步写队列（先写库）
  └─ refreshDynamicRefTexts(tx.nodes, tx.trees)   ← 级联刷新动态锚文本
       └─ (最多 7 次迭代) refreshDynamicRefTexts0
            ├─ 收集所有被引用块的 refs （从缓存）
            ├─ 遍历每个引用树，updateRefText 更新 AST
            ├─ sql.UpdateRefsTreeQueue(refTree)   ← refs 表入队更新
            ├─ updateAttributeViewBlockText(...)  ← AV 主键内容同步
            └─ indexWriteTreeUpsertQueue(tree)    ← 变更后的引用树写入
```

### 2.3 refreshDynamicRefTexts 的 7 次级联

`kernel/model/push_reload.go` L261-L280：

```go
func refreshDynamicRefTexts(updatedDefNodes map[string]*ast.Node, updatedTrees map[string]*parse.Tree) (changedRootIDs []string) {
    for t := range updatedTrees {
        changedRootIDs = append(changedRootIDs, t)
    }
    for i := 0; i < 7; i++ {
        updatedRefNodes, updatedRefTrees := refreshDynamicRefTexts0(updatedDefNodes, updatedTrees)
        if 1 > len(updatedRefNodes) {
            break   // 无变化提前退出
        }
        updatedDefNodes, updatedTrees = updatedRefNodes, updatedRefTrees  // 以新 refs 作为下一轮 defs
        for t := range updatedTrees {
            changedRootIDs = append(changedRootIDs, t)
        }
    }
    changedRootIDs = gulu.Str.RemoveDuplicatedElem(changedRootIDs)
    return
}
```

**级联原理**：
- 第 0 轮：A（def）变更 → B、C 引用 A，B、C 的锚文本更新
- 第 1 轮：如果 B 的内容本身也被 D、E 引用 → D、E 的锚文本也需要更新
- ... 最多迭代 7 次，或中途无变化时 break

### 2.4 refreshDynamicRefTexts0 的核心逻辑

`kernel/model/push_reload.go` L282-L358，分三步：

**Step 1：从缓存收集引用并更新每棵引用树的 AST**

```go
// 构建 treeRefNodeIDs: map[refTreeID]*hashset.Set (该文档中哪些块引用了变更的 def)
for _, updateNode := range updatedDefNodes {
    refs, changedNodes = getRefsCacheByDefNode(updateNode)   // 关键：从 go-cache 取引用
    for _, ref := range refs {
        treeRefNodeIDs[ref.RootID].Add(ref.BlockID)
    }
}

for refTreeID, refNodeIDs := range treeRefNodeIDs {
    refTree, _ := LoadTreeByBlockID(refTreeID)
    ast.Walk(refTree.Root, func(n *ast.Node, entering bool) ast.WalkStatus {
        if n.IsBlock() && refNodeIDs.Contains(n.ID) {
            changed, changedDefNodes := updateRefText(n, updatedDefNodes)  // 更新该块内所有引用锚文本
            if changed {
                sql.UpdateRefsTreeQueue(refTree)    // refs 入异步写队列
            }
        }
    })
}
```

**Step 2：同步更新属性视图主键**

```go
updateAttributeViewBlockText(updatedDefNodes)
```

**Step 3：将所有变更的引用树再次入写队列**

```go
for _, tree := range changedRefTree {
    indexWriteTreeUpsertQueue(tree)
}
```

### 2.5 getRefsCacheByDefNode 的容器块展开

`kernel/model/transaction.go` L1951-L2001 实现了"容器块引用"语义：如果用户直接引用了 ListItem 或 Heading（容器块），其内部任何叶子块内容变化都应触发该引用的锚文本刷新。反之，若引用了叶子块，但叶子块所在容器也被引用了，也要触发容器引用的刷新。

```go
func getRefsCacheByDefNode(updateNode *ast.Node) (ret []*sql.Ref, changedNodes []*ast.Node) {
    // 1) 精确匹配：当前块自身作为 def 的引用
    ret = sql.GetRefsCacheByDefID(updateNode.ID)

    // 2) 向上查找：如果是容器块下第一个叶子块，向上找父容器引用
    if updateNode.Parent.IsContainerBlock() && updateNode == treenode.FirstLeafBlock(updateNode.Parent) {
        for parent := updateNode.Parent; nil != parent; parent = parent.Parent {
            parentRefs := sql.GetRefsCacheByDefID(parent.ID)
            ret = append(ret, parentRefs...)
        }
    }

    // 3) 向下查找：如果当前节点是容器块，递归查找所有子块引用
    if updateNode.IsContainerBlock() {
        ast.Walk(updateNode, func(n *ast.Node, entering bool) ast.WalkStatus {
            childRefs := sql.GetRefsCacheByDefID(n.ID)
            ret = append(ret, childRefs...)
        })
    }

    // 4) 折叠标题：向下查找折叠的子块引用
    if ast.NodeHeading == updateNode.Type && "1" == updateNode.IALAttr("fold") {
        for _, child := range treenode.HeadingChildren(updateNode) {
            childRefs := sql.GetRefsCacheByDefID(child.ID)
            ret = append(ret, childRefs...)
        }
    }
    return
}
```

### 2.6 updateRefText 与 SetDynamicBlockRefText

**`updateRefText`** (`kernel/model/transaction.go` L2033-L2068) 在引用块的 AST 内遍历所有 TextMark，匹配到变更的 defID 时执行：

```go
if "d" == subtype {  // 仅动态锚文本(d)需要更新文本内容
    refText = strings.TrimSpace(getNodeRefText(defNode))   // 从新 def 节点渲染锚文本
    if "" == refText { refText = n.TextMarkBlockRefID }
    treenode.SetDynamicBlockRefText(n, refText)
}
```

**`SetDynamicBlockRefText`** (`kernel/treenode/node.go` L455-L478) 同步更新两处存储：

```go
func SetDynamicBlockRefText(blockRef *ast.Node, refText string) {
    // 1) 修改 AST 结构：插入 NodeBlockRefDynamicText 节点（持久化到磁盘）
    if ast.NodeBlockRef == blockRef.Type {
        refID.InsertAfter(&ast.Node{Type: ast.NodeBlockRefDynamicText, Tokens: []byte(refText)})
    }
    // 2) 更新内存 sync.Map（编辑事务中 doUpdate 会读取它）
    DynamicRefTexts.Store(blockRef.TextMarkBlockRefID, refText)
}
```

**`getNodeRefText`** (`kernel/model/blockinfo.go` L278-L345) 的渲染优先级：
1. 块的 `name` 属性（自定义命名）
2. 调用 `getNodeRefText0` 根据块类型渲染：
   - 特殊块：QueryEmbed/IFrame/TB/Video/Audio/AV → 固定前缀文本
   - 容器块 → 取 `FirstLeafBlock` 的内容
   - 普通块 → `renderBlockText` 渲染后截断至 `BlockRefDynamicAnchorTextMaxLen`

---

## 3. 关系图使用引用查询结果的展示链路

### 3.1 HTTP API 入口

`kernel/api/graph.go` 暴露两个接口：

| 接口 | 位置 | 调用模型函数 |
|------|------|-------------|
| `/api/graph/getGraph` | L53-L104 | `model.BuildGraph(query)` — 全局关系图 |
| `/api/graph/getLocalGraph` | L106-L163 | `model.BuildTreeGraph(id, keyword)` — 当前文档局部关系图 |

两个接口均支持传入 `conf` 参数覆盖临时图配置（类型过滤、日记过滤、最小引用数、箭头显示等）。

### 3.2 BuildTreeGraph — 局部关系图构建流程

`kernel/model/graph.go` L62-L165，完整流程分为 9 步：

```
BuildTreeGraph(id, query)
  │
  ├─ 1. 加载目标树并定位节点
  │     LoadTreeByBlockID(id) + treenode.GetNodeInTree
  │
  ├─ 2. 查询条件构造
  │     query2Stmt(query)  // 关键字/标签/ID 解析
  │     + graphTypeFilter  // 按用户配置过滤块类型 p/h/m/c/t/l/i/b/s/callout/d
  │     + graphDailyNoteFilter
  │     + 将 content 替换为 ref.content (使搜索条件作用于引用内容)
  │
  ├─ 3. buildFullLinks(stmt)  ← 构建双向正反向引用关系
  │     └─ buildDefsAndRefs
  │          └─ sql.DefRefs(condition, MaxBlocks)  // SQL: JOIN blocks + refs 表
  │
  ├─ 4. 获取当前文档/块的子块树
  │     ├─ 文档节点：sql.GetAllChildBlocks([rootID], stmt, MaxBlocks)
  │     └─ 非文档：sql.GetChildBlocks(block.ID, stmt, MaxBlocks)
  │
  ├─ 5. 文档级引用补全（仅当目标为文档时）
  │     ├─ 按引用处理：sql.QueryDefRootBlocksByRefRootID(rootID)
  │     │                → sql.QueryRefRootBlocksByDefRootIDs(...)
  │     └─ 按定义处理：sql.QueryRefRootBlocksByDefRootIDs([rootID])
  │                     + 日记路径二次过滤 (issue #7547)
  │
  ├─ 6. filterDailyNote + genTreeNodes  ← 生成块的父子层级节点和边
  │
  ├─ 7. growTreeGraph → growLinkedNodes  ← 正反向 BFS 扩展引用网络
  │     └─ 最多扩展 16 层 forwardDepth/backDepth
  │
  ├─ 8. buildLinks  ← 生成引用边（From=ref.ID, To=def.ID, Ref=true）
  │
  ├─ 9. linkTagBlocks + markLinkedNodes + pruneUnref + removeDuplicatedUnescape
        └─ markLinkedNodes: 用 log2(Defs) 计算节点 Size
        └─ pruneUnref: 按 Global.MinRefs 过滤孤立节点
```

### 3.3 buildFullLinks 与 sql.DefRefs

`kernel/model/backlink.go` L934-L999 是关系图和反链面板共享的引用聚合核心：

```go
func buildFullLinks(condition string) (forwardlinks, backlinks []*Block) {
    defs := buildDefsAndRefs(condition)
    backlinks = append(backlinks, defs...)
    for _, def := range defs {
        for _, ref := range def.Refs {
            forwardlinks = append(forwardlinks, ref)  // 扁平化：refs 列表 = 正向链接
        }
    }
    return
}
```

`buildDefsAndRefs` 调用 **`sql.DefRefs`** (`kernel/sql/block_ref_query.go` L453-L502)，其 SQL 策略为**两轮扫描**：

```sql
-- 第一轮：获取所有 ref 块及其关联键
SELECT ref.*, r.block_id || '@' || r.def_block_id AS rel
FROM blocks AS ref, refs AS r
WHERE ref.id = r.block_id AND <condition>

-- 第二轮：获取所有 def 块（带 LIMIT）
SELECT def.* FROM blocks AS def, refs AS r
WHERE def.id = r.def_block_id LIMIT ?
```

在 Go 代码中通过 `rel = block_id@def_block_id` 字符串键把两个结果集拼接成 `map[*Block]*Block`（def → ref）双向结构，最终组装为每个 Block 的 `Defs []*Block` 和 `Refs []*Block` 字段。

### 3.4 引用查询函数一览（关系图使用）

| 查询函数 | 位置 | 用途 |
|---------|------|------|
| `sql.DefRefs(condition, limit)` | `kernel/sql/block_ref_query.go` L453 | 构建 def↔ref 双向 Block 映射（关系图/反链核心） |
| `sql.QueryDefRootBlocksByRefRootID(refRootID)` | `kernel/sql/block_ref_query.go` L162 | 查哪些文档被当前文档引用（作为 ref 方） |
| `sql.QueryRefRootBlocksByDefRootIDs(defRootIDs)` | `kernel/sql/block_ref_query.go` L177 | 按 defRootID 分组查引用该文档的所有 ref 文档 |
| `sql.QueryRefsByDefID(defID, containChildren)` | `kernel/sql/block_ref_query.go` L413 | 查指定 def 块的所有 refs（含/不含子块） |
| `sql.QueryRefIDsByDefID(defID, containChildren)` | `kernel/sql/block_ref_query.go` L360 | 只返回 ref 块 ID（轻量查询） |

### 3.5 growLinkedNodes — 图的 BFS 扩展

`kernel/model/graph.go` L301-L367 实现最大 16 层的正反向广度优先扩展：

```go
func growLinkedNodes(forwardlinks, backlinks *[]*Block, nodes, all *[]*GraphNode, forwardDepth, backDepth *int) {
    // Forward: 当前 node 在 forwardlinks 中 → 收集它的 Defs （即它引用了谁）
    if 16 > *forwardDepth {
        for _, ref := range *forwardlinks {
            for _, node := range *nodes {
                if node.ID == ref.ID {
                    for _, refDef := range ref.Defs { /* 添加被引用节点 */ }
                }
            }
        }
    }
    // Backward: 当前 node 在 backlinks 中 → 收集它的 Refs （即谁引用了它）
    if 16 > *backDepth {
        for _, def := range *backlinks {
            for _, node := range *nodes {
                if node.ID == def.ID {
                    for _, ref := range def.Refs { /* 添加引用者节点 */ }
                }
            }
        }
    }
    *forwardDepth++
    *backDepth++
    growLinkedNodes(forwardlinks, backlinks, generation, nodes, forwardDepth, backDepth)  // 递归
}
```

### 3.6 节点大小与标记

`kernel/model/graph.go` L425-L456 `markLinkedNodes` 遍历所有 link，对节点统计：
- **Defs++**：节点作为被引用方（入度），使用 `log2(Defs) * baseSize + baseSize` 做非线性放大
- **Refs++**：节点作为引用方（出度），线性累加

### 3.7 剪枝与去重

`kernel/model/graph.go` L471-L511 `pruneUnref`：
- 按 `Global.MinRefs` 过滤：`node.Refs` 或 `node.Defs` 任一达到阈值才保留
- 按 `MaxBlocks`（默认 2000）限制最大渲染节点数
- 二次扫描 links，剔除端点不在最终 nodes 集合中的悬挂边

---

## 4. 与索引维护、缓存一致性的关联核对

### 4.1 引用缓存写入与索引维护的时间线

下表展示一次编辑（doUpdate 包含 block-ref 变更）从缓存写入到索引落地的完整时序：

```
T1  doUpdate 遍历新 subTree
    │  ├─ sql.CacheRef() → 写 defIDRefsCache (go-cache)           ←【缓存优先，即时可读】
    │  ├─ 从 DynamicRefTexts sync.Map 读取动态锚文本覆盖            ←【读内存缓存】
    │  └─ 对比 oldDefIDs vs newDefIDs
    │       └─ 如有变化：task.AppendAsyncTask(SetDefRefCount)       ←【延迟计数刷新】
    │
T2  tx.commit()
    │  ├─ writeTreeUpsertQueue(tree)                                ←【树入 SQL 异步写队列】
    │  │    └─ sql.UpsertTreeQueue → ... → upsertRefs()
    │  │         └─ deleteRefsByPath + insertRefs                   ←【先删后插，幂等更新 refs 表】
    │  │
    │  └─ refreshDynamicRefTexts(tx.nodes, tx.trees)                ←【级联刷新锚文本】
    │       └─ 第 N 轮 refreshDynamicRefTexts0
    │            ├─ getRefsCacheByDefNode → sql.GetRefsCacheByDefID ←【优先读 go-cache，miss 则查 DB】
    │            ├─ updateRefText → SetDynamicBlockRefText
    │            │    ├─ AST 插入 NodeBlockRefDynamicText           ←【修改内存树】
    │            │    └─ DynamicRefTexts.Store(defID, text)         ←【更新 sync.Map 缓存】
    │            ├─ sql.UpdateRefsTreeQueue(refTree)                ←【变更的引用树 refs 入队】
    │            └─ indexWriteTreeUpsertQueue(tree)                 ←【引用树重新入 SQL 写队列】
    │
T3  sql.FlushQueue() 触发（util.SQLFlushInterval ≈ 2s）
    │  └─ 批量事务写入 SQLite refs / blocks / blocks_fts 表         ←【持久化落地】
    │     └─ 发布 EvtSQLIndexFlushed 事件                            ←【缓存一致性通知】
    │
T4  异步任务 SetDefRefCount 执行（延迟 util.SQLFlushInterval）
    │  └─ refreshRefCount(blockID)
    │       ├─ sql.FlushQueue()                                     ←【先阻塞等待写队列落地，避免脏读】
    │       ├─ sql.QueryRefIDsByDefID(...)                          ←【从 DB 查询精确引用数】
    │       └─ util.PushSetDefRefCount(...)                         ←【推送到前端 UI 更新角标】
```

### 4.2 三层缓存的一致性保障机制

| 缓存层 | 写入时机 | 失效/清理时机 | 一致性风险点 |
|--------|---------|-------------|------------|
| **L1: `defIDRefsCache`** (go-cache 30min) | ① `sql.CacheRef()` doUpdate 实时写入<br>② `GetRefsCacheByDefID` miss 时回填 | ① `removeRefCacheByDefID(defID)` 显式删除<br>② 30 分钟 TTL 自动过期 | **缓存写穿透缺失**：`doInsert/doDelete/doMove` 不直接更新缓存，依赖 TTL 或后续 miss 回填。短期可能读到旧引用列表。 |
| **L2: `DynamicRefTexts`** (sync.Map) | `SetDynamicBlockRefText()` 修改 AST 时同步写入 | 进程重启即丢失（无持久化） | **启动瞬间空白**：刚启动时 Map 为空，前几次 doUpdate 无法读到动态锚文本，需依赖 DB 重建。 |
| **L3: `blockCache`** (ristretto LRU) | `putBlockCache()` SQL 查询后回填 | `removeBlockCache(id)` 关联触发 `removeRefCacheByDefID` | **块删除联动缺失**：删除块时必须显式调用 `removeBlockCache` 才会同时清理引用缓存。 |

### 4.3 关键一致性设计

#### 4.3.1 写队列 + Flush 屏障

`refreshRefCount`（`kernel/model/push_reload.go` L222-L224）在查询引用计数前先调用 `sql.FlushQueue()`：

```go
func refreshRefCount(blockID string) {
    sql.FlushQueue()   // 阻塞等待所有待写操作落地到 SQLite
    bt := treenode.GetBlockTree(blockID)
    ...
    refIDs := sql.QueryRefIDsByDefID(bt.ID, isDoc)  // 此时查询 DB 是最新的
    ...
}
```

这是最核心的一致性保障：**任何需要精确引用数据的异步任务，执行前必须先 FlushQueue**。

#### 4.3.2 唯一任务去重

`task.AppendAsyncTaskWithDelay(task.SetDefRefCount, ...)` 利用 `uniqueActions` 机制（在 `kernel/task/queue.go` 中定义）保证同一 defID 的计数刷新任务在队列里最多只存在一个，避免短时间多次编辑导致的重复刷新风暴。

#### 4.3.3 upsertRefs 的先删后插幂等策略

`kernel/sql/block_ref.go` L70 核心思路：

```go
func upsertRefs(tx *sql.Tx, tree *parse.Tree) (err error) {
    deleteRefsByPath(tx, tree.Box, tree.Path)           // 1) 删除该文档所有旧 refs
    deleteFileAnnotationRefsByPath(tx, tree.Box, tree.Path)
    err = insertRefs(tx, tree)                           // 2) 重新插入当前所有 refs
    return
}
```

配合 `sql.UpdateRefsTreeQueue(refTree)` 只需传入 tree（按 box+path 删除），即可保证即使同一文档被多次调度更新，最终结果一致。

#### 4.3.4 go-cache miss 的 DB 回退

`kernel/sql/cache.go` L86-L101：

```go
func GetRefsCacheByDefID(defID string) (ret []*Ref) {
    // 先遍历 go-cache 查找
    for defBlockID, refs := range defIDRefsCache.Items() { ... }
    if 1 > len(ret) {
        // Cache miss → 直查 SQLite 并回填缓存
        ret = QueryRefsByDefID(defID, false)
        for _, ref := range ret {
            putRefCache(ref)
        }
    }
    return
}
```

### 4.4 已识别的一致性边界问题

| 场景 | 影响 | 缓解措施 |
|------|------|---------|
| **doInsert/doDelete 不写 go-cache** | 刚插入/删除引用块后的 `refreshDynamicRefTexts0` 若命中缓存可能漏处理 | ① 任务延迟了 `SQLFlushInterval`；② 级联刷新最多 7 次迭代；③ 下一次 miss 会从 DB 回填 |
| **DynamicRefTexts sync.Map 无持久化** | 进程重启后首次编辑时，doUpdate 无法读到缓存锚文本（issue #5891 缓解方案） | `doUpdate` 中判断 `ok && "" != dRefText`，不满足时保留原值 |
| **7 次级联上限** | A→B→C→D→E→F→G→H（8 跳链式引用）最末级 H 的锚文本不会自动刷新 | 上限设计为防止循环引用导致的死循环；实际可接受 |
| **关系图实时性** | 关系图不主动订阅 EvtSQLIndexFlushed，需用户手动刷新 | 用户点击关系图时实时查询 DB，不缓存 |
| **WriteTreeUpsertQueue 与 CacheRef 不一致窗口** | `CacheRef` 已写 go-cache，但 SQL 队列未 Flush 前，DB 仍为旧值 | 仅 `GetRefsCacheByDefID` 这种"读缓存优先"的函数可能受影响；精确计数刷新前先 FlushQueue |

---

## 5. 附：关键相对路径索引

### 5.1 编辑事务与引用缓存

| 相对路径 | 行范围 | 内容摘要 |
|---------|--------|---------|
| `kernel/model/transaction.go` | L64-L132 | 事务队列：txQueue channel、flushQueue、flushTx 加锁执行 |
| `kernel/model/transaction.go` | L1410-L1594 | `doUpdate`：CacheRef 写入 + DynamicRefTexts 读取 + 引用 ID 对比 |
| `kernel/model/transaction.go` | L961-L1027 | `doDelete0`：getRefDefIDs + SetDefRefCount 调度 |
| `kernel/model/transaction.go` | L1280-L1367 | `doInsert` 引用 ID 收集与计数刷新 |
| `kernel/model/transaction.go` | L1619-L1635 | `getRefDefIDs`：AST 扫描 block-ref/embed-block-ref |
| `kernel/model/transaction.go` | L1865-L1895 | `tx.commit()`：writeTreeUpsertQueue + refreshDynamicRefTexts |
| `kernel/model/transaction.go` | L1951-L2001 | `getRefsCacheByDefNode`：容器块向上/向下展开查找引用 |
| `kernel/model/transaction.go` | L2033-L2068 | `updateRefText`：AST 内匹配 defID 并调用 SetDynamicBlockRefText |

### 5.2 动态锚文本同步

| 相对路径 | 行范围 | 内容摘要 |
|---------|--------|---------|
| `kernel/model/push_reload.go` | L221-L249 | `refreshRefCount`：FlushQueue 后查 DB，推送引用计数 |
| `kernel/model/push_reload.go` | L251-L280 | `refreshDynamicRefTexts`：7 次级联迭代框架 |
| `kernel/model/push_reload.go` | L282-L358 | `refreshDynamicRefTexts0`：收集 refs→遍历引用树→UpdateRefsTreeQueue→AV 同步→写树 |
| `kernel/model/blockinfo.go` | L278-L345 | `getNodeRefText` / `getNodeRefText0`：def 节点到锚文本的渲染策略 |
| `kernel/treenode/node.go` | L110-L118 | `GetBlockRef`：提取 refID/refText/subtype |
| `kernel/treenode/node.go` | L453-L478 | `DynamicRefTexts` sync.Map + `SetDynamicBlockRefText`：AST+内存双写 |

### 5.3 引用缓存与 SQL 查询

| 相对路径 | 行范围 | 内容摘要 |
|---------|--------|---------|
| `kernel/sql/cache.go` | L42-L81 | ristretto blockCache：put/get/remove（含 remove 级联清理 ref cache） |
| `kernel/sql/cache.go` | L84-L119 | go-cache defIDRefsCache：CacheRef/putRefCache/GetRefsCacheByDefID/removeRefCacheByDefID |
| `kernel/sql/block_ref_query.go` | L99-L119 | `QueryRefCount`：按 defID 分组统计引用数 |
| `kernel/sql/block_ref_query.go` | L162-L202 | `QueryDefRootBlocksByRefRootID` / `QueryRefRootBlocksByDefRootIDs`：关系图文档级引用查询 |
| `kernel/sql/block_ref_query.go` | L360-L383 | `QueryRefIDsByDefID`：轻量 ID 查询（计数刷新用） |
| `kernel/sql/block_ref_query.go` | L413-L436 | `QueryRefsByDefID`：完整 Ref 结构体查询 |
| `kernel/sql/block_ref_query.go` | L453-L502 | `DefRefs`：两轮 SQL 扫描构建 def↔ref Block 映射 |

### 5.4 关系图

| 相对路径 | 行范围 | 内容摘要 |
|---------|--------|---------|
| `kernel/api/graph.go` | L53-L163 | HTTP 接口：getGraph/getLocalGraph（含配置临时覆盖） |
| `kernel/model/graph.go` | L62-L165 | `BuildTreeGraph`：局部关系图 9 步构建流程 |
| `kernel/model/graph.go` | L167-L218 | `BuildGraph`：全局关系图构建流程 |
| `kernel/model/graph.go` | L296-L367 | `growTreeGraph` / `growLinkedNodes`：16 层 BFS 正反向扩展 |
| `kernel/model/graph.go` | L378-L423 | `buildLinks` / `genTreeNodes`：引用边与父子层级边生成 |
| `kernel/model/graph.go` | L425-L511 | `markLinkedNodes` / `pruneUnref`：节点大小计算与孤立节点剪枝 |
| `kernel/model/backlink.go` | L934-L999 | `buildFullLinks` / `buildDefsAndRefs`：关系图与反链共享的引用聚合核心 |
