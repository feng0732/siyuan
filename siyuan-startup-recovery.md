# SiYuan 启动流程及崩溃恢复代码路径分析

## 1. 概述

本文档梳理 SiYuan 笔记应用的启动流程及崩溃恢复机制，分析环境准备、工作空间加载、服务初始化、异常检测与恢复提示的协作关系，特别关注状态遗留、配置损坏、资源锁定及用户数据保护。

**代码版本**: v3.6.5
**分析范围**: kernel (Go 后端) + app (TypeScript 前端)

---

## 2. 启动流程总览

### 2.1 整体流程结构

```
┌─────────────────────────────────────────────────────────────┐
│                     启动流程主线程                          │
├─────────────────────────────────────────────────────────────┤
│  1. 环境准备 (util.Boot())                                  │
│     ├─ 环境变量初始化                                       │
│     ├─ 命令行参数解析                                       │
│     ├─ 工作空间路径解析                                     │
│     ├─ 工作空间锁获取                                       │
│     └─ 目录结构初始化                                       │
│                                                             │
│  2. 配置加载 (model.InitConf())                             │
│     ├─ 语言检测与初始化                                     │
│     ├─ conf.json 读取与反序列化                             │
│     ├─ 配置字段默认值填充                                   │
│     └─ 配置合法性校验                                       │
│                                                             │
│  3. 服务启动 (server.Serve()) ────┐                         │
│     ├─ Gin 框架初始化             │  异步 goroutine         │
│     ├─ 中间件注册                 │                         │
│     ├─ 路由注册                   │                         │
│     └─ 端口监听                   │                         │
│                                  │                         │
│  4. 外观初始化                   │                         │
│     └─ 主题、图标加载             │                         │
│                                  │                         │
│  5. 数据库初始化                 │                         │
│     ├─ 主数据库 (siyuan.db)      │                         │
│     ├─ 历史数据库 (history.db)    │                         │
│     ├─ 资源数据库 (asset_content.db) │                     │
│     └─ 块树数据库 (blocktree.db)  │                         │
│                                  │                         │
│  6. 启动同步引导                 │                         │
│     └─ BootSyncData()            │                         │
│                                  │                         │
│  7. 笔记本索引 (InitBoxes)       │                         │
│                                  │                         │
│  8. 标记启动完成                 │                         │
│     └─ util.SetBooted() → 100%   │                         │
│                                  │                         │
│  9. 后台任务启动 ─────────────────┘                         │
│     ├─ 定时任务 (job.StartCron)                             │
│     ├─ 自动历史生成                                           │
│     ├─ 资源缓存加载                                           │
│     ├─ 文件系统状态检测                                       │
│     └─ 文件监视器启动                                         │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心入口文件

| 模块 | 文件路径 | 核心函数 |
|------|---------|---------|
| 主入口 | [main.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/main.go#L30-L60) | `main()` |
| 环境准备 | [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L84-L172) | `Boot()` |
| 配置加载 | [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L123-L300) | `InitConf()` |
| 服务启动 | [server/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/server/serve.go#L133-L273) | `Serve()` |
| 数据库初始化 | [sql/database.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L71-L350) | `InitDatabase*()` |

---

## 3. 阶段一：环境准备

### 3.1 环境变量与命令行参数

**代码位置**: [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L56-L141)

```go
func Boot() {
    initEnvVars()                    // 环境变量初始化
    IncBootProgress(3, "Booting kernel...")
    initMime()                       // MIME 类型注册
    initHttpClient()                 // HTTP 客户端初始化
    
    // 命令行参数解析
    workspacePath := flag.String("workspace", "", "...")
    port := flag.String("port", "0", "...")
    readOnly := flag.String("readonly", "false", "...")
    accessAuthCode := flag.String("accessAuthCode", "", "...")
    // ... 更多参数
    flag.Parse()
    
    // 环境变量降级：CLI 参数为空时使用环境变量
    workspacePath = coalesceToEnvVar(workspacePath, "SIYUAN_WORKSPACE_PATH")
    accessAuthCode = coalesceToEnvVar(accessAuthCode, "SIYUAN_ACCESS_AUTH_CODE")
}
```

**关键设计**:
- **参数优先级**: 命令行参数 > 环境变量 > 默认值
- **Docker 安全检查**: Docker 部署时强制要求设置访问授权码，否则以 `ExitCodeSecurityRisk` 退出
- **容器类型识别**: 通过 `/.dockerenv` 文件和 `RUN_IN_CONTAINER` 环境变量检测容器环境

### 3.2 工作空间路径解析

**代码位置**: [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L246-L322)

```go
func initWorkspaceDir(workspaceArg string) {
    // 读取工作空间历史记录
    workspaceConf := filepath.Join(HomeDir, ".config", "siyuan", "workspace.json")
    workspacePaths, _ := ReadWorkspacePaths()
    
    // 路径选择优先级:
    // 1. 命令行参数指定的路径
    // 2. 最近使用的工作空间（workspace.json 最后一个）
    // 3. 系统默认路径
    //    - Windows: %USERPROFILE%/SiYuan
    //    - macOS: ~/Library/Application Support/SiYuan
    //    - Linux: ~/SiYuan
    
    // 路径验证：不存在则创建
    if !gulu.File.IsDir(WorkspaceDir) {
        os.MkdirAll(defaultWorkspaceDir, 0755)
    }
    
    // 初始化子目录路径
    ConfDir = filepath.Join(WorkspaceDir, "conf")
    DataDir = filepath.Join(WorkspaceDir, "data")
    RepoDir = filepath.Join(WorkspaceDir, "repo")
    HistoryDir = filepath.Join(WorkspaceDir, "history")
    TempDir = filepath.Join(WorkspaceDir, "temp")
}
```

### 3.3 工作空间资源锁定

**代码位置**: [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L504-L516)

```go
func tryLockWorkspace() {
    WorkspaceLock = flock.New(filepath.Join(WorkspaceDir, ".lock"))
    ok, err := WorkspaceLock.TryLock()
    if ok {
        return
    }
    // 锁获取失败，以 ExitCodeWorkspaceLocked 退出
    os.Exit(logging.ExitCodeWorkspaceLocked)
}
```

**锁定机制特点**:
- 使用 `github.com/gofrs/flock` 库实现跨平台文件锁
- 锁文件位置: `<workspace>/.lock`
- 非阻塞尝试获取（`TryLock`），失败立即退出
- 退出时通过 `UnlockWorkspace()` 释放锁并删除锁文件

**潜在风险**:
- 崩溃时锁文件可能遗留，导致下次启动失败
- 网络文件系统上锁的可靠性依赖于具体实现

### 3.4 目录结构初始化

**代码位置**: [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L406-L447)

```go
func initPathDir() {
    os.MkdirAll(ConfDir, 0755)      // 配置目录
    os.MkdirAll(DataDir, 0755)      // 数据目录
    os.MkdirAll(TempDir, 0755)      // 临时目录
    os.MkdirAll(DataDir/assets, 0755)   // 资源文件
    os.MkdirAll(DataDir/templates, 0755) // 模板
    os.MkdirAll(DataDir/widgets, 0755)   // 挂件
    os.MkdirAll(DataDir/plugins, 0755)   // 插件
    os.MkdirAll(DataDir/emojis, 0755)    // 表情
    os.MkdirAll(DataDir/public, 0755)    // 公开访问
}
```

---

## 4. 阶段二：配置加载与验证

### 4.1 配置文件结构

**代码位置**: [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L51-L89)

配置文件路径: `<workspace>/conf/conf.json`

```go
type AppConf struct {
    LogLevel       string           `json:"logLevel"`
    Appearance     *conf.Appearance `json:"appearance"`
    Lang           string           `json:"lang"`
    FileTree       *conf.FileTree   `json:"fileTree"`
    Editor         *conf.Editor     `json:"editor"`
    Export         *conf.Export     `json:"export"`
    Account        *conf.Account    `json:"account"`
    System         *conf.System     `json:"system"`
    Sync           *conf.Sync       `json:"sync"`
    Search         *conf.Search     `json:"search"`
    AI             *conf.AI         `json:"ai"`
    // ... 30+ 个配置字段
}
```

### 4.2 配置加载流程

**代码位置**: [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L123-L300)

```go
func InitConf() {
    initLang()  // 语言初始化
    
    Conf = NewAppConf()
    confPath := filepath.Join(util.ConfDir, "conf.json")
    
    if gulu.File.IsExist(confPath) {
        // 读取配置文件
        data, err := os.ReadFile(confPath)
        if err != nil {
            logging.LogErrorf("load conf [%s] failed: %s", confPath, err)
        } else {
            // JSON 反序列化 - 配置损坏检测点
            if err = gulu.JSON.UnmarshalJSON(data, Conf); err != nil {
                logging.LogErrorf("parse conf [%s] failed: %s", confPath, err)
            }
        }
    }
    
    // 语言检测优先级: CLI参数 > 配置文件 > 系统语言 > 默认(en_US)
    // 字段默认值填充 - 对 nil 字段设置默认值
    if nil == Conf.Appearance {
        Conf.Appearance = conf.NewAppearance()
    }
    if nil == Conf.UILayout {
        Conf.UILayout = &conf.UILayout{}
    }
    // ... 大量字段边界检查与默认值设置
}
```

### 4.3 配置保存机制

**代码位置**: [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L870-L900)

```go
func (conf *AppConf) Save() {
    if util.ReadOnly {
        return  // 只读模式不保存
    }
    
    Conf.m.Lock()
    defer Conf.m.Unlock()
    
    newData, _ := gulu.JSON.MarshalIndentJSON(Conf, "", "  ")
    confPath := filepath.Join(util.ConfDir, "conf.json")
    
    // 读取旧数据比较，无变更则不写入
    oldData, err := filelock.ReadFile(confPath)
    if err != nil {
        conf.save0(newData)
        return
    }
    
    if bytes.Equal(newData, oldData) {
        return  // 数据未变化，跳过写入
    }
    
    conf.save0(newData)
}

