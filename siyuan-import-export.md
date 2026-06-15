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
| 导入业务 | [import.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go) | `.sy.zip`、`data.zip`、Markdown 文件夹/文件/ZIP 导入，ID 重映射、引用修正 |
| 导出业务 | [export.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go) | Markdown、HTML、PDF、DOCX、`.sy.zip`、多种 Pandoc 格式导出 |
| 导出合并 | [export_merge.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export_merge.go) | 子文档合并为单一文档（用于 Word/PDF 导出） |
| 资源管理 | [assets.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/assets.go) | 资源文件（图片、附件）处理、OCR、缩略图 |
| Pandoc 集成 | [pandoc.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/util/pandoc.go) | Pandoc 二进制初始化、路径校验、命令行调用 |
| 导入 API | [import.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/api/import.go) | 4 个导入端点：`importSY`、`importData`、`importStdMd`、`importZipMd` |
| 导出 API | [export.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/api/export.go) | 30+ 个导出端点，覆盖所有导出格式 |
| 路由注册 | [router.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/api/router.go) | 统一鉴权（`CheckAuth`、`CheckAdminRole`、`CheckReadonly`） |
| 导出配置 | [export.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/conf/export.go) | 20+ 项导出参数配置（块引模式、水印、标签标记等） |
| 树形节点 | [tree.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/treenode/tree.go) | AST 节点操作、文档规范版本（Spec）校验 |
| API 结果 | [result.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/util/result.go) | 统一结果结构 `Result{Code, Msg, Data}` |

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

**关键实现位置：** [parseStdMd](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L1210-L1222)

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

**安全校验：** [IsValidPandocBin](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/util/pandoc.go#L228-L302) 实现了严格的二进制校验：
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

**关键实现：** [ImportFromLocalPath](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L771-L1208)

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

**关键实现：** [exportTree](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L2423-L2627)

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

**关键实现：** [processBase64Img](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L1250-L1328)

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

**定义位置：** [result.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/util/result.go#L35-L46)

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

---

## 6. 格式差异处理

### 6.1 块引用模式差异

SiYuan 的核心特性——**块引用**在不同格式中映射方式完全不同，由 `BlockRefMode` 配置控制：

| Mode | 内部表示 | Markdown 导出 | HTML 导出 |
|------|---------|--------------|----------|
| 2 | 锚文本块链 | `[text](siyuan://blocks/ID)` | 超链接到 siyuan 协议 |
| 3 | 仅锚文本 | 纯文本 | 纯文本 |
| 4 | 脚注 + 锚点哈希（默认） | `text[^1]` + 文末脚注定义 | 内部锚点超链接 |

**实现细节：** [exportTree 中的块引用处理](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L2504-L2554)

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

**实现位置：** [mergeSubDocs](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export_merge.go#L27-L70)

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

**实现位置：** [tree.go CheckSpec](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/treenode/tree.go#L135-L150)

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
| .sy.zip 导入 | `ImportSY` | [import.go:L110-L702](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L110-L702) |
| 数据全量还原 | `ImportData` | [import.go:L704-L769](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L704-L769) |
| Markdown 本地导入 | `ImportFromLocalPath` | [import.go:L771-L1208](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L771-L1208) |
| Markdown 标准解析 | `parseStdMd` | [import.go:L1210-L1222](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L1210-L1222) |
| Base64 图片解码 | `processBase64Img` | [import.go:L1250-L1328](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L1250-L1328) |
| HTML→AST 转换 | `HTML2Tree` | [import.go:L59-L108](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L59-L108) |
| Markdown 超链接转块引 | `convertMdHyperlinks2WikiLinks` | [import.go:L1542-L1590](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/import.go#L1542-L1590) |
| 核心导出变换 | `exportTree` | [export.go:L2423-L2627](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L2423-L2627) |
| Markdown 内容导出 | `exportMarkdownContent0` | [export.go:L2282-L2421](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L2282-L2421) |
| Pandoc 格式批量导出 | `ExportPandocConvertZip` | [export.go:L1724-L1749](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L1724-L1749) |
| .sy.zip 打包 | `exportSYZip` | [export.go:L1853-L2021](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L1853-L2021) |
| HTML 导出 | `ExportHTML` | [export.go:L1017-L1187](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L1017-L1187) |
| DOCX 导出 | `ExportDocx` | [export.go:L755-L840](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L755-L840) |
| PDF 后处理 | `ProcessPDF` | [export.go:L1245-L1301](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export.go#L1245-L1301) |
| 子文档合并 | `mergeSubDocs` | [export_merge.go:L27-L70](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/model/export_merge.go#L27-L70) |
| Pandoc 初始化 | `InitPandoc` | [util/pandoc.go:L109-L206](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/util/pandoc.go#L109-L206) |
| Pandoc 二进制校验 | `IsValidPandocBin` | [util/pandoc.go:L228-L302](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/util/pandoc.go#L228-L302) |
| 文档 Spec 版本校验 | `CheckSpec` | [treenode/tree.go:L139-L150](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/treenode/tree.go#L139-L150) |
| 导出配置定义 | `Export` 结构体 | [conf/export.go:L19-L48](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/conf/export.go#L19-L48) |
| API 路由注册 | `ServeAPI` | [api/router.go:L25-L486](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/api/router.go#L25-L486) |
| 导入 API 端点 | 4 个 handler | [api/import.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/api/import.go) |
| 导出 API 端点 | 30+ 个 handler | [api/export.go](file:///d:/fz/0601/solo-dogfeeding/code/298-siyuan/kernel/api/export.go) |
