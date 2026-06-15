# SiYuan 导入导出格式适配实现分析

## 1. 整体架构概览

SiYuan（思源笔记）的导入导出系统采用**分层架构**设计，由 API 层、业务模型层、格式转换层和底层工具层构成。系统核心围绕 Lute 引擎提供的 AST（抽象语法树）进行数据流转，实现多种外部格式与内部数据模型之间的双向转换。

### 1.1 流程分层

```
┌─────────────────────────────────────────────────────────┐
│                      API 接入层                          │
│  [router.go]  HTTP/WebSocket 路由 + 鉴权中间件            │
├─────────────────────────────────────────────────────────┤
│                    API 处理层                             │
│  [api/import.go]  [api/export.go]  [api/pandoc.go]       │
│   请求参数校验 → 文件上传接收 → 调用 Model 层             │
├─────────────────────────────────────────────────────────┤
│                   业务模型层                              │
│  [model/import.go]  [model/export.go]                    │
│  [model/export_merge.go]  [model/assets.go]              │
│   格式解析 → AST 转换 → 资源处理 → 数据持久化             │
├─────────────────────────────────────────────────────────┤
│                   格式转换层                              │
│  [util/pandoc.go]  Lute 引擎 (HTML/Markdown/AST)         │
│   Pandoc 外部调用 + Lute 内部解析渲染                     │
├─────────────────────────────────────────────────────────┤
│                   数据持久层                              │
│  [treenode/]  [sql/]  [filesys/]  [cache/]               │
│   文件系统读写 + 数据库索引 + 内存缓存                    │
└─────────────────────────────────────────────────────────┘
```

### 1.2 核心模块功能清单

| 模块 | 文件 | 核心职责 |
|------|------|---------|
| 导入业务 | [import.go](kernel/model/import.go) | `.sy.zip`、`data.zip`、Markdown 文件夹/文件/ZIP 导入，ID 重映射、引用修正 |
| 导出业务 | [export.go](kernel/model/export.go) | Markdown、HTML、PDF、DOCX、`.sy.zip`、多种 Pandoc 格式导出 |
| 导出合并 | [export_merge.go](kernel/model/export_merge.go) | 子文档合并为单一文档（用于 Word/PDF 导出） |
| 资源管理 | [assets.go](kernel/model/assets.go) | 资源文件（图片、附件）处理、OCR、缩略图 |
| Pandoc 集成 | [pandoc.go](kernel/util/pandoc.go) | Pandoc 二进制初始化、路径校验、命令行调用 |
| 导入 API | [import.go](kernel/api/import.go) | 4 个导入端点：`importSY`、`importData`、`importStdMd`、`importZipMd` |
| 导出 API | [export.go](kernel/api/export.go) | 30+ 个导出端点，覆盖所有导出格式 |
| 路由注册 | [router.go](kernel/api/router.go) | 统一鉴权（`CheckAuth`、`CheckAdminRole`、`CheckReadonly`） |
| 导出配置 | [export.go](kernel/conf/export.go) | 20+ 项导出参数配置（块引模式、水印、标签标记等） |
| 树形节点 | [tree.go](kernel/treenode/tree.go) | AST 节点操作、文档规范版本（Spec）校验 |
| API 结果 | [result.go](kernel/util/result.go) | 统一结果结构 `Result{Code, Msg, Data}` |
| WS 广播 | [websocket.go](kernel/util/websocket.go) | WebSocket 消息广播、进度推送、消息推送 |
| 前端 HTTP 封装 | [fetch.ts](app/src/util/fetch.ts) | 统一 POST 请求 + `processMessage` 拦截 |
| 前端消息处理 | [processMessage.ts](app/src/util/processMessage.ts) | 全局消息拦截器：错误 Toast、进度/消息命令分发 |
| 前端 Toast | [message.ts](app/src/dialog/message.ts) | 消息提示渲染（showMessage/hideMessage） |
| 前端进度 | [processSystem.ts](app/src/dialog/processSystem.ts) | 进度遮罩渲染（progressLoading）、事务错误弹窗 |
| 前端 WS 连接 | [Model.ts](app/src/layout/Model.ts) | WebSocket 连接管理、消息分发、断连重连 |

---

## 2. 外部格式解析

### 2.1 支持的格式矩阵

SiYuan 的导入导出支持**原生格式**与**Pandoc 桥接格式**两大类：

#### 导入格式

| 格式 | 入口函数 | 解析方式 | 说明 |
|------|---------|---------|------|
| `.sy.zip` | `ImportSY()` | dataparser.ParseJSON() | SiYuan 原生导出包，内含 `.sy` JSON 文档 |
| `data.zip` | `ImportData()` | 直接文件复制 | 全量数据备份还原 |
| Markdown (文件夹) | `ImportFromLocalPath()` | Lute `parse.Parse()` | 支持 YAML Front Matter |
| Markdown (单文件) | `ImportFromLocalPath()` | Lute `parse.Parse()` | 同上 |
| Markdown (ZIP) | `importZipMd` API | 解压 + 同上 | ZIP 包内 Markdown 批量导入 |
| HTML | `HTML2Tree()` | Lute `HTML2Tree()` | 用于粘贴/导入时的 HTML 转换 |

#### 导出格式

| 格式 | 入口函数 | 渲染方式 | 说明 |
|------|---------|---------|------|
| Markdown | `ExportMarkdownContent()` | Lute ProtyleExportMdRenderer | 支持 YFM、块引脚注、资源 ID 移除 |
| `.sy.zip` | `exportSYZip()` | 直接复制 `.sy` 文件 | 保留完整内部结构，含数据库/闪卡 |
| HTML | `ExportHTML()` | Lute ProtyleExportRenderer | 完整静态站点导出（含主题/图标/表情） |
| PDF | `ProcessPDF()` | HTML → Chrome/Puppeteer → pdfcpu 后处理 | 支持书签、水印、附件嵌入 |
| DOCX | `ExportDocx()` | HTML → Pandoc → DOCX | 需 Pandoc 二进制，含颜色过滤器 |
| Pandoc 系列 | `ExportPandocConvertZip()` | Markdown → Pandoc | EPUB、RTF、ODT、Org、AsciiDoc、MediaWiki、OPML、Textile、reST 等 |
| CSV (数据库) | `ExportAv2CSV()` | 手动构建 CSV | 属性视图导出，UTF-8 BOM |
| 代码块文件 | `ExportCodeBlock()` | 直接写出 | 单个代码块导出为源文件 |

### 2.2 Markdown 解析流程

Markdown 解析的入口是 `parseStdMd()` 函数，关键步骤如下：

```
Markdown 字节流
    │
    ▼
parse.Parse()  ──── Lute 引擎解析为 AST
    │
    ▼
normalizeTree()  ── 提取 YFM（id/title/updated）、规范化节点
    │
    ▼
htmlBlock2Inline()  ── HTML Block 中的 <img>/<a> 转为 Markdown 语法节点
    │
    ▼
parse.TextMarks2Inlines()  ── TextMark 转换为行级内联节点
    │
    ▼
parse.NestedInlines2FlattedSpansHybrid()  ── 嵌套内联展平
    │
    ▼
parse.Tree (内部标准 AST)
```

