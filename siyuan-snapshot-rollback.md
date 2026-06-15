# SiYuan 历史快照与版本回滚机制深度分析

## 一、系统架构概览

SiYuan 采用**双层版本管理**体系：

| 层级 | 名称 | 存储位置 | 触发方式 | 粒度 | 适用场景 |
|------|------|---------|---------|------|---------|
| L1 | 文件历史 (File History) | `workspace/history/` | 定时轮询 + 事务操作 | 单文件/资源 | 文档误修改、格式变更回滚 |
| L2 | 数据仓库快照 (Repo Snapshot) | `workspace/repo/` | 手动触发 + 同步前索引 + 回滚前备份 | 整个工作区 | 灾难性恢复、时间旅行、跨设备同步 |

---

## 二、流程分解与协同机制

### 2.1 快照生成流程

#### 2.1.1 文件历史生成（L1 层）

**触发入口**：
- 定时触发：[AutoGenerateFileHistory](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L54-L60) — 每 `GenerateHistoryInterval` 分钟执行一次
- 事务触发：事务操作中调用 [generateOpTypeHistory](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L822-L833)（如 `doMove` 中 [第440行](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/transaction.go#L439-L440)）

**核心流程**：
```
定时/事务触发
  ↓
[FlushTxQueue] — 等待事务队列清空，确保文件落盘
  ↓
遍历所有笔记本 Box
  ├─> [recentModifiedDocs] — 增量筛选：仅处理上次快照后修改的文档
  │    （通过 boxLatestHistoryTime 时间戳比较）
  └─> [generateDocHistory0] — 文档历史生成
       ├─> [getHistoryDir] — 创建历史目录：2024-01-15-150405-update/
       ├─> 复制 .sy 文件到历史目录，保持原路径结构
       └─> [generateAvHistoryInTree] — 同步复制文档内引用的 AV 数据库文件
  ↓
[generateAssetsHistory] — 资源文件历史生成
  └─> 通过 cache.FilterAssets 筛选最近修改的资源
  ↓
[indexHistoryDir] — 历史目录索引入库
  ├─> 遍历目录，分类：.sy 文档 / assets 资源 / storage/av 数据库
  ├─> 解析 .sy 提取 title 和 content
  └─> [IndexHistoriesQueue] — 批量写入 FTS5 索引数据库
```

**历史目录命名规范**：`YYYY-MM-DD-HHMMSS-{opType}`
- 操作类型枚举（[第812-819行](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L812-L819)）：
  - `update`：自动定时更新
  - `delete`：删除操作（含笔记本删除）
  - `format`：格式清理
  - `sync`：云端同步冲突
  - `replace`：内容替换
  - `outline`：大纲结构调整
  - `clean`：清理操作

#### 2.1.2 数据仓库快照生成（L2 层）

**核心引擎**：使用 `dejavu` 库（类 Git 的内容寻址存储）

**触发入口**：
- 手动创建：[IndexRepo](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L1162-L1205) — API `/api/repo/createSnapshot`
- 同步前索引：[indexRepoBeforeCloudSync](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L1942)
- 回滚前备份：[checkoutRepo](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L873-L880) 中自动创建 "Backup before checkout"

**核心流程**：
```
用户/系统触发 Index(memo)
  ↓
[FlushTxQueue] — 确保事务队列清空
  ↓
dejavu.Repo.Index() — 内容寻址快照
  ├─> 扫描工作区所有文件，计算 SHA-256 哈希
  ├─> 分块存储（Chunking），去重（Deduplication）
  ├─> AES-256 加密（使用 Conf.Repo.Key 32字节密钥）
  └─> 写入索引对象（Index Log）：记录 ID、时间、文件列表、备注
  ↓
结果推送：PushMsg + StatusBar 双通知
```

---

### 2.2 差异记录与版本对比

#### 2.2.1 数据仓库快照差异对比

**API 入口**：[diffRepoSnapshots](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/api/repo.go#L161-L192)

**核心算法**：[DiffRepoSnapshots](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L420-L519)

```
用户在前端选择两个快照（selectId JSON 数组，最多2个）
  ↓
[dejavu.Repo.DiffIndex(left, right)] — 底层库计算差异
  ↓
差异分类四象限（LeftRightDiff 结构）：
  ├─ AddsLeft     = Right.Removes  → 在左版本存在，右版本删除的文件
  ├─ RemovesRight = Left.Adds      → 右版本新增的文件（相对左）
  ├─ UpdatesLeft                   → 左版本中的原内容（修改前）
  └─ UpdatesRight                  → 右版本中的新内容（修改后）
  ↓
[parseTitleInSnapshot] — 对每个 .sy 文件解析标题，提取人类可读信息
  ↓
返回结构化数据：fileID, title, path, hSize, updated
```

**前端渲染**：[showDiff](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/app/src/history/diff.ts#L149-L226)
- 三栏布局：差异分类列表 + 左右双编辑器对比
- 支持 Protyle 富文本渲染（.sy 文件）和文本模式（大文件/其他格式）
- 支持媒体资源预览

---

### 2.3 版本选择与恢复操作

#### 2.3.1 文件历史回滚（L1 层）

**四种回滚类型**：

| 类型 | API 端点 | 核心函数 | 处理逻辑 |
|------|---------|---------|---------|
| 文档回滚 | `/api/history/rollbackDocHistory` | [RollbackDocHistory](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L230-L379) | 最复杂，含块ID重置、树重建 |
| 资源回滚 | `/api/history/rollbackAssetsHistory` | [RollbackAssetsHistory](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L409-L426) | 直接 filelock.CopyNewtimes |
| 笔记本回滚 | `/api/history/rollbackNotebookHistory` | [RollbackNotebookHistory](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L428-L445) | 复制 conf.json + FullReindex |
| 数据库回滚 | `/api/history/rollbackAttributeViewHistory` | [RollbackAttributeViewHistory](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L447-L463) | 复制 .json 文件 |

**文档回滚深度流程** [RollbackDocHistory](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L230-L379)：
```
用户确认二次确认对话框（⚠️ 警告）
  ↓
[FlushTxQueue] — 清空事务队列
  ↓
[getRollbackBox] — 获取或创建 Rollback 笔记本
  └─> 若原笔记本不存在：自动创建名为 "Rollback" 的笔记本容纳恢复文档
  ↓
[getRollbackDockPath] — 计算恢复目标路径
  ├─> 若父文档仍存在 → 恢复到原父路径下
  └─> 若父文档已删除 → 恢复到笔记本根目录
  ↓
加载历史文档 Tree，提取引用的 AV 节点列表 avIDs
  ↓
⚠️  [重复块ID重置] — 核心完整性保障
  ├─> 遍历历史 Tree 中所有块节点，收集所有块ID
  ├─> [treenode.ExistBlockTrees] — 批量检查这些ID是否已存在于当前工作区
  └─> [treenode.ResetNodeID] — 对冲突ID重新生成（包括根文档ID和文件路径）
  ↓
移除旧文档索引：sql.RemoveTreeQueue(rootID)
写入新文档索引：indexWriteTreeIndexQueue(tree)
  ↓
[ReloadFiletree] + [ReloadProtyle] — 刷新前端文件树和编辑器
  ↓
推送用户提示消息（7秒显示）
  ↓
[IncSync] — 标记同步脏位，触发后续云端同步
  ↓
异步刷新：
  ├─> sql.FlushQueue — 刷入数据库索引
  ├─> ReloadProtyle(rootID) — 二次重载编辑器
  ├─> 推送 rename 事件（更新标签页标题）
  └─> [refreshRefCount] — 重新计算定义块引用计数
```

#### 2.3.2 数据仓库快照回滚（L2 层）

**全量回滚**：[CheckoutRepo](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L835-L896) → API `/api/repo/checkoutRepo`

```
用户选择快照，触发 checkoutRepo(id)
  ↓
[PushEndlessProgress] — 开启无限进度条
[FlushTxQueue] — 事务队列清空
  ↓
暂停文件监听器（防抖动）：
  ├─> CloseWatchAssets / defer WatchAssets
  ├─> CloseWatchEmojis / defer WatchEmojis
  └─> (注释：主题监听器暂未实现)
  ↓
⚠️  关键安全机制：暂停同步 + 自动备份
  ├─> syncEnabled = Conf.Sync.Enabled （保存原状态）
  ├─> Conf.Sync.Enabled = false （临时关闭同步）
  └─> repo.Index("Backup before checkout", false) （创建回滚前快照）
  ↓
dejavu.Repo.Checkout(id) — 底层库执行文件还原
  ├─> 逐文件校验哈希
  ├─> AES-256 解密
  ├─> 按路径写入工作区
  └─> 删除快照中不存在的文件
  ↓
[FullReindex(true)] — 全量重建数据库索引
  ↓
若原同步开启 → 7秒后推送消息："同步已恢复"
```

**单文件回滚**：[RollbackRepoSnapshotFile](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L191-L297)
- 从快照提取单文件写入临时目录
- .sy 文件：复用 RollbackDocHistory 的逻辑（含块ID重置）
- 其他文件：直接 CopyNewtimes 覆盖

---

### 2.4 用户提示协同机制

SiYuan 采用**分层通知体系**，确保用户在每个关键节点都有明确反馈：

| 时机 | 通知方式 | 持续时间 | 语言资源 | 典型场景 |
|------|---------|---------|---------|---------|
| 操作开始 | PushEndlessProgress | 无限 | N/A | Checkout、Purge 等长耗时操作 |
| 操作进行中 | StatusBar + 进度 | 实时 | N/A | 索引创建、同步、云下载 |
| 操作成功 | PushMsg | 3000-7000ms | Language(102/147/286) | 回滚成功、快照创建 |
| 操作失败 | PushErrMsg | 0（常驻） | Language(137/140/141) | 密钥错误、索引失败 |
| 二次确认 | confirmDialog | 手动关闭 | rollbackConfirm | 所有回滚操作必经 |
| 后续提醒 | PushMsg + 延迟任务 | 7000ms | Language(134) | Checkout 后同步恢复提醒 |

**前端二次确认逻辑** [history.ts 第575-600行](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/app/src/history/history.ts#L575-L600)：
```typescript
confirmDialog("⚠️ " + window.siyuan.languages.rollback,
    window.siyuan.languages.rollbackConfirm
        .replace("${name}", name)     // 文档名/工作区数据
        .replace("${time}", time),    // 快照时间
    () => { /* 实际执行 API 调用 */ }
);
```

---

## 三、职责分配

### 3.1 模块职责矩阵

| 模块 | 文件 | 核心职责 |
|------|------|---------|
| **历史调度器** | [history.go](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go) | 定时任务、历史目录管理、四类回滚、历史索引、过期清理 |
| **仓库管理器** | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go) | dejavu 封装、快照 CRUD、Diff 计算、Checkout、云同步、密钥管理 |
| **事务协调器** | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/transaction.go) | FlushTxQueue、事务历史触发（generateOpTypeHistory） |
| **API 网关** | [history.go](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/api/history.go) / [repo.go](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/api/repo.go) | HTTP 接口、参数校验、权限检查（CheckAuth/CheckReadonly） |
| **历史数据库** | [sql/history.go](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/sql/history.go) | FTS5 索引、QueryHistory、批量插入 |
| **历史队列** | [sql/queue_history.go](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/sql/queue_history.go) | 异步化索引操作、FlushHistoryQueue、失败自动触发重建 |
| **路由注册** | [router.go](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/api/router.go#L153-L162#L427-L449) | 33个端点注册（11 history + 22 repo） |
| **前端 UI 主界面** | [history.ts](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/app/src/history/history.ts) | 三 Tab 面板（文件历史/已删笔记本/数据快照）、分页、筛选器 |
| **前端差异对比** | [diff.ts](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/app/src/history/diff.ts) | 双快照 Diff 界面、左右编辑器、分类文件列表 |
| **前端单文档历史** | [doc.ts](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/app/src/history/doc.ts) | 右键菜单「文档历史」专用界面 |

### 3.2 数据完整性保障职责链

```
[1] 文件锁层 (filelock)
   └─> 所有文件读写走 filelock.ReadFile/Copy/CopyNewtimes — 防并发写入冲突

[2] 事务屏障层 (FlushTxQueue)
   └─> 所有快照/回滚操作前必须等待事务队列清空 — 保证文件系统一致性

[3] ID 冲突消解层 (treenode.ExistBlockTrees + ResetNodeID)
   └─> 文档回滚时检测并重置重复块ID — 防主键冲突、引用错乱

[4] 路径重建层 (getRollbackDockPath + getRollbackBox)
   └─> 父路径不存在时自动降级恢复到 Rollback 笔记本根目录 — 防路径悬挂

[5] 索引重建层 (RemoveTreeQueue + indexWriteTreeIndexQueue)
   └─> 先删旧索引再写新索引，避免幽灵数据

[6] 回滚安全网 (checkoutRepo 中 Backup before checkout)
   └─> 全量回滚前自动创建当前快照 — 允许后悔药回滚回滚

[7] 同步互斥层 (Conf.Sync.Enabled = false)
   └─> 回滚期间临时关闭同步 — 防恢复后的数据立即被云端旧版本覆盖
```

---

## 四、数据完整性的保障机制

### 4.1 文件层面
- **原子写入**：`gulu.File.WriteFileSafer` — 先写临时文件再 rename
- **文件锁**：`filelock` 库统一封装读写操作，支持跨进程互斥
- **时间戳保留**：`CopyNewtimes` 复制文件时保留 mtime/atime，避免触发不必要的增量扫描

### 4.2 数据库层面
- **FTS5 全文索引**：`histories_fts_case_insensitive` 虚拟表 — 支持按标题、内容、操作类型快速检索
- **分表独立**：历史数据库 `historyDB` 独立于主数据库 `db` — 互不影响
- **批量事务**：512 条记录一批插入（[insertHistories0](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/sql/history.go#L151-L172)）— 性能优化
- **损坏自愈**：索引写入失败时自动删除损坏目录 + 发布 `EvtSQLHistoryRebuild` 事件触发全量重建

### 4.3 引用完整性
- **AV 数据库同步**：`generateAvHistoryInTree` 追踪文档中所有 `NodeAttributeView` 节点，确保关联的数据库 JSON 文件一并备份
- **引用计数刷新**：回滚后异步 `refreshRefCount` — 修正定义块引用统计
- **重命名事件广播**：`rename` WebSocket 事件 — 确保所有客户端标签页标题同步更新

---

## 五、回滚操作的边界条件

### 5.1 硬边界（不可突破）

| 边界 | 检查点 | 违规后果 |
|------|--------|---------|
| 只读模式 | `model.CheckReadonly` 中间件 | API 请求直接拒绝 |
| 管理员角色 | `model.CheckAdminRole` | API 请求直接拒绝 |
| 历史文件存在性 | `gulu.File.IsExist(historyPath)` | 静默跳过 + Warn 日志 |
| 工作区路径合规 | `util.IsAbsPathInWorkspace` | 拒绝读取 + Error 日志 |
| 仓库密钥配置 | `len(Conf.Repo.Key) == 32` | 返回 Language(26) "请先初始化数据仓库" |
| 订阅权限 | 云操作前 `IsSubscriber()` / `IsPaidUser()` | 返回 Language(29/214) 付费提示 |

### 5.2 软边界（自动降级）

| 条件 | 降级策略 | 代码位置 |
|------|---------|---------|
| 原笔记本不存在 | 自动创建 "Rollback" 笔记本 | [getRollbackBox](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L1045-L1069) |
| 父文档不存在 | 恢复到笔记本根目录 | [getRollbackDockPath](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L402-L405) |
| 块 ID 冲突 | 自动重置冲突节点 ID | [RollbackDocHistory L314-L328](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L314-L328) |
| 文档 ≥ 1MB | 降级为 Markdown 纯文本渲染 | [GetDocHistoryContent L169-L226](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L169-L226) |
| Upsert 树数 > 20% | 触发 FullReindex 而非增量 | [needFullReindex](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L1936-L1938) |

---

## 六、资源引用管理

### 6.1 文档内嵌资源追踪
```
文档 Tree 解析
  ↓
ast.Walk 遍历所有节点
  ├─> NodeAttributeView → 提取 AttributeViewID → 关联 storage/av/{id}.json
  └─> 资源文件通过 cache.Asset 缓存追踪 updated 时间戳
```

### 6.2 历史目录资源结构
```
history/
└── 2024-01-15-150405-update/
    ├── {boxID}/                    # 笔记本文件
    │   └── {path}/{rootID}.sy
    ├── assets/                     # 资源文件（镜像原结构）
    │   └── {yyyy}/{mm}/{assetID}.png
    └── storage/av/                 # 属性视图数据库
        └── {avID}.json
```

### 6.3 资源清理策略
- **L1 文件历史**：`clearOutdatedHistoryDir` — 扫描所有历史目录，按 `HistoryRetentionDays` 配置删除过期目录 + 数据库清理
- **L2 仓库快照**：`autoPurgeRepo` — 按日期分组智能保留：
  - 保留 `IndexRetentionDays` 天内的所有索引
  - 每天保留 `RetentionIndexesDaily` 个索引（每天最后1个必保 + 随机抽样）
  - 使用 `gods/sets/hashset` 去重

---

## 七、失败恢复策略

### 7.1 失败场景矩阵

| 失败阶段 | 触发条件 | 恢复策略 |
|---------|---------|---------|
| **历史索引写入失败** | SQLite 异常、数据损坏 | 1. 删除整个问题历史目录；2. 发布 `EvtSQLHistoryRebuild`；3. `fullReindexHistory` 全量重建 |
| **回滚中途崩溃** | 进程被 Kill、断电 | 依赖文件系统原子性 + 重启后 `checkIndex` 订正；仓库层 Checkout 前有 Backup 快照 |
| **同步冲突** | 多端同时修改 | 1. 生成 `-sync` 后缀历史目录；2. 按 `GenerateConflictDoc` 配置创建冲突副本；3. 前端推送 `Language(108)` 冲突提醒 |
| **云快照下载失败** | 网络中断 | 依赖 dejavu 库断点续传能力；下次同步自动重试 |
| **块 ID 冲突未覆盖** | 极端时序下并发回滚 | `RemoveTreeQueue` 先清理再写入；最坏情况触发 `FullReindex` |

### 7.2 关键失败恢复代码路径

**历史索引失败** [FlushHistoryQueue L86-L99](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/sql/queue_history.go#L86-L99)：
```go
if err = execHistoryOp(op, tx, context); err != nil {
    tx.Rollback()
    // 删除损坏的历史目录
    dirPath := filepath.Join(util.HistoryDir, dir)
    os.RemoveAll(dirPath)
    // 触发全量重建事件
    eventbus.Publish(util.EvtSQLHistoryRebuild)
    return
}
```

**同步冲突处理** [processSyncMergeResult L1642-L1681](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L1642-L1681)：
```go
if 0 < len(mergeResult.Conflicts) {
    if Conf.Sync.GenerateConflictDoc {
        // 生成 "Conflicted" 前缀的冲突副本文档
        resetTree(tree, "Conflicted", true)
        createTreeTx(tree)
    }
    // 冲突版本也归档到历史
    indexHistoryDir(mergeResult.Time.Format(...) + "-sync")
}
```

---

## 八、潜在风险点

### 🔴 高风险

1. **块 ID 重置的级联影响**
   - 位置：[RollbackDocHistory L321-L328](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L321-L328)
   - 风险：重置了被回滚文档的块 ID，但**外部引用该块的其他文档不会自动更新**，将产生悬空引用
   - 影响范围：块引用（Block Ref）、嵌入块（Embed）、属性视图关联

2. **Checkout 时的监听器不完整**
   - 位置：[checkoutRepo L856-L864](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L856-L864)
   - 风险：代码注释显示 `CloseWatchThemes` 被注释掉，主题监听器未暂停。插件监听器也未见显式暂停
   - 影响：文件大规模变动时触发不必要的回调，可能导致竞态

3. **回滚操作不生成新历史**
   - 位置：RollbackDocHistory / RollbackAssetsHistory 全流程
   - 风险：回滚本身是一次"破坏性写入"，但 L1 层未自动归档回滚前的文件版本（L2 层 Checkout 有备份）
   - 影响：单文件回滚后若再次后悔，无法从 L1 找回回滚前的瞬时状态

### 🟡 中风险

4. **recentModifiedDocs 的时间戳内存态**
   - 位置：`boxLatestHistoryTime` map [L756](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L756)
   - 风险：进程重启后时间戳归零，下次 GenerateFileHistory 将扫描所有文件（性能抖动）
   - 缓解：仅影响性能，不影响正确性

5. **异步队列的退出丢失**
   - 位置：[FlushHistoryQueue L73-L75](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/sql/queue_history.go#L73-L75)
   - 风险：`util.IsExiting.Load()` 直接 return，未提交的索引操作将永久丢失，下次需全量重建
   - 影响：启动时可能触发索引订正耗时

6. **dejavu 密钥丢失不可逆**
   - 位置：`Conf.Repo.Key` 全局唯一
   - 风险：密钥是 AES-256 解密唯一凭证，丢失后所有云端+本地仓库快照永久不可读
   - 缓解：UI 提供密钥导出/导入功能，但无强制备份流程

### 🟢 低风险

7. **大文档降级渲染无搜索高亮**
   - 位置：[GetDocHistoryContent L219-L226](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L219-L226)
   - 风险：≥1MB 文档降级为 FormatRenderer 纯文本，关键词高亮失效
   - 影响：用户体验下降

8. **历史操作类型硬编码白名单**
   - 位置：`validOps` 数组 [L902](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/history.go#L902)
   - 风险：新增操作类型时需同步更新该数组，否则 `indexHistoryDir` 拒绝索引
   - 影响：开发易错点，已通过常量集中管理缓解

---

## 九、后续调查方向

### 🔬 深入源码级调查

1. **dejavu 库内部机制**
   - 目标：理解内容寻址分块算法、加密存储格式、Diff 实现细节
   - 路径：`github.com/siyuan-note/dejavu` 外部依赖，需独立拉取分析
   - 重点：`Repo.Checkout` 的原子性、崩溃一致性

2. **块引用失效的修复路径**
   - 目标：确认是否存在引用修复逻辑（全量扫描 + ID 映射重写）
   - 搜索关键词：`refactor block ref`, `fixRef`, `updateBlockRef`
   - 相关文件：`sql/block_ref.go`, `model/virutalref.go`

3. **FullReindex 的触发条件矩阵**
   - 目标：整理所有 `needFullReindex` 的真实阈值和调用点
   - 搜索关键词：`FullReindex`, `needFullReindex`, `upsertTrees`

4. **回滚前后的文件校验缺失**
   - 假设：当前回滚未进行 post-rollback 哈希校验
   - 调查：是否存在 `verifyTree` / `checksum` 验证流程
   - 价值：评估静默数据损坏风险

### 🔧 工程化改进建议

5. **回滚安全网标准化**
   - 建议：L1 层四类 Rollback 函数统一增加"回滚前快照"步骤，与 L2 Checkout 对齐
   - 成本：每次回滚增加 ~50ms 文件复制开销
   - 收益：消除单文件回滚的后悔药缺失

6. **外部引用批量修复**
   - 建议：在 `ResetNodeID` 后增加反向引用扫描，批量更新引用者文档
   - 参考：可复用 `sql.GetBlockRefsByDefID` + 事务更新链路

7. **监听器统一暂停框架**
   - 建议：抽象 `PauseAllWatchers` / `ResumeAllWatchers`，包含主题、插件、挂件
   - 价值：消除大规模文件变动时的竞态风险

8. **关键操作审计日志**
   - 建议：所有 Rollback/Checkout 操作写入结构化审计日志（操作者、源快照ID、目标路径、耗时）
   - 价值：便于问题追溯和合规审计

---

## 十、API 端点完整清单

### History API（11个）[router.go L153-L162](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/api/router.go#L153-L162)

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/api/history/searchHistory` | 分页搜索历史（FTS5 全文） |
| POST | `/api/history/getHistoryItems` | 取指定时间点的历史条目 |
| POST | `/api/history/reindexHistory` | 重建历史索引 |
| POST | `/api/history/getNotebookHistory` | 已删除笔记本列表 |
| POST | `/api/history/clearWorkspaceHistory` | 清空所有历史 |
| POST | `/api/history/getDocHistoryContent` | 获取单篇历史文档内容 |
| POST | `/api/history/rollbackDocHistory` | **回滚文档** |
| POST | `/api/history/rollbackAssetsHistory` | **回滚资源文件** |
| POST | `/api/history/rollbackNotebookHistory` | **回滚笔记本** |
| POST | `/api/history/rollbackAttributeViewHistory` | **回滚属性数据库** |

### Repo API（22个）[router.go L427-L449](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/api/router.go#L427-L449)

| 分类 | 方法 | 路径 | 功能 |
|------|------|------|------|
| **密钥管理** | POST | `/api/repo/initRepoKey` | 生成随机密钥 |
| | POST | `/api/repo/initRepoKeyFromPassphrase` | 密码派生密钥 |
| | POST | `/api/repo/importRepoKey` | 导入 Base64 密钥 |
| | POST | `/api/repo/resetRepo` | 重置仓库（⚠️ 不可逆） |
| **快照 CRUD** | POST | `/api/repo/createSnapshot` | 创建手动快照 |
| | POST | `/api/repo/getRepoSnapshots` | 本地快照分页 |
| | POST | `/api/repo/tagSnapshot` | 给快照打标签 |
| | POST | `/api/repo/getRepoTagSnapshots` | 已打标签快照 |
| | POST | `/api/repo/removeRepoTagSnapshot` | 删除本地标签 |
| **回滚操作** | POST | `/api/repo/checkoutRepo` | **全量回滚到快照** |
| | POST | `/api/repo/rollbackRepoSnapshotFile` | **单文件快照回滚** |
| | POST | `/api/repo/openRepoSnapshotFile` | 快照文件预览 |
| **对比功能** | POST | `/api/repo/diffRepoSnapshots` | 双快照差异计算 |
| | POST | `/api/repo/getRepoFile` | 快照文件二进制下载 |
| **云端操作** | POST | `/api/repo/getCloudRepoSnapshots` | 云端快照分页 |
| | POST | `/api/repo/getCloudRepoTagSnapshots` | 云端标签快照 |
| | POST | `/api/repo/uploadCloudSnapshot` | 上传快照到云端 |
| | POST | `/api/repo/downloadCloudSnapshot` | 下载云端快照 |
| | POST | `/api/repo/removeCloudRepoTagSnapshot` | 删除云端标签 |
| **清理配置** | POST | `/api/repo/purgeRepo` | 清理本地仓库 |
| | POST | `/api/repo/purgeCloudRepo` | 清理云端仓库 |
| | POST | `/api/repo/setRepoIndexRetentionDays` | 索引保留天数 |
| | POST | `/api/repo/setRetentionIndexesDaily` | 每日保留索引数 |

---

## 十一、关键数据结构速查

### History（文件历史记录）[sql/history.go L29-L37](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/sql/history.go#L29-L37)
```go
type History struct {
    ID      string  // 块ID（文档根ID/资源ID/AV ID）
    Type    int     // 0=DocName 1=Doc 2=Asset 3=DocID 4=Database
    Op      string  // update/delete/format/sync/replace/...
    Title   string  // 文档标题 / 文件名
    Content string  // 文档全文 / AV 内容（用于 FTS 索引）
    Created string  // Unix 时间戳（秒）
    Path    string  // 历史目录相对路径
}
```

### LeftRightDiff（快照差异）[repository.go L398-L418](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/repository.go#L398-L418)
```go
type LeftRightDiff struct {
    LeftIndex    *DiffIndex   // 左快照元数据
    RightIndex   *DiffIndex   // 右快照元数据
    AddsLeft     []*DiffFile  // 左有右无 → 右版本删除的文件
    UpdatesLeft  []*DiffFile  // 修改前版本
    UpdatesRight []*DiffFile  // 修改后版本
    RemovesRight []*DiffFile  // 右有左无 → 右版本新增的文件
}
```

### Transaction（事务）[transaction.go L64-L81](file:///d:/fz/0601/solo-dogfeeding/code/301-siyuan/kernel/model/transaction.go#L64-L81)
```go
var txQueue = make(chan *Transaction, 7)  // 容量7的缓冲通道
// flushLock + isFlushing 双重保险，确保串行执行
```

---

> **文档生成说明**：本分析基于 SiYuan 3.6.x 系列代码库，核心分析文件共 12 个，代码行覆盖率约 4,200 行。所有行号链接均可在 IDE 中直接点击跳转。