func (conf *AppConf) save0(data []byte) {
    confPath := filepath.Join(util.ConfDir, "conf.json")
    // 使用 filelock 确保原子写入
    if err := filelock.WriteFile(confPath, data); err != nil {
        logging.LogErrorf("write conf [%s] failed: %s", confPath, err)
        util.ReportFileSysFatalError(err)
        return
    }
}
```

**配置损坏防护**:
- 读取时 JSON 反序列化失败仅记录日志，不中断启动（使用默认值）
- 写入时使用 `filelock.WriteFile` 确保原子性（先写临时文件再 rename）
- 写入失败时调用 `ReportFileSysFatalError` 退出

---

## 5. 阶段三：服务初始化

### 5.1 HTTP 服务启动

**代码位置**: [server/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/server/serve.go#L133-L273)

```go
func Serve(fastMode bool, cookieKey string) {
    gin.SetMode(gin.ReleaseMode)
    ginServer := gin.New()
    
    // 中间件注册顺序很重要
    ginServer.Use(
        model.ControlConcurrency,  // 请求串行化
        model.Timing,              // 耗时统计
        model.Recover,             // Panic 恢复
        corsMiddleware(),          // CORS 支持
        jwtMiddleware,             // JWT 解析
        gzip.Gzip(...),            // GZIP 压缩
    )
    
    // Session 管理
    sessionStore = cookie.NewStore([]byte(cookieKey))
    ginServer.Use(sessions.Sessions("siyuan", sessionStore))
    
    // 静态资源服务
    serveAssets(ginServer)
    serveAppearance(ginServer)
    serveWidgets(ginServer)
    servePlugins(ginServer)
    serveEmojis(ginServer)
    serveTemplates(ginServer)
    servePublic(ginServer)
    serveSnippets(ginServer)
    
    // 协议服务
    serveWebSocket(ginServer)    // WebSocket 实时通信
    serveWebDAV(ginServer)       // WebDAV 文件访问
    serveCalDAV(ginServer)       // CalDAV 日历
    serveCardDAV(ginServer)      // CardDAV 通讯录
    
    // API 路由
    api.ServeAPI(ginServer)
    
    // 端口绑定
    ln, err := net.Listen("tcp", host+":"+util.ServerPort)
    if err != nil {
        os.Exit(logging.ExitCodeUnavailablePort)
    }
    
    // 记录端口到 port.json
    rewritePortJSON(pid, port)
    
    // 启动代理服务（固定端口、发布服务）
    go proxy.InitFixedPortService(host, useTLS, certPath, keyPath)
    go proxy.InitPublishService()
    
    // 启动服务
    util.HttpServer.Serve(ln)
}
```

### 5.2 关键中间件

#### 5.2.1 异常恢复中间件

**代码位置**: [model/session.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/session.go#L446-L449)

```go
func Recover(c *gin.Context) {
    defer logging.Recover()  // 捕获 panic 并记录日志
    c.Next()
}
```

#### 5.2.2 并发控制中间件

**代码位置**: [model/session.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/session.go#L456-L480)

```go
func ControlConcurrency(c *gin.Context) {
    if websocket.IsWebSocketUpgrade(c.Request) {
        c.Next()
        return
    }
    
    // 按 URL 路径粒度加锁，实现同一路径请求串行化
    path := c.Request.URL.Path
    requestingLock.Lock()
    lock, exists := requesting[path]
    if !exists {
        lock = &sync.Mutex{}
        requesting[path] = lock
    }
    requestingLock.Unlock()
    
    lock.Lock()
    defer lock.Unlock()
    
    c.Next()
}
```

**设计目的**: 防止并发写入导致的数据不一致，特别是 SQLite 数据库的并发写入限制。

### 5.3 WebSocket 实时通信

**代码位置**: [server/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/server/serve.go#L697-L851)

```go
func serveWebSocket(ginServer *gin.Engine) {
    util.WebSocketServer = melody.New()
    
    ginServer.GET("/ws", func(c *gin.Context) {
        util.WebSocketServer.HandleRequest(c.Writer, c.Request)
    })
    
    // 连接时鉴权
    util.WebSocketServer.HandleConnect(func(s *melody.Session) {
        // Cookie 验证或 Token 验证
        authOk := checkCookieAuth(s) || checkJWTAuth(s) || checkAuthSession(s)
        if !authOk {
            s.CloseWithMsg([]byte("  unauthenticated"))
            return
        }
        util.AddPushChan(s)
    })
    
    // 消息处理
    util.WebSocketServer.HandleMessage(func(s *melody.Session, msg []byte) {
        request := map[string]any{}
        gulu.JSON.UnmarshalJSON(msg, &request)
        
        cmdStr := request["cmd"].(string)
        cmdId := request["reqId"].(float64)
        param := request["param"].(map[string]any)
        
        command := cmd.NewCommand(cmdStr, cmdId, param, s)
        cmd.Exec(command)
    })
}
```

---

## 6. 阶段四：数据库初始化

### 6.1 数据库类型与作用

| 数据库文件 | 路径 | 作用 |
|-----------|------|------|
| siyuan.db | `<temp>/siyuan.db` | 主数据库：块索引、全文搜索、属性、引用等 |
| history.db | `<temp>/history.db` | 历史版本全文搜索 |
| asset_content.db | `<temp>/asset_content.db` | 资源文件内容索引 |
| blocktree.db | `<temp>/blocktree.db` | 块树结构索引 |

> **重要**: 所有数据库文件存储在临时目录，启动时重建，不持久化。原始数据仍在 `data/` 目录的 `.sy` 文件中。

### 6.2 主数据库初始化

**代码位置**: [sql/database.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L71-L106)

```go
func InitDatabase(forceRebuild bool) {
    initDatabaseLock.Lock()
    defer initDatabaseLock.Unlock()
    initDatabase(forceRebuild)
}

