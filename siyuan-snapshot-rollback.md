# SiYuan 历史快照与版本回滚代码实现深度分析

> **版本依据**：基于 SiYuan v3.6.5 内核源码分析
> - 内核版本：Ver = "3.6.5" (`kernel/util/working.go:47`)
> - Go 版本：1.25.4 (`kernel/go.mod:3`)
> - 数据库版本：DatabaseVer = "20220501" (`kernel/util/runtime.go:90`)
> - dejavu 仓库库：v0.0.0-20260411080619-1de6197a80f4 (`kernel/go.mod:66`)
> - 分析代码行覆盖：约 5,200 行
>
> **引用说明**：全文代码位置采用「仓库相对路径 + 行号范围」格式，例如 `kernel/model/history.go:54-60` 表示仓库根目录下 `kernel/model/history.go` 文件的第 54 至 60 行。单一行号如 `kernel/model/transaction.go:440` 表示精确行定位。

---

## 一、系统架构总览

SiYuan 采用 **双层版本管理体系**，L1 层面向文件级快速回滚，L2 层面向工作区级灾难恢复。

| 层级 | 名称 | 存储位置 | 触发方式 | 粒度 | 核心引擎 |
|------|------|---------|---------|------|---------|
| L1 | 文件历史<br/>(File History) | `workspace/history/` | 定时轮询 + 事务事件 | 单文档 / 资源 / AV 数据库 | 文件系统直接复制 |
| L2 | 数据仓库快照<br/>(Repo Snapshot) | `workspace/repo/` | 手动 + 同步前 + 回滚前 | 整个工作区所有文件 | dejavu 内容寻址存储 |

### 1.1 核心模块职责矩阵

| 模块 | 仓库相对路径 | 核心职责 |
|------|-------------|---------|
| 历史调度器 | `kernel/model/history.go` | 定时生成、四类回滚、历史索引、过期清理 |
| 仓库管理器 | `kernel/model/repository.go` | dejavu 封装、快照 CRUD、Diff 计算、Checkout、云同步 |
| 事务协调器 | `kernel/model/transaction.go` | FlushTxQueue、事务内历史触发 |
| API 层 - 历史 | `kernel/api/history.go` | 10 个 HTTP 端点、参数校验 |
| API 层 - 仓库 | `kernel/api/repo.go` | 23 个 HTTP 端点、权限检查 |
| 历史数据库 | `kernel/sql/history.go` | FTS5 全文索引、查询、批量插入 |
| 历史队列 | `kernel/sql/queue_history.go` | 异步索引、失败自愈重建 |
| 路由注册 | `kernel/api/router.go` | 端点注册 + 中间件链 |
| 前端主界面 | `app/src/history/history.ts` | 三 Tab 面板、分页、筛选 |
| 前端差异界面 | `app/src/history/diff.ts` | 双快照 Diff 渲染、左右编辑器 |
| 前端单文档历史 | `app/src/history/doc.ts` | 右键菜单「文档历史」专用视图 |

---

## 二、快照生成机制

### 2.1 L1 文件历史生成

#### 2.1.1 触发入口

**定时触发**：
```go
// kernel/model/history.go:54-60
func AutoGenerateFileHistory() {
    historyTicker = time.NewTicker(time.Minute * time.Duration(Conf.Editor.GenerateHistoryInterval))
    for range historyTicker.C {
        task.AppendTask(task.HistoryGenerateFile, GenerateFileHistory)
    }
}
```
- 启动位置：`kernel/main.go:52`、`kernel/mobile/kernel.go:229`、`kernel/harmony/kernel.go:69`
- 间隔配置：`Conf.Editor.GenerateHistoryInterval`（分钟）

**事务触发**（操作级即时历史）：

| 操作类型 | 触发位置 | 历史操作符 |
|---------|---------|-----------|
| 文档移动 | `kernel/model/transaction.go:440` | `HistoryOpUpdate` |
| 标题调整 | `kernel/model/heading.go:187` | `HistoryOpUpdate` |
| 格式清理 | `kernel/model/format.go:42` | `HistoryOpFormat` |
| 大纲重排 | `kernel/model/outline.go:85/136/176` | `HistoryOpOutline` |
| 标签重命名 | `kernel/model/tag.go:51/158` | `HistoryOpReplace` |
| 书签替换 | `kernel/model/bookmark.go:50/122` | `HistoryOpReplace` |
| 搜索替换 | `kernel/model/search.go:516` | `HistoryOpReplace` |
| 删除操作 | `kernel/model/file.go:1568` | `HistoryOpDelete` |
| 挂载删除 | `kernel/model/mount.go:141` | `HistoryOpDelete` |
| AV 清理 | `kernel/model/attribute_view.go:59/97` | `HistoryOpClean` |
| 资源清理 | `kernel/model/assets.go:790/860` | `HistoryOpClean` |
| AV 更新 | `kernel/model/attribute_view.go:3811` | `HistoryOpUpdate` |
| 资源替换 | `kernel/model/assets.go:945` | `HistoryOpReplace` |

**历史操作符常量**（`kernel/model/history.go:812-819`）：
```go
HistoryOpClean   = "clean"
HistoryOpUpdate  = "update"
HistoryOpDelete  = "delete"
HistoryOpFormat  = "format"
HistoryOpSync    = "sync"
HistoryOpReplace = "replace"
HistoryOpOutline = "outline"
```

#### 2.1.2 生成流程

**主流程入口**：`GenerateFileHistory()` (`kernel/model/history.go:62-90`)

```
GenerateFileHistory()
  ├─> FlushTxQueue()                     // 事务屏障：确保文件落盘
  ├─> 遍历所有已打开笔记本
  │    └─> box.generateDocHistory0()     // 每本笔记本生成文档历史
  ├─> generateAssetsHistory()            // 资源文件历史
  └─> 清理老版本历史目录（向前兼容）
```

**文档历史生成**：`generateDocHistory0()` (`kernel/model/history.go:720-810`)

