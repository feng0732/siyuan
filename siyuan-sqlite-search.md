# SiYuan SQLite 索引与全文检索机制分析

## 1. 架构概览

SiYuan（思源笔记）采用 **SQLite + FTS5 全文搜索引擎** 作为核心数据检索引擎，配合 **异步队列 + 定时刷盘** 的更新策略，实现笔记内容的高性能存储与检索。系统包含三个独立的 SQLite 数据库实例，通过 `go-sqlite3` 驱动的 `sqlite3_extended` 自定义注册（注入 `regexp` 函数）与底层交互。

### 1.1 三大数据库实例

| 数据库 | 路径变量 | 核心用途 | 连接池 |
|--------|---------|---------|--------|
| 主索引库 | `util.DBPath` | 块索引、FTS全文索引、引用关系、资源元数据 | MaxOpen=20 |
| 历史库 | `util.HistoryDBPath` | 历史版本全文检索 | MaxOpen=3 |
| 资源内容库 | `util.AssetContentDBPath` | PDF/Office 等附件内容全文索引 | MaxOpen=3 |

参考代码：[database.go#L228-L370](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L228-L370)

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

**B-Tree 索引配置**（参考 [database.go#L128-L207](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L128-L207)）：

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

系统采用 **双 FTS5 表切换** 方案支持大小写敏感配置（参考 [database.go#L148-L164](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L148-L164)）：

```sql
-- 大小写敏感
CREATE VIRTUAL TABLE blocks_fts USING fts5(
  id UNINDEXED, parent_id UNINDEXED, ...,
  name, alias, memo, tag, content, fcontent, ial, ...
  tokenize="siyuan"
);

-- 大小写不敏感（默认）
CREATE VIRTUAL TABLE blocks_fts_case_insensitive USING fts5(
  ...,
  tokenize="siyuan case_insensitive"
);
```

**设计要点**：
- 使用 SiYuan 自研的 `siyuan` tokenizer 分词器（支持中文切分）
- 大量字段标记为 `UNINDEXED`（id/parent_id/root_id/hash/box/path 等 13 个字段），仅对检索相关列构建倒排索引，显著降低存储开销
- 可索引列：`name`, `alias`, `memo`, `tag`, `content`, `fcontent`, `ial`（共 7 列）
- 搜索时根据 `Conf.Search.CaseSensitive` 动态选择目标表

**历史库与资源库 FTS5 表**：
- `histories_fts_case_insensitive`：仅大小写不敏感模式（参考 [database.go#L304-L310](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L304-L310)）
- `asset_contents_fts_case_insensitive`：附件内容索引，列含 `name, ext, path, content`（参考 [database.go#L364-L370](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L364-L370)）

### 2.3 SQLite 连接关键参数

参考 [database.go#L232-L241](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L232-L241)：

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
    │  参考 cron.go#L39
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
    │     │   └─ 参考 upsert.go#L399-L426
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

参考 [queue.go#L37-L437](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/queue.go#L37-L437)

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

参考 [upsert.go#L399-L453](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/upsert.go#L399-L453)

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

参考 [model/search.go#L1156-L1214](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go#L1156-L1214)

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

参考 [model/search.go#L1619-L1663](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go#L1619-L1663)

**步骤 1：列过滤**（`columnFilter()` 函数）

根据配置选择要检索的 FTS5 列（参考 [model/search.go#L1979-L1996](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go#L1979-L1996)）：
```go
// 示例：Name/Alias/Memo 开启，IAL 关闭
"{content name alias memo tag}"
```

**步骤 2：关键词转义**（`stringQuery()` 函数）

参考 [model/search.go#L2017-L2038](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go#L2017-L2038)：
- 双引号内部双写：`"` → `""`
- 单引号双写：`'` → `''`
- 多空格分隔 → 每词独立双引号包裹

**步骤 3：完整 SQL 组装**

```sql
SELECT
  id, parent_id, root_id, hash, box, path,
  snippet(blocks_fts_case_insensitive, 6, '__@mark__', '__mark@__', '...', 512) AS hpath,
  snippet(..., 7, ...) AS name,   -- alias/memo/tag/content 共 7 列高亮
  fcontent, markdown, length, type, subtype, ial, sort, created, updated
FROM blocks_fts_case_insensitive
WHERE (blocks_fts_case_insensitive MATCH '{content name alias memo tag}:("关键词")')
  AND type IN ('d','h','c','m','t','html','av','p')  -- TypeFilter()
  AND (box = 'box1' OR box = 'box2')                  -- 笔记本过滤
  AND (path LIKE 'path1%' OR path LIKE 'path2%')      -- 路径过滤
ORDER BY CASE WHEN name='kw' THEN 10 ... END          -- 自定义排序
LIMIT 32 OFFSET 0
```

**`snippet()` 函数**：FTS5 内置，返回含高亮片段的文本，参数为：
- 表名、列索引、左标记、右标记、省略符、每片段最大长度（512 字符）

### 4.3 多关键词文档模式（LIKE + GROUP_CONCAT）

参考 [model/search.go#L1665-L1719](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go#L1665-L1719)

当查询包含 2+ 空格分隔的关键词时，切换到文档级聚合策略：

```sql
-- Step 1: 找出命中文档（所有关键词在同一文档内至少出现一次）
WITH docBlocks AS (
  SELECT root_id,
         MAX(CASE WHEN type='d' THEN content||name||alias||memo||ial||tag END) AS docContent
  FROM blocks
  WHERE type IN (...) ...
  GROUP BY root_id
  HAVING GROUP_CONCAT(content||name||...) LIKE '%kw1%'
     AND GROUP_CONCAT(content||name||...) LIKE '%kw2%'
  ORDER BY (docContent LIKE '%kw1%') + ... DESC, MAX(updated) DESC
)

-- Step 2: 拉取命中文档内的具体块（文档块 + 具体命中块）
SELECT *, (content||name||...) AS concatContent
FROM blocks
WHERE type IN (...) ...
  AND (id IN (SELECT root_id FROM docBlocks LIMIT 32)
    OR (root_id IN (SELECT root_id FROM docBlocks LIMIT 32)
       AND concatContent LIKE '%kw1%' AND concatContent LIKE '%kw2%'))
ORDER BY ...  -- 自定义排序 + blockSort 标志位
```

### 4.4 正则表达式搜索

参考 [model/search.go#L1586-L1617](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go#L1586-L1617)

通过自定义注册的 `regexp` SQLite 函数实现：

```go
// database.go#L58-L69 注册
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

### 5.1 八种排序模式

参考 [buildOrderBy()](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go#L1367-L1398)

| orderBy | 模式 | SQL 片段 |
|---------|------|---------|
| **0** | **按块类型（默认，最复杂）** | 多层 CASE + sort ASC + updated DESC |
| 1 | 创建时间升序 | `ORDER BY created ASC` |
| 2 | 创建时间降序 | `ORDER BY created DESC` |
| 3 | 更新时间升序 | `ORDER BY updated ASC` |
| 4 | 更新时间降序 | `ORDER BY updated DESC` |
| 5 | 内容顺序（仅分组） | 遍历树记录 sortVal，内存排序 |
| 6 | 相关度升序 | FTS: `ORDER BY rank DESC`（反向） |
| 7 | 相关度降序 | FTS: `ORDER BY rank` |

### 5.2 「按块类型」排序算法详解

这是 SiYuan 的默认排序，**语义分层排序**，核心思想：精确匹配 > 标题匹配 > 文档内容匹配 > 列表项匹配 > 其他。

参考 `fullTextSearchRefBlock()` 中的完整版本 [model/search.go#L1544-L1559](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go#L1544-L1559)：

```sql
ORDER BY CASE
  WHEN name = '${keyword}'                        THEN 10   -- 命名属性精确匹配
  WHEN alias = '${keyword}'                       THEN 20   -- 别名精确匹配
  WHEN memo = '${keyword}'                        THEN 30   -- 备注精确匹配
  WHEN content = '${keyword}' AND type = 'd'      THEN 40   -- 文档标题精确匹配
  WHEN content LIKE '%${keyword}%' AND type='d'  THEN 41   -- 文档标题部分匹配
  WHEN name LIKE '%${keyword}%'                   THEN 50   -- 命名模糊匹配
  WHEN alias LIKE '%${keyword}%'                  THEN 60   -- 别名模糊匹配
  WHEN content = '${keyword}' AND type = 'h'      THEN 70   -- 标题块精确匹配
  WHEN content LIKE '%${keyword}%' AND type='h'  THEN 71   -- 标题块模糊匹配
  WHEN fcontent = '${keyword}' AND type = 'i'     THEN 80   -- 列表项首块精确匹配
  WHEN fcontent LIKE '%${keyword}%' AND type='i' THEN 81   -- 列表项首块模糊匹配
  WHEN memo LIKE '%${keyword}%'                   THEN 90   -- 备注模糊匹配
  WHEN content LIKE '%${keyword}%'
    AND type != 'i' AND type != 'l'               THEN 100  -- 普通块内容模糊匹配
  ELSE 65535
END ASC,
sort ASC,          -- 块类型二级排序（文档=0，标题=5，段落=10...）
length ASC         -- 同级别按长度，短的优先
```

**块类型 sort 值**（参考 `nSort()` [database.go#L1532-L1567](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L1532-L1567)）：

| sort 值 | 块类型 |
|---------|--------|
| 0 | Document 文档 |
| 5 | Heading 标题 |
| 10 | Paragraph / CodeBlock / MathBlock / Table / HTMLBlock |
| 20 | List / ListItem / Blockquote / Callout |
| 30 | SuperBlock / AttributeView |
| 200 | Text / TextMark |
| 205 | Tag 标签 |
| 100 | 其他 |

### 5.3 FTS5 相关度排序

对于 method=0/1 且 orderBy=6/7，使用 FTS5 内置的 `rank` 排序值：
```sql
-- 相关度降序（默认相关度搜索）
ORDER BY rank

-- 相关度升序（注意 rank 越小越相关，所以 ASC 实际是最相关在前，这里作者做了反直觉处理）
ORDER BY rank DESC
```

FTS5 的 `rank` 值基于 **BM25 算法** 计算：综合词频(TF)、逆文档频率(IDF)、文档长度归一化。

---

## 6. 一致性保障机制

### 6.1 事务粒度

**每操作 = 每事务**：参考 [FlushQueue()](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/queue.go#L102-L180)

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

参考 [insertBlocks0()](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/upsert.go#L85-L141)：
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

**块缓存层**（ristretto 高性能缓存，参考 [cache.go#L42-L82](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/cache.go#L42-L82)）：
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
| **SQL 注入（用户自定义 SQL）** | method=2 管理员模式 | 虽然有 sqlparser 注入 LIMIT，但本质允许任意 DQL |
| **FTS5 长查询** | 单关键词过长 + snippet(512) | 高亮计算开销随片段数线性增长 |

### 7.3 一致性边界问题

1. **3 秒延迟窗口内搜索旧数据**：队列未刷盘时，最新编辑不可搜。属于「写入后读己之写」不一致，需靠前端提示或 `WaitFlushTx()`。

2. **大小写切换不重建**：`SetCaseSensitive()` 仅切换当前查询表，不做数据迁移。切换后旧数据只存在于另一张 FTS 表中，需手动重建索引（参考 [database.go#L377-L391](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L377-L391)）。

3. **引用解析的两步法**：先入队 `upsertTree`（不建 refs），后 `IndexRefs` 二次扫描。两步骤之间引用关系为空。且 `IndexRefs` 仅启动时执行，运行期新增引用需靠下次写文档触发。

4. **删除路径前缀的 LIKE 匹配风险**：`path LIKE 'foo%'` 会匹配到 `foobar/`，存在误删隐患。正确写法是 `foo/%`，但代码在 `batchDeleteByPathPrefix()` 中使用 `pathPrefix+"%"`（参考 [database.go#L1208-L1246](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go#L1208-L1246)）。

5. **FTS5 插入非 UNINDEXED 列空值**：`name/alias/memo/tag` 等可索引列若为 NULL 或空串不参与倒排，但仍占用 FTS 行存储。

### 7.4 资源与内存风险

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

### 8.2 性能优化方向

5. **多关键词搜索优化**：当前 2+ 关键词走 `GROUP_CONCAT` 全表聚合。可改为：
   ```
   FTS5('"kw1"') INTERSECT FTS5('"kw2"') INTERSECT ...
   ```
   或使用 FTS5 `phrase` 查询而非纯 LIKE。

6. **查询结果预取 + 滚动游标**：当前 `SelectBlocksRawStmt` 一次加载全量结果到内存。对大 LIMIT 可改为流式游标，配合前端虚拟滚动。

7. **二级索引增强**：当前仅有 root_id / parent_id / id 单列索引。可增加 `(box, path, type)` 复合索引，加速笔记本内类型过滤。

8. **tokenizer 热更新**：当前 `siyuan` tokenizer 为编译期绑定。可研究支持自定义词典、停用词表在线 reload。

### 8.3 一致性增强方向

9. **队列级事务合并**：对同一 root_id 的连续 upsert+rename+move 合并为单事务执行，减少 IO 次数并消除中间可见状态。

10. **刷盘确认回执**：`FlushQueue` 完成后通过 WebSocket 推送 `databaseIndexCommit` 事件（已实现，见 [queue.go#L177](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/queue.go#L177)），前端据此展示搜索图标状态，并在 API 层提供「等待索引完成」参数。

11. **FTS 表一致性巡检**：启动时对比 `blocks COUNT` 与 `blocks_fts COUNT`，不一致自动修复。当前仅检测表结构版本（`DatabaseVer`），不检测行数一致性。

12. **大小写切换平滑迁移**：`SetCaseSensitive` 变更时自动同步迁移数据（INSERT ... SELECT），而非依赖重建。

### 8.4 边界与鲁棒性研究

13. **超大数据集基准测试**：测试 100w 块 / 10GB 数据规模下的 FTS5 首次查询延迟、WAL checkpoint 抖动、B-Tree 页分裂频率。

14. **并发写入场景压测**：多个协程同时 `UpsertTreeQueue` 不同笔记本时，观察 `sqlite3_busy_timeout` 触发频率与 WAL 竞争。

15. **`_synchronous=OFF` 的 crash-consistency 验证**：使用 kill -9 / 断电测试，验证 WAL 重放能否保证数据库完整，以及最多丢失多少秒的数据。

16. **嵌入块递归查询防护**：A 嵌入 B 的查询结果，B 的结果又包含 A → 无限递归。当前靠 `content = "no query result"` 中断，但需检测循环引用。

---

## 附录：关键源码索引表

| 模块 | 文件 | 关键函数/结构 |
|------|------|-------------|
| DB 初始化 | [database.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/database.go) | `InitDatabase`, `initDBTables`, `SetCaseSensitive` |
| 操作队列 | [queue.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/queue.go) | `FlushQueue`, `UpsertTreeQueue`, `execOp` |
| 数据插入 | [upsert.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/upsert.go) | `upsertTree`, `insertBlocks0`, `insertTree0` |
| 块查询 | [block_query.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/block_query.go) | `SelectBlocksRawStmt`, `GetBlock`, `Query` |
| 全文搜索 | [model/search.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/search.go) | `FullTextSearchBlock`, `fullTextSearchByFTS` |
| 搜索高亮 | [search/mark.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/search/mark.go) | `MarkText`, `EncloseHighlighting` |
| 缓存层 | [sql/cache.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/sql/cache.go) | `putBlockCache`, `defIDRefsCache` |
| 索引构建 | [model/index.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/model/index.go) | `indexBox`, `IndexRefs`, `IndexEmbedBlockJob` |
| 任务调度 | [task/queue.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/task/queue.go) | `AppendTask`, `ExecTaskJob`, `popTask` |
| 定时任务 | [job/cron.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/job/cron.go) | `StartCron`, `every` |
| 搜索配置 | [conf/search.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/conf/search.go) | `Search.TypeFilter`, `NewSearch` |
| HTTP API | [api/search.go](file:///d:/fz/0601/solo-dogfeeding/code/290-siyuan/kernel/api/search.go) | `fullTextSearchBlock`, `searchRefBlock` |
