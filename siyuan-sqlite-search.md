# SiYuan SQLite 索引与全文检索机制分析

## 1. 架构概览

SiYuan（思源笔记）采用 **SQLite + FTS5 全文搜索引擎** 作为核心数据检索引擎，配合 **异步队列 + 定时刷盘** 的更新策略，实现笔记内容的高性能存储与检索。系统包含三个独立的 SQLite 数据库实例，通过 `go-sqlite3` 驱动的 `sqlite3_extended` 自定义注册（注入 `regexp` 函数）与底层交互。

### 1.1 三大数据库实例

| 数据库 | 路径变量 | 核心用途 | 连接池 |
|--------|---------|---------|--------|
| 主索引库 | `util.DBPath` | 块索引、FTS全文索引、引用关系、资源元数据 | MaxOpen=20 |
| 历史库 | `util.HistoryDBPath` | 历史版本全文检索 | MaxOpen=3 |
| 资源内容库 | `util.AssetContentDBPath` | PDF/Office 等附件内容全文索引 | MaxOpen=3 |

### 1.2 模块关系图

```
┌─────────────────────────────────────────────────────────────┐
│                      HTTP API 层                            │
│  api/search.go ── fullTextSearchBlock / searchRefBlock 等    │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    业务模型层 (model/)                       │
│  search.go: FullTextSearchBlock / FindReplace / IndexRefs   │
│  index.go:  indexBox / UpsertIndexes / IndexEmbedBlockJob   │
│  block.go:  writeTreeUpsertQueue (树写入→入队)              │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                   数据库操作层 (sql/)                        │
│  queue.go:     操作队列 + FlushQueue (3s定时刷盘)           │
│  upsert.go:    indexTree / upsertTree (hash增量对比)        │
│  block_query.go: SelectBlocksRawStmt / GetBlock + 缓存层    │
│  database.go:  FTS5虚拟表 + BTree索引 + 连接配置            │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    SQLite 引擎                               │
│  blocks / blocks_fts[_case_insensitive] / spans / assets    │
│  refs / file_annotation_refs / attributes + 触发器与索引    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 表结构与索引设计

### 2.1 核心实体表与 B-Tree 索引

主数据库 `blocks` 表包含 21 个字段，覆盖块的完整元数据与内容：

```sql
CREATE TABLE blocks (
  id, parent_id, root_id, hash, box, path, hpath,
  name, alias, memo, tag, content, fcontent, markdown,
  length, type, subtype, ial, sort, created, updated
);
```

共 21 个字段，其中 `type` 存储块类型的缩写（如 `d`=文档、`h`=标题、`p`=段落），`sort` 存储块类型排序码（由 `nSort()` 函数生成，见下表），`fcontent` 存储容器块的首叶子块内容（用于列表项搜索）。

**`sort` 字段值（`nSort()` 函数）**：

| sort 值 | 块类型 | type 缩写 |
|---------|--------|----------|
| 0 | Document 文档 | `d` |
| 5 | Heading 标题 | `h` |
| 10 | Paragraph / CodeBlock / MathBlock / Table / HTMLBlock | `p`/`c`/`m`/`t`/`html` |
| 20 | List / ListItem / Blockquote / Callout | `l`/`i`/`b`/`callout` |
| 30 | SuperBlock / AttributeView | `s`/`av` |
| 100 | 其他未列出的块类型 | — |
| 200 | Text / TextMark（非标签） | `text`/`textmark` |
| 205 | TextMark 标签 | `textmark`(tag) |

此排序码在所有 ORDER BY 子句中作为 CASE 之后的二级排序依据，确保同权重的块按类型优先级排列（文档 > 标题 > 段落 > 列表 > 超级块 > 其他）。

**B-Tree 索引配置**：

| 索引名 | 字段 | 用途 |
|-------|------|------|
| `idx_blocks_id` | `id` | 按块 ID 精确查找 |
| `idx_blocks_parent_id` | `parent_id` | 父子块层级遍历 |
| `idx_blocks_root_id` | `root_id` | 按文档根批量操作 |
| `idx_blocks_root_id_id_hash` | `(root_id, id, hash)` | **复合覆盖索引**，用于 upsert 时的 hash 增量对比 |

其余辅助表的索引：
- `spans` → `idx_spans_root_id`
- `assets` → `idx_assets_root_id`
- `attributes` → `idx_attributes_block_id` + `idx_attributes_root_id`

### 2.2 FTS5 全文索引虚拟表

系统采用 **双 FTS5 表切换** 方案支持大小写敏感配置：

```sql
-- 大小写敏感
CREATE VIRTUAL TABLE blocks_fts USING fts5(
  id UNINDEXED, parent_id UNINDEXED, root_id UNINDEXED, hash UNINDEXED,
  box UNINDEXED, path UNINDEXED, hpath UNINDEXED,
  name, alias, memo, tag, content, fcontent,
  markdown UNINDEXED, length UNINDEXED, type UNINDEXED, subtype UNINDEXED,
  ial, sort UNINDEXED, created UNINDEXED, updated UNINDEXED,
  tokenize="siyuan"
);