```
generateDocHistory0(boxID)
  ├─> 计算 boxLatestHistoryTime[boxID]（上次快照时间）
  ├─> 获取所有 .sy 文件列表
  ├─> recentModifiedDocs()              // 增量筛选
  │    └─> 通过 cache.LastUpdateTime 比较时间戳
  ├─> getHistoryDir(HistoryOpUpdate)    // 创建历史目录
  │    └─> 命名：YYYY-MM-DD-HHMMSS-update
  └─> 遍历修改的文档
       ├─> 复制 .sy 文件到历史目录（保持原路径）
       └─> generateAvHistoryInTree()    // 同步复制关联的 AV 数据库 JSON
```

**资源历史生成**：`generateAssetsHistory()` (`kernel/model/history.go:836-898`)

```
generateAssetsHistory()
  ├─> cache.FilterAssets(prevUpdateTime) // 增量筛选资源
  ├─> getHistoryDir(HistoryOpUpdate)
  └─> 逐个复制到 historyDir/assets/ 目录（保持原 yyyy/mm 结构）
```

**AV 数据库同步**：`generateAvHistoryInTree()` (`kernel/model/history.go:702-718`)
- 遍历 Tree 中所有 `NodeAttributeView` 节点
- 提取 `AttributeViewID`
- 复制 `storage/av/{id}.json` 到历史目录

#### 2.1.3 历史目录索引入库

**入口**：`indexHistoryDir()` (`kernel/model/history.go:900-981`)

```
indexHistoryDir(dir)
  ├─> 验证操作类型（validOps 白名单）
  ├─> 遍历目录所有文件
  │    ├─> .sy 文档 → 解析 title + content
  │    ├─> assets/ 资源 → 取文件名
  │    └─> storage/av/*.json → 解析数据库内容
  └─> sql.IndexHistoriesQueue(histories) // 批量入队列
```

**索引数据库结构**（`kernel/sql/history.go:29-37`）：
```go
type History struct {
    ID      string  // 块ID / 资源ID / AV ID
    Type    int     // 0=DocName 1=Doc 2=Asset 3=DocID 4=Database
    Op      string  // update/delete/format/...
    Title   string  // 文档标题 / 文件名
    Content string  // 全文内容（FTS 索引用）
    Created string  // Unix 时间戳
    Path    string  // 历史目录相对路径
}
```

**FTS5 全文索引**：`histories_fts_case_insensitive` 虚拟表
- 支持按标题、内容、操作类型快速检索
- 大小写不敏感

### 2.2 L2 数据仓库快照生成

#### 2.2.1 触发入口

| 触发场景 | 代码位置 | 备注 |
|---------|---------|------|
| 手动创建 | `kernel/model/repository.go:1162` | `IndexRepo(memo)` |
| 同步前索引 | `kernel/model/repository.go:1942` | `indexRepoBeforeCloudSync()` |
| 回滚前备份 | `kernel/model/repository.go:874` | "Backup before checkout" |
| 初始化索引 | `kernel/model/repository.go:830` | "[Init] Init local data repo" |

**手动创建 API**：`/api/repo/createSnapshot` → `createSnapshot()` (`kernel/api/repo.go`)

#### 2.2.2 核心流程

**IndexRepo 流程** (`kernel/model/repository.go:1162-1205`)：

```
IndexRepo(memo)
  ├─> 前置检查
  │    ├─> len(Conf.Repo.Key) > 0       // 密钥存在性
  │    └─> memo 非空（去除不可见字符）
  ├─> newRepository()                   // 创建 dejavu.Repo 实例
  ├─> PushEndlessProgress               // 开启无限进度条
  ├─> repo.Latest()                     // 获取当前最新快照 ID
  ├─> FlushTxQueue()                    // 事务屏障
  ├─> repo.Index(memo, true, ctx)       // dejavu 底层执行索引
  │    ├─> 扫描工作区所有文件
  │    ├─> SHA-256 分块哈希
  │    ├─> AES-256 加密（32字节密钥）
  │    ├─> 去重存储（Content-Addressed）
  │    └─> 写入 Index Log
  └─> 结果通知
       ├─> 若 ID 变化 → Language(147) "创建数据仓库快照耗时 %.1fs"
       └─> 若 ID 不变 → Language(148) "数据仓库快照未变化，耗时 %.1fs"
```

#### 2.2.3 dejavu 底层机制

**dejavu 导入**：`kernel/model/repository.go:49-52`
```go
import (
    "github.com/siyuan-note/dejavu"
    "github.com/siyuan-note/dejavu/cloud"
    "github.com/siyuan-note/dejavu/entity"
)
```

**核心概念**：
- **内容寻址存储（CAS）**：文件内容哈希作为地址，相同内容只存一份
- **分块存储（Chunking）**：大文件切块存储，增量同步效率高
- **AES-256 加密**：所有块数据加密后存储，密钥为 32 字节
- **Index Log**：每次索引生成一条日志记录，包含文件列表、时间、备注

**错误码体系**（dejavu 导出错误）：

| 错误变量 | 含义 | 处理方式 |
|---------|------|---------|
| `dejavu.ErrNotFoundIndex` | 索引不存在 | 返回空列表 |
| `dejavu.ErrRepoFatal` | 仓库致命错误 | 自动重试 + 记录错误计数 |
| `dejavu.ErrLockCloudFailed` | 云端锁获取失败 | 计划下次同步 |
| `dejavu.ErrCloudLocked` | 云端已锁定 | 等待释放 |
| `dejavu.ErrCloudBackupCountExceeded` | 云备份数量超限 | Language(84/154) 用户提示 |
| `dejavu.ErrCloudStorageSizeExceeded` | 云存储空间超限 | 显示配额 + 升级提示 |
| `cloud.ErrCloudObjectNotFound` | 云对象不存在 | 视为首次同步 |

---

## 三、差异对比机制

### 3.1 L2 仓库快照差异计算

