# SiYuan 标签与属性管理实现架构分析

## 1. 整体架构概览

SiYuan 的标签与属性管理体系遵循「前端交互 → API 层 → 业务模型层 → 缓存层 → 数据索引层 → 持久化层」的多层架构设计，各层之间通过明确的接口边界协作，实现标签识别、属性读写、索引维护、界面展示与搜索过滤的完整闭环。

### 1.1 模块划分

| 层级 | 模块路径 | 核心职责 |
|------|---------|---------|
| **前端 UI 层** | `app/src/layout/dock/Tag.ts`、`app/src/menus/tag.ts` | 标签面板渲染、右键菜单、事件监听、实时刷新 |
| **API 路由层** | `kernel/api/tag.go`、`kernel/api/attr.go` | HTTP 接口定义、参数校验、权限控制 |
| **业务模型层** | `kernel/model/tag.go`、`kernel/model/blockial.go`、`kernel/model/transaction.go` | 核心业务逻辑：标签 CRUD、属性读写、事务处理、批量操作 |
| **内存缓存层** | `kernel/cache/ial.go` | 基于 Ristretto 的块属性/文档属性缓存（200MB 上限） |
| **SQL 索引层** | `kernel/sql/span.go`、`kernel/sql/upsert.go`、`kernel/sql/queue.go`、`kernel/sql/attribute.go` | 数据库表结构、异步索引队列、标签 Span 提取、属性行存储 |
| **索引构建层** | `kernel/model/index.go`、`kernel/model/tree.go` | 文档树加载、全量/增量索引调度、BlockTree 索引 |
| **持久化层** | `filesys/*.go`（文件系统） | `.sy` JSON 文档持久化、文件锁 |

---

## 2. 标签识别与解析机制

### 2.1 标签的两种存在形态

SiYuan 中的标签以**两种互相关联的形态**存在，对应不同的存储位置：

#### 形态一：行内标签（Inline Tag，TextMark 节点）
- **解析时机**：由 Lute 引擎在 Markdown 解析阶段，识别 `#标签名#` 语法并转换为 `NodeTextMark` 类型节点，`TextMarkType = "tag"`
- **存储位置**：嵌入在文档树的块内容内，随块一起序列化为 JSON 存储到 `.sy` 文件
- **索引提取**：在 SQL 索引阶段通过 `ast.Walk` 遍历提取到 `spans` 表

核心提取代码位于 `kernel/sql/database.go` 的 `tagFromNode` 函数（约 L913-L944）：

```go
func tagFromNode(node *ast.Node) (ret string) {
    tagBuilder := bytes.Buffer{}
    if ast.NodeDocument == node.Type {
        // 文档标签从 IAL 的 tags 属性提取
        tagIAL := html.UnescapeString(node.IALAttr("tags"))
        tags := strings.Split(tagIAL, ",")
        for _, t := range tags {
            tagBuilder.WriteString("#" + t + "# ")
        }
        return strings.TrimSpace(tagBuilder.String())
    }
    // 行内标签通过遍历 TextMark 节点提取
    ast.Walk(node, func(n *ast.Node, entering bool) ast.WalkStatus {
        if !entering { return ast.WalkContinue }
        if n.IsTextMarkType("tag") {
            tagBuilder.WriteString("#" + n.Content() + "# ")
        }
        return ast.WalkContinue
    })
    return strings.TrimSpace(tagBuilder.String())
}
```

#### 形态二：文档标签（Document Tag，IAL 属性）
- **解析时机**：在块属性面板中设置，或通过 API `/api/attr/setBlockAttrs` 对文档根节点设置 `tags` 属性
- **存储位置**：存储在文档根节点的 IAL（Inline Attribute List）中，格式为逗号分隔字符串，如 `tags="工作,重要"`
- **特殊处理**：在 `kernel/model/blockial.go` 的 `setNodeAttrs0` 中对 `tags` 属性做去重和规范化处理

### 2.2 标签的树形层级解析

标签支持层级结构（如 `项目/后端/API`），在 `kernel/model/tag.go` 的 `buildTags` 函数中按 `/` 分隔符递归构建层级树：

```go
func buildTags(root Tags, labels []string, depth int) Tags {
    i := 0
    for ; i < len(root); i++ {
        if (root)[i].Name == labels[0] { break }
    }
    if i == len(root) {
        root = append(root, &Tag{Name: util.EscapeHTML(labels[0]), Type: "tag", Depth: depth})
    }
    depth++
    root[i].tags = buildTags(root[i].tags, labels[1:], depth)
    return root
}
```

---

## 3. 属性（IAL）读写与数据模型

### 3.1 属性命名规则与验证

属性名在 `kernel/model/blockial.go` 中有严格的验证逻辑 `isValidAttrName`：

| 规则 | 说明 |
|------|------|
| 首字符 | 必须为小写字母 `a-z` |
| 后续字符 | 小写字母 `a-z`、数字 `0-9`、连字符 `-` |
| 自定义属性 | 必须使用 `custom-` 前缀，前缀后首字符仍需为小写字母 |
| 大小写 | 属性名统一转为小写存储，防止重复 |
| 受保护属性 | `data-task` 不允许通过通用接口修改 |

### 3.2 属性写入的标准流程（setNodeAttrs0）

`setNodeAttrs0` 是所有属性修改的底层入口，执行以下关键步骤：

1. **提取旧属性**：通过 `parse.IAL2Map` 和 `parse.IAL2MapUnEsc` 分别获取转义和未转义版本
2. **属性值清理**：`RemoveInvalidRetainCtrl` 移除非法控制字符，`TrimSpace` 去除首尾空白
3. **命名验证**：`isValidAttrName` 验证失败则直接返回错误
4. **tags 属性特殊处理**：按逗号分割、去重、去除空值后重新拼接
5. **空值 = 删除**：value 为空字符串时，从 IAL 中移除该属性
6. **大小写归一化**：属性名转为小写后存储，同时保留对含大写字母属性的精确删除支持
7. **HTML 转义**：属性值通过 `html.EscapeAttrVal` 转义
8. **触发标签刷新**：若 `tags` 属性变化，调用 `ReloadTag()` 通知前端刷新标签面板

### 3.3 属性的三层存储模型

| 存储层 | 位置 | 格式 | 访问方式 | 一致性保证 |
|--------|------|------|---------|-----------|
| **源存储** | `.sy` 文件中的文档树 JSON | `KramdownIAL` 节点字段 | `node.IALAttr()` / `node.SetIALAttr()` | 文件写入时保证 |
| **缓存层** | Ristretto 内存缓存（`blockIALCache` / `docIALCache`） | `map[string]string` | `cache.PutBlockIAL()` / `cache.GetBlockIAL()` | 属性修改后同步更新 |
| **索引层** | SQLite 数据库 | `blocks` 表 `ial` 列（整体 JSON） + `attributes` 表（逐行展开） | `sql.GetBlockAttrs()` / `sql.BatchGetBlockAttrs()` | 异步队列 `UpsertTreeQueue` 保证最终一致 |

### 3.4 属性的逐行展开索引

在 `kernel/sql/database.go` 的 `buildAttributeFromNode` 中，属性从 IAL 中筛选并逐行写入 `attributes` 表，`isAttr` 函数定义了需要被索引的属性白名单：

