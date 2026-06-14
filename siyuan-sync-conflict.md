# SiYuan 多端实时同步与冲突合并机制深度解析

## 1. 架构概览

### 1.1 核心依赖

SiYuan 的同步机制基于自研的 `dejavu` 库实现，该库提供了完整的数据仓库（Repository）抽象，包括：

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

SiYuan 采用**多点侵入式触发**而非目录级文件监听（文件监听仅作补充），确保所有数据变更都能准确捕获。核心触发函数为 `IncSync()`，在代码库中有 **48+ 处调用**：

[IncSync() 实现](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L703-L706)

```go
func IncSync() {
    syncSameCount.Store(0)           // 重置"无变更连续同步"计数
    planSyncAfter(time.Duration(Conf.Sync.Interval) * time.Second)  // 按配置间隔排程下一次同步
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
| 标题操作 | [heading.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/heading.go) | 转换/排序/折叠 |
| 上传 | [upload.go](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/upload.go) | 资源上传 |
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
                          └─ rollback() 异常回退
```

关键点：
- **异步批处理**：`txQueue` 缓冲 7 个事务，由独立 goroutine 消费
- **全局互斥**：`flushLock` 确保同一时间只有一个事务在执行写文件
- **提交即触发**：`commit()` 末尾调用 `IncSync()`，保证修改立即进入同步排程
- **FlushTxQueue**：同步前显式等待队列排空，见 [indexRepoBeforeCloudSync()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1946)

### 2.3 本地索引与快照

同步前的索引操作是**本地修改最终持久化到仓库**的关键步骤：

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
    // 若快照有变化，更新备注加耗时并持久化
}
```

**快照/索引工作原理**：
1. dejavu 扫描 data 目录，计算每个文件的 SHA-256 分块哈希
2. 与上一个快照对比，生成增量索引（新增/删除/修改的文件列表）
3. 新快照写入本地仓库（`storage/repo/` 目录）
4. 通过 `beforeIndex.ID != afterIndex.ID` 判定本次是否有实际变更

**自动清理策略**：[autoPurgeRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L66-L150)
- 按 `IndexRetentionDays` 保留天内的快照
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
| [SyncDataJob()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L113-L122) | 定时器触发（默认每 5 min 检查） | 检查 syncPlanTime，到期后执行 |
| [SyncData(byHand)](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L160-L162) | API 调用、WS 通知 | 双向同步 Sync() |
| [SyncDataUpload()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L73-L99) | Mode 3 手动上传 | 仅本地上传 |
| [SyncDataDownload()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L46-L71) | WS synced 通知、Mode 3 手动下载 | 仅云端下载 |
| ExitSync | 应用退出前 | exit=true 标记 |

### 3.3 WebSocket 实时感知层

[WebSocket 连接管理](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L745-L908)

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
if 1 == Conf.Sync.Mode && dataChanged {
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
- **S3 Bucket 名校验**：必须符合 CloudDirName 规则（与 Provider=0 目录名共用）

### 3.5 云端锁定机制

dejavu 库内部实现了分布式锁，对应错误类型：
- `dejavu.ErrLockCloudFailed`：获取锁失败（对应用户语 188）
- `dejavu.ErrCloudLocked`：云端已被其他设备锁定（对应用户语 189）
- `dejavu.ErrRepoFatal`：仓库致命损坏，需重置（语 23）

---

## 4. 冲突检测与合并策略

### 4.1 合并结果数据结构

dejavu 的 `Sync()` 返回 `*dejavu.MergeResult`，核心字段在 SiYuan 中的使用如下：

[processSyncMergeResult()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1632-L1894)

```go
type MergeResult struct {
    Time      time.Time     // 合并发生时间
    Conflicts []*entity.File // 无法自动合并的冲突文件
    Upserts   []*entity.File // 已写入本地的新增/更新文件
    Removes   []*entity.File // 已从本地删除的文件
    // SiYuan 扩展字段:
    UpsertPetals []string    // 花瓣(插件)变更
    RemovePetals []string
}
```

### 4.2 冲突检测机制

dejavu 内部使用**三方合并**算法（公共祖先 + 本地版本 + 远程版本），判定规则：

1. **文件哈希对比**：同一路径在两端快照中哈希不同
2. **公共祖先查找**：追溯两个快照索引的最近共同 Index
3. **无冲突场景**：
   - 仅一端有修改 → 直接采用修改版本
   - 两端修改内容完全一致（哈希相同） → 无需操作
4. **冲突场景**：
   - 两端对同一文件做了不同修改 → 进入 Conflicts 列表
   - 一端修改、另一端删除 → 进入 Conflicts 列表（按配置策略处理）

### 4.3 冲突处理策略

SiYuan 提供两种冲突处理模式，由 `GenerateConflictDoc` 开关控制：

**模式 A：保留云版本（默认，GenerateConflictDoc=false）**
- `Conflicts` 中的文件直接采用云端版本覆盖本地
- 用户本地未同步的修改**会丢失**
- 仅写入历史目录供事后恢复：[代码位置](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1679-L1681)
  ```go
  historyDir := HistoryDir + "/{时间戳}-sync"
  indexHistoryDir(...)  // 归档到历史快照目录
  ```

**模式 B：生成冲突副本（GenerateConflictDoc=true）**
- 对每个 `.sy` 冲突文件，在同目录下创建一个新文档副本
- [实现逻辑](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1647-L1674)：
  ```go
  for _, file := range mergeResult.Conflicts {
      if !strings.HasSuffix(file.Path, ".sy") { continue }
      // 从临时目录加载冲突版本（本地/云版本取决于 dejavu 保留策略）
      tree := loadTree(TempDir + "/repo/sync/conflicts/{时间戳}/" + file.Path)
      resetTree(tree, "Conflicted", true)   // 生成新 ID，标题加 (Conflicted) 后缀
      createTreeTx(tree)                    // 作为新文档写入笔记本
      box.addSort(previousPath, tree.ID)    // 挂在原文档之后的排序位置
  }
  ```
- 主文件仍采用云版本
- 用户在 UI 中看到：原始文档 + 标题带 "Conflicted" 的冲突副本，自行对比合并

**非 .sy 文件**（assets、storage、配置等）：
- 不生成冲突副本，采用覆盖策略
- 依赖快照历史进行人工恢复

### 4.4 数据流：同步主循环详解

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
  │        ├─ Checkout 写入本地磁盘
  │        ├─ 本地上传新增分块
  │        ├─ 推送合并后的 Index 到云端
  │        └─ 释放云端锁
  │
  ├─ 6. dataChanged 判定：快照变了 OR 合并结果有数据变化
  │
  ├─ 7. Conf.Sync.Synced = 当前时间戳；Conf.Sync.Stat = 流量统计字符串
  │
  ├─ 8. calcPetalDiff() → 补填 mergeResult.{Upsert,Remove}Petals
  │
  ├─ 9. processSyncMergeResult(exit, byHand, mergeResult, ...)
  │
  └─ 10. 异步：checkIndex() 索引订正 → autoPurgeRepo() 清理快照
```