**API 入口**：`/api/repo/diffRepoSnapshots` → `diffRepoSnapshots()` (`kernel/api/repo.go:161-192`)

**参数**：`selectId` 数组（最多 2 个快照 ID）

#### 3.1.1 差异算法

**核心函数**：`DiffRepoSnapshots()` (`kernel/model/repository.go:420-519`)

```
DiffRepoSnapshots(selectIds)
  ├─> 若只有 1 个 ID → 与最新快照对比（左=选中，右=最新）
  ├─> 若有 2 个 ID → 两快照互相比对
  ├─> repo.DiffIndex(left, right)       // dejavu 底层计算差异
  │    └─> 返回 LeftRightDiff 结构
  └─> parseTitleInSnapshot()            // 为每个 .sy 文件解析标题
       ├─> repo.GetFileContent(id)
       ├─> luteEngine.FormatRenderer   // 纯文本渲染（性能考虑）
       └─> 提取标题
```

**LeftRightDiff 结构** (`kernel/model/repository.go:398-418`)：

```go
type LeftRightDiff struct {
    LeftIndex    *DiffIndex   // 左快照元数据
    RightIndex   *DiffIndex   // 右快照元数据
    AddsLeft     []*DiffFile  // 左有右无 → 右版本删除的文件
    UpdatesLeft  []*DiffFile  // 修改前版本（左）
    UpdatesRight []*DiffFile  // 修改后版本（右）
    RemovesRight []*DiffFile  // 右有左无 → 右版本新增的文件
}
```

**DiffFile 字段**：`fileID`, `title`, `path`, `hSize`（人类可读大小）, `updated`

#### 3.1.2 前端差异渲染

**入口文件**：`app/src/history/diff.ts`

**渲染流程**（`showDiff()`）：

```
用户在前端选择 1-2 个快照
  ↓
调用 /api/repo/diffRepoSnapshots 获取差异数据
  ↓
渲染三栏布局：
  ├─ 左侧：差异分类列表
  │    ├─> Updates（修改，UpdatesLeft/UpdatesRight 合并去重）
  │    ├─> Adds（新增，RemovesRight）
  │    └─> Removes（删除，AddsLeft）
  ├─ 中间：左编辑器（修改前 / 删除前）
  └─ 右侧：右编辑器（修改后 / 新增后）
  ↓
点击文件 → 调 /api/repo/openRepoSnapshotFile 加载内容
  ↓
按文件类型分支渲染：
  ├─> 图片/音视频 → renderAssetsPreview() 媒体预览
  ├─> displayInText → textarea 纯文本
  └─> .sy 文档 → Protyle 富文本编辑器（禁用态）
```

**Protyle 双编辑器** (`app/src/history/diff.ts:39-82`)：
- 左/右各一个 `Protyle` 实例
- `disabledProtyle()` 禁用编辑
- 配置：`action: [Constants.CB_GET_HISTORY]`
- 隐藏：背景、行号、面包屑、文档名

### 3.2 L1 文件历史差异

L1 层不提供精确的 Diff 计算，采用**列表式时间线**展示：
- 按历史目录分组
- 每个操作一条记录（含操作类型、时间、标题、路径）
- 用户点击单条记录 → 加载完整文档内容
- **不支持双版本并列对比**（需切换到 L2 数据快照视图）

---

## 四、版本选择与恢复操作

### 4.1 L1 文件历史回滚

#### 4.1.1 四种回滚类型

| 回滚类型 | API 端点 | 核心函数 | 复杂度 |
|---------|---------|---------|--------|
| 文档回滚 | `/api/history/rollbackDocHistory` | `RollbackDocHistory()` | ⭐⭐⭐⭐⭐ |
| 资源回滚 | `/api/history/rollbackAssetsHistory` | `RollbackAssetsHistory()` | ⭐⭐ |
| 笔记本回滚 | `/api/history/rollbackNotebookHistory` | `RollbackNotebookHistory()` | ⭐⭐⭐ |
| AV 数据库回滚 | `/api/history/rollbackAttributeViewHistory` | `RollbackAttributeViewHistory()` | ⭐⭐ |

#### 4.1.2 文档回滚深度剖析

**核心函数**：`RollbackDocHistory(boxID, historyPath)` (`kernel/model/history.go:230-379`)

```
RollbackDocHistory
  │
  ├─ 【前置检查】
  │    ├─> gulu.File.IsExist(historyPath)   // 历史文件存在性
  │    └─> FlushTxQueue()                   // 事务队列清空
  │
  ├─ 【目标定位】
  │    ├─> getRollbackBox(boxID)            // 获取或创建 Rollback 笔记本
  │    │    ├─> 若原笔记本存在 → 使用原笔记本
  │    │    ├─> 若不存在 → 找名为 "Rollback" 的笔记本
  │    │    └─> 都不存在 → 创建 "Rollback" 笔记本 + Mount
  │    └─> getRollbackDockPath()            // 计算目标路径
  │         ├─> 父文档存在 → 恢复到原父路径
  │         └─> 父文档不存在 → 恢复到笔记本根目录
  │
  ├─ 【AV 同步】
  │    └─> 遍历 Tree 中所有 AV 节点
  │         ├─> 从历史目录复制 storage/av/{id}.json
  │         └─> 仅当目标已存在时才覆盖（安全策略）
  │
  ├─ 【路径重建】
  │    └─> tree.Box / tree.Path / tree.HPath 重新赋值
  │
  ├─ 【块 ID 冲突消解】⭐ 核心完整性保障
  │    ├─> ast.Walk 收集 Tree 中所有块 ID
  │    ├─> treenode.ExistBlockTrees(ids)    // 批量检查 ID 是否已存在
  │    └─> 对冲突 ID 执行 treenode.ResetNodeID(node)
  │         ├─> 生成新的雪花 ID
  │         └─> 若是文档根节点 → 同步更新 tree.Path
  │
  ├─ 【旧文档清理】
  │    ├─> 删除工作区中的旧文档文件
  │    └─> treenode.RemoveBlockTreesByRootID(rootID) // 移除内存索引
  │
  ├─ 【写入新文档】
  │    ├─> filelock.Copy 历史文件 → 目标路径
  │    ├─> sql.RemoveTreeQueue(rootID)       // 旧索引入删除队列
  │    └─> indexWriteTreeIndexQueue(tree)    // 新索引入写队列
  │
  ├─ 【前端通知】
  │    ├─> ReloadFiletree                    // 刷新文件树
  │    ├─> ReloadProtyle(rootID)             // 刷新编辑器
  │    ├─> PushMsg(Language(102), 7000)      // 用户提示
  │    └─> IncSync(rootID)                   // 标记同步脏位
  │
  └─ 【异步善后】
       ├─> sql.FlushQueue                    // 刷入数据库
       ├─> ReloadProtyle(rootID)             // 二次重载
       ├─> 广播 rename 事件（更新标签页标题）
       └─> refreshRefCount()                 // 重新计算定义块引用计数
```