```go
func isAttr(name string) bool {
    return strings.HasPrefix(name, "custom-") || 
           "name" == name || "alias" == name || "memo" == name || 
           "bookmark" == name || "fold" == name || "heading-fold" == name || "style" == name
}
```

> **设计要点**：只有 `custom-*` 前缀和少数内置属性才会被展开到 `attributes` 表，其他 IAL 属性（如 `id`、`created`、`updated`）仅保留在 `blocks.ial` 字段中，避免索引膨胀。

---

## 4. 索引更新流程

### 4.1 索引队列与异步刷新

索引更新采用**异步批量队列机制**，核心实现在 `kernel/sql/queue.go`。

#### 队列操作类型

| action | 触发场景 | 说明 |
|--------|---------|------|
| `index` | 首次构建索引 / Box.Index() | 完全重建文档下所有索引表 |
| `upsert` | 编辑后保存 / 属性修改 | **blocks 表 Hash 增量 + 子表全量重建** |
| `delete` | 删除文档/目录 | 按路径前缀批量删除 |
| `delete_id` | 删除单个文档 | 按 root_id 删除 |
| `rename` | 文档重命名 | 更新 hpath 和文档标题 |
| `move` | 文档移动 | 更新 path |
| `update_refs` | 引用锚文本变更/反向链接刷新/启动引用索引 | **仅重建 refs + file_annotation_refs，不涉及其他表** |
| `delete_refs` | 删除文档引用 | 仅删除 refs + file_annotation_refs |
| `index_node` | 单块内容重索引 | 仅更新单块 content 字段 |

#### 队列去重优化

同一文档的同类型操作会被覆盖合并，避免重复索引。例如 `UpsertTreeQueue`：

```go
func UpsertTreeQueue(tree *parse.Tree) {
    newOp := &dbQueueOperation{upsertTree: tree, action: "upsert"}
    for i, op := range operationQueue {
        if "upsert" == op.action && op.upsertTree.ID == tree.ID {
            operationQueue[i] = newOp  // 直接覆盖（Last Write Wins）
            return
        }
    }
    appendOperation(newOp)
}
```

### 4.2 Upsert 增量索引的真实机制（修正版）

> ⚠️ **重要修正**：之前对 upsert 增量更新的理解存在偏差。实际上，**只有 `blocks` 主表做了 Hash 对比的增量更新，而 `spans`、`assets`、`attributes`、`refs` 等子表全部是按文档粒度全量删除后重新插入**。

在 `kernel/sql/upsert.go` 的 `upsertTree` 函数中，完整执行流程如下：

#### 步骤 1：计算 blocks 表的增量（仅 blocks 表）

```go
oldBlockHashes := queryBlockHashes(tx, tree.ID)   // 查询旧块 Hash
blocks, spans, assets, attributes := fromTree(tree.Root, tree)  // 从新树提取全量数据

// 计算未变化的块（Hash 相同）
unChanges := hashset.New()
var toRemoves []string
for id, hash := range oldBlockHashes {
    if newHash, ok := newBlockHashes[id]; ok {
        if newHash == hash {
            unChanges.Add(id)  // 未变化，跳过
        }
    } else {
        toRemoves = append(toRemoves, id)  // 旧有新无：需要删除
    }
}

// 过滤 blocks：只保留有变化的块用于插入
tmp := blocks[:0]
for _, b := range blocks {
    if !unChanges.Contains(b.ID) {
        tmp = append(tmp, b)
    }
}
blocks = tmp

// 有变化的块也加入 toRemoves（先删后插）
for _, b := range blocks {
    toRemoves = append(toRemoves, b.ID)
}
```

#### 步骤 2：删除操作（分层策略）

| 表 | 删除范围 | 删除粒度 |
|-----|---------|---------|
| `blocks` | `deleteBlocksByIDs(tx, toRemoves)` | 仅删除 Hash 变化的 + 旧有新无的块（**增量删除**） |
| `spans` | `deleteSpansByRootID(tx, tree.ID)` | 整文档所有 spans 全部删除（**全量删除**） |
| `assets` | `deleteAssetsByRootID(tx, tree.ID)` | 整文档所有 assets 全部删除（**全量删除**） |
| `attributes` | `deleteAttributesByRootID(tx, tree.ID)` | 整文档所有 attributes 全部删除（**全量删除**） |
| `refs` | `deleteRefsByPathTx(tx, tree.Box, tree.Path)` | 整文档所有 refs 全部删除（**全量删除**） |
| `file_annotation_refs` | `deleteFileAnnotationRefsByPathTx(tx, tree.Box, tree.Path)` | 整文档全部删除（**全量删除**） |

#### 步骤 3：插入操作

```go
refs, fileAnnotationRefs := refsFromTree(tree)
insertTree0(tx, tree, context, blocks, spans, assets, attributes, refs, fileAnnotationRefs)
```

在 `insertTree0` 中还有一个额外细节：**spans 插入前会再删一次**（`deleteSpansByRootID(tx, tree.Root.ID)`），注释说是「移除文档标签，否则会重复添加」（对应 issue #3723），相当于双保险。

#### 各表更新策略汇总

| 表 | 更新策略 | 粒度 | 设计原因 |
|-----|---------|------|---------|
| **blocks** | Hash 增量对比，先删后插 | 块级 | 主表数据量大，块级 Hash 易计算，增量收益最高 |
| **spans** | 全量删除 + 全量插入 | 文档级 | 单文档内 span 数量有限，全量重建实现简单、不易出错 |
| **assets** | 全量删除 + 全量插入 | 文档级 | 同上 |
| **attributes** | 全量删除 + 全量插入 | 文档级 | 同上，且属性逐行展开后 diff 逻辑复杂 |
| **refs** | 全量删除 + 全量插入 | 文档级（按 path） | 引用关系整体重建更可靠 |
| **file_annotation_refs** | 全量删除 + 全量插入 | 文档级（按 path） | 同上 |

> **设计权衡**：只有主表 blocks 做增量，子表全部全量重建。这种「主表精细化 + 子表简单化」的设计，在保证主要性能收益的同时，大大降低了子表增量 diff 的实现复杂度和出错概率。

### 4.3 引用索引的单独刷新（update_refs）

> ⚠️ **修正**：之前将 `update_refs` 描述为「增量更新 refs 表」是不准确的。实际上 `update_refs` 只涉及 refs 和 file_annotation_refs 两个引用表的**全量删除+全量插入**，且**完全不涉及 blocks/spans/assets/attributes 表**。

#### upsertRefs 的执行逻辑

`kernel/sql/block_ref.go` 中 `upsertRefs` 的实现非常简洁：

```go
func upsertRefs(tx *sql.Tx, tree *parse.Tree) (err error) {
    deleteRefsByPath(tx, tree.Box, tree.Path)           // 删除旧引用
    deleteFileAnnotationRefsByPath(tx, tree.Box, tree.Path) // 删除旧文件标注引用
    insertRefs(tx, tree)                                 // 从文档树重新提取并插入
}
```

关键特征：
- **只操作 refs + file_annotation_refs 两张表**，不影响 blocks/spans/assets/attributes
- **全量删除+全量插入**（按 path 删除后重新提取）
- **不受索引忽略配置影响**（`upsertRefs` 中没有 `getIndexIgnoreLines` 检查）
- **与 upsert 操作的独立性**：`upsertTree` 中也会重建 refs，两者路径不同但最终效果相同

