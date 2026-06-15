# SiYuan 笔记本与文档树管理核心实现分析

## 1. 概述

SiYuan（思源笔记）是一款基于本地优先理念的个人知识管理系统。其笔记本（Box）与文档树（Tree）管理是整个系统的核心基石。本文档从代码结构层面深入分析层级关系、节点移动、排序状态、权限约束和持久化更新的联动机制，特别关注并发变更、缓存刷新和异常恢复。

## 2. 代码结构与职责划分

### 2.1 核心模块架构

```
kernel/
├── model/
│   ├── box.go              # 笔记本（Box）核心管理
│   ├── tree.go             # 文档树加载与索引
│   ├── file.go             # 文件树操作（移动/重命名/删除）
│   ├── transaction.go      # 事务处理核心
│   ├── publish_access.go  # 发布访问控制
│   └── role.go             # 角色权限管理
├── treenode/
│   ├── node.go             # 节点模型与工具函数
│   └── blocktree.go        # SQLite 块树索引
├── cache/
│   ├── tree.go             # Ristretto 树数据缓存
│   └── ial.go              # Ristretto 文档 IAL 缓存
├── filesys/
│   └── tree.go             # 文件系统持久化
└── api/
    ├── filetree.go         # HTTP API 处理
    └── block_op.go         # 块操作 API
```

### 2.2 各层职责

| 层级 | 模块 | 主要职责 |
|------|------|----------|
| API 层 | `api/filetree.go` | 接收 HTTP 请求，参数校验，权限检查 |
| 业务层 | `model/*.go` | 核心业务逻辑，事务编排，状态流转 |
| 缓存层 | `cache/tree.go`, `cache/ial.go` | 内存缓存，加速树数据和文档属性读取 |
| 索引层 | `treenode/blocktree.go` | SQLite 二级索引，快速块查找 |
| 持久层 | `filesys/tree.go` | 文件系统读写，数据持久化 |

## 3. 核心数据模型

### 3.1 笔记本（Box）模型

定义于 `kernel/model/box.go`：

```go
type Box struct {
    ID                  string  `json:"id"`
    Name                string  `json:"name"`
    Icon                string  `json:"icon"`
    Sort                int     `json:"sort"`
    SortMode            int     `json:"sortMode"`
    Closed              bool    `json:"closed"`
    NewFlashcardCount   int     `json:"newFlashcardCount"`
    DueFlashcardCount   int     `json:"dueFlashcardCount"`
    FlashcardCount      int     `json:"flashcardCount"`
}
```

**关键特性：**

- `ID`：笔记本唯一标识，也是文件系统目录名
- `SortMode`：排序模式，支持名称、创建时间、更新时间、自定义等
- `Closed`：笔记本关闭状态，关闭后不加载到内存

### 3.2 文档树节点模型

定义于 `kernel/treenode/node.go`：

**节点类型缩写映射：**

```go
NodeDocument    -> "d"    // 文档
NodeHeading     -> "h"    // 标题
NodeParagraph   -> "p"    // 段落
NodeList        -> "l"    // 列表
NodeListItem    -> "i"    // 列表项
NodeBlockquote  -> "b"    // 引用
NodeCodeBlock   -> "c"    // 代码块
```

### 3.3 BlockTree 索引模型

定义于 `kernel/treenode/blocktree.go` L36-L45：

```go
type BlockTree struct {
    ID       string  // 块 ID
    RootID   string  // 根文档 ID
    ParentID string  // 父块 ID
    BoxID    string  // 笔记本 ID
    Path     string  // 文档数据路径
    HPath    string  // 文档可读路径
    Updated  string  // 更新时间
    Type     string  // 块类型
}
```

**数据库表结构：**

```sql
CREATE TABLE blocktrees (
    id, root_id, parent_id, box_id, path, hpath, updated, type
)
CREATE INDEX idx_blocktrees_id ON blocktrees(id)
CREATE INDEX idx_blocktrees_root_id ON blocktrees(root_id)
```

### 3.4 文件节点模型

定义于 `kernel/model/file.go`：

```go
type File struct {
    Path            string `json:"path"`
    Name            string `json:"name"`
    Icon            string `json:"icon"`
    ID              string `json:"id"`
    Count           int    `json:"count"`
    Size            int64  `json:"size"`
    Mtime           string `json:"mtime"`
    Sort            int    `json:"sort"`
    SubFileCount    int    `json:"subFileCount"`
    Hidden          bool   `json:"hidden"`
}
```

## 4. 层级关系与节点模型

### 4.1 层级结构设计

SiYuan 采用**文件系统目录结构**作为文档树的物理存储，配合**内存 AST 树**作为逻辑表示，通过**SQLite 索引**加速查询。

```
物理层（文件系统）：
data/
└── 20210808180117-abc1234/          # 笔记本目录 (Box ID)
    ├── 20210808180117-def5678.sy  # 根文档
    └── 20210808180117-ghi9012/     # 子文档目录
        └── 20210808180117-jkl3456.sy  # 子文档

逻辑层（内存 AST）：
parse.Tree
└── NodeDocument (root)
    ├── NodeHeading
    ├── NodeParagraph
    └── NodeList

索引层（SQLite）：
blocktrees 表记录每个块的位置信息
```

### 4.2 父子关系维护

**路径规则：**

- 文档路径 = 父文档路径 + `/` + 文档 ID + `.sy`
- 子文档目录 = 父文档路径去掉 `.sy` 后缀
- HPath（可读路径）= 所有祖先文档名用 `/` 连接

**关键函数：**

1. **节点查找** — `GetNodeInTree()`（`kernel/treenode/node.go`）
   - 在 AST 树中按 ID 递归查找节点
   - 支持所有块类型

2. **父节点链获取** — `ParentNodesWithHeadings()`（`kernel/treenode/node.go`）
   - 获取节点的所有祖先节点
   - 包括标题层级关系

3. **更新时间刷新** — `RefreshUpdated()`（`kernel/treenode/node.go`）
   - 更新节点的 updated 时间戳
   - 级联更新所有父节点

### 4.3 树加载的两种入口

SiYuan 有两种树加载入口，行为不同：

**1. 普通加载** — `LoadTreeByBlockID()`（`kernel/model/tree.go`）：