**关键安全设计**：
1. **ID 冲突检测**（Issue #14358）：`treenode.ExistBlockTrees` 批量检查后逐个重置，防止主键冲突
2. **Rollback 笔记本**：原笔记本不存在时自动降级，避免恢复失败
3. **AV 仅覆盖不创建**：目标 AV 已存在才覆盖，防止历史垃圾数据扩散

#### 4.1.3 资源回滚

**函数**：`RollbackAssetsHistory()` (`kernel/model/history.go:409-426`)
- 直接 `filelock.CopyNewtimes()` 从历史目录覆盖到工作区
- 保留文件修改时间

#### 4.1.4 笔记本回滚

**函数**：`RollbackNotebookHistory()` (`kernel/model/history.go:428-445`)
- 复制历史目录下的 `conf.json` 到笔记本目录
- 执行 `FullReindex(false)` 全量重建索引

#### 4.1.5 AV 数据库回滚

**函数**：`RollbackAttributeViewHistory()` (`kernel/model/history.go:447-463`)
- 复制 `.json` 文件覆盖
- 简单直接，无 ID 重置逻辑

### 4.2 L2 仓库快照回滚

#### 4.2.1 全量回滚（Checkout）

**API**：`/api/repo/checkoutRepo` → `CheckoutRepo()` → `checkoutRepo(id)`

**核心函数**：`checkoutRepo(id)` (`kernel/model/repository.go:839-896`)

```
checkoutRepo(id)
  │
  ├─ 【前置检查】
  │    ├─> len(Conf.Repo.Key) > 0          // 密钥配置
  │    └─> newRepository()                  // 仓库实例化
  │
  ├─ 【准备工作】
  │    ├─> PushEndlessProgress(Language(63))// "正在恢复快照..."
  │    ├─> FlushTxQueue()                   // 事务屏障
  │    ├─> CloseWatchAssets()               // 暂停资源监听器
  │    ├─> CloseWatchEmojis()               // 暂停表情监听器
  │    │   // 注意：主题监听器被注释（CloseWatchThemes）
  │    ├─> syncEnabled = Conf.Sync.Enabled  // 保存原同步状态
  │    ├─> Conf.Sync.Enabled = false        // 临时关闭同步
  │    └─> Conf.Save()
  │
  ├─ 【⭐ 回滚安全网】
  │    └─> repo.Index("Backup before checkout", false, ctx)
  │         // 回滚前自动创建当前快照 → 允许"回滚回滚"
  │
  ├─ 【执行回滚】
  │    └─> repo.Checkout(id, ctx)           // dejavu 底层还原
  │         ├─> 逐文件解密 + 校验哈希
  │         ├─> 写入工作区文件
  │         └─> 删除快照中不存在的文件
  │
  └─ 【收尾工作】
       ├─> FullReindex(true)                // 全量重建索引
       └─> 若原同步开启 → 7秒后推送 Language(134) "同步已恢复"
```

**设计要点**：
- **监听器暂停**：防止大规模文件变动触发不必要的回调
- **同步暂停**：避免刚恢复的数据被云端旧版本覆盖
- **自动备份**：回滚前一定创建备份快照，提供后悔药
- **全量重建索引**：文件全量变后，增量索引不可靠，必须 FullReindex

#### 4.2.2 单文件回滚

**API**：`/api/repo/rollbackRepoSnapshotFile`

**核心函数**：`RollbackRepoSnapshotFile()` (`kernel/model/repository.go:191-297`)

```
RollbackRepoSnapshotFile
  ├─> repo.GetFileContent(id)               // 从快照提取文件
  ├─> 写入临时目录 tempDir/filepath
  └─> 分支处理：
       ├─> .sy 文档 → RollbackDocHistory(boxID, tempPath)
       │    // 复用 L1 文档回滚逻辑（含 ID 重置）
       └─> 其他文件 → filelock.CopyNewtimes 直接覆盖
```

---

## 五、用户提示协同机制

### 5.1 通知分层体系

| 通知级别 | 函数 | 典型场景 | 持续时间 |
|---------|------|---------|---------|
| 无限进度 | `PushEndlessProgress()` | Checkout、Purge、批量重命名 | 无限 |
| 状态更新 | `PushUpdateMsg()` | 同步中、下载中、上传中 | 实时更新 |
| 成功提示 | `PushMsg()` | 回滚成功、快照创建 | 3000-7000ms |
| 错误提示 | `PushErrMsg()` | 密钥错误、索引失败 | 0（常驻，需手动关闭） |
| 状态栏 | `PushStatusBar()` | 后台操作状态 | 常驻 |
| 清理进度 | `PushClearProgress()` | 操作完成 | - |

### 5.2 消息推送上下文（eventbus）

**Context 键**：`eventbus.CtxPushMsg`
- `CtxPushMsgToStatusBar`：仅推送到状态栏
- `CtxPushMsgToStatusBarAndProgress`：状态栏 + 进度条

