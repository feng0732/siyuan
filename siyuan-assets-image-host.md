# SiYuan 资源文件管理与图床功能代码分析

## 目录

1. [整体架构概述](#整体架构概述)
2. [附件引用机制](#附件引用机制)
3. [上传设置与流程](#上传设置与流程)
4. [路径映射策略](#路径映射策略)
5. [清理策略与一致性保证](#清理策略与一致性保证)
6. [失败回滚与事务机制](#失败回滚与事务机制)
7. [重复资源处理](#重复资源处理)
8. [外部服务异常处理](#外部服务异常处理)
9. [数据迁移与导入导出](#数据迁移与导入导出)
10. [关键代码文件索引](#关键代码文件索引)

---

## 整体架构概述

SiYuan 采用前后端分离的资源管理架构，核心由以下层级组成：

```
前端 (TypeScript/React)                          内核 (Go)
┌────────────────────────────┐              ┌────────────────────────────┐
│  protyle/upload/index.ts   │──HTTP/API──▶│  api/asset.go               │
│  config/image.ts           │              │  model/upload.go            │
│  asset/renderAssets.ts     │              │  model/assets.go            │
└────────────────────────────┘              │  model/cloud_service.go     │
                                            │  cache/asset.go             │
                                            │  sql/asset.go               │
                                            │  util/cloud.go              │
                                            └────────────────────────────┘
```

资源文件存储采用 **三级目录结构**：
1. 文档同级 `assets/`（优先级最高）
2. 笔记本根目录 `assets/`（次优先级）
3. 全局 `data/assets/`（兜底目录）

相关实现参见 [getAssetsDir](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/upload.go#L355-L364)。

---

## 附件引用机制

### 引用类型覆盖范围

SiYuan 的附件引用检测覆盖多种 AST 节点类型，核心实现在 [getAssetsLinkDests](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L1362-L1475)：

| 节点类型 | 说明 | 提取方式 |
|---------|------|---------|
| `NodeLinkDest` | 标准 Markdown 链接/图片 | `n.Tokens` |
| TextMark `a` | 超链接文本标记 | `n.TextMarkAHref` |
| TextMark `file-annotation-ref` | PDF 文件标注引用 | `n.TextMarkFileAnnotationRefID` |
| `NodeHTMLBlock` / `NodeInlineHTML` | 嵌入式 HTML | `treenode.GetNodeSrcTokens` |
| `NodeIFrame` | iframe 嵌入 | `treenode.GetNodeSrcTokens` |
| `NodeWidget` | 小组件 | `custom-data-assets` 属性 |
| `NodeAudio` / `NodeVideo` | 音视频 | `treenode.GetNodeSrcTokens` |
| `NodeAttributeView` | 数据库视图 | `av.KeyTypeMAsset` / `av.KeyTypeURL` |
| 块属性 `custom-data-assets*` | 自定义块级资源属性 | `n.KramdownIAL` 遍历 |

### 引用路径判定

通过 [IsAssetLinkDest](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/path.go#L293-L299) 判定是否为本地资源引用：

```go
func IsAssetLinkDest(dest []byte, includeServePath bool) bool {
    return bytes.HasPrefix(dest, []byte("assets/")) ||
        (includeServePath && (bytes.HasPrefix(dest, []byte("emojis/")) ||
            bytes.HasPrefix(dest, []byte("plugins/")) ||
            bytes.HasPrefix(dest, []byte("public/")) ||
            bytes.HasPrefix(dest, []byte("widgets/"))))
}
```

### 特殊引用处理

- **题头图（Title Image）**：通过 `treenode.GetDocTitleImgPath` 单独提取，确保清理时不误删
- **查询嵌入块（Query Embed）**：递归执行 SQL 并收集嵌入块内资源引用，参见 [getQueryEmbedNodesAssetsLinkDests](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L1331-L1360)
- **PDF 标注文件（.sya）**：与 PDF 主文件绑定，引用时自动计入 `.sya` 附属文件
- **macOS .rtfd 格式**：自动追加 `/` 标识为文件夹类型

---

## 上传设置与流程

### 前端上传流程

前端核心实现在 [upload/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/app/src/protyle/upload/index.ts)，分三个阶段：

**阶段 1：文件校验（validateFile）**
- 文件名为空检查
- 单文件大小限制校验（`protyle.options.upload.max`）
- 文件类型白名单校验（支持扩展名和 MIME 大类匹配）
- 大文件（> `Constants.SIZE_UPLOAD_TIP_SIZE`）二次确认提示

**阶段 2：上传执行（uploadFiles）**
- 文件夹自动通过 `uploadLocalFiles` 走本地文件插入路径
- 自定义 Hook 支持：`handler` / `file` / `validate` 前置处理
- 支持自定义 `extraData` 和 `X-Upload-Token` 请求头
- 原生 `XMLHttpRequest` 实现，支持进度条回调（`xhr.upload.onprogress`）

**阶段 3：结果处理（genUploadedLabel）**
- 错误文件逐条提示
- 多文件按文件名 **自然升序** 排列插入（支持数字排序）
- 表格/段落上下文智能判断是否独立成块
- 数据库面板（`.av__panel`）中自动写入 mAsset 单元格
- 支持事务级 undo/redo（`transaction` + `undoOperations`）

### 内核上传处理

内核入口在 [Upload](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/upload.go#L131-L353)，处理流程：

```
MultipartForm 解析
    │
    ▼
确定目标 assets 目录（id / assetsDirPath 参数）
    │
    ▼
遍历文件列表：
  ├─ 计算文件哈希（七牛云 ETag 算法）
  ├─ 查询 hash → path 缓存（重复检测）
  │   ├─ 重复且文件名相同 → 直接复用已有路径
  │   └─ 重复但文件名不同 → 强制生成新文件（random_2_ 前缀）
  ├─ skipIfDuplicated 模式（PDF 标注场景）
  │   └─ Glob 模式匹配已有文件避免重复插入
  ├─ 文件名规范化（FilterUploadFileName + AssetName）
  ├─ macOS .rtfd.zip 解压特殊处理
  └─ filelock 写入 + 缓存 hash 映射
    │
    ▼
返回 {errFiles, succMap} + IncSync() 触发同步
```

### 文件名规范化链

参见 [FilterUploadFileName](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/file.go#L225-L247) 和 [AssetName](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/file.go#L182-L197)：

```
原始文件名
  │
  ▼ FilterFileName
  ├─ 替换非法字符：\ / : * ? " ' < > | → _
  ├─ 去除零宽不可见字符
  └─ 去除首尾空格和末尾点号
  │
  ▼ FilterUploadFileName
  ├─ 移除 Markdown 冲突符号：~ [ ] ( ) ! ` & { } = # % $ ;
  └─ TruncateLenFileName：UTF-8 字节长度截断至 189（含扩展）
  │
  ▼ AssetName(name, nodeID)
  └─ 追加 22 位节点 ID：{name}-{YYYYMMDDHHMMSS}-{rand7}.ext
```

**PDF 标注 PNG 文件**有额外的模式保留逻辑（`--P1--270-id.png`），防止截断损坏页码和旋转信息。

---

## 路径映射策略

### 三级资源目录解析

上传时通过 [getAssetsDir](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/upload.go#L355-L364) 确定写入目录，优先级从高到低：
1. `{docDir}/assets`（存在即使用）
2. `{boxDir}/assets`（笔记本级）
3. `data/assets`（全局兜底）

### 相对路径 → 绝对路径解析

读取时通过 [GetAssetAbsPath](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L536-L564) 进行二级回退：

**Step 1：原始路径解析**（[getAssetAbsPath](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L566-L611)）
1. 直接拼接 `data/{relativePath}` 检查存在性
2. 若路径以 `assets/` 开头，遍历所有笔记本，检查是否为 `{notebook}/**/assets/xxx` 的后缀匹配
3. 全程校验 `IsSubPath(WorkspaceDir, p)` 防止路径穿越

**Step 2：URL 反转义重试**
若 Step 1 失败，对路径执行 `url.PathUnescape` 后重试，支持 URL 编码文件名场景。

### 全局资源路径

[GetDataAssetsAbsPath](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/working.go#L491-L502) 支持符号链接跟随（`filepath.EvalSymlinks`），允许用户通过软链将 assets 目录挂载到独立存储。

---

## 清理策略与一致性保证

### 未引用资源检测算法

[UnusedAssets](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L1041-L1215) 执行步骤：

```
Step 1: 构建全量资源池 allAssetAbsPaths()
  ├─ 遍历所有笔记本下的 assets 子目录
  └─ 遍历全局 data/assets 目录
  └─ 过滤：隐藏文件 / .sya 标注 / .tmp 临时文件

Step 2: 扫描所有文档提取引用集合
  ├─ 分页加载（32 页批处理，优化内存）
  ├─ 提取 getAssetsLinkDests + 题头图
  ├─ 剥离 ?query 参数（pdf?page 场景）
  ├─ 区分文件夹链接和文件链接
  └─ PDF 文件自动关联 .sya 标注文件

Step 3: 文件夹链接的前缀排除
  ├─ assets/folder/ 类型链接 → 排除该目录下所有子资源
  └─ assets/sub/file 是 assets/sub 的前缀 → 排除父目录自身

Step 4: 数据库资源引用扫描
  └─ 遍历 storage/av/*.json，检查 bytes.Contains(data, assetPath)

Step 5: 特殊文件排除
  ├─ ocr-texts.json（OCR 索引）
  └─ android-notification-texts.txt（安卓缓存）

Step 6: 差集计算 → 返回未引用列表
```

### 清理执行流程

[RemoveUnusedAssets](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L778-L852)：

```
调用 UnusedAssets 获取待清理列表
    │
    ▼
创建历史目录 history/clean-{timestamp}
    │
    ▼
遍历待清理文件：
  ├─ filelock.Copy → history 目录（先备份！）
  ├─ 计算并收集 hash
  ├─ cache.RemoveAssetHash(hash)
    │
    ▼
sql.BatchRemoveAssetsQueue(hashes) → 批量移除数据库索引
    │
    ▼
遍历待清理文件：
  ├─ HandleAssetsRemoveEvent → 移除缩略图、OCR 索引
  ├─ filelock.RemoveWithoutFatal → 物理删除（失败不中止流程）
  └─ util.RemoveAssetText → 移除 OCR 文本缓存
    │
    ▼
IncSync() → 触发云同步
indexHistoryDir() → 历史目录可检索
cache.LoadAssets() → 刷新缓存
```

### 缺失资源检测

[MissingAssets](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L1217-L1299) 反向扫描：
- 收集文档内所有 `assets/xxx` 引用
- 检查文件系统中是否存在
- 忽略 `.` 开头的隐藏资源和 PDF 标注节点（`.pdf/{id}` 路径）

---

## 失败回滚与事务机制

### 资源操作的历史保障

SiYuan 对资源文件操作采用 **先备份后操作** 的防御性模式：

| 操作类型 | 历史目录类型 | 触发点 |
|---------|-------------|--------|
| 清理未引用 | `HistoryOpClean` | `RemoveUnusedAsset[s]` |
| 文档内容变更 | `HistoryOpReplace` | 重命名资源引用时批量替换 |

历史保留由配置项 `Editor.HistoryRetentionDays`（默认 30 天）控制，`ClearOutdatedHistoryDirJob` 每 24 小时执行清理。

### 事务回滚（Transaction）

文档级内容操作通过 [performTx](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/transaction.go#L148) 执行：

```go
flushTx(tx *Transaction) {
    if txErr := performTx(tx); nil != txErr {
        switch txErr.code {
        case TxErrCodeBlockNotFound:   // 块不存在，推送提示
        case TxErrCodeDataIsSyncing:   // 同步中冲突
        case TxErrCodeWriteTree:       // 写入文件失败
        case TxErrHandleAttributeView: // 数据库操作失败
        case TxErrCodePushMsg:         // 自定义消息
        default:                       // 致命错误 → Fatal 退出
        }
    }
}
```

**注意**：当前事务机制主要针对文档树（.sy 文件），**资源文件的上传/删除本身不参与两阶段提交**。若上传中途失败，已写入磁盘的文件将成为"孤儿"，需通过 `UnusedAssets` 事后清理。

---

## 重复资源处理

### 基于内容哈希的去重

采用 **七牛云 ETag 算法**（参见 [etag.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/etag.go)）：

```
文件大小 ≤ 4MB：
  etag = base64url(0x16 + SHA1(file))

文件大小 > 4MB：
  blocks = ceil(size / 4MB)
  sha1BlockBuf = concat(SHA1(block_0), SHA1(block_1), ..., SHA1(block_n-1))
  etag = base64url(0x96 + SHA1(sha1BlockBuf))
```

哈希缓存采用双层 Map（参见 [cache/asset.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/cache/asset.go)）：
- `assetHashCache[hash] → {Hash, Path}`：快速查重
- `assetPathHashCache[path] → {Hash, Path}`：反向查询

### 查重与强制保留策略

参见 [Upload](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/upload.go#L220-L226)：

```go
existAssetPath := GetAssetPathByHash(hash)
if "" != existAssetPath {
    originalName := util.RemoveID(filepath.Base(existAssetPath))
    if strings.ToLower(fName) != strings.ToLower(originalName) {
        // 哈希相同但文件名不同 → 用户可能想保留副本，强制新建
        hash = "random_2_" + gulu.Rand.String(12)
    }
}
```

**空文件（size ≤ 1）** 使用 `random_1_` 前缀跳过哈希匹配。

### PDF 标注图片场景的特殊查重（skipIfDuplicated）

通过文件名后缀匹配而非内容哈希：
- 标准模式：`{dir}/{namePrefix}*{ext}` Glob
- 文件名过长截断模式：`{dir}*{lastID}{ext}` 利用尾部 22 位 ID 匹配

---

## 外部服务异常处理

### 云端图床上传

[uploadAssets2Cloud](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L662-L776) 核心逻辑：

```
参数检查 → IsSubscriber() 订阅者校验
    │
    ▼
GetAssetAbsPath 批量解析路径 → 去重
    │
    ▼
LoadUploadToken() → 有效期 3600s，失败直接返回
    │
    ▼
单文件校验：
  ├─ size > 3MB（非订阅）/ 10MB（订阅）→ 跳过 + 提示（最多 3 条）
  ├─ Stat 失败 → 立即中止（返回已上传数量）
    │
    ▼
逐文件上传 HTTP：
  ├─ endpoint: {cloudServer}/apis/siyuan/upload
  ├─ auth: Cookie symphony={uploadToken}
  ├─ headers: meta-type（5=图床, 4=社区发帖）, biz-type
  ├─ 失败返回：
  │   ├─ 401 → 登录失效（ErrCode 31）
  │   ├─ 网络错误 → ErrFailedToConnectCloudServer
  │   └─ 业务错误 → 包装服务端消息
  └─ 成功 → 记录 completedUploadAssets
    │
    ▼
返回成功计数 count
```

**异常行为**：上传为顺序执行，中途任意文件失败会 **中止整个批量并返回当前 count**，不执行已上传文件的回滚。已成功上传的云端文件不会被清理。

### 网络资源本地化

[netAssets2LocalAssets0](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L254-L445)：

```
重试机制：SetRetryCount(1) + 3s 间隔（总计 2 次尝试）
TLS 指纹：自定义 httpclient，绕过部分站点反爬
Referer：支持传入 originalURL，提升下载成功率

下载失败处理策略：
  ├─ 403/401 → forbiddenCount++，全部完成后汇总提示
  ├─ Content-Type: text/html → 静默跳过（判定为网页而非资源）
  ├─ 非 200 → 记录日志继续下一个
  ├─ ContentLength > 96MB → 跳过 + 日志告警
  └─ 扩展名校验链（优先级从高到低）：
       1. URL 路径提取
       2. mimetype.Detect(data) 内容嗅探
       3. SVG 标签特征匹配
       4. HTTP Content-Type → mime.ExtensionsByType
```

### 云端区域切换

通过 `CurrentCloudRegion` 支持双区域：
- `0` → 中国大陆（阿里云 + 七牛云）
- `1` → 北美（Cloudflare + 七牛云）

所有服务端点在 [cloud.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/cloud.go#L67-L84) 统一定义，包括：同步服务、图床服务、社区图床、账号服务。

---

## 数据迁移与导入导出

### 资源重命名全量替换

[RenameAsset](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go#L898-L1033) 执行全局一致性更新：

```
Step 1: 文件系统层
  ├─ filelock.Copy(old → new) 先复制后删旧
  ├─ 同步复制 {old}.sya → {new}.sya（PDF 标注）
  └─ OCR 文本：util.SetAssetText(newPath, GetAssetText(oldPath))

Step 2: 所有文档内容遍历
  ├─ 分页（32）遍历所有笔记本下的 .sy 文件
  ├─ bytes 级全文替换：bytes.Replace(data, oldName, newName, -1)
  ├─ 重新解析 tree → 生成历史版本 → 写入 SQL 队列
  └─ cache.RemoveTreeData 失效缓存

Step 3: 数据库（Attribute View）JSON 替换
  └─ storage/av/*.json 全量 bytes.ReplaceAll(oldPath, newPath)

Step 4: IncSync() → 触发云同步
```

### 导入过程中的资源处理

[ImportSY](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/import.go#L110)：
- 解压 .sy.zip 后自动跟随内部 assets 目录结构
- 块 ID 重生成（保留时间戳前缀 14 位，随机后缀 7 位重算）
- 资源文件路径由相对位置决定，遵循三级目录约定

### 导出过程中的资源处理

[export.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/export.go) 中多种格式的资源策略：

| 导出格式 | 资源处理方式 | 关键实现 |
|---------|-------------|---------|
| **发布到链滴社区** | 调用 `uploadAssets2Cloud(bizTypeExport2Liandi)` 上传社区图床，替换链接前缀为 `{forumAssetsServer}/{yyyymm}/siyuan/{userId}/` | L338, L389 |
| **标准 Markdown** | 订阅者模式下链接前缀为 `{cloudAssetsServer}/{userId}/`；否则保持本地相对路径 | L1686-L1688 |
| **PDF / DOCX** | `removeAssets=false` 时复制资源文件到导出目录并嵌入；`removeAssets=true` 时保持链接不复制 | L755, L1245 |
| **导出压缩包** | 遍历所有引用 → `copiedAssets` HashSet 去重 → 按相对路径复制到导出目录，AV JSON 同步替换 | L1957-L2008 |

### 笔记本级资源迁移辅助

`copyBoxAssetsToDataAssets` / `copyDocAssetsToDataAssets` 提供从笔记本/文档级 assets 向全局 assets 的批量复制，用于合并工作空间或导出前预处理。

---

## 关键代码文件索引

| 文件 | 职责 |
|------|------|
| [kernel/model/assets.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets.go) | 资源管理核心：清理、重命名、引用提取、云端上传、网络转本地 |
| [kernel/model/upload.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/upload.go) | 上传主流程、本地文件插入、三级目录选择 |
| [kernel/api/asset.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/api/asset.go) | HTTP API 层：unused/missing/remove/rename/ocr/上传云端 |
| [kernel/cache/asset.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/cache/asset.go) | 哈希-路径双层缓存、全局资产搜索缓存 |
| [kernel/sql/asset.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/sql/asset.go) | 资产数据库表 CRUD、按哈希查询 |
| [kernel/model/cloud_service.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/cloud_service.go#L177-L211) | 上传 Token 获取与缓存（1h TTL） |
| [kernel/util/cloud.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/cloud.go) | 双区域云端端点配置 |
| [kernel/util/etag.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/etag.go) | 七牛云 ETag 哈希算法实现 |
| [kernel/util/file.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/file.go#L182-L319) | 文件名规范化、长度截断、非法字符过滤 |
| [kernel/util/path.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/util/path.go#L293-L370) | 资源链接判定、MIME 类型、工作空间路径校验 |
| [kernel/model/assets_watcher.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/assets_watcher.go) | 非 macOS 平台文件系统监视器（fsnotify + 100ms 防抖） |
| [kernel/model/history.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/history.go) | 历史生成定时任务、保留策略 |
| [kernel/job/cron.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/job/cron.go) | 所有周期任务注册：历史清理、OCR、SQL 刷盘等 |
| [kernel/model/transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/kernel/model/transaction.go) | 文档级事务执行与错误分类处理 |
| [app/src/protyle/upload/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/app/src/protyle/upload/index.ts) | 前端上传：校验、进度、结果渲染、事务回滚 |
| [app/src/config/image.ts](file:///d:/fz/0601/solo-dogfeeding/code/291-siyuan/app/src/config/image.ts) | 前端资源管理面板：未引用 / 缺失 / AV 清理 UI |