-- 大小写不敏感（默认）
CREATE VIRTUAL TABLE blocks_fts_case_insensitive USING fts5(
  ...同上...,
  tokenize="siyuan case_insensitive"
);
```

**设计要点**：
- 使用 SiYuan 自研的 `siyuan` tokenizer 分词器（支持中文切分）
- 总列数 21 列，其中 **14 个 UNINDEXED 列** + **7 个可索引列**

**UNINDEXED 列（14 个，不参与倒排索引，仅存储原值用于返回）**：

| 列号 | 列名 | 说明 |
|------|------|------|
| 0 | `id` | 块 ID |
| 1 | `parent_id` | 父块 ID |
| 2 | `root_id` | 文档根 ID |
| 3 | `hash` | 内容哈希 |
| 4 | `box` | 笔记本 ID |
| 5 | `path` | 文档路径 |
| 6 | `hpath` | 可读路径（人类友好路径） |
| 13 | `markdown` | Markdown 原文 |
| 14 | `length` | 内容长度 |
| 15 | `type` | 块类型缩写（d/h/l/i/c/m/...） |
| 16 | `subtype` | 块子类型（h1-h6 / u/o/t 列表类型） |
| 18 | `sort` | 块排序码（nSort 函数生成） |
| 19 | `created` | 创建时间 |
| 20 | `updated` | 更新时间 |

**可索引列（7 个，构建倒排索引，参与 MATCH 和 snippet）**：

| 列号 | 列名 | 默认参与搜索 | 说明 |
|------|------|------------|------|
| 7 | `name` | 是（`Conf.Search.Name`） | 块命名属性 |
| 8 | `alias` | 是（`Conf.Search.Alias`） | 块别名 |
| 9 | `memo` | 是（`Conf.Search.Memo`） | 备注 |
| 10 | `tag` | 始终参与 | 标签 |
| 11 | `content` | 始终参与 | 块纯文本内容 |
| 12 | `fcontent` | **否** | 容器块的首叶子块内容（用于列表项排序，不参与 MATCH） |
| 17 | `ial` | 否（`Conf.Search.IAL` 默认关） | IAL 属性列表字符串 |

> **注意**：7 个可索引列中，`columnFilter()` 实际只返回最多 6 个用于 MATCH 查询（content + 可选的 name/alias/memo/ial + tag），`fcontent` 虽然是可索引列但**不参与搜索**，仅在引用搜索排序的 CASE 中用 `fcontent LIKE` 匹配（这是 SQL 层的 LIKE，不是 FTS5 MATCH，不走倒排）。

- 搜索时根据 `Conf.Search.CaseSensitive` 动态选择 `blocks_fts` 或 `blocks_fts_case_insensitive`

**历史库与资源库 FTS5 表**：
- `histories_fts_case_insensitive`：仅大小写不敏感模式
- `asset_contents_fts_case_insensitive`：附件内容索引，列含 `name, ext, path, content`

### 2.3 SQLite 连接关键参数

```
_journal_mode=WAL          -- 写前日志，支持高并发读
_synchronous=OFF           -- 异步刷盘，极致性能（有丢数据风险）
_mmap_size=2684354560      -- 2.5GB 内存映射
_secure_delete=OFF         -- 禁用安全删除，加速写入
_cache_size=-20480         -- 20MB 页缓存
_page_size=32768           -- 32KB 页大小
_busy_timeout=7000         -- 锁等待超时 7s
_temp_store=MEMORY         -- 临时表放内存
_case_sensitive_like=OFF   -- LIKE 不区分大小写
```

---

## 3. 内容写入后的索引过程

### 3.1 完整写入流程

```
用户编辑保存
    │
    ▼
model/tree.go: writeTreeUpsertQueue(tree)
    │
    ▼
treenode.IndexBlockTree(tree)        ─── 更新 BlockTree 内存结构
    │
    ▼
sql.UpsertTreeQueue(tree)            ─── 入队（相同 root_id 覆盖旧操作）
    │
    ▼
定时任务 Cron (every 3s)             ─── util.SQLFlushInterval = 3000ms
    │
    ▼
sql.FlushTxJob() → FlushQueue()
    │
    ├─ getOperations()               ─── 原子取出整批操作
    │
    ├─ 遍历 ops: 每 op 独立事务 beginTx()
    │     │
    │     ▼
    │   execOp("upsert") → upsertTree()
    │     │
    │     ├─ queryBlockHashes(root_id)    ─── 查旧块哈希（用复合索引）
    │     ├─ fromTree() → 构造新 Block[]
    │     ├─ hash 对比：unchanges / toRemoves
    │     │
    │     ├─ deleteBlocksByIDs(toRemoves)
    │     │   └─ 同步删除 blocks + FTS 表（按 ROWID）
    │     │
    │     ├─ deleteSpans/Assets/Attributes/Refs
    │     │
    │     └─ insertTree0()
    │         ├─ insertBlocks() 512条/批
    │         │   ├─ blocks 表 (B-Tree)
    │         │   ├─ blocks_fts 或 blocks_fts_case_insensitive (FTS5)
    │         │   └─ putBlockCache() → ristretto 缓存
    │         ├─ insertBlockRefs() + insertFileAnnotationRefs()
    │         ├─ insertSpans() / insertAssets()
    │         └─ insertAttributes()
    │
    └─ commitTx() 每 op 独立提交
         │
         ▼
    eventbus.EvtSQLIndexFlushed → Conf.DataIndexState = 0
```

### 3.2 操作队列（Operation Queue）设计

**数据结构**：
```go
type dbQueueOperation struct {
    inQueueTime  time.Time
    action       string        // 13种操作类型
    indexTree    *parse.Tree   // index/rename/move
    upsertTree   *parse.Tree   // upsert/update_refs
    // ... 各操作专用字段
}
```

**操作类型（13 种）**：

| action | 入队函数 | 去重键 |
|--------|---------|--------|
| `index` | `IndexTreeQueue()` | `indexTree.ID` |
| `upsert` | `UpsertTreeQueue()` | `upsertTree.ID` |
| `delete` | `RemoveTreePathQueue()` | `(box, pathPrefix)` |
| `delete_id` | `RemoveTreeQueue()` | `removeTreeID` |
| `delete_ids` | `BatchRemoveTreeQueue()` | 无（批量） |
| `rename` | `RenameTreeQueue()` | `indexTree.ID` |
| `move` | `MoveTreeQueue()` | `indexTree.ID` |
| `delete_box` | `DeleteBoxQueue()` | `box` |
| `delete_box_refs` | `DeleteBoxRefsQueue()` | `box` |
| `update_refs` | `UpdateRefsTreeQueue()` | `upsertTree.ID` |
| `delete_refs` | `DeleteRefsTreeQueue()` | `upsertTree.ID` |
| `update_block_content` | `UpdateBlockContentQueue()` | `block.ID` |
| `delete_assets` | `BatchRemoveAssetsQueue()` | 无 |
| `index_node` | `IndexNodeQueue()` | `id` |

**关键去重机制**：入队前遍历队列，若存在相同 key 的同类型操作则直接覆盖，避免短时间内对同一文档反复写入。例如连续编辑同一文档时，队列中始终只保留最新的 `upsertTree`。

### 3.3 Hash 增量更新算法

核心思想：**仅重写有变更的块**。

```go
// 1. 查询旧块哈希（利用复合索引 idx_blocks_root_id_id_hash）
oldBlockHashes := queryBlockHashes(tx, tree.ID)  // map[blockID]hash

// 2. 从 AST 树生成新块
blocks, spans, assets, attributes := fromTree(tree.Root, tree)

// 3. 对比
for id, oldHash := range oldBlockHashes {
    if newHash, exists := newBlockHashes[id]; exists {
        if newHash == oldHash {
            unChanges.Add(id)   // 完全不变，跳过
        }
    } else {
        toRemoves = append(toRemoves, id)  // 旧块已被删除
    }
}

// 4. 过滤不变块，变更块也先删后插
blocks = filter(blocks, !unChanges.Contains(id))
toRemoves = append(toRemoves, changedBlockIDs...)

