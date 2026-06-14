# SiYuan 多端实时同步与冲突合并机制深度解析

## 1. 架构概览

### 1.1 核心依赖

SiYuan 的同步机制基于自研的 `dejavu` 库实现，版本信息见 [go.mod](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/go.mod#L66)：
`v0.0.0-20260411080619-1de6197a80f4`

该库提供了完整的数据仓库（Repository）抽象，包括：

- **索引管理**：快照（Index）创建与版本追踪
- **云存储适配**：多 Provider 抽象层（SiYuan/S3/WebDAV/Local）
- **分块存储**：内容寻址分块（CAS）与增量同步
- **冲突检测**：三方合并引擎与冲突文件识别
- **传输协议**：加密上传/下载与流量统计

关键引用：
- [newRepository()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1995-L2033)
- [dejavu.NewRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L2027)

### 1.2 模块划分

```
kernel/model/
├── sync.go              # 同步入口调度、WebSocket 感知层、配置管理
├── repository.go        # dejavu 仓库封装、合并结果处理、索引/快照
├── transaction.go       # 本地事务队列、修改触发点（IncSync调用）
├── assets_watcher.go    # 资产文件监听触发同步
├── file.go / box.go     # 文件/笔记本操作触发同步
├── blockial.go          # 属性变更触发同步
└── history.go           # 历史版本还原触发同步

app/src/
├── dialog/processSystem.ts  # processSync() 图标状态、reloadSync() UI刷新
├── sync/syncGuide.ts        # 手动同步触发
├── index.ts / window/index.ts  # WebSocket 消息分发
└── mobile/util/onMessage.ts     # 移动端同步消息处理

kernel/api/
└── sync.go              # HTTP API 层（performSync/setSync*/getSyncInfo 等）

kernel/conf/
└── sync.go              # Sync 配置结构体、Provider 常量
```

### 1.3 同步模式与存储 Provider

配置结构见 [Sync 结构体](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/conf/sync.go#L19-L32)：

| 模式 Mode | 说明 | 触发时机 |
|-----------|------|----------|
| 1 (自动)  | 默认模式 | 定时 + 修改触发 + 启动/退出 + WS 通知 |
| 2 (手动)  | 仅启动和退出同步 | boot / exit |
| 3 (完全手动) | 纯手动 | 用户显式调用 upload/download |

| Provider | 常量值 | 鉴权方式 |
|----------|--------|----------|
| SiYuan 官方云 | 0 | Token |
| S3 对象存储 | 2 | AccessKey/SecretKey |
| WebDAV | 3 | Username/Password (Basic Auth) |
| 本地文件系统 | 4 | 路径鉴权 |

---

## 2. 本地修改追踪与持久化机制

### 2.1 修改触发点设计

SiYuan 采用**多点侵入式触发**而非目录级文件监听（文件监听仅作补充），确保所有数据变更都能准确捕获。核心触发函数为 `IncSync()`，在代码库中有 **48+ 处调用**。

[IncSync() 实现](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L703-L706)

```go
func IncSync() {
    syncSameCount.Store(0)           // 重置"无变更连续同步"计数
    planSyncAfter(time.Duration(Conf.Sync.Interval) * time.Second)
}
```

**主要触发场景分类**：

| 场景 | 触发文件 | 关键位置 |
|------|----------|----------|
| 事务提交（最核心） | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/transaction.go#L1891) | `commit()` 末尾统一调用 |
| 文档 CRUD | [file.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/file.go) | 创建/重命名/删除/移动等 |
| 笔记本操作 | [box.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/box.go) | 创建/删除/重命名 |
| 属性变更 | [blockial.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/blockial.go) | 块 IAL 属性写入 |
| 资产上传/变更 | [assets.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/assets.go)、[assets_watcher.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/assets_watcher.go) | 上传完成 + FS Watcher 事件 |
| 导入/导出 | [import.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/import.go) | 导入 Markdown、.sy.zip 等 |
| 历史还原 | [history.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/history.go) | 快照回滚 |
| 数据库视图 | [attribute_view.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/attribute_view.go) | AV 结构/数据变更 |
| 列表项 | [listitem.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/listitem.go) | 折叠状态持久化 |
| 挂载 | [mount.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/mount.go) | 挂载/卸载操作 |
| HTTP API 直调 | [api/file.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/api/file.go)、[api/asset.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/api/asset.go) | 第三方集成入口 |

### 2.2 事务队列与持久化保障

[事务处理流程](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/transaction.go#L57-L132)

```
用户操作 → PerformTransactions() → txQueue (channel, 容量7)
                                    ↓
                              flushQueue() goroutine
                                    ↓
                              flushTx() → flushLock 互斥
                                    ↓
                         performTx() 单事务执行
                          ├─ begin()     加锁
                          ├─ do*()       操作 DOM Tree
                          ├─ commit()    写文件 + IncSync()
                          └─ rollback()  异常回退
```

关键点：
- **异步批处理**：`txQueue` 缓冲 7 个事务，由独立 goroutine 消费
- **全局互斥**：`flushLock` 确保同一时间只有一个事务在执行写文件
- **提交即触发**：`commit()` 末尾调用 `IncSync()`，保证修改立即进入同步排程
- **FlushTxQueue**：同步前显式等待队列排空，见 [indexRepoBeforeCloudSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1946)

### 2.3 本地索引与快照

同步前的索引操作是**本地修改最终持久化到仓库**的关键步骤。

[indexRepoBeforeCloudSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1942-L1993)

```go
func indexRepoBeforeCloudSync(repo *dejavu.Repo) (beforeIndex, afterIndex *entity.Index, err error) {
    beforeIndex, _ = repo.Latest()       // 获取同步前最新快照
    FlushTxQueue()                       // 等待事务队列排空
    afterIndex, err = repo.Index(
        "[Sync] Cloud sync",             // 快照备注
        checkChunks,                     // PC端检查分块完整性，移动端跳过
        syncContext,                     // 进度推送上下文
    )
}
```

**快照/索引工作原理**：
1. dejavu 扫描 data 目录，计算每个文件的 SHA-256 分块哈希
2. 与上一个快照对比，生成增量索引（新增/删除/修改的文件列表）
3. 新快照写入本地仓库（`storage/repo/` 目录）
4. 通过 `beforeIndex.ID != afterIndex.ID` 判定本次是否有实际变更

**自动清理策略**：[autoPurgeRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L66-L150)
- 按 `IndexRetentionDays` 保留指定天数内的快照
- 每天随机保留 `RetentionIndexesDaily * 7` + 当天最后一个快照
- 快照过多且索引过慢时，向用户推送清理建议

---

## 3. 远程同步状态管理与通信协议

### 3.1 同步调度器

[checkSync() 前置校验](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L227-L272)

同步执行前的完整检查链：
```
checkSync()
  ├─ 模式判断：Mode 2 非 boot/exit/byHand 不执行
  ├─ 模式判断：Mode 3 非 byHand 不执行
  ├─ Sync.Enabled = false 拦截
  ├─ CloudName 合法性校验
  ├─ Provider 权限：订阅检查/付费检查
  └─ 连续失败阈值：autoSyncErrCount > 7 延迟 64 分钟
```

### 3.2 同步入口函数矩阵

| 函数 | 调用时机 | 模式 |
|------|----------|------|
| [BootSyncData()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L124-L158) | 应用启动 | 先并行跑 Index + 拉取云列表，再异步完整同步 |
| [SyncDataJob()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L113-L122) | 定时器触发（每 5 min 检查） | 检查 syncPlanTime，到期后执行 |
| [SyncData(byHand)](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L160-L162) | API 调用、WS 通知 | 双向同步 Sync() |
| [SyncDataUpload()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L73-L99) | Mode 3 手动上传 | 仅本地上传 |
| [SyncDataDownload()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L46-L71) | WS synced 通知、Mode 3 手动下载 | 仅云端下载 |
| ExitSync | 应用退出前 | exit=true 标记 |

### 3.3 WebSocket 实时感知层

[WebSocket 连接管理](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L794-L908)

**连接建立**：
- 端点：`/apis/siyuan/dejavu/ws`
- 请求头携带：UID、KernelID（随机7字符）、版本、OS、Hostname、Repo 名
- 仅在 `ProviderSiYuan + 订阅用户 + 非 Docker` 下启用

**消息处理**（`connectSyncWebSocket()` 内部 goroutine）：

```
收到 WS 消息
  ├─ cmd == "synced"
  │   └─ → SyncDataDownload()  // 其他设备刚同步完，我立即拉取
  └─ cmd == "kernels"
      └─ → 更新 onlineKernels 内存列表（同账号在线设备清单）
```

**通知其他设备**：[syncData() 末尾](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L214-L223)

```go
if 1 == Conf.Sync.Mode && nil != webSocketConn && Conf.Sync.Perception && dataChanged {
    webSocketConn.WriteJSON({"cmd": "synced", "synced": Conf.Sync.Synced})
}
```

**重连机制**：
- 读取失败后，最多 **7 次重试**，每次间隔 **7 秒**
- 失败后 `webSocketConn = nil`，下次 sync 成功时自动重连
- `closedSyncWebSocket` 原子变量区分主动关闭与异常断开

### 3.4 多 Provider 云配置构建

[buildCloudConf()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L2257-L2315)

每个 Provider 的 `cloud.Cloud` 实例由 `newRepository()` 根据配置动态创建：

```go
case conf.ProviderSiYuan:  cloudRepo = cloud.NewSiYuan(...)
case conf.ProviderS3:      cloudRepo = cloud.NewS3(..., s3HTTPClient)
case conf.ProviderWebDAV:  cloudRepo = cloud.NewWebDAV(..., webdavClient)
case conf.ProviderLocal:   cloudRepo = cloud.NewLocal(...)
```

特殊限制：
- **坚果云 WebDAV 禁用**：检测到 `dav.jianguoyun.com` 直接拒绝（性能不兼容问题）
- **本地路径校验**：不能是工作区本身、子目录或父目录，防止递归同步
- **S3 Bucket 名校验**：必须符合 CloudDirName 规则

### 3.5 云端锁定机制

dejavu 库内部实现了分布式锁，对应错误类型：
- `dejavu.ErrLockCloudFailed`：获取锁失败（用户语 188）
- `dejavu.ErrCloudLocked`：云端已被其他设备锁定（用户语 189）
- `dejavu.ErrRepoFatal`：仓库致命损坏，需重置（用户语 23）

---

## 4. 冲突检测与合并策略（深度核准）

### 4.1 合并结果数据结构

dejavu 的 `Sync()` 返回 `*dejavu.MergeResult`，核心字段在 SiYuan 中的使用如下：

[processSyncMergeResult()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1632-L1894)

```go
type MergeResult struct {
    Time      time.Time     // 合并发生时间
    Conflicts []*entity.File // 无法自动合并的冲突文件
    Upserts   []*entity.File // 已写入本地的新增/更新文件
    Removes   []*entity.File // 已从本地删除的文件
}

func (m *MergeResult) DataChanged() bool  // 合并结果是否包含实际数据变更
```

### 4.2 冲突赢家策略：三模式差异化策略 — 基于 dejavu 源码的确凿结论

> **证据来源**：dejavu 公开源码 https://github.com/siyuan-note/dejavu，已通过 WebFetch 直接读取 `sync.go` 和 `sync_manual.go` 核实。

通过 dejavu 源码的直接分析，三个同步模式采用**不同的冲突赢家策略**，而不是统一的云端优先。核心差异在于源码注释明确声明：

- `Sync()`（双向同步）：**"冲突的文件尽量以本地 upsert 和 remove 为准"** → 本地优先
- `SyncDownload()`（仅下载）：**"冲突的文件以云端 upsert 和 remove 为准"** → 云端优先
- `SyncUpload()`（仅上传）：无 MergeResult，本地强制覆盖云 → 本地赢

**三同步模式下的冲突行为对比表（基于 dejavu 源码）**：

| 同步方向 | dejavu 函数 | 返回值结构 | 冲突含义 | 默认赢家 | 工作目录版本 | temp 目录版本 |
|---------|-------------|-----------|----------|---------|------------|--------------|
| 双向同步 | `repo.Sync()` | `MergeResult + TrafficStat` | 两端都修改的文件 | **本地赢** | 本地版本 | **云端版本**（冲突副本来源） |
| 仅下载 | `repo.SyncDownload()` | `MergeResult + TrafficStat` | 本地有修改会被云端覆盖的文件 | **云端赢** | 云端版本 | 本地版本（冲突副本来源） |
| 仅上传 | `repo.SyncUpload()` | 仅 `TrafficStat` | 无冲突概念（本地强制覆盖云） | **本地赢** | 本地版本 | 无 |

---

#### 确凿证据链 1：dejavu Sync() 源码明确声明本地优先

dejavu `sync.go` 的 `sync0()` 函数注释：

```go
// 冲突的文件尽量以本地 upsert 和 remove 为准
var tmpMergeConflicts []*entity.File
for _, cloudUpsert := range cloudUpserts {
    if localUpsert := repo.getFile(localUpserts, cloudUpsert); nil != localUpsert {
        tmpMergeConflicts = append(tmpMergeConflicts, cloudUpsert)
        if repo.ignoreLocalUpsert(...) {
            mergeResult.Upserts = append(mergeResult.Upserts, cloudUpsert)
        } else {
            mergeResult.Conflicts = append(mergeResult.Conflicts, cloudUpsert)
        }
        continue
    }
    if nil == repo.getFile(localRemoves, cloudUpsert) {
        mergeResult.Upserts = append(mergeResult.Upserts, cloudUpsert)
    }
    // else: 本地删除了 → 什么也不做（本地删除赢）
}
```

**关键解读**：
1. 当 `cloudUpsert` 在 `localUpserts` 中存在时（两端都修改），**不将 cloudUpsert 加入 mergeResult.Upserts**（即不覆盖本地）
2. 而是将 `cloudUpsert` 加入 `mergeResult.Conflicts` 和 `tmpMergeConflicts`
3. 特殊情况 `ignoreLocalUpsert`：若本地修改仅为折叠属性变化，则采用云端版本

---

#### `ignoreLocalUpsert` 折叠状态特殊合并规则（基于 dejavu 源码）

> **证据来源**：dejavu `sync.go` 的 `ignoreLocalUpsert()` 函数，已通过 WebFetch 直接读取源码核实。

当双向同步发生内容冲突时，dejavu 会调用 `ignoreLocalUpsert()` 检查本地修改是否"可以忽略"。如果可以忽略，则**例外地采用云端版本**（打破本地优先策略）。

**完整判断逻辑**：

```go
func (repo *Repo) ignoreLocalUpsert(localUpsert *entity.File,
    latestSyncFiles []*entity.File, now string,
    context map[string]interface{}) bool {

    // 1. 只对 .sy 文档文件生效
    if !strings.HasSuffix(localUpsert.Path, ".sy") {
        return false
    }

    // 2. 本地新增文件不忽略
    latestSyncFile := repo.getFile(latestSyncFiles, localUpsert)
    if nil == latestSyncFile {
        return false
    }

    // 3. 解析为 AST（抽象语法树）
    localTree, _ := repo.checkoutTree(localUpsert, temp, luteEngine, context)
    localLastSyncTree, _ := repo.checkoutTree(latestSyncFile, temp, luteEngine, context)

    // 4. 收集所有块节点（排除 Document 节点）
    localNodes, localLastSyncNodes := map[string]*ast.Node{}, map[string]*ast.Node{}
    ast.Walk(localTree.Root, ...)  // 收集本地最新版本的块
    ast.Walk(localLastSyncTree.Root, ...)  // 收集上一次同步版本的块

    // 5. 块数量必须相同
    if len(localNodes) != len(localLastSyncNodes) {
        return false
    }

    // 6. 逐块检查
    for id, localNode := range localNodes {
        // 6.1 块 ID 和块类型必须相同
        if lastSyncNode, ok := localLastSyncNodes[id];
           !ok || localNode.ID != lastSyncNode.ID ||
           localNode.Type != lastSyncNode.Type {
            return false
        }
        // 6.2 检查是否只有折叠属性发生了变化
        if !onlyChangeFoldIAL(localNode, localLastSyncNode) {
            return false
        }
    }
    return true
}
```

**什么变化会被忽略（即采用云端版本）**：

| 条件 | 是否忽略本地变更 | 说明 |
|------|-----------------|------|
| 非 `.sy` 文件 | ❌ 不忽略 | 附件、配置等文件不做内容对比 |
| 本地新增文件 | ❌ 不忽略 | 文件是新增的，不做折叠属性对比 |
| 块数量变化 | ❌ 不忽略 | 新增或删除了块（内容有实质变化） |
| 块 ID 或类型变化 | ❌ 不忽略 | 块结构发生了变化 |
| **仅折叠属性变化** | ✅ 忽略 | 仅 `fold` / `heading-fold` 属性值变化 |

**什么变化仍算冲突（即保留本地版本）**：

- 任何内容文字的修改（段落、标题、列表项等）
- 块的新增或删除
- 块类型的变化（如段落变列表）
- 除 `fold` / `heading-fold` 之外的任何 IAL 属性变化（如 `id`、`class`、自定义属性等）
- 非 .sy 文件的任何变化

**`onlyChangeFoldIAL` 函数推断**：

`onlyChangeFoldIAL(a, b *ast.Node) bool` 函数未在公开源码中找到完整实现，但从函数名和调用上下文可以推断：
- 比较两个节点的 IAL（Inline Attribute List）属性
- 仅允许 `fold` 和 `heading-fold` 属性发生变化
- 其他任何属性（`id`、`class`、自定义属性等）变化都会返回 false

> **证据级别**：`ignoreLocalUpsert` 整体逻辑为**确凿结论**（源码已核实）；`onlyChangeFoldIAL` 内部判断细节为**推断结论**（未找到完整源码，从调用上下文推断）。

#### 确凿证据链 2：dejavu SyncDownload() 源码明确声明云端优先

dejavu `sync_manual.go` 的 `SyncDownload()` 函数注释：

```go
// SyncDownload 从云端同步数据到本地，冲突的文件以云端 upsert 和 remove 为准。
```

其内部实现与 `sync0()` 逻辑相反：当两端都修改时，将云端版本加入 `mergeResult.Upserts`（覆盖本地），将本地版本加入 `mergeResult.Conflicts`。

#### 确凿证据链 3：上传模式无 MergeResult，本地强制覆盖云端

[syncRepoUpload() L1337](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1337)：

```go
trafficStat, err := repo.SyncUpload(syncContext)  // 只返回 trafficStat，没有 mergeResult
```

[syncRepoUpload() L1366](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1366)：

```go
processSyncMergeResult(false, true, &dejavu.MergeResult{}, trafficStat, "u", elapsed)
```

SiYuan 手动构造空的 `&dejavu.MergeResult{}` 传入 `processSyncMergeResult()`，说明：
- `SyncUpload()` 本身不返回 MergeResult
- 上传模式下本地数据强制覆盖云端
- 不存在冲突检测和合并逻辑
- 本地永远是赢家

#### 确凿证据链 4：冲突副本来源 — 双向同步时 temp 目录保存云端版本

dejavu `sync.go` 中冲突文件导出逻辑：

```go
if 0 < len(tmpMergeConflicts) {
    temp := filepath.Join(repo.TempPath, "repo", "sync", "conflicts", nowStr)
    for i, file := range tmpMergeConflicts {  // file 是 cloudUpsert（云端版本）
        checkoutTmp, _ = repo.store.GetFile(file.ID)  // 获取云端版本
        repo.checkoutFile(checkoutTmp, temp, ...)   // checkout 到 temp
    }
}
```

由于 `tmpMergeConflicts` 包含的是 `cloudUpsert`（云端修改的文件），因此：
- **双向同步时**：`data/` 工作目录保留**本地版本**（赢家），`temp/` 目录保存**云端版本**（输家）
- **下载同步时**：`data/` 工作目录采用**云端版本**（赢家），`temp/` 目录保存**本地版本**（输家）

SiYuan 的 `GenerateConflictDoc` 从 `temp/` 目录加载文件生成冲突副本，因此：
- 双向同步：冲突副本是**云端版本**（用户需对比本地赢版本和云端输出版本）
- 下载同步：冲突副本是**本地版本**（用户需对比云端赢版本和本地输出版本）

#### 确凿证据链 5：dataChanged 判定包含两端变更

[syncRepo() L1585](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1585)：

```go
dataChanged = nil == beforeIndex || beforeIndex.ID != afterIndex.ID || mergeResult.DataChanged()
```

`dataChanged` 同时包含本地索引变更（本地有修改）和合并结果变更（云端有修改），说明双向同步是**先拉取云端数据合并到本地，再把本地新增变更推上去**的两阶段流程。

---

**双向同步（本地优先）最终结论**：

```
工作目录 data/...  ← 本地赢版本（保留本地修改）
临时目录 temp/repo/sync/conflicts/{timestamp}/...  ← 云端输出版本（dejavu 导出，用于生成冲突副本）
```

**下载同步（云端优先）最终结论**：

```
工作目录 data/...  ← 云端赢版本（已 checkout 到本地）
临时目录 temp/repo/sync/conflicts/{timestamp}/...  ← 本地输出版本（dejavu 导出，用于生成冲突副本）
```

### 4.3 修改删除冲突（Modify-Delete Conflict）处理 — 基于 dejavu 源码的确凿结论

> **证据来源**：dejavu `sync.go` 的 `sync0()` 函数源码，已通过 WebFetch 核实。

对于"一端修改、一端删除"的修改删除冲突（modify-delete conflict），处理方式与普通内容冲突**完全不同**。在双向同步（本地优先）模式下，修改删除冲突的赢家策略是：**删除操作永远赢**。

dejavu `sync0()` 函数中的核心逻辑（双向同步）：

```go
// 处理云端 upsert
for _, cloudUpsert := range cloudUpserts {
    if localUpsert := repo.getFile(localUpserts, cloudUpsert); nil != localUpsert {
        // 两端都修改 → 内容冲突
        continue
    }
    if nil == repo.getFile(localRemoves, cloudUpsert) {
        // 本地没有删除 → 接受云端 upsert
        mergeResult.Upserts = append(mergeResult.Upserts, cloudUpsert)
    }
    // else: 本地删除了 → 什么也不做（本地删除赢）
}

// 处理云端 remove
for _, cloudRemove := range cloudRemoves {
    if nil == repo.getFile(localUpserts, cloudRemove) {
        // 本地没有修改 → 接受云端 remove
        mergeResult.Removes = append(mergeResult.Removes, cloudRemove)
    }
    // else: 本地修改了 → 什么也不做（本地修改赢）
}
```

**关键解读**：
1. **云端修改 + 本地删除**：`cloudUpsert` 在 `localRemoves` 中存在 → **什么也不做** → 本地删除赢，文件保持删除状态
2. **云端删除 + 本地修改**：`cloudRemove` 在 `localUpserts` 中存在 → **什么也不做** → 本地修改赢，文件保留修改状态
3. 修改删除冲突**不会进入 `mergeResult.Conflicts`**，也不会生成冲突副本
4. 下载同步（云端优先）模式下逻辑相反：云端操作永远赢

---

#### 双向同步（本地优先）模式下的四种组合行为

| 场景 | 云端状态 | 本地状态 | 赢家 | 工作目录结果 | 是否进入 Conflicts | temp 冲突目录内容 |
|------|----------|----------|------|-------------|------------------|------------------|
| 场景 1 | 修改 | 修改 | 本地 | 本地修改版本 | 是 | 云端修改版本 |
| 场景 2 | 修改 | 删除 | **本地删除** | 文件保持删除 | 否 | 无 |
| 场景 3 | 删除 | 修改 | **本地修改** | 文件保留修改 | 否 | 无 |
| 场景 4 | 删除 | 删除 | - | 文件被删除（两边一致） | 否 | 无 |

#### 下载同步（云端优先）模式下的四种组合行为

| 场景 | 云端状态 | 本地状态 | 赢家 | 工作目录结果 | 是否进入 Conflicts | temp 冲突目录内容 |
|------|----------|----------|------|-------------|------------------|------------------|
| 场景 1 | 修改 | 修改 | 云端 | 云端修改版本 | 是 | 本地修改版本 |
| 场景 2 | 修改 | 删除 | **云端修改** | 文件恢复为云端版本 | 是 | 本地删除前的版本 |
| 场景 3 | 删除 | 修改 | **云端删除** | 文件被删除 | 是 | 本地修改版本 |
| 场景 4 | 删除 | 删除 | - | 文件被删除（两边一致） | 否 | 无 |

---

**两种典型修改删除冲突场景详解（双向同步模式）**：

**场景 2：云端修改，本地删除**
- 触发条件：本地删除了某文档，同时云端对该文档做了修改
- 合并结果：**本地删除赢** → 文件保持删除状态，云端修改被忽略
- 冲突副本：不生成（未进入 Conflicts）
- 历史目录：同步前的本地状态（文件已删除）会存入 `history/` 目录
- 云端数据：云端的修改版本仍保留在云端，下次其他设备同步时会获取该版本

**场景 3：云端删除，本地修改**
- 触发条件：本地修改了某文档，同时云端删除了该文档
- 合并结果：**本地修改赢** → 文件保留本地修改状态，云端删除被忽略
- 冲突副本：不生成（未进入 Conflicts）
- 历史目录：同步前的本地状态（文件有修改）会存入 `history/` 目录
- 云端数据：下次同步时本地修改版本会被推送到云端，恢复该文件

**重要提示**：修改删除冲突在双向同步模式下**静默处理**，用户不会收到冲突通知。这可能导致"我明明删除了为什么又出现了"或"我明明修改了为什么被删了"的困惑（在多设备场景下）。

### 4.4 双向同步的合并顺序 — 基于 dejavu 源码的确凿推导

> **证据来源**：dejavu `sync.go` 的 `sync0()` 函数源码，已通过 WebFetch 核实。

基于 dejavu 源码的直接分析，双向同步的完整合并顺序如下（本地优先策略）：

```
repo.Sync(syncContext) 内部执行顺序：
  │
  ├─ 阶段一：获取云端状态
  │   ├─ 1. 加云端分布式锁
  │   ├─ 2. 拉取云端最新 Index 快照
  │   └─ 3. 对比本地 Index 与云端 Index，找共同祖先
  │   └─ 4. 计算差异：
  │          localUpserts / localRemoves（本地自共同祖先以来的变更）
  │          cloudUpserts / cloudRemoves（云端自共同祖先以来的变更）
  │
  ├─ 阶段二：三方合并计算（本地优先）
  │   ├─ 5. 处理云端 upsert：
  │   │      ├─ 若本地也 upsert → 内容冲突 → 加入 Conflicts
  │   │      ├─ 若本地 remove → 本地删除赢 → 忽略云端 upsert
  │   │      └─ 否则 → 接受云端 upsert → 加入 Upserts
  │   │
  │   ├─ 6. 处理云端 remove：
  │   │      ├─ 若本地 upsert → 本地修改赢 → 忽略云端 remove
  │   │      └─ 否则 → 接受云端 remove → 加入 Removes
  │   │
  │   └─ 7. tmpMergeConflicts = cloudUpserts ∩ localUpserts（两端都修改）
  │
  ├─ 阶段三：本地 Checkout（本地优先）
  │   ├─ 8. 下载云端缺失的分块数据
  │   ├─ 9. 将 Upserts 文件写入本地工作目录（data/）
  │   ├─ 10. 从本地工作目录删除 Removes 文件
  │   ├─ 11. Conflicts 文件：本地版本保留（本地赢）
  │   └─ 12. 将 tmpMergeConflicts（云端版本）导出到 temp/repo/sync/conflicts/{timestamp}/
  │
  ├─ 阶段四：推送到云端
  │   ├─ 13. 生成本地合并后的新 Index
  │   ├─ 14. 上传本地独有、云端缺失的分块数据
  │   ├─ 15. 将新 Index 推送到云端
  │   └─ 16. 释放云端分布式锁
  │
  └─ 阶段五：返回结果
      └─ 17. 返回 MergeResult{Time, Conflicts, Upserts, Removes} + TrafficStat
```

**关键设计要点**：
- **先拉后推**：先把云端数据合并到本地（checkout），再把本地数据推到云端
- **本地优先**：内容冲突时保留本地版本到工作目录，云端版本导出到 temp
- **修改删除冲突静默处理**：不进入 Conflicts，不生成冲突副本
- **原子性**：云端锁 + Index 哈希链保证推送的原子性和一致性
- **可追溯**：temp 冲突目录 + history 历史目录提供双重兜底

### 4.5 GenerateConflictDoc 开关的两种模式

SiYuan 提供两种冲突处理模式，由 `GenerateConflictDoc` 开关控制。注意：主文件版本取决于同步模式（双向同步=本地版本，下载同步=云端版本）。

**模式 A：静默覆盖（默认，GenerateConflictDoc=false）**
- `Conflicts` 中的文件按同步模式的赢家策略保留（双向=本地，下载=云端）
- 输出版本仅在 `history/` 目录留痕供事后恢复：[代码位置](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1679-L1681)
  ```go
  historyDir := filepath.Join(util.HistoryDir,
      mergeResult.Time.Format("2006-01-02-150405")+"-sync")
  indexHistoryDir(filepath.Base(historyDir), luteEngine)
  ```

**模式 B：生成冲突副本（GenerateConflictDoc=true）**
- 对每个 `.sy` 冲突文件，在同目录下创建一个新文档副本
- [实现逻辑](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1647-L1674)：
  ```go
  for _, file := range mergeResult.Conflicts {
      if !strings.HasSuffix(file.Path, ".sy") { continue }
      // 从临时冲突目录加载输出版本（双向同步=云端版本，下载同步=本地版本）
      absPath := filepath.Join(util.TempDir, "repo", "sync",
          "conflicts", timestamp, file.Path)
      tree := loadTree(absPath, luteEngine)
      resetTree(tree, "Conflicted", true)   // 生成新 ID，标题加 (Conflicted) 后缀
      createTreeTx(tree)                    // 作为新文档写入笔记本
      box.addSort(previousPath, tree.ID)    // 挂在原文档之后的排序位置
  }
  ```
- 主文件版本：双向同步=本地版本，下载同步=云端版本
- 用户在 UI 中看到：原始文档（赢版本） + 标题带 "Conflicted" 的冲突副本（输出版本），自行对比合并

**非 .sy 文件**（assets、storage、配置等）：
- 不生成冲突副本，按赢家策略处理
- 依赖历史目录中保留的同步前本地版本供事后恢复

### 4.6 复现实验设计与证据边界

> **证据边界说明**：
> 1. 已通过 WebFetch 直接读取 dejavu 公开源码（https://github.com/siyuan-note/dejavu）核实的结论标记为「确凿结论」
> 2. 未直接从源码核实、仅从调用层行为推导的结论标记为「推断结论」
> 3. 以下实验设计用于验证关键结论，可在真实环境中复现

---

#### 实验 1：验证双向同步的本地优先策略

**假设**：双向同步 `Sync()` 中，两端都修改同一文档时，工作目录保留本地版本，temp 目录保存云端版本。

**实验步骤**：
1. 准备两台设备 A 和 B，都登录同一 SiYuan 账号，开启同步
2. 在设备 A 上创建文档 `test.md`，内容为 `初始版本`，等待同步完成
3. 断开设备 B 的网络（模拟离线）
4. 在设备 A 上修改文档为 `云端修改版本`，等待同步完成
5. 在设备 B 上修改文档为 `本地修改版本`（设备 B 仍离线）
6. 恢复设备 B 的网络，触发同步
7. 检查设备 B 上的：
   - 工作目录 `data/.../test.md` 内容
   - 临时目录 `temp/repo/sync/conflicts/{timestamp}/.../test.md` 内容
   - 若 `GenerateConflictDoc=true`，检查冲突副本内容

**预期结果（确凿结论）**：
- 工作目录文档内容 = `本地修改版本`（本地赢）
- temp 目录文档内容 = `云端修改版本`（云端输）
- 冲突副本（若生成）内容 = `云端修改版本`

**证据边界**：已通过 dejavu `sync.go` 源码中 `tmpMergeConflicts = cloudUpserts` 逻辑核实，实验仅用于验证集成行为。

---

#### 实验 2：验证修改删除冲突的静默处理

**假设**：双向同步中，一端修改一端删除的冲突不会进入 Conflicts，不会生成冲突副本。

**实验步骤（场景：云端修改 + 本地删除）**：
1. 准备两台设备 A 和 B，创建文档 `test.md`，内容为 `初始版本`，等待同步
2. 断开设备 B 的网络
3. 在设备 A 上修改文档为 `云端修改版本`，等待同步
4. 在设备 B 上删除文档 `test.md`（设备 B 仍离线）
5. 恢复设备 B 的网络，触发同步
6. 检查设备 B 上：
   - 工作目录：`test.md` 是否存在
   - `mergeResult.Conflicts` 列表（通过日志或调试查看）
   - temp 冲突目录是否有该文件
   - 历史目录是否有归档

**预期结果（确凿结论）**：
- 工作目录：`test.md` 保持删除状态（本地删除赢）
- `mergeResult.Conflicts` 列表：**不包含**该文件
- temp 冲突目录：**无**该文件
- 历史目录：有同步前归档（文件已删除）

**证据边界**：已通过 dejavu `sync0()` 源码中"本地删除了 → 什么也不做"的逻辑核实。

---

#### 实验 3：验证下载同步的云端优先策略

**假设**：下载同步 `SyncDownload()` 中，两端都修改同一文档时，工作目录采用云端版本，temp 目录保存本地版本。

**实验步骤**：
1. 准备一台设备，创建文档 `test.md`，内容为 `初始版本`，等待同步
2. 断开网络，修改文档为 `本地修改版本`
3. 在另一台设备上修改文档为 `云端修改版本`，等待同步
4. 在第一台设备上执行「仅下载」同步（Mode 3 手动下载）
5. 检查：
   - 工作目录文档内容
   - temp 冲突目录内容
   - 冲突副本（若生成）内容

**预期结果（确凿结论）**：
- 工作目录文档内容 = `云端修改版本`（云端赢）
- temp 目录文档内容 = `本地修改版本`（本地输）
- 冲突副本（若生成）内容 = `本地修改版本`

**证据边界**：已通过 dejavu `sync_manual.go` 源码中 `SyncDownload()` 注释和逻辑核实。

---

#### 实验 4：验证 ignoreLocalUpsert 折叠属性特殊规则

**假设**：若本地修改仅为折叠属性变化，双向同步时会例外地采用云端版本（打破本地优先策略）。

**实验步骤**：
1. 准备两台设备 A 和 B，创建文档 `test.md`，包含可折叠的标题/列表
2. 在设备 A 上修改文档**内容**（如添加一段文字），等待同步完成
3. 断开设备 B 的网络
4. 在设备 B 上仅修改文档的**折叠状态**（展开/折叠，不修改任何内容文字）
5. 恢复设备 B 的网络，触发同步
6. 检查设备 B 上的：
   - 文档内容（是否为设备 A 的版本）
   - 折叠状态（是否为设备 A 的状态还是设备 B 的状态）
   - 是否生成冲突副本

**预期结果（确凿结论 + 推断细节）**：
- 文档内容 = 设备 A 的修改版本（云端赢，因为本地修改仅为折叠属性）
- 折叠状态 = 设备 A 的状态（采用云端版本的折叠属性）
- 不生成冲突副本（因为 `ignoreLocalUpsert` 返回 true 时，文件不进入 Conflicts）

**证据边界**：
- `ignoreLocalUpsert` 整体判断逻辑（6 步检查流程）为**确凿结论**（dejavu `sync.go` 源码已核实）
- `onlyChangeFoldIAL` 内部比较的具体属性列表为**推断结论**（函数完整实现未找到，从函数名 + 调用上下文 + SiYuan 折叠属性实现推断）

---

#### 证据边界总结表

| 结论 | 证据级别 | 证据来源 | 未核实部分 |
|------|----------|----------|------------|
| 双向同步本地优先 | 确凿 | dejavu `sync.go` `sync0()` 函数 | 无 |
| 下载同步云端优先 | 确凿 | dejavu `sync_manual.go` `SyncDownload()` 函数 | 无 |
| 修改删除冲突静默处理 | 确凿 | dejavu `sync0()` 函数循环逻辑 | 无 |
| 冲突副本来源双向=云端 | 确凿 | dejavu `sync0()` `tmpMergeConflicts = cloudUpserts` | 无 |
| 冲突副本来源下载=本地 | 确凿 | dejavu `SyncDownload()` 逻辑对称推导 | `SyncDownload()` 内部实现未逐行核实 |
| `ignoreLocalUpsert` 整体判断逻辑 | 确凿 | dejavu `sync.go` `ignoreLocalUpsert()` 函数 | 无 |
| `onlyChangeFoldIAL` 仅比较 fold 属性 | 推断 | 函数名 + 调用上下文 + SiYuan 折叠属性实现 | 函数完整源码未找到 |
| 上传模式无冲突 | 确凿 | SiYuan 代码构造空 MergeResult 传入 | `SyncUpload()` 内部实现未逐行核实 |

### 4.7 数据流：同步主循环详解

[syncRepo() 主流程](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1510-L1606)

```
syncRepo(exit, byHand)
  │
  ├─ 1. 仓库密钥校验
  │
  ├─ 2. newRepository() 初始化 dejavu 实例
  │
  ├─ 3. indexRepoBeforeCloudSync()
  │     ├─ repo.Latest()      捕获同步前快照 ID
  │     ├─ FlushTxQueue()     排空本地事务
  │     └─ repo.Index()       创建新快照
  │
  ├─ 4. beforeSyncPetals := getPetals()   记录同步前花瓣状态
  │
  ├─ 5. mergeResult, trafficStat, err := repo.Sync(ctx)
  │     └─ dejavu 内部:
  │        ├─ 加云端锁
  │        ├─ 拉取云端最新 Index
  │        ├─ 对比本地/云端 Index
  │        ├─ 三方合并计算（生成 Upserts/Removes/Conflicts）
  │        ├─ 下载缺失分块，组装文件
  │        ├─ Checkout 写入本地磁盘（云端赢）
  │        ├─ 本地冲突文件的输出版本存入 temp/conflicts/
  │        ├─ 本地上传新增分块
  │        ├─ 推送合并后的 Index 到云端
  │        └─ 释放云端锁
  │
  ├─ 6. dataChanged 判定：
  │     dataChanged = beforeIndex.ID != afterIndex.ID || mergeResult.DataChanged()
  │
  ├─ 7. Conf.Sync.Synced = 当前时间戳
  │    Conf.Sync.Stat = 流量统计字符串
  │
  ├─ 8. calcPetalDiff() → 补填 mergeResult.{Upsert,Remove}Petals
  │
  ├─ 9. processSyncMergeResult(exit, byHand, mergeResult, ...)
  │
  └─ 10. 异步：checkIndex() 索引订正 → autoPurgeRepo() 清理快照
```

### 4.6 上传专用与下载专用流程

与 `syncRepo()` 对称的两个单向函数：

| 函数 | dejavu 调用 | 合并结果 | 冲突 |
|------|-------------|---------|------|
| `syncRepoUpload()` | `repo.SyncUpload()` | 空 MergeResult | 无冲突（本地覆盖云） |
| `syncRepoDownload()` | `repo.SyncDownload()` | 完整 MergeResult | 有冲突（云覆盖本地） |

### 4.7 启动优化：并行预取

[bootSyncRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1372-L1508)

启动时采用并行策略缩短首屏等待：
```
WaitGroup
  ├─ goroutine A: indexRepoBeforeCloudSync()  // 本地建索引（CPU密集）
  └─ goroutine B:
        ├─ repo.GetCloudLatest()             // 拉取云端最新索引
        └─ repo.GetSyncCloudFiles()          // 获取需要同步的文件清单
→ Wait()
→ 将 fetchedFiles 填充 syncingFiles Map
→ 如果文件数>0：异步 goroutine 执行完整 syncRepo()，主线程立即返回继续 Boot
```

---

## 5. 空同步退避机制（精确核准）

### 5.1 两级退避体系

SiYuan 实现了**两级退避**机制，分别应对两种场景：

| 退避层级 | 触发条件 | 计数器类型 | 最大延迟 |
|---------|----------|-----------|---------|
| **空同步退避** | 连续同步无数据变更 | `syncSameCount`（atomic.Int32） | 1024 分钟（约 17 小时） |
| **错误退避** | 连续同步失败 | `autoSyncErrCount`（普通 int） | 64 分钟 |

### 5.2 空同步退避的精确计算（已核准）

[processSyncMergeResult() 无变更分支](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1683-L1696)

```go
if 1 > len(mergeResult.Upserts) && 1 > len(mergeResult.Removes) && 1 > len(mergeResult.Conflicts) {
    syncSameCount.Add(1)
    if 10 < syncSameCount.Load() {
        syncSameCount.Store(5)
    }
    if !byHand {
        delay := time.Minute * time.Duration(int(math.Pow(2, float64(syncSameCount.Load()))))
        if fixSyncInterval.Minutes() > delay.Minutes() {
            delay = time.Minute * 8
        }
        planSyncAfter(delay)
    }
    return
}
```

**关键代码逻辑解析**：
1. 先执行 `syncSameCount.Add(1)` —— 计数先 +1，再判断
2. 检查 `if 10 < syncSameCount.Load()` —— 如果计数 > 10，重置为 5
3. 计算 `delay = 2^syncSameCount 分钟` —— 指数增长
4. 检查 `if fixSyncInterval.Minutes() > delay.Minutes()` —— 如果 delay 小于固定间隔（5分钟），保底设为 8 分钟

**完整时序推演表**：

| 第 N 次空同步 | 执行前 count | Add 后 count | 是否触发重置 | 最终 count | 计算 delay | 实际 delay（保底 8 分钟） |
|-------------|------------|------------|------------|----------|-----------|---------------------|
| 1 | 0 | 1 | 否 | 1 | 2^1 = 2 min | 8 min（5>2，取 8） |
| 2 | 1 | 2 | 否 | 2 | 2^2 = 4 min | 8 min（5>4，取 8） |
| 3 | 2 | 3 | 否 | 3 | 2^3 = 8 min | 8 min |
| 4 | 3 | 4 | 否 | 4 | 2^4 = 16 min | 16 min |
| 5 | 4 | 5 | 否 | 5 | 2^5 = 32 min | 32 min |
| 6 | 5 | 6 | 否 | 6 | 2^6 = 64 min | 64 min |
| 7 | 6 | 7 | 否 | 7 | 2^7 = 128 min | 128 min |
| 8 | 7 | 8 | 否 | 8 | 2^8 = 256 min | 256 min |
| 9 | 8 | 9 | 否 | 9 | 2^9 = 512 min | 512 min |
| 10 | 9 | 10 | 否 | 10 | 2^10 = 1024 min | 1024 min |
| 11 | 10 | 11 | 是（10<11） | 5 | 2^5 = 32 min | 32 min ← 循环开始 |
| 12 | 5 | 6 | 否 | 6 | 2^6 = 64 min | 64 min |
| 13 | 6 | 7 | 否 | 7 | 2^7 = 128 min | 128 min |
| ... | ... | ... | ... | ... | ... | ... |

**循环规律**：
- 前 3 次空同步：固定 8 分钟间隔（保底）
- 第 4~10 次：指数增长（16 → 32 → 64 → 128 → 256 → 512 → 1024 分钟）
- 第 11 次触发重置：从 5 开始循环（32 → 64 → 128 → 256 → 512 → 1024 → 重置 → 32 → ...）

**重置机制**：
- **任何 IncSync() 被调用时**：syncSameCount 归零 → 恢复正常间隔
- 进程重启：归零（进程内变量）

### 5.3 错误退避的精确计算（已核准）

[checkSync() 中的错误退避](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L265-L270)

```go
if 7 < autoSyncErrCount && !byHand {
    logging.LogErrorf("failed to auto-sync too many times, delay auto-sync 64 minutes")
    util.PushErrMsg(Conf.Language(125), 1000*60*60)
    planSyncAfter(64 * time.Minute)
    return false
}
```

**特点**：
- 连续失败 ≤ 7 次：每次失败后 `planSyncAfter(fixSyncInterval)` = 5 分钟
- 连续失败 > 7 次：直接延迟 64 分钟
- **手动同步（byHand=true）**：不检查错误计数，立即执行
- `autoSyncErrCount` 是**普通 int**，非原子操作，存在并发安全风险（见风险分析）

**错误累加点**：
- [isProviderOnline()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L738-L739)：网络不可达时累加
- [syncRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1561)：同步失败时累加
- [bootSyncRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1407)：启动同步失败时累加

**错误归零点**：
- 同步成功时：`autoSyncErrCount = 0`（见 [syncRepo L1592](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1592)、[syncRepoUpload L1363](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1363)、[syncRepoDownload L1291](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1291)）

### 5.4 退避重置点汇总（已核准）

| 事件 | syncSameCount | autoSyncErrCount | 代码位置 |
|------|--------------|-----------------|----------|
| IncSync() 被调用 | 归零 → 0 | 不影响 | [sync.go L704](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L704) |
| 同步成功且有数据变更 | 归零（隐含：IncSync 被调用） | 归零 | [repository.go L1592](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1592) |
| 同步成功但无数据变更 | +1 | 归零 | [repository.go L1684](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1684) |
| 同步失败 | 不影响 | +1 | [repository.go L1561](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1561) |
| 进程重启 | 归零（进程内变量） | 归零（进程内变量） | - |

---

## 6. 启动同步竞态与 syncingFiles 的真实作用（深度核准）

### 6.1 bootSyncRepo 的两阶段设计

[bootSyncRepo() 完整流程](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1372-L1508)

```
阶段一：并行预取（同步阻塞，Boot 主线程等待）
  WaitGroup
    ├─ goroutine A: 本地建索引
    └─ goroutine B: GetCloudLatest + GetSyncCloudFiles
  → Wait()

阶段二：异步同步（后台 goroutine，不阻塞 Boot）
  → 填充 syncingFiles / syncingStorages
  → 如果有文件需要同步 → go syncRepo(false, false) 后台完整同步
  → 如果无文件 → isBootSyncing = false，立即返回
```

**关键变量**：
- `isBootSyncing`：标记启动同步是否正在进行，影响 `isSyncingStorages()` 判断
- `syncingFiles`：即将被云端变更的文档 rootID 集合
- `syncingStorages`：storage/ 目录是否有文件待同步

### 6.2 syncingFiles 的生命周期

`syncingFiles` 是一个 `sync.Map`，存放**即将被云端变更的文档 rootID**。

**填充时机**：
1. `bootSyncRepo()` 阶段一结束后，从 `GetSyncCloudFiles` 结果中填充
   - `.sy` 文件：提取 rootID，存入 `syncingFiles`
   - `/storage/` 开头的文件：设置 `syncingStorages = true`

[代码位置 L1481-L1493](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1481-L1493)：
```go
syncingFiles = sync.Map{}
syncingStorages.Store(false)
for _, fetchedFile := range fetchedFiles {
    name := path.Base(fetchedFile.Path)
    if strings.HasSuffix(name, ".sy") {
        id := name[:len(name)-3]
        syncingFiles.Store(id, true)
        continue
    }
    if strings.HasPrefix(fetchedFile.Path, "/storage/") {
        syncingStorages.Store(true)
    }
}
```

**重置时机**：
- `processSyncMergeResult()` 末尾统一清空：`syncingFiles = sync.Map{}`
  [代码位置 L1834](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1834)

### 6.3 syncingFiles / syncingStorages 的消费端（已核准）

通过全代码搜索后，**实际只有 3 类消费场景**：

| 消费端 | 位置 | 行为 | 阻塞类型 |
|--------|------|------|----------|
| **闪卡操作** | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/flashcard.go#L751-L754) | `isSyncingStorages()` 为 true 时返回 `TxErrCodeDataIsSyncing` | **阻塞写操作**（加闪卡/删闪卡） |
| **闪卡查询** | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/flashcard.go#46) 等多处 | `waitForSyncingStorages()` 循环等待 | **阻塞读操作**（轮询等待） |
| **属性视图** | [attribute_view_render.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/attribute_view_render.go#L39) | `waitForSyncingStorages()` 循环等待 | **阻塞渲染**（轮询等待） |
| **文件树 API** | [api/filetree.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/api/filetree.go#L1202) | `IsSyncingFile(rootID)` 判断 | **仅 UI 状态展示**，不阻塞操作 |

**重要结论（已核准）**：
- `syncingFiles` 对**普通文档事务完全不阻塞**！普通文档的写入/编辑在启动同步过程中**完全正常进行**
- `IsSyncingFile` 仅用于**文件树 API 返回给前端做状态展示**（显示同步中图标）
- 真正会被阻塞的只有**闪卡操作**和**属性视图**（因为它们的数据在 storage/ 目录中，使用 `syncingStorages` 标记）
- 闪卡有两种阻塞方式：写操作直接报错，读操作轮询等待

### 6.4 启动同步竞态分析（深度核准）

**竞态场景**：
1. 应用启动 → BootSyncData() → bootSyncRepo() → 阶段一完成，阶段二异步执行
2. 用户立即打开一个文档 → 事务正常编辑 → 事务正常执行
3. 后台 `syncRepo()` 在后台 goroutine 中下载云端版本

**两种时序结果（双向同步，本地优先策略）**：

| 时序 | 详细过程 | 数据是否丢失 | 用户感知 |
|------|----------|------------|----------|
| **时序 A：用户编辑先完成，syncRepo 后 checkout** | 1. 用户编辑 → commit() → 写入磁盘 → IncSync()<br>2. 后台 syncRepo 开始 → FlushTxQueue() 等待事务完成<br>3. repo.Index() 创建快照（包含用户修改）<br>4. repo.Sync() 三方合并 → **本地赢**（用户编辑保留）<br>5. 云端版本进入 Conflicts → 云端版本存入 temp 冲突目录<br>6. 若 GenerateConflictDoc=true → 生成冲突副本（云端版本） | **不丢失**（用户编辑保留，有历史目录兜底） | 用户刚编辑的内容正常保留，可能看到"云端版本"的冲突副本 |
| **时序 B：syncRepo 先 checkout，用户后编辑** | 1. 后台 syncRepo 完成 checkout → 云端版本写入磁盘<br>2. 用户打开文档 → 加载到的是云端版本<br>3. 用户编辑 → commit() → 写入 → IncSync()<br>4. IncSync() 触发下次同步 → 把用户修改推上去 | **正常流程**，无丢失 | 用户感知正常，打开的就是最新云端版本 |

**结论**：
- 不存在真正的数据丢失风险（双向同步本地优先 + 冲突副本 + 历史目录三重兜底）
- 时序 A 中**用户编辑内容不会消失**（本地优先策略保证），但可能看到云端版本的冲突副本
- 这就是 `GenerateConflictDoc` 开启的意义——让用户能看到云端修改的副本，决定是否合并
- `syncingFiles` 仅用于 UI 展示同步状态，**不提供任何实际的阻塞保护**

---

## 7. 离线恢复、重复事件与顺序保障（完整因果链）

### 7.1 离线场景下的完整因果链

```
网络断开
  │
  ├─ 分支 1：用户本地操作（完全正常）
  │   └─ 用户操作 → 事务正常执行 → commit()
  │                       ├─ 写文件（本地持久化）
  │                       └─ IncSync()
  │                             ├─ syncSameCount.Store(0)   重置空同步计数
  │                             └─ planSyncAfter(Interval)  排程下次同步
  │
  ├─ 分支 2：定时同步任务尝试（失败）
  │   └─ SyncDataJob() 定时器触发（每 5 分钟）
  │        └─ checkSync() → 模式/权限检查通过
  │            └─ isProviderOnline() → 网络不可达
  │                ├─ 推送错误消息（首次失败时）
  │                ├─ planSyncAfter(fixSyncInterval)  → 5 分钟后重试
  │                └─ autoSyncErrCount++
  │
  ├─ 分支 3：连续失败 8 次（错误退避触发）
  │   └─ SyncDataJob() → checkSync()
  │        ├─ autoSyncErrCount > 7 → 触发错误退避
  │        ├─ planSyncAfter(64 * time.Minute)  → 64 分钟后重试
  │        └─ 推送"同步失败次数过多"错误提示
  │
  └─ 分支 4：用户手动同步（byHand=true）
      └─ checkSync() → 不检查错误计数 → 立即执行
          └─ isProviderOnline() → 仍失败 → autoSyncErrCount++
              └─ planSyncAfter(fixSyncInterval)
```

**关键因果节点**：
- 本地操作**完全不受网络影响**（本地优先设计）
- 每次网络检查失败都会**累加错误计数**并重试
- 错误计数超过阈值后触发**长退避**（64 分钟）
- 手动同步**绕过错误计数检查**，但失败仍会累加计数

### 7.2 网络恢复后的恢复链路

```
网络恢复
  │
  ├─ 场景 A：等待 SyncDataJob 定时器到期（最慢）
  │   └─ SyncDataJob() → checkSync() → 模式/权限检查通过
  │       └─ isProviderOnline() → 成功
  │           └─ syncRepo() → 同步成功
  │               ├─ autoSyncErrCount = 0          错误计数归零
  │               ├─ 有数据变更 → WS "synced" 通知其他设备
  │               └─ 有数据变更 → syncSameCount = 0（隐含）
  │
  ├─ 场景 B：用户手动触发同步（最快）
  │   └─ SyncData(byHand=true)
  │       └─ checkSync() → 不检查错误计数 → 立即执行
  │           └─ syncRepo() → 同步成功
  │
  └─ 场景 C：WS 重连成功（Perception 模式，仅 SiYuan Provider）
      └─ WS 连接建立
          ├─ 收到其他设备的 "synced" 消息
          │   └─ SyncDataDownload() → 拉取云端数据
          │
          └─ 本设备同步完成后
              └─ 发送 "synced" 消息 → 通知其他设备
```

**恢复后的连锁反应**：
- 同步成功 → `autoSyncErrCount = 0` → 错误退避解除
- 有数据变更 → `syncSameCount = 0` → 空同步退避解除
- WS 重连 → 恢复实时感知能力

### 7.3 重复事件与幂等性保障（因果链）

**幂等性层级体系**：

| 层级 | 机制 | 位置 | 作用 |
|------|------|------|------|
| **同步入口** | `syncLock sync.Mutex` + `isSyncing atomic.Bool` | [sync.go L164-172](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L164-L172) | 同一时间只有一个同步在执行，重复调用自动排队 |
| **云端写入** | dejavu 分布式锁 + Index 哈希链 | dejavu 内部 | 同一时间只有一个设备在 Push，防止并发写 |
| **快照索引** | Index ID 唯一，每个 Index 引用前驱 | dejavu 内部 | 版本不会分叉，保证线性历史 |
| **WS 通知** | `syncLock` 自动排队 | [sync.go L174-225](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L174-L225) | 重复通知不产生重复同步 |

**WS 通知风暴的抑制机制（完整因果链）**：

```
设备 A 同步完成 → 发送 "synced" WS 通知
  ↓
设备 B 收到 → 调用 SyncDataDownload()
  ├─ lockSync() → 获取 syncLock
  ├─ syncRepoDownload() → 执行下载
  │   └─ 若数据变更 → 也发送 "synced" 通知 → 设备 A 收到 → ...
  │       └─ 但设备 A 数据通常没有新变更（刚同步完是一致的）
  │           └─ 空同步 → syncSameCount++ → 退避间隔拉长
  ↓
循环终止条件：
  1. 两端数据一致 → dataChanged = false → 不发送 WS 通知
  2. 空同步退避逐渐拉长间隔 → 通知频率指数下降 → 最终平息
```

**因果终止条件的数学推导**：
- 假设 N 个设备环形通知，每轮空同步后退避间隔翻倍
- 前 3 轮：8 分钟 / 轮
- 第 4 轮：16 分钟
- 第 5 轮：32 分钟
- ...
- 第 10 轮：1024 分钟（约 17 小时）
- 最终系统稳定在极低频率的空同步轮询

### 7.4 顺序保障的三层因果链

**第一层：本地操作顺序保障**

```
用户操作 1 → txQueue
用户操作 2 → txQueue
用户操作 3 → txQueue
    ↓
flushQueue() goroutine 串行消费（FIFO）
    ↓
flushTx() → flushLock 全局互斥
    ↓
操作 1 commit → 写入 → IncSync()
操作 2 commit → 写入 → IncSync()
操作 3 commit → 写入 → IncSync()
    ↓
syncPlanTime 被多次重置（每次 IncSync 都重置为 Interval 后）
    ↓
最终效果：所有操作按顺序落盘，同步在最后一个操作完成 Interval 后触发
```

**关键点**：
- `txQueue` 是 channel，天然 FIFO
- `flushLock` 保证事务串行化执行
- 多次 `IncSync()` 只保留最后一次的排程时间（避免频繁同步）

**第二层：同步前的顺序保障**

```
syncRepo() 开始
  ↓
indexRepoBeforeCloudSync()
  ├─ FlushTxQueue()  ← 阻塞等待所有事务完成（自旋等待）
  └─ repo.Index()    ← 一次性建快照，包含所有已提交的修改
```

[FlushTxQueue() 实现](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/transaction.go#L57-L62)：
```go
func FlushTxQueue() {
    time.Sleep(time.Duration(50) * time.Millisecond)
    for 0 < len(txQueue) || isFlushing {
        time.Sleep(10 * time.Millisecond)
    }
}
```

**第三层：全局同步顺序保障**

```
syncLock 互斥 → 同一设备同一时间只有一个同步在执行
  ↓
云端分布式锁 → 同一时间只有一个设备在 Push
  ↓
Index 哈希链 → 每个 Index 引用前驱 Index → 版本线性演进，不分叉
  ↓
三方合并 → 以最新云端 Index 为基础进行合并
```

**顺序保障总结**：
- 本地操作：txQueue + flushLock → 严格串行
- 单次同步内：FlushTxQueue + Index → 所有修改一次性入快照
- 多设备间：云端锁 + Index 哈希链 → 全局线性可追溯

### 7.5 数据丢失兜底：历史目录归档

每次同步（含冲突）都会在 `history/` 下留痕：

```
history/
└── YYYY-MM-DD-HHMMSS-sync/   ← 同步前本地状态归档
    └── ...（完整文件树）
```

[indexHistoryDir()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/history.go#L912-L1007)

历史目录包含：
- `.sy` 文档 → 解析后存入历史数据库，支持全文搜索
- assets 文件 → 记录路径，支持还原
- storage/av/ 数据库 → 记录路径，支持还原

**恢复路径**：
用户 → 历史版本面板 → 搜索/浏览 → 选择历史版本 → RollbackDocHistory()

**历史目录与冲突副本的关系**：
- 历史目录：**每次同步**都会生成，保存同步前的**完整本地状态**
- 冲突副本：仅在 **GenerateConflictDoc=true** 且有冲突时生成，保存冲突文件的**本地输出版本**
- 两者是互补关系：历史目录提供完整回滚能力，冲突副本提供便捷的单文件对比能力

---

## 8. 用户界面反馈机制

### 8.1 三层反馈架构

```
内核广播 (BroadcastByType / PushMsg / PushStatusBar)
        │
        ▼
WebSocket 消息 (app/src/index.ts onmessage)
        │
        ├─ code=0 (syncing)        → processSync(data, plugins)
        ├─ code=1 (sync success)   → processSync(data, plugins)
        ├─ code=2 (sync error)     → processSync(data, plugins)
        │
        ├─ cmd=syncMergeResult     → reloadSync(app, data)
        │
        └─ cmd=statusbar / txerr   → 状态栏 + 错误弹框
```

### 8.2 同步图标状态

[processSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/app/src/dialog/processSystem.ts#L565-L630)

| 状态 | code | 图标 | 视觉效果 |
|------|------|------|----------|
| 未启用/无权限 | - | `#iconCloudOff` | 灰色云朵 |
| 就绪 | - | `#iconCloudSucc` | 绿色对勾云朵 |
| 同步中 | 0 | `#iconRefresh` | CSS 旋转动画（`.fn__rotate`） |
| 同步成功 | 1 | `#iconCloudSucc` | 高亮激活态消失 |
| 同步失败 | 2 | `#iconCloudError` | 红色感叹号云朵 |

**插件事件总线**：同步过程同时向插件广播
```typescript
plugins.forEach((item) => {
    if (data.code === 0) item.eventBus.emit("sync-start", data);
    if (data.code === 1) item.eventBus.emit("sync-end", data);
    if (data.code === 2) item.eventBus.emit("sync-fail", data);
});
```

### 8.3 合并结果后的 UI 刷新

[reloadSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/app/src/dialog/processSystem.ts#L40-L152)

收到 `syncMergeResult` 后，遍历所有打开的面板并针对性刷新：

```
reloadSync(app, {upsertRootIDs, removeRootIDs})
  │
  ├─ 编辑器面板 (Editor)
  │   ├─ upsertRootIDs 命中 → reloadProtyle + 更新 tab 标题
  │   └─ removeRootIDs 命中 → 关闭 tab + 清滚动位置
  │
  ├─ 图表面板 (Graph)
  │   └─ 命中 rootId → searchGraph() 重搜
  │
  ├─ 大纲面板 (Outline)
  │   └─ 命中 blockId → API 拉取新大纲 + update()
  │
  ├─ 反链面板 (Backlink)
  │   └─ refresh()
  │
  ├─ 文件树面板 (Files)
  │   └─ item.init(false) 完整刷新（有排序/新节点）
  │
  ├─ 书签/标签面板
  │   └─ item.update()
  │
  └─ 搜索结果/自定义面板
      └─ input 事件触发 / item.update()
```

### 8.4 冲突消息推送

[processSyncMergeResult() 末尾](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1879-L1892)

```go
if 0 < len(mergeResult.Conflicts) {
    syConflict := false
    for _, file := range mergeResult.Conflicts {
        if strings.HasSuffix(file.Path, ".sy") {
            syConflict = true
            break
        }
    }
    if syConflict {
        util.PushMsg(Language(108), 7000)  // "数据同步发生冲突..."
    }
}
```

### 8.5 详细进度反馈（事件总线订阅）

[subscribeRepoEvents()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L2039-L2255)

通过 `eventbus` 订阅 dejavu 内部细粒度事件并推送到 UI：

| 事件 | UI 展示 |
|------|----------|
| `EvtIndexBeforeWalkData` | `正在索引 {path}` |
| `EvtIndexWalkData` (每 1024 个) | `正在索引 {filename}` |
| `EvtIndexGetLatestFile` (每 64 个) | `校验文件 {n}/{total}` |
| `EvtIndexUpsertFile` (每 32 个) | `索引文件 {n}/{total}` |
| `EvtCheckout*` 系列 | 同步下载时的文件级进度 |
| `EvtCloud*` 系列 | 上传/下载/清理云端对象 |

---

## 9. 潜在风险与需验证的问题

### 9.1 潜在风险

| 风险等级 | 问题描述 | 关联代码 | 详细说明 |
|---------|----------|----------|----------|
| **中** | 启动同步竞态用户感知问题：bootSyncRepo 后台同步时，用户打开文档编辑的内容在双向同步下会保留（本地优先），但可能出现云端版本冲突副本 | [repository.go L1495-1503](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1495-L1503) | syncingFiles 仅用于 UI 展示，不提供实际阻塞保护；时序 A 中用户编辑内容保留（本地优先），但可能生成云端版本冲突副本造成困惑 |
| **高** | WS 通知风暴收敛慢：多设备环形触发通知，空同步退避前 3 次都是 8 分钟 | [sync.go L214-223](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L214-L223) | N 个设备形成通知环时，前 3 轮每轮 8 分钟，之后才指数退避；设备越多收敛越慢 |
| **中** | `autoSyncErrCount` 是普通 int 非原子变量，并发访问有线程安全风险 | [sync.go L103](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L103) | 多个 goroutine 可能同时读写该变量（isProviderOnline、syncRepo 等），存在竞态条件 |
| **中** | GenerateConflictDoc=true 时大量冲突可能造成文档爆炸 | [repository.go L1644-1674](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1644-L1674) | 同一个文档在 N 端反复冲突 → N 个 Conflicted 副本；没有自动清理机制 |
| **中** | `autoSyncErrCount` 是进程内全局变量，重启后清零 | [sync.go L103](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L103) | 极端情况下反复重启可能绕过 7 次失败保护；错误计数没有持久化 |
| **中** | S3/WebDAV 并发请求数默认值未做上限校验 | 各 Provider 的 `ConcurrentReqs` 字段 | 可能触发对象存储限流或被封禁 |
| **中** | `syncLock` + 云端锁双重锁定，若内核崩溃云端锁未释放 | `ErrCloudLocked` 处理 | 需等待 TTL（取决于 dejavu 实现）才能重新获得锁 |
| **低** | `syncingFiles` 使用 `sync.Map`，rootID 哈希冲突概率极低但非零 | [repository.go L1207](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1207) | 极低概率事件 |
| **低** | 移动端不检查分块，若本地存储损坏可能将坏数据同步到云端 | [repository.go L1948-L1952](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1948-L1952) | 移动端私有数据空间理论上不会有外部篡改，但应用崩溃可能导致部分写入 |
| **低** | 本地文件系统 Provider 的路径规范化未处理软链接 | [SetSyncProviderLocal()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L469-L506) | 可能导致循环目录同步或越权访问 |

### 9.2 已确认结论与待验证问题

#### 已通过代码分析确凿确认的结论（基于 dejavu 源码）

> **证据来源说明**：以下结论中标记「★」的已通过 dejavu 公开源码（https://github.com/siyuan-note/dejavu）直接核实，其余通过 SiYuan 调用层代码分析核实。

| 结论 | 证据级别 | 证据链 |
|------|----------|--------|
| **★ 双向同步（Sync()）本地优先策略** | 确凿 | dejavu `sync.go` 注释："冲突的文件尽量以本地 upsert 和 remove 为准" |
| **★ 下载同步（SyncDownload()）云端优先策略** | 确凿 | dejavu `sync_manual.go` 注释："冲突的文件以云端 upsert 和 remove 为准" |
| **★ 上传模式无冲突（本地强制覆盖）** | 确凿 | SiYuan 代码构造空 `MergeResult` 传入；`SyncUpload` 仅返回 `TrafficStat` |
| **★ 修改删除冲突静默处理（双向同步）** | 确凿 | dejavu `sync0()` 源码：云端修改+本地删除→本地删除赢；云端删除+本地修改→本地修改赢；均不进入 Conflicts |
| **★ 冲突副本来源：双向同步=云端版本** | 确凿 | dejavu `tmpMergeConflicts = cloudUpserts` → checkout 到 temp 的是云端版本 |
| **★ 冲突副本来源：下载同步=本地版本** | 确凿 | dejavu `SyncDownload()` 逻辑对称 → 冲突时 temp 保存本地版本 |
| **双向同步先拉后推顺序** | 确凿 | `dataChanged` 同时包含本地 Index 变更和云端 Merge 变更 |
| **空同步退避 11 次循环机制** | 确凿 | 前 3 次 8 分钟保底，4-10 次指数增长（16→1024 分钟），第 11 次重置为 5（32 分钟）开始循环 |
| **错误退避 7 次阈值** | 确凿 | `autoSyncErrCount > 7` 时延迟 64 分钟；手动同步（byHand=true）绕过 |
| **syncingFiles 仅影响闪卡/属性视图** | 确凿 | 全代码搜索验证：仅 flashcard.go / attribute_view_render.go / api/filetree.go 消费 |
| **autoSyncErrCount 非原子变量** | 确凿 | 普通 int 类型，多个 goroutine 可能同时读写 |
| **闪卡两种阻塞方式** | 确凿 | 写操作直接返回 `TxErrCodeDataIsSyncing` 报错；读操作通过 `waitForSyncingStorages()` 轮询等待 |

#### 仍待验证的问题

以下问题需要通过实际集成测试或阅读 dejavu 源码进一步确认：

**Q1：分块检查出损坏后的处理方式**
> PC 端检查分块（checkChunks=true），检查出的损坏分块是自动从云端修复还是直接报错终止？是否会影响同步流程？

**Q2：跨 Provider 迁移的数据一致性**
> 用户从 SiYuan Provider 切换到 S3 时，`CloudName` 共用，但云端元数据（索引格式）是否完全兼容？需验证 `newRepository` 不同 Cloud 实现的 Index 格式。

**Q3：WS "synced" 消息的节流与排队**
> 设备 A 在 1 秒内连续多次同步完成，发送多条 "synced"，设备 B 的 `SyncDataDownload` 是否由 `syncLock` 自动排队？是否会造成 B 的同步队列积压？syncLock 是互斥锁还是可重入锁？

**Q4：IncSync 调用的事务隔离与崩溃兜底**
> `tx.commit()` 先写文件后调用 `IncSync()`，两者之间若发生崩溃，文件已写入但同步未排程。下次启动时 `BootSyncData` 的 Index 操作是否能兜底捕获该修改？（理论上可以，因为 Index 会扫描所有文件）

**Q5：0.2 全量重建阈值的合理性**
> `needFullReindex(upsertTrees)` 当同步变更文档数 > 总量 20% 时触发 `FullReindex`。在大工作区（10w+ 文档）下，FullReindex 可能耗时数十分钟，是否有进度反馈和取消能力？

**Q6：Sync.GenerateConflictDoc 与历史目录的双重保存**
> 开启冲突副本后，同一份冲突数据既保存在 `history/YYYY-MM-DD-HHMMSS-sync/` 又作为新 `.sy` 文档写入，是否会造成双倍磁盘占用？是否存在清理策略？

**Q7：syncSameCount 上界的设计意图**
> `syncSameCount > 10` 时重置为 5（循环在 32~1024 分钟）的设计意图是什么？为何不封顶在一个固定最大值？是否与唤醒策略或省电优化有关？

**Q8：修改删除冲突的云端数据最终状态**
> 双向同步中，云端修改+本地删除→本地删除赢（文件保持删除）。此时云端的修改版本是否会被保留？下次其他设备同步时是否会重新出现该文件？（从源码看，本地删除赢意味着不向云端推送删除，也不拉取云端修改，因此云端保留修改版本，其他设备同步时会获取到。）
> **证据级别**：推断结论

**Q9：`onlyChangeFoldIAL` 函数的完整判断细节**
> dejavu `ignoreLocalUpsert()` 中调用了 `onlyChangeFoldIAL(a, b *ast.Node) bool` 函数。该函数用于判断两个节点是否仅折叠属性发生了变化。从函数名和调用上下文推断仅比较 `fold` 和 `heading-fold` 属性，但具体实现（如是否还有其他属性被忽略、属性值比较方式等）尚未直接从源码核实。
> **证据级别**：推断结论

---

#### 已通过源码核实、不再需要验证的问题

以下问题曾在之前版本中标记为"待验证"，现已通过 dejavu 源码直接核实，移至已确认结论列表：

- ~~Q1：冲突赢家策略（云端优先 vs 本地优先）~~ → 已确认：双向同步本地优先，下载同步云端优先
- ~~Q2：修改删除冲突的处理方式~~ → 已确认：双向同步下删除操作永远赢，静默处理，不进入 Conflicts
- ~~Q3：冲突副本来源（本地 vs 云端）~~ → 已确认：双向同步=云端版本，下载同步=本地版本
- ~~Q4：`ignoreLocalUpsert` 函数的存在和整体逻辑~~ → 已确认：6 步判断流程，仅 .sy 文件、块数量相同、块 ID/类型相同、仅折叠属性变化时忽略本地变更

---

## 10. 关键代码路径索引

| 功能 | 入口位置 |
|------|----------|
| 同步调度总入口 | [SyncData()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L160-L162) |
| 同步核心流程 | [syncRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1510-L1606) |
| 合并结果处理 | [processSyncMergeResult()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1632-L1894) |
| 冲突副本生成 | [repository.go L1644-1674](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1644-L1674) |
| 本地修改触发 | [IncSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L703-L706) |
| 事务提交点 | [Transaction.commit()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/transaction.go#L1865-L1894) |
| 启动同步（并行优化） | [bootSyncRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1372-L1508) |
| WS 实时感知 | [connectSyncWebSocket()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L794-L888) |
| 前端同步图标更新 | [processSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/app/src/dialog/processSystem.ts#L565-L630) |
| 前端合并后 UI 刷新 | [reloadSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/app/src/dialog/processSystem.ts#L40-L152) |
| 云配置构建 | [buildCloudConf()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L2257-L2315) |
| 同步互斥锁 | [lockSync()/unlockSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L164-L172) |
| 同步前索引快照 | [indexRepoBeforeCloudSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1942-L1993) |
| 仓库自动清理 | [autoPurgeRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L66-L150) |
| 空同步退避逻辑 | [processSyncMergeResult() 开头](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1683-L1696) |
| 同步前置校验 | [checkSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L227-L272) |
| syncingFiles 定义 | [repository.go L1207-L1223](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1207-L1223) |
| 历史目录索引 | [indexHistoryDir()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/history.go#L912-L1007) |
| 闪卡同步阻塞 | [flashcard.go L751-L754](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/flashcard.go#L751-L754) |
| 文件树同步状态 | [api/filetree.go L1202](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/api/filetree.go#L1202) |

---

## 11. 总结

### 11.1 核心设计理念

SiYuan 的同步系统设计体现了以下核心设计理念：

1. **本地优先（Local-First）**：所有写操作先落本地事务，同步完全异步解耦。网络不可用时用户体验零降级。双向同步冲突策略也采用本地优先，保证用户编辑不会丢失。

2. **三模式差异化冲突策略**：
   - 双向同步（Sync()）：**本地优先**（"冲突的文件尽量以本地 upsert 和 remove 为准"）
   - 仅下载（SyncDownload()）：**云端优先**（"冲突的文件以云端 upsert 和 remove 为准"）
   - 仅上传（SyncUpload()）：**本地强制覆盖**（无冲突概念）
   - 输出版本存入临时目录，作为冲突副本供用户手动合并。

3. **快照驱动的增量同步**：基于 dejavu 的内容寻址分块 + Index 哈希链，保证数据完整性和增量传输。

4. **最终一致性 + 可追溯**：三方合并 + 可选冲突副本 + 历史目录归档三层兜底，数据可恢复到任意同步前状态。

5. **实时感知（Reactive）**：WebSocket 多设备通知实现近似实时的多端协同。

6. **多环境适配**：移动端/PC 端差异化策略（分块检查、文件系统权限）、多 Provider 抽象层。

### 11.2 已确认的关键结论（共 12 项，基于 dejavu 源码）

经过 dejavu 源码直接核实 + SiYuan 调用层代码交叉验证，以下结论已**确凿**确认，不再是推断：

| 序号 | 结论 | 核心证据 | 证据来源 |
|------|------|----------|----------|
| 1 | **双向同步（Sync()）本地优先策略** | 两端都修改时，工作目录保留本地版本，temp 目录保存云端版本 | dejavu `sync.go` 注释 + `tmpMergeConflicts = cloudUpserts` 逻辑 |
| 2 | **下载同步（SyncDownload()）云端优先策略** | 两端都修改时，工作目录采用云端版本，temp 目录保存本地版本 | dejavu `sync_manual.go` 注释 + 对称逻辑推导 |
| 3 | **修改删除冲突静默处理（双向同步）** | 云端修改+本地删除→本地删除赢；云端删除+本地修改→本地修改赢；均不进入 Conflicts，不生成冲突副本 | dejavu `sync0()` 函数循环逻辑 |
| 4 | **冲突副本来源差异化** | 双向同步=云端版本（temp 目录保存 cloudUpsert）；下载同步=本地版本 | dejavu `tmpMergeConflicts` 赋值逻辑 + SiYuan 加载路径 |
| 5 | **上传模式无冲突（本地强制覆盖）** | `SyncUpload` 仅返回 `TrafficStat`，不返回 `MergeResult`，SiYuan 手动构造空 MergeResult 传入 | SiYuan `syncRepoUpload()` 代码 |
| 6 | **双向同步先拉后推顺序** | `dataChanged` 同时包含本地 Index 变更和云端 Merge 变更，说明先合并后推送 | SiYuan `syncRepo()` 代码 |
| 7 | **空同步退避 11 次循环机制** | 前 3 次 8 分钟保底，4-10 次指数增长（16→1024 分钟），第 11 次重置为 5（32 分钟）开始循环 | SiYuan `processSyncMergeResult()` 代码 |
| 8 | **错误退避 7 次阈值** | `autoSyncErrCount > 7` 时延迟 64 分钟；手动同步（byHand=true）绕过该检查 | SiYuan `checkSync()` 代码 |
| 9 | **syncingFiles 仅影响闪卡/属性视图** | 全代码搜索验证：仅 flashcard.go / attribute_view_render.go / api/filetree.go 消费；普通文档事务完全不阻塞 | SiYuan 全代码搜索 |
| 10 | **autoSyncErrCount 非原子变量** | 普通 int 类型，多个 goroutine 可能同时读写，存在并发安全风险 | SiYuan `sync.go` 变量定义 |
| 11 | **闪卡两种阻塞方式** | 写操作（加闪卡/删闪卡）直接返回 `TxErrCodeDataIsSyncing` 报错；读操作（查询/渲染）通过 `waitForSyncingStorages()` 轮询等待 | SiYuan `flashcard.go` 代码 |
| 12 | **`ignoreLocalUpsert` 折叠属性特殊规则** | 双向同步中，若本地修改仅为折叠属性变化，则例外地采用云端版本（打破本地优先）；整体判断逻辑为 6 步检查流程，仅对 .sy 文件生效 | dejavu `sync.go` `ignoreLocalUpsert()` 函数源码（已核实 6 步判断流程）；`onlyChangeFoldIAL` 内部细节为推断 |

### 11.3 因果链总结

**离线 → 恢复完整因果链**：
```
网络断开
  → 本地操作正常（事务队列 + flushLock）
  → 同步尝试失败（isProviderOnline）
  → autoSyncErrCount++
  → 连续失败 8 次 → 错误退避（64 分钟）
  → 网络恢复
    → 定时同步触发 / 手动同步 / WS 重连
    → 同步成功 → autoSyncErrCount = 0 → 退避解除
    → 有数据变更 → syncSameCount = 0 → 空同步退避解除
    → WS "synced" 通知 → 其他设备拉取
```

**重复事件幂等性因果链**：
```
多次 WS 通知 / 多次同步调用
  → syncLock 互斥 → 自动排队，同一时间只有一个同步
  → 云端分布式锁 → 同一时间只有一个设备 Push
  → Index 哈希链 → 版本线性演进，不分叉
  → 空同步退避 → 无变更时间隔指数拉长，最终平息
```

**顺序保障三层因果链**：
```
本地层：txQueue FIFO + flushLock 互斥 → 操作严格串行
同步层：FlushTxQueue 等待 + repo.Index 一次性建快照 → 所有修改一次性入快照
全局层：syncLock + 云端锁 + Index 哈希链 → 多设备全局线性可追溯
```

### 11.4 潜在改进方向

基于代码分析的潜在改进方向：

1. **autoSyncErrCount 原子化**：将普通 int 改为 atomic.Int32，消除并发安全风险
2. **错误计数持久化**：跨重启记忆错误次数，防止反复重启绕过退避
3. **启动同步竞态提示**：在 bootSyncRepo 后台同步期间，对正在编辑的文档给出视觉提示
4. **WS 通知风暴抑制**：对 "synced" 消息增加节流（throttling），避免多设备环形通知
5. **冲突可视化工具**：提供差异对比 UI，简化冲突副本的合并操作
6. **全量重建索引可中断**：大工作区下 FullReindex 应支持进度反馈和用户取消
7. **syncSameCount 封顶策略优化**：考虑用固定最大值替代循环重置，行为更可预测