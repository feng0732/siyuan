# SiYuan 工作空间锁判定逻辑与数据库清理边界深度对照分析

> 版本: SiYuan 内核 Go + 桌面端
> 分析日期: 2026-06-15
> 关联文档: 
> - [siyuan-startup-recovery.md](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/siyuan-startup-recovery.md)
> - [siyuan-startup-recovery-followup.md](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/siyuan-startup-recovery-followup.md)

---

## 一、核心概念辨析：三层锁机制与三类数据库文件

### 1.1 一张表彻底讲清三个经常混淆的"锁"概念

| 概念层 | 存储位置 | 生命周期 | 崩溃后状态 | 是否跨进程 | 技术实现 |
|-------|---------|---------|:---------:|:---------:|---------|
| **🔴 flock 系统锁** | OS 内核（文件描述符表） | 与进程绑定 | ✅ 自动释放 | ✅ 跨进程，内核可见 | `fcntl(F_SETLK)` / `LockFileEx` |
| **🟡 .lock 物理文件** | 文件系统 `<workspace>/.lock` | 持久化，除非手动删 | ❌ 遗留不自动清除 | ✅ 所有进程可见 | 普通 inode，仅作句柄载体 |
| **🟢 WorkspaceLock Go 对象** | Go 进程堆内存 | 随 goroutine 结束回收 | ✅ 随进程销毁 | ❌ 仅本进程可见 | `flock.Flock` 结构体，含 `*os.File` fd |

> **最关键的误区**：「`.lock` 文件存在 ⇄ 工作空间已锁定」这个等式 **99% 的情况下成立，但在崩溃恢复场景完全不成立**。
>
> 真实判定依据是 **flock 系统锁是否被某 fd 持有**，而不是物理文件是否存在。

### 1.2 SQLite WAL 模式下的三类数据库文件

以 `siyuan.db` 为例，打开后会产生**三位一体**的文件簇：

| 文件名后缀 | 正式名称 | 作用 | 关闭时是否 checkpoint | 删除优先级 |
|:---------:|---------|-----|:-------------------:|:---------:|
| `siyuan.db` | **主数据库文件** | B-Tree 页面 + 主元数据 | -（保留用作下次快速启动） | ❌ 正常关闭绝不删 |
| `siyuan.db-wal` | **Write-Ahead Log** | 预写日志，所有写先落此文件 | ✅ `db.Close()` 自动合并入 db | ⚠️ 崩溃后可遗留，是一致性风险 |
| `siyuan.db-shm` | **Shared Memory File** | WAL 索引哈希表（多个 reader 共享） | ✅ 仅内存映射的持久化镜像，无 db 即无意义 | ⚠️ 崩溃后与 -wal 成对遗留 |

4 个数据库组成完整索引体系：

```
<workspace>/temp/
├── siyuan.db              主索引：blocks / refs / assets / fts 全文
├── siyuan.db-wal
├── siyuan.db-shm
├── history.db             历史索引：版本快照
├── history.db-wal
├── history.db-shm
├── asset_content.db       附件内容索引：OCR/文档解析结果
├── asset_content.db-wal
├── asset_content.db-shm
└── blocktree.db           块树缓存：父子关系快速查询
    ├── blocktree.db-wal
    └── blocktree.db-shm
```

> **路径赋值位置**：[working.go L316-L319](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L316-L319)
> ```go
> DBPath             = filepath.Join(TempDir, DBName)         // "siyuan.db"
> HistoryDBPath      = filepath.Join(TempDir, "history.db")
> AssetContentDBPath = filepath.Join(TempDir, "asset_content.db")
> BlockTreeDBPath    = filepath.Join(TempDir, "blocktree.db")
> ```

---

## 二、工作空间锁真实判定逻辑——逐行对照代码

### 2.1 获取锁：`tryLockWorkspace()` 完整流程

**代码位置**：[working.go L504-L516](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L504-L516)