#### update_refs 的三种触发场景

| 触发路径 | 入口函数 | 场景 |
|---------|---------|------|
| **反向链接手动刷新** | `kernel/model/backlink.go` → `refreshRefsByDefID(defID)` | 用户点击「刷新」反向链接按钮，按 defID 查找所有引用该定义块的文档，逐文档入 `UpdateRefsTreeQueue` |
| **动态锚文本级联更新** | `kernel/model/push_reload.go` → `refreshDynamicRefTexts0()` | 定义块内容变化后，级联更新引用该块的所有文档中的动态锚文本。对受影响的引用文档同时入 `UpdateRefsTreeQueue` + `indexWriteTreeUpsertQueue`（两个操作分别入队） |
| **启动时引用索引构建** | `kernel/model/index.go` → `IndexRefs()` | 启动时两阶段构建：先扫描所有含引用的文档 rootID，再逐文档 `UpdateRefsTreeQueue` |

#### update_refs 与 upsert 的协作与竞态

在动态锚文本更新场景中，同一文档可能同时进入队列两个操作：`update_refs` + `upsert`。由于队列去重只针对**同 action** 的同文档操作（`update_refs` 和 `upsert` 是不同 action，不会互相覆盖），两者都会被执行：

1. 如果 `update_refs` 先执行：重建 refs → 随后 `upsert` 执行时会**再次删除并重建 refs**（upsertTree 步骤 2 中的 `deleteRefsByPathTx`），等于 `update_refs` 的结果被 `upsert` 覆盖
2. 如果 `upsert` 先执行：完整重建所有表 → 随后 `update_refs` 只重建 refs（删了再插），但此时 refs 内容与 upsert 结果一致，属于冗余操作但不会出错

> **一致性影响**：无论哪种执行顺序，最终 refs 都与文档树一致。但 `update_refs` 的「仅刷引用」语义在遇到同文档 `upsert` 时会被降级为冗余操作。

### 4.4 索引忽略配置的影响范围

SiYuan 支持通过 `{DataDir}/.siyuan/indexignore` 文件配置忽略索引的文档路径（对应 issue #9198）。索引忽略的检查位于 `insertTree0` 函数开头：

```go
func insertTree0(tx *sql.Tx, tree *parse.Tree, ...) (err error) {
    if ignoreLines := getIndexIgnoreLines(); 0 < len(ignoreLines) {
        matcher := ignore.CompileIgnoreLines(ignoreLines...)
        if matcher.MatchesPath("/" + path.Join(tree.Box, tree.Path)) {
            return  // 命中忽略规则，跳过所有插入
        }
    }
    // ... 实际插入 blocks/spans/assets/attributes/refs
}
```

#### 各操作类型受索引忽略影响的情况

| 操作 | 是否受索引忽略影响 | 影响 |
|------|-------------------|------|
| `index`（indexTree） | **是**（通过 `insertTree0`） | 全量索引时跳过被忽略文档，**无副作用**（因为 indexTree 不做前置删除） |
| `upsert`（upsertTree） | **是**（通过 `insertTree0`） | ⚠️ **有副作用**：upsertTree 先执行了子表全量删除，再调 `insertTree0`，如果此时命中忽略规则，插入被跳过但删除已执行，**造成 spans/assets/attributes/refs 数据丢失** |
| `update_refs`（upsertRefs） | **否** | `upsertRefs` 直接操作 refs 表，无索引忽略检查 |
| 其他操作 | **否** | `delete`/`rename`/`move` 等不经过 `insertTree0` |

#### upsert 命中索引忽略时的数据丢失路径

```
upsertTree 执行流程（命中忽略时）：
  1. 计算 blocks 增量                                ← 已执行
  2. deleteBlocksByIDs(toRemoves)                    ← 已执行：blocks 被删
  3. deleteSpansByRootID(tree.ID)                    ← 已执行：spans 被删
  4. deleteAssetsByRootID(tree.ID)                   ← 已执行：assets 被删
  5. deleteAttributesByRootID(tree.ID)               ← 已执行：attributes 被删
  6. deleteRefsByPathTx(tree.Box, tree.Path)         ← 已执行：refs 被删
  7. deleteFileAnnotationRefsByPathTx(...)           ← 已执行
  8. insertTree0 → getIndexIgnoreLines → 命中! → return ← 插入全部跳过
                                                     ← 结果：该文档的索引数据被清空
```

> **一致性影响**：如果被忽略文档此前已被索引过（索引忽略配置是后加的），一旦触发 upsert，该文档的 spans（标签索引）、attributes（属性索引）、refs（引用索引）将全部丢失。但 blocks 表中 Hash 未变化的旧块不会受影响（步骤 2 只删了变化的块），而变化的块被删后也不再插入。这意味着被忽略文档在 upsert 后将**几乎完全从索引中消失**——这是预期行为（配置忽略的目的就是如此），但实现方式不是「不索引」而是「删了不插」。

> **注意**：`update_refs` 不受索引忽略影响，意味着即使文档被配置为忽略索引，当其他文档引用它时，引用关系仍然会被更新到 refs 表。这在索引忽略场景下可能导致 refs 中存在「指向一个索引中不存在块」的悬挂引用。

#### 索引忽略配置的生效机制

- 配置文件路径：`{DataDir}/.siyuan/indexignore`
- 匹配规则：使用 gitignore 语法（`go-gitignore` 库），匹配路径格式为 `/{boxID}/{docPath}`
- 缓存机制：首次读取后设置 `IndexIgnoreCached = true`，后续直接返回缓存结果
- 缓存失效：`FullReindex()` 中显式设置 `sql.IndexIgnoreCached = false`，触发重新加载

#### 三种生命周期路径下的索引状态对比

根据索引忽略配置生效的时机和触发操作的不同，被忽略文档在各索引表中的最终状态存在显著差异。以下是三种典型路径的详细对比：

##### 路径一：全量重建（FullReindex）时文档已被配置为忽略

即配置 `indexignore` 在先，然后执行完整重建（`FullReindex` → `InitDatabase(true)` → `indexBox`）。

**执行流程**：
1. `InitDatabase(true)` 清空数据库所有表
2. `indexBox` 逐文档加载并调用 `IndexTreeQueue`（action="index"）
3. `indexTree` → `insertTree0` → 命中 `indexignore` → 直接 return
4. 随后 `IndexRefs` 阶段直接遍历文件系统，发现含引用的文档 → 调用 `UpdateRefsTreeQueue`（action="update_refs"）
5. `upsertRefs` 无索引忽略检查 → refs 被正常写入

**各索引表最终状态**：

| 索引表 | 状态 | 原因 |
|--------|------|------|
| **blocks 表**（含 tag 列、ial 列） | ❌ 无数据 | `insertTree0` 被忽略，块级索引完全缺失 |
| **spans 表**（标签行级） | ❌ 无数据 | 同上，标签面板不会显示该文档的标签 |
| **assets 表** | ❌ 无数据 | 同上 |
| **attributes 表**（属性展开） | ❌ 无数据 | 同上，属性搜索搜不到该文档 |
| **refs 表**（引用） | ⚠️ **有数据**（若文档含引用） | `IndexRefs` 阶段通过 `update_refs` 补回，`upsertRefs` 无忽略检查 |
| **file_annotation_refs 表** | ⚠️ **有数据**（若文档含文件标注引用） | 同上 |