### 4.5 上传专用与下载专用流程

与 `syncRepo()` 对称的两个单向函数：

| 函数 | dejavu 调用 | 合并结果处理 |
|------|-------------|-------------|
| `syncRepoUpload()` | `repo.SyncUpload()` | 空 MergeResult（无冲突） |
| `syncRepoDownload()` | `repo.SyncDownload()` | 完整 MergeResult，含冲突处理 |

### 4.6 启动优化：并行预取

[bootSyncRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1372-L1508)

启动时采用并行策略缩短首屏等待：
```
WaitGroup
  ├─ goroutine A: indexRepoBeforeCloudSync()  // 本地建索引（CPU密集）
  └─ goroutine B:
        ├─ repo.GetCloudLatest()             // 拉取云端最新索引
        └─ repo.GetSyncCloudFiles()          // 获取需要同步的文件清单
→ Wait()
→ 将 fetchedFiles 填充 syncingFiles Map（供后续事务排重检查）
→ 如果文件数>0：异步 goroutine 执行完整 syncRepo()，主线程立即返回继续 Boot
```

---

## 5. 离线数据恢复、重复事件处理、顺序保障

### 5.1 离线场景下的工作机制

**本地操作不受影响**：
- 事务机制完全本地执行，不依赖网络
- `IncSync()` 正常排程，但 `checkSync()` → `isProviderOnline()` 会失败
- 连续失败超过 7 次，自动同步延迟 **64 分钟**（指数退避）

[isProviderOnline()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L714-L743)