```go
func tryLockWorkspace() {
    // Step 1: 创建 Go 对象（内存态，此时完全不碰文件系统，不拿锁）
    WorkspaceLock = flock.New(filepath.Join(WorkspaceDir, ".lock"))

    // Step 2: 真正执行 3 件事
    //   a) open(O_CREATE|O_RDWR) 若文件不存在则创建物理文件
    //   b) flock(fd, LOCK_EX|LOCK_NB) 非阻塞尝试获取独占锁
    //   c) OS 查内核表：该 inode 是否被其他 fd 持有 EX 锁？
    ok, err := WorkspaceLock.TryLock()

    if ok {
        return  // ✅ 两步都成功：物理文件已存在（或刚创建）+ 系统锁已持有
    }

    // Step 3: 拿不到锁，分两类情况写日志
    if err != nil {
        // 比如文件系统只读、权限不足、路径含中文在某些 OS 上的编码问题
        logging.LogErrorf("lock workspace [%s] failed: %s", WorkspaceDir, err)
    } else {
        // err == nil 但 ok==false：典型的"另一个进程正在持有锁"
        logging.LogErrorf("lock workspace [%s] failed", WorkspaceDir)
    }

    // Step 4: 不做任何恢复，直接退出（ExitCode=24）
    // 关键观察：这里 **不会尝试删除 .lock 物理文件**！
    // 因为无法区分"真有进程在跑"还是"锁遗留"
    os.Exit(logging.ExitCodeWorkspaceLocked)
}
```

**在启动调用链中的位置**：[Boot() 函数内部](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L149-L156)

```go
initWorkspaceDir(*workspacePath)   // ← 路径解析、目录创建、TempDir 初始化、DBPath 赋值...
// ...
tryLockWorkspace()                  // ← 所有路径确定后才加锁（L156）
// ...
```

### 2.2 预检锁：`IsWorkspaceLocked()` 的精妙探测算法

**代码位置**：[working.go L518-L535](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L518-L535)

```go
func IsWorkspaceLocked(workspacePath string) bool {
    // Step 1: 工作空间目录本身不存在 → 必然没锁
    if !gulu.File.IsDir(workspacePath) {
        return false
    }

    lockFilePath := filepath.Join(workspacePath, ".lock")

    // Step 2: **物理文件不存在 → 必然没锁**
    // （注意：这不是充分条件，文件存在≠被锁；但反过来文件不存在=不可能被锁）
    if !gulu.File.IsExist(lockFilePath) {
        return false
    }

    // Step 3: 核心探测 —— 再拿一次同一文件的锁
    f := flock.New(lockFilePath)
    defer f.Unlock()             // 无论如何函数结束时释放自己刚拿的
    ok, _ := f.TryLock()

    if ok {
        // ✅ 能拿到锁 → 说明当前没有其他进程持有锁
        // （即使 .lock 物理文件存在，它只是个空壳，OS 层面无人占用）
        return false
    }

    // ❌ 自己拿不到 → 说明有其他进程真的持有系统锁 → 已锁定
    return true
}
```

**算法正确性分析**：
- 它不依赖"文件是否存在"这种表象，而是用**真实锁申请的结果**做判断
- 本质是 TCP `connect()` 探测端口占用同款思路
- 边界：极端竞态窗口：`f.TryLock()` 成功后 → `defer f.Unlock()` 执行前 → 另一进程刚好处在 `tryLockWorkspace()` 也在试锁 → 可能都判断"可锁"但一方真正拿到 —— 但因为主启动流程最后还是会走 `tryLockWorkspace()` 二次校验，所以安全

### 2.3 释放锁：`UnlockWorkspace()` 的两步协议

**代码位置**：[working.go L537-L551](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L537-L551)