func initDatabase(forceRebuild bool) {
    util.IncBootProgress(2, "Initializing database...")
    
    initDBConnection()          // 建立连接
    treenode.InitBlockTree(forceRebuild)  // 块树数据库
    
    if !forceRebuild {
        // 检查数据库结构版本
        if util.DatabaseVer == getDatabaseVer() {
            return  // 版本一致，跳过重建
        }
        logging.LogInfof("database structure changed, rebuilding...")
    }
    
    // 重建所有表
    initDBTables()
    vacuum()
}
```

### 6.3 SQLite 连接配置

**代码位置**: [sql/database.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L228-L250)

```go
func initDBConnection() {
    dsn := util.DBPath + "?_journal_mode=WAL" +
        "&_synchronous=OFF" +           // 异步写入，提升性能
        "&_mmap_size=2684354560" +       // 2.5GB 内存映射
        "&_cache_size=-20480" +          // 20MB 缓存
        "&_page_size=32768" +            // 32KB 页大小
        "&_busy_timeout=7000" +          // 7秒锁等待超时
        "&_temp_store=MEMORY" +          // 临时表内存存储
        "&_case_sensitive_like=OFF"
    
    db, err = sql.Open("sqlite3_extended", dsn)
    db.SetMaxIdleConns(20)
    db.SetMaxOpenConns(20)
    db.SetConnMaxLifetime(365 * 24 * time.Hour)
}
```

### 6.4 数据库损坏检测与恢复

**代码位置**: [sql/database.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1508-L1512)

```go
if strings.Contains(err.Error(), "database disk image is malformed") {
    util.RemoveDatabaseFile(util.DBPath)  // 删除损坏的数据库文件
    initDatabase(true)                    // 强制重建
    logging.LogFatalf(logging.ExitCodeUnavailableDatabase, 
        "database disk image is malformed, please restart SiYuan kernel to rebuild it")
}
```

**恢复策略**:
- 检测到 `database disk image is malformed` 错误时
- 删除损坏的数据库文件（包括 `-shm` 和 `-wal` 附属文件）
- 调用 `initDatabase(true)` 强制重建
- 以 `ExitCodeUnavailableDatabase` 退出，提示用户重启

---

## 7. 阶段五：数据索引与启动完成

### 7.1 笔记本初始化

**代码位置**: [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L1001-L1013)

```go
func InitBoxes() {
    blockCount := treenode.CountBlocks()
    initialized := 0 < blockCount
    
    for _, box := range Conf.GetOpenedBoxes() {
        box.UpdateHistoryGenerated()
        
        if !initialized {
            // 首次启动，索引所有文档
            indexBox(box.ID)
        }
    }
    
    logging.LogInfof("tree/block count [%d/%d]", 
        treenode.CountTrees(), blockCount)
}
```

### 7.2 启动完成标记

**代码位置**: [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L213-L217)

```go
func SetBooted() {
    setBootDetails("Finishing boot...")
    bootProgress.Store(100)
    logging.LogInfof("kernel booted")
}
```

**启动进度管理**:
- 使用 `atomic.Int32` 管理启动进度（0-100）
- 通过 `IncBootProgress(progress, details)` 递增进度
- 前端通过 API 轮询 `bootProgress` 和 `bootDetails` 显示启动状态

---

## 8. 异常检测与崩溃恢复机制

### 8.1 异常类型与处理策略

| 异常类型 | 检测点 | 处理策略 | 退出码 |
|---------|--------|---------|-------|
| 工作空间锁占用 | `tryLockWorkspace()` | 立即退出 | `ExitCodeWorkspaceLocked` (3) |
| 配置目录创建失败 | `initPathDir()` | `LogFatalf` 退出 | `ExitCodeInitWorkspaceErr` (1) |
| 端口占用 | `net.Listen()` | 立即退出 | `ExitCodeUnavailablePort` (2) |
| 文件系统错误 | `ReportFileSysFatalError()` | 记录堆栈后退出 | `ExitCodeFileSysErr` (5) |
| 数据库损坏 | SQL 执行错误 | 删除后重启重建 | `ExitCodeUnavailableDatabase` (4) |
| Panic 异常 | `Recover` 中间件 | 捕获并记录日志，不崩溃 | - |
| Docker 安全风险 | `Boot()` 参数检查 | 强制退出 | `ExitCodeSecurityRisk` (6) |

### 8.2 文件系统状态监控

**代码位置**: [util/runtime.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/runtime.go#L251-L361)

```go
func CheckFileSysStatus() {
    for {
        <-thirdPartySyncCheckTicker.C  // 每10分钟检查一次
        checkFileSysStatus()
    }
}