**网络恢复后的自动同步**：
- `SyncDataJob()` 每 5 分钟检查一次 `syncPlanTime`
- 延迟到期后尝试下一次同步
- 若为用户手动触发，不检查失败计数，立即重试

**WebSocket 断线重连**（见 3.3 节）：
- 最多 7 次重试间隔 7s
- 重新连接后恢复实时感知

### 5.2 重复事件与幂等性保障

**同步幂等键**：
- dejavu 仓库使用 `System.ID`（设备唯一 ID）作为标识
- 快照 Index 包含创建者 ID、时间、哈希链
- 云端锁+哈希链保证不会重复写入同一快照

**无变更检测（空同步优化）**：
[processSyncMergeResult() 开头](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1683-L1696)
```go
if 0 == len(Upserts) && 0 == len(Removes) && 0 == len(Conflicts) {
    syncSameCount.Add(1)
    // 指数退避：2^N 分钟，最大 8 分钟
    delay := time.Minute * (2 ** syncSameCount)
    if delay > 8 * time.Minute { delay = 8 * time.Minute }
    planSyncAfter(delay)
    return
}
```

**同步中的文件保护**：
- `syncingFiles sync.Map`：保存正在从云端拉取的 rootID
- `syncingStorages atomic.Bool`：标记 storage 目录是否在同步
- 事务/资产写入时检查 `IsSyncingFile(rootID)`，必要时返回 `TxErrCodeDataIsSyncing`
- 同步完成后统一重置两个变量

### 5.3 顺序保障

**本地操作顺序**：
- 单线程事务执行：`flushQueue()` 串行消费 `txQueue`
- 同步前 `FlushTxQueue()` 保证所有本地修改入盘后再建索引

**同步时序**：
- 全局同步互斥：`syncLock sync.Mutex` + `isSyncing atomic.Bool`
  ```go
  func lockSync()   { syncLock.Lock(); isSyncing.Store(true) }
  func unlockSync() { isSyncing.Store(false); syncLock.Unlock() }
  ```
- 保证同一 Kernel 内同一时刻只进行一次同步

**云端操作顺序**：
- dejavu 的分布式锁保证同一仓库同一时刻只有一个设备在 Push
- 索引哈希链（每个 Index 引用前驱）保证不会分叉
- 合并时以最新云端 Index 为基础进行三方合并

### 5.4 数据丢失兜底：历史目录归档

每次同步（含冲突）都会在 `history/` 下留痕：
```
history/
└── YYYY-MM-DD-HHMMSS-sync/   ← 同步前本地状态归档
    └── ...（完整文件树）
```

用户可通过 **历史版本** 功能回溯到任意同步前的状态。

---

## 6. 用户界面反馈机制

### 6.1 三层反馈架构

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

### 6.2 同步图标状态

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

### 6.3 合并结果后的 UI 刷新

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

### 6.4 冲突消息推送

[processSyncMergeResult() 末尾](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1879-L1892)

```go
if 0 < len(mergeResult.Conflicts) {
    // 检查是否有 .sy 冲突（可感知级别）
    syConflict := any(file.Path endsWith .sy)
    if syConflict {
        util.PushMsg(Language(108), 7000)  // "数据同步发生冲突..."
    }
}
```

### 6.5 详细进度反馈（事件总线订阅）

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

## 7. 潜在风险与需验证的问题

### 7.1 潜在风险