**关键实现位置：** [parseStdMd](kernel/model/import.go#L1210-L1222)

### 2.3 Pandoc 桥接机制

对于超出 Lute 引擎能力的格式（DOCX、EPUB、Org 等），SiYuan 采用 **Pandoc 作为外部转换器**：

1. **初始化阶段** (`InitPandoc()`)：
   - 优先检查用户自定义 Pandoc 路径（需通过二进制魔数校验）
   - 回退到内置 Pandoc ZIP 包（按平台/架构选择）
   - 解压到临时目录并赋予执行权限
   - 通过 `--version` 输出验证可用性

2. **调用阶段** (`Pandoc()` / `ConvertPandoc()`)：
   - 先生成中间格式 Markdown (GFM + footnotes + hard_line_breaks)
   - 构造 `exec.Command`，设置工作目录为资源路径
   - 合并 stderr 和 stdout 用于错误报告

**安全校验：** [IsValidPandocBin](kernel/util/pandoc.go#L228-L302) 实现了严格的二进制校验：
- 解析符号链接
- 文件存在性 + 非目录 + 普通文件
- 读取文件头 16 字节，拒绝 shebang (`#!`) 脚本
- 匹配 ELF(0x7fELF)、PE(MZ)、Mach-O、FAT 等魔数
- 最终 `--version` 输出必须以 `"pandoc"` 开头

---

## 3. 内部模型转换

### 3.1 核心数据结构

SiYuan 内部统一使用 Lute 引擎的 `parse.Tree` 作为文档表示：

```go
type Tree struct {
    Root       *ast.Node   // 文档根节点（NodeDocument）
    ID         string      // 文档根块 ID
    Box        string      // 笔记本 ID
    Path       string      // 相对路径 (如 "/20230101000000-abcdefg.sy")
    HPath      string      // 人类可读路径 (如 "/笔记/技术")
    // ...
}
```

每个块节点 (`ast.Node`) 带有：
- `ID`：全局唯一标识符（时间戳+随机串，如 `20230101000000-abcdefg`）
- `KramdownIAL`：属性列表（IAL），存储自定义元数据
- `Type`：节点类型枚举（段落、标题、列表、块引、表格等）

### 3.2 导入时的转换流程

以 **Markdown 文件夹导入** (`ImportFromLocalPath`) 为例，完整的转换链路为：

```
本地文件系统
    │
    ├─ 目录遍历 (filelock.Walk)
    │     ├─ 跳过 . 开头的隐藏文件
    │     ├─ Markdown 文件 → parseStdMd() → parse.Tree
    │     └─ 其他文件 → 作为资源文件复制到 assets/
    │
    ▼
AST 后处理（4 步转换链）
    │
    ├─ 1. initSearchLinks()
    │     建立 HPath#Heading → BlockID 的映射表
    │
    ├─ 2. convertMdHyperlinks2WikiLinks()
    │     [text](path.md) → [[path|text]]
    │
    ├─ 3. convertWikiLinksAndTags()
    │     [[path|text]] → ((BlockID 'text'))  （块引用语法）
    │     #tag# → 内部标签标记
    │
    └─ 4. mergeTextAndHandlerNestedInlines()
          文本合并 + 嵌套行级节点展平
    │
    ▼
持久化
    ├─ indexWriteTreeIndexQueue()  → 写入块索引
    ├─ sql.IndexTreeQueue()        → 写入 SQL 全文索引
    └─ box.setSort()               → 更新文档排序
```

**关键实现：** [ImportFromLocalPath](kernel/model/import.go#L771-L1208)

### 3.3 导出时的转换流程

导出的核心函数是 `exportTree()`，负责将内部 AST 转换为**目标格式友好**的中间形式：

```
内部 parse.Tree
    │
    ▼
exportTree()  ── 配置化 AST 变换
    │
    ├─ resolveEmbedR()        解析查询嵌入节点（递归展开）
    ├─ blockLink2Ref()        块超链接转块引用
    ├─ collectFootnotesDefs() 收集跨文档块引定义（脚注模式）
    │
    ├─ 节点级变换（ast.Walk）：
    │   ├─ 超级块标记 → 移除（非所见即所得模式）
    │   ├─ 标题 → 设置 id 属性（锚点）
    │   ├─ 行级备注 → 根据配置保留或清空
    │   ├─ 标签 TextMark → #tag# 文本（自定义标记符）
    │   ├─ 文件标注引用 → 文件名-页码-锚文本 / 仅锚文本
    │   └─ 块引用处理（按 BlockRefMode）：
    │       ├─ Mode 2: 锚文本块链 (siyuan://blocks/ID)
    │       ├─ Mode 3: 仅锚文本
    │       └─ Mode 4: 脚注 + 锚点哈希（默认）
    │
    ├─ addTitle (可选)        在开头插入 H1 标题
    └─ resolveFootnotesDefs() 生成脚注定义块
    │
    ▼
目标格式 Renderer
    ├─ Markdown → NewProtyleExportMdRenderer
    ├─ HTML     → NewProtyleExportRenderer / ProtylePreview
    └─ DOCX     → NewProtyleExportDocxRenderer
```

**关键实现：** [exportTree](kernel/model/export.go#L2423-L2627)

---

## 4. 资源文件处理

### 4.1 导入时的资源处理

资源处理在导入流程中发生在**多个层次**：

#### 4.1.1 Base64 图片处理 (`processBase64Img`)

```
data:image/png;base64,...
    │
    ├─ 解析 MIME 类型 (image/png, image/jpeg, image/svg+xml)
    ├─ Base64 解码
    │    └─ PNG 解码失败时回退尝试 JPEG
    ├─ 使用 alt 文本 + 块 ID 生成文件名: "imagename-20230101000000-abc.png"
    ├─ 文件名过滤 (FilterUploadFileName) 防路径穿越
    ├─ 写入临时目录 → 复制到 data/assets/
    └─ AST 链接目标替换为 "assets/imagename-20230101000000-abc.png"
```

**关键实现：** [processBase64Img](kernel/model/import.go#L1250-L1328)

#### 4.1.2 HTML SVG 图片处理 (`processHTMLBlockSvgImg`)

检测到 HTML Block 中包含 `<svg>` 时：
1. 提取 SVG 内容
2. 保存为独立 `.svg` 资源文件
3. 将原 HTML Block 转为段落 + Markdown 图片语法节点

#### 4.1.3 相对路径资源文件

在 `ImportFromLocalPath` 中，对每个链接节点（`NodeLinkDest` 或 TextMark `a`）：
1. 检查是否为相对路径且本地文件存在
2. Markdown 文件链接（非 assets/ 下）保留为文档链接，后续转为块引
3. 其他文件统一视为资源：
   - 过滤文件名
   - 生成带块 ID 的唯一文件名
   - 复制到目标 assets 目录
   - 引用路径改写为 `assets/filename`
4. 使用 `assetsDone` map 去重，避免同一资源重复复制

#### 4.1.4 `.sy.zip` 导入资源整合

在 `ImportSY()` 中：
- 遍历所有 `assets/` 子目录 → 统一合并到 `data/assets/`
- 遍历所有 `emojis/` 子目录 → 统一合并到 `data/emojis/`（同时过滤非法文件名防 XSS）
- `storage/av/` 数据库文件 → 合并到 `data/storage/av/`（同时重生成 ID 并修正引用）
- `storage/riff/` 闪卡数据 → 合并到内置闪卡卡包

### 4.2 导出时的资源处理

#### 4.2.1 资源发现 (`getAssetsLinkDests`)

通过 `ast.Walk` 遍历 AST，收集以下节点中的资源路径：
- `NodeLinkDest`（Markdown 图片/链接）
- TextMark `a`（行级超链接）
- `NodeIFrame` / `NodeAudio` / `NodeVideo`（嵌入媒体）

#### 4.2.2 资源复制策略

| 导出格式 | 资源处理方式 |
|---------|-------------|
| `.sy.zip` | 保持 `assets/` 相对路径结构，PDF 附带 `.sya` 标注文件 |
| HTML | 复制到导出目录下的 `assets/`，同时复制主题/图标/表情 |
| DOCX | Pandoc 通过 `--resource-path` 查找，可选复制独立 assets 文件夹 |
| PDF | 资源以 HTTP 临时 URL 提供给浏览器渲染，支持附件嵌入 |
| Markdown | 默认保持 `assets/` 相对路径，可配置移除资源文件名中的 ID |
| 社区发布 (Liandi) | 上传到云端图床，URL 替换为云端地址 |

#### 4.2.3 网络资源本地化

`netAssets2LocalAssets` API 提供单独的资源本地化操作，将文档中的网络图片 URL 下载到本地 assets。

---

## 5. 错误反馈与异常处理

### 5.1 统一结果结构

所有 API 端点使用 `gulu.Ret.NewResult()` 返回标准结构：

```go
type Result struct {
    Cmd       string         `json:"cmd"`
    Code      int            `json:"code"`      // 0=成功, -1/1=失败
    Msg       string         `json:"msg"`       // 用户可读错误信息
    Data      any            `json:"data"`      // 返回数据
    // ...
}
```

**定义位置：** [result.go](kernel/util/result.go#L35-L46)

### 5.2 API 层错误处理模式

以 `importSY` 为例的标准错误处理链路：

```go
func importSY(c *gin.Context) {
    ret := gulu.Ret.NewResult()
    defer c.JSON(200, ret)  // 始终返回 HTTP 200，错误在 Code 字段

    form, err := c.MultipartForm()
    if err != nil {
        logging.LogErrorf("parse import .sy.zip failed: %s", err)
        ret.Code = -1
        ret.Msg = err.Error()  // 错误消息透传
        return                 // 提前返回，defer 确保 JSON 响应
    }
    // ... 后续步骤同理
}
```

**关键点：**
- HTTP 状态码始终为 200，业务错误通过 `Code` 字段传递
- 所有错误同时写入 `logging.LogErrorf` 进行日志持久化
- 文件操作使用多重 `defer close()`，防止资源泄漏

### 5.3 Panic 恢复机制

在 `ImportFromLocalPath` 中使用了 `recover()` 进行顶层兜底：

```go
defer func() {
    if e := recover(); nil != e {
        stack := debug.Stack()
        msg := fmt.Sprintf("PANIC RECOVERED: %v\n\t%s\n", e, stack)
        logging.LogErrorf("import from local path failed: %s", msg)
        err = errors.New("import from local path failed, please check kernel log for details")
    }
}()
```

这确保了即使底层发生空指针、越界等严重错误，也不会导致内核进程崩溃，而是返回友好的用户提示并保留完整堆栈日志。

### 5.4 渐进式进度反馈

导入导出操作使用 `util.PushEndlessProgress()` / `util.ClearPushProgress()` 通过 WebSocket 向前端推送进度消息，格式为 `操作描述 当前步骤/总步骤`，例如：

```
正在导入 15/42 /笔记/技术/架构设计
正在导出 正在处理 assets/image-20230101000000-abc.png
```

### 5.5 常见错误场景与反馈

| 错误场景 | 处理方式 | 用户反馈 |
|---------|---------|---------|
| 上传文件路径穿越 | `gulu.File.IsSubPath()` 校验 | "import path is not sub path of import dir" |
| 敏感路径导入 | `util.IsSensitivePath()` 校验 | "local path is sensitive path" |
| Pandoc 未安装 | `IsValidPandocBin()` 检测 + 自动解压内置包 | 配置提示或使用纯 Markdown 回退 |
| 磁盘空间不足 | `ExportDataInFolder()` 预检查 | 导出数据需要至少 N 字节可用空间 |
| `.sy.zip` 结构非法 | 解压后目录数量/名称模式校验 | 使用 `Conf.Language(199)` 国际化提示 |
| 数据与工作区冲突 | `ImportData()` 预检查 `.sy` 文件存在性 | 使用 `Conf.Language(198)` 国际化提示 |

### 5.6 前端错误与进度反馈传递链路

导入导出操作的反馈通过**两条独立通道**从后端传递到前端 UI：HTTP 响应通道（同步结果）和 WebSocket 推送通道（异步进度/消息）。

#### 5.6.1 双通道架构

```
┌────────────────────────────────────────────────────────────────────┐
│                        后端 Kernel                                 │
│                                                                    │
│  Model 层                                                          │
│  ├─ util.PushEndlessProgress(msg) ──┐                              │
│  ├─ util.PushProgress(code,cur,tot) ├─→ websocket.go              │
│  ├─ util.PushMsg(msg, timeout)  ────┘   BroadcastByType()         │
│  │                                       ↓                        │
│  └─ ret.Code = -1 / ret.Msg = "..." ──→ API JSON 响应              │
│                                          ↓                        │
└──────────────────────────────────────────┼─────────────────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
             WebSocket 通道          HTTP 响应通道                │
             (异步推送)              (同步返回)                  │
                    │                      │                      │
                    ▼                      ▼                      │
┌─────────────────────────────────────────────────────────────────┐
│                       前端 App                                   │
│                                                                  │
│  Model.ts (WebSocket 消息入口)                                   │
│  ├─ ws.onmessage → processMessage(data) ──→ msgCallback          │
│  │     ├─ cmd="msg"    → showMessage()    (Toast 提示)          │
│  │     ├─ cmd="cmsg"   → hideMessage()    (关闭指定提示)        │
│  │     ├─ cmd="progress" → progressLoading() (进度遮罩)         │
│  │     ├─ cmd="cprogress" → 移除进度遮罩                        │
│  │     └─ code < 0     → showMessage(type=error/info)           │
│  │                                                               │
│  fetch.ts (HTTP 请求入口)                                        │
│  └─ fetchPost(url, data, cb)                                     │
│        └─ response → processMessage(response)                    │
│              ├─ 返回 false → 不调用 cb (错误已处理)              │
│              └─ 返回 response → 调用 cb (正常流程)               │
│                                                                  │
│  导入调用示例 (menus/navigation.ts)                              │
│  ├─ fetchPost("/api/import/importSY", formData, () => {          │
│  │     reloadDocTree();  // 仅 code=0 时执行                     │
│  │  });                                                          │
│  └─ 错误由 processMessage 自动展示，业务代码无需处理              │
└──────────────────────────────────────────────────────────────────┘
```

#### 5.6.2 同步错误通道：HTTP → processMessage → showMessage

前端所有 API 调用统一通过 `fetchPost()` 发起，响应经过 `processMessage()` 统一拦截：

1. **`fetchPost()`** ([fetch.ts](app/src/util/fetch.ts)) 发起 POST 请求，接收 JSON 响应
2. **`processMessage(response)`** ([processMessage.ts](app/src/util/processMessage.ts)) 拦截判断：
   - `code < 0`：自动调用 `showMessage(msg, timeout, type)`，返回 `false` 阻断回调
     - `code === -1`：类型为 `"error"`（红色 Toast，附加版本号）
     - `code === -2`：类型为 `"info"`（蓝色 Toast）
   - `code === 0`：正常，返回原始 `response`，由业务回调处理
3. **`showMessage()`** ([message.ts](app/src/dialog/message.ts)) 渲染 DOM：
   - 在 `#message` 容器中插入 `.b3-snackbar` 元素
   - 错误类型添加 `.b3-snackbar--error` 红色样式
   - `timeout > 0` 时自动定时隐藏，`timeout === 0` 时显示关闭按钮需手动关闭
   - 支持按 `messageId` 去重更新（同一 ID 的消息只保留最新内容）

**导入场景示例：** 前端调用 `/api/import/importSY` 时，后端若返回 `{code: -1, msg: "import path is not sub path..."}` → `processMessage` 判断 `code < 0` → 自动弹出红色 Toast → **不执行** `reloadDocTree()` 回调。

#### 5.6.3 异步进度通道：WebSocket → progressLoading

耗时操作（批量导入/导出）通过 WebSocket 实时推送进度：

1. **后端推送** ([websocket.go](kernel/util/websocket.go#L307-L316))：
   - `PushEndlessProgress(msg)` → `BroadcastByType("main", "progress", 1, msg, ...)` — 无确定进度条
   - `PushProgress(0, current, total, msg)` → 有确定进度条
   - `ClearPushProgress(total)` → `PushProgress(2, total, total, "")` — 关闭进度

2. **前端接收**：
   - **桌面端主窗口** ([index.ts](app/src/index.ts))：`case "progress": progressLoading(data)`
   - **移动端** ([onMessage.ts](app/src/mobile/util/onMessage.ts))：`case "progress": progressLoading(data)`
   - **独立窗口** ([window/index.ts](app/src/window/index.ts))：同上

3. **`progressLoading()`** ([processSystem.ts](app/src/dialog/processSystem.ts#L440-L469)) 渲染进度 UI：
   - `code === 0`（有进度）：显示进度条 + `current/total` + 消息文本
   - `code === 1`（无进度/无尽）：显示条纹动画进度条 + 消息文本
   - `code === 2`（完成）：移除 `#progress` 元素

4. **`processMessage()` 中的 `cprogress` 命令**：直接移除 `#progress` 元素，用于紧急取消进度遮罩

#### 5.6.4 异步消息通道：WebSocket → msg/cmsg

后端通过 `PushMsg()` / `PushMsgWithApp()` 推送消息类通知：

1. **`PushMsg(msg, timeout)`** ([websocket.go](kernel/util/websocket.go#L230-L234))：
   - 生成随机 `msgId`，通过 `BroadcastByType("main", "msg", 0, msg, {id, closeTimeout})` 广播
   - 前端 `processMessage` 中 `cmd === "msg"` → `showMessage(msg, timeout, "info"/"error", id)`

2. **`PushClearMsg(msgId)`** ([websocket.go](kernel/util/websocket.go#L319-L321))：
   - 前端 `cmd === "cmsg"` → `hideMessage(data.id)`，精确关闭指定消息

3. **`ContextPushMsg()`** ([websocket.go](kernel/util/websocket.go#L278-L290))：
   - 根据 context 中的 `CtxPushMsg` 值选择推送方式：
     - `CtxPushMsgToNone`：不推送
     - `CtxPushMsgToProgress`：推送到进度遮罩
     - `CtxPushMsgToStatusBar`：推送到状态栏
     - `CtxPushMsgToStatusBarAndProgress`：同时推送状态栏和进度

#### 5.6.5 前端消息流转汇总

| 通道 | 后端函数 | WebSocket cmd | 前端处理函数 | UI 表现 |
|------|---------|---------------|-------------|---------|
| HTTP 同步错误 | `ret.Code = -1` | — | `processMessage()` → `showMessage()` | 右上角红色/蓝色 Toast |
| WS 异步消息 | `PushMsg()` | `msg` | `processMessage()` → `showMessage()` | 右上角 Toast（可带关闭按钮） |
| WS 消息关闭 | `PushClearMsg()` | `cmsg` | `processMessage()` → `hideMessage()` | 关闭指定 Toast |
| WS 有进度 | `PushProgress(0,...)` | `progress` | `progressLoading()` | 全屏遮罩 + 进度条 |
| WS 无尽进度 | `PushEndlessProgress()` | `progress` (code=1) | `progressLoading()` | 全屏遮罩 + 条纹动画 |
| WS 进度完成 | `ClearPushProgress()` | `progress` (code=2) | `progressLoading()` | 移除遮罩 |
| WS 紧急取消 | `PushClearProgress()` | `cprogress` | `processMessage()` | 直接移除 `#progress` |
| 事务错误 | — | `txerr` | `transactionError()` | 弹窗（重建索引/退出） |
| 内核崩溃 | — | — (连接断开) | `kernelError()` | 弹窗（重连提示） |

#### 5.6.6 关键设计特征

1. **错误处理去中心化**：`processMessage()` 作为全局拦截器，业务代码（如导入回调）无需手动处理 `code < 0` 的错误场景，减少了遗漏错误处理的风险
2. **进度与消息解耦**：进度通过全屏遮罩展示（阻断用户操作），消息通过右上角 Toast 展示（非阻断），两者独立运行互不干扰
3. **消息去重与更新**：`showMessage()` 支持按 `messageId` 更新已有消息内容，避免同类消息重复弹出造成视觉干扰
4. **多端消息统一**：桌面端（`index.ts`）、移动端（`onMessage.ts`）、独立窗口（`window/index.ts`）三个入口使用相同的 `progressLoading()` / `showMessage()` 函数，保证跨端体验一致
5. **WebSocket 断连恢复**：`Model.ts` 中 `ws.onclose` 检测非主动关闭后，3 秒自动重连；重连成功后自动执行 `reloadSync()` 刷新数据并关闭 `kernelError` 弹窗

### 5.7 导出侧前端反馈分支全景

导出操作的前端反馈逻辑因格式而异，呈现 **5 种独立分支模式**，各自有不同的提示生命周期、文件打开方式和错误处理路径。

#### 5.7.1 五种分支模式总览

| 分支模式 | 适用格式 | 等待提示位置 | 成功后动作 | 关键实现文件 |
|---------|---------|------------|-----------|-------------|
| **A. 简单异步** | `.sy.zip`、Markdown `.zip`、ReST、AsciiDoc、Textile、OPML、Org-Mode、MediaWiki、ODT、RTF、EPUB 等 | 调用 API 前 | `hideMessage` + `openByMobile` 下载 | [commonMenuItem.ts](app/src/menus/commonMenuItem.ts#L608-L785) |
| **B. 路径选择 + 本地保存** | 桌面端 HTML(SiYuan/Markdown)、Word `.docx` | 用户选择目录后 | `afterExport`（6s Toast + "显示在文件夹"） | [index.ts saveExport/getExportPath](app/src/protyle/export/index.ts#L690-L749) |
| **C. 浏览器二次请求** | 浏览器端 HTML(SiYuan/Markdown) | 调用 API 前 | `hideMessage` + `window.open` + Toast | [index.ts saveExport](app/src/protyle/export/index.ts#L34-L61) |
| **D. PDF 预览 + IPC** | 桌面端 PDF | 预览窗口内 | IPC 发送给主进程静默生成 | [index.ts renderPDF](app/src/protyle/export/index.ts#L138-L688) |
| **E. 图片渲染 + 上传** | PNG 图片 | 确认按钮点击时 | `hideMessage` + `openByMobile` | [util.ts exportImage](app/src/protyle/export/util.ts#L28-L188) |

#### 5.7.2 导出前等待提示的统一模式

所有分支的**前置提示**都使用 `showMessage(window.siyuan.languages.exporting, -1)`：
- `-1` 作为 `timeout` 参数，表示**永不自动关闭**（需手动 `hideMessage(msgId)`）
- 返回值 `msgId` 被保存用于后续精确关闭或内容更新

**调用位置分类：**

```
┌─ 立即显示（API 调用前立刻锁定 UI）
│  ├─ 分支 A：.sy.zip / Markdown.zip / ReST / AsciiDoc...
│  │     msgId = showMessage(exporting, -1) → fetchPost(...)
│  │
│  ├─ 分支 C：浏览器 HTML
│  │     msgId = showMessage(exporting, -1) → 2 次 fetchPost
│  │
│  └─ 分支 E：图片导出
│        msgId = showMessage(exporting, 0) → addScript + toBlob
│        (timeout=0 表示显示关闭按钮，用户可手动取消)
│
└─ 延迟显示（等待用户交互后才显示）
   ├─ 分支 B：桌面端 HTML/Word
   │     showOpenDialog（选路径）→ 确认后才 showMessage
   │     （避免用户取消路径选择时出现多余提示）
   │
   └─ 分支 D：PDF
         无前置 Toast → 改为新窗口内 <img loading-pure.svg> 渲染占位
         （预览窗口本身就是等待态，无需全局提示）
```

#### 5.7.3 导出成功后提示关闭的四条路径

##### 路径 1：直接关闭（分支 A、C、E）

```typescript
// 分支 A：Markdown .zip —— [commonMenuItem.ts:L624-L631]
const msgId = showMessage(window.siyuan.languages.exporting, -1);
fetchPost("/api/export/exportMd", {id}, response => {
    hideMessage(msgId);                          // 1. 关闭等待提示
    openByMobile(response.data.zip);             // 2. 打开下载
    // 无额外成功提示：浏览器下载弹窗本身就是视觉反馈
});
```

##### 路径 2：更新提示内容 + 自动关闭（分支 B：`afterExport`）

```typescript
// [util.ts:L16-L26] afterExport(exportPath, msgId)
showMessage(
  `${window.siyuan.languages.exported} ${escapeHtml(exportPath)}
   <div class="fn__space"></div>
   <button class="b3-button b3-button--white">${window.siyuan.languages.showInFolder}</button>`,
  6000,    // 6 秒后自动关闭
  "info",
  msgId    // ← 关键：复用原 msgId，将「导出中」更新为「导出成功」
);
// 为按钮绑定事件：useShell("showItemInFolder") 调用系统文件管理器
document.querySelector(`#message [data-id="${msgId}"] button`).addEventListener("click", () => {
    useShell("showItemInFolder", path.join(exportPath));
    hideMessage(msgId);
});
```

**设计要点：** `showMessage` 传入已存在的 `msgId` 时会执行**更新而非新建**，实现无缝的状态切换动画。

##### 路径 3：隐藏提示 + 独立成功提示（分支 C 浏览器 HTML）

```typescript
// [index.ts:L52-L59] 浏览器 HTML 二次请求成功
hideMessage(msgId);                          // 关闭「导出中」
if (zipResponse.code === -1) { /* 错误处理见 5.7.4 */ }
window.open(zipResponse.data.zip);           // 触发浏览器下载
showMessage(window.siyuan.languages.exported);  // 弹出新的短 Toast（2s）
```

##### 路径 4：无显式关闭（分支 D：PDF）

PDF 导出在独立窗口内执行，用户点击"确认"后：
- `actionElement.remove()` 移除控制面板（视觉上变为纯文档）
- `previewElement.classList.add("exporting")` 调整样式
- 通过 IPC `send(Constants.SIYUAN_EXPORT_PDF, config)` 将任务交给主进程
- 主进程完成后通过系统通知回调，前端无需再 `hideMessage`

#### 5.7.4 浏览器导出二次请求的失败处理

**分支 C** 是唯一包含**两次后端请求**的路径。但由于 `fetchPost` + `processMessage` 的全局拦截机制，代码中显式的 `if (zipResponse.code === -1)` 分支实际上是**死代码**。

##### 拦截机制的三层嵌套

```
fetchPost 调用链
    ↓
[fetch.ts:L88-L91] 统一拦截点
    if (typeof response === "object" 
        && typeof response.msg === "string" 
        && typeof response.code === "number") {
        if (processMessage(response) && cb) {  // ← 关键短路逻辑
            cb(response);                     // 仅 processMessage 返回真值时才执行回调
        }
    }
    ↓
[processMessage.ts:L70-L74] code<0 处理
    if (response.code < 0) {
        showMessage(response.msg, ..., response.code === -1 ? "error" : "info");
        return false;   // ← 返回 false → cb 永远不会被调用！
    }
    return response;     // code ≥ 0 时返回原对象 → cb 正常执行
```

**结论：`code < 0` 的响应在 `fetchPost` 内部就被 `processMessage` 拦截了，回调函数 `cb` 永远不会被执行。**

##### 完整的错误流程时序

```
用户点击导出 HTML
    ↓
msgId = showMessage("导出中", -1)  // 永不自动关闭
    ↓
第 1 次 fetchPost("/api/export/exportHTML", ...)
    ├─ 成功（code=0）→ onExport() 组装 HTML → 第 2 次请求
    └─ 失败（code=-1）→ processMessage 拦截
            ├─ 自动 showMessage(红色错误 Toast)
            ├─ return false
            └─ cb 不执行 → hideMessage(msgId) 不调用 → "导出中"Toast 残留
    ↓
第 2 次 fetchPost("/api/export/exportBrowserHTML", ...)
    ├─ 成功（code=0）→ cb 执行 → hideMessage + window.open + 成功 Toast
    └─ 失败（code=-1）→ processMessage 拦截
            ├─ 自动 showMessage(红色错误 Toast)
            ├─ return false
            └─ cb 不执行 → 以下代码永远不执行：
               - hideMessage(msgId)
               - if (zipResponse.code === -1) { ... }  ← 死代码
               - window.open(...)
```

##### 源码中的死代码

代码中显式编写的错误分支永远不会被执行（L53-L56 在浏览器导出分支）：

```typescript
// [index.ts:L33-L62] saveExport 浏览器环境（#if BROWSER 编译宏）
/// #if BROWSER
if (["html", "htmlmd"].includes(option.type)) {
    const msgId = showMessage(window.siyuan.languages.exporting, -1);  // L36
    const url = option.type === "htmlmd" ? "/api/export/exportMdHTML"   // L38
                                         : "/api/export/exportHTML";
    // 第 1 次 fetchPost
    fetchPost(url, {id, pdf:false, removeAssets:false, merge:true, savePath:""},
      async exportResponse => {
        const html = await onExport(exportResponse, undefined, "", option);
        // 第 2 次 fetchPost
        fetchPost("/api/export/exportBrowserHTML", {
            folder: exportResponse.data.folder,
            html: html,
            name: exportResponse.data.name
        }, zipResponse => {
            // L52 hideMessage：code<0 时永远不会执行
            hideMessage(msgId);
            // ── L53-L56 以下显式 if 判断为死代码 ──
            if (zipResponse.code === -1) {
                showMessage(window.siyuan.languages._kernel[14].replace("%s", zipResponse.msg), 0, "error");
                return;
            }
            // L57-L58 在失败时同样无法到达
            window.open(zipResponse.data.zip);
            showMessage(window.siyuan.languages.exported);
        });
    });
    return;
}
/// #else
```

##### 按代码执行顺序的拦截验证

以第 2 次请求（`/api/export/exportBrowserHTML`）失败为例，逐行追踪：

1. **L47**：调用 `fetchPost(url, data, zipResponse => {...})` 发起请求
2. **`fetch.ts:L41`**：`fetch(url, init)` 发送 HTTP 请求
3. **`fetch.ts:L65-L67`**：检测到 `content-type: application/json` → 调用 `response.json()` 解析
4. **`fetch.ts:L88`**：命中条件 `typeof response.msg==="string"` 且 `typeof response.code==="number"`
5. **`fetch.ts:L89`**：调用 `processMessage(response)`
6. **`processMessage.ts:L71`**：命中 `if (response.code < 0)`（code === -1）
7. **`processMessage.ts:L72`**：自动弹出红色错误 Toast（错误信息来自后端返回）
8. **`processMessage.ts:L73`**：`return false`
9. **`fetch.ts:L89`**：`if (false && cb)` → 短路，**不执行 `zipResponse` 回调**
10. **结果**：`zipResponse` 回调函数体（L52-L58 共 7 行）全部被跳过

对于第 1 次请求失败也是同样的链路，区别在于失败时 L46 `onExport()` 不会被调用，流程停在更早的位置。

##### 潜在 UX Bug

当两次请求中任意一次失败时，用户界面会出现**两个 Toast 同时存在**：
1. **错误 Toast**（红色）：由 `processMessage.ts:L72` 自动弹出，描述具体错误原因（`timeout=0`，需手动关闭）
2. **"导出中"Toast**：由于回调被跳过，`hideMessage(msgId)`（L52）从未被调用，**永久显示**

用户必须手动关闭错误 Toast 后，才能看到卡住的"导出中"Toast，且无法自动消除。

##### 修复方案分析

**错误的修复思路**：在请求发出前先 `hideMessage(msgId)`
- ❌ 问题：如果请求成功，等待提示会过早消失，用户失去状态反馈
- ❌ 问题：无法区分第 1 次请求和第 2 次请求的状态

**正确的修复方案**（按优先级排列）：

1. **方案 A：使用 `fetchSyncPost` + `try/finally`（最简洁）**
   - 利用 `fetchSyncPost` 在 [fetch.ts:L131-L133] 中只调用 `processMessage` 但**不检查返回值**、**总是返回 response** 的特性
   - `finally` 块保证 `hideMessage(msgId)` 无论成功失败都会执行

   ```typescript
   const msgId = showMessage(window.siyuan.languages.exporting, -1);
   try {
       const exportResponse = await fetchSyncPost(url, {...});
       // 检查第 1 次请求是否出错（code<0 时 fetchSyncPost 仍返回对象）
       if (exportResponse.code < 0) return;
       const html = await onExport(exportResponse, undefined, "", option);
       const zipResponse = await fetchSyncPost("/api/export/exportBrowserHTML", {...});
       if (zipResponse.code < 0) return;
       window.open(zipResponse.data.zip);
       showMessage(window.siyuan.languages.exported);
   } finally {
       hideMessage(msgId);  // 永远执行
   }
   ```

2. **方案 B：移除死代码 + 为 fetchPost 增加通用 cleanup 钩子**
   - 删除 L53-L56 的冗余 `if (zipResponse.code === -1)` 分支
   - 为 `fetchPost` 增加第 6 个 `finallyCallback` 参数，在 `.then` 末端和 `.catch` 末端统一调用
   - 优点：所有使用 `fetchPost` 的场景都能受益

3. **方案 C：局部 watchdog 超时兜底（临时补丁）**
   - 调用 `fetchPost` 后注册 setTimeout，60 秒后检测 msgId 对应的 DOM 是否还存在
   - 若存在则强制 `hideMessage`
   - 缺点：治标不治本，只解决 Toast 残留，不解决死代码

##### 失败场景

二次请求可能失败的场景：
1. **临时目录写满**：`exportBrowserHTML` 写入 ZIP 时磁盘空间不足
2. **HTML 体积过大**：`zipResponse.data.zip` URL 超出浏览器限制
3. **临时资源已清理**：第 1 次与第 2 次请求间隔过长，`exportResponse.data.folder` 被定时任务回收
4. **session 过期**：两次请求之间鉴权失效（401 时 `fetch.ts` 会直接 `location.reload()`）

#### 5.7.5 文件打开：`openByMobile` 的 4 平台分发

所有非本地保存格式最终通过 `openByMobile(uri)` 触发文件下载/打开：

```typescript
// [compatibility.ts:L68-L99]
export const openByMobile = (uri: string) => {
    if (isInIOS()) {
        if (uri.startsWith("assets/")) {
            // iOS <16.7 特殊编码路径
            webkit.messageHandlers.openLink.postMessage(origin + "/assets/" + encodeURIComponent(...));
        } else if (uri.startsWith("/")) {
            // 导出 zip 路径已 encode，直接拼接 origin
            webkit.messageHandlers.openLink.postMessage(origin + uri);
        } else {
            // 外部 URL 尝试检测合法性，失败自动补 https://
            try { new URL(uri); postMessage(uri); }
            catch { postMessage("https://" + uri); }
        }
    } else if (isInAndroid()) {
        window.JSAndroid.openExternal(uri);      // Android 原生桥
    } else if (isInHarmony()) {
        window.JSHarmony.openExternal(uri);      // HarmonyOS 原生桥
    } else {
        window.open(uri);                        // 桌面/浏览器：新标签页/下载
    }
};
```

**移动端附加的 `exportByMobile` 版本**（[compatibility.ts:L101-L114]）使用 `JSAndroid.exportByDefault` 强制走系统"导出文件"分享面板，而非直接打开。

#### 5.7.6 分支 D（PDF）的独特反馈链

PDF 导出是最复杂的分支，其反馈链路如下：

```
用户点击"导出 PDF"
    │
    ▼
暗色模式？ ──是──→ confirmDialog 二次确认提示
    │否
    ▼
renderPDF(id)
    │
    ├─ 前端本地生成完整预览 HTML（注入主题/插件/尺寸配置）
    │
    ├─ fetchPost("/api/export/exportTempContent", {html})
    │      → 后端将 HTML 写入临时文件并返回 URL
    │
    └─ ipcRenderer.send(SIYUAN_EXPORT_NEWWINDOW, tempUrl)
           │
           ▼ 打开新的 Electron 窗口
           │
           ├─ 左侧：导出配置面板（纸张/边距/缩放/分栏/水印...）
           │     └─ 任一配置变更 → refreshPreview() → fetchPost("/api/export/exportPreviewHTML")
           │
           └─ 用户点击「确认」按钮
                │
                ├─ actionElement.remove()    // 隐藏配置面板
                ├─ preview.classList.add("exporting")  // 切换样式
                │
                └─ ipcRenderer.send(SIYUAN_EXPORT_PDF, config)
                       │
                       ▼ 主进程执行
                       ├─ webContents.printToPDF(pdfOptions)
                       ├─ showSaveDialog 选择路径
                       └─ 写入文件后系统通知
                              (前端无需反馈操作，IPC 单向)
```

**PDF 分支的独特设计：**
- **全局无 `showMessage`**：避免独立窗口与主窗口 Toast 重叠
- **独立窗口作为视觉反馈载体**：`fn__loading` SVG 动画、配置面板禁用态
- **预览 → 导出的样式切换**：`.exporting` class 调整尺寸、移除滚动条
- **单向 IPC 通信**：导出结果由系统通知/文件管理器体现，前端无回传状态

---

## 6. 格式差异处理

### 6.1 块引用模式差异

SiYuan 的核心特性——**块引用**在不同格式中映射方式完全不同，由 `BlockRefMode` 配置控制：

| Mode | 内部表示 | Markdown 导出 | HTML 导出 |
|------|---------|--------------|----------|
| 2 | 锚文本块链 | `[text](siyuan://blocks/ID)` | 超链接到 siyuan 协议 |
| 3 | 仅锚文本 | 纯文本 | 纯文本 |
| 4 | 脚注 + 锚点哈希（默认） | `text[^1]` + 文末脚注定义 | 内部锚点超链接 |

**实现细节：** [exportTree 中的块引用处理](kernel/model/export.go#L2504-L2554)

### 6.2 超级块与嵌套布局

SiYuan 的**超级块**（SuperBlock）是自定义块类型，在标准 Markdown/HTML 中无对应概念：
- 非所见即所得导出时，移除 `NodeSuperBlockOpenMarker/LayoutMarker/CloseMarker` 标记节点
- 子块按线性顺序输出，布局信息丢失
- 导出 HTML 预览时，同样移除超级块的属性列表以保持输出简洁

### 6.3 表格与数学公式

- **表格单元格**：导入时将 `\|` 转义处理，避免与 Markdown 表格分隔符冲突
- **数学公式**：导出时 `NodeMathBlockContent` 首尾空格被 trim，保证 LaTeX 渲染兼容
- **DOCX 导出**：使用 `html+tex_math_dollars` 输入格式，Pandoc 将 `$...$` 转换为 Word 公式域

### 6.4 IFrame 降级策略

PDF 和 DOCX 导出无法渲染 `<iframe>`，通过 `processIFrame()` 将其降级为超链接：
```
<iframe src="https://example.com"> → [https://example.com](https://example.com)
```

### 6.5 标签语法适配

标签在导出时可自定义包裹标记（`TagOpenMarker` / `TagCloseMarker`），默认 `#tag#`，可适配 Obsidian (`#tag`)、Logseq 等不同工具的标签语法。

---

## 7. 批量处理

### 7.1 导入批量处理

**Markdown 文件夹导入** (`ImportFromLocalPath`) 的批量策略：

1. **递归遍历**：使用 `filelock.Walk` 深度优先遍历目录树
2. **目录转文档**：空目录若包含子 Markdown 文件，自动创建对应文档节点
3. **进度节流**：每处理 4 个文档推送一次进度，避免 WebSocket 消息风暴
4. **后处理批量执行**：
   - `hPathsIDs` 映射：收集所有文档 HPath → ID
   - `idPaths` 映射：收集所有文档 ID → Path
   - 排序：按路径字母顺序排序，构建父子层级排序值
   - `box.setSort()`：一次性写入排序配置

### 7.2 导出批量处理

**多文档导出** (`ExportPandocConvertZip` / `ExportSYs`) 的批量策略：

1. **文档收集**：
   - 起始块对应的文档路径
   - 若 `IncludeSubDocs=true`，递归展开所有子文档
   - `prepareExportTrees()`：预加载并缓存所有树

2. **引用树导出** (`.sy.zip`)：
   - 通过 `exportRefTrees()` DFS 递归发现所有被引用的文档
   - 引用文档平放在导出根目录，使用 `ID.sy` 命名避免路径冲突

3. **资源去重**：
   - 使用 `hashset.New()` 记录已复制资源路径
   - PDF 资源额外检查并复制 `.sya`（PDF 标注）关联文件

4. **子文档合并** (`mergeSubDocs`)：
   - 导出 PDF/DOCX 时可将文档树合并为单一大文档
   - 使用栈遍历，按层级插入标题（`hLevel` 最大为 6，超出则保持在 H6）
   - 跳过末尾空段落

**实现位置：** [mergeSubDocs](kernel/model/export_merge.go#L27-L70)

---

## 8. 兼容性回退

### 8.1 文档规范版本 (Spec) 机制

每个 `.sy` 文档在根节点上带有 `Spec` 字段标识其数据结构版本：

```go
var CurrentSpec = "2"
var ErrSpecTooNew = fmt.Errorf("the document spec is too new")

func CheckSpec(tree *parse.Tree) (err error) {
    if CurrentSpec == tree.Root.Spec || "" == tree.Root.Spec {
        return  // 匹配或为空（旧文档兼容）
    }
    spec, _ := strconv.Atoi(tree.Root.Spec)
    currentSpec, _ := strconv.Atoi(CurrentSpec)
    if spec > currentSpec {
        return ErrSpecTooNew  // 文档由更新版本创建，拒绝加载
    }
    // spec < currentSpec: 需要调用 UpgradeSpec 升级
    return UpgradeSpec(tree)
}
```

**实现位置：** [tree.go CheckSpec](kernel/treenode/tree.go#L135-L150)

导入 `.sy.zip` 时通过 `treenode.UpgradeSpec(tree)` 对旧规范文档进行原地升级。

### 8.2 Pandoc 可用性回退

DOCX/EPUB 等格式的导出检查链路：

```
用户请求导出 DOCX
    │
    ▼
IsValidPandocBin(Conf.Export.PandocBin)?  ── 用户自定义路径
    │
    ├─ 是 → 使用该路径
    │
    └─ 否 → 尝试内置 Pandoc (InitPandoc)
              │
              ├─ 成功 → 自动保存到 Conf.Export.PandocBin
              └─ 失败 → 返回错误 "not found executable pandoc"
```

### 8.3 图标主题回退

HTML 导出时图标处理：
- 优先复制用户当前选择的图标集
- 若非内建图标（ant/material），额外复制 material 图标集作为**后备**，确保导出页面在缺失自定义图标时仍可正常显示

### 8.4 废弃 API 兼容

Router 中大量端点标记为 `deprecated`，计划于 2026-06-30 删除，但目前仍保留实现：
- `/api/system/reloadUI` → 建议 `/api/ui/reloadUI`
- `/api/storage/setLocalStorage` → 建议 `/api/storage/setLocalStorageVal`
- `/api/attr/resetBlockAttrs` → 建议 `/api/attr/setBlockAttrs`

这些端点通过在 handler 链中追加 `deprecated` 中间件进行标记。

### 8.5 属性名兼容回退

在 `Export2Liandi` 中处理社区发布的文章 ID 属性：
```go
const liandiArticleIdAttrName = "custom-liandi-articleid"
const liandiArticleIdAttrNameOld = "custom-liandi-articleId"  // 兼容旧属性名

articleId := tree.Root.IALAttr(liandiArticleIdAttrName)
if "" == articleId {
    articleId = tree.Root.IALAttr(liandiArticleIdAttrNameOld)  // 回退到旧驼峰命名
}
```

---

## 9. 数据完整性保障

### 9.1 ID 重映射机制

导入 `.sy.zip` 时必须为所有块重新生成 ID，避免与现有数据冲突。实现位于 `ImportSY()`，策略极其精细：

**ID 生成规则：**
```go
// 新 ID 保留时间部分（前 14 位），仅修改随机值部分
// 避免时间变化导致更新时间早于创建时间
newNodeID := util.TimeFromID(n.ID) + "-" + util.RandString(7)
```

**引用修正范围（`blockIDs` 映射表应用场景）：**
1. **块引用** (`NodeBlockRef`)：`n.TextMarkBlockRefID = newDefID`
2. **块超链接** (TextMark `a` + `siyuan://blocks/` 前缀)：更新 href
3. **查询嵌入脚本** (`NodeBlockQueryEmbedScript`)：全文本替换所有 ID
4. **数据库关联** (`av.NodeAttrNameAvs` 前缀属性)：键和值双重替换
5. **属性视图节点**：`n.AttributeViewID` 指向新 AV ID
6. **文件路径**：目录名含块 ID 的全部重命名
7. **排序配置** (`sort.json`)：ID 映射后合并
8. **闪卡**：`deck.AddCard(ast.NewNodeID(), blockIDs[card.BlockID()])`
9. **任务队列**：`task.AppendTask(task.UpdateIDs, ...)` 异步更新全文索引

### 9.2 数据库 (Attribute View) 完整性

`.sy.zip` 中的数据库导入流程：

```
storage/av/*.json
    │
    ├─ 1. 生成 avIDs 映射：每个数据库文件重命名为新 ID
    ├─ 2. 文件内容全文本替换：旧 AV ID → 新 AV ID + 旧块 ID → 新块 ID
    ├─ 3. 复制到 data/storage/av/
    ├─ 4. 遍历所有文档，更新块 IAL 中的 av 属性
    ├─ 5. av.BatchUpsertBlockRel() 建立块-数据库关联
    ├─ 6. updateBoundBlockAvsAttribute() 修复文档外绑定块的属性
    └─ 7. av.UpsertAvBackRel() 重建数据库间的关联关系
```

### 9.3 文件操作原子性

- 所有文件写入使用 `filelock.WriteFile()` / `filelock.Copy()`，内置文件锁机制
- 写树使用 `writeTreeUpsertQueue()` 进入事务队列，确保批量写入的 ACID 特性
- 导出时先写入临时目录再打包 ZIP，避免半写文件被用户获取

### 9.4 同步与索引触发

导入完成后统一触发：
```go
IncSync()                              // 标记需要同步
task.AppendTask(task.UpdateIDs, ...)   // 异步更新全文索引
sql.IndexTreeQueue(tree)              // 块级 SQL 索引入队
treenode.IndexBlockTree(tree)         // 块树缓存更新
```

### 9.5 资源文件完整性校验

- `util.FilterUploadFileName()` / `util.FilterUploadEmojiFileName()`：过滤路径穿越字符 (`..`, `/`, `\`) 和 XSS 风险字符
- 资源复制使用 `assetsDone` map 去重，防止同一源路径重复处理
- `GetAssetAbsPath()` 解析资源路径时做安全校验

---

## 10. 潜在风险分析

### 10.1 安全风险

| 风险点 | 影响 | 当前防护 | 残余风险 |
|-------|------|---------|---------|
| 文件名路径穿越 | 任意文件写入 | `FilterUploadFileName` + `IsSubPath` 校验 | ZIP 内符号链接可能绕过（需确认 filelock.Copy 行为） |
| 自定义 Pandoc 路径 RCE | 任意命令执行 | 二进制魔数校验 + 拒绝 shebang 脚本 | 用户将恶意二进制重命名为 pandoc 仍可能执行（需校验输出前缀） |
| Emoji 文件名 XSS | 存储型 XSS | `FilterUploadEmojiFileName` 重命名非法文件 | 历史遗留数据可能未被清理 |
| Base64 图片资源耗尽 | DoS | 无明确大小限制 | 超大 Base64 可能导致内存耗尽 |
| SQL 注入 | 数据泄露 | 使用参数化查询 (gulu ORM) | 动态 SQL 拼接处需审计 |

### 10.2 性能风险

| 风险点 | 触发场景 | 影响 |
|-------|---------|------|
| 批量导入 AST 全量遍历 | 大型 Markdown 文件夹导入 | `ast.Walk` 被调用 4-6 次，复杂度 O(N×深度) |
| ID 替换正则爆炸 | `.sy.zip` 含大量跨文档引用 | `strings.NewReplacer` 对每个 AV 文件内容做全量替换 |
| 资源去重 Hash 碰撞 | 大量同名资源 | 目前使用路径字符串去重而非内容哈希 |
| PDF 内存占用 | 千页级文档导出 | pdfcpu 全量加载到内存处理 |
| 同步阻塞 | 导入导出期间 | `lockSync()` / `unlockSync()` 全局锁，阻塞同步操作 |

### 10.3 一致性风险

| 风险点 | 场景 | 后果 |
|-------|------|------|
| 部分成功导入中断 | 大批量导入过程中进程崩溃 | 已写入的 `.sy` 文件与索引不一致 |
| 跨文档引用悬空 | 引用目标不在导入范围内 | `.sy.zip` 中 `defBlockIDs` 缺失导致块引失效 |
| 数据库引用不一致 | AV 文件复制失败但块属性已更新 | 属性视图打开时找不到对应 JSON |
| 资源引用路径错位 | Markdown 导入时 assets 目录判断歧义 | 链接指向 `assets/*.md` 被误判为资源或文档 |
| Spec 版本不兼容 | 旧内核打开新高版本数据 | `ErrSpecTooNew` 导致文档无法加载 |

---

## 11. 后续排查方向

### 11.1 功能正确性排查

1. **块引用 ID 映射完整性**
   - 检查点：`ImportSY()` 中 `ast.Walk` 是否覆盖了所有包含 ID 的节点类型
   - 排查方法：构造包含嵌入块、数据库、虚拟引用的测试用 `.sy.zip`，导入后逐块比对引用

2. **YAML Front Matter 往返一致性**
   - 检查点：导入时 `parseStdMd` 提取 YFM → 导出时 `yfm()` 生成 YFM 的字段对应关系
   - 排查方法：导入含自定义 YFM 字段的 Markdown 后重新导出，比对字段丢失情况

3. **资源路径大小写/空格敏感性**
   - 检查点：`assetsDestSpace2Underscore` 仅对社区导出生效，本地导出是否存在空格路径问题
   - 排查方法：Windows + Linux 跨平台导入含空格/中文文件名的资源

### 11.2 性能瓶颈排查

1. **导入热点分析**
   - 启用 Go pprof，分析 `ImportFromLocalPath` 在 1000+ 文件时的 CPU 火焰图
   - 重点关注 `ast.Walk`、`parse.Parse`、`filelock.Copy` 的耗时占比

2. **导出内存分析**
   - `ExportData()` 在大工作区时需要 `dataSize * 2` 的临时空间，评估是否需要流式压缩
   - `mergeSubDocs` 合并 100+ 子文档时 AST 节点数爆炸问题

3. **数据库批量操作**
   - `sql.IndexTreeQueue` 的批处理大小与 flush 策略
   - AV 导入时的 ID 替换算法优化（当前为 O(N×M) 字符串替换）

### 11.3 错误恢复排查

1. **导入中断后数据清理**
   - 是否存在临时目录残留：`temp/import/`、`temp/base64/`
   - 已写入但未完成索引的文档是否有自动修复机制

2. **Pandoc 故障诊断**
   - `CombinedOutput` 中文化编码问题（GBK 终端输出乱码）
   - 自定义参数 `shellquote.Split` 失败时的回退行为

3. **脚注定义泄漏**
   - Mode 4 块引导出时，`resolveFootnotesDefs` 生成的脚注定义块可能包含未引用项
   - 需验证 `Improve focus export` 逻辑在聚焦导出场景下的覆盖率

4. **浏览器 HTML 导出死代码**
   - [index.ts:L53-L56](app/src/protyle/export/index.ts#L53-L56) 中 `if (zipResponse.code === -1)` 分支为死代码（`#if BROWSER` 编译宏内）
   - 精确位置：`saveExport` 第 2 次 `fetchPost` 的回调内部（L47-L59）
   - 由于 `fetch.ts:L89` + `processMessage.ts:L70-L74` 全局拦截 `code < 0`，回调永远执行不到该分支
   - 需移除 L53-L56 冗余代码；**不能**在请求发出前就 `hideMessage(msgId)`，否则成功时用户会丢失等待状态

5. **浏览器 HTML 导出 UX Bug**
   - 第 1/2 次请求失败时，回调被跳过 → `hideMessage(msgId)`（L52）不会被调用 → "导出中"Toast 永久残留
   - 用户会同时看到红色错误 Toast + 卡住的"导出中"Toast，体验不佳
   - 修复方案推荐：
     - 方案 A（最简洁）：改用 `fetchSyncPost` + `try/finally` 包裹（`fetchSyncPost` 不拦截 code<0 响应，`finally` 保证 hideMessage 总执行）
     - 方案 B（通用）：为 `fetchPost` 增加第 6 个 `finallyCallback` 参数，在 `.then/.catch` 末端统一执行 cleanup
     - **注意**：不能在请求发出前就关闭等待提示，也不能依赖当前仅对 `/api/file/getFile` 生效的 `failCallback` 参数

### 11.4 兼容性验证

1. **Spec 升级路径**
   - 构造 Spec=1 的历史 `.sy` 文件，验证 `UpgradeSpec` 是否正确升级到 Spec=2
   - Spec 未来升级至 3 时的向后兼容测试

2. **旧属性名覆盖率**
   - 全局搜索旧命名模式（如驼峰 → 下划线过渡），建立兼容回退清单

3. **Pandoc 版本矩阵**
   - 测试 Pandoc 2.x / 3.x 对 DOCX 数学公式、颜色过滤器的行为差异

---

## 12. 关键代码索引

| 功能 | 函数/方法 | 文件位置 |
|------|----------|---------|
| .sy.zip 导入 | `ImportSY` | [import.go:L110-L702](kernel/model/import.go#L110-L702) |
| 数据全量还原 | `ImportData` | [import.go:L704-L769](kernel/model/import.go#L704-L769) |
| Markdown 本地导入 | `ImportFromLocalPath` | [import.go:L771-L1208](kernel/model/import.go#L771-L1208) |
| Markdown 标准解析 | `parseStdMd` | [import.go:L1210-L1222](kernel/model/import.go#L1210-L1222) |
| Base64 图片解码 | `processBase64Img` | [import.go:L1250-L1328](kernel/model/import.go#L1250-L1328) |
| HTML→AST 转换 | `HTML2Tree` | [import.go:L59-L108](kernel/model/import.go#L59-L108) |
| Markdown 超链接转块引 | `convertMdHyperlinks2WikiLinks` | [import.go:L1542-L1590](kernel/model/import.go#L1542-L1590) |
| 核心导出变换 | `exportTree` | [export.go:L2423-L2627](kernel/model/export.go#L2423-L2627) |
| Markdown 内容导出 | `exportMarkdownContent0` | [export.go:L2282-L2421](kernel/model/export.go#L2282-L2421) |
| Pandoc 格式批量导出 | `ExportPandocConvertZip` | [export.go:L1724-L1749](kernel/model/export.go#L1724-L1749) |
| .sy.zip 打包 | `exportSYZip` | [export.go:L1853-L2021](kernel/model/export.go#L1853-L2021) |
| HTML 导出 | `ExportHTML` | [export.go:L1017-L1187](kernel/model/export.go#L1017-L1187) |
| DOCX 导出 | `ExportDocx` | [export.go:L755-L840](kernel/model/export.go#L755-L840) |
| PDF 后处理 | `ProcessPDF` | [export.go:L1245-L1301](kernel/model/export.go#L1245-L1301) |
| 子文档合并 | `mergeSubDocs` | [export_merge.go:L27-L70](kernel/model/export_merge.go#L27-L70) |
| Pandoc 初始化 | `InitPandoc` | [util/pandoc.go:L109-L206](kernel/util/pandoc.go#L109-L206) |
| Pandoc 二进制校验 | `IsValidPandocBin` | [util/pandoc.go:L228-L302](kernel/util/pandoc.go#L228-L302) |
| 文档 Spec 版本校验 | `CheckSpec` | [treenode/tree.go:L139-L150](kernel/treenode/tree.go#L139-L150) |
| 导出配置定义 | `Export` 结构体 | [conf/export.go:L19-L48](kernel/conf/export.go#L19-L48) |
| API 路由注册 | `ServeAPI` | [api/router.go:L25-L486](kernel/api/router.go#L25-L486) |
| 导入 API 端点 | 4 个 handler | [api/import.go](kernel/api/import.go) |
| 导出 API 端点 | 30+ 个 handler | [api/export.go](kernel/api/export.go) |
| WS 广播核心 | `BroadcastByType` | [util/websocket.go:L82-L92](kernel/util/websocket.go#L82-L92) |
| WS 进度推送 | `PushProgress` / `PushEndlessProgress` / `ClearPushProgress` | [util/websocket.go:L292-L316](kernel/util/websocket.go#L292-L316) |
| WS 消息推送 | `PushMsg` / `PushClearMsg` / `ContextPushMsg` | [util/websocket.go:L230-L290](kernel/util/websocket.go#L230-L290) |
| 前端 WS 连接 | `Model` 类 | [app/src/layout/Model.ts](app/src/layout/Model.ts) |
| 前端消息拦截 | `processMessage` | [app/src/util/processMessage.ts](app/src/util/processMessage.ts) |
| 前端 Toast 渲染 | `showMessage` / `hideMessage` | [app/src/dialog/message.ts](app/src/dialog/message.ts) |
| 前端进度渲染 | `progressLoading` / `progressStatus` | [app/src/dialog/processSystem.ts](app/src/dialog/processSystem.ts) |
| 前端 HTTP 封装 | `fetchPost` / `fetchSyncPost` | [app/src/util/fetch.ts](app/src/util/fetch.ts) |
| 前端导入调用 | `importSY` / `importStdMd` / `importZipMd` | [app/src/menus/navigation.ts](app/src/menus/navigation.ts) |
| 移动端消息分发 | `onMessage` | [app/src/mobile/util/onMessage.ts](app/src/mobile/util/onMessage.ts) |
| 导出前端入口（5 分支） | `saveExport` | [app/src/protyle/export/index.ts:L33-L115](app/src/protyle/export/index.ts#L33-L115) |
| PDF 预览窗口构建 | `renderPDF` | [app/src/protyle/export/index.ts:L138-L688](app/src/protyle/export/index.ts#L138-L688) |
| 桌面端路径选择 | `getExportPath` | [app/src/protyle/export/index.ts:L690-L749](app/src/protyle/export/index.ts#L690-L749) |
| HTML 完整渲染包装 | `onExport` | [app/src/protyle/export/index.ts:L752-L845](app/src/protyle/export/index.ts#L752-L845) |
| 导出成功提示（显示在文件夹） | `afterExport` | [app/src/protyle/export/util.ts:L16-L26](app/src/protyle/export/util.ts#L16-L26) |
| 图片导出流程 | `exportImage` | [app/src/protyle/export/util.ts:L28-L188](app/src/protyle/export/util.ts#L28-L188) |
| 4 平台文件打开分发 | `openByMobile` | [app/src/protyle/util/compatibility.ts:L68-L99](app/src/protyle/util/compatibility.ts#L68-L99) |
| 块菜单导出子菜单 | `exportMd` | [app/src/menus/commonMenuItem.ts:L528-L834](app/src/menus/commonMenuItem.ts#L528-L834) |
