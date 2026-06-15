# SiYuan 资源文件管理与图床功能代码分析

## 目录

1. [整体架构概述](#整体架构概述)
2. [附件引用机制](#附件引用机制)
3. [上传配置项详解](#上传配置项详解)
4. [上传设置与流程](#上传设置与流程)
5. [三种云端资源处理场景的严格区分](#三种云端资源处理场景的严格区分)
6. [路径映射策略](#路径映射策略)
7. [权限校验机制](#权限校验机制)
8. [清理策略与一致性保证](#清理策略与一致性保证)
9. [消息提示机制](#消息提示机制)
10. [失败回滚与事务机制](#失败回滚与事务机制)
11. [重复资源处理](#重复资源处理)
12. [外部服务异常处理](#外部服务异常处理)
13. [数据迁移与导入导出](#数据迁移与导入导出)
14. [关键代码文件索引](#关键代码文件索引)

---

## 整体架构概述

SiYuan 采用前后端分离的资源管理架构，核心由以下层级组成：

```
前端 (TypeScript/React)                          内核 (Go)
┌────────────────────────────┐              ┌────────────────────────────┐
│  protyle/upload/index.ts   │──HTTP/API──▶│  api/asset.go               │
│  config/image.ts           │              │  model/upload.go            │
│  util/fetch.ts             │              │  model/assets.go            │
│  util/needSubscribe.ts     │              │  model/cloud_service.go     │
│  asset/renderAssets.ts     │              │  cache/asset.go             │
└────────────────────────────┘              │  sql/asset.go               │
                                            │  util/cloud.go              │
                                            │  model/session.go           │
                                            └────────────────────────────┘
```

资源文件存储采用 **三级目录结构**：
1. 文档同级 `assets/`（优先级最高）
2. 笔记本根目录 `assets/`（次优先级）
3. 全局 `data/assets/`（兜底目录）

相关实现参见 `kernel/model/upload.go#L355-L364`。

---

## 附件引用机制

### 引用类型覆盖范围

SiYuan 的附件引用检测覆盖多种 AST 节点类型，核心实现在 `kernel/model/assets.go#L1362-L1475`：

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

通过 `kernel/util/path.go#L293-L299` 的 `IsAssetLinkDest` 判定是否为本地资源引用：

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
- **查询嵌入块（Query Embed）**：递归执行 SQL 并收集嵌入块内资源引用，参见 `kernel/model/assets.go#L1331-L1360`
- **PDF 标注文件（.sya）**：与 PDF 主文件绑定，引用时自动计入 `.sya` 附属文件
- **macOS .rtfd 格式**：自动追加 `/` 标识为文件夹类型

---

## 上传配置项详解

### 前端 Protyle 上传配置（IUpload 接口）

类型定义见 `app/src/types/protyle.d.ts#L314-L360`，默认值在 `app/src/protyle/util/Options.ts#L94-L102`：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `url` | `string` | `Constants.UPLOAD_ADDRESS`（即 `/upload`） | 上传目标 HTTP 地址 |
| `max` | `number` | `1024*1024*1024*16`（16 GB） | 单文件最大体积，单位 Byte |
| `linkToImgUrl` | `string` | `""` | 剪切板中仅含图片 URL 时，使用该 URL 执行"网络图床重新上传" |
| `token` | `string` | 未配置 | 自定义鉴权头，以 `X-Upload-Token` 发送 |
| `accept` | `string` | 未配置 | `<input accept>` 过滤，如 `"image/*,.pdf"` |
| `withCredentials` | `boolean` | `false` | 跨域请求是否携带 Cookie |
| `headers` | `IObject` | 未配置 | 固定请求头 |
| `extraData` | `{[key]:string\|Blob}` | `{}` | 随文件一起提交的额外 Form 字段 |
| `fieldName` | `string` | `"file[]"` | 上传文件表单字段名 |
| `filename(name)` | `(name)=>string` | 正则移除 `\/:*?"'<>|[]()~!`&{}=#%$` 字符 | 文件名安全处理函数 |
| `validate(files)` | `(File[])=>string\|boolean` | 未配置 | 自定义前置校验，返回错误字符串或 true |
| `handler(files)` | `(File[])=>string\|null` | 未配置 | 完全接管上传，返回 null 视为成功，字符串为错误信息 |
| `format(files,resp)` | `(File[],string)=>string` | 未配置 | 将服务端响应转换为内置 `{errFiles, succMap}` 结构 |
| `file(files)` | `(File[])=>File[]` | 未配置 | 对 FileList 进行预处理（压缩/加解密等） |
| `setHeaders()` | `()=>IObject` | 未配置 | 每次上传前动态计算请求头 |
| `success(editor,msg)` | 回调 | 未配置 | 上传成功回调 |
| `error(msg)` | 回调 | 未配置 | 上传失败回调 |
| `linkToImgCallback(resp)` | 回调 | 未配置 | 图片链接转本地后回调 |

### 后端相关配置

后端没有独立的上传 size 配置，但以下间接配置会影响资源管理行为：

| 配置项 | 位置 | 说明 |
|--------|------|------|
| `Editor.HistoryRetentionDays` | `kernel/conf/editor.go` | 历史保留天数，影响已删除资源可恢复的时间窗口（默认 30 天） |
| `Api.Token` | `kernel/conf/api.go` | API Token，第三方客户端上传时使用 `Authorization: Token xxx` 头 |
| `AccessAuthCode` | `kernel/model/session.go` | 访问授权码，非 `127.0.0.1` 访问时强制校验 |
| `CurrentCloudRegion` | `kernel/util/cloud.go` | 云端区域：0=中国大陆、1=北美 |

---

## 上传设置与流程

### 前端上传流程

前端核心实现在 `app/src/protyle/upload/index.ts`，分三个阶段：

**阶段 1：文件校验（validateFile）**
- 文件名为空检查
- 单文件大小限制校验（`protyle.options.upload.max`），超限后以 `{size}M` 为单位提示
- 文件类型白名单校验（支持扩展名（`.png`）和 MIME 大类（`image/*`）两种匹配方式）
- 大文件（> `Constants.SIZE_UPLOAD_TIP_SIZE`）二次确认提示

**阶段 2：上传执行（uploadFiles）**
- 文件夹自动通过 `uploadLocalFiles` 走本地文件插入路径（不经过远端 HTTP）
- 自定义 Hook 顺序：`handler` → `file` → `validate`，任一返回非空字符串即中止
- 支持自定义 `extraData`（FormData append）和 `X-Upload-Token` 请求头
- 原生 `XMLHttpRequest` 实现，支持进度条回调（`xhr.upload.onprogress` → `protyle.upload.element` 进度条）
- URL 为空或 `protyle.upload` 未初始化时提示 `"please config: options.upload.url"`

**阶段 3：结果处理（genUploadedLabel）**
- `errFiles` 逐条生成 `<li>` 项通过 `showMessage` 以红色 HTML 列表提示
- 多文件按文件名 **自然升序** 排列插入（通过 `String.localeCompare("zh")` 支持中文数字排序）
- 表格/段落上下文智能判断是否独立成块
- 数据库面板（`.av__panel`）中自动写入当前聚焦单元格所在行的 mAsset 列
- 支持事务级 undo/redo（组装 `doOperations` + `undoOperations` 调用 `protyle.transaction.transaction`）

### 内核上传处理

内核入口在 `kernel/model/upload.go#L131-L353`，处理流程：

```
MultipartForm 解析（c.Request.MultipartForm）
    │
    ▼
确定目标 assets 目录（id / assetsDirPath 参数）
    │
    ▼
遍历文件列表：
  ├─ 计算文件哈希（七牛云 ETag 算法）
  ├─ 查询 hash → path 缓存（重复检测）
  │   ├─ 重复且文件名相同 → 直接复用已有路径，跳过写入
  │   └─ 重复但文件名不同 → 强制生成新文件（random_2_ 前缀）
  ├─ skipIfDuplicated 模式（PDF 标注场景）
  │   └─ Glob 模式匹配已有文件避免重复插入
  ├─ 文件名规范化（FilterUploadFileName + AssetName）
  ├─ macOS .rtfd.zip 解压特殊处理
  └─ filelock 写入 + 缓存 hash 映射 + sql.PutAssetQueue
    │
    ▼
返回 {errFiles, succMap} + IncSync() 触发云同步
```

上传入口 HTTP 路由定义在 `kernel/server/serve.go#L585`：`POST /upload`，以及 `kernel/api/router.go#L313` 的 JSON API 形式 `POST /api/asset/upload`。

### 文件名规范化链

参见 `kernel/util/file.go#L225-L247` 和 `kernel/util/file.go#L182-L197`：

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

## 三种云端资源处理场景的严格区分

SiYuan 涉及"云端"的资源处理存在三种**完全不同**的语义，代码证据表明它们的行为、触发方式和对正文的影响均严格区分，不可混淆：

| 维度 | 场景一：云端批量上传 | 场景二：语雀复制（标准 Markdown 渲染） | 场景三：发布到链滴社区 |
|------|---------------------|--------------------------------------|----------------------|
| **核心语义** | 把本地资源推到 SiYuan 云端做备份/CDN | 复制为 Markdown 时给链接加云端前缀（含空格→下划线） | 先上传社区图床再发帖 |
| **触发函数** | `UploadAssets2Cloud` / `UploadAssets2CloudByAssetsPaths` | `ExportStdMarkdown` | `Export2Liandi` |
| **内部调用** | `uploadAssets2Cloud(assets, bizTypeUploadAssets, ignorePushMsg)` | `exportMarkdownContent0(tree, cloudAssetsBase, …)` | 先 `uploadAssets2Cloud(assets, bizTypeExport2Liandi, false)`，再 `exportMarkdownContent0(tree, forumAssetsBase, assetsDestSpace2Underscore=true, …)` |
| **bizType 参数** | `"upload-assets"` | 不参与上传 | `"export-liandi"` |
| **meta-type** | `5`（SiYuan 内部） | N/A | `4`（社区 Client） |
| **上传目标** | `{cloudServer}/apis/siyuan/upload` | 不上传 | `{cloudServer}/apis/siyuan/upload`，最终链接前缀 `{forumAssetsServer}/{yyyymm}/siyuan/{userId}/` |
| **内存 tree 修改** | 只读遍历，不改 tree | `exportTree` 直接修改传入 tree（`ret=tree` 无克隆） | 同左，且 `assetsDestSpace2Underscore` 会把 tree 中链接的空格替换为下划线 |
| **是否回写正文（.sy 文件）** | **完全不回写**，保留本地 `assets/` 链接 | **完全不回写**，tree 是一次性副本 | **仅回写 `custom-liandi-articleid` 属性**，资源链接修改只在内存 |
| **返回值** | `count`（成功上传个数） | `string`（渲染后的 Markdown 文本） | `error`（社区发帖是否成功） |

### 场景一：云端批量上传（上传到云端图床菜单）

**触发入口（2 个）：**

| 入口 | 前端位置 | API | 参数 |
|------|---------|-----|------|
| 文档面包屑菜单「上传到云端图床」 | `app/src/protyle/breadcrumb/index.ts#L363-L374` | `POST /api/asset/uploadCloud` | `{id: protyle.block.id}` |
| 按路径直接指定 | 供插件/脚本使用 | `POST /api/asset/uploadCloudByAssetsPaths` | `{paths: [string,...], ignorePushMsg?: bool}` |

**完整调用链：**

```
点击菜单
  └─ needSubscribe() 本地判断订阅状态
       ├─ 非订阅 → showMessage 拦截中止
       └─ 订阅者 → confirmDialog 二次确认
            └─ fetchPost("/api/asset/uploadCloud", {id})
                 └─ Gin 中间件链：CheckAuth → CheckAdminRole → CheckReadonly
                      └─ kernel/api/asset.go: uploadCloud(c)
                           ├─ 参数：id 必填，ignorePushMsg 可选
                           └─ UploadAssets2Cloud(id, ignorePushMsg)
                                ├─ IsSubscriber() 再次校验（后端防御）
                                ├─ LoadTreeByBlockID(id) 加载文档
                                ├─ 标题块额外包含 HeadingChildren
                                ├─ getAssetsLinkDests + getQueryEmbedNodesAssetsLinkDests
                                │    提取文档内 assets/ 路径
                                ├─ gulu.Str.RemoveDuplicatedElem 去重
                                └─ uploadAssets2Cloud(assets, bizTypeUploadAssets, ignorePushMsg)
```

**`uploadAssets2Cloud` 真实行为（`kernel/model/assets.go#L662-L776`）：**

函数签名只有 3 个参数，**不存在 `needReplaceDocAssets` 参数**：

```go
func uploadAssets2Cloud(assetPaths []string, bizType string, ignorePushMsg bool) (count int, err error)
```

执行步骤：
1. `GetAssetAbsPath` 批量解析绝对路径 + 去重
2. 推送进度消息（Language 27 "正在上传 N 个资源文件…"）
3. `LoadUploadToken()` 获取 1 小时有效期的 symphony Cookie
4. 大小限制：3MB（免费）/ 10MB（订阅者），超限 PushErrMsg 跳过（最多 3 条）
5. 循环 `httpclient.NewCloudFileRequest2m()` 逐文件上传
   - 端点：`{cloudServer}/apis/siyuan/upload?ver=`
   - Cookie：`symphony={uploadToken}`
   - Header：`meta-type`=5、`biz-type`="upload-assets"
6. 失败：401→语言31（未登录）、网络错误→ErrFailedToConnectCloudServer、业务错误→包装消息
7. 完成：仅记录 `completedUploadAssets` 和 `count`
8. **⚠️ 没有任何遍历 .sy 文件 / bytes.Replace / writeTree 的代码**——本地文档正文的 `assets/` 链接完全不变

**结论：** 该场景的作用是把资源做云端备份/发布前的预热，本地仍保留相对链接。后续需要渲染云端 URL 时由导出场景（场景二/三）在内存中加前缀。

---

### 场景二：语雀复制（标准 Markdown 渲染）

**触发入口（唯一 1 个，有代码证据）：**

| 入口 | 前端位置 | API | 参数 |
|------|---------|-----|------|
| 预览窗右上角「复制为 Markdown → 语雀」 | `app/src/protyle/preview/index.ts#L65-L66` 定义按钮，`L280-L290` 点击调用 | `POST /api/lute/copyStdMarkdown` | `{id: protyle.block.id || protyle.options.blockId || protyle.block.parentID, assetsDestSpace2Underscore: true, fillCSSVar: true, adjustHeadingLevel: true}` |

**关键证据：**
- 前端：`app/src/protyle/preview/index.ts#L281` → `fetchPost("/api/lute/copyStdMarkdown", {...})`，调用后 `writeText(response.data)` 写入剪贴板 + `showMessage(pasteToYuque)` 提示
- 路由：`kernel/api/router.go#L176` → `POST /api/lute/copyStdMarkdown`，仅 `CheckAuth` 鉴权（不需要 Admin 角色，只读上下文也可调用）
- Handler：`kernel/api/lute.go#L67` → 唯一调用 `model.ExportStdMarkdown(id, assetsDestSpace2Underscore, fillCSSVar, adjustHeadingLevel, imgTag=false)` 的地方
- **⚠️ 代码证明：** 在 `kernel/` 目录全文搜索 `ExportStdMarkdown`，仅 `kernel/api/lute.go:L67` 这一处调用，没有其他内核逻辑复用，也没有"导出菜单"对应的后端端点调用

**完整调用链：**

```
预览窗点击「复制为语雀」
  └─ fetchPost("/api/lute/copyStdMarkdown", {id, assetsDestSpace2Underscore:true})
       └─ CheckAuth 鉴权
            └─ kernel/api/lute.go: copyStdMarkdown(c)
                 └─ model.ExportStdMarkdown(id, true, true, true, false)
                      ├─ prepareExportTree(bt) → filesys.LoadTree → parseJSON2Tree
                      │    ★ 每次返回新 tree 对象，与缓存/磁盘无引用关联
                      ├─ IsSubscriber() 判定 → 决定 cloudAssetsBase 是否非空
                      └─ exportMarkdownContent0(id, tree, cloudAssetsBase, assetsDestSpace2Underscore=true, ...)
```

**关键代码 `kernel/model/export.go#L1678-L1722`：**

```go
func ExportStdMarkdown(id string, assetsDestSpace2Underscore, fillCSSVar, adjustHeadingLevel, imgTag bool) string {
    bt := treenode.GetBlockTree(id)
    tree := prepareExportTree(bt)
    cloudAssetsBase := ""
    if IsSubscriber() {
        cloudAssetsBase = util.GetCloudAssetsServer() + Conf.GetUser().UserId + "/"
        // 中国大陆示例："https://assets.b3logfile.com/siyuan/{userId}/"
    }
    return exportMarkdownContent0(id, tree, cloudAssetsBase, assetsDestSpace2Underscore, ...)
}
```

**前缀注入 + 空格替换的真实影响边界（`kernel/model/export.go#L2300-L2326`）：**

```go
// 步骤 A：LinkBase 前缀（仅影响 lute 渲染输出，不修改 AST）
if "" != cloudAssetsBase {
    luteEngine.RenderOptions.LinkBase = cloudAssetsBase
}

// 步骤 B：空格→下划线（直接修改内存 tree 的 Tokens 字段）
if assetsDestSpace2Underscore {
    ast.Walk(tree.Root, func(n *ast.Node, entering bool) ast.WalkStatus {
        if ast.NodeLinkDest == n.Type && util.IsAssetLinkDest(n.Tokens, false) {
            n.Tokens = bytes.ReplaceAll(n.Tokens, []byte(" "), []byte("_"))
        } else if n.IsTextMarkType("a") && util.IsAssetLinkDest([]byte(href), false) {
            n.TextMarkAHref = strings.ReplaceAll(href, " ", "_")
        } else if (ast.NodeIFrame == n.Type || ast.NodeAudio == n.Type || ast.NodeVideo == n.Type) {
            setAssetsLinkDest(n, dest, strings.ReplaceAll(dest, " ", "_"))
        }
        return ast.WalkContinue
    })
}
```

⚠️ **边界区分（有代码证据）：**

| 操作 | 是否修改内存 tree | 是否修改磁盘 .sy | 代码位置 |
|------|-------------------|------------------|----------|
| `LinkBase = cloudAssetsBase` | ❌ 不修改 AST，仅渲染时附加前缀 | ❌ | L2300-L2302 |
| `assetsDestSpace2Underscore` 替换 | ✅ 直接修改 `n.Tokens` / `TextMarkAHref` | ❌ 无任何 writeTree | L2303-L2326 |

**结论：** 该场景所有 tree 修改只在内存中，`ExportStdMarkdown` 返回的 string 是已替换空格并加了前缀的 Markdown 文本，用于前端复制到剪贴板；工作空间 `.sy` 文件完全不受影响。

---

### 场景三：发布到链滴社区

**触发入口（唯一 1 个，有代码证据）：**

| 入口 | 前端位置 | API | 参数 |
|------|---------|-----|------|
| 面包屑菜单「分享到链滴」 | `app/src/protyle/breadcrumb/index.ts#L375-L387` | `POST /api/export/export2Liandi` | `{id: protyle.block.parentID}`（整个文档，注意这里是 **parentID**，不是 uploadCloud 用的 id） |

**完整调用链 + 内存/磁盘边界分析（`kernel/model/export.go#L322-L439`）：**

```
export2Liandi(c)
  └─ Export2Liandi(id)
       │
       ├─ ★ 内存 tree A 创建：LoadTreeByBlockID(id) → 新对象
       │    证据：kernel/model/tree.go#L217-L242 → loadTreeByBlockTree
       │    → filesys.LoadTreeWithFix → cache.GetTreeData → LoadTreeByData
       │    → parseJSON2Tree → dataparser.ParseJSON（每次新建 *parse.Tree）
       │
       ├─ IsUserGuide(tree.Box) → 拒绝用户指南文档发布
       │
       ├─ Step 1: 上传到社区图床（只读遍历 tree A）
       │    ├─ getAssetsLinkDests + getQueryEmbedNodesAssetsLinkDests 去重
       │    └─ uploadAssets2Cloud(assets, bizTypeExport2Liandi="export-liandi", ignorePushMsg=false)
       │         └─ 上传时 meta-type=4（Client 标识），不回写正文
       │
       ├─ Step 2: 内存构造社区格式 Markdown（★ 修改 tree A）
       │    └─ exportMarkdownContent0(id, tree, util.GetCloudForumAssetsServer()
       │                + time.Now().Format("2006/01") + "/siyuan/" + Conf.GetUser().UserId + "/",
       │         assetsDestSpace2Underscore=true, ...)
       │         │
       │         ├─ 子调用 1：exportTree(tree, ...) L2429 → ret = tree（无克隆）
       │         │    ├─ resolveEmbedR：修改查询嵌入节点的子节点
       │         │    ├─ blockLink2Ref：修改超链接 TextMark 类型
       │         │    └─ 多处 ast.Walk 直接修改 n.Type / n.Tokens / n.IAL
       │         │
       │         └─ 子调用 2：assetsDestSpace2Underscore=true 空格替换 L2303-L2326
       │              ├─ ast.Walk(tree.Root, ...) 遍历
       │              ├─ NodeLinkDest: n.Tokens = bytes.ReplaceAll(..., " ", "_")
       │              ├─ TextMark a:   n.TextMarkAHref = strings.ReplaceAll(..., " ", "_")
       │              ├─ IFrame/Audio/Video: setAssetsLinkDest(n, dest, strings.ReplaceAll(...))
       │              └─ ★ 代码证明：此处直接修改 tree A 的 AST 节点字段，但
       │                 整个 exportMarkdownContent0 函数体内无任何 writeTree 调用
       │
       ├─ Step 3: 查询是否已发布过（custom-liandi-articleid）
       │    ├─ GET /api/v2/article/update/{id}
       │    └─ 200/404 判定是否存在
       │
       ├─ Step 4: 发帖/更帖
       │    ├─ POST 或 PUT {accountServer}/api/v2/article
       │    └─ Body: {articleTitle, articleTags, articleContent=渲染的Markdown}
       │
       └─ Step 5: ★ 唯一的磁盘写回操作
            └─ 首次发布成功 → 注释 L430：
                 "tree, _ = LoadTreeByBlockID(id) // 这里必须重新加载，因为前面导出时已经修改了树结构"
                 │
                 ├─ ★ 内存 tree B 创建：第二次 LoadTreeByBlockID(id)
                 │    从磁盘/缓存重建全新对象，tree A 的所有修改都不影响 tree B
                 │
                 ├─ tree.Root.SetIALAttr(liandiArticleIdAttrName, articleId)
                 │    仅修改 IAL 属性，不涉及任何资源链接字段
                 │
                 └─ writeTreeUpsertQueue(tree) → 写回磁盘
                      证据：kernel/model/tree.go 的 writeTreeUpsertQueue
                      只把 tree B（包含新 IAL 属性）持久化到 .sy 文件
```

⚠️ **内存 tree 修改 vs 磁盘写回边界（逐条有代码证据）：**

| 操作 | 作用对象 | 是否修改内存 | 是否写回磁盘 | 代码位置 |
|------|---------|-------------|-------------|----------|
| `getAssetsLinkDests` 提取 | tree A | ❌ 只读 | ❌ | L334 |
| `uploadAssets2Cloud` 上传 | tree A | ❌ 只读 | ❌ | L338 |
| `exportTree` AST 转换 | tree A | ✅ 直接修改节点 | ❌ | L2429 |
| `assetsDestSpace2Underscore` 空格替换 | tree A | ✅ 修改 `Tokens` / `TextMarkAHref` | ❌ | L2303-L2326 |
| **第二次 `LoadTreeByBlockID`** | 创建 tree B | ✅ 新对象，含磁盘原始内容 | ❌ | L430 |
| `SetIALAttr("custom-liandi-articleid")` | tree B | ✅ 仅修改 IAL | ❌ | L431 |
| `writeTreeUpsertQueue(tree)` | tree B | ❌ | ✅ 持久化 .sy 文件 | L432 |

**关键证据：**
- `assetsDestSpace2Underscore=true` 对应社区图床对含空格文件的自动重命名逻辑
- `GetCloudForumAssetsServer`（社区）≠ `GetCloudAssetsServer`（订阅者导出）—— 两套独立服务域名
- **⚠️ 代码证明：** L430 的注释明确说明必须重新加载 tree，因为前面的导出流程已经修改了 tree 结构——这是 `exportMarkdownContent0` 会修改传入 tree 对象的直接证据
- **回写只有 `custom-liandi-articleid` 属性这一处**，且是在重新加载的新 tree B 上进行的，tree A 中所有资源链接的空格修改完全被丢弃，不会写回磁盘

---

## 路径映射策略

### 三级资源目录解析

上传时通过 `kernel/model/upload.go#L355-L364` 的 `getAssetsDir` 确定写入目录，优先级从高到低：
1. `{docDir}/assets`（存在即使用）
2. `{boxDir}/assets`（笔记本级）
3. `data/assets`（全局兜底）

### 相对路径 → 绝对路径解析

读取时通过 `kernel/model/assets.go#L536-L564` 的 `GetAssetAbsPath` 进行二级回退：

**Step 1：原始路径解析**（`kernel/model/assets.go#L566-L611`）
1. 直接拼接 `data/{relativePath}` 检查存在性
2. 若路径以 `assets/` 开头，遍历所有笔记本，检查是否为 `{notebook}/**/assets/xxx` 的后缀匹配
3. 全程校验 `IsSubPath(WorkspaceDir, p)` 防止路径穿越

**Step 2：URL 反转义重试**
若 Step 1 失败，对路径执行 `url.PathUnescape` 后重试，支持 URL 编码文件名场景。

### 全局资源路径

`kernel/util/working.go#L491-L502` 的 `GetDataAssetsAbsPath` 支持符号链接跟随（`filepath.EvalSymlinks`），允许用户通过软链将 assets 目录挂载到独立存储。

---

## 权限校验机制

所有资源管理 API 均在 `kernel/api/router.go` 中挂载了 Gin 中间件链，典型组合为：

```
CheckAuth → CheckAdminRole → CheckReadonly → Handler
```

### CheckAuth（kernel/model/session.go#L207-L384）

依次尝试 7 种身份识别方式，命中即放行：

1. **JWT 角色上下文**：`GetGinContextRole(c)` 返回有效角色（Admin/Editor/Reader）时直接通过
2. **Header `Authorization`**：支持 `Token ` / `Bearer ` 前缀，与 `Conf.Api.Token` 相等则赋予 Admin 角色
3. **Query 参数 `token`**：同上
4. **空授权码 + 本地访问**：`AccessAuthCode` 为空且请求来源为 `127.0.0.1` / `localhost`，同时校验 `Origin` / `Host` / `X-Forwarded-Host` 非外部域名（`chrome-extension://` 白名单除外），否则返回 401 提示"为安全起见，请设置访问授权码"
5. **静态资源白名单**：`/appearance/`、`/stage/build/export/`、`/stage/protyle/` 无条件放行
6. **Cookie Session**：`workspaceSession.AccessAuthCode == Conf.AccessAuthCode` 则通过
7. **HTTP BasicAuth**：用户名=`WorkspaceName`、密码=`AccessAuthCode` 时通过

全部失败时：浏览器 GET 请求 302 重定向到 `/check-auth`；其他请求返回 `{"code":-1,"msg":Language(156)}`。

### CheckAdminRole（kernel/model/session.go#L386-L392）

仅允许 Admin 角色通过，否则返回 `403 Forbidden`（无 JSON Body）。用于所有会修改工作空间状态的写操作（上传、删除、重命名、OCR、导出等）。

### CheckReadonly（kernel/model/session.go#L195-L205）

当全局只读开关 `util.ReadOnly` 为 true 或当前上下文为 Reader 角色时拦截：

```go
result.Code = -1
result.Msg = Conf.Language(34)   // "当前工作空间处于只读模式，无法写入"
result.Data = {"closeTimeout": 5000}
c.JSON(200, result)
c.Abort()
```

### IsSubscriber（kernel/model/conf.go#L1015-L1018）

云端图床等付费功能的业务级权限，独立于 HTTP 中间件，在 Handler 内部调用：

```go
func IsSubscriber() bool {
    u := Conf.GetUser()
    return nil != u &&
        (-1 == u.UserSiYuanProExpireTime || 0 < u.UserSiYuanProExpireTime) &&
        0 == u.UserSiYuanSubscriptionStatus
}
```

含义：
- `UserSiYuanProExpireTime = -1`：终身会员
- `UserSiYuanProExpireTime > 0`：在有效期内的时间戳
- `UserSiYuanSubscriptionStatus = 0`：订阅状态正常（非取消/冻结）

前端对应逻辑在 `app/src/util/needSubscribe.ts#L4-L19`，额外处理 iOS 平台提示差异（`_kernel[122]`）。

---

## 清理策略与一致性保证

### 未引用资源检测算法

`kernel/model/assets.go#L1041-L1215` 的 `UnusedAssets` 执行步骤：

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

前端 UI 在 `app/src/config/image.ts`：三个 Tab（未引用资源 / 未引用 AV / 缺失资源），首次渲染时调用 `fetchPost("/api/asset/getUnusedAssets")`。后端返回最多 **512 条**（见 `kernel/api/asset.go#L361-L365`），超出时通过 `PushMsg(Language(251), 5000)` 提示"共 N 个，仅列前 512 个"。

### 清理执行流程

`kernel/model/assets.go#L778-L852` 的 `RemoveUnusedAssets`：

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

API 端点：
- `POST /api/asset/removeUnusedAsset`（单条）：`kernel/api/asset.go#L327-L341`
- `POST /api/asset/removeUnusedAssets`（批量）：`kernel/api/asset.go#L343-L351`

### 缺失资源检测

`kernel/model/assets.go#L1217-L1299` 的 `MissingAssets` 反向扫描：
- 收集文档内所有 `assets/xxx` 引用
- 检查文件系统中是否存在
- 忽略 `.` 开头的隐藏资源和 PDF 标注节点（`.pdf/{id}` 路径）

---

## 消息提示机制

SiYuan 使用 WebSocket 广播 + 前端 `processMessage` 路由的双层消息系统。

### 后端推送（kernel/util/websocket.go#L230-L250）

| 函数 | 用法 | code 值 |
|------|------|---------|
| `PushMsg(msg, timeout)` | 普通提示 | `0` |
| `PushErrMsg(msg, timeout)` | 错误提示 | `-1` |
| `PushUpdateMsg(id, msg, timeout)` | 带 id，前端可局部更新 | `0` |
| `PushTxErr(msg, code, data)` | 事务类错误 | 自定义 |
| `PushStatusBar(msg)` | 仅显示在状态栏 | - |

消息通过 `BroadcastByType("main", "msg", code, msg, {"id","closeTimeout"})` 广播给所有 WebSocket 连接的 main 频道。`timeout` 为 3000~7000ms，会透传给前端作为 toast 自动关闭时间。

### 前端接收（app/src/util/processMessage.ts）

流程：
1. WebSocket `onmessage` 接收 JSON
2. 若 `msg` 非空且 `code` 为数字 → 调用 `processMessage(response)`
3. `processMessage` 根据 `code` 分支：
   - `code === 1` → `window.location.href = "/check-auth"`（鉴权失效跳转）
   - `code < 0` → `showMessage(response.msg, 7000, "error")`
   - `code === 0` → `showMessage(response.msg, response.data.closeTimeout || 7000, "info")`
   - 其他正数 → 按业务需求路由（搜索提示、背景任务等）

### 前端 HTTP fetchPost 的本地提示

`app/src/util/fetch.ts#L88-L91` 在 HTTP 响应收到后同步走 `processMessage(response)`，保证 API 同步错误和 WS 异步提示用同一套 UI 组件。HTTP 401 特殊处理：3 秒后 `window.location.reload()`。

---

## 失败回滚与事务机制

### 资源操作的历史保障

SiYuan 对资源文件操作采用 **先备份后操作** 的防御性模式：

| 操作类型 | 历史目录类型 | 触发点 |
|---------|-------------|--------|
| 清理未引用 | `HistoryOpClean` | `RemoveUnusedAsset[s]` |
| 文档内容变更 | `HistoryOpReplace` | 重命名资源引用时批量替换 |
| 资源回滚 | API `/api/history/rollbackAssetsHistory` | `kernel/api/router.go#L156` |

历史保留由配置项 `Editor.HistoryRetentionDays`（默认 30 天）控制，`ClearOutdatedHistoryDirJob` 每 24 小时执行清理（`kernel/job/cron.go`）。

### 事务回滚（Transaction）

文档级内容操作通过 `kernel/model/transaction.go#L148` 的 `performTx` 执行：

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

采用 **七牛云 ETag 算法**（参见 `kernel/util/etag.go`）：

```
文件大小 ≤ 4MB：
  etag = base64url(0x16 + SHA1(file))

文件大小 > 4MB：
  blocks = ceil(size / 4MB)
  sha1BlockBuf = concat(SHA1(block_0), SHA1(block_1), ..., SHA1(block_n-1))
  etag = base64url(0x96 + SHA1(sha1BlockBuf))
```

哈希缓存采用双层 Map（参见 `kernel/cache/asset.go`）：
- `assetHashCache[hash] → {Hash, Path}`：快速查重
- `assetPathHashCache[path] → {Hash, Path}`：反向查询

### 查重与强制保留策略

参见 `kernel/model/upload.go#L220-L226`：

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

### 云端图床上传（uploadAssets2Cloud）

`kernel/model/assets.go#L662-L776` 已在前文详述，异常处理总结：

| 异常类型 | 处理方式 | 是否中断 |
|---------|---------|---------|
| 资源路径解析失败 | LogWarnf + return | 是，返回错误 |
| 资源找不到 | LogErrorf + continue | 否，跳过该文件 |
| LoadUploadToken 失败 | PushErrMsg + return | 是 |
| Stat 失败（文件消失/权限） | LogErrorf + return count, statErr | 是，保留已成功 count |
| 文件超 3MB / 10MB 限制 | PushErrMsg（最多 3 条） + continue | 否，跳过超限文件 |
| 网络连接错误 | LogErrorf + ErrFailedToConnectCloudServer | 是 |
| HTTP 401 | 返回 Language(31)（未登录） | 是 |
| 业务 code ≠ 0 | LogErrorf + Language(94) 包装 | 是 |

**⚠️ 注意：** 中途失败不执行已上传文件的回滚。云端已接收的文件不会被主动清理，这符合 CDN/对象存储的"最终一致"设计。

### 网络资源本地化

`kernel/model/assets.go#L254-L445` 的 `netAssets2LocalAssets0`：

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

API 端点：
- `POST /api/format/netImg2LocalAssets`（仅图片）
- `POST /api/format/netAssets2LocalAssets`（全资源）

前端菜单位置 `app/src/protyle/breadcrumb/index.ts#L345-L362`。

### 云端区域切换

通过 `CurrentCloudRegion` 支持双区域，`kernel/util/cloud.go#L67-L84` 统一定义：

| 服务 | 中国大陆 (0) | 北美 (1) |
|------|-------------|---------|
| 云端服务/同步 | `siyuan-sync.b3logfile.com` | `siyuan-cloud.liuyun.io` |
| 同步 OSS | `siyuan-data.b3logfile.com/` | `siyuan-data.liuyun.io/` |
| 订阅者导出图床 (GetCloudAssetsServer) | `assets.b3logfile.com/siyuan/` | `assets.liuyun.io/siyuan/` |
| 社区图床 (GetCloudForumAssetsServer) | `b3logfile.com/file/` | `assets.liuyun.io/file/` |
| 账号/发布服务 (GetCloudAccountServer) | `ld246.com` | `liuyun.io` |

---

## 数据迁移与导入导出

### 资源重命名全量替换

`kernel/model/assets.go#L898-L1033` 的 `RenameAsset` 执行全局一致性更新：

```
Step 1: 文件系统层
  ├─ filelock.Copy(old → new) 先复制后删旧
  ├─ 同步复制 {old}.sya → {new}.sya（PDF 标注）
  └─ OCR 文本：util.SetAssetText(newPath, GetAssetText(oldPath))

Step 2: 所有文档内容遍历 ★（与前三个场景对比：这是**唯一**会回写正文资源链接的函数）
  ├─ 证据：kernel/model/assets.go#L952-L972 分页（32）遍历所有笔记本下的 .sy 文件
  ├─ bytes.Contains 检测旧文件名 → bytes.Replace(oldName, newName, -1) 替换
  ├─ filelock.WriteFile → 写回磁盘（L968）
  ├─ 重新解析 tree → generateTreeHistory → UpsertBlockTree + UpsertTreeQueue
  └─ cache.RemoveTreeData 失效缓存

Step 3: 数据库（Attribute View）JSON 替换
  └─ kernel/model/assets.go#L992-L999: storage/av/*.json 全量 bytes.ReplaceAll(oldPath, newPath)

Step 4: IncSync() → 触发云同步
```

⚠️ **与其他场景的写回对比（代码证据）：**

| 函数 | 是否回写 .sy 正文资源链接 | 代码位置 |
|------|--------------------------|----------|
| `UploadAssets2Cloud` | ❌ 不回写，只上传 | `kernel/model/assets.go#L662-L776` 无 writeTree/WriteFile |
| `ExportStdMarkdown` | ❌ 不回写，只返回渲染文本 | `kernel/model/export.go#L1678-L1722` 无 writeTree/WriteFile |
| `Export2Liandi` | ⚠️ 仅回写 IAL 属性 `custom-liandi-articleid`，不修改资源链接 | `kernel/model/export.go#L430-L434` 仅 SetIALAttr |
| `RenameAsset` | ✅ 全文替换资源链接并写回 .sy | `kernel/model/assets.go#L963-L972` bytes.Replace + WriteFile |

API 端点：`POST /api/asset/renameAsset`（`kernel/api/asset.go#L177-L198`）。

### 导入过程中的资源处理

`kernel/model/import.go#L110` 的 `ImportSY`：
- 解压 .sy.zip 后自动跟随内部 assets 目录结构
- 块 ID 重生成（保留时间戳前缀 14 位，随机后缀 7 位重算）
- 资源文件路径由相对位置决定，遵循三级目录约定

### 导出过程中的资源处理

`kernel/model/export.go` 中多种格式的资源策略：

| 导出格式 | 资源处理方式 | 关键实现 |
|---------|-------------|---------|
| **发布到链滴社区** | 见前文场景三：上传社区图床 + 内存前缀渲染（含空格→下划线），POST 到社区 API | `Export2Liandi` L322-L439 |
| **语雀复制（标准 Markdown）** | 预览窗「复制为语雀」触发；订阅者 `IsSubscriber()` 时通过 `LinkBase` 给输出 Markdown 加云端前缀，`assetsDestSpace2Underscore=true` 替换空格，仅影响内存输出 | `ExportStdMarkdown` L1678-L1722，仅被 `kernel/api/lute.go#L67` 调用 |
| **PDF / DOCX (`removeAssets=false`)** | 复制 assets 目录到临时导出文件夹并嵌入（DOCX: `tmpAssets` 目录；PDF: 内嵌图片流） | `ExportDocx` L832、`ProcessPDF` L1291 |
| **PDF / DOCX (`removeAssets=true`)** | 保持超链接形式，不复制资源文件到输出物 | `processPDFLinkEmbedAssets` L1488 |
| **导出压缩包（批量）** | 遍历所有引用 → `copiedAssets` HashSet 去重 → 按相对路径复制到导出目录，AV JSON 同步替换路径 | `export.go` L1957-L2008 |

### 笔记本级资源迁移辅助（有代码证据的内部函数）

代码位置：`kernel/model/assets.go#L1722-L1758`

| 函数 | 触发场景 | 行为 |
|------|---------|------|
| `copyBoxAssetsToDataAssets(boxID)` | 卸载笔记本时 `model/mount.go#L153` 自动调用 | 递归遍历笔记本下所有 `/assets/` 目录，复制到全局 `data/assets/` |
| `copyDocAssetsToDataAssets(boxID, parentDocPath)` | 文档被移出笔记本层级时 `model/file.go#L1582` 自动调用 | 把该文档及其子文档的同级 `assets/` 复制到全局 `data/assets/` |

⚠️ 这两个函数只有内核内部调用，**没有公开 API 或 UI 触发点**。

---

## 关键代码文件索引

> 以下均为仓库相对路径，便于脱离本机环境复核。

| 文件 | 职责 |
|------|------|
| `kernel/model/assets.go` | 资源管理核心：清理、重命名、引用提取、上传云端、网络转本地、Unused/Missing 算法 |
| `kernel/model/upload.go` | 上传主流程、本地文件插入、三级目录选择、文件名规范化、七牛 ETag 查重 |
| `kernel/api/asset.go` | HTTP API 层：unused/missing/remove/rename/ocr/上传云端两个端点 |
| `kernel/api/export.go` | export2Liandi HTTP 封装、导出系列 API 参数解析 |
| `kernel/api/lute.go` | **copyStdMarkdown Handler**，唯一调用 `ExportStdMarkdown` 的入口（`L67`） |
| `kernel/api/router.go` | 所有 API 路由注册与中间件挂载（含鉴权链） |
| `kernel/server/serve.go` | 文件上传端点 `/upload`、静态资源鉴权 Group |
| `kernel/model/session.go` | CheckAuth（7 种身份识别）/CheckAdminRole/CheckReadonly 权限中间件 |
| `kernel/model/conf.go` | IsSubscriber/IsPaidUser 订阅状态判断、GetUser 用户配置 |
| `kernel/model/export.go` | Export2Liandi 完整发布链路、ExportStdMarkdown、exportMarkdownContent0 前缀渲染、PDF/DOCX 导出 |
| `kernel/cache/asset.go` | 哈希-路径双层缓存、全局资产搜索缓存 |
| `kernel/sql/asset.go` | 资产数据库表 CRUD、按哈希查询 |
| `kernel/model/cloud_service.go` | LoadUploadToken 获取与缓存（1h TTL） |
| `kernel/util/cloud.go` | 双区域 6 组云端端点常量 |
| `kernel/util/etag.go` | 七牛云 ETag 分块哈希算法实现 |
| `kernel/util/file.go` | 文件名规范化、长度截断、非法字符过滤链 |
| `kernel/util/path.go` | IsAssetLinkDest 资源链接判定、工作空间路径校验 |
| `kernel/util/websocket.go` | PushMsg/PushErrMsg/PushUpdateMsg 等 WebSocket 广播函数 |
| `kernel/model/assets_watcher.go` | 非 macOS 平台文件系统监视器（fsnotify + 100ms 防抖） |
| `kernel/model/history.go` | 历史生成定时任务、保留策略、rollbackAssetsHistory 实现 |
| `kernel/job/cron.go` | 所有周期任务注册：历史清理、OCR、SQL 刷盘等 |
| `kernel/model/transaction.go` | 文档级事务 performTx 执行与 5 类错误码分类处理 |
| `kernel/conf/editor.go` | Editor 配置结构体（含 HistoryRetentionDays） |
| `app/src/protyle/upload/index.ts` | 前端上传：三阶段校验、XMLHttpRequest 进度、结果智能渲染、事务 undo/redo |
| `app/src/protyle/util/Options.ts` | Protyle 默认 upload 18 项配置初始化值 |
| `app/src/protyle/breadcrumb/index.ts` | 面包屑菜单：上传到云端图床 L363、分享到链滴 L375、网络资源本地化 L345 |
| `app/src/protyle/preview/index.ts` | 预览窗「复制为 Markdown → 语雀」触发 `copyStdMarkdown` API（L65 按钮定义，L280-L290 点击调用） |
| `app/src/config/image.ts` | 前端资源管理面板：未引用 / 缺失 / AV 清理三个 Tab UI |
| `app/src/util/fetch.ts` | fetchPost 统一 HTTP 封装、401 自动刷新、processMessage 路由 |
| `app/src/util/needSubscribe.ts` | 前端订阅者判断（含 iOS 平台差异化提示）+ showMessage 拦截 |
| `app/src/util/processMessage.ts` | WebSocket 消息分发（按 code 分支）与 Toast 呈现 |
| `app/src/types/protyle.d.ts` | IUpload 接口类型定义（18 字段完整文档） |