```
1. 校验 ID 格式
2. 查 BlockTree 索引
3. 索引不存在 -> 返回 ErrTreeNotFound（不搜索文件系统）
4. 索引存在 -> loadTreeByBlockTree
     ├─> LoadTreeWithFix 加载文件
     │    ├─> 有缓存直接用缓存
     │    └─> 读文件 -> fixTreeJSONData 数据修复 -> needFix?
     └─> needFix 为 true 时更新 BlockTree 和 SQL 索引
```

> **注意**：普通加载不会因为索引与文件不一致而自动修正。只有数据修复（XSS、Unicode、ID 不一致）触发 `needFix` 时，才会更新索引。

**2. 带重建的加载** — `LoadTreeByBlockIDWithReindex()`（`kernel/model/tree.go` L188-L215）：

```
1. 查 BlockTree 索引
2. 索引不存在 -> indexTreeInFilesystem() 全量搜索文件系统
     ├─> 限频保护（3秒1次）
     ├─> findUnindexedTreePathInAllBoxes 遍历所有笔记本
     └─> 找到后加载树并建立索引
3. 索引存在 -> 正常加载
```

> **使用场景**：仅在特定场景下调用，如打开文档时找不到索引的容错处理。并非每次加载都会触发。

### 4.4 索引修复链路（checkIndex）

除了上述加载时的局部修复，还有一套完整的索引校验修复机制，仅在**数据同步完成后执行一次**（`sync.Once` 保护）：

**执行顺序**（`kernel/model/index_fix.go` checkIndex）：

```
1. removeDuplicateDatabaseIndex  — 删除数据库索引重复项
2. resetDuplicateBlocksOnFileSys — 重置文件系统上重复的块 ID
3. fixBlockTreeByFileSys         — 以文件系统为准订正 BlockTree
4. fixDatabaseIndexByBlockTree   — 以 BlockTree 为准订正 SQL 搜索索引
5. removeDuplicateDatabaseRefs   — 删除数据库引用重复项
```

**fixBlockTreeByFileSys 核心逻辑**：

```
遍历文件系统所有 .sy 文件 -> 得到 paths 列表
   ├─> ClearRedundantBlockTrees — 删除索引中有、文件中没有的（冗余）
   └─> GetNotExistPaths         — 找出文件中有、索引中没有的（缺失）
        └─> 逐个 reindexTreeByPath 重建索引
```

> **关键纠正**：BlockTree 索引与文件系统不一致时，**不会自动按文件修正**。只有在以下三种场景才会修正：
> 1. 加载时检测到数据需要修复（needFix）
> 2. 显式调用带重建的加载接口
> 3. 同步完成后的一次性索引校验（checkIndex）

## 5. 节点移动机制

### 5.1 移动类型

1. **文档级移动**（跨笔记本/跨目录）
2. **块级移动**（文档内/文档间）

### 5.2 文档移动核心流程

`MoveDocs()`（`kernel/model/file.go` L1296-L1356）：

```
1. 前置检查
   ├─> 笔记本存在性检查
   ├─> 过滤无效路径（移动到自身父级视为不移动）
   └─> 深度检查（最大 7 层限制，除非 AllowCreateDeeper 开启）

2. 事务同步
   └─> FlushTxQueue() // 先清空事务队列，确保移动前所有数据落盘

3. 逐文档移动
   └─> moveDoc()
        ├─> 加载源文档和目标文档
        ├─> 创建目标目录
        ├─> 移动子文档目录（如果有）
        ├─> filelock.Rename() 文件系统重命名
        ├─> 重新加载文档
        ├─> moveTree() 更新索引
        │    ├─> treenode.SetBlockTreePath()  // 更新 BlockTree 索引
        │    └─> sql.MoveTreeQueue()          // SQL 索引队列
        ├─> moveSorts() // 跨笔记本时迁移排序配置
        └─> 递归处理子文档并推送事件

4. 后置处理
   ├─> cache.ClearDocsIAL() // 清空全部 docIAL 缓存
   └─> IncSync() // 增加同步计数
```

> **重要**：文档移动是**非事务操作**，直接操作文件系统。移动前调用 `FlushTxQueue()` 确保没有未提交的块事务，避免数据不一致。

### 5.3 移动验证机制

**块级移动验证**（事务内移动块时）：

1. `isMovingParentIntoChild()`（`kernel/model/transaction.go`）
   - 检查是否将父节点移动到其子节点中
   - 防止循环引用和结构破坏

2. `isMovingFoldHeadingIntoSelf()`（`kernel/model/transaction.go`）
   - 检查折叠标题移动到自身内部
   - 防止折叠内容丢失

> **文档级移动**：文档级移动通过 `MoveDocs()` 函数执行，在文件系统层面重命名，不经过事务队列，因此没有上述验证。文档级移动的循环引用检查通过路径比较实现。

### 5.4 子文档级联移动

`moveTree()`（`kernel/model/box.go` L467-L491）：

```go
func moveTree(tree *parse.Tree) {
    treenode.SetBlockTreePath(tree)
    sql.MoveTreeQueue(tree)

    box := Conf.Box(tree.Box)
    subFiles := box.ListFiles(tree.Path)
    for _, subFile := range subFiles {
        if !strings.HasSuffix(subFile.path, ".sy") {
            continue
        }

        subTree, err := filesys.LoadTree(box.ID, subFile.path, luteEngine)
        
        treenode.SetBlockTreePath(subTree)
        sql.MoveTreeQueue(subTree)
    }

    refreshDocInfo(tree)
}
```

> **关键点**：子文档的 HPath 由 `LoadTree` 自动重新构造，通过读取各级父文档的标题拼接而成。

## 6. 排序状态管理

### 6.1 排序模式

定义于 `kernel/model/box.go`：

| 模式 | 值 | 说明 |
|------|----|------|
| Name ASC | 0 | 名称升序 |
| Name DESC | 1 | 名称降序 |
| Updated ASC | 2 | 更新时间升序 |
| Updated DESC | 3 | 更新时间降序 |
| Created ASC | 4 | 创建时间升序 |
| Created DESC | 5 | 创建时间降序 |
| Custom | 6 | 自定义排序 |
| RefCount | 7 | 引用数排序 |
| Size | 9 | 大小排序 |
| SubDocCount | 10 | 子文档数排序 |
| Alphanum | 11 | 字母数字排序 |

### 6.2 排序实现

**笔记本排序** — `ListNotebooks()`（`kernel/model/box.go`）

```go
func ListNotebooks() ([]*Box, error) {
    // 1. 读取数据目录
    // 2. 过滤有效 ID 目录
    // 3. 加载每个笔记本的 conf.json
    // 4. 按 SortMode 排序
    // 5. 返回排序结果
}
```