```go
func UnlockWorkspace() {
    // Step 0: 空指针保护（比如 Boot() 还没走到 tryLock 就已触发退出）
    if nil == WorkspaceLock {
        return
    }

    // Step 1: 释放 flock 系统锁
    // 对应 flock(fd, LOCK_UN)
    if err := WorkspaceLock.Unlock(); err != nil {
        logging.LogErrorf("unlock workspace [%s] failed: %s", WorkspaceDir, err)
        return  // ⚠️ 如果这里失败，直接 return，**不执行下面的删文件**
    }

    // Step 2: 删除 .lock 物理文件（给用户视觉上的"已解锁"）
    // 这是可选操作——即使留着 .lock 文件，下次启动也能通过 flock 判断
    if err := os.Remove(filepath.Join(WorkspaceDir, ".lock")); err != nil {
        logging.LogErrorf("remove workspace lock failed: %s", err)
        return
    }
}
```

**关键观察**：两步之间存在一个**理论竞态窗口**：

```
时间轴:
T0: 进程 A 刚 Unlock 系统锁，但还没 Remove 物理文件
T1: 进程 B tryLockWorkspace() → TryLock 成功（系统锁释放了）→ 继续启动
T2: 进程 A 执行 os.Remove(".lock") → 把进程 B 刚打开 fd 的文件删了！
T3: 进程 B 持有的 fd 指向"已被 unlink 的幽灵文件"
T4: 进程 C 启动时 flock.New → 因为 inode 已无目录项，会创建新 .lock → 因为 B 持
    有的是旧 inode 的 fd，两个 fd 不在同一个文件上！
T5: B 和 C 都以为自己拿到了锁 → 💥 实际双写风险
```

> 实际中这个窗口只有几十微秒，而且需要 3 个进程精确交错触发，概率极低，但**从理论上不是无懈可击**。
>
> 可以改进：不删物理文件，只 truncate 为空；或者用 O(1) 的 rename + 原子替换协议。

### 2.4 完整的锁生命周期时序

```
启动阶段:
  进程 A (正常启动)
  ┌──────────────────────────────────────────────────────────┐
  │ initWorkspaceDir()                                        │
  │   └─ TempDir = <workspace>/temp                           │
  │   └─ DBPath = <workspace>/temp/siyuan.db                 │
  │   └─ 删除 <temp>/os, <temp>/repo                          │
  │                                                           │
  │ tryLockWorkspace()                      [Boot() L156]    │
  │   ├─ flock.New(".lock")           → 创建 Go 对象        │
  │   └─ TryLock()                      → 创建物理文件 + 占系统锁│
  │                                                           │
  │ ... (启动主流程: InitConf, Serve, InitDB, InitBoxes) ... │
  │                                                           │
  │ Close()                               [conf.go L738]     │
  │   ├─ FlushTxQueue()                                        │
  │   ├─ sql.CloseDatabase()            → 仅 close() 连接    │
  │   ├─ clearWorkspaceTemp()          → 删 7 类临时目录     │
  │   └─ UnlockWorkspace()                                    │
  │        ├─ WorkspaceLock.Unlock()   → 释放系统锁          │
  │        └─ os.Remove(.lock)        → 删除物理文件         │
  │                                                           │
  │ os.Exit(0)                                                │
  └──────────────────────────────────────────────────────────┘

崩溃场景 (未调用 Close()):
  进程 B (崩溃 / kill -9 / 断电 / 任务管理器"结束任务")
  ┌──────────────────────────────────────────────────────────┐
  │ tryLockWorkspace() → 创建物理文件 + 占系统锁              │
  │ ... (正常运行一段时间)                                    │
  │ ... 突然崩溃！Close() 永不执行 ...                        │
  │                                                           │
  │ 结果:                                                     │
  │   ✅ flock 系统锁 → OS 自动回收（所有 fd 随进程销毁）     │
  │   ❌ .lock 物理文件 → 遗留在文件系统！                    │
  │   ❌ .db-wal / .db-shm → 遗留在 temp/ ！                  │
  │   ❌ <temp>/os, <temp>/repo → 遗留在 temp/ ！             │
  └──────────────────────────────────────────────────────────┘
```

---

## 三、数据库索引文件清理边界——逐代码对照

### 3.1 `RemoveDatabaseFile()`：唯一的 3 文件清道夫

**代码位置**：[working.go L563-L587](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L563-L587)