### 5.3 二次确认机制

**前端确认对话框**：`confirmDialog()` (`app/src/dialog/confirmDialog.ts`)

**回滚确认调用**（`app/src/history/history.ts` 附近）：
```typescript
confirmDialog("⚠️ " + window.siyuan.languages.rollback,
    window.siyuan.languages.rollbackConfirm
        .replace("${name}", name)     // 文档名 / 工作区数据
        .replace("${time}", time),    // 快照时间
    () => { /* 实际执行 API 调用 */ }
);
```

### 5.4 关键语言资源码

| 语言码 | 含义 | 使用位置 |
|-------|------|---------|
| 26 | 请先初始化数据仓库 | 密钥检查失败 |
| 36 | 云端快照已下载 | 下载成功 |
| 63 | 正在恢复快照... | Checkout 进度条 |
| 102 | 撤销成功 | 文档回滚成功 |
| 134 | 同步已恢复 | Checkout 后恢复同步 |
| 136 | 正在清理数据仓库 | Purge 进度 |
| 140 | 创建数据仓库快照失败：%s | Index 失败 |
| 141 | 恢复数据仓库快照失败 | Checkout 失败 |
| 142 | 快照备注不能为空 | 空 memo 检查 |
| 143 | 正在创建数据快照... | Index 进度 |
| 144 | 正在下载数据快照... | 下载进度 |
| 147 | 创建数据仓库快照耗时 %.1fs | Index 成功 |
| 148 | 数据仓库快照未变化，耗时 %.1fs | Index 无变化 |
| 206 | 正在重命名第 %d/%d 个文档 | 重命名进度 |
| 286 | 成功恢复已删除文档 %s | 文档回滚成功（新版本） |

---

## 六、API 端点完整清单

### 6.1 History API（10 个）

**路由位置**：`kernel/api/router.go:153-162`

| # | 方法 | 路径 | 权限链 | 功能 |
|---|------|------|--------|------|
| 1 | POST | `/api/history/searchHistory` | Auth + Admin | 分页搜索历史（FTS5 全文） |
| 2 | POST | `/api/history/getHistoryItems` | Auth + Admin | 获取指定时间点的历史条目 |
| 3 | POST | `/api/history/reindexHistory` | Auth + Admin + Readonly | 重建历史索引 |
| 4 | POST | `/api/history/getNotebookHistory` | Auth + Admin | 已删除笔记本列表 |
| 5 | POST | `/api/history/clearWorkspaceHistory` | Auth + Admin + Readonly | 清空所有历史 |
| 6 | POST | `/api/history/getDocHistoryContent` | Auth + Admin | 获取单篇历史文档内容 |
| 7 | POST | `/api/history/rollbackDocHistory` | Auth + Admin + Readonly | **回滚文档** |
| 8 | POST | `/api/history/rollbackAssetsHistory` | Auth + Admin + Readonly | **回滚资源文件** |
| 9 | POST | `/api/history/rollbackNotebookHistory` | Auth + Admin + Readonly | **回滚笔记本** |
| 10 | POST | `/api/history/rollbackAttributeViewHistory` | Auth + Admin + Readonly | **回滚属性数据库** |

> **权限中间件说明**：
> - `CheckAuth`：登录态校验
> - `CheckAdminRole`：管理员角色校验
> - `CheckReadonly`：只读模式拦截（写操作必须）

### 6.2 Repo API（23 个）

**路由位置**：`kernel/api/router.go:427-449`

#### 密钥管理（4 个）

| # | 方法 | 路径 | 功能 |
|---|------|------|------|
| 1 | POST | `/api/repo/initRepoKey` | 生成随机 32 字节密钥 |
| 2 | POST | `/api/repo/initRepoKeyFromPassphrase` | 通过密码派生密钥 |
| 3 | POST | `/api/repo/importRepoKey` | 导入 Base64 编码密钥 |
| 4 | POST | `/api/repo/resetRepo` | ⚠️ 重置仓库（不可逆） |

#### 快照管理（6 个）

| # | 方法 | 路径 | 功能 |
|---|------|------|------|
| 5 | POST | `/api/repo/createSnapshot` | 创建手动快照 |
| 6 | POST | `/api/repo/getRepoSnapshots` | 本地快照分页（32/页） |
| 7 | POST | `/api/repo/tagSnapshot` | 给快照打标签 |
| 8 | POST | `/api/repo/getRepoTagSnapshots` | 已打标签的快照 |
| 9 | POST | `/api/repo/removeRepoTagSnapshot` | 删除本地标签快照 |
| 10 | POST | `/api/repo/diffRepoSnapshots` | 双快照差异计算 |

#### 回滚与文件操作（4 个）

| # | 方法 | 路径 | 功能 |
|---|------|------|------|
| 11 | POST | `/api/repo/checkoutRepo` | **全量回滚到快照** |
| 12 | POST | `/api/repo/rollbackRepoSnapshotFile` | **单文件快照回滚** |
| 13 | POST | `/api/repo/openRepoSnapshotFile` | 快照文件预览 |
| 14 | POST | `/api/repo/getRepoFile` | 快照文件二进制下载 |

#### 云端操作（5 个）

| # | 方法 | 路径 | 功能 |
|---|------|------|------|
| 15 | POST | `/api/repo/getCloudRepoSnapshots` | 云端快照分页 |
| 16 | POST | `/api/repo/getCloudRepoTagSnapshots` | 云端标签快照 |
| 17 | POST | `/api/repo/uploadCloudSnapshot` | 上传快照到云端 |
| 18 | POST | `/api/repo/downloadCloudSnapshot` | 下载云端快照 |
| 19 | POST | `/api/repo/removeCloudRepoTagSnapshot` | 删除云端标签快照 |

#### 清理与配置（4 个）