> **一致性影响**：全量重建后，被忽略文档的 refs 表与其他表出现不一致——refs 表中有「该文档引用了其他块」的记录，但该文档的块在 blocks/spans 表中不存在。这会导致反向链接面板中能找到这些引用，但点击跳转时可能找不到块内容。

##### 路径二：先正常索引，后加入忽略规则，再触发 upsert（编辑保存）

即文档一开始在索引中，后来被加入 `indexignore`，然后用户编辑该文档并保存（触发 `upsertTree`）。

**执行流程**：
1. upsertTree 先计算 blocks 增量 → 删除变化的块
2. upsertTree 全量删除 spans/assets/attributes/refs 子表
3. 调用 `insertTree0` → 命中 `indexignore` → return，所有插入被跳过

**各索引表最终状态**：

| 索引表 | 状态 | 原因 |
|--------|------|------|
| **blocks 表**（含 tag 列、ial 列） | ⚠️ **部分保留** | Hash 未变化的块保留旧数据，变化的块被删除且不重插。文档根节点（通常不会变）大概率保留，但内容块可能丢失 |
| **spans 表**（标签行级） | ❌ **全空** | 整文档全量删除后未重插，标签面板计数会减少 |
| **assets 表** | ❌ **全空** | 同上 |
| **attributes 表**（属性展开） | ❌ **全空** | 同上，属性搜索搜不到该文档的 custom 属性 |
| **refs 表**（引用） | ❌ **全空** | 同上 |
| **file_annotation_refs 表** | ❌ **全空** | 同上 |

> **标签索引一致性影响**：blocks 表的 tag 列（块级标签）可能还有残留（因为部分块未被删除），但 spans 表的行级标签索引已全部清空。两级标签索引不一致：全文搜索可能还能搜到标签（通过 blocks.tag 的 FTS），但标签面板统计计数会减少（基于 spans 表）。

> **属性索引一致性影响**：blocks.ial 中仍有完整属性数据（未变化的块），但 attributes 展开表已空。通过 `getBlockAttrs`（读 blocks.ial）仍能获取属性，但通过属性搜索/属性视图（查 attributes 表）找不到该文档。

##### 路径三：仅触发引用刷新（update_refs）

即文档已被配置为忽略索引，不做任何编辑保存，仅触发引用相关操作（手动刷新反向链接、动态锚文本级联更新、IndexRefs）。

**执行流程**：
1. `UpdateRefsTreeQueue` 入队 → action="update_refs"
2. `upsertRefs` → 先删后插 refs（**无索引忽略检查**）
3. 其他表完全不涉及

**各索引表最终状态**：

| 索引表 | 状态 | 原因 |
|--------|------|------|
| **blocks 表** | ✅ **不变** | `update_refs` 不操作 blocks |
| **spans 表**（标签） | ✅ **不变** | `update_refs` 不操作 spans |
| **assets 表** | ✅ **不变** | 同上 |
| **attributes 表**（属性） | ✅ **不变** | 同上 |
| **refs 表**（引用） | ✅ **正常重建** | `upsertRefs` 不受忽略影响，全量删插正常执行 |
| **file_annotation_refs 表** | ✅ **正常重建** | 同上 |

> **与路径二的组合效应**：如果路径二之后再触发引用刷新（很常见，因为编辑保存后通常会刷新反向链接），最终状态会变成：blocks 部分保留 + spans/assets/attributes 全空 + refs 被补回。即引用索引与其他索引的不一致程度进一步扩大。

##### 三种路径汇总对比表

| 路径 | blocks | tag 列（块级） | spans（标签行级） | attributes（属性） | refs（引用） |
|------|--------|--------------|-------------------|-------------------|-------------|
| 全量重建 + 忽略 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ⚠️ 有（含引用时） |
| 后加忽略 + upsert | ⚠️ 部分 | ⚠️ 部分残留 | ❌ 全空 | ❌ 全空 | ❌ 全空 |
| 仅触发 update_refs | ✅ 不变 | ✅ 不变 | ✅ 不变 | ✅ 不变 | ✅ 正常重建 |
| 后加忽略 + upsert + 再刷引用 | ⚠️ 部分 | ⚠️ 部分残留 | ❌ 全空 | ❌ 全空 | ✅ 被补回 |

##### 对标签、属性、引用三类索引一致性的具体影响

**标签索引**：存在两级不一致的风险
- blocks.tag（块级聚合，FTS 搜索用）：可能有残留（upsert 未变化的块保留）
- spans 表（行级，标签面板/标签搜索用）：全空
- 表现：全文搜索 `#标签名#` 可能还能搜到该文档的某些块，但标签面板的标签计数不包含该文档

**属性索引**：存在两层不一致的风险
- blocks.ial（整体 JSON，`GetBlockAttrs` 用）：可能有残留
- attributes 展开表（逐行，属性视图/属性搜索用）：全空
- 表现：通过 API 读属性正常，但属性视图和属性搜索找不到该文档

**引用索引**：最复杂的不一致源
- 全量重建场景：refs 有数据但 blocks/spans 无对应块 → 反向链接能列出但跳转可能异常
- upsert 后刷引用场景：refs 被补回但 attributes/spans 为空 → 引用跳转正常但相关属性/标签搜索缺失
- `update_refs` 完全绕过索引忽略，是「索引忽略语义不完整」的核心体现

### 4.5 标签索引的两级结构

标签在数据库中存在**两级索引**，分别服务于不同查询场景：

#### 一级：blocks 表 tag 列（块级聚合）
- 内容格式：`#工作# #重要#`（空格分隔的完整标签串）
- 作用：全文搜索、快速按标签过滤块、FTS 索引
- 来源：`tagFromNode()` 函数聚合

#### 二级：spans 表 type=tag（行级粒度）
- 每行对应一个标签 TextMark 节点
- 关键字段：`block_id`、`root_id`、`box`、`path`、`content`（标签名）、`type`（含 tag 标识）
- 查询函数：`QueryTagSpansByLabel`、`QueryTagSpansByKeyword`、`QueryTagSpans`
- 作用：标签面板构建、精确标签搜索、标签统计计数
- **更新方式**：随 spans 表整文档全量重建

### 4.6 索引更新触发链路

以属性修改为例，完整链路如下：

```
setBlockAttrs API
  → model.SetBlockAttrs()
    → LoadTreeByBlockID()       [加载文档树]
    → setNodeAttrs(node, tree)  [修改 IAL]
      → setNodeAttrs0()         [底层属性更新 + tags 规范化]
        → ReloadTag()           [tags 变化时触发]
      → indexWriteTreeUpsertQueue(tree)
        → treenode.UpsertBlockTree(tree)   [更新内存 BlockTree 索引]
        → filesys.WriteTree(tree)          [写 .sy 文件持久化]
        → sql.UpsertTreeQueue(tree)        [加入 SQL 异步队列]
      → cache.PutBlockIAL(id, attrs)       [更新缓存]
      → pushBlockAttrs(oldAttrs, node)     [通过 transactions 事件广播到前端]
      → sql.FlushQueue()                   [异步落库：
                                              blocks 表 Hash 增量更新
                                              spans/assets/attributes/refs 全量重建]
      → refreshDynamicRefText()            [刷新动态引用锚文本]
```