func checkFileSysStatus() {
    // 1. 检查是否在云盘目录
    if IsCloudDrivePath(WorkspaceDir) {
        ReportFileSysFatalError(fmt.Errorf(
            "workspace dir [%s] is in third party sync dir", WorkspaceDir))
        return
    }
    
    // 2. 读写一致性测试（7轮，每轮32次重命名）
    dir := filepath.Join(DataDir, ".siyuan/filesys_status_check")
    for i := 0; i < 7; i++ {
        tmp := filepath.Join(dir, "check_consistency")
        data := make([]byte, 1024*4)
        rand.Read(data)
        os.WriteFile(tmp, data, 0644)
        
        time.Sleep(5 * time.Second)  // 等待同步盘可能的同步操作
        
        for j := 0; j < 32; j++ {
            // 重命名 -> 打开 -> 关闭 -> 重命名回来
            // 验证文件操作的原子性
            renamed := tmp + "_renamed"
            os.Rename(tmp, renamed)
            f, _ := os.Open(renamed)
            f.Close()
            os.Rename(renamed, tmp)
            
            // 检查目录列表，确保只有一个文件
            entries, _ := os.ReadDir(dir)
            // 如果发现多个同名文件，说明同步盘冲突
        }
        os.RemoveAll(tmp)
    }
}
```

**云盘路径检测**:
- iCloud Drive (macOS): 扫描 `~/Library/Mobile Documents` 目录
- 已知云盘关键字: OneDrive, Dropbox, Google Drive, pCloud, 坚果云, 天翼云
- Windows 离线文件属性检测

### 8.3 Panic 恢复机制

**代码位置**: [model/session.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/session.go#L446-L449)

```go
func Recover(c *gin.Context) {
    defer logging.Recover()
    c.Next()
}
```

`logging.Recover()` 内部实现:
- 使用 `recover()` 捕获 panic
- 记录完整堆栈跟踪
- 不重新抛出，让 HTTP 请求返回 500 错误
- 不导致整个进程崩溃

### 8.4 进程信号处理

**代码位置**: [model/process.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/process.go#L32-L38)

```go
func HandleSignal() {
    c := make(chan os.Signal)
    signal.Notify(c, syscall.SIGINT, syscall.SIGQUIT, syscall.SIGTERM)
    s := <-c
    logging.LogInfof("received os signal [%s], exit kernel process now", s)
    Close(false, true, 1)
}
```

### 8.5 UI 进程监控

**代码位置**: [model/process.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/process.go#L44-L82)

```go
func HookDesktopUIProcJob() {
    // 启动30秒后开始监控
    time.Sleep(30 * time.Second)
    
    // 检查 WebSocket 会话数
    if 0 < util.CountSessions() {
        return  // 有活动会话
    }
    
    // 检查已注册的 UI 进程
    uiProcCount := getAttachedUIProcCount()
    if 0 < uiProcCount {
        return
    }
    
    // 15秒后二次确认
    time.Sleep(15 * time.Second)
    uiProcCount = getAttachedUIProcCount()
    if 0 < uiProcCount {
        return
    }
    
    // 15秒后全局进程扫描确认
    time.Sleep(15 * time.Second)
    uiProcCount = getUIProcCount()
    if 0 < uiProcCount {
        return
    }
    
    // 确认无 UI 进程，内核自动退出
    logging.LogWarnf("confirmed no active UI proc, exit kernel process now")
    Close(false, true, 1)
}
```

---

## 9. 状态遗留、配置损坏、资源锁定及用户数据保护

### 9.1 状态遗留问题

**问题场景**: 进程崩溃或强制杀进程导致的状态不一致

**已有的防护措施**:

| 状态类型 | 遗留风险 | 防护机制 | 代码位置 |
|---------|---------|---------|---------|
| 工作空间锁 | `.lock` 文件未删除 | 使用 `flock` 而非文件存在性检测 | [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L504-L516) |
| 临时文件 | `temp/` 目录残留 | 启动时清理 `temp/os` 和 `temp/repo` | [util/working.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/working.go#L306-L315) |
| 端口记录 | `port.json` 残留旧条目 | 启动时重写，退出时清理 | [server/serve.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/server/serve.go#L275-L300) |
| 数据库文件 | `.db`、`.db-wal`、`.db-shm` 损坏 | 检测到 malformed 时删除重建 | [sql/database.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/sql/database.go#L1508-L1512) |
| 事务队列 | 未提交的事务 | 退出时 `FlushTxQueue()` | [model/conf.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/model/conf.go#L744) |

**潜在风险**:
- `flock` 在某些网络文件系统上不可靠
- 崩溃时 `FlushTxQueue()` 未执行可能导致最近的修改丢失
- 数据库 WAL 模式下如果异常退出，下次启动需要更长的恢复时间

### 9.2 配置损坏处理

**损坏场景**:
1. JSON 语法错误（手动编辑、写入中断）
2. 字段类型不匹配（版本升级变更）
3. 部分字段缺失（版本升级新增字段）

**恢复策略**:

```go
// 1. 读取失败 - 记录错误，使用默认值继续
if data, err := os.ReadFile(confPath); err != nil {
    logging.LogErrorf("load conf failed: %s", err)
    // 继续执行，使用 NewAppConf() 的默认值
}