| # | 方法 | 路径 | 功能 |
|---|------|------|------|
| 20 | POST | `/api/repo/purgeRepo` | 清理本地仓库 |
| 21 | POST | `/api/repo/purgeCloudRepo` | 清理云端仓库 |
| 22 | POST | `/api/repo/setRepoIndexRetentionDays` | 设置索引保留天数 |
| 23 | POST | `/api/repo/setRetentionIndexesDaily` | 设置每日保留索引数 |

---

## 七、数据完整性保障机制

### 7.1 七层完整性保障链

```
[L1] 文件锁层 (filelock)
   └─> 所有文件读写走统一封装，防并发写入冲突
   代码：gofrs/flock 库 + filelock 自定义封装

[L2] 事务屏障层 (FlushTxQueue)
   └─> 所有快照/回滚操作前必须等待事务队列清空
   代码：kernel/model/transaction.go:123-150

[L3] ID 冲突消解层
   └─> treenode.ExistBlockTrees + ResetNodeID
   代码：kernel/model/history.go:301-328

[L4] 路径重建层
   └─> getRollbackBox + getRollbackDockPath 自动降级
   代码：kernel/model/history.go:1045-1069

[L5] 索引一致性层
   └─> 先 RemoveTreeQueue 再 indexWriteTreeIndexQueue
   代码：kernel/model/history.go:330-341

[L6] 回滚安全网层
   └─> Checkout 前自动 "Backup before checkout"
   代码：kernel/model/repository.go:873-880

[L7] 同步互斥层
   └─> Conf.Sync.Enabled = false 防止刚恢复被覆盖
   代码：kernel/model/repository.go:866-869
```

### 7.2 文件层面保障

- **原子写入**：`gulu.File.WriteFileSafer` — 先写 `.tmp` 再 `rename`
- **文件锁**：`gofrs/flock` — 跨进程文件锁
- **时间戳保留**：`CopyNewtimes` — 复制时保留 mtime/atime，避免触发不必要的增量扫描

### 7.3 数据库层面保障

- **独立数据库**：`historyDB` 与主库 `db` 分离，互不影响
- **批量事务**：512 条/批插入（`kernel/sql/history.go:151-172`）
- **失败自愈**：索引写入失败 → 删除问题目录 → 发布 `EvtSQLHistoryRebuild` → 全量重建

### 7.4 引用完整性

- **AV 同步**：`generateAvHistoryInTree` 追踪文档内所有 AV 节点
- **引用计数刷新**：回滚后异步 `refreshRefCount` 修正定义块引用统计
- **重命名广播**：`rename` WebSocket 事件确保多端标签页标题同步

---

## 八、回滚操作边界条件

### 8.1 硬边界（不可突破）

| 边界 | 检查位置 | 违规后果 |
|------|---------|---------|
| 只读模式 | `CheckReadonly` 中间件 | API 403 拒绝 |
| 管理员角色 | `CheckAdminRole` 中间件 | API 403 拒绝 |
| 登录态 | `CheckAuth` 中间件 | API 401 拒绝 |
| 历史文件存在 | `RollbackDocHistory` L231-234 | 静默 Warn + 返回 nil |
| 工作区路径 | `util.IsAbsPathInWorkspace` | Error + 日志 |
| 仓库密钥 | `len(Conf.Repo.Key) > 0` | Language(26) 提示 |
| 订阅权限 | 云操作前 `IsSubscriber()` | Language(29) 付费提示 |
| 付费用户 | WebDAV/S3 前 `IsPaidUser()` | Language(214) 付费提示 |

### 8.2 软边界（自动降级）

| 条件 | 降级策略 | 代码位置 |
|------|---------|---------|
| 原笔记本不存在 | 自动创建 "Rollback" 笔记本 | `getRollbackBox()` |
| 父文档不存在 | 恢复到笔记本根目录 | `getRollbackDockPath()` |
| 块 ID 冲突 | 自动重置冲突节点 ID | `RollbackDocHistory` L314-328 |
| 文档 ≥ 1MB | 降级为 Markdown 纯文本渲染 | `GetDocHistoryContent` L219-226 |
| Upsert 树数 > 20% | 触发 FullReindex 而非增量 | `needFullReindex()` |
| dejavu ErrNotFoundIndex | 返回空列表，不报错 | `GetRepoSnapshots` L582-584 |

---

## 九、资源引用管理

### 9.1 引用追踪机制

**文档内嵌资源**：
- AV 数据库：`NodeAttributeView` → `AttributeViewID` → `storage/av/{id}.json`
- 资源文件：通过 `cache.Asset` 缓存追踪 `updated` 时间戳

**历史目录结构**：
```
history/
└── 2024-01-15-150405-update/
    ├── {boxID}/{path}/{rootID}.sy    # 文档（保持原路径结构）
    ├── assets/{yyyy}/{mm}/{asset}.*  # 资源文件（镜像原结构）
    └── storage/av/{avID}.json        # 属性视图数据库
```

### 9.2 清理策略

**L1 文件历史清理**：
- 函数：`clearOutdatedHistoryDir()`
- 策略：按 `HistoryRetentionDays` 配置删除过期目录 + 数据库同步清理
- 触发：定时任务

**L2 仓库快照清理**：
- 函数：`autoPurgeRepo()` / `PurgeRepo()`
- 算法：`gods/sets/hashset` 去重 + 按日期分组智能保留
- 保留规则：
  - `IndexRetentionDays` 天内的所有索引全保留
  - 每天保留 `RetentionIndexesDaily` 个索引
  - 每天最后一个索引必保 + 随机抽样补充

---

## 十、失败恢复策略

### 10.1 失败场景全景

