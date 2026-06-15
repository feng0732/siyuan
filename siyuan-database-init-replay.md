# SiYuan 数据库初始化生命周期与文件残留校准分析

> 版本: SiYuan 内核 Go + SQLite 3 + mattn/go-sqlite3 扩展驱动
> 分析日期: 2026-06-15
> 关联文档:
> - [siyuan-database-cleanup-failure.md](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/siyuan-database-cleanup-failure.md)
> - [siyuan-lock-database-boundary.md](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/siyuan-lock-database-boundary.md)

---

## 一、全局启动调用总览

### 1.1 四个数据库在 main() 中的初始化顺序

**代码位置**：[main.go L36-L40](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/main.go#L36-L40)

```go
sql.InitDatabase(false)                // 1. 主索引 siyuan.db
sql.InitHistoryDatabase(false)      // 2. 历史 history.db
sql.InitAssetContentDatabase(false) // 3. 附件内容 asset_content.db
sql.SetCaseSensitive(...)           // (配置项，不动 DB)
sql.SetIndexAssetPath(...)         // (同上
```

然后是 `model.InitBoxes()` 才会真正往数据库里插数据。

**blocktree DB 的初始化嵌入在第一个里面：`sql.initDatabase()` → `treenode.InitBlockTree(forceRebuild)` → `treenode.initDatabase()`。

### 1.2 四种初始化类型速查表

| 数据库 | Go 变量 | 路径常量 | forceRebuild=false 判定方式 | 判定代码 |
|--------|--------|---------|:-----------------------:|--------|
| **主索引** (A型) | `db` | `util.DBPath` = temp/siyuan.db | stat 表 version 字符串比较 | [database.go L94](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L94) `util.DatabaseVer == getDatabaseVer() |
| **历史** (B型) | `historyDB` | `util.HistoryDBPath` = temp/history.db | 主文件存在性 | [database.go L260](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L260) `gulu.File.IsExist(path)` |
| **附件** (B型) | `assetContentDB` | `util.AssetContentDBPath` = temp/asset_content.db | 主文件存在性 | [database.go L320](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L320) `gulu.File.IsExist(path)` |
| **块树** (C型) | `treenode/db` | `util.BlockTreeDBPath` = temp/blocktree.db | 主文件存在性 | [blocktree.go L60](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/treenode/blocktree.go#L60) `!gulu.File.IsExist(path)` → 设 forceRebuild=true |

> ⚠️ 重要：B 型和 C 型都**只检查主文件是否存在，完全不管 -wal/-shm。
> 只要 .db 在，哪怕 -wal 损坏到天荒地老也认定为"数据库完好"直接用"。

---

## 二、sql.Open 与物理文件创建时机校准

### 2.1 Go `database/sql` 的 Lazy Init 特性

Go 标准库 `sql.Open(driverName, dsn)` 的设计是**典型懒加载**：
- ✅ 校验驱动注册
- ✅ 解析 DSN 参数
- ✅ 创建连接池管理器
- ❌ **不会立刻打开数据库文件**
- ❌ **不会创建物理文件**

真正的数据库文件创建/打开发生在**第一次真实数据库操作**时：
- 第一个 `db.Ping()`
- 第一个 `db.Exec()` / `db.Query()` / `db.QueryRow()`
- 第一个 `db.Begin()`
- 第一个 `tx.Prepare()`

**但是**：mattn/go-sqlite3 驱动因为 `_journal_mode=WAL` 等参数是通过 PRAGMA 设置的——这些 PRAGMA 什么时候执行？

答案是：**在第一个连接建立时执行**。也就是第一个 `db.Ping()` 或第一次查询会触发第一个连接创建，驱动在连接回调里跑一系列 PRAGMA，这是 SQLite 就会创建/打开文件并设置 WAL 模式。

### 2.2 SiYuan 中每个 DB 的"首次真实读写"位置校准

逐库校准每个 DB 初始化函数中，第一步就是 `initDBConnection()` → `sql.Open()`，它本身不打文件。

那么**第一笔真实 I/O 发生在哪里**？

#### A 型（siyuan.db）
路径：`initDatabase(forceRebuild=false)

```
initDBConnection()
  → sql.Open()  (lazy, 无文件 I/O
  → db.SetMaxIdleConns(20) ... (纯内存操作)
  → 还没有文件创建？

treenode.InitBlockTree(forceRebuild)
  → treenode.initDBConnection()
     → 同 lazy sql.Open()
     → 还没打文件

getDatabaseVer()   ← ⚡ 第一个真实读！
  → db.QueryRow("SELECT value FROM stat WHERE `key` = ?")
  → 触发第一个连接创建 + PRAGMA 设置 + 文件打开
  → 如果文件不存在？SQLite 自动创建空 .db + .db-wal + .db-shm
  → 如果 stat 表不存在？返回空字符串 ""
  → 走 initDBTables() 重建
```

**首次真实读代码位置**：[stat.go L35](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/stat.go#L35) `db.QueryRow(...)`

> 注意：这里有个 **trick —— `getDatabaseVer()` 如果 `no such table: stat` error 是**不算错误日志（不包含 `LogFatalf），
> 因为新库本来就没有 stat 表，走重建。但 **创建空文件后，再在 initDBTables 里建表。

#### B 型（history / asset_content）
路径：`InitHistoryDatabase(forceRebuild=false)`

```
initHistoryDBConnection()
  → sql.Open()  (lazy)
  → 还没打文件

gulu.File.IsExist(util.HistoryDBPath)   ← ⚡ 第一个"检查的是磁盘文件路径字符串，**不是 DB 打开！
  → 返回 true（有 .db 就直接 return
  → 返回 false → 走 forceRebuild 分支

只有 forceRebuild 才会真打文件
```

**首次真实读写**：只有在 `!forceRebuild && IsExist=true` 路径下 → **直接 return，数据库从未打文件！**
连接池里还没开过任何一个连接都还没建立！

> 这个很容易被忽略：B 型在"数据库已存在"场景下，`InitHistoryDatabase` 函数执行完毕后，**SQLite 文件系统上 .db-wal 和 .db-shm 可能根本不存在也可能还不存在也可能还没被打开过**。
> 下一次真实读写发生在 `beginHistoryTx() → historyDB.Begin() → 建立第一个连接。

#### C 型（blocktree）
路径：`InitBlockTree(forceRebuild=force)` → `initDatabase(force)`

```
initDBConnection()  → lazy
gulu.File.IsExist(path)  → 只判断文件存在就 return
```

与 B 型完全一样，forceRebuild=false 且文件存在路径 return，**第一个连接还没建！

### 2.3 "forceRebuild=true 时的首个真实 I/O

如果 forceRebuild=true （或者 !IsExist 导致设为 true）：

**A 型**：直接跳过 version 检查 → `initDBTables()` → 第一个 `db.Exec("DROP TABLE IF EXISTS stat")` → 首次真实写

**B 型**：
```
historyDB.Close()   (如果已有连接关闭)
historyDB = nil + GC
os.RemoveAll(HistoryDBPath)  ← ⚠️ 只删主文件
initHistoryDBConnection()  → 新的 sql.Open，lazy
initHistoryDBTables()
  → historyDB.Exec("CREATE VIRTUAL TABLE ...")
  → 首次真实写 → 创建新空库
```

**C 型**：
```
initDBConnection() → lazy
forceRebuild = true → 跳过 IsExist 判断
initDBTables() → db.Exec("DROP TABLE IF EXISTS blocktrees") → 首次真实写
```

### 2.4 关键校准结论表

| 场景 | sql.Open 后文件是否创建 | 首次真实 I/O 发生在 | .db | .db-wal | .db-shm |
|-----|:-------------------:|:---------------:|:---:|:-------:|:-------:|
| forceRebuild=false + 文件不存在 | ❌ | getDatabaseVer 查询 (A型) / 首次业务查询 (B/C型) | ✅ 打开时自动创建 | ✅ 打开时自动创建 | ✅ 打开时自动创建 |
| forceRebuild=false + 文件存在 & 完好 | ❌ | getDatabaseVer 查询 (A) / return 后首次业务查询 (B/C) | 打开前已存在 | 可能存在 | 可能存在 |
| forceRebuild=true + A 型 | ❌ | initDBTables DROP TABLE → 首次写 | 已存在（覆盖 | 建表时写 WAL | 建表时建 shm |
| forceRebuild=true + B 型 history | ❌（先删 .db → 再 sql.Open(lazy) → 首次 CREATE 建表 | initHistoryDBTables() CREATE → 首次写 | 删旧建 | 新建空再新建空再建时创建 | 新建 |
| forceRebuild=true + C 型 blocktree | ❌ | initDBTables DROP/CREATE → 首次写 | 已存在（原地重建） | 建表时创建 | 建表时创建 |

---

## 三、initDatabase() 详细时序与阶段划分

### 3.1 A 型 (siyuan.db) forceRebuild=false 完整时序

以 `initDatabase(false)` 为例，从调用栈

```
时间轴 →
  │
  ├── ① ClearCache() + disableCache()   [缓存相关，内存操作
  │
  ├── ② util.IncBootProgress(2, "Initializing database...")
  │     (进度条推进，纯内存+HTTP 返回
  │
  ├── ③ initDBConnection()   [database.go L228]
  │     ├── closeDatabase() → db 是 nil，什么都不做
  │     ├── util.LogDatabaseSize(util.DBPath) → os.Stat 读文件大小
  │     │   ⚠️ 注意：这里 **读了一次文件元信息
  │     │   如果 .db 不存在，Stat返回 error，logging.LogWarnf
  │     ├── 拼接 DSN 字符串
  │     ├── sql.Open("sqlite3_extended", dsn)
  │     │   └─ Go database/sql 懒加载
  │     │   └─ **还没打开文件
  │     ├── db.SetMaxIdleConns(20)
  │     ├── db.SetMaxOpenConns(20)
  │     └── db.SetConnMaxLifetime(365*24h)
  │        （全部纯内存，文件还没动过
  │
  ├── ④ treenode.InitBlockTree(forceRebuild=false)   [database.go L90]
  │     └── treenode.initDatabase(false) [blocktree.go L53]
  │         ├── initDBConnection() → lazy sql.Open() → 同 ③
  │         ├── !forceRebuild → gulu.File.IsExist(BlockTreeDBPath)
  │         │   ├─ 存在 → return（blocktree 也没打开！）
  │         │   └─ 不存在 → forceRebuild=true → 走 initDBTables() 建表
  │         └── ...
  │
  ├── ⑤ if !forceRebuild → getDatabaseVer()   [database.go L94]
  │     └── db.QueryRow("SELECT value FROM stat WHERE `key` = ?")  [stat.go L35]
  │         └── ⚡ **第一个真实数据库操作！**
  │             ├── 触发第一个连接建立
  │             ├── 驱动执行 PRAGMA 设置 (journal_mode=WAL 等)
  │             ├── SQLite 打开/创建 .db 文件
  │             ├── 创建/打开 .db-wal 和 .db-shm
  │             └── 执行 SELECT
  │
  ├── ⑥ 版本比较
  │     ├── 相等 → return ✅ 结束（最快路径）
  │     └── 不等 → 继续
  │
  ├── ⑦ initDBTables()   [database.go L108-L226]
  │     ├── DROP TABLE IF EXISTS stat
  │     ├── CREATE TABLE stat ...
  │     ├── setDatabaseVer() → 写入 "20220501"
  │     ├── DROP/CREATE blocks, blocks_fts, ... 共 10+ 张表和索引
  │     └── 所有写入走 WAL
  │
  └── ⑧ vacuum()  [database.go L103]
        ├── db.Exec("VACUUM")
        └── 压缩、重建整个数据库（重写所有页，碎片整理
```

**如果 .db 不存在时，第 ⑤ 步 db.QueryRow 创建空库，第 ⑦ 步建表填数据。

### 3.2 forceRebuild=true 的时序差异

```
initDatabase(true):
  │
  ├── ClearQueue()  (清理 SQL 语句队列
  │
  ├── initDBConnection() → lazy
  │
  ├── treenode.InitBlockTree(true) → 强制重建
  │
  ├── 跳过 version 比较 (因为 forceRebuild=true
  │
  ├── initDBTables()
  │   ├── DROP TABLE ... (每个表都 DROP IF EXISTS
  │   ├── CREATE TABLE ...
  │   └── setDatabaseVer()
  │
  └── vacuum()
```

> ⚠️ **重要校准**：forceRebuild=true 的 A 型路径 **不会先删文件！只是原地 DROP + CREATE。
> 也就是说，如果 .db 文件还在，-wal 还在，只是表被重建。
> 这个跟 B 型 forceRebuild 的 `os.RemoveAll(主文件)` 完全不同策略。

---

## 四、关闭阶段：db.Close() 与 checkpoint 校准

### 4.1 `CloseDatabase()` 调用链

调用位置：[database.go L1320-L1332](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1320-L1332)

```go
func CloseDatabase() {
    if err := db.Close(); err != nil {
        logging.LogErrorf("close database failed: %s", err)
    }
    if err := historyDB.Close(); err != nil {
        logging.LogErrorf("close history database failed: %s", err)
    }
    if err := assetContentDB.Close(); err != nil {
        logging.LogErrorf("close asset content database failed: %s", err)
    }
    treenode.CloseDatabase()
    logging.LogInfof("closed database")
}
```

每个 DB 的 `closeDatabase()`（内部）实现：
- [siyuan.db 版](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1579-L1589)
- [blocktree 版](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/treenode/blocktree.go#L123-L135)

```go
func closeDatabase() {
    if nil == db { return }
    db.Close()         ← 调用 Go sql.DB.Close()
    debug.FreeOSMemory()
    db = nil
    runtime.GC()
    return
}
```

### 4.2 SQLite `sqlite3_close() 与 WAL Checkpoint 的关系

SQLite 文档规定：当最后一个连接关闭时（`sqlite3_close()）：

在 WAL 模式下，关闭过程中：

1. 所有未提交的事务全部回滚
2. **自动执行一次 checkpoint（把 WAL 中已提交的帧合并入主 db）
3. 如果 checkpoint 成功 → 则 -wal 文件被截断/清空
4. 释放 -shm 文件？→ 关闭 mmap 解除映射

**但是**，有多种情况 checkpoint 可能**不会成功：
- SQLITE_BUSY: 还有其他 reader 在读写（比如连接池其他连接还没关）
- I/O error: 磁盘满 / 文件系统只读
- 锁冲突：另一个进程持有锁

Go 的 `sql.DB.Close()` 会关闭所有连接，所以 **理论上应该成功的最后一个连接也会关闭。但是：
但如果关闭过程中出错了错误，`db.Close()` 返回 error。

### 4.3 SiYuan 中 Close 与 Checkpoint 的真实行为校准

| 阶段 | 调用 | 是否检查返回值 | 失败后做什么 |
|-----|------|:----------:|:-----------:|
| siyuan.db Close | [database.go L1321](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1321) | ✅ 检查 err != nil → LogErrorf | 不阻塞，继续关下一个 |
| historyDB Close | [L1324](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1324) | ✅ | 同上 |
| assetContentDB Close | [L1327](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1327) | ✅ | 同上 |
| treenode Close | [L1330](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1330) | ❓ 调 treenode.CloseDatabase() 内部有关闭 closeDatabase() 里检查 err 但只 LogError | 同上 |

> 所有 Close 失败**都只是记日志不重试**。

### 4.4 正常关闭后文件残留可能性

| 场景 | .db | -wal | -shm | 说明 |
|-----|:---:|:----:|:----:|-----|
| 完美正常关闭 | ✅ 完整 | ❌ 空或被截断 | ❌ 被删除 | checkpoint 100% 成功 |
| Close 返回 SQLITE_BUSY | ✅ | ✅ 仍有数据 | ✅ | 罕见：有其他连接没释放？可能因 SetConnMaxLifetime 长连接未释放 |
| 磁盘满 / 文件系统只读 | ✅ | ✅ | ✅ | I/O error |
| 崩溃 / kill-9 / 断电 | ✅（可能部分页损坏 | ✅ 可能半写帧 | ✅ | 最后 ~N 个事务可能丢 (因为 _synchronous=OFF |

> 注意：`_synchronous=OFF` 是 SiYuan 所有 DB 的默认配置。这意味着 **SQLite 不主动 fsync**。
> 操作系统回写缓存（Windows 默认 ~几秒到几十秒 flush 一次）。崩溃场景下，
> 最后几秒到几十秒前的写操作可能只在 OS page cache 里，还没落盘 → 崩溃就丢了。
> 用户感知："刚才编辑的几行字不见了。

---

## 五、主文件删除后附属文件残留校准

### 5.1 三种 forceRebuild 路径的删除策略对比

| 数据库 | forceRebuild=true 时的删除方式 | 是否删 .db | 是否删 -wal | 是否删 -shm | 代码位置 |
|--------|:-----------------------:|:---------:|:----------:|:----------:|---------|
| siyuan.db (A) | DROP + CREATE 原地 | ❌ 不删 | ❌ 不删 | ❌ 不删 | [database.go L102](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L102) |
| history.db (B) | `os.RemoveAll(主文件)` | ✅ 删 | ❌ 不删 | ❌ 不删 | [database.go L267](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L267) |
| asset_content.db (B) | `os.RemoveAll(主文件)` | ✅ 删 | ❌ 不删 | ❌ 不删 | [database.go L327](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L327) |
| blocktree.db (C) | DROP + CREATE 原地 | ❌ 不删 | ❌ 不删 | ❌ 不删 | [blocktree.go L68](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/treenode/blocktree.go#L68) |

### 5.2 为什么 os.RemoveAll(主文件) 的残留证明

**已知前提**：
1. `util.HistoryDBPath = filepath.Join(TempDir, "history.db")`
   （完整字符串匹配。

2. `os.RemoveAll(path)` 语义：
   - 如果 path 是文件 → 只删这一个文件
   - 如果 path 是目录 → 递归删整个目录树
   - 路径字符串精确匹配字符串，不带后缀扩展

3. `history.db-wal` 的完整文件名**不是** `history.db`，
   它们是两个不同路径。

**结论**：**`os.RemoveAll(history.db)` 绝对不会碰到 `history.db-wal` 和 `history.db-shm`。
**删完后文件系统状态从 `111 → 011（剩 -wal 和 -shm 全在！

### 5.3 删除后再 initDBConnection 的行为

B 型 forceRebuild 完整流程：

```
T0 状态： 111 (.db + .db-wal + .db-shm
            ↓
T1 historyDB.Close()
            ↓
T2 historyDB = nil + runtime.GC()
            ↓
T3 os.RemoveAll(util.HistoryDBPath)    ← 只删 .db 本身！
            ↓
T4 现在状态： 011 (只剩 -wal + -shm)
            ↓
T5 initHistoryDBConnection()
     ├── sql.Open("...history.db?_journal_mode=WAL...")
     │    └── Go database/sql lazy → **还没打文件
     └── SetMaxIdleConns...（还没打文件
            ↓
T6 initHistoryDBTables()
     └── historyDB.Exec("CREATE VIRTUAL TABLE ...")
          └── ⚡ 首次真实写 → 触发第一个连接 + PRAGMA + 文件创建
```

**到 T6 时 SQLite 看到什么？**

没有主库 .db 文件，但有对应的 -wal 和 -shm。

SQLite 的行为：
- SQLite 打开时如果检测到 -wal 文件存在但主 db 不存在或不匹配
- WAL header 里记录的 db page_size / change counter 与实际 db 不一致 → 不一致
- SQLite 可能：
  - 有些版本：**忽略无效的旧 -wal，直接创建新的空 db + 新 wal**（旧 wal 文件被覆盖或留在那
  - 有些版本：报 **"not a database" / "disk I/O error"** 打开失败
  - 具体行为取决于 SQLite 版本和具体状态

> 这是 SiYuan 代码中真实存在的**不一致状态，但是否真的在 B 型 DB forceRebuild 路径会触发**。

### 5.4 残留必然性的校准结论

**结论**：B 型 history 和 asset_content 的 forceRebuild=true 路径下，
**-wal 和 -shm 是必残留**！

| 残留文件 | 是否必然残留 | 原因 |
|-------|:----------:|------|
| `.db-wal` | ✅ **必然残留** | `os.RemoveAll(主文件路径) 精确匹配路径字符串，不带 -wal 后缀 |
| `.db-shm` | ✅ **必然残留** | 同上，不带 -shm 后缀 |

> 注意：如果在 Windows 上某些杀软可能恰好也 lock 文件被占用的情况下有可能还可能还可能还可能，等等。
> 但只要 os.RemoveAll 执行时文件没被占用就能删掉吗？→ 但在大多数正常运行完 Close() 后文件句柄应该释放了 → 此时 .db 能删掉
> 但 -wal 和 -shm **没人删 → 肯定还在。

**残留的后果**：
- 如果下次打开时 SQLite 可能忽略它们，也可能报错。
- 即使成功打开成功，这些旧文件占位，之后新 db 同名覆盖或留着占空间。
- 极端情况：下一次 forceRebuild 的路径下，如果每次都只删 .db 不删 wal/shm，
  这俩文件可能越堆越多？
  不，不会越堆越多，因为文件名是固定的，但内容可能错乱。

---

## 六、四类数据库生命周期全景对比

### 6.1 正常启动流程（forceRebuild=false）对比

| 阶段 | A型 siyuan | B型 history | B型 asset_content | C型 blocktree |
|-----|:----------:|:----------:|:----------------:|:-------------:|
| 1. 加锁 | Mutex Lock | Mutex Lock | Mutex Lock | RWMutex Lock |
| 2. initDBConnection (sql.Open 打开？ | lazy | lazy | lazy | lazy |
| 3. 存在性判断 | stat 表 version | IsExist(.db) | IsExist(.db) | IsExist(.db) |
| 4. 首次真实 I/O | getDatabaseVer → SELECT | 直接 return！无 I/O | 直接 return！无 I/O | 直接 return！无 I/O |
| 5. 完好时结果 | ✅ return | ✅ return (DB 还没打开！) | ✅ return (DB 还没打开！) | ✅ return (DB 还没打开！) |
| 6. 需重建时 | initDBTables + vacuum | Close+删 .db + 重 Open + 建表 | Close+删 .db + 重 Open + 建表 | initDBTables + vacuum |

> 最反直觉的一点：B/C 型 forceRebuild=false + 文件存在 = 函数返回时，**数据库连接还没建立过，文件系统上的 -wal/-shm 是否存在完全没被检查过**。
> 它们可能存在也可能不存在，可能完好也可能损坏，全都不管。

### 6.2 forceRebuild=true 重建流程对比

| 阶段 | A型 siyuan | B型 history | B型 asset_content | C型 blocktree |
|-----|:----------:|:----------:|:----------------:|:-------------:|
| 策略 | 原地 DROP+CREATE | 删除主文件重建 | 删除主文件重建 | 原地 DROP+CREATE |
| 删 .db | ❌ 不删 | ✅ 删 | ✅ 删 | ❌ 不删 |
| 删 -wal | ❌ 不删（但 DROP 可能 checkpoint 合并？ | ❌ **残留 | ❌ **残留** | ❌ 不删 |
| 删 -shm | ❌ 不删 | ❌ **残留** | ❌ **残留** | ❌ 不删 |
| 重建方式 | DROP TABLE + CREATE | 删除后新建空库再建表 | 删除后新建空库再建表 | DROP TABLE + CREATE |
| vacuum | ✅ 有 | ❌ 没有 | ❌ 没有 | ✅ 有 |
| 残留状态 | 111 (文件全在，内容重建 | 011 → 后新建 111 | 011 → 后新建 111 | 111 (文件全在，内容重建) |

### 6.3 malformed 发生时的处理路径对比

| 数据库 | malformed 检测点数量 | 调 RemoveDatabaseFile 吗 | 调 initDatabase(true) 吗 |
|--------|:-----------------:|:----------------------:|:---------------------:|
| siyuan.db | 2 处 (prepareExecInsertTx / execStmtTx) | 1/2 处调 | 2/2 处都调 |
| history.db | **0 处** ❌ | ❌ | ❌ |
| asset_content.db | **0 处** ❌ | ❌ | ❌ |
| blocktree.db | 2 处 (Prepare / Exec) | ❌ 都不调 | 都调 |

---

## 七、已校准后的关键结论

### 7.1 六大校准点汇总

| 校准项目 | 之前的推测 | 代码证实的真相 |
|---------|:-------:|:-----------:|
| sql.Open 后文件是否已创建 | "Open 就建文件 | ❌ **错**。lazy 加载，首次查询/写入时才建 |
| forceRebuild = 先删文件再重建 | "forceRebuild 都是干净重建" | ❌ **错**。A/C 型是原地 DROP+CREATE，B 型才删主文件 |
| B 型 forceRebuild 后文件干净 | "删了重建所以干净" | ❌ **错**。只删 .db，-wal/-shm 必残留 |
| InitHistoryDatabase 返回时 DB 已打开 | "函数返回=数据库就绪" | ❌ **不一定**。文件存在直接 return，连接还没建 |
| db.Close() 一定完成 checkpoint | "Close 就是 checkpoint 完成" | ⚠️ **不一定**。失败只记日志不重试，且 _synchronous=OFF |
| 所有 DB 的 malformed 都能触发清理 | "malformed 就删文件重建" | ❌ **错**。仅 siyuan.db 1/2 路径调了 RemoveDatabaseFile |

### 7.2 一句话总结

> **SiYuan 的四个数据库采用三种初始化和三种判定策略、三种重建策略、三种损坏处理策略，彼此之间的行为并不完全不一致。**
>
> - siyuan.db 是一等公民（有 version 校验、malformed 部分路径调清理、原地重建、vacuum）；
> - blocktree 是二等公民（IsExist 判定、原地重建、有 vacuum、malformed 检测但不调清理）；
> - history 和 asset_content 是三等公民（IsExist 判定、删主文件重建、无 vacuum、完全没有 malformed 检测）。

这种不一致性是**技术债务**，历史发展过程中四个 DB 各自独立演化，没有统一的基础设施层。
在正常运行时感知不到，
但在边界场景（崩溃恢复、forceRebuild、malformed）下行为差异会导致难以调试的 bug。

### 7.3 改进优先级建议

| 优先级 | 改进项 | 理由 |
|:-----:|-------|------|
| 🔴 P0 | B 型 forceRebuild 路径补删 -wal/-shm | 真实 011 死穴状态，真实存在的 bug |
| 🟠 P1 | 所有 malformed 路径统一调 RemoveDatabaseFile | 目前覆盖度不一致 |
| 🟡 P2 | 统一 4 DB 的初始化/重建模式 | 减少技术债务，降低维护成本 |
| 🟡 P2 | db.Close() 失败加重试 + 失败时手动 checkpoint | 提高崩溃恢复一致性 |
| 🟢 P3 | 启动时 `PRAGMA quick_check` 预检 | 提前发现损坏，避免用户跑业务才崩 |

---

## 八、附录：initDatabase 完整时序图（A型 forceRebuild=false，库完好场景）

```
 调用者: sql.InitDatabase(false)
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│  initDatabase(false)                                   │
│  ├─ ClearCache()       ← 清内存缓存                    │
│  ├─ disableCache()    ← 禁用缓存                      │
│  ├─ IncBootProgress(2, "Initializing database...")│
│  ├─ initDBConnection()                              │
│  │   ├─ closeDatabase() (db 为 nil → skip     │
│  │   ├─ LogDatabaseSize() → os.Stat 读元数据       │
│  │   ├─ sql.Open("sqlite3_extended", dsn)          │
│  │   │   └── ⚠️  lazy！还没打文件              │
│  │   └─ SetMaxIdleConns / SetMaxOpenConns             │
│  │                                                   │
│  ├─ treenode.InitBlockTree(false)                   │
│  │   └─ treenode.initDatabase(false)               │
│  │      ├─ initDBConnection() (lazy)               │
│  │      └─ gulu.File.IsExist(BlockTreeDBPath)      │
│  │         └─ 存在 → return (blocktree 也没打开!   │
│  │                                                   │
│  └─ !forceRebuild → getDatabaseVer()               │
│  │   └── db.QueryRow("SELECT ... FROM stat")     │
│  │       └── ⚡ 第一个真实读！                         │
│  │           触发第一个连接 + PRAGMA + 文件打开   │
│  │                                                   │
│  ├─ version 比较                                    │
│  │   ├─ 相等 → return ✅ (函数返回                │
│  │   └─ 不等 → initDBTables() → 建表 → vacuum()    │
│  └─ (结束
```