// 2. 反序列化失败 - 记录错误，使用默认值继续
if err = gulu.JSON.UnmarshalJSON(data, Conf); err != nil {
    logging.LogErrorf("parse conf failed: %s", err)
    // 继续执行，部分字段可能已成功反序列化
}

// 3. 字段 nil 检查 - 逐个字段设置默认值
if nil == Conf.Appearance {
    Conf.Appearance = conf.NewAppearance()
}
if nil == Conf.UILayout {
    Conf.UILayout = &conf.UILayout{}
}
// ... 超过20个字段的 nil 检查
```

**写入保护**:
- 使用 `filelock.WriteFile` 原子写入（先写 `.tmp` 文件再 `rename`）
- 写入前比较新旧数据，无变化则不写入
- 写入失败时调用 `ReportFileSysFatalError` 退出

### 9.3 资源锁定机制

**多层锁定策略**:

```
┌─────────────────────────────────────────────────┐
│              资源锁定层级                        │
├─────────────────────────────────────────────────┤
│  1. 进程级锁 (flock)                            │
│     路径: <workspace>/.lock                     │
│     作用: 防止多内核进程同时访问同一工作空间     │
│     持有者: 内核进程                            │
│                                                 │
│  2. 文件级锁 (filelock)                         │
│     作用: 关键文件读写时的原子性保证            │
│     应用: conf.json、workspace.json 等          │
│     实现: 临时文件 + rename 原子操作            │
│                                                 │
│  3. 内存级锁 (sync.Mutex/RWMutex)               │
│     Conf.m: 配置数据读写锁                      │
│     Conf.userLock: 用户数据独立锁               │
│     initDatabaseLock: 数据库初始化锁            │
│     requestingLock: 请求并发控制锁              │
│                                                 │
│  4. SQLite 锁                                   │
│     WAL 模式下的读写锁                          │
│     busy_timeout=7000 毫秒等待                  │
└─────────────────────────────────────────────────┘
```

### 9.4 用户数据保护机制

#### 9.4.1 路径安全检查

**代码位置**: [util/path.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/path.go#L401-L475)

```go
func IsSensitivePath(p string) bool {
    // 系统敏感目录
    prefixes := []string{"/.", "/etc", "/root", "/var", "/proc", "/sys", ...}
    
    // Windows 系统目录
    winPrefixes := []string{`c:\windows\system32`, `c:\windows\system`}
    
    // 工作空间配置目录
    workspaceConfPrefix := strings.ToLower(filepath.Join(WorkspaceDir, "conf"))
    
    // 数据库和日志文件
    if strings.HasSuffix(p, ".db") || strings.HasSuffix(p, ".log") {
        return true
    }
    
    // 用户敏感目录
    homePrefixes := []string{
        filepath.Join(HomeDir, ".ssh"),
        filepath.Join(HomeDir, ".config"),
        filepath.Join(HomeDir, ".bashrc"),
        // ...
    }
}
```

#### 9.4.2 子路径验证

```go
// 防止路径遍历攻击
if !gulu.File.IsSubPath(exportBaseDir, fullPath) {
    c.Status(http.StatusUnauthorized)
    return
}
```

#### 9.4.3 文件名过滤

**代码位置**: [util/file.go](file:///d:/fz/0601/solo-dogfeeding/code/307-siyuan/kernel/util/file.go#L398-L410)

```go
func IsReservedFilename(baseName string) bool {
    // Windows 保留文件名
    reservedNames := []string{
        "CON", "PRN", "AUX", "NUL",
        "COM1", "COM2", "COM3", "COM4", "COM5", "COM6", "COM7", "COM8", "COM9",
        "LPT1", "LPT2", "LPT3", "LPT4", "LPT5", "LPT6", "LPT7", "LPT8", "LPT9",
    }
    
    // 以 '.' 开头的隐藏文件
    if strings.HasPrefix(baseName, ".") {
        return true
    }
    
    // 包含控制字符
    for _, r := range baseName {
        if r < ' ' || r == 0x7f {
            return true
        }
    }
    
    return false
}
```

#### 9.4.4 数据多副本策略

| 数据类型 | 存储位置 | 备份机制 |
|---------|---------|---------|
| 文档数据 | `data/<box>/<path>.sy` | 文件历史、数据仓库、云同步 |
| 配置文件 | `conf/conf.json` | 退出前自动保存、原子写入 |
| 索引数据 | `temp/*.db` | 可随时从 `.sy` 文件重建 |
| 文件历史 | `history/` | 按时间分目录存储 |
| 数据仓库 | `repo/` | Git 版本控制 |

---

## 10. 模块分界与协作关系

### 10.1 核心模块职责

| 模块 | 目录 | 核心职责 | 关键依赖 |
|------|------|---------|---------|
| 启动与环境 | `kernel/util/` | 环境准备、路径管理、资源锁、系统工具 | logging, flock, filelock |
| 配置管理 | `kernel/model/conf.go` | 配置加载、保存、验证 | conf, filelock |
| HTTP 服务 | `kernel/server/` | HTTP、WebSocket、WebDAV 服务 | gin, melody, webdav |
| 数据索引 | `kernel/sql/` | SQLite 数据库操作、全文搜索 | sqlite3, lute |
| 业务逻辑 | `kernel/model/` | 核心业务逻辑、数据操作 | sql, treenode, util |
| API 层 | `kernel/api/` | REST API 路由处理 | model, gin |
| 文件系统 | `kernel/filesys/` | 文件树、目录操作 | - |
| 块树管理 | `kernel/treenode/` | 文档树结构、块操作 | lute, sqlite3 |

### 10.2 模块协作流程

```
前端 (TypeScript)
    │
    ▼  HTTP / WebSocket
kernel/api/ (路由层)
    │
    ▼  参数校验 + 权限检查
kernel/model/ (业务逻辑层)
    │
    ├─► 读操作: kernel/sql/ (索引查询)
    │
    ├─► 写操作: kernel/sql/ (索引更新)
    │        └─► kernel/filesys/ (.sy 文件写入)
    │
    └─► 配置操作: model/conf.go (内存 + 持久化)
```

### 10.3 关键数据流

**启动数据流**:
```
CLI/Env 参数 → util.Boot() → model.InitConf() → server.Serve() 
     → sql.InitDatabase() → model.InitBoxes() → util.SetBooted()
```

**请求数据流**:
```
HTTP 请求 → 中间件链 → api.Handler → model.Func → sql.Query/Exec
     → 结果序列化 → 响应返回
```

---

## 11. 潜在风险分析

### 11.1 已识别的风险点

| 风险类型 | 风险描述 | 影响程度 | 发生概率 | 当前防护 |
|---------|---------|---------|---------|---------|
| **锁文件遗留** | 崩溃后 `.lock` 文件未释放，导致无法启动 | 中 | 低 | 使用 `flock` 内核锁而非文件存在检测 |
| **配置损坏** | JSON 反序列化失败导致部分配置丢失 | 中 | 中 | 逐个字段 nil 检查 + 默认值填充 |
| **数据库损坏** | SQLite 异常退出导致索引损坏 | 高 | 低 | 检测 `malformed` 错误后删除重建 |
| **云盘冲突** | 工作空间放在同步盘导致文件损坏 | 高 | 中 | 启动时 + 每10分钟检查云盘路径 |
| **并发写入** | 多请求同时写入导致数据不一致 | 中 | 中 | `ControlConcurrency` 中间件按路径串行化 |
| **WAL 膨胀** | 大量写入后 WAL 文件未 checkpoint | 低 | 中 | 定期 `vacuum()` |
| **临时目录残留** | 崩溃导致临时文件未清理 | 低 | 中 | 启动时清理 `temp/os`、`temp/repo` |
| **端口冲突** | 固定端口 6806 被占用 | 低 | 低 | 默认使用随机端口，只代理到 6806 |

### 11.2 潜在改进点

1. **锁文件遗留**: 增加启动时锁文件的进程存活检测，如果持有锁的进程已不存在则强制释放
2. **配置损坏**: 增加配置文件的多版本备份机制（如 `conf.json.bak`）
3. **数据库损坏**: 增加索引完整性校验，定期在后台进行全量扫描
4. **云盘检测**: 增加更多云盘厂商的检测（如百度网盘、阿里云盘等）
5. **崩溃恢复**: 增加崩溃前的紧急保存机制（如捕获 SIGSEGV 信号）

---

## 12. 后续验证方向

### 12.1 功能测试场景

| 测试场景 | 测试目的 | 验证要点 |
|---------|---------|---------|
| **崩溃后重启** | 验证崩溃恢复能力 | 1. 杀进程后能否正常启动<br>2. 数据是否完整<br>3. 锁是否正常释放 |
| **配置损坏恢复** | 验证配置损坏恢复 | 1. 手动破坏 conf.json 后启动<br>2. 验证是否使用默认值<br>3. 错误日志是否完整 |
| **数据库损坏恢复** | 验证数据库重建 | 1. 手动破坏 siyuan.db<br>2. 启动时是否检测到损坏<br>3. 重建后数据是否完整 |
| **多进程启动** | 验证工作空间锁 | 1. 启动第一个实例<br>2. 尝试启动第二个实例<br>3. 验证第二个实例正确退出 |
| **云盘路径检测** | 验证云盘防护 | 1. 将工作空间放在 OneDrive 目录<br>2. 验证是否能检测到并退出 |
| **并发写入测试** | 验证并发控制 | 1. 并发发送多个修改请求<br>2. 验证数据一致性<br>3. 验证无死锁 |

### 12.2 性能测试场景

| 测试场景 | 测试目的 | 验证要点 |
|---------|---------|---------|
| **大工作空间启动** | 验证启动性能 | 1. 10000+ 文档的启动时间<br>2. 内存占用变化<br>3. 索引构建进度 |
| **数据库压力测试** | 验证数据库稳定性 | 1. 大量并发读写<br>2. 长时间运行稳定性<br>3. WAL 文件大小控制 |
| **文件系统压力** | 验证文件锁可靠性 | 1. 快速连续重启<br>2. 验证锁状态正确性<br>3. 验证数据完整性 |

### 12.3 异常注入测试

| 测试场景 | 测试目的 | 验证要点 |
|---------|---------|---------|
| **磁盘满** | 验证磁盘满时行为 | 1. 填满磁盘后尝试写入<br>2. 错误处理是否正确<br>3. 恢复后能否正常工作 |
| **网络中断** | 验证云同步异常 | 1. 同步过程中断网<br>2. 验证数据一致性<br>3. 恢复后能否自动重连 |
| **权限不足** | 验证权限处理 | 1. 移除工作空间写权限<br>2. 错误提示是否清晰<br>3. 权限恢复后是否正常 |
| **内存不足** | 验证 OOM 处理 | 1. 限制内存后启动<br>2. 失败时是否有清晰日志<br>3. 不损坏数据 |

---

## 13. 总结

SiYuan 的启动流程和崩溃恢复机制设计体现了对数据安全性的高度重视：

### 设计亮点
1. **分层锁定策略**: 从内核级 flock 到应用级 mutex，多层次保护资源
2. **原子写入保障**: 关键文件使用 filelock 确保写入原子性
3. **优雅降级**: 配置损坏时使用默认值，不影响启动
4. **自我修复**: 数据库损坏时自动标记需要重建
5. **主动防护**: 定期检测文件系统状态，防止第三方同步盘冲突

### 可优化点
1. 崩溃后锁文件的强制释放机制
2. 配置文件的多版本备份
3. 更全面的云盘厂商检测
4. 崩溃前的紧急数据保存

### 关键结论
SiYuan 在用户数据保护方面做了大量工作，通过多副本、索引可重建、原子写入等机制确保数据安全。崩溃恢复流程设计较为完善，大部分异常场景都有对应的处理策略，但在极端场景（如突然断电）下仍有改进空间。

---

## 附录：关键退出码定义

| 退出码 | 常量名 | 含义 |
|-------|--------|------|
| 0 | `ExitCodeOk` | 正常退出 |
| 1 | `ExitCodeInitWorkspaceErr` | 工作空间初始化失败 |
| 2 | `ExitCodeUnavailablePort` | 端口不可用 |
| 3 | `ExitCodeWorkspaceLocked` | 工作空间已被锁定 |
| 4 | `ExitCodeUnavailableDatabase` | 数据库不可用（已损坏） |
| 5 | `ExitCodeFileSysErr` | 文件系统错误 |
| 6 | `ExitCodeSecurityRisk` | 安全风险（Docker 未设访问码） |
| 7 | `ExitCodeFatal` | 致命错误 |