---

## 5. 界面展示与交互逻辑

### 5.1 标签面板（Tag Dock Panel）

前端 `app/src/layout/dock/Tag.ts` 实现标签树的完整展示：

#### 初始化与数据加载
1. 构造函数创建 Tree 组件，配置 `click`、`rightClick` 回调
2. `update(ignoreMaxListHint)` 调用 `/api/tag/getTag` 拉取标签树数据
3. 维护 `openNodes` 状态，数据更新后恢复展开节点

#### 实时刷新机制（事件驱动）

标签面板订阅 WebSocket 消息，在以下事件触发时自动刷新：

| 事件 cmd | 触发条件 | 判断逻辑 |
|----------|---------|---------|
| `transactions` | 任何块操作 | 检查 doOperations 中 `update/insert` 的 data 是否含 `data-type="tag"`，或 `delete` 操作 |
| `closeBox` / `removeBox` | 笔记本关闭/移除 | 直接刷新 |
| `removeDoc` | 删除文档 | 直接刷新 |
| `mount` | 挂载事件（code≠1） | 直接刷新 |

#### 排序模式支持（6 种）

前端排序菜单与后端 `Conf.Tag.Sort` 同步，对应 `kernel/model/tag.go` 中的 `sortTags`：

| 排序值 | 模式 | 实现 |
|--------|------|------|
| 0 | 名称升序（拼音） | `util.PinYinCompare` |
| 1 | 名称降序（拼音） | `util.PinYinCompare` 反向 |
| 4 | 自然排序升序 | `util.NaturalCompare` |
| 5 | 自然排序降序 | `util.NaturalCompare` 反向 |
| 7 | 引用计数升序 | `Count` 字段比较 |
| 8 | 引用计数降序 | `Count` 字段比较 |

### 5.2 标签右键菜单

`app/src/menus/tag.ts` 的 `openTagMenu` 提供两项操作：
- **重命名标签**：`renameTag()` → `/api/tag/renameTag`
- **删除标签**：`confirmDialog` 二次确认 → `/api/tag/removeTag`

---

## 6. 搜索过滤机制

### 6.1 标签搜索的两条路径

#### 路径一：标签面板内关键词过滤（SearchTags）
- 入口：标签树顶部搜索框
- 实现：`kernel/model/tag.go` 的 `SearchTags` → `labelBlocksByKeyword`
- SQL 逻辑：`kernel/sql/span.go` 的 `QueryTagSpansByKeyword`
  - 空格分隔的多关键词使用 `LIKE ? AND LIKE ?` 连接（全部命中）
  - `GROUP BY markdown` 去重
  - `LIMIT` 控制结果数量（`Conf.Search.Limit`）
- 高亮：`search.MarkText()` 对标签名做关键词高亮

#### 路径二：全局搜索中的标签语法 `#tag#`
- 入口：全局搜索面板输入 `#工作#`
- 底层：利用 FTS 索引（`blocks_fts` / `blocks_fts_case_insensitive`）的 tag 列
- 替换逻辑：在搜索替换流程中（`kernel/model/search.go`），标签 TextMark 会被转换为纯文本再做替换

### 6.2 标签点击跳转

标签面板中点击标签项触发 `openGlobalSearch`，传入 `#${label}#` 格式查询，`method=0`（搜索方法）。

---

## 7. 批量修改与一致性保障

### 7.1 标签批量重命名（RenameTag）

`kernel/model/tag.go` 的 `RenameTag` 实现完整的批量标签重命名流程：

#### 步骤详解

1. **合法性检查**：
   - `treenode.ContainsMarker(newLabel)` 检查非法字符
   - 去除首尾 `/` 和空白，空值报错
   - 新旧相同直接返回

2. **定位受影响块**：
   ```go
   tags := sql.QueryTagSpansByLabel(oldLabel)  // 从 spans 表查所有标签位置
   treeBlocks := map[string][]string{}         // 按 root_id 分组：文档ID → [块ID列表]
   ```

3. **逐文档处理**（遍历 treeBlocks）：
   - `LoadTreeByBlockIDWithReindex(treeID)` 加载文档树
   - `generateTreeHistory(tree, historyDir)` **写入历史快照**（可回滚）
   - 遍历块列表：
     - **文档节点**：处理 IAL 中 `tags` 属性，替换 `oldLabel` 前缀匹配（含子标签）
     - **普通块**：遍历 `NodeTextMark` 类型子节点，`type="tag"` 的替换 `TextMarkTextContent`
   - `writeTreeUpsertQueue(tree)` 写文件 + 入索引队列
   - `util.RandomSleep(50, 150)` 防阻塞，给 UI 喘息时间

4. **收尾工作**：
   - `indexHistoryDir()` 索引历史目录
   - `sql.FlushQueue()` 强制落库
   - `ReloadProtyle(id)` 刷新所有受影响编辑器
   - `updateAttributeViewBlockText(updateNodes)` 同步属性视图中的块文本

5. **前缀匹配替换规则**：`strings.HasPrefix(content, oldLabel+"/") || content == oldLabel`，确保 `a/b` 重命名为 `c/d` 时，`a/b/e` 也会被同步替换为 `c/d/e`

### 7.2 标签批量删除（RemoveTag）

`kernel/model/tag.go` 的 `RemoveTag` 与重命名流程高度相似，差异在于：
- 文档节点：从 tags 逗号分隔字符串中移除匹配项
- 普通块：`n.Unlink()` 直接移除 TextMark 节点（而非修改内容）

### 7.3 批量属性设置（BatchSetBlockAttrs）

`kernel/model/blockial.go` 的 `BatchSetBlockAttrs` 实现跨块属性批量设置：

1. **批量加载文档树**：`filesys.LoadTrees(blockIDs)` 一次加载所有涉及的文档
2. **节点定位与属性设置**：逐块调用 `setNodeAttrs0`
3. **缓存更新**：`cache.PutBlockIAL` 同步
4. **事件推送**：`pushBlockAttrs` 广播属性变化
5. **统一索引入队**：所有涉及的 tree 入队 `indexWriteTreeUpsertQueue`
6. **同步标记**：`IncSync()` 触发云同步

> **注意**：BatchSetBlockAttrs **不做动态锚文本刷新**（代码注释 `// 不做锚文本刷新`），与单块 SetBlockAttrs 的区别。

### 7.4 事务处理与一致性保障

#### 事务队列（txQueue）架构

`kernel/model/transaction.go` 中定义了核心事务机制：

```
前端 transactions 消息
  → PerformTransactions()
    → 每个 tx 加锁后入 chan txQueue (buffer=7)
    → flushQueue() 单协程消费，逐个 flushTx
      → flushLock 互斥锁（全局串行执行）
      → performTx(tx)
        → tx.begin()       [DB 事务开启]
        → processLargeInsert / processLargeDelete  [批量优化]
        → 逐个执行 doOperations (create/update/insert/delete/setAttrs/...)
        → tx.commit()      [DB 事务提交]
```

#### 错误处理分类（TxErrCode）