**文档排序** — `ListDocTree()`（`kernel/model/file.go`）

```go
func ListDocTree() {
    // 1. 读取目录下所有 .sy 文件
    // 2. 加载每个文档的 IAL
    // 3. 按笔记本配置的 SortMode 排序
    // 4. 填充 Sort 字段
}
```

### 6.3 自定义排序

`ChangeFileTreeSort()`（`kernel/model/file.go` L1949-L2010）：

```
1. 接收排序配置（paths []string 按顺序排列）
2. 遍历 paths 按索引设置 sort 值
3. 持久化到文档 IAL
4. 更新 BlockTree 索引
```

### 6.4 排序持久化

排序值存储在文档的 IAL（Inline Attribute List）中：

```
---
{
    "id": "20210808180117-abc1234",
    "title": "文档标题",
    "sort": 3
}
---
```

## 7. 权限约束机制

### 7.1 角色权限

定义于 `kernel/model/role.go`：

| 角色 | 值 | 权限 |
|------|----|------|
| Administrator | 0 | 完全控制 |
| Editor | 1 | 编辑权限 |
| Reader | 2 | 只读 |
| Visitor | 3 | 访客（只读） |

**只读检查：**

```go
func IsReadOnlyRole(role int) bool {
    return RoleReader == role || RoleVisitor == role
}
```

### 7.2 发布访问控制

定义于 `kernel/model/publish_access.go`：

```go
type PublishAccessItem struct {
    ID       string `json:"id"`
    Visible  bool   `json:"visible"`
    Password string `json:"password"`
    Disable  bool   `json:"disable"`
}
```

**访问控制流程：**

1. **可见性检查** — `CheckPathAccessableByPublishIgnore()`（`kernel/model/publish_access.go`）

```
路径遍历向上查找第一个设置了可见性的祖先
├─> visible=false -> 不可访问
├─> visible=true -> 可访问
└─> 未设置 -> 继续向上
```

2. **密码检查** — `GetPathPasswordByPublishAccess()`（`kernel/model/publish_access.go`）

```
向上遍历查找密码
├─> 找到密码 -> 检查 Cookie
│   ├─> Cookie 有效 -> 放行
│   └─> Cookie 无效 -> 返回密码页
└─> 未找到 -> 放行
```

3. **内容过滤** — `FilterContentByPublishAccess()`（`kernel/model/publish_access.go`）

```
无权访问时替换内容
├─> 有密码 -> 密码输入页 HTML
└─> 无密码 -> 禁止访问 HTML
```

**缓存策略：**

- PublishAccess 缓存 30 秒
- Cookie 有效期 24 小时
- Cookie 值 = SHA256(ID + Password)

## 8. 持久化更新联动机制

### 8.1 三层持久化架构

```
┌─────────────────────────────────────────────────┐
│              内存 AST 树 (parse.Tree)            │
└─────────────────┬───────────────────────────────┘
                  │
               写入
                  ▼
┌─────────────────────────────────────────────────┐
│         文件系统 (.sy JSON 文件)                 │
│  ────────────────────────────────────────       │
│  • mmap 优先，writeFile 降级                    │
│  • filelock 跨进程锁                            │
└─────────────────┬───────────────────────────────┘
                  │
             索引更新
                  ▼
┌─────────────────────────────────────────────────┐
│        SQLite BlockTree 索引                     │
│  ────────────────────────────────────────       │
│  • WAL 模式，异步写入                            │
│  • mmap_size 2.5GB                              │
│  • 自动重建损坏数据库                             │
└─────────────────┬───────────────────────────────┘
                  │
             缓存更新
                  ▼
┌─────────────────────────────────────────────────┐
│       Ristretto 缓存                            │
│  ────────────────────────────────────────       │
│  • 200MB 容量                                   │
│  • LRU 淘汰策略                                 │
└─────────────────────────────────────────────────┘
```

### 8.2 写入主流程

`WriteTree()`（`kernel/filesys/tree.go` L244-L260）：

```go
func WriteTree(tree *parse.Tree) (uint64, error) {
    // 1. 准备写入数据（序列化为 JSON）
    data, filePath, err := prepareWriteTree(tree)
    
    // 2. 双写策略：mmap 优先，失败降级到 writeFile
    if err = writeTreeByMmap(filePath, data); nil != err {
        if err = writeTreeByWriteFile(filePath, data); nil != err {
            return 0, err
        }
    }
    
    // 3. 更新树数据缓存
    cache.SetTreeData(tree.ID, data)
    
    // 4. 后置处理：更新 docIAL 缓存
    afterWriteTree(tree)
}
```

**执行顺序确认：**

1. 先写入文件系统（mmap → writeFile 降级）
2. 再更新 `treeCache`（原始 JSON 数据缓存）
3. 最后更新 `docIALCache`（文档属性缓存）

> **重要**：文件写入成功后才更新缓存。若文件写入失败，缓存保持不变。

### 8.3 mmap 写入策略

```go
func writeTreeByMmap(filePath string, data []byte) error {
    // 1. 打开文件
    // 2. 调整文件大小
    // 3. 内存映射
    // 4. 拷贝数据
    // 5. 同步到磁盘
    // 6. 解除映射
}
```

**优势：**

- 减少内存拷贝
- 写入性能更高
- 大文件性能优势明显

### 8.4 数据修复机制

`fixTreeJSONData()`（`kernel/filesys/tree.go` L397-L500）：

```
1. XSS 防护：escapeAttributeValues()
   └─> 重新编码属性值，防止 XSS 攻击

2. Unicode 空字符清理：removeUnescapedUnicodeNull()
   └─> 移除未转义的 \u0000

3. 缺失属性补全
   └─> 确保必要属性存在
```

### 8.5 SQLite 配置优化

`initDBConnection()`（`kernel/treenode/blocktree.go` L95-L117）：

```go
dsn := util.BlockTreeDBPath + 
    "?_journal_mode=WAL" +                // WAL 模式
    "&_synchronous=OFF" +                 // 异步写入
    "&_mmap_size=2684354560" +            // 2.5GB mmap
    "&_cache_size=-20480" +               // 20MB 缓存
    "&_page_size=32768" +                 // 32KB 页大小
    "&_busy_timeout=7000"                 // 7秒超时
```

## 9. 并发变更处理

### 9.1 事务队列机制

