# SiYuan 桌面端错误提示与异常边界协作分析

> 版本: SiYuan 内核 + Electron 桌面端
> 分析日期: 2026-06-15
> 关联文档: [siyuan-startup-recovery.md](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/siyuan-startup-recovery.md)

---

## 一、整体协作架构总览

桌面端错误提示、内核退出码、启动进度轮询、数据重建提示构成了一个**四层闭环反馈系统**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Electron 主进程 (Node.js)                             │
│                                                                             │
│  ┌─────────────────┐    ┌──────────────────────────┐   ┌─────────────────┐  │
│  │ initKernel()    │    │ 退出码分发中心 (switch)   │   │ showErrorWindow │  │
│  │ 拉起内核子进程  │───▶│  code=20/21/24/25/26     │──▶│  error.html     │  │
│  └────────┬────────┘    └──────────────────────────┘   └─────────────────┘  │
│           │                                                                │
│           │ spawn (detached=false)                                          │
└───────────┼────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          内核 (Go)                                           │
│                                                                             │
│  ┌─────────┐   ┌──────────┐   ┌────────────┐   ┌──────────┐   ┌──────────┐ │
│  │  Boot() │──▶│ InitConf │──▶│ server.Serve│──▶│ InitDB   │──▶│ InitBoxes│ │
│  │ (锁/路径)│   │ (配置)   │   │ (端口监听)  │   │ (索引)   │   │ (重建)   │ │
│  └────┬────┘   └────┬─────┘   └─────┬──────┘   └────┬─────┘   └────┬─────┘ │
│       Exit24        不退出          Exit21         Exit20        setProgress│
└───────┼──────────────┼───────────────┼──────────────┼──────────────┼───────┘
        │              │               │              │              │
        ▼              ▼               ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HTTP 轮询层 (fetch API)                                   │
│                                                                             │
│  ┌──────────────────────────┐    ┌───────────────────────────────┐          │
│  │ 轮询 1: /system/version  │    │ 轮询 2: /system/bootProgress  │          │
│  │ 最多15次 × 500ms = 7.5s  │    │ 每100ms → 进度条+详情更新     │          │
│  └──────────────────────────┘    └───────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**核心协作原则**：
1. **早期致命错误**：内核在 HTTP 服务就绪前检测到的问题 → 直接 `os.Exit(code)`，Electron 通过进程 `close` 事件捕获
2. **启动过程进度**：HTTP 服务就绪后 → 通过 REST API 暴露进度，前端自主轮询
3. **运行时异常**：已进入正常使用阶段 → 通过 `panic recover` + 日志 + 响应码 500 反馈

---

## 二、内核退出码与桌面端错误提示映射

### 2.1 退出码定义及触发场景

| 退出码 | 常量名 | 内核触发位置 | 桌面端错误标题 | 严重度 |
|--------|--------|-------------|---------------|--------|
| **0** | `ExitCodeOk` | 正常退出 / API `/api/system/exit` | - | ✅ 正常 |
| **1** | `ExitCodeFatal` | 节点格式化/导出失败、事务失败 | "内核因未知原因退出" | 🔴 未知 |
| **20** | `ExitCodeUnavailableDatabase` | 数据库文件损坏、表结构操作失败 | "数据库不可用" | 🔴 高 |
| **21** | `ExitCodeUnavailablePort` | HTTP 端口监听被占用/无权限 | "监听端口 N 失败" | 🟡 中 |
| **24** | `ExitCodeWorkspaceLocked` | flock 获取工作空间锁失败 | "工作空间已被锁定" | 🟡 中 |
| **25** | `ExitCodeInitWorkspaceErr` | 目录创建无权限、路径解析失败 | "初始化工作空间失败" | 🔴 高 |
| **26** | `ExitCodeFileSysErr` | 云盘检测命中、文件系统读写一致性测试失败 | "已成功避免潜在的数据损坏" | 🔴 严重 |
| **29** | `ExitCodeSecurityRisk` | Docker 部署未设置 accessAuthCode | （默认分支）未知原因 | 🟡 中 |

> 注：退出码常量定义在外部 logging 包中，上述数值通过 Electron 的 `switch(code)` 反向映射确认。

### 2.2 Electron 退出码分发核心逻辑