// 5. 同步 FTS 删除（按 ROWID 批量）
deleteBlocksByIDs(tx, toRemoves)   // blocks + FTS 双表删除
```

**Hash 计算**：`treenode.NodeHash(n, tree, luteEngine)` 对块的结构+内容生成指纹，粒度精确到单个块。

### 3.4 索引时机分类

| 时机 | 触发方式 | 延迟 | 说明 |
|------|---------|------|------|
| 文档保存 | `writeTreeUpsertQueue()` | 0~3000ms | 正常编辑路径，最常见 |
| 启动重建 | `indexBox()` 并发池 | 同步阻塞 | 笔记本首次打开/数据损坏恢复 |
| 引用解析 | `IndexRefs()` 二次遍历 | 启动后+队列 | 扫描含 `TextMarkBlockRefID` 的文档 |
| 嵌入块索引 | `IndexEmbedBlockJob()` | 每 10 分钟 | 查询空内容嵌入块，执行 SQL 回填 content（最多64个/次） |
| 笔记本操作 | `(box *Box) Index()` | 任务队列 | 打开/关闭笔记本 |
| 重命名/移动 | `RenameTreeQueue()` | 0~3000ms | 仅更新 path/hpath 字段，不重建内容 |
| OCR 索引 | `FlushAssetsTextsJob()` | 每 30 秒 | 图片 OCR 文本入索引 |

---

## 4. 查询条件转换与全文检索

### 4.1 四种搜索方法

`FullTextSearchBlock()` 的 `method` 参数：

| method | 名称 | 执行路径 | 适用场景 |
|--------|------|---------|---------|
| **0** | 关键字（默认） | 单关键词 → FTS5 / 多关键词 → LIKE + GROUP_CONCAT | 普通搜索框 |
| **1** | 查询语法 | FTS5（支持 query syntax） | 高级语法：`AND/OR/NOT/前缀*` |
| **2** | SQL | 原始 SQL 执行（需管理员权限） | 开发者/高级用户自定义查询 |
| **3** | 正则表达式 | `content REGEXP ?` 全表扫描 | 复杂模式匹配 |

**方法 0 的路由逻辑**（关键分支）：
```go
if 2 > len(strings.Split(strings.TrimSpace(query), " ")) {
    // 单关键词 → FTS5 全文检索（高性能）
    query = stringQuery(query)  // 加双引号转义
    blocks = fullTextSearchByFTS(...)
} else {
    // 多关键词 → LIKE + 文档级 GROUP_CONCAT
    // 原因：FTS5 的 NEAR/短语匹配对多词中文支持不佳
    docMode = true
    blocks = fullTextSearchByLikeWithRoot(...)
}
```

### 4.2 FTS5 查询构造流程

**步骤 1：列过滤**（`columnFilter()` 函数）

根据配置选择要检索的 FTS5 可索引列，返回 FTS5 MATCH 语法的列过滤器：
```go
// 默认配置（Name=true, Alias=true, Memo=true, IAL=false）
"{content name alias memo tag}"    // 5 列

// IAL 也开启时
"{content name alias memo ial tag}"  // 6 列
```

> **关键差异**：FTS5 共有 7 个可索引列（name/alias/memo/tag/content/fcontent/ial），但 `columnFilter()` **不包含 fcontent**。fcontent 虽然构建了倒排索引，但不参与 MATCH 搜索，仅在引用搜索排序的 CASE 表达式中通过 `LIKE` 使用（走全量扫描，不走倒排）。

**步骤 2：关键词转义**（`stringQuery()` 函数）

- 双引号内部双写：`"` → `""`
- 单引号双写：`'` → `''`
- 多空格分隔 → 每词独立双引号包裹

**步骤 3：完整 SQL 组装**

```sql
SELECT
  id, parent_id, root_id, hash, box, path,
  snippet(fts, 6, '__@mark__', '__mark@__', '...', 512) AS hpath,    -- 列6: hpath（UNINDEXED，无高亮标记）
  snippet(fts, 7, '__@mark__', '__mark@__', '...', 512) AS name,     -- 列7: name
  snippet(fts, 8, '__@mark__', '__mark@__', '...', 512) AS alias,    -- 列8: alias
  snippet(fts, 9, '__@mark__', '__mark@__', '...', 512) AS memo,     -- 列9: memo
  snippet(fts, 10, '__@mark__', '__mark@__', '...', 64) AS tag,      -- 列10: tag（仅64 token）
  snippet(fts, 11, '__@mark__', '__mark@__', '...', 512) AS content, -- 列11: content
  fcontent, markdown, length, type, subtype, ial, sort, created, updated
FROM blocks_fts_case_insensitive fts
WHERE (fts MATCH '{content name alias memo tag}:("关键词")')
  AND type IN ('d','h','c','m','t','html','av','p')  -- TypeFilter()，默认8种类型
  AND (box = 'box1' OR box = 'box2')                  -- 笔记本过滤
  AND (path LIKE 'path1%' OR path LIKE 'path2%')      -- 路径前缀过滤
ORDER BY CASE WHEN name='kw' THEN 10 ... END          -- 自定义排序
LIMIT 32 OFFSET 0
```

**`type` 字段缩写对照表**（由 `treenode.TypeAbbr()` 生成）：

| AST 节点类型 | 缩写 | 默认开启搜索 |
|-------------|------|------------|
| NodeDocument | `d` | ✓ |
| NodeHeading | `h` | ✓ |
| NodeList | `l` | ✗ |
| NodeListItem | `i` | ✗ |
| NodeCodeBlock | `c` | ✓ |
| NodeMathBlock | `m` | ✓ |
| NodeTable | `t` | ✓ |
| NodeBlockquote | `b` | ✗ |
| NodeSuperBlock | `s` | ✗ |
| NodeParagraph | `p` | ✓ |
| NodeHTMLBlock | `html` | ✓ |
| NodeBlockQueryEmbed | `query_embed` | ✗ |
| NodeAttributeView | `av` | ✓ |
| NodeIFrame | `iframe` | ✗ |
| NodeWidget | `widget` | ✗ |
| NodeAudio | `audio` | ✗ |
| NodeVideo | `video` | ✗ |
| NodeCallout | `callout` | ✗ |

注意：引用搜索排序 CASE 中的 `type = 'd'`（文档块）、`type = 'h'`（标题块）、`type = 'i'`（列表项块）、`type != 'l'`（排除列表块）均使用此缩写体系。