`kernel/model/transaction.go`：

**架构设计：**

```
HTTP 请求
      │
      ▼
PerformTransactions() -> txQueue (channel 缓冲 7)
      │
      ▼
flushQueue() (goroutine 消费)
      │
      ▼
flushLock (互斥锁，串行执行)
      │
      ▼
performTx()
      │
      ├─> begin() 开启事务
      ├─> 执行操作（create/update/delete/move 等 50+ 种操作）
      └─> commit() 提交事务
```

**关键组件：**

```go
var (
    txQueue    = make(chan *Transaction, 7)  // 事务队列
    flushLock  = sync.Mutex{}               // 刷盘锁
    isFlushing = false                      // 刷盘状态
)
```

### 9.2 并发控制锁与协作范围

| 锁名称 | 位置 | 保护资源 | 锁范围 |
|--------|------|----------|--------|
| `flushLock` | `kernel/model/transaction.go` L64 | 事务串行执行 | 整个 flushTx 函数 |
| `tx.m` | `kernel/model/transaction.go` L1860 | 单个事务内部状态 | begin 获取，commit/rollback 释放 |
| `indexBlockTreeLock` | `kernel/treenode/blocktree.go` L522 | BlockTree 索引表读写 | IndexBlockTree / UpsertBlockTree 内部 |
| `initDatabaseLock` | `kernel/treenode/blocktree.go` L50 | 数据库初始化 | initDatabase 函数内部 |

**锁的层级关系：**

```
flushLock (事务级，最外层)
    │
    ├─> tx.m (事务内部状态保护)
    │
    └─> indexBlockTreeLock (BlockTree 索引操作，独立于事务锁)
```

> **注意**：`flushLock` 和 `indexBlockTreeLock` 是嵌套关系。`flushLock` 保护整个事务执行过程，在事务执行过程中会调用 `UpsertBlockTree()`，后者内部再获取 `indexBlockTreeLock`。由于事务串行执行，`indexBlockTreeLock` 是否冗余？不，`indexBlockTreeLock` 还被非事务路径调用（如 moveTree、rename 等）。

**事务状态机：**

```
    begin() → state=1 → 执行操作 → commit() → state=2
        ↑                │
        │                └─> 出错/panic → rollback() → state=3
        │
        └─> 释放 tx.m
```

状态值：

- `0`：未开始
- `1`：进行中（已获取 tx.m）
- `2`：已提交
- `3`：已回滚

### 9.3 跨进程文件锁

使用 `filelock` 包实现跨进程文件锁：

```go
data, err := filelock.ReadFile(filePath)  // 读锁
err := filelock.WriteFile(filePath, data)  // 写锁
```

### 9.4 批量操作优化

**大插入优化** — `processLargeInsert()`（`kernel/model/transaction.go` L500-L550）

```
操作数 > 32 时：
├─> 直接批量写入文件
├─> 跳过逐操作处理
└─> 提升性能
```

## 10. 缓存刷新与索引更新联动

### 10.1 缓存层设计

SiYuan 有**两套独立的 Ristretto 缓存**：

**1. 树数据缓存** — `kernel/cache/tree.go`：

```go
treeCache, _ = ristretto.NewCache(&ristretto.Config{
    NumCounters: 100000,
    MaxCost:     200 * 1024 * 1024,  // 200MB
    BufferItems: 64,
})
```

- Key：`rootID`（文档根块 ID）
- Value：原始 JSON 字节数据
- 作用：加速文档树加载，避免重复读取文件和解析 JSON

**2. 文档 IAL 缓存** — `kernel/cache/ial.go`：

```go
docIALCache, _ = ristretto.NewCache(&ristretto.Config{
    NumCounters: 100000,
    MaxCost:     200 * 1024 * 1024,  // 200MB
    BufferItems: 64,
})
```

- Key：文档路径（path）
- Value：文档 IAL 属性 map（id、title、updated 等）
- 作用：加速文件树列表渲染，避免逐个打开 .sy 文件

### 10.2 事务中的索引与缓存更新顺序

**块操作事务（create/update/delete/move 等）：**

```
performTx()
    │
    ├─> begin()
    │    └─> 初始化 tx.trees、tx.nodes
    │
    ├─> 执行 doXxx 操作
    │    └─> tx.writeTree(tree)
    │         ├─> tx.trees[tree.ID] = tree   // 暂存到事务上下文中
    │         └─> treenode.UpsertBlockTree(tree)  // ⚠️ 立即更新 BlockTree 索引
    │              └─> indexBlockTreeLock.Lock()
    │                  └─> SQLite DELETE + INSERT
    │
    └─> commit()
         └─> 遍历 tx.trees
              └─> writeTreeUpsertQueue(tree)
                   ├─> filesys.WriteTree(tree)  // 写入文件系统
                   │    ├─> mmap / writeFile
                   │    ├─> cache.SetTreeData()   // 更新树数据缓存
                   │    └─> afterWriteTree()
                   │         └─> cache.PutDocIAL()  // 更新 docIAL 缓存
                   └─> sql.UpsertTreeQueue(tree)  // 加入 SQL 索引队列（异步）
```

**关键发现：**

1. **BlockTree 索引在事务执行过程中就已更新**（不是在 commit 时才更新）
2. **文件系统写入和缓存更新在 commit 阶段才执行**
3. 若事务在操作阶段失败并 rollback，BlockTree 索引已变更，但**不会回滚**
4. 但文件系统尚未写入，因此不会出现数据不一致（文件系统是权威数据源）

### 10.3 文档移动时的索引与缓存更新

`MoveDocs()`（`kernel/model/file.go` L1296-L1356）：

```
MoveDocs()
    │
    ├─> FlushTxQueue()              // 先清空事务队列，确保移动前数据落盘
    │
    ├─> filelock.Rename()           // 文件系统重命名（原子操作）
    │
    ├─> moveTree(tree)              // 更新索引
    │    ├─> treenode.SetBlockTreePath(tree)  // 更新 BlockTree 索引
    │    │    ├─> RemoveBlockTreesByRootID()
    │    │    └─> IndexBlockTree()
    │    └─> sql.MoveTreeQueue(tree)         // 加入 SQL 移动队列
    │
    ├─> 递归处理子文档
    │
    └─> cache.ClearDocsIAL()        // ⚠️ 清空全部 docIAL 缓存
```

> **注意**：文档移动是**非事务操作**，直接操作文件系统和索引，不经过事务队列。移动前先 `FlushTxQueue()` 确保没有未提交的事务。