退出码分发中心位于 [main.js](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/app/electron/main.js#L633-L668)，结构如下：

```javascript
kernelProcess.on("close", (code) => {
    if (0 !== code) {
        let errorWindowId;
        switch (code) {
            case 20: // DB 损坏 → 提示查看 temp/siyuan.log
            case 21: // 端口占用 → 建议检查防火墙/杀毒软件
            case 24: // 工作空间锁定 → **特殊**: 尝试显示第一个已打开工作空间
            case 25: // 目录权限 → 提示 temp/siyuan.log
            case 26: // 云盘冲突 → 🚒 emoji，建议迁移路径+加入白名单
            case 0:  // 正常退出，不处理
            default: // code=1 或其他 → 未知原因，建议重启
        }
        exitApp(currentKernelPort, errorWindowId);
        bootWindow.destroy();
        resolve(false);
    }
});
```

**关键点**：ExitCode 24（工作空间锁定）有特殊优化逻辑——如果当前有其他工作空间窗口打开，优先 `showWindow(workspaces[0].browserWindow)`，避免用户完全无法操作。

### 2.3 错误窗口系统

错误提示通过独立的 BrowserWindow 加载 [error.html](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/app/electron/error.html)：

**窗口特性**：
- 大小：屏幕宽度 50% × 工作区高度 80%
- 无边框（Windows/Linux）、`titleBarStyle: "hidden"`（macOS）
- `nodeIntegration: true`、`contextIsolation: false`（用于 ipcRenderer 通信）
- 支持亮/暗色自动切换（`prefers-color-scheme: dark`）

**参数传递机制**（通过 URL query string）：

```javascript
errWindow.loadFile(errorHTMLPath, {
    query: {
        home: app.getPath("home"),       // 用户家目录
        v: appVer,                        // 版本号
        title: `<h2>中文标题</h2><h2>EN Title</h2>`,  // 双语标题
        emoji: "⚠️",                       // 装饰 emoji
        content: "...HTML 内容...",         // 详细说明（含超链接）
        icon: path.join(appDir, "stage", "icon-large.png"),
    },
});
```

**error.html 渲染逻辑**：
- `#titleEmoji`：显示大尺寸状态 emoji
- `#titleText`：渲染原始 HTML 双语标题
- `#content`：渲染 HTML 内容（支持超链接）
- `#time` / `#systemInfo`：自动注入发生时间和系统信息
- 底部固定 4 个帮助链接：中文求助 / 英文支持 / 中英下载页

---

## 三、启动进度轮询的双层协作机制

启动进度展示采用**两阶段渐进式轮询**，在启动的不同时间点切换轮询目标和频率。

### 3.1 轮询第一层：内核可用性探测（Electron 主进程）

**触发时机**：`initKernel()` 函数在 `spawn` 内核进程后立即启动

**轮询对象**：`GET http://127.0.0.1:{port}/api/system/version`

**参数**：
- 最大重试次数：15 次
- 重试间隔：500ms
- 超时时间：约 7.5 秒（15 × 500ms）

**代码片段**（[main.js](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/app/electron/main.js#L671-L691)）：

```javascript
let count = 0;
for (; ;) {
    try {
        const apiResult = await net.fetch(getServer() + "/api/system/version");
        apiData = await apiResult.json();
        bootWindow.loadURL(getServer() + "/appearance/boot/index.html");
        break;
    } catch (e) {
        if (14 < ++count) {
            showErrorWindow("获取内核服务端口失败", "...");
            bootWindow.destroy();
            resolve(false);
            return;
        }
        await sleep(500);
    }
}
```

**状态转换条件**：
- ✅ **成功**：`/api/system/version` 返回 HTTP 200 → 切换 boot.html → 进入第二层轮询
- ❌ **超时**：15 次全部失败 → 弹出"获取内核服务端口失败"错误窗口
- ⚠️ **版本不匹配**：apiData.data !== appVer（开发环境跳过）→ 调用 `/api/system/exit` 关闭旧内核，整体返回 false 重新调度

### 3.2 轮询第二层：启动进度状态机（boot.html 页面）

**页面来源**：[appearance/boot/index.html](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/app/appearance/boot/index.html)

**轮询对象**：`GET /api/system/bootProgress`

**轮询频率**：100ms（快速感知进度变化）

**后端 API 实现**（[system.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/api/system.go#L696-L702)）：

```go
func bootProgress(c *gin.Context) {
    ret := gulu.Ret.NewResult()
    defer c.JSON(http.StatusOK, ret)

    progress, details := util.GetBootProgressDetails()
    ret.Data = map[string]any{"progress": progress, "details": details}
}
```

**前端渲染循环**：

```javascript
while (!progressing) {
    const progressResult = await fetch('/api/system/bootProgress')
    const progressData = await progressResult.json()
    document.getElementById('progress').style.width = progressData.data.progress + '%'
    document.getElementById('details').textContent = progressData.data.details
    if (progressData.data.progress >= 100) {
        progressing = true
        if (navigator.userAgent.indexOf('Electron') === -1) {
            redirect()  // 浏览器端跳主界面，Electron 端由 ipc 通知主进程
        }
    } else {
        await sleep(100)
    }
}
```

### 3.3 启动进度节点分布

后端通过 `IncBootProgress(delta, details)` 在各关键阶段设置进度（[working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L189-L195)）：

| 进度值 | 阶段 | 触发函数 | 典型 details |
|:------:|------|---------|-------------|
| ~3 | 环境初始化 | `Boot()` | "Booting kernel..." |
| ~5 | 数据库 | `InitDatabase()` | "Initializing database..." |
| ~8 | 云同步 | `BootSyncData()` | "Syncing data from the cloud..." / "Sync reindexing..." |
| ~11 | 索引列出文件 | `InitBoxes()` → `indexBox()` | "Listing files..." |
| 11~99 | 索引笔记本 | `indexBox()` 逐笔记本分配 | "正在索引 [笔记本路径]..." |
| ~99 | 引用解析 | `InitBoxes()` 后段 | "Resolving refs..." / "Indexing refs..." |
| **100** | 启动完成 | `SetBooted()` | "Finishing boot..." |

**进度存储结构**（[working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L197-L217)）：

```go
var (
    bootProgress     atomic.Int32    // 进度值，原子读写
    bootDetails      string          // 详情文本
    bootDetailsLock  sync.Mutex      // 详情读写锁
)

func SetBooted() {
    setBootDetails("Finishing boot...")
    bootProgress.Store(100)
    logging.LogInfof("kernel booted")
}
```

**Electron 主进程补充轮询**（[main.js](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/app/electron/main.js#L701-L720)）：

```javascript
let progressing = false;
while (!progressing) {
    const progressResult = await net.fetch(getServer() + "/api/system/bootProgress");
    const progressData = await progressResult.json();
    if (progressData.data.progress >= 100) {
        resolve(true);  // initKernel Promise resolve，启动主窗口
        progressing = true;
    } else {
        await sleep(100);
    }
}
```

> **设计观察**：Electron 主进程和 boot.html 页面**同时独立轮询**同一接口，前者用于 Promise 状态机流转，后者用于 UI 实时渲染。两者之间无直接依赖——主进程先 resolve 触发主窗口创建，主窗口 `siyuan-ready-to-show` IPC 消息触发关闭 bootWindow。

---

## 四、数据重建提示的触发与协作链路

数据重建的核心设计哲学是：**索引数据库（temp/*.db）可随时删除重建，.sy 文件才是最终数据源**。这决定了重建提示的交互方式。

### 4.1 数据库损坏的三种检测时机

```
┌──────────────────────────────────────────────────────────────────┐
│                     数据库损坏检测层级                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   时机 1: 启动时结构版本检查 (冷启动)                             │
│   ├── 位置: initDatabase() L92-L98                               │
│   ├── 触发: util.DatabaseVer != getDatabaseVer()                 │
│   └── 行为: 静默重建，不通知用户（后台进行）                      │
│                                                                  │
│   时机 2: 运行时 SQL 执行失败 (热路径)                           │
│   ├── 位置: prepareExecInsertTx() L1508 / execStmtTx() L1523     │
│   ├── 触发: err contains "database disk image is malformed"      │
│   └── 行为: 删除文件 + initDatabase(true) + Exit(20)             │
│           → Electron 弹 "数据库不可用" 错误窗                    │
│           → 用户手动重启后，冷启动触发真正的 rebuild              │
│                                                                  │
│   时机 3: 连接初始化失败 (启动初期)                               │
│   ├── 位置: initDBConnection() sql.Open 失败                     │
│   ├── 触发: 磁盘不可用 / 权限不足                                │
│   └── 行为: LogFatalf → Exit(20)                                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 损坏文件清理流程

**删除逻辑**位于 [RemoveDatabaseFile()](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L563-L578)：

```go
func RemoveDatabaseFile(dbPath string) {
    // 1. 主数据库文件
    os.RemoveAll(dbPath)                    // e.g., temp/siyuan.db
    // 2. WAL 共享内存文件
    os.RemoveAll(dbPath + "-shm")           // temp/siyuan.db-shm
    // 3. WAL 预写日志文件
    // (注意: 代码里实际还需要清理 -wal 文件)
}
```

**清理边界**：
- ✅ 清理：`siyuan.db` / `siyuan.db-shm` / `history.db` / `assetContents.db` / `blocktree.db`
- ❌ 不碰：`data/**/*.sy`（原始数据）、`conf.json`（配置）、`workspace.json`
- ❌ 不碰：`temp/siyuan.log`（日志，便于事后分析）

### 4.3 重建链路的用户视角时序

```
用户侧:  点击启动 ──▶ 启动画面 (Booting...) ──▶ ❌ 意外退出
                                                    │
内核侧:                  ... 运行一段时间后 ...      │ 写入日志:
                         prepareExecInsertTx()       │ "database disk image is
                         检测 malformed 错误         │  malformed, please restart"
                              │                      │
                              ├─ RemoveDatabaseFile()┘  清理损坏文件
                              ├─ initDatabase(true)     标记下次需重建
                              └─ os.Exit(20)            带码退出
                                                    │
Electron:                 捕获 close code=20 ────────┘
                              │
                              ▼
                        showErrorWindow(
                          "数据库不可用",
                          "请查看 工作空间/temp/siyuan.log"
                        )
                              │
用户侧:  关闭错误窗 ──▶ 手动点击重启 ──▶ 启动画面
                                                    │
内核侧:                  InitDatabase(false)         │ 检测无 .db 文件
                              │                      │ 或 DatabaseVer 不匹配
                              ▼                      │
                         initDBTables() ─────────────┘ 空库重建表结构
                              │
                              ▼
                         InitBoxes() → !initialized → indexBox()
                              │
                              ▼ 进度 11→99 逐个笔记本
                       "正在索引 [笔记本路径]..."
```

**关键设计**：**不在同一个进程生命周期内完成"检测→删除→重建→继续服务"的全链路**，而是先 Exit(20)，让用户知晓并重启。原因：

1. **数据一致性**：重建索引需要扫描所有 `.sy` 文件，耗时不确定（可能数分钟至数十分钟）
2. **错误隔离**：如果文件系统真的有问题，原地继续操作可能加剧损坏
3. **用户知情权**：静默修复可能让用户误以为一切正常，而实际索引可能不完整

---

## 五、三类核心异常的边界判断

### 5.1 工作空间锁（Workspace Lock）

**技术基础**：`github.com/gofrs/flock` 实现的**建议性文件锁**（Advisory File Lock）

#### 判断边界

| 维度 | 详细说明 |
|------|---------|
| **锁文件位置** | `<workspace>/.lock` |
| **锁类型** | `LOCK_EX | LOCK_NB`（独占锁 + 非阻塞） |
| **获取时机** | `Boot()` → 解析完工作空间路径后 → `initWorkspaceDir()` 后 → `tryLockWorkspace()` |
| **获取失败条件** | 另一个进程持有 LOCK_EX → `TryLock()` 返回 `ok=false` |
| **退出行为** | `os.Exit(ExitCodeWorkspaceLocked=24)` |

#### 真阳性 / 假阳性 / 假阴性 边界

```
✅ 真阳性（正确拒绝）:
   场景: 正常启动的另一个 SiYuan 实例仍在运行
   原理: flock 与进程生命周期绑定，进程存活 → 锁有效
   用户体验: 弹出"工作空间已被锁定"，并尝试显示已打开的窗口

⚠️ 假阳性（误拒绝 - 锁文件遗留）:
   场景 A: 前一次崩溃 / kill -9 / 任务管理器强杀（在部分文件系统上）
   场景 B: 工作空间在网络盘/NFS/SMB，flock 实现不可靠
   场景 C: 系统休眠唤醒后，部分 OS 状态异常
   原理: .lock 物理文件 ≠ 内核锁状态，物理文件会遗留但锁已释放
   用户体验: 弹出锁定错误，但实际已无进程占用 → 需手动删除 .lock

❌ 假阴性（误允许 - 绕过保护）:
   场景 A: 用户手动复制工作空间文件夹后启动（锁文件路径改变）
   场景 B: 网络文件系统跨客户端 flock 不生效
   场景 C: 不同容器/Pod 挂载同一 HostPath（Linux namespace 隔离）
   后果: 两个内核同时写 data/ → .sy 文件内容冲突、WAL 竞争损坏
```

**代码片段**（[working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L504-L516)）：

```go
func tryLockWorkspace() {
    WorkspaceLock = flock.New(filepath.Join(WorkspaceDir, ".lock"))
    ok, err := WorkspaceLock.TryLock()
    if ok {
        return  // 获取成功，正常继续
    }
    if err != nil {
        logging.LogErrorf("lock workspace [%s] failed: %s", WorkspaceDir, err)
    } else {
        logging.LogErrorf("lock workspace [%s] failed", WorkspaceDir)
    }
    os.Exit(logging.ExitCodeWorkspaceLocked)  // 直接退出，无恢复尝试
}
```

#### 防护盲区与改进空间

- **盲区 1**：没有 `IsWorkspaceLocked()` 预检 → 锁定时已在日志记录错误，用户需自行排查
- **盲区 2**：无法判断是"真占用"还是"锁遗留" → 用户需要手动结束进程 vs 删除文件二选一
- **盲区 3**：多实例启动时，第一个实例正常但窗口被遮挡 → 改进后 Electron 已尝试 `showWindow(workspaces[0])`

---

### 5.2 配置损坏（Conf.json Corruption）

**技术基础**：原子写入 + 启动时宽松解析

#### 判断边界

| 阶段 | 检测方式 | 损坏类型 | 处理策略 | 中断启动？ |
|------|---------|---------|---------|:----------:|
| **读取阶段** | `os.ReadFile` 返回 error | 文件被删/权限不足 | 记录 LogErrorf，继续 | ❌ 不中断 |
| **解析阶段** | `json.Unmarshal` 返回 error | 语法错误/括号不配对/截断 | 记录 LogErrorf，继续 | ❌ 不中断 |
| **字段 nil 检查** | 30+ 字段 `if nil == Conf.X` | 字段缺失/版本迁移遗留 | 逐项填充默认值 | ❌ 不中断 |
| **值范围检查** | `if 1 > MaxListCount` | 非法值（0 或负数） | 夹取至合法区间 | ❌ 不中断 |
| **保存阶段** | `filelock.WriteFile` 失败 | 磁盘满/只读 | `ReportFileSysFatalError` → **Exit(26)** | ⚠️ 中断 |

#### 核心加载逻辑

**读取→解析**（[InitConf()](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L123-L138)）：

```go
confPath := filepath.Join(util.ConfDir, "conf.json")
if gulu.File.IsExist(confPath) {
    if data, err := os.ReadFile(confPath); err != nil {
        logging.LogErrorf("load conf [%s] failed: %s", confPath, err)
        // 不中断：使用 NewAppConf() 创建的默认值
    } else {
        if err = gulu.JSON.UnmarshalJSON(data, Conf); err != nil {
            logging.LogErrorf("parse conf [%s] failed: %s", confPath, err)
            // 不中断：部分字段可能已解析成功，随后的 nil 检查兜底
        }
    }
}
```

**字段兜底示例**（[InitConf()](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L185-L240)）：

```go
if nil == Conf.Appearance { Conf.Appearance = conf.NewAppearance() }
if nil == Conf.UILayout   { Conf.UILayout = &conf.UILayout{} }
if nil == Conf.Keymap     { Conf.Keymap = &conf.Keymap{} }
if 1 > Conf.FileTree.MaxListCount    { Conf.FileTree.MaxListCount = 512 }
if 1 > Conf.FileTree.MaxOpenTabCount { Conf.FileTree.MaxOpenTabCount = 8 }
if 32 < Conf.FileTree.MaxOpenTabCount { Conf.FileTree.MaxOpenTabCount = 32 }
if 2 > Conf.FileTree.LargeFileWarningSize { Conf.FileTree.LargeFileWarningSize = 8 }
```

#### 原子写入保护

**保存逻辑**（`save0()`）：

```go
func (conf *AppConf) save0(data []byte) {
    confPath := filepath.Join(util.ConfDir, "conf.json")
    if err := filelock.WriteFile(confPath, data); err != nil {
        util.ReportFileSysFatalError(err)  // 写入失败 → Exit(26)
        return
    }
}
```

`filelock.WriteFile` 内部流程：
1. `os.WriteFile(confPath + ".tmp", data)` 写入临时文件
2. `os.Rename(confPath + ".tmp", confPath)` 原子替换
3. 任何一步失败都不影响原 conf.json

#### 损坏类型矩阵与可恢复性

| 损坏场景 | 可自动恢复 | 用户体验 | 风险等级 |
|---------|:----------:|---------|:--------:|
| conf.json 被完全删除 | ✅ 完全自动 | 启动后所有配置回到默认 | 🟢 低 |
| JSON 截断（写一半断电） | ✅ 完全自动 | 同上，回到默认 | 🟢 低 |
| JSON 语法错误但部分解析 | ✅ 部分自动 | 正确解析的字段保留，错误字段用默认 | 🟡 中 |
| 字段值语义错误（端口="abc"） | ⚠️ 部分场景 | 某些非数值夹取无法覆盖 → 子系统失败 | 🟠 中高 |
| 原子写入时磁盘满 | ❌ 需人工 | Exit(26) → "避免潜在数据损坏"窗口 → 腾空间后重启 | 🔴 高 |
| 版本跨度过大迁移失败 | ⚠️ 依赖 nil 检查 | 新增字段默认值填充，已废弃字段残留不影响 | 🟡 中 |

---

### 5.3 数据库恢复（Database Recovery）

#### 判断边界

| 判断维度 | 启动时检测 | 运行时检测 |
|---------|:----------:|:----------:|
| **触发位置** | `InitDatabase(false)` → `initDatabase()` | `prepareExecInsertTx()` / `execStmtTx()` |
| **检测信号** | `DatabaseVer != getDatabaseVer()` 或 表不存在 | `err.Error() contains "database disk image is malformed"` |
| **检测范围** | 4 个 DB：siyuan / history / assetContents / blocktree | 仅当前执行 SQL 所在的 DB |
| **处理动作** | `initDBTables()` 原地重建表 | `RemoveDatabaseFile()` + `initDatabase(true)` + **Exit(20)** |
| **用户可见** | 不可见（后台静默，可能延长启动） | 可见（错误窗口 + 需手动重启） |
| **数据损失** | 仅索引数据，可重建 | 同上，不影响 .sy 原文件 |

#### 版本号驱动的重建机制

**版本检查逻辑**（[initDatabase()](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L92-L98)）：

```go
if !forceRebuild {
    // 检查数据库结构版本，如果版本不一致的话说明改过表结构，需要重建
    if util.DatabaseVer == getDatabaseVer() {
        return  // 版本一致 → 跳过，不用重建
    }
    logging.LogInfof("the database structure is changed, rebuilding database...")
}
// 版本不一致或 forceRebuild → 走 initDBTables 重建
```

**结构版本的持久化**：
- `setDatabaseVer()` → 写入 `stat` 表 `{key="database_ver", value=util.DatabaseVer}`
- 每次版本库升级时若 DB Schema 有变更，`util.DatabaseVer` 常量 bump
- 这样下次冷启动自动触发全量重建索引

#### 数据库可恢复性 vs. 数据保护边界

```
┌──────────────────────────────────────────────────────────────┐
│                      数据分层与可重建性                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  🔴 不可重建层 (需要备份保护)                                │
│  ├── data/<box>/**/*.sy        原始块树 JSON                │
│  ├── data/assets/**            用户上传的图片/附件           │
│  ├── conf/conf.json            用户配置                      │
│  ├── history/                  文件历史快照                  │
│  └── repo/                     版本库快照                    │
│                                                              │
│  🟢 可重建层 (随时删，自动恢复)                               │
│  ├── temp/siyuan.db           块索引 + FTS 全文索引         │
│  ├── temp/siyuan.db-wal       SQLite 预写日志               │
│  ├── temp/siyuan.db-shm       SQLite 共享内存               │
│  ├── temp/history.db          历史索引                       │
│  ├── temp/assetContents.db    附件内容索引                   │
│  └── temp/blocktree.db        块树缓存                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**关键不变量**：`*.db` 文件中的所有信息都可以从 `data/**/*.sy` + `data/assets/**` + `history/**` 重新生成。这是数据库损坏恢复的**理论基石**。

#### 重建性能影响评估

| 场景 | 重建主体 | 估计耗时 | 进度反馈 |
|-----|---------|:--------:|:--------:|
| 首次启动（空工作空间） | InitBoxes → 无文件 | <1s | 直接到 100% |
| 1000 文档，100MB 附件 | indexBox 逐笔记本 | 5~30s | 可见 "正在索引 [...]" 逐笔记本推进 |
| 10000 文档，1GB 附件 | indexBox + 附件 OCR | 2~10 min | 细粒度 IncBootProgress，可见每个笔记本 |
| 100000 文档级超大型库 | 多轮索引（文件→引用→...） | 30min+ | SetBootDetails 每秒变化 |

---

## 六、异常协作的完整状态转移图

```
                    ┌──────────────────────┐
                    │   用户启动 SiYuan    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
       ┌────────────│  Electron initKernel  │
       │            └──────────┬───────────┘
       │                       │ spawn()
       │                       ▼
       │            ┌──────────────────────┐
       │            │   内核 Boot() 阶段    │
       │            └───┬──────────┬───────┘
       │                │          │
       │          锁成功 ▼          ▼ 锁失败 (Exit24)
       │      ┌─────────────┐  ┌───────────────┐
       │      │ InitConf()  │  │ Electron:     │
       │      └───┬─────────┘  │ "工作空间已锁定"│
       │          │            └───────────────┘
       │  conf损坏│ 正常
       │     ┌────┘
       │     ▼
       │  ┌──────────────┐     端口失败 (Exit21)
       │  │ server.Serve()│────────────────┐
       │  └──────┬───────┘                │
       │         │ : 监听成功             ▼
       │         ▼              ┌─────────────────┐
       │   版本检查轮询         │ "监听端口N失败"  │
       │   (最多15次 × 500ms)  └─────────────────┘
       │         │
       │    ✅  ┌┘
       │        ▼
       │   boot.html 加载
       │   bootProgress 轮询 (100ms)
       │        │
       │        ├── progress < 100 ──► 更新进度条
       │        │
       │        │ progress=100 ──► 进入主界面
       │        │
       │        └────────────────────────┐
       │                                 │
       │    ╔════════ 正常运行 ════════╗ │
       │    ║  用户编辑/保存/同步...   ║ │
       │    ╚════════╤═════════════════╝ │
       │             │                    │
       │   SQL 异常  │                    │ 云盘冲突
       │ (malformed) │                    │ (每10分钟)
       │             ▼                    ▼
       │   ┌──────────────────┐  ┌─────────────────────┐
       │   │ RemoveDatabaseFile│  │ReportFileSysFatalError│
       │   │ initDatabase(true)│  │   Exit(26)          │
       │   │ os.Exit(20)       │  └────────┬────────────┘
       │   └────────┬──────────┘           │
       │            │ Exit(20)             │ Exit(26)
       │            ▼                      ▼
       │   ┌──────────────────┐  ┌──────────────────────┐
       │   │ "数据库不可用"   │  │ "避免潜在数据损坏"   │
       │   │   (提示重启)     │  │  🚒 (迁移路径/白名单)│
       │   └────────┬─────────┘  └──────────┬───────────┘
       │            │                        │
       └────────────┴────────────────────────┘
                用户手动重启后 → 回到流程顶部 (重新索引)
```

---

## 七、关键协作风险与改进方向

### 7.1 退出码定义的 DRY 风险

**现状**：
- Go 端：常量在外部 logging 包，数值与常量名绑定
- Electron 端：`switch(code)` 硬编码魔数 20/21/24/25/26
- **风险点**：如果 Go 端调整 ExitCode 数值，Electron 不会同步报错，导致错误提示与实际问题张冠李戴

**改进建议**：
- 方案 A：通过构建脚本将 ExitCode 输出为 JSON，Electron 启动时读取
- 方案 B：约定 ExitCode 永不改值，新增异常只能用新码值
- 方案 C：在 `/api/system/version` 响应中附带 `exitCodes` 映射表

### 7.2 工作空间锁遗留的用户体验缺陷

**现状**：用户遇到 ExitCode 24 时，错误提示给出两个操作建议（结束进程 / 重启系统），但无法判断实际是哪一种情况

**改进建议**：
1. Exit 24 前尝试写入 `.lock/owner` 文件记录 PID
2. Electron 捕获 Exit 24 后，读取该 PID 并检查进程是否存活：
   ```javascript
   const pid = parseInt(fs.readFileSync(lockOwnerPath, 'utf8'));
   const isAlive = process.kill(pid, 0);  // 信号 0 不杀进程，仅测试存在性
   ```
3. 根据结果调整错误文案：
   - 存活 → "已有实例 PID=xxx 运行中，已切换至该窗口"
   - 不存活 → "检测到锁遗留，是否自动清理？[是] [否]"

### 7.3 数据库重建的交互闭环不完整

**现状**：
1. 运行时检测到损坏 → Exit(20) → 弹"数据库不可用" → 用户手动重启
2. 重启后进入长时索引 → 用户看到"正在索引 xxx..."但不知道原因（可能以为是首次启动的正常流程）

**改进建议**：
1. 删除损坏 DB 后写入 `temp/rebuild_needed.json: {"reason":"malformed_detected","timestamp":...}`
2. 下次启动时若检测到该文件：
   - 在 bootProgress 的 details 前缀加"⚠️ 检测到索引损坏，正在重建..."
   - 重建完成后自动清理该标记文件
   - 重建完成后弹出一次性通知："已为您重建索引，上次检测到索引文件损坏"

### 7.4 配置损坏缺少多版本备份机制

**现状**：conf.json 仅依赖原子写入保证单一版本完整性。如果连续多次保存期间遭遇问题（如磁盘逐步耗尽），可能连默认值都无法恢复。

**改进建议**：
1. 采用 `conf.json + conf.json.bak1 + conf.json.bak2` 三级滚动备份
2. InitConf 检测主文件损坏时，自动尝试最近一个可用备份

### 7.5 云盘检测覆盖面扩展

**当前检测规则**（[IsCloudDrivePath()](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/runtime.go#L363-L377)）：
- iCloud 专用 API（isICloudPath）
- 已知路径关键字：OneDrive / Dropbox / Google Drive / pCloud / 坚果云
- 文件可用性状态测试（existAvailabilityStatus，macOS 云盘占位文件特有）

**缺失覆盖**：
- 百度网盘（`BaiduNetdisk` / `BaiduSyncdisk`）
- 阿里云盘（`AliyunDrive`）
- 腾讯微云（`QQMicroCloud` / `Weiyun`）
- 亿方云、飞书云文档本地同步等

---

## 八、结论：设计哲学的权衡

SiYuan 的错误提示与异常边界体系体现了三个核心的工程权衡：

| 权衡维度 | 选择 | 代价 |
|---------|------|------|
| **数据安全 vs. 启动成功率** | **数据安全优先** — 配置损坏不中断启动、云盘检测直接 Exit(26)、DB 损坏跨生命周期重建 | 用户可能遇到"反复崩溃直到手动清理"的体验问题 |
| **简单性 vs. 可观测性** | **简单性优先** — 退出码纯整数传递、无错误详情结构体、轮询而非 WebSocket push | 调试难度高、新退出码需要两处同步修改 |
| **静默修复 vs. 用户知情** | **用户知情优先** — 索引损坏不原地重建、退出后让用户重启前看到明确错误窗口 | 多了一次"重启"的交互步骤 |

整体来看，这一体系在**保护用户数据不被进一步损坏**上设计非常严谨，但在**降低用户认知负担**和**提升一键恢复体验**方面仍有可优化空间。上述七个改进方向的共同目标都是：在不牺牲数据安全底线的前提下，让用户更轻松地从异常状态中恢复。