**`snippet()` 函数**：FTS5 内置，返回含高亮标记的文本片段。参数依次为：表名、列索引号、左标记、右标记、省略符、**最大 token 数**（注意：是 token 数，不是字符数，由 siyuan 分词器切分后的 token 计数）。

- 普通搜索共 **6 个 snippet 输出列**：hpath(6, 512) / name(7, 512) / alias(8, 512) / memo(9, 512) / tag(10, 64) / content(11, 512)
- hpath 虽是 UNINDEXED 列（不参与 MATCH 匹配），但 snippet 仍可返回其原值（不会插入高亮标记）
- `fcontent`（列12）和 `ial`（列17）不在 snippet 输出中
- 引用搜索所有 snippet 统一使用 64 token 长度（反链面板空间有限）

### 4.3 多关键词文档模式（LIKE + GROUP_CONCAT）

当查询包含 2+ 空格分隔的关键词时，切换到文档级聚合策略（`docMode=true`）。此模式下不走 FTS5 索引，完全基于 B-Tree 表的 `LIKE` 匹配和 `GROUP_CONCAT` 聚合。

**`columnConcat()` 拼接字符串**（与 `columnFilter()` 列集对称）：
- 默认配置：`content||name||alias||memo||tag`（5 个字段拼接）
- IAL 开启时：`content||name||alias||memo||ial||tag`（6 个字段拼接）
- 同样**不包含 fcontent**

```sql
-- Step 1: CTE 第一阶段 — 找出命中文档（所有关键词在同一文档内至少出现一次）
WITH docBlocks AS (
  SELECT root_id,
         MAX(CASE WHEN type='d' THEN content||name||alias||memo||tag END) AS docContent
  FROM blocks
  WHERE type IN ('d','h','c','m','t','html','av','p')
    AND (box = 'box1' OR box = 'box2')
  GROUP BY root_id
  HAVING GROUP_CONCAT(content||name||alias||memo||tag) LIKE '%kw1%'
     AND GROUP_CONCAT(content||name||alias||memo||tag) LIKE '%kw2%'
     ...
  ORDER BY (CAST(docContent LIKE '%kw1%' AS INT) + CAST(docContent LIKE '%kw2%' AS INT) + ...) DESC,
           MAX(updated) DESC
)

-- Step 2: 第二阶段 — 拉取命中文档内的具体块（文档块本身 + 文档内命中的块）
SELECT *,
       (content||name||alias||memo||tag) AS concatContent,
       CASE WHEN (root_id IN (SELECT root_id FROM docBlocks LIMIT 32)
                  AND (content||name||alias||memo||tag LIKE '%kw1%'
                   AND content||name||alias||memo||tag LIKE '%kw2%'))
            THEN 1 ELSE 0 END AS blockSort
FROM blocks
WHERE type IN ('d','h','c','m','t','html','av','p')
  AND (id IN (SELECT root_id FROM docBlocks LIMIT 32)       -- 文档块本身
    OR (root_id IN (SELECT root_id FROM docBlocks LIMIT 32)  -- 文档内命中的块
       AND concatContent LIKE '%kw1%' AND concatContent LIKE '%kw2%'))
ORDER BY  -- 注入的 ORDER BY，详见排序章节
LIMIT 32 OFFSET 0
```

> **性能代价**：第一阶段对全文档做 `GROUP_CONCAT` 字符串拼接，每个文档的拼接结果可能达 MB 级。大语料库下此查询可能需要数秒甚至数十秒。


### 4.4 正则表达式搜索

通过自定义注册的 `regexp` SQLite 函数实现：

```go
// database.go 注册
sql.Register("sqlite3_extended", &sqlite3.SQLiteDriver{
    ConnectHook: func(conn *sqlite3.SQLiteConn) error {
        return conn.RegisterFunc("regexp", regex, true)
    }
})
```

构造条件：
```sql
WHERE (content REGEXP 'exp' OR name REGEXP 'exp' OR alias REGEXP 'exp' OR ...)
```

注意：**正则搜索无法利用 FTS5 索引**，走全表扫描，需配合 `type IN (...)` 过滤减少范围。

---

## 5. 排序策略深度分析

SiYuan 的排序体系根据 **搜索模式** 和 **orderBy 参数** 的组合产生完全不同的行为。很多排序仅在特定搜索路径下生效，不能一概而论。

### 5.1 `buildOrderBy()` —— 全局排序子句生成器

`buildOrderBy(query, method, orderBy)` 是 `FullTextSearchBlock()` 统一调用的排序子句生成函数，它根据 `orderBy` 参数返回 SQL ORDER BY 片段：

| orderBy | 含义 | 返回的 SQL |
|---------|------|-----------|
| 0 | 按块类型（默认） | `ORDER BY CASE WHEN name='${kw}' THEN 10 WHEN alias='${kw}' THEN 20 WHEN name LIKE '%${kw}%' THEN 50 WHEN alias LIKE '%${kw}%' THEN 60 ELSE 65535 END ASC, sort ASC, updated DESC` |
| 1 | 创建时间升序 | `ORDER BY created ASC` |
| 2 | 创建时间降序 | `ORDER BY created DESC` |
| 3 | 更新时间升序 | `ORDER BY updated ASC` |
| 4 | 更新时间降序 | `ORDER BY updated DESC` |
| 5 | 内容顺序 | **不生成 SQL**（由 Go 层内存排序处理） |
| 6 | 相关度升序 | method=0/1 时：`ORDER BY rank DESC`；method=2/3 时：降级为 `ORDER BY sort DESC, updated DESC` |
| 7 | 相关度降序 | method=0/1 时：`ORDER BY rank`；method=2/3 时：降级为 `ORDER BY sort ASC, updated DESC` |

**关键发现 1**：`buildOrderBy()` 的 orderBy=0（默认）CASE 分支 **只有 4 层**，远比引用搜索的 13 层简单。它仅关注 name 和 alias 两个字段的精确/模糊匹配，不涉及 content/type/memo/fcontent 等字段。

**关键发现 2**：orderBy=6/7 的「相关度」排序，**仅在 method=0（关键字）和 method=1（查询语法）时才使用 FTS5 的 `rank`**；在 method=2（SQL）和 method=3（正则）时，由于不经过 FTS5 表（没有 `rank` 列），**会静默降级为 sort + updated 排序**，此时「相关度」名不副实。

**关键发现 3**：`rank` 的语义——FTS5 的 `rank` 值越小代表越相关（基于 BM25），所以 `ORDER BY rank` 是相关度降序（最相关排最前），而 `ORDER BY rank DESC` 才是相关度升序。代码注释 `// 默认是按相关度降序，所以按相关度升序要反过来使用 DESC` 对此做了解释。