### 10.4 文档删除时的索引与缓存更新

`removeDoc()`（`kernel/model/file.go` L1562-L1642）：

```
removeDoc()
    │
    ├─> 备份到 history 目录
    ├─> 删除文件和子目录
    ├─> treenode.RemoveBlockTreesByPathPrefix()  // 删除 BlockTree 索引
    ├─> cache.RemoveDocIAL(ret.Path)             // 删除 docIAL 缓存
    ├─> cache.RemoveTreeData(ret.ID)              // 删除树数据缓存
    └─> task.AppendTask(task.DatabaseIndex, ...) // 异步 SQL 索引清理
```

### 10.5 缓存失效时机汇总

| 操作 | 触发位置 | treeCache | docIALCache | BlockTree 索引 |
|------|----------|-----------|-------------|----------------|
| 写入文档 | WriteTree | SetTreeData | PutDocIAL | UpsertBlockTree |
| 删除文档 | removeDoc | RemoveTreeData | RemoveDocIAL | RemoveBlockTreesByPathPrefix |
| 移动文档 | MoveDocs | 不直接操作 | ClearDocsIAL（全量清空） | SetBlockTreePath |
| 重命名文档 | RenameDoc | 不直接操作 | 不直接操作 | SetBlockTreePath |
| 重建索引 | Reindex | ClearTreeCache | ClearDocsIAL | 全量重建 |

> **注意**：移动文档时清空了**全部** docIAL 缓存（`ClearDocsIAL()`），而不是只清除受影响的文档。这是因为移动会导致大量文档的 HPath 变化，逐个清除效率更低。

### 10.6 索引恢复链路四场景对比

| 场景 | 触发时机 | 索引行为 | 修正方式 | 代码位置 |
|------|----------|----------|----------|----------|
| **普通加载** | 每次读文档 | 索引不存在直接返回错误 | 不修正，返回 ErrTreeNotFound | `LoadTreeByBlockID()` |
| **缺索引重建** | 特定加载路径（如打开文档） | 索引不存在则搜索文件系统 | 找到后建立索引（限频 3 秒 1 次） | `LoadTreeByBlockIDWithReindex()` + `indexTreeInFilesystem()` |
| **路径过期/全量修复** | 同步完成后执行一次（sync.Once） | 以文件系统为准全量订正 | 删冗余 + 补缺失，双向修正 | `checkIndex()` → `fixBlockTreeByFileSys()` |
| **事务提交失败** | commit 阶段写文件出错 | 操作阶段已写入索引，可能与文件不一致 | 不自动修正，错误码决定是否退出 | `commit()` → `writeTreeUpsertQueue()` |

**关键结论**：

1. **不存在自动按文件修正机制**：普通加载路径下，索引缺失就是缺失，不会去文件系统找
2. **缺索引重建有严格限频**：3 秒内最多触发一次，防止性能问题
3. **全量修正是被动触发的**：仅在数据同步完成后执行一次，平时不运行
4. **事务失败可能留下不一致**：操作阶段写了索引但 commit 失败，索引会比文件"新"，需等下次修改或全量校验修正

## 11. 异常恢复机制

### 11.1 Panic 恢复与事务回滚

`performTx()`（`kernel/model/transaction.go` L168-L178）：

```go
defer func() {
    if e := recover(); nil != e {
        logging.LogErrorf("PANIC RECOVERED: %v\n\t%s", e, logging.ShortStack())

        if 1 == tx.state.Load() {  // 只有进行中状态才回滚
            tx.rollback()
            return
        }
    }
}()
```

**回滚行为：**

- `tx.rollback()` 仅**清空内存中的 trees 和 nodes 引用**
- **不会回滚**已经更新的 BlockTree 索引
- **不会回滚**文件系统（因为 commit 前还没写文件）
- 释放 `tx.m` 互斥锁
- 最终由 `defer` 释放 `flushLock`

> **关键点**：事务回滚是"软回滚"，只回滚内存状态，不回滚已持久化的数据。由于文件系统写入在 commit 阶段才执行，所以操作阶段失败不会导致文件系统损坏。

### 11.2 事务失败分类与处理

`flushTx()`（`kernel/model/transaction.go` L83-L125）中的错误码处理：

| 错误码 | 名称 | 处理方式 |
|--------|------|----------|
| 0 | TxErrCodeBlockNotFound | 推送错误消息给前端，不退出 |
| 1 | TxErrCodeDataIsSyncing | 推送提示消息，不退出 |
| 2 | TxErrCodeWriteTree | **致命错误**，LogFatalf 退出程序 |
| 3 | TxErrHandleAttributeView | 推送错误消息，记录日志，不退出 |
| 4 | TxErrCodePushMsg | 推送错误消息，不退出 |

**失败恢复顺序：**

```
事务执行失败
    │
    ├─> 操作执行中失败（doXxx 返回错误）
    │    └─> tx.rollback() → 清空内存引用，释放 tx.m → 返回 TxErr
    │
    ├─> Panic 触发
    │    └─> 检查 state==1 → tx.rollback() → 返回
    │
    └─> commit 失败
         └─> 直接返回错误，不回滚（已写入的文件无法撤回）
              │
              └─> 根据错误码决定是否 Fatal 退出
```

> **重要误判排除**：commit 阶段失败时，**部分文件可能已经写入成功**，不会回滚。这意味着如果 commit 中途失败，可能出现部分文档已更新、部分未更新的情况。由于 BlockTree 索引在操作阶段已更新，索引与文件系统可能暂时不一致。
>
> **索引不会自动修正**：下一次加载**不会**自动以文件系统为准修正索引。不一致状态会持续到：
> - 文档内容修改触发索引更新
> - 触发 `LoadTreeByBlockIDWithReindex` 重建
> - 同步完成后执行 `checkIndex` 全量校验

### 11.3 BlockTree 数据库损坏恢复

`execInsertBlocktrees()`（`kernel/treenode/blocktree.go` L618-L649）：

```go
if strings.Contains(err.Error(), "database disk image is malformed") {
    initDatabase(true)  // 强制重建数据库
    logging.LogFatalf(logging.ExitCodeUnavailableDatabase, 
        "database disk image [%s] is malformed, please restart SiYuan kernel to rebuild it", 
        util.BlockTreeDBPath, err)
}
```

**恢复流程：**