```go
func RemoveDatabaseFile(dbPath string) {
    // 📦 文件组 1/3: 主数据库文件
    if gulu.File.IsExist(dbPath) {
        err := os.RemoveAll(dbPath)
        if err != nil {
            logging.LogErrorf("remove database file [%s] failed: %s", dbPath, err)
            return   // ⚠️ 任何一个删失败 → 整个函数立刻返回！
        }              //   这意味着如果 .db 被占用删不掉，
                       //   -shm 和 -wal 根本不会被尝试删除！
    }

    // 📦 文件组 2/3: WAL 共享内存索引文件
    if gulu.File.IsExist(dbPath + "-shm") {
        err := os.RemoveAll(dbPath + "-shm")
        if err != nil {
            logging.LogErrorf("remove database file [%s] failed: %s", dbPath+"-shm", err)
            return   // 同上，失败短路
        }
    }

    // 📦 文件组 3/3: WAL 预写日志文件
    if gulu.File.IsExist(dbPath + "-wal") {
        err := os.RemoveAll(dbPath + "-wal")
        if err != nil {
            logging.LogErrorf("remove database file [%s] failed: %s", dbPath+"-wal", err)
            return
        }
    }
}
```

**设计点评**：
- ✅ 正确覆盖了 SQLite 3 文件簇
- ⚠️ **严重问题 1**：串行短路逻辑 —— 主文件被杀毒软件暂锁 → `-wal`/`-shm` 完全不删 → 下次新建 `siyuan.db` 却读取到旧 `-wal`，可能造成严重错乱
- ⚠️ **严重问题 2**：`os.RemoveAll` 是递归删除，如果 dbPath 指向目录（罕见 bug 场景），会误删整棵子树，更安全的选择是 `os.Remove` 只删单文件

### 3.2 `RemoveDatabaseFile()` 的唯一调用点

**代码位置**：[database.go L1504-L1514](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1504-L1514)，`prepareExecInsertTx()` 函数内

```go
if _, err = stmt.Exec(args...); err != nil {
    tx.Rollback()
    logging.LogErrorf("exec database stmt [%s] failed: %s\n  %s", stmtSQL, err, logging.ShortStack())

    if strings.Contains(err.Error(), "database disk image is malformed") {
        util.RemoveDatabaseFile(util.DBPath)   // ← 唯一调用点，仅清理 siyuan.db
        initDatabase(true)                      // 强制重建标记
        logging.LogFatalf(logging.ExitCodeUnavailableDatabase,
            "database disk image [%s] is malformed, please restart SiYuan kernel to rebuild it\n\t%s\n\t%v",
            util.DBPath, stmtSQL, args)
    }
    return
}
```

### 3.3 全量扫描：4 个 DB 的 malformed 处理对照表

对代码库中所有 `"database disk image is malformed"` 检测点逐一检查：