### 5.2 普通搜索（method=0）排序行为

普通搜索是最常用的搜索模式，其排序行为根据关键词数量分为两条完全不同的路径：

#### 5.2.1 单关键词路径 → `fullTextSearchByFTS()`

- **执行层**：FTS5 虚拟表（`blocks_fts` 或 `blocks_fts_case_insensitive`）
- **排序子句**：直接使用 `buildOrderBy()` 返回值
- **orderBy=0（默认）**：CASE 4 层（name精确=10, alias精确=20, name模糊=50, alias模糊=60, ELSE 65535）→ sort ASC → updated DESC
- **orderBy=6/7**：使用 FTS5 `rank`，真正的 BM25 相关度
- **性能**：FTS5 内部利用倒排索引 + 辅助函数 `rank` 计算，无需全表扫描

**风险**：默认排序（orderBy=0）的 CASE 仅基于 name/alias，content 命中的块全部落入 ELSE 65535，**内容命中与未命中之间没有区分度**，完全靠二级 sort（块类型）和三级 updated 区分。这意味着一个内容精确匹配的段落块可能排在 name 模糊匹配的块之后。

#### 5.2.2 多关键词路径 → `fullTextSearchByLikeWithRoot()`

- **执行层**：普通 `blocks` 表 + CTE 子查询
- **排序子句**：对 `buildOrderBy()` 返回值做了**大幅改写**

多关键词模式中，由于查询走的是 `blocks` 表而非 FTS5 表，`rank` 列不存在。代码中对 orderBy 做了如下降级和注入处理：

```
原始 orderBy               → 实际注入的排序
─────────────────────────────────────────────────────────────────
ORDER BY rank DESC (升序)   → 降级为 buildOrderBy(0,0)，即 CASE 4层
                              注入 blockSort ASC（命中的块优先）
ORDER BY rank (降序)        → 降级为 buildOrderBy(0,0)，即 CASE 4层
                              注入 blockSort DESC（命中的块优先）
含 "sort ASC" 的子句        → 在 CASE 后注入 blockSort DESC
其他 orderBy                → 原样使用
```

**`blockSort` 字段**的生成逻辑：
```sql
CASE WHEN (root_id IN (SELECT root_id FROM docBlocks)
       AND (concatContent LIKE '%kw1%' AND concatContent LIKE '%kw2%'))
     THEN 1 ELSE 0 END AS blockSort
```

即：该块所在文档命中 **且** 该块自身也命中所有关键词 → blockSort=1；否则 blockSort=0。排序时将 blockSort 注入到 CASE 和 sort 之间，确保自身命中的块排在仅文档命中的块之前。

**CTE 第一阶段排序**（文档级）：
```sql
ORDER BY (docContent LIKE '%kw1%') + (docContent LIKE '%kw2%') DESC, MAX(updated) DESC
```
命中的关键词越多，文档越靠前；关键词命中数相同则按更新时间降序。

**风险**：
1. **rank 降级后语义丢失**：用户选择「按相关度排序」，但在多关键词模式下实际得到的是 CASE name/alias 匹配 + blockSort + sort + updated，与 BM25 完全无关
2. **GROUP_CONCAT 性能问题**：CTE 第一阶段对每文档做 `GROUP_CONCAT(content||name||...)`，大文档下字符串拼接开销极大
3. **matchedBlockCount = matchedRootCount**：多关键词模式中，`matchedBlockCount` 被设为文档数而非实际块数，这是因为 COUNT 查询走的是 `docBlocks` 子查询

### 5.3 查询语法搜索（method=1）排序行为

- **执行层**：与单关键词路径相同，走 `fullTextSearchByFTS()`
- **排序子句**：直接使用 `buildOrderBy()` 返回值
- **与 method=0 单关键词的唯一区别**：query 不经过 `stringQuery()` 包裹双引号，用户可以直接写 FTS5 查询语法（如 `A AND B`、`prefix*`、`NEAR(...)` 等）
- **orderBy=6/7**：同样使用 FTS5 `rank`，语义正确

**风险**：
1. **默认排序的 CASE 用原始 query 替换 `${keyword}`**：如果用户输入的是复杂语法（如 `siyuan AND note`），CASE 中的 `name = 'siyuan AND note'` 几乎不可能命中，默认排序退化为纯 sort+updated
2. **FTS5 语法错误**：用户输入的查询语法不合法时，FTS5 会返回错误，代码中通过 `sql.SelectBlocksRawStmt` 内部的 `sqlparser` 解析容错，但可能导致空结果

### 5.4 引用搜索（`fullTextSearchRefBlock`）排序行为

引用搜索是独立于 `FullTextSearchBlock()` 的搜索路径，用于反向链接、提及等场景。它 **不使用 `buildOrderBy()`**，而是内置了一套更精细的 13 层 CASE 排序：

```sql
ORDER BY CASE
  WHEN name = '${keyword}'                        THEN 10   -- 命名精确匹配
  WHEN alias = '${keyword}'                       THEN 20   -- 别名精确匹配
  WHEN memo = '${keyword}'                        THEN 30   -- 备注精确匹配
  WHEN content = '${keyword}' AND type = 'd'      THEN 40   -- 文档块内容精确匹配（d=NodeDocument）
  WHEN content LIKE '%${keyword}%' AND type = 'd' THEN 41   -- 文档块内容模糊匹配
  WHEN name LIKE '%${keyword}%'                   THEN 50   -- 命名模糊匹配
  WHEN alias LIKE '%${keyword}%'                  THEN 60   -- 别名模糊匹配
  WHEN content = '${keyword}' AND type = 'h'      THEN 70   -- 标题块内容精确匹配（h=NodeHeading）
  WHEN content LIKE '%${keyword}%' AND type = 'h' THEN 71   -- 标题块内容模糊匹配
  WHEN fcontent = '${keyword}' AND type = 'i'     THEN 80   -- 列表项首块精确匹配（i=NodeListItem）
  WHEN fcontent LIKE '%${keyword}%' AND type = 'i'THEN 81   -- 列表项首块模糊匹配
  WHEN memo LIKE '%${keyword}%'                   THEN 90   -- 备注模糊匹配
  WHEN content LIKE '%${keyword}%'
    AND type != 'i' AND type != 'l'               THEN 100  -- 普通块内容模糊匹配（排除列表项和列表容器）
  ELSE 65535
END ASC,
sort ASC,
length ASC
```

**与普通搜索默认排序的关键差异**：