1. 检测到 "database disk image is malformed" 错误
2. 调用 `initDatabase(true)` 重新建表（**只建表，不重建数据**）
3. `LogFatalf` 退出程序
4. 用户重启后，系统会自动重新索引所有文档

> **关键点**：损坏后程序**直接退出**，不尝试在线恢复。用户需重启程序，启动时会重建索引。这是因为索引损坏后继续运行可能导致更多问题。

### 11.4 损坏 .sy 文件处理

`docIAL()`（`kernel/model/file.go` L107-L141）：

```go
func (box *Box) docIAL(p string) (ret map[string]string) {
    // 1. 先查缓存
    ret = cache.GetDocIAL(p)
    if nil != ret {
        return ret
    }

    // 2. 读取文件
    filePath := filepath.Join(util.DataDir, box.ID, p)
    ret = filesys.DocIAL(filePath)
    if 1 > len(ret) {  // Properties 不存在或为空
        logging.LogWarnf("properties not found in file [%s]", filePath)
        box.moveCorruptedData(filePath)  // 移走损坏文件
        return nil
    }

    // 3. 写入缓存
    cache.PutDocIAL(p, ret)
    return ret
}
```

`moveCorruptedData()`（`kernel/model/file.go` L129-L141）：

```go
func (box *Box) moveCorruptedData(filePath string) {
    base := filepath.Base(filePath)
    to := filepath.Join(util.WorkspaceDir, "corrupted", 
        time.Now().Format("2006-01-02-150405"), box.ID, base)
    
    if copyErr := filelock.Copy(filePath, to); nil != copyErr {
        return  // 复制失败则保留原文件
    }
    if removeErr := filelock.Remove(filePath); nil != removeErr {
        return  // 删除失败则保留原文件
    }
}
```

**触发条件：**

- 读取 .sy 文件时，无法解析出 `Properties` 字段
- 或 `Properties` 为空 map

**处理方式：**

1. **先复制**到 `workspace/corrupted/日期/boxID/` 目录（保留备份）
2. **再删除**原文件
3. 若复制失败，保留原文件（不删除）
4. 记录警告日志

> **排查提示**：发现文档突然消失，先检查 `workspace/corrupted/` 目录。文件名保留原 ID，可手动恢复。

### 11.5 孤儿文档（父文档缺失）自动补全

`LoadTreeByData()`（`kernel/filesys/tree.go` L170-L196）：

```go
// 构造 HPath 时，遍历父路径
for i := range parts {
    parentAbsPath := strings.Join(parts[:i+1], "/") + ".sy"
    parentDocIAL := DocIAL(parentAbsPath)
    
    if 1 > len(parentDocIAL) {
        // 子文档缺失父文档时自动补全
        parentTree := treenode.NewTree(boxID, parentPath, 
            hPathBuilder.String()+"Untitled", "Untitled")
        
        if _, writeErr := WriteTree(parentTree); nil != writeErr {
            logging.LogErrorf("rebuild parent tree [%s] failed: %s", 
                parentAbsPath, writeErr)
        } else {
            logging.LogInfof("rebuilt parent tree [%s]", parentAbsPath)
            treenode.UpsertBlockTree(parentTree)  // 更新索引
        }
        hPathBuilder.WriteString("Untitled/")
        continue
    }
    // ...
}
```

**触发条件：**

- 加载子文档时，向上查找父文档
- 发现某个层级的父文档 .sy 文件不存在或 Properties 为空

**处理方式：**

1. 自动创建一个标题为 "Untitled" 的父文档
2. 调用 `WriteTree()` 写入文件系统
3. 调用 `UpsertBlockTree()` 更新 BlockTree 索引
4. HPath 中使用 "Untitled" 作为缺失父文档的名称

> **排查提示**：发现文档树中出现大量 "Untitled" 文档，可能是父文档文件丢失后自动补全的结果。检查 `workspace/corrupted/` 目录，或查看日志中的 `rebuilt parent tree` 记录。

### 11.6 临时文件清理

`Box.Ls()`（`kernel/model/box.go`）相关逻辑：

**清理规则：**

- `.tmp` 后缀的文件
- 文件存在时间超过 30 分钟
- 自动删除

> **排查提示**：大量 `.tmp` 文件残留可能意味着写入过程频繁中断。检查磁盘空间、文件系统权限或是否有其他进程锁定文件。

### 11.7 空文档规范化

文档加载时，如果文档没有内容节点，会自动添加一个空段落节点，保证文档最少有一个可编辑的块。

## 12. 潜在风险分析

### 12.1 并发风险

1. **事务队列溢出**
   - 风险：`txQueue` 仅缓冲 7 个事务
   - 场景：高并发写入时可能阻塞发送方
   - 影响：前端操作卡顿、延迟增加
   - 实际情况：由于 `flushLock` 串行执行，队列满时 `PerformTransactions` 会阻塞

2. **BlockTree 索引与文件系统不一致窗口**
   - 风险：事务操作阶段先更新 BlockTree 索引，commit 阶段才写文件
   - 场景：操作阶段成功但 commit 阶段前进程崩溃
   - 影响：BlockTree 索引中有记录但文件系统中没有（或反之）
   - 恢复：不会自动修正，需依赖 checkIndex 全量校验或手动重建索引

3. **锁粒度粗**
   - 风险：`flushLock` 是全局锁，所有事务串行执行
   - 场景：编辑不同文档也会互斥等待
   - 影响：多用户并发编辑时吞吐量受限

### 12.2 数据一致性风险

1. **commit 部分失败**
   - 风险：commit 阶段遍历多个树写入，中途失败则已写入的不会回滚
   - 场景：一个事务修改了多个文档，写入第二个文档时磁盘满
   - 影响：部分文档已更新，部分未更新
   - 严重性：中等，因为每个文档是独立的

2. **移动文档中途失败**
   - 风险：文件系统重命名是原子的，但索引更新可能失败
   - 场景：重命名成功但 BlockTree 更新失败
   - 影响：文件在新位置，但索引还指向旧位置
   - 恢复：重建索引即可

3. **缓存与数据不一致**
   - 风险：文件写入成功但缓存更新失败
   - 场景：`cache.SetTreeData()` 内部出错（虽然概率极低）
   - 影响：读取到旧数据
   - 恢复：清除缓存或重启

### 12.3 性能风险

1. **docIAL 缓存全量失效**
   - 风险：移动文档时调用 `ClearDocsIAL()` 清空全部缓存
   - 场景：频繁移动文档
   - 影响：文件树列表加载变慢，需要逐个打开 .sy 文件