| 失败阶段 | 触发条件 | 恢复策略 | 代码位置 |
|---------|---------|---------|---------|
| **历史索引写入失败** | SQLite 异常 / 数据损坏 | 1. 删除问题历史目录<br/>2. 发布 EvtSQLHistoryRebuild<br/>3. fullReindexHistory 全量重建 | `kernel/sql/queue_history.go:86-99` |
| **回滚中途崩溃** | 进程 Kill / 断电 | 文件系统原子性 + 重启后 checkIndex 订正<br/>L2 Checkout 前有 Backup 快照 | `kernel/model/index_fix.go:50` |
| **同步冲突** | 多端同时修改 | 1. 生成 -sync 后缀历史目录<br/>2. 可选生成 "Conflicted" 冲突副本<br/>3. 前端推送 Language(108) | `kernel/model/repository.go:1642-1681` |
| **云快照下载失败** | 网络中断 | dejavu 断点续传 + 下次自动重试 | dejavu 内部 |
| **块 ID 冲突未覆盖** | 极端时序并发回滚 | RemoveTreeQueue 先清理再写入<br/>最坏情况 FullReindex | 事务屏障设计 |
| **仓库致命错误** | dejavu.ErrRepoFatal | autoSyncErrCount 计数<br/>fixSyncInterval 间隔重试 | `kernel/model/repository.go:1437` |
| **云锁竞争** | 多端同时同步 | ErrLockCloudFailed → 等待 + 重试 | `kernel/model/sync.go:625` |

### 10.2 关键恢复代码路径

**历史索引失败自愈** (`kernel/sql/queue_history.go:86-99`)：
```go
if err = execHistoryOp(op, tx, context); err != nil {
    tx.Rollback()
    dirPath := filepath.Join(util.HistoryDir, dir)
    os.RemoveAll(dirPath)  // 删除损坏的历史目录
    eventbus.Publish(util.EvtSQLHistoryRebuild)  // 触发全量重建
    return
}
```

**同步冲突处理** (`kernel/model/repository.go:1642-1681`)：
```go
if 0 < len(mergeResult.Conflicts) {
    if Conf.Sync.GenerateConflictDoc {
        resetTree(tree, "Conflicted", true)  // 冲突前缀 + 新 ID
        createTreeTx(tree)                    // 创建冲突副本文档
    }
    indexHistoryDir(timestamp + "-sync")     // 冲突版本归档历史
}
```

**事务错误码** (`kernel/model/transaction.go:135-142`)：
```go
TxErrCodeSuccess         = 0  // 成功
TxErrCodeFrequent        = 1  // 操作过于频繁
TxErrCodeBlockNotFound   = 2  // 块不存在
TxErrCodePathUnavailable = 3  // 路径不可用
TxErrCodePushMsg         = 4  // 普通错误（用户消息）
```

---

## 十一、潜在风险点

### 🔴 高风险

1. **块 ID 重置的级联引用失效**
   - 位置：`kernel/model/history.go:321-328`
   - 风险：重置回滚文档的块 ID 后，**外部引用该块的其他文档不会自动更新**，产生悬空引用
   - 影响：块引用（Block Ref）、嵌入块（Embed）、AV 关联视图、反链查询
   - 建议：增加反向引用扫描 + 批量更新链路

2. **Checkout 监听器暂停不完整**
   - 位置：`kernel/model/repository.go:862-864`
   - 风险：`CloseWatchThemes()` 被注释，主题监听器未暂停；插件/挂件监听器未见显式暂停
   - 影响：大规模文件变动时触发不必要的回调，可能导致竞态和性能抖动
   - 建议：抽象 `PauseAllWatchers()` / `ResumeAllWatchers()` 统一框架

3. **L1 单文件回滚无安全网**
   - 位置：`RollbackDocHistory/RollbackAssetsHistory` 全流程
   - 风险：回滚本身是破坏性写入，但 L1 层未自动归档回滚前的版本（L2 Checkout 有）
   - 影响：单文件回滚后若后悔，只能从下一个定时快照恢复，可能丢失中间修改
   - 建议：回滚前先对目标文件创建一条历史记录

### 🟡 中风险

4. **boxLatestHistoryTime 内存态丢失**
   - 位置：`kernel/model/history.go:756`
   - 风险：进程重启后时间戳归零，下次 GenerateFileHistory 将全量扫描所有文件
   - 影响：首次启动后的历史生成性能抖动
   - 缓解：仅影响性能，不影响正确性

5. **退出时异步队列丢失**
   - 位置：`kernel/sql/queue_history.go:73-75`
   - 风险：`util.IsExiting.Load()` 直接 return，未提交的索引操作永久丢失
   - 影响：下次启动需全量重建历史索引，耗时增加

6. **dejavu 密钥丢失不可逆**
   - 位置：`Conf.Repo.Key` 全局唯一
   - 风险：AES-256 密钥是解密唯一凭证，丢失后所有本地+云端仓库快照永久不可读
   - 建议：增加强制密钥备份流程 + 多副本存储提示

### 🟢 低风险

7. **大文档降级渲染无搜索高亮**
   - 位置：`kernel/model/history.go:219-226`
   - 风险：≥1MB 文档降级为 FormatRenderer 纯文本，关键词高亮失效
   - 影响：用户体验下降

8. **历史操作类型硬编码白名单**
   - 位置：`validOps` 数组 `kernel/model/history.go:902`
   - 风险：新增操作类型需同步更新白名单，否则 `indexHistoryDir` 拒绝索引
   - 缓解：通过常量集中管理，开发时一并修改

---

## 十二、后续调查方向

### 🔬 源码级深度调查

1. **dejavu 库内部机制**
   - 目标：内容寻址分块算法、加密存储格式、Diff 实现、原子性保证
   - 依赖：`github.com/siyuan-note/dejavu v0.0.0-20260411080619`
   - 重点：`Repo.Checkout` 崩溃一致性、`Index` 事务性

2. **块引用失效修复路径**
   - 目标：确认是否存在引用修复逻辑（全量扫描 + ID 映射重写）
   - 搜索关键词：`fixRef`, `updateBlockRef`, `refactor block ref`
   - 相关文件：`kernel/sql/block_ref.go`, `kernel/model/virtualref.go`