| 对比维度 | 普通搜索 orderBy=0 | 引用搜索 |
|---------|-------------------|---------|
| CASE 层数 | 4 层 | 13 层 |
| 涉及字段 | name, alias | name, alias, memo, content, fcontent |
| 类型感知 | 无 | 有（`d`=文档/`h`=标题/`i`=列表项 三层类型分层） |
| 精确 vs 模糊 | name/alias 各一层 | 每字段精确+模糊两层 |
| 三级排序 | sort → updated | sort → **length** |
| 输出限制 | 分页 LIMIT+OFFSET | 单一 LIMIT（`Conf.Search.Limit`） |

**引用搜索排序的设计逻辑**：引用场景下用户更关心语义关联度——命名精确匹配 > 文档标题匹配 > 标题块匹配 > 列表项匹配 > 内容模糊匹配。这与普通搜索的「粗粒度 name/alias 优先」形成鲜明对比。

**引用搜索的 snippet 参数**：`snippet(..., 64)` —— 片段长度仅 64 字符（普通搜索为 512），因为反链面板空间有限。

**风险**：
1. **13 层 CASE 依赖 content 全量比较**：`content = '${keyword}'` 需要完整内容精确匹配，对长文本块几乎不可能命中；`content LIKE '%${keyword}%'` 在 B-Tree 表上无法利用索引
2. **无 rank 可用**：引用搜索固定走 FTS5 MATCH 查结果，但排序完全由 CASE 覆盖，未利用 FTS5 的 BM25 相关度信息
3. **LIMIT 无 OFFSET**：引用搜索使用 `Conf.Search.Limit` 做硬截断，不支持分页

### 5.5 正则搜索（method=3）排序行为

- **执行层**：普通 `blocks` 表 + `REGEXP` 函数（全表扫描 + Go 层二次过滤）
- **排序子句**：直接使用 `buildOrderBy()` 返回值
- **orderBy=6/7**：**降级为 `sort DESC, updated DESC` / `sort ASC, updated DESC`**，因为 `blocks` 表无 `rank` 列
- **orderBy=0**：CASE 4 层同普通搜索
- **二级排序**：Go 层 `SelectBlocksRegex()` 在内存中做正则二次过滤后手动分页，可能影响最终结果顺序

**风险**：
1. **相关度排序名不副实**：用户选择「按相关度排序」，实际得到的是块类型+更新时间排序
2. **内存层过滤与 SQL 排序的交互**：SQL 的 ORDER BY 在全表扫描后执行，但 Go 层的正则过滤可能在 SQL LIMIT 之后做二次裁剪，导致分页不准

### 5.6 SQL 搜索（method=2）排序行为

- **执行层**：用户自定义 SQL，直接透传执行
- **排序子句**：由用户 SQL 自带，`buildOrderBy()` 的返回值被**完全忽略**
- **无任何排序干预**：代码仅通过 `sqlparser` 注入分页 LIMIT/OFFSET，不修改 ORDER BY

**风险**：
1. **用户 SQL 无 ORDER BY**：结果顺序不确定，分页无意义
2. **SQL 注入**：虽然需要管理员权限，但允许执行任意 DQL（包括 UNION、子查询等）

### 5.7 按文档分组（groupBy=1）的排序叠加

无论哪种搜索模式，当 `groupBy=1` 时，排序结果会在 Go 层被重新组织：

1. **提取文档根**：遍历搜索结果，按 `rootID` 去重
2. **加载 AST 树**：对每个文档根调用 `loadTreeByBlockTree()` 获取完整树
3. **orderBy=5 时**：遍历 AST 树，为每个块赋值 `contentSorts[blockID] = sortVal++`（DFS 顺序），此排序仅在 Go 层生效
4. **文档内排序**（Children 排序）：
   - orderBy=1/2/3/4：按 created/updated 排序
   - orderBy=5：按 contentSorts 值排序（AST 遍历序）
   - 默认（orderBy=0）：按 sort 值排序
5. **文档间排序**（Root 排序）：
   - orderBy=1/2/3/4：按 created/updated 排序
   - orderBy=5：按 updated 降序（代码注释：都是文档，按更新时间降序）
   - orderBy=6/7：已在 SQL 中处理（但 FTS rank 在分组后语义已变）
   - 默认：不排序（代码注释：都是文档，不需要再次排序）

**风险**：
1. **AST 加载开销**：每个命中文档都要 `loadTreeByBlockTree()`，命中文档数多时内存和 CPU 开销大
2. **分组后 rank 语义丢失**：文档根的 rank 值是根块的 rank，不反映子块的聚合相关度
3. **orderBy=0 分组后退化**：文档间不排序，文档内按 sort（块类型码）排序，丢失了 CASE name/alias 的精细排序

### 5.8 各模式排序汇总

| 搜索模式 | orderBy=0 | orderBy=6 | orderBy=7 | 三级排序 |
|---------|-----------|-----------|-----------|---------|
| 普通-单关键词(FTS) | CASE 4层(name/alias) + sort + updated | `rank DESC` (BM25升序) | `rank` (BM25降序) | sort→updated |
| 普通-多关键词(LIKE) | CASE 4层 + blockSort + sort + updated | 降级为CASE+blockSort ASC | 降级为CASE+blockSort DESC | blockSort→sort→updated |
| 查询语法(FTS) | CASE 4层(query原文) + sort + updated | `rank DESC` | `rank` | sort→updated |
| 引用搜索 | **CASE 13层**(含content/`d`/`h`/`i`类型/fcontent) + sort + length | N/A(固定CASE排序) | N/A | sort→length |
| 正则(blocks表) | CASE 4层 + sort + updated | 降级为sort DESC,updated DESC | 降级为sort ASC,updated DESC | sort→updated |
| SQL(自定义) | 用户SQL自带 | 用户SQL自带 | 用户SQL自带 | 由用户SQL决定 |

---

## 6. 一致性保障机制

### 6.1 事务粒度

**每操作 = 每事务**：

```go
for i, op := range ops {
    tx, err := beginTx()       // 每个 op 独立 BEGIN
    if err = execOp(op, tx); err != nil {
        tx.Rollback()
        continue
    }
    if err = commitTx(tx); err != nil { ... }
}
```

**优点**：单个操作失败不影响其他操作（一个文档损坏不中断整体索引）。
**风险**：跨操作的一致性无保障（如 rename 和 upsert 是两个操作，中间状态可被读到）。

### 6.2 双表同步写

blocks（B-Tree）和 blocks_fts（FTS5）的写入通过**同一事务**中的两条 INSERT 语句保证原子性：