| 风险等级 | 问题描述 | 关联代码 |
|---------|----------|----------|
| **高** | WebSocket 通知风暴：多设备环形触发。设备 A 同步 → 通知 B → B 拉取 → B 数据变更 → 再通知 A → A 再拉取，空同步由指数退避抑制，但仍可能造成循环 | [sync.go L214-223](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L214-L223) |
| **高** | 大文档集合首启同步：`bootSyncRepo` 异步同步可能与用户操作竞态，若用户打开的文档正在 syncingFiles 中，事务会报 `DataIsSyncing` | [repository.go L1495-1503](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1495-L1503) |
| **中** | GenerateConflictDoc=true 时大量冲突可能造成文档爆炸：同一个文档在 N 端反复冲突 → N 个 Conflicted 副本 | [repository.go L1644-1674](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1644-L1674) |
| **中** | `autoSyncErrCount` 是进程内全局变量，重启后清零。极端情况下反复重启可能绕过 7 次失败保护 | [sync.go L103](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L103) |
| **中** | S3/WebDAV 并发请求数默认值未做上限校验，可能触发对象存储限流 | 各 Provider 的 `ConcurrentReqs` 字段 |
| **中** | `syncLock` + 云端锁双重锁定，若内核崩溃云端锁未释放，需等待 TTL（取决于 dejavu 实现） | `ErrCloudLocked` 处理 |
| **低** | `syncingFiles` 使用 `sync.Map`，rootID 哈希冲突概率极低但非零 | [repository.go L1207](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1207) |
| **低** | 移动端不检查分块（`checkChunks=false`），若本地存储损坏可能将坏数据同步到云端 | [repository.go L1948-L1952](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L1948-L1952) |
| **低** | 本地文件系统 Provider 的路径规范化未处理软链接，可能导致循环目录同步 | [SetSyncProviderLocal()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/sync.go#L469-L506) |

### 7.2 需验证的问题

以下问题需要通过实际集成测试或阅读 dejavu 源码进一步确认：

**Q1：冲突文件的赢家策略**
> `MergeResult.Conflicts` 中的文件最终采用本地版本还是云版本写入磁盘？临时目录 `/repo/sync/conflicts/` 下存放的是哪一侧版本？需对照 dejavu `Sync()` 实现确认。

**Q2：分块 CAS 的缓存一致性**
> PC 端检查分块（checkChunks=true），检查出的损坏分块是自动从云端修复还是直接报错终止？

**Q3：一端修改、一端删除的"修改删除冲突"**
> 此类冲突是否进入 Conflicts？处理策略如何？是否有用户可见的提示？

**Q4：跨 Provider 迁移的数据一致性**
> 用户从 SiYuan Provider 切换到 S3 时，`CloudName` 共用，但云端元数据（索引格式）是否完全兼容？需验证 `newRepository` 不同 Cloud 实现的 Index 格式。

**Q5：WS "synced" 消息的节流机制**
> 设备 A 在 1 秒内连续多次同步完成，发送多条 "synced"，设备 B 的 `SyncDataDownload` 是否由 `syncLock` 自动排队？是否会造成 B 的同步队列积压？

**Q6：IncSync 调用的事务隔离**
> `tx.commit()` 先写 `writeTreeUpsertQueue` 后调用 `IncSync()`，两者之间若发生崩溃，文件已写入但同步未排程。下次启动时 `BootSyncData` 的 Index 操作是否能兜底捕获该修改？（理论上可以，因为 Index 会扫描所有文件）

**Q7：0.2 全量重建阈值的合理性**
> `needFullReindex(upsertTrees)` 当同步变更文档数 > 总量 20% 时触发 `FullReindex`。在大工作区（10w+ 文档）下，FullReindex 可能耗时数十分钟，是否有进度反馈和取消能力？

**Q8：Sync.GenerateConflictDoc 与历史目录的双重保存**
> 开启冲突副本后，同一份冲突数据既保存在 `history/YYYY-MM-DD-HHMMSS-sync/` 又作为新 `.sy` 文档写入，是否会造成双倍磁盘占用？是否存在清理策略？

---

## 8. 关键代码路径索引

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
| 仓库自动清理 | [autoPurgeRepo()](file:///d:/fz/0601/solo-dogfeeding/code/285-siyuan/kernel/model/repository.go#L75-L150) |

---

## 9. 总结

SiYuan 的同步系统设计体现了以下核心设计理念：

1. **本地优先（Local-First）**：所有写操作先落本地事务，同步完全异步解耦。网络不可用时用户体验零降级。
2. **快照驱动的增量同步**：基于 dejavu 的内容寻址分块 + Index 哈希链，保证数据完整性和增量传输。
3. **最终一致性 + 可追溯**：三方合并 + 可选冲突副本 + 历史目录归档三层兜底，数据可恢复到任意同步前状态。
4. **实时感知（Reactive）**：WebSocket 多设备通知实现近似实时的多端协同。
5. **多环境适配**：移动端/PC 端差异化策略（分块检查、文件系统权限）、多 Provider 抽象层。

潜在的改进方向集中在：WS 通知风暴抑制、冲突可视化工具（差异对比 UI）、全量重建索引的可中断性、以及错误计数的持久化（跨重启记忆）。