| 数据库 | 检测位置 | malformed 时处理 | 是否调用 `RemoveDatabaseFile()` |
|:-----:|---------|----------------|:------------------------------:|
| **siyuan.db** | [database.go L1508](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1508) `prepareExecInsertTx()` | 删文件 + initDatabase(true) + Exit(20) | ✅ **调用了** |
| **siyuan.db** | [database.go L1523](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1523) `execStmtTx()` | initDatabase(true) + Exit(20) | ❌ **没调！** |
| **blocktree.db** | [blocktree.go L625](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/treenode/blocktree.go#L625) `execInsertBlocktrees()` Prepare 阶段 | initDatabase(true) + Exit(20) | ❌ **没调！** |
| **blocktree.db** | [blocktree.go L642](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/treenode/blocktree.go#L642) `execInsertBlocktrees()` Exec 阶段 | initDatabase(true) + Exit(20) | ❌ **没调！** |
| **history.db** | 全局 Grep 结果 | **完全没有 malformed 检测** | ❌ 根本没检测 |
| **asset_content.db** | 全局 Grep 结果 | **完全没有 malformed 检测** | ❌ 根本没检测 |

**风险全景**：
```
 malformed 覆盖率
  siyuan.db (prepareExecInsertTx) ████████░░  50% (2 处调用 1 处没删)
  siyuan.db (execStmtTx)         ████████░░
  blocktree.db (2 处)            ████░░░░░░  0% (都只 initDatabase(true))
  history.db                     ░░░░░░░░░░  0% (无检测)
  asset_content.db               ░░░░░░░░░░  0% (无检测)
```

### 3.4 其他非 malformed 清理入口

#### (a) 启动时临时目录清理（不碰 .db 文件！）

**代码位置**：[initWorkspaceDir() L306-L312](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L306-L312)

```go
osTmpDir := filepath.Join(TempDir, "os")
os.RemoveAll(osTmpDir)            // ✅ 清：temp/os 系统临时
// ...
os.RemoveAll(filepath.Join(TempDir, "repo"))  // ✅ 清：temp/repo 克隆缓存
// ❌ 不清理：temp/*.db, temp/*.db-wal, temp/*.db-shm
```

**设计原因**：正常关闭场景下 .db 文件完好，**保留可省去下次启动的索引重建时间**（分钟级节省）。

**问题**：崩溃场景下的 -wal / -shm 遗留文件不会在此处清理，下次 SQLite 打开时会尝试 replay WAL，如果 replay 时的 db 页 checksum 不匹配 → malformed → 此时才触发上面的删除逻辑。

#### (b) 正常退出时 clearWorkspaceTemp()

**代码位置**：[conf.go L1136-L1160](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L1136-L1160)

```go
func clearWorkspaceTemp() {
    os.RemoveAll(filepath.Join(util.TempDir, "bazaar"))    // 集市资源缓存
    os.RemoveAll(filepath.Join(util.TempDir, "export"))    // 导出临时
    os.RemoveAll(filepath.Join(util.TempDir, "import"))    // 导入临时
    os.RemoveAll(filepath.Join(util.TempDir, "convert"))   // 格式转换临时
    os.RemoveAll(filepath.Join(util.TempDir, "repo"))      // 仓库临时（重复清，启动时也清）
    os.RemoveAll(filepath.Join(util.TempDir, "os"))        // 系统临时（同上）
    os.RemoveAll(filepath.Join(util.TempDir, "base64"))    // 图片 base64 临时
    // ... 超 7 天的安装包过期清理逻辑 ...

    // ❌ 同样：不碰任何 *.db / *.db-wal / *.db-shm 文件！
}
```

#### (c) 结构版本变更的"静默重建"

**代码位置**：[initDatabase() L92-L106](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L92-L106)

```go
if !forceRebuild {
    // DatabaseVer 是代码中硬编码的常量
    // getDatabaseVer() 读 stat 表中上次保存的版本
    if util.DatabaseVer == getDatabaseVer() {
        return  // ✅ 版本一致：什么都不做，直接复用现有 DB
    }
    logging.LogInfof("the database structure is changed, rebuilding database...")
}
// 版本不一致 or forceRebuild → 直接 initDBTables() 原地 DROP + CREATE
initDBTables()    // DROP TABLE IF EXISTS blocks → CREATE TABLE ...
vacuum()          // VACUUM 回收空间
```

**关键点**：这种场景**完全不调用 RemoveDatabaseFile()**，因为文件本身完好只是 schema 旧了，直接在原文件内 DROP/CREATE 更高效。

#### (d) `beginTx()` 中的 "database is locked" 处理

**代码位置**：[database.go L1371-L1379](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1371-L1379)

```go
func beginTx() (tx *sql.Tx, err error) {
    if tx, err = db.Begin(); err != nil {
        logging.LogErrorf("begin tx failed: %s\n  %s", err, logging.ShortStack())
        if strings.Contains(err.Error(), "database is locked") {
            os.Exit(logging.ExitCodeUnavailableDatabase)   // ← Exit(20)，**完全不清理**
        }
    }
    return
}
```

> `historyDB` / `assetContentDB` 的 beginTx 也是同样逻辑（[database.go L1395-L1402](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1395-L1402)、[L1419-L1426](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1419-L1426)）

### 3.5 完整的清理矩阵

**横轴**：6 类触发场景 / **纵轴**：4 类清理动作

| 触发场景 | 系统锁释放 | 删 .lock 物理文件 | 删 .db / -wal / -shm | 清 temp 子目录 |
|---------|:---------:|:---------------:|:------------------:|:-------------:|
| 正常关闭 `Close()` | ✅ Unlock() | ✅ os.Remove() | ❌ 保留（下次快速启动） | ✅ 7 类子目录 |
| 崩溃 / Kill-9 | ✅ OS 自动 | ❌ 遗留 | ❌ 遗留 | ❌ 遗留 |
| malformed (prepareExecInsertTx) | N/A | N/A | ✅ RemoveDatabaseFile() | N/A |
| malformed (execStmtTx / blocktree) | N/A | N/A | ❌ 仅 initDatabase(true) | N/A |
| schema 版本升级 | N/A | N/A | ❌ 原地 DROP/CREATE | N/A |
| "database is locked" | N/A | N/A | ❌ 直接 Exit(20) | N/A |

---

## 四、概念辨析表与常见误判场景

### 4.1 6 大常见误判的真相对照

| # | 常见误解 | 代码证实的真相 |
|:-:|---------|-------------|
| 1 | "**如果工作空间目录下有 .lock 文件，就说明实例正在运行**" | ❌ 错。崩溃后 .lock 必然遗留，但 flock 锁已释放。用 `IsWorkspaceLocked()` 的逻辑才是正解：**再拿一次试试**，能拿到就是"假锁遗留" |
| 2 | "**删掉 .lock 文件就等于解锁了工作空间**" | ⚠️ 一半对。如果真的有其他进程在跑，你删了它的 .lock 文件——因为它的 fd 还在，系统锁仍有效，**其他进程还是锁不住**。但后果是它的 UnlockWorkspace() 最后删文件时会报错，而且下次探测因文件不存在会误判"未锁定"——最终引发双写。**永远不要手动删正在运行实例的 .lock** |
| 3 | "**malformed 错误时代码会自动清理所有数据库文件**" | ❌ 错。4 个数据库中只有 siyuan.db 的 1 个执行路径（prepareExecInsertTx）会真的调 RemoveDatabaseFile()，execStmtTx 路径、blocktree.db、history.db、asset_content.db **全不清理** |
| 4 | "**temp/*.db 每次启动都会被删掉重建，因为是索引嘛**" | ❌ 完全错。启动时只删 temp/os 和 temp/repo，**绝不碰 .db 文件**。正常关闭后 .db 完好，下次 0ms 打开省掉全量重建。删掉反而是灾难——重新索引几万文档要数十分钟 |
| 5 | "**flock 是跨平台的强一致性锁，什么情况下都能防双写**" | ❌ 错。在 NFSv3、SMB 老版本、某些 FAT32 外置盘上 flock 要么退化成空实现要么返回假成功。Docker 宿主机不同 namespace 挂载同一 HostPath 也能绕过。**flock 是建议性锁不是强制性锁**，绕过保护的方法有很多 |
| 6 | "**正常 Close() 后数据库没有 -wal 文件了，就说明 checkpoint 完成，下次绝对没问题**" | ⚠️ 99% 情况下对。但极端边缘（最后一次 commit 刚写完 WAL 还没 checkpoint，就在 db.Close() 前断电）→ 下次打开 SQLite 会自动 replay WAL，如果 WAL 头部页损坏 → 依旧报 malformed。**不是 SiYuan 的 bug，是任何 WAL 数据库的固有特性** |

### 4.2 崩溃后首次重新启动的逐步判定流程

```
用户视角: 上次可能崩了，这次点启动
            │
            ▼
     initWorkspaceDir()
     │  └─ TempDir/ 不完整？可能有半写的 WAL...
     │
     tryLockWorkspace()
     │  ├─ 创建/打开 .lock 文件
     │  └─ flock TryLock() ⇐ 这是真正的判定点！
     │
     │    TryLock 成功  ←───────────────┐
     │       │                            │
     │       ▼                            │ TryLock 失败
     │  进入正常启动                     │
     │  (物理文件 .lock 即使遗留         │
     │   也不影响结果)                   │
     │                                    │
     │                                    ▼
     │                           Exit(24) → Electron 弹"工作空间锁定"
     │
     InitDatabase(false)
     │  ├─ sql.Open 打开 siyuan.db
     │  ├─ SQLite 自动检测到 -wal 文件 → 自动 replay
     │  │  ├─ replay 成功 → ✅ 正常，比对 DatabaseVer
     │  │  │    └─ 版本一致 → return（不重建，快）
     │  │  │    └─ 版本不一致 → initDBTables 原地重建表
     │  │  └─ replay 失败 → 打开阶段返回 "malformed" / "not a db"
     │  │       └─ LogFatalf(ExitCodeUnavailableDatabase)
     │  │           → Electron 弹"数据库不可用"
     │  │           → 用户重启后上面流程重来，
     │  │             但此时 .db 文件因崩溃可能仍在
     │  │             → 除非 prepareExecInsertTx 中被 RemoveDatabaseFile
     │  │
     │  └─ initDBTables 表创建失败（比如文件系统只读）
     │      └─ Exit(20)
     │
     ... 后续流程 ...
```

---

## 五、已识别的边界缺陷与改进建议

### 5.1 锁相关的 3 个改进点

| # | 缺陷 | 现有代码行为 | 改进方案 |
|:-:|-----|------------|---------|
| L-1 | **TryLock 失败不分"真占用"与"锁遗留"** | 一概 Exit(24)，让用户手动二选一 | Exit 前写入 `<workspace>/.lock_owner` 含 PID、启动时间。Electron 捕获 Exit 24 时 `process.kill(pid, 0)` 测存活后给不同文案 |
| L-2 | **Unlock→Remove TOCTOU 竞态** | A 进程 Unlock 后、Remove 前 B 进程拿到锁 → A 把 B 用到的文件删了 | 不 `os.Remove()`，改为 `os.Truncate(fd, 0)` 清空内容为 0 字节。下次 `IsWorkspaceLocked` 仍能通过 TryLock 判定。或者用 `rename(".lock", ".lock.released")` |
| L-3 | **网络盘 / 外部盘 flock 可靠性无法自检** | 直接信任 TryLock 返回值，在 NFS/SMB/FAT32 上可能 100% 返回 ok | 启动后立即做一轮 **双进程自检测试**：Go 程 A 拿锁 → 子进程 B 起同一个 TryLock → B 必须失败。如果 B 也成功，说明底层文件系统锁不可靠，需弹出"⚠️ 当前工作空间位于不支持文件锁的磁盘上，可能导致数据损坏"的严重警告 |

### 5.2 数据库清理的 5 个改进点

| # | 缺陷 | 严重性 | 改进方案 |
|:-:|-----|:-----:|---------|
| D-1 | **`RemoveDatabaseFile()` 短路式失败** — 主文件被锁 → -wal/-shm 永不删 | 🟠 中高 | 改为**并行三段清理**，每个文件独立 try-catch（Go 中为独立 if+log），3 个全部尝试完再汇总返回。不用 return 短路 |
| D-2 | **3 处 malformed 检测没调 RemoveDatabaseFile()** | 🟠 中高 | `execStmtTx`、blocktree 的 2 处，全部补上 `RemoveDatabaseFile(util.BlockTreeDBPath)` 等对应调用 |
| D-3 | **history / asset_content DB 完全无 malformed 检测** | 🔴 高 | 给 `beginHistoryTx`、`beginAssetContentTx`、`history exec` 路径全部补上 `strings.Contains(err, "malformed")` 检测 + `RemoveDatabaseFile(对应Path)` + Exit(20) |
| D-4 | **"database is locked"直接 Exit(20) 不清理任何东西** | 🟡 中 | `busy_timeout` 已 7000ms 还锁冲突，说明死锁/多连接问题。Exit 前至少加 `db.Close()` 尝试 checkpoint，避免 -wal 半写状态遗留 |
| D-5 | **崩溃重启后打开旧 .db + 旧 -wal 组合，仅 malformed 后才删**，用户需先等一次失败再重启 | 🟡 中 | 启动阶段 `initDBConnection()` 后立刻加一个 `PRAGMA integrity_check`（或者更快速的 `PRAGMA quick_check`），如果返回 not ok 再在**进入业务逻辑前**主动调 RemoveDatabaseFile，避免用户多跑一次失败循环 |

### 5.3 改进后的理想 malformed 处理闭环

```
当前 SiYuan (v现状):
  ┌──────────────────────────────────────────────┐
  │ 运行中 SQL 报错 malformed                     │
  │   → (少数路径) RemoveDatabaseFile             │
  │   → Exit(20)                                  │
  │   → Electron 弹"数据库不可用，请查看日志"      │
  │                                              │
  │ 用户手动重启                                  │
  │   → 检测是否有 .db → 有 → open → 打开坏文件   │
  │     → 启动可能成功（schema 还能读）            │
  │     → 再次运行到某 malformed SQL → 重复循环    │
  │   → 如果 .db 已被 RemoveDatabaseFile 删除      │
  │     → 空库 → initDBTables → 版本不一致重建     │
  │     → InitBoxes → indexBox → 全量重建索引     │
  └──────────────────────────────────────────────┘

改进后 SiYuan (v理想):
  ┌──────────────────────────────────────────────┐
  │ 任何路径 malformed 检测触发                    │
  │   → 所有 4 个 DB 全部 RemoveDatabaseFile      │
  │   → 写 temp/rebuild_needed.json 标记           │
  │   → Exit(20)                                  │
  │                                              │
  │ 用户重启（或者 Electron 可选自动重启 Exit 20） │
  │   → initWorkspaceDir → 检测到 rebuild_needed   │
  │   → 版本轮询时额外加 "⚠️ 正在重建索引..."前缀   │
  │   → initDBTables → 空库重建                    │
  │   → indexBox → 进度条告诉用户"检测到上次索引"  │
  │   → 重建完成 → 弹通知告知"上次损坏已修复"       │
  │   → 删 rebuild_needed.json                     │
  └──────────────────────────────────────────────┘
```

---

## 六、结论：锁与清理设计的取舍哲学

SiYuan 在锁与数据库清理上采用了**非常保守、非常克制**的策略：

| 决策点 | 实际选择 | 好处 | 代价 |
|-------|---------|-----|-----|
| **锁失败策略** | 直接 Exit，不做任何尝试 | 绝不在"不确定是否安全"的情况下继续写数据 | 用户体验差（需手动排查：真占用 vs 锁遗留） |
| **正常关闭 DB 清理** | 完全不清理，保留完好 .db | 下次启动秒开，不用分钟级重建 | 用户可能误以为 temp/ 都是垃圾，手动清空后造成大卡顿 |
| **malformed 清理** | 只在 1 处路径真删文件，其他仅标记 | 最大程度保留可恢复机会（也许下次 replay 就好了） | 不一致的覆盖度让某些角落 bug 经历多次失败才自愈 |
| **崩溃后启动** | 不主动 integrity_check，直接尝试打开 | 99% 场景下最快恢复，不做额外耗时检查 | 极端边缘 case 下用户先经历一次失败重启才触发删文件 |

**一句话总结**：
> **SiYuan 愿意让用户多跑一次失败流程，也绝不冒险在不确定安全性的前提下删除任何可能还有恢复价值的文件。**

这是一种数据安全 > 用户体验 > 代码一致性优先级的选择。所有上述改进建议都遵循**不降低当前数据安全**的原则，只在信息获取、覆盖度一致性、错误说明文案三个维度上提升用户体验和代码对称性。