```go
prepareExecInsertTx(tx, BlocksInsert, ...)       // blocks 表
if caseSensitive {
    prepareExecInsertTx(tx, BlocksFTSInsert, ...) // FTS 表（相同参数）
} else {
    prepareExecInsertTx(tx, BlocksFTSCaseInsensitiveInsert, ...)
}
// 两个 INSERT 在同一事务中，要么同时成功要么同时回滚
```

删除同理：`deleteBlocksByIDs()` 先通过 id 查询 ROWID，再分别从两表删除（同事务）。

### 6.3 最终一致性保障

| 机制 | 作用 |
|------|------|
| `WAL 日志模式` | 崩溃后自动回滚未提交事务 |
| `_busy_timeout=7000` | 并发写时最长等 7 秒（超时报错退出进程） |
| `EvtSQLIndexChanged / Flushed` | `Conf.DataIndexState` 标志位：1=有脏数据未刷盘，0=已持久化 |
| `WaitFlushTx()` | 关键路径（如导出/关闭）主动等待队列空 |
| `database disk image is malformed` | 检测到损坏→删除DB文件→强制重建（致命错误处理） |
| `.siyuan/indexignore` | 支持 gitignore 语法排除指定文档不索引 |

### 6.4 缓存一致性

**块缓存层**（ristretto 高性能缓存）：
- 写入时：`putBlockCache(block)` 克隆后存入，移除高亮标记
- 删除时：`removeBlockCache(id)` 同时删除引用缓存
- 批量操作（>512 ops）：`disableCache()` 清空缓存跳过写入，避免缓存失效风暴

**引用缓存层**（go-cache，TTL=30min）：
- `defIDRefsCache`: `map[defBlockID]map[refBlockID]*Ref`
- 块删除时联动清除：`removeBlockCache() → removeRefCacheByDefID()`

---

## 7. 性能风险点与边界问题

### 7.1 写入性能风险

| 风险 | 触发条件 | 影响 |
|------|---------|------|
| **FTS5 双写放大** | 每次 INSERT 走 B-Tree + 倒排索引两套存储 | 写入 IOPS 翻倍，大数据量下明显 |
| **批量过大** | 一次导入 10000+ 文档，队列积压 | 512 条/批但每批独立事务，提交开销大 |
| **Hash 失效风暴** | 大文档 AST 微小改动（如时间戳）影响所有子块 hash | 全文档删除重插，1000+ 块需重建 |
| **_synchronous=OFF** | 极端断电/崩溃 | 最近 3 秒数据可能丢失（但文件完整性靠 WAL） |
| **嵌入块级联** | 嵌入块查询返回 10 万条，拼接 Content | `IndexEmbedBlockJob` 单次最多 64 个，但每块可能膨胀 |

### 7.2 查询性能风险

| 风险 | 触发条件 | 性能表现 |
|------|---------|---------|
| **多关键词 LIKE 模式** | 空格分隔的 2+ 关键词搜索 | `GROUP_CONCAT` + 全表聚合，百万块级查询 >5s |
| **正则搜索** | method=3 且无其他过滤条件 | 全表扫描 + Go 层 regexp 二次校验，O(N) |
| **按文档分组 + 内容顺序** | groupBy=1, orderBy=5 | 需重新遍历每棵 AST 树记录 sortVal，内存开销大 |
| **SQL 搜索（用户自定义 SQL）** | method=2 管理员模式 | 虽然有 sqlparser 注入 LIMIT，但本质允许任意 DQL |
| **FTS5 长查询** | 单关键词过长 + snippet(512) | 高亮计算开销随片段数线性增长 |
| **引用搜索 13 层 CASE** | 反链面板打开时 | CASE 中 `content LIKE` 对每行求值，无法利用索引 |

### 7.3 排序相关的一致性与语义风险

1. **默认排序对内容命中无区分度**：普通搜索 orderBy=0 的 CASE 仅覆盖 name/alias（4 层），content 命中全部落入 ELSE 65535。**一个 content 精确匹配的块与一个完全不匹配的块在 CASE 层面同权重**，仅靠 sort（块类型）和 updated 区分，用户体验上可能感到「搜到了但排序不合理」。

2. **多关键词模式下 rank 降级隐蔽**：用户在 UI 选择「按相关度排序」时，无法感知多关键词模式已将 rank 降级为 CASE+blockSort。BM25 的「词频+逆文档频率」语义完全丢失，blockSort 仅区分「自身命中/仅文档命中」两档。

3. **正则/SQL 模式下相关度排序名不副实**：orderBy=6/7 在 method=2/3 时降级为 sort+updated，前端 UI 仍显示「按相关度排序」选项，但实际与「按块类型」排序几乎等效。

4. **引用搜索排序独立于 `buildOrderBy()`**：引用搜索硬编码 13 层 CASE + length 三级排序，与普通搜索的 4 层 CASE + updated 三级排序完全不同。同一关键词在普通搜索和引用搜索中可能出现在不同位置。

5. **分组后排序体系重构**：groupBy=1 时 Go 层对结果重新排序，文档根的排序逻辑与 SQL 层不一致。例如 orderBy=0 分组后文档间不排序，而 SQL 中有 CASE 排序。

### 7.4 一致性边界问题

1. **3 秒延迟窗口内搜索旧数据**：队列未刷盘时，最新编辑不可搜。属于「写入后读己之写」不一致，需靠前端提示或 `WaitFlushTx()`。

2. **大小写切换不重建**：`SetCaseSensitive()` 仅切换当前查询表，不做数据迁移。切换后旧数据只存在于另一张 FTS 表中，需手动重建索引。

3. **引用解析的两步法**：先入队 `upsertTree`（不建 refs），后 `IndexRefs` 二次扫描。两步骤之间引用关系为空。且 `IndexRefs` 仅启动时执行，运行期新增引用需靠下次写文档触发。

4. **删除路径前缀的 LIKE 匹配风险**：`path LIKE 'foo%'` 会匹配到 `foobar/`，存在误删隐患。正确写法是 `foo/%`，但代码在 `batchDeleteByPathPrefix()` 中使用 `pathPrefix+"%"`。

5. **FTS5 插入非 UNINDEXED 列空值**：`name/alias/memo/tag` 等可索引列若为 NULL 或空串不参与倒排，但仍占用 FTS 行存储。

### 7.5 资源与内存风险

| 风险点 | 分析 |
|--------|------|
| **2.5GB mmap** | `_mmap_size=2684354560`，32 位环境不可用；多库合计 mmap 达 7.5GB |
| **连接池泄漏** | `db.SetConnMaxLifetime(365*24h)` 连接几乎不回收，长期运行后连接状态不可控 |
| **Prepared Statement 缓存** | `txStmtCache` 按 tx 缓存 SQL，长事务中如果 SQL 模板过多，内存膨胀 |
| **WAL 文件体积** | `_synchronous=OFF` + 高频写，WAL 可能膨胀至数 GB，需依赖 checkpoint（SQLite 自动） |