2. **大事务阻塞**
   - 风险：单个事务包含大量操作，会长时间占用 `flushLock`
   - 场景：批量粘贴、批量删除
   - 影响：其他操作排队等待

3. **子文档级联加载**
   - 风险：移动或重命名文档时需要遍历所有子文档
   - 场景：深层级、多子文档的大树移动
   - 影响：耗时较长，UI 可能卡顿

### 12.4 安全风险

1. **XSS 攻击**
   - 风险：属性值未正确转义可能导致 XSS
   - 修复：`escapeAttributeValues()` 在加载时重新编码
   - 历史漏洞：GHSA-ff66-236v-p4fg

2. **路径遍历**
   - 风险：路径参数未校验可能访问非预期文件
   - 修复：`ast.IsNodeIDPattern()` 校验 ID 格式
   - 保护：文件操作都限制在 data 目录内

### 12.5 恢复能力边界

| 故障类型 | 是否可自动恢复 | 数据丢失风险 | 恢复方式 |
|---------|--------------|------------|----------|
| .sy 文件损坏（Properties 丢失） | 是（自动移走） | 有（文件损坏） | 从 corrupted 目录手动恢复 |
| BlockTree 索引损坏 | 否（需重启） | 无（索引可重建） | 删除 db 文件后重启 |
| BlockTree 索引与文件不一致 | 否（不自动修正） | 无 | 手动重建索引或等同步后 checkIndex |
| 索引缺失（文件存在、索引无） | 特定场景可（带重建加载） | 无 | LoadTreeByBlockIDWithReindex 或全量重建 |
| 事务执行中 panic | 是（自动回滚） | 低（只回滚内存状态） | 自动恢复 |
| commit 中途失败 | 否 | 中（部分写入） | 手动检查一致性 |
| 磁盘空间不足写入失败 | 否 | 低 | 释放空间后重试 |
| 父文档缺失 | 是（自动补全） | 低（补全空白文档） | 自动恢复，可能需要手动编辑 |

## 13. 问题排查方法

### 13.1 日志分析关键线索

**关键日志关键词：**

| 日志关键词 | 含义 | 严重程度 |
|-----------|------|----------|
| `op tx [xxxxms]` | 事务执行超过 2 秒 | 警告 |
| `database disk image is malformed` | BlockTree 数据库损坏 | 致命 |
| `PANIC RECOVERED` | 事务执行中发生 panic 已恢复 | 警告 |
| `properties not found in file` | .sy 文件缺少 Properties，可能已损坏 | 警告 |
| `moved corrupted data file` | 已将损坏文件移至 corrupted 目录 | 提示 |
| `rebuilt parent tree` | 自动补全了缺失的父文档 | 提示 |
| `transaction failed` | 事务执行失败 | 警告 |
| `reinitialized database` | 数据库已重新初始化 | 提示 |
| `searching tree on filesystem` | 正在文件系统中搜索丢失的索引 | 提示 |
| `reindexed tree by filesystem` | 已通过文件系统重建索引 | 提示 |
| `tree not found on filesystem` | 文件系统中也找不到树 | 警告 |
| `exist more than one tree duplicated` | 检测到重复的树 ID | 警告 |

**日志位置：**