| 错误码 | 含义 | 处理方式 |
|--------|------|---------|
| 0 | 块未找到 | 推送错误消息，不崩溃 |
| 1 | 数据同步中 | 提示语言包 222，静默 |
| 2 | 写树失败 | 致命错误 → `logging.LogFatalf` 退出 |
| 3 | 属性视图处理失败 | 提示 + 错误日志 |
| 4 | 通用推送消息 | 推送自定义消息 |

#### 大操作优化

- **LargeInsert**：超过 32 个连续 insert 操作，走 `doLargeInsert` 合并路径，减少重复 tree 加载
- **LargeDelete**：超过 32 个连续 delete 操作，走 `doLargeDelete` 合并路径

---

## 8. 冲突解决机制

### 8.1 队列级冲突：同文档操作覆盖

SQL 索引队列（`kernel/sql/queue.go`）中，**同一 tree.ID 的同类操作会被新操作覆盖**，这是一种「最终写入者胜」（Last Write Wins）的冲突消解策略：
- 避免了短时间内对同一文档多次重复索引
- 缺点是如果两次 upsert 之间有查询，可能读到中间状态（但由于数据库事务串行，最终一致）

### 8.2 属性级冲突：大小写归一化

`setNodeAttrs0` 中属性名强制转为小写存储，从根源上避免了 `Custom-A` 与 `custom-a` 的命名冲突。删除操作同时支持「精确大小写删除」和「小写删除」两种模式，处理历史遗留数据。

### 8.3 并发冲突：全局事务串行化

`flushTx` 中的 `flushLock.Lock()` 确保所有写事务**全局串行执行**，从架构层面消除了并发写冲突。代价是吞吐量受限，但对于桌面端单用户场景完全足够。

### 8.4 编辑冲突：前端状态不一致兜底

在 `doInsert` / `doPrependInsert` 等操作中，若 `treenode.GetBlockTree` 返回 nil（说明前后端状态不一致），直接调用 `util.ReloadUI()` 强制重载整个界面，以状态重置的方式解决冲突。

---

## 9. 扩展字段（custom-*）的处理

### 9.1 扩展字段的生命周期

| 阶段 | 处理逻辑 | 位置 |
|------|---------|------|
| **写入验证** | 验证 `custom-` 前缀后首字符为小写字母，整体长度 > 7 | `isValidAttrName` |
| **存储格式** | IAL 中转义字符串 + 逐行 attributes 表记录 | `setNodeAttrs0` + `buildAttributeFromNode` |
| **索引判断** | `isAttr(name)` 返回 true → 写入 attributes 表 | `strings.HasPrefix(name, "custom-")` |
| **类型标记** | attributes.type: `"b"`（块级）/ `"s"`（行级 span 级） | `buildAttributeFromNode` |
| **缓存同步** | 属性修改后同步写入 `blockIALCache` | `cache.PutBlockIAL` |
| **搜索索引** | 随 blocks.ial / blocks_fts 全文可搜 | `upsertTree`（blocks 增量 + attributes 全量） |

### 9.2 扩展字段在属性视图（Attribute View）中的深度集成

`kernel/model/attribute_view*.go` 系列文件实现了 custom 属性与数据库视图的深度绑定：
- 块可通过 `custom-*` 属性被绑定到属性视图行
- 属性值变化通过 `updateAttributeViewBlockText` 同步到视图显示
- 删除块时 `syncDelete2AvBlock` 同步解除属性视图绑定
- 插入/移动块时 `upsertAvBlockRel` 重建关联关系

---

## 10. 流程梳理总结图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           用户操作（前端）                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐  ┌────────────────┐  │
│  │ 标签面板点击  │   │ 标签右键菜单  │   │ 属性面板编辑 │  │ 搜索框 #tag#   │  │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘  └───────┬────────┘  │
└─────────┼──────────────────┼──────────────────┼───────────────────┼────────────┘
          │                  │                  │                   │
          ▼                  ▼                  ▼                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           API 层（kernel/api）                                 │