3. **FullReindex 触发阈值矩阵**
   - 目标：整理所有 `needFullReindex` 的真实阈值和调用点
   - 搜索关键词：`FullReindex`, `needFullReindex`, `upsertTrees`

4. **Checkout 后文件校验机制**
   - 目标：是否存在 post-checkout 哈希校验或一致性检查
   - 搜索关键词：`verifyTree`, `checksum`, `verifyIndex`

### 🔧 工程化改进建议

5. **回滚安全网标准化**
   - 建议：L1 层四类 Rollback 函数统一增加"回滚前快照"步骤
   - 成本：每次回滚增加 ~50ms 文件复制开销
   - 收益：消除单文件回滚后悔药缺失

6. **外部引用批量修复**
   - 建议：在 `ResetNodeID` 后增加反向引用扫描，批量更新引用者文档
   - 参考：复用 `sql.GetBlockRefsByDefID` + 事务更新链路

7. **监听器统一暂停框架**
   - 建议：抽象 `PauseAllWatchers()` / `ResumeAllWatchers()`
   - 覆盖：资源、表情、主题、插件、挂件、数据库等所有文件监听器

8. **关键操作审计日志**
   - 建议：所有 Rollback/Checkout 操作写入结构化审计日志
   - 字段：操作者、源快照 ID、目标路径、耗时、是否成功
   - 价值：问题追溯 + 合规审计

---

## 十三、版本依据核对结论

### 13.1 版本信息确认

| 项目 | 值 | 代码位置 |
|------|----|---------|
| 内核版本 | v3.6.5 | `kernel/util/working.go:47` `Ver = "3.6.5"` |
| 内测标记 | false | `kernel/util/working.go:48` `IsInsider = false` |
| Go 版本 | 1.25.4 | `kernel/go.mod:3` `go 1.25.4` |
| 数据库版本 | 20220501 | `kernel/util/runtime.go:90` `DatabaseVer` |
| dejavu 版本 | v0.0.0-20260411080619-1de6197a80f4 | `kernel/go.mod:66` |
| lute 解析器 | v1.7.7-0.20260419134724 | `kernel/go.mod:11` |
| gulu 工具库 | v1.2.3-0.20260409163331 | `kernel/go.mod:10` |
| Gin 框架 | v1.11.0 | `kernel/go.mod:37` |

### 13.2 代码结构核对

| 模块 | 文件数 | 预估行数 | 核心函数数 |
|------|--------|---------|-----------|
| 历史模型层 | 1 个主文件 | ~1,100 行 | 20+ 导出函数 |
| 仓库模型层 | 1 个主文件 | ~2,000 行 | 30+ 导出函数 |
| SQL 层 | 2 个文件 | ~300 行 | 10+ 导出函数 |
| API 层 | 2 个文件 | ~800 行 | 33 个端点 |
| 前端历史 | 3 个文件 | ~1,000 行 | 多个组件 |
| **合计** | **9 个核心模块** | **~5,200 行** | **33 个 API 端点 + 50+ 核心函数** |

### 13.3 功能完整性核对

| 功能 | L1 文件历史 | L2 仓库快照 | 备注 |
|------|-----------|------------|------|
| 自动生成 | ✅ 定时 + 事务 | ✅ 同步前 + 回滚前 | 触发机制不同 |
| 手动创建 | ❌ 无 | ✅ IndexRepo | L1 无手动入口 |
| 全文搜索 | ✅ FTS5 | ❌ 无 | L2 仅支持标题路径搜索 |
| 双版本对比 | ❌ 无 | ✅ DiffRepoSnapshots | L2 才有并列 Diff |
| 单文件回滚 | ✅ 4 种类型 | ✅ RollbackRepoSnapshotFile | L2 单文件复用 L1 逻辑 |
| 全量回滚 | ❌ 无 | ✅ CheckoutRepo | 仅 L2 支持 |
| 回滚安全网 | ❌ 无 | ✅ 自动 Backup | L1 缺失 |
| 云端同步 | ❌ 无 | ✅ 上传/下载/标签 | 仅 L2 支持云端 |
| 加密存储 | ❌ 明文 | ✅ AES-256 | L2 块数据加密 |
| 去重存储 | ❌ 全量复制 | ✅ Content-Addressed | L2 空间效率更高 |

> **结论**：L1 和 L2 形成互补关系 — L1 侧重细粒度即时回滚（操作级），L2 侧重全量灾难恢复（时间旅行+云端备份）。两套机制各有侧重，共同构成 SiYuan 的版本保护体系。

---

## 附录：关键常量速查

### A.1 历史操作类型

```go
// kernel/model/history.go:812-819
HistoryOpClean   = "clean"    // 清理操作
HistoryOpUpdate  = "update"   // 更新操作
HistoryOpDelete  = "delete"   // 删除操作
HistoryOpFormat  = "format"   // 格式清理
HistoryOpSync    = "sync"     // 同步冲突
HistoryOpReplace = "replace"  // 内容替换
HistoryOpOutline = "outline"  // 大纲调整
```

### A.2 历史记录类型

```go
// kernel/sql/history.go:29-37
HistoryTypeDocName  = 0  // 文档名
HistoryTypeDoc      = 1  // 文档全文
HistoryTypeAsset    = 2  // 资源文件
HistoryTypeDocID    = 3  // 文档 ID
HistoryTypeDatabase = 4  // 数据库
```

### A.3 历史目录命名规则

```
格式：YYYY-MM-DD-HHMMSS-{opType}
示例：2026-06-15-153045-update
位置：workspace/history/{name}/
```

---

> **生成时间**：2026-06-15
> **分析版本**：SiYuan v3.6.5 (kernel)
> **分析范围**：9 个核心模块 / ~5,200 行代码 / 33 个 API 端点
> **代码引用格式**：全文采用「仓库相对路径 + 行号范围」格式引用，例如 `kernel/model/history.go:54-60` 表示仓库根目录下 `kernel/model/history.go` 文件的第 54 至 60 行