---

## 8. 后续研究方向

### 8.1 架构优化方向

1. **增量 FTS5 替代方案**：当前 upsert 为「删 + 插」，即使单字段变更也会重建整条 FTS 倒排。可研究 FTS5 的 `UPDATE` 语义对 content=contentless 表的效率，或引入 **外部倒排 + contentless FTS5** 模式。

2. **内容可寻址存储 (CAS) + 块内容去重**：当前 hash 仅用于增量对比，未做跨文档块共享。对大量重复内容（如模板、引用）可引入 `block_content(content_hash → text)` 表，blocks 表仅存 hash。

3. **WAL 自动 checkpoint 策略**：当前依赖 SQLite 默认自动 checkpoint（1000 页）。可监控 WAL 体积，达到阈值（如 512MB）主动 `PRAGMA wal_checkpoint(TRUNCATE)`。

4. **向量检索混合查询**：将当前 keyword-only 搜索扩展为「关键词倒排 + 语义向量 ANN」混合检索，支持自然语言相似度查询。需要新增向量索引（如 sqlite-vss 扩展）。

### 8.2 排序语义优化方向

5. **统一排序体系**：当前普通搜索（4 层 CASE）、引用搜索（13 层 CASE）和 rank 排序是三套独立逻辑。建议将引用搜索的精细 CASE 也回迁到普通搜索的默认排序中，至少增加 content/memo 的匹配层级，解决「内容命中无区分度」问题。

6. **多关键词模式 rank 降级透明化**：当 `fullTextSearchByLikeWithRoot` 接收到 orderBy=6/7 时，应向用户明确提示「多关键词搜索不支持 BM25 相关度排序，已切换为语义匹配排序」，或在 CTE 中模拟近似 rank（如基于命中关键词数的加权评分）。

7. **正则/SQL 模式下隐藏 rank 选项**：method=2/3 时 orderBy=6/7 已降级为 sort+updated，前端应禁用或标注降级，避免误导用户。

8. **引用搜索引入 FTS5 rank 辅助排序**：当前引用搜索的 13 层 CASE 完全覆盖了排序逻辑，未利用 FTS5 的 BM25 信息。可在 CASE ELSE 分支中引入 `rank` 作为更深层的区分因子。

### 8.3 性能优化方向

9. **多关键词搜索优化**：当前 2+ 关键词走 `GROUP_CONCAT` 全表聚合。可改为：
   ```
   FTS5('"kw1"') INTERSECT FTS5('"kw2"') INTERSECT ...
   ```
   或使用 FTS5 `phrase` 查询而非纯 LIKE。

10. **查询结果预取 + 滚动游标**：当前 `SelectBlocksRawStmt` 一次加载全量结果到内存。对大 LIMIT 可改为流式游标，配合前端虚拟滚动。

11. **二级索引增强**：当前仅有 root_id / parent_id / id 单列索引。可增加 `(box, path, type)` 复合索引，加速笔记本内类型过滤。

12. **tokenizer 热更新**：当前 `siyuan` tokenizer 为编译期绑定。可研究支持自定义词典、停用词表在线 reload。

### 8.4 一致性增强方向

13. **队列级事务合并**：对同一 root_id 的连续 upsert+rename+move 合并为单事务执行，减少 IO 次数并消除中间可见状态。

14. **刷盘确认回执**：`FlushQueue` 完成后通过 WebSocket 推送 `databaseIndexCommit` 事件，前端据此展示搜索图标状态，并在 API 层提供「等待索引完成」参数。

15. **FTS 表一致性巡检**：启动时对比 `blocks COUNT` 与 `blocks_fts COUNT`，不一致自动修复。当前仅检测表结构版本（`DatabaseVer`），不检测行数一致性。

16. **大小写切换平滑迁移**：`SetCaseSensitive` 变更时自动同步迁移数据（INSERT ... SELECT），而非依赖重建。

### 8.5 边界与鲁棒性研究

17. **超大数据集基准测试**：测试 100w 块 / 10GB 数据规模下的 FTS5 首次查询延迟、WAL checkpoint 抖动、B-Tree 页分裂频率。

18. **并发写入场景压测**：多个协程同时 `UpsertTreeQueue` 不同笔记本时，观察 `sqlite3_busy_timeout` 触发频率与 WAL 竞争。

19. **`_synchronous=OFF` 的 crash-consistency 验证**：使用 kill -9 / 断电测试，验证 WAL 重放能否保证数据库完整，以及最多丢失多少秒的数据。

20. **嵌入块递归查询防护**：A 嵌入 B 的查询结果，B 的结果又包含 A → 无限递归。当前靠 `content = "no query result"` 中断，但需检测循环引用。

---

## 附录：关键源码索引表

| 模块 | 文件 | 关键函数/结构 |
|------|------|-------------|
| DB 初始化 | `kernel/sql/database.go` | `InitDatabase`, `initDBTables`, `SetCaseSensitive` |
| 操作队列 | `kernel/sql/queue.go` | `FlushQueue`, `UpsertTreeQueue`, `execOp` |
| 数据插入 | `kernel/sql/upsert.go` | `upsertTree`, `insertBlocks0`, `insertTree0` |
| 块查询 | `kernel/sql/block_query.go` | `SelectBlocksRawStmt`, `GetBlock`, `Query` |
| 全文搜索 | `kernel/model/search.go` | `FullTextSearchBlock`, `fullTextSearchByFTS`, `fullTextSearchByLikeWithRoot`, `fullTextSearchRefBlock`, `buildOrderBy` |
| 搜索高亮 | `kernel/search/mark.go` | `MarkText`, `EncloseHighlighting` |
| 缓存层 | `kernel/sql/cache.go` | `putBlockCache`, `defIDRefsCache` |
| 索引构建 | `kernel/model/index.go` | `indexBox`, `IndexRefs`, `IndexEmbedBlockJob` |
| 任务调度 | `kernel/task/queue.go` | `AppendTask`, `ExecTaskJob`, `popTask` |
| 定时任务 | `kernel/job/cron.go` | `StartCron`, `every` |
| 搜索配置 | `kernel/conf/search.go` | `Search.TypeFilter`, `NewSearch` |
| HTTP API | `kernel/api/search.go` | `fullTextSearchBlock`, `searchRefBlock` |