│  /api/tag/*          /api/tag/renameTag      /api/attr/setBlockAttrs    search│
│  /api/tag/removeTag   /api/tag/getTag        /api/attr/batchSetBlockAttrs     │
└─────────┬──────────────────┬──────────────────┬───────────────────┬────────────┘
          │                  │                  │                   │
          ▼                  ▼                  ▼                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        业务模型层（kernel/model）                               │
│  ┌─────────────────────────────────────────────────────────────────────┐      │
│  │ 标签：BuildTags / RenameTag / RemoveTag / SearchTags                │      │
│  │ 属性：SetBlockAttrs / BatchSetBlockAttrs / ResetBlockAttrs          │      │
│  │ 核心：setNodeAttrs0（属性验证+tags特殊处理+大小写归一化）             │      │
│  │ 事务：PerformTransactions → flushTx（全局串行化+错误分类处理）       │      │
│  └──────────────────────────────┬──────────────────────────────────────┘      │
│                                 │                                              │
│         ┌───────────────────────┼───────────────────────┐                      │
│         ▼                       ▼                       ▼                      │
│  ┌──────────────┐      ┌────────────────┐      ┌──────────────────┐           │
│  │ BlockTree 索引│      │ Cache (Ristretto) │      │ .sy 文件持久化    │           │
│  │ 内存块树映射  │      │ blockIALCache   │      │ filesys.WriteTree│           │
│  └──────────────┘      │ docIALCache     │      └────────┬─────────┘           │
│                         └────────────────┘               │                      │
└──────────────────────────────────────────────────────────┼──────────────────────┘
                                                           │
                                                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                       SQL 索引层（kernel/sql）                                 │
│  ┌──────────────────────────────────────────────────────────────────────┐     │
│  │ dbQueueOperation 异步队列：                                           │     │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌──────┐ ┌────────┐                         │     │
│  │  │index│ │upsert│ │delete│ │rename│ │update..│   (队列去重合并)       │     │
│  │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬───┘ └───┬────┘                         │     │
│  └─────┼───────┼───────┼───────┼─────────┼──────────────────────────────┘     │
│        ▼       ▼       ▼       ▼         ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐     │
│  │ FlushQueue → execOp → upsertTree                                     │     │
│  │   ├─ blocks 表：Hash 增量对比（仅变化的块先删后插）                    │     │
│  │   ├─ spans 表： 整文档全量删 + 全量插（标签/链接/图片等行内元素）       │     │
│  │   ├─ assets 表：整文档全量删 + 全量插                                  │     │
│  │   ├─ attributes 表：整文档全量删 + 全量插（custom-* + 内置属性逐行）    │     │
│  │   └─ refs / file_annotation_refs：整文档全量删 + 全量插                │     │
│  └──────────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘
                                                           │
                                                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          查询路径（反向数据流）                                  │
│  标签面板 ← BuildTags ← QueryTagSpans ← spans 表                             │
│  属性面板 ← GetBlockAttrs ← blocks.ial 解析                                  │
│  全局搜索 ← FTS MATCH ← blocks_fts / blocks_fts_case_insensitive            │
│  标签跳转 ← openGlobalSearch ← #tag# 语法                                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. 潜在风险分析

### 11.1 一致性风险

| 风险点 | 严重程度 | 说明 |
|--------|---------|------|
| **异步索引延迟** | 中 | SQL 队列异步执行，属性修改后存在短暂的「文件已写入但索引未更新」窗口，期间搜索结果可能不准确 |
| **缓存与 DB 不一致** | 低 | `PutBlockIAL` 在入队前直接更新缓存，若后续 upsert 失败则缓存脏读，但概率极低（DB 事务失败会有日志） |
| **队列溢出** | 低 | txQueue buffer=7，大量连续操作可能阻塞；dbQueueOperation 无显式长度限制，极端情况内存增长 |
| **批量操作中断** | 中 | RenameTag/RemoveTag 逐文档循环，中途中断会导致部分文档已更新、部分未更新（无全局事务回滚） |
| **索引忽略导致 upsert 数据丢失** | 高 | upsertTree 先删后插，如果 `insertTree0` 中命中 `indexignore` 规则，子表数据（spans/attributes/refs）已被删除但不会被重新插入，导致标签、属性、引用索引全部丢失 |
| **两级标签索引不一致** | 中 | 索引忽略 + upsert 场景下，blocks.tag（块级标签，FTS 用）可能有残留（未变化的块保留），但 spans 表（行级标签，标签面板用）已全空。全文搜索能搜到但标签面板计数减少 |
| **两层属性索引不一致** | 中 | 索引忽略 + upsert 场景下，blocks.ial（整体 JSON，GetBlockAttrs 用）可能有残留，但 attributes 展开表（属性视图/属性搜索用）已全空。API 能读到属性但属性搜不到 |
| **update_refs 与 upsert 竞态冗余** | 低 | 同一文档可能同时入队 `update_refs` 和 `upsert`，两者都会操作 refs 表但互不感知，存在冗余删除+插入，最终结果正确但浪费 IO |
| **索引忽略悬挂引用** | 低 | `update_refs` 不受 `indexignore` 影响，被忽略文档的 refs 仍会被更新，但该文档的 blocks/spans/attributes 已被删除，导致 refs 中存在「指向索引中不存在块」的悬挂引用 |
| **全量重建时 refs 与主索引脱节** | 中 | 全量重建场景下，被忽略文档的 blocks/spans/attributes 全无，但 refs 会被 IndexRefs 阶段通过 update_refs 补回。反向链接能列出引用但点击跳转可能找不到对应块 |

### 11.2 性能风险

| 风险点 | 严重程度 | 说明 |
|--------|---------|------|
| **大标签重命名 O(N×M)** | 高 | 标签使用量极大时，RenameTag 会遍历 `treeBlocks` 加载每棵文档树，涉及大量文件 IO + 解析 + 重写 |
| **子表全量重建开销** | 中 | `upsertTree` 中 spans/assets/attributes/refs 全部全量删除后重插。如果文档含大量标签、链接、图片或 custom 属性，子表重建开销不可忽略，即使只改了一个块 |
| **标签构建全量扫描** | 中 | `BuildTags` 每次都调用 `QueryTagSpans("")` 全表扫描 spans，标签数量巨大时可能影响面板刷新 |
| **随机休眠策略** | 低 | RenameTag/RemoveTag 中的 `RandomSleep` 在大量文档时会显著拉长操作时间，但避免了 UI 线程阻塞 |
| **spans 双删除冗余** | 低 | `insertTree0` 中 spans 插入前会再删一次（双保险），存在冗余删除操作，但单次删除性能影响可忽略 |

### 11.3 正确性风险

| 风险点 | 严重程度 | 说明 |
|--------|---------|------|
| **标签前缀误替换** | 中 | RenameTag 使用 `strings.Replace(docTag, oldLabel, newLabel, 1)`，若标签名是另一个标签名的子串（如 `a/b` 与 `aa/b`）且未用 `/` 边界严格校验，可能误替换（虽然有 `HasPrefix(oldLabel+"/")` 判断，但 `Replace` 本身不保证位置） |
| **属性大小写敏感丢失** | 中 | 强制属性名小写可能破坏用户对大小写区分的预期，尤其是通过 API 导入的外部数据 |
| **custom 属性未验证值类型** | 低 | 属性值统一为 string，由调用方自行负责 JSON/数字/日期的序列化与解析 |
| **子表重建丢失增量** | 低 | 子表全量重建意味着每次保存都要删插所有 attribute/span 记录，理论上存在事务失败导致索引残缺的风险，但 SQLite 事务原子性保证了要么全成要么全败 |

### 11.4 安全风险

| 风险点 | 严重程度 | 说明 |
|--------|---------|------|
| **属性值 XSS** | 低 | 属性值写入时做了 `html.EscapeAttrVal`，但如果有渲染路径绕过转义直接输出则存在风险 |
| **空值删除语义歧义** | 低 | API 中 `null` 和 `""` 都会触发删除，但两者语义不同（前者显式删除，后者可能是用户误输入空串） |

---

## 12. 后续验证事项

### 12.1 功能验证清单

| 编号 | 验证项 | 验证步骤 | 预期结果 |
|------|--------|---------|---------|
| V-01 | 标签重命名前缀准确性 | 创建标签 `test` 和 `test1`，各绑定若干块，重命名 `test` → `demo` | `test1` 不应被修改，`test` 及其子标签正确替换 |
| V-02 | 文档 tags IAL 去重 | 通过 API 设 `tags="a,b,a,c,b"` | 实际存储为 `tags="a,b,c"` |
| V-03 | 属性大小写归一化 | 设 `Custom-Name` 和 `custom-name` 两个属性 | 最终只保留一个小写版本，值为最后一次设置 |
| V-04 | 批量属性设置事务性 | 100 个跨文档 blockAttrs，其中第 50 个 blockID 不存在 | 前 49 个应成功还是全部回滚？需确认设计语义 |
| V-05 | 删除带属性的块 | 给块设置多个 custom 属性后删除该块 | attributes 表对应行被同步删除，无残留 |
| V-06 | 标签面板增量刷新 | 编辑文档添加 `#新标签#`，观察标签面板 | 面板自动刷新，新标签出现且计数正确 |
| V-07 | 重命名后属性视图同步 | 属性视图使用某 custom 属性过滤，重命名属性关联块的标签后 | 属性视图中的标签文本/块内容同步更新 |
| V-08 | upsert 增量正确性 | 文档有 100 个块，仅修改 1 个块的内容后保存 | blocks 表仅变化 1 块的 Hash，spans/assets/attributes 全部重建 |
| V-09 | 标签嵌套计数准确性 | 标签 `a/b/c` 有 2 个块，`a/b` 有 1 个块，`a` 有 1 个块 | 标签树中：a.Count=4，a/b.Count=3，a/b/c.Count=2 |
| V-10 | 只读模式属性保护 | 以只读角色登录，调用 setBlockAttrs API | 被拒绝，无任何修改 |
| V-11 | update_refs 隔离性 | 对文档 A 触发 `update_refs`，检查 blocks/spans/attributes 表 | blocks/spans/attributes 不受影响，仅 refs 和 file_annotation_refs 被重建 |
| V-12 | 索引忽略 + upsert 数据完整性 | 先索引文档 A，再在 `indexignore` 中添加 A 的路径，然后编辑保存 A | A 的 spans/attributes/refs 被清空，标签面板和属性搜索不再出现 A 的内容 |
| V-13 | 索引忽略悬挂引用 | 文档 B 引用文档 A，A 被 `indexignore` 忽略，对 B 触发 `update_refs` | refs 表中仍存在 B→A 的引用记录，但 A 的 blocks 表中无对应行（悬挂引用） |
| V-14 | 全量重建 + 索引忽略 refs 残留 | 配置 `indexignore` 忽略含引用的文档 A → 执行 FullReindex | blocks/spans/attributes 中无 A 的数据，但 refs 表中有 A 的引用记录（IndexRefs 阶段补回） |
| V-15 | 标签两级索引不一致验证 | 文档 A 先索引，加忽略规则后修改部分块（部分块 Hash 不变） | blocks.tag 中仍有未变化块的标签数据，但 spans 表行级标签全空；全文搜索能搜到，标签面板计数减少 |
| V-16 | 属性两层索引不一致验证 | 文档 A 设 custom 属性后加忽略规则，修改部分块后保存 | API `getBlockAttrs` 仍能读到属性（从 blocks.ial），但属性视图/属性搜索搜不到（attributes 表已空） |
| V-17 | 忽略 + upsert + 刷引用组合效应 | 文档 A 被忽略，编辑保存后立即刷反向链接 | blocks 部分保留 + spans/assets/attributes 全空 + refs 被补回，三态不一致 |

### 12.2 性能与压力测试

| 编号 | 测试项 | 测试方法 | 基线指标 |
|------|--------|---------|---------|
| P-01 | 大标签重命名 | 创建 10000 个文档，每文档含 `#largeTag#`，执行重命名 | 总耗时 < 60s，内存峰值 < 1GB |
| P-02 | 属性高频写入 | 1000 次/s 频率调用 setBlockAttrs 修改同一块属性 | 队列无堆积，最终值一致，无内存泄漏 |
| P-03 | 标签树构建性能 | spans 表 10 万行 tag 记录，BuildTags 单次调用 | 耗时 < 500ms |
| P-04 | 跨文档批量属性设置 | 1000 个不同文档的各 1 个块，BatchSetBlockAttrs | 耗时 < 10s，所有 blocks/attributes 表正确更新 |
| P-05 | 大文档子表重建开销 | 文档含 1000 个 spans、500 个 attributes，仅修改 1 个块后保存 | 测试子表全量重建的耗时占比 |
| P-06 | 重启后索引完整性 | 批量修改过程中强制 kill 进程，重启后重建索引 | 索引一致性校验通过，无 orphan 行 |

### 12.3 异常与边界测试

| 编号 | 测试项 | 输入 | 预期 |
|------|--------|------|------|
| E-01 | 重名为空字符串 | newLabel = "" / "   " | 返回语言包错误 |
| E-02 | 重名包含非法字符 | newLabel 含 `#`, `\n` 等 marker 字符 | 被 ContainsMarker 拦截 |
| E-03 | 属性名非法格式 | 如 `123Custom`、`custom-`、`Custom-A` | 返回错误，不写入 |
| E-04 | 不存在的块 ID | setBlockAttrs 传入随机 ID | 返回 "block not found" 错误，事务无副作用 |
| E-05 | 递归标签路径 | `a/a/a/.../a`（深度 50） | 正常显示，渲染无栈溢出 |
| E-06 | 属性值超长 | custom 属性值 100KB 字符串 | 正常存储、搜索不崩溃 |
| E-07 | 删除不存在的标签 | removeTag 传入从未使用过的标签名 | 静默成功，无副作用 |
| E-08 | upsert 事务中断 | 在 upsertTree 执行到一半时模拟崩溃 | 重启后数据完整，无半写状态 |
| E-09 | indexignore 配置热更新 | 运行中修改 `indexignore` 文件添加/删除规则 | 需执行 `FullReindex` 才能生效（`IndexIgnoreCached` 机制） |
| E-10 | update_refs 对被忽略文档 | 文档 A 在 `indexignore` 中，其他文档引用了 A 的块，触发引用刷新 | refs 表仍被更新（`upsertRefs` 无忽略检查），但 A 的 blocks 无对应行 |

---

## 13. 关键文件索引

| 文件 | 核心内容 | 关键行（约） |
|------|---------|-------------|
| `kernel/model/tag.go` | 标签业务逻辑核心：RenameTag、RemoveTag、BuildTags、SearchTags | L35-L432 |
| `kernel/model/blockial.go` | 属性读写：SetBlockAttrs、BatchSetBlockAttrs、setNodeAttrs0、isValidAttrName | L38-L372 |
| `kernel/model/transaction.go` | 事务处理：flushTx、performTx、doLargeInsert、doSetAttrs | L57-L1709 |
| `kernel/model/index.go` | 全量/增量索引调度、Box.Index、嵌入块索引 | L49-L442 |
| `kernel/model/file.go` | writeTreeUpsertQueue、indexWriteTreeUpsertQueue 关键链路函数 | L944-L994 |
| `kernel/sql/queue.go` | SQL 异步队列、操作去重合并、FlushQueue | L37-L437 |
| `kernel/sql/upsert.go` | upsertTree 核心实现、blocks Hash 增量、子表全量重建、索引忽略检查 | L399-L541 |
| `kernel/sql/block_ref.go` | upsertRefs/deleteRefs/insertRefs：引用索引独立刷新逻辑 | L40-L70 |
| `kernel/sql/span.go` | 标签 Span 查询：QueryTagSpans* 系列函数 | L40-L173 |
| `kernel/sql/database.go` | fromTree、tagFromNode、buildAttributeFromNode、isAttr、各表删除函数 | L520-L615, L968-L1140 |
| `kernel/sql/attribute.go` | Attribute 结构体定义 | L19-L28 |
| `kernel/cache/ial.go` | Ristretto 缓存实现：blockIALCache、docIALCache | L25-L79 |
| `kernel/api/tag.go` | 标签 HTTP 接口：getTag/renameTag/removeTag | L28-L110 |
| `kernel/api/attr.go` | 属性 HTTP 接口：setBlockAttrs/batchSetBlockAttrs 等 | L31-L196 |
| `kernel/model/backlink.go` | 反向链接刷新：refreshRefsByDefID | L41-L62 |
| `kernel/model/push_reload.go` | 动态锚文本级联更新：refreshDynamicRefTexts0 | L260-L358 |
| `app/src/layout/dock/Tag.ts` | 前端标签面板：Tree 渲染、事件订阅、排序、刷新 | L15-L204 |
| `app/src/menus/tag.ts` | 标签右键菜单：重命名、删除 | L10-L42 |

---

**文档生成时间**：2026-06-15
**分析范围**：SiYuan kernel（Go）+ app（TypeScript）核心源码
**修正记录**：
- v2 - 修正 upsertTree 增量更新理解：仅 blocks 表做 Hash 增量，spans/assets/attributes/refs 均为整文档全量重建
- v3 - 修正 update_refs 理解：仅重建 refs+file_annotation_refs，不涉及其他表；补充索引忽略配置对 upsert 的副作用分析（先删后忽略=数据丢失）、update_refs 与 upsert 的竞态分析
- v4 - 新增索引忽略三种生命周期路径对比：全量重建、后加忽略+upsert、仅触发引用刷新；明确标签两级索引不一致、属性两层索引不一致、refs 与主索引脱节等具体一致性风险