- Windows：`%APPDATA%\siyuan\log\`
- macOS：`~/Library/Application Support/siyuan/log/`
- Linux：`~/.config/siyuan/log/`

### 13.2 常见问题排查

#### 问题 1：文档树不显示某文档

**排查步骤（按顺序）：**

1. **检查文件系统**：确认 `.sy` 文件是否存在
   ```
   data/笔记本ID/路径/文档ID.sy
   ```

2. **检查 BlockTree 索引**：
   ```sql
   SELECT * FROM blocktrees WHERE root_id = '文档ID' AND type = 'd';
   ```

3. **检查是否被标记为损坏**：
   - 查看 `workspace/corrupted/` 目录
   - 搜索日志中是否有 `properties not found` 或 `moved corrupted`

4. **检查是否为隐藏文档**：
   - 文档 IAL 中 `hidden` 属性是否为 `true`

5. **尝试重建索引**：
   - 设置 → 搜索 → 重建索引
   - 或调用 API：`POST /api/filetree/reindex`

> **常见误判排除**：
> - 文档是否在已关闭的笔记本中？
> - 文档是否被移到了其他笔记本？
> - 是否是子文档，父文档被隐藏/删除？

#### 问题 2：移动文档失败

**排查步骤：**

1. **检查权限**：
   - 当前用户角色是否为 Reader 或 Visitor（只读）
   - 发布模式下是否有编辑权限

2. **检查深度限制**：
   - 目标路径深度 + 子文档深度是否超过 7 层
   - 可在 `conf.json` 中启用 `fileTree.allowCreateDeeper`

3. **检查文件锁**：
   - 是否有其他进程占用文件
   - 检查是否有未完成的同步操作

4. **检查事务队列**：
   - 移动前会 `FlushTxQueue()`，若事务队列卡住会导致移动等待

> **常见误判排除**：
> - 移动到自身父目录（不移动是正常的）
> - 目标位置已有同名文档（ID 相同不会冲突，名称相同可能冲突）

#### 问题 3：BlockTree 数据库损坏

**症状：**

- 文档树空白
- 搜索全部失效
- 日志出现 `database disk image is malformed`

**恢复步骤：**

1. 关闭思源笔记
2. 删除 `storage/blocktrees.db` 文件
3. 重新启动，程序会自动重建索引
4. 或使用 `POST /api/filetree/reindex` API 重建

> **注意**：BlockTree 是索引数据库，损坏不会丢失数据，只是查询变慢。所有数据都在 `.sy` 文件中。

#### 问题 4：缓存不一致

**症状：**

- 文档列表显示旧标题
- 修改后列表没刷新

**排查步骤：**

1. 刷新页面（前端缓存）
2. 调用 `POST /api/filetree/clearCache` 清除缓存
3. 检查 docIALCache 是否已失效
4. 检查 treeCache 是否已失效

> **常见误判排除**：
> - 是前端缓存还是后端缓存？
> - 移动文档会清空全部 docIAL 缓存，列表刷新慢是正常的

#### 问题 5：BlockTree 索引与文件系统不一致

**症状：**

- 通过 ID 能搜到块，但打开文档报错"找不到"
- 或文档存在但文件树列表不显示
- 搜索结果指向错误的路径

**排查步骤：**

1. **确认是索引问题**：
   - 用文件管理器确认 `.sy` 文件是否存在于预期路径
   - 用 SQL 查询 BlockTree 索引：`SELECT * FROM blocktrees WHERE root_id = '文档ID'`

2. **判断不一致方向**：
   - 文件有、索引无 → 索引缺失
   - 文件无、索引有 → 索引冗余
   - 两边都有但路径/标题对不上 → 路径过期

3. **修复方式**：
   - 单文档：打开文档触发 `LoadTreeByBlockIDWithReindex`（如果支持）
   - 全量：设置 → 搜索 → 重建索引
   - 或调用 API：`POST /api/filetree/reindex`

4. **日志关键词**：
   - `searching tree on filesystem` — 正在文件系统中搜索丢失的树
   - `reindexed tree by filesystem` — 已通过文件系统重建索引
   - `tree not found on filesystem` — 文件系统中也找不到

> **常见误判排除**：
> - 文档在已关闭的笔记本中（索引会被清除）
> - 文档被移到了其他笔记本
> - 是权限问题还是索引问题（发布模式下可能隐藏）
> - 普通加载不会自动修复索引，别等"自动好"

#### 问题 6：大量 "Untitled" 文档突然出现

**可能原因：**

- 父文档文件损坏被移走，自动补全生成
- 同步冲突导致父文档丢失

**排查步骤：**

1. 检查 `workspace/corrupted/` 目录
2. 搜索日志中的 `rebuilt parent tree` 记录
3. 确认是否有同步操作正在进行

#### 问题 7：事务执行缓慢

**症状：**

- 编辑操作延迟高
- 日志中频繁出现 `op tx [xxxxms]`

**可能原因：**

1. 大文档操作（内容很多）
2. 事务队列堆积
3. 磁盘 IO 慢
4. BlockTree 索引锁竞争

**排查步骤：**

1. 查看慢事务日志（超过 2000ms）
2. 检查是否有大文档在操作
3. 检查磁盘 IO 性能
4. 检查是否有大量并发操作

### 13.3 调试工具与 API

| 工具 | 类型 | 用途 |
|------|------|------|
| `POST /api/filetree/reindex` | API | 重建文件树索引 |
| `POST /api/filetree/clearCache` | API | 清除文件树缓存 |
| `POST /api/system/getConf` | API | 查看系统配置 |
| `workspace/corrupted/` | 目录 | 损坏文件存放处 |
| `storage/blocktrees.db` | 文件 | BlockTree 索引数据库 |
| `data/*/.sy` | 文件 | 实际文档数据 |

### 13.4 数据一致性检查方法

**快速验证索引与文件一致性：**

1. 统计文件系统中 .sy 文件数量
2. 统计 BlockTree 中文档块（type='d'）数量
3. 两者应大致相等（排除损坏、隐藏等）

```sql
-- 统计 BlockTree 中的文档数量
SELECT COUNT(*) FROM blocktrees WHERE type = 'd';
```

若差异较大，可能需要重建索引。

### 13.5 避免误判的排查原则

1. **先文件，后索引**：文件系统是权威数据源，索引只是加速查询
2. **先日志，后猜测**：先查日志确认错误类型，不要盲目尝试
3. **先备份，后操作**：任何修复操作前先备份数据目录
4. **先单例，后并发**：关闭其他设备同步，排除并发干扰
5. **区分缓存层级**：前端缓存、后端缓存、数据库缓存，逐层排查

## 14. 总结

SiYuan 文档树管理采用了**三层持久化 + 两级缓存**的架构设计：

**三层持久化：**

1. **文件系统**（权威数据源）：`.sy` JSON 文件，保证数据可移植性
2. **SQLite BlockTree 索引**（二级索引）：加速块位置查找，可重建
3. **SQL 搜索索引**（三级索引）：全文搜索加速，异步更新

**两级缓存：**

1. **treeCache**：原始 JSON 数据缓存，200MB Ristretto
2. **docIALCache**：文档属性缓存，200MB Ristretto

### 核心机制总结

| 机制 | 实现方式 | 关键代码 |
|------|----------|----------|
| 事务串行化 | channel 队列 + flushLock 互斥 | `kernel/model/transaction.go` flushTx |
| 索引更新时机 | 操作阶段更新 BlockTree，commit 阶段写文件 | `kernel/model/transaction.go` tx.writeTree |
| 缓存更新时机 | 文件写入成功后更新缓存 | `kernel/filesys/tree.go` WriteTree |
| 损坏文件处理 | 先复制备份，再删除原文件 | `kernel/model/file.go` moveCorruptedData |
| 孤儿文档补全 | 加载子文档时自动补全缺失的父文档 | `kernel/filesys/tree.go` LoadTreeByData |
| 事务回滚 | 仅清空内存引用，不回滚索引 | `kernel/model/transaction.go` tx.rollback |
| 文档移动 | 非事务操作，移动前清空事务队列 | `kernel/model/file.go` MoveDocs |
| 缺索引重建 | 索引缺失时搜索文件系统重建（限频） | `kernel/model/tree.go` indexTreeInFilesystem |
| 全量索引修复 | 同步后 checkIndex 校验，以文件为准订正 | `kernel/model/index_fix.go` fixBlockTreeByFileSys |

### 设计亮点

1. **文件系统优先**：以文件系统为权威数据源，索引和缓存均可重建，数据安全性高
2. **双写策略**：mmap 优先，writeFile 降级，兼顾性能和兼容性
3. **软事务回滚**：操作阶段修改内存状态，commit 才落盘，简化回滚逻辑
4. **多级异常恢复**：从 panic 恢复、损坏文件隔离到数据库重建，多维度保障系统稳定性
5. **向前兼容的加载修复**：加载时自动修复 XSS、Unicode 空字符等历史问题

### 权衡与取舍

- **一致性 vs 性能**：选择最终一致性，优先保证写入性能。索引先更新、文件后写入，存在短暂不一致窗口
- **粗粒度锁 vs 细粒度锁**：选择全局串行事务，简化实现，牺牲并发性能
- **全量缓存失效 vs 精确失效**：移动文档时选择全量清空 docIAL 缓存，简化实现
- **被动索引修复 vs 主动修正**：选择被动修复（同步后 checkIndex + 特定场景重建），避免频繁扫描文件系统

这是一个经过生产验证的、以**本地优先和数据安全**为核心设计原则的文档树管理实现。
