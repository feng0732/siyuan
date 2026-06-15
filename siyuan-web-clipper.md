# SiYuan 剪藏与网页抓取功能实现分析

## 一、概述

SiYuan（思源笔记）的剪藏与网页抓取功能是一个多层次的内容处理系统，涵盖了从外部内容获取、清洗转换、资源归档到文档插入的完整链路。该功能主要由前端（TypeScript/Electron）和后端内核（Go）协同完成，通过 HTTP API 进行通信。

### 核心模块定位

| 模块 | 主要职责 | 关键文件 |
|------|---------|---------|
| 剪藏接入层 | 浏览器扩展内容接收、链滴文章直取 | [extension.go](kernel/api/extension.go) |
| 网络转发层 | 安全的 HTTP 请求转发、SSRF 防护 | [network.go](kernel/api/network.go) |
| 内容转换层 | HTML 转 Markdown、格式清洗 | [import.go](kernel/model/import.go)、[lute.go](kernel/util/lute.go) |
| 资源管理层 | 上传、去重、哈希缓存、网络资源本地化 | [upload.go](kernel/model/upload.go)、[assets.go](kernel/model/assets.go)、[asset.go](kernel/cache/asset.go) |
| 前端粘贴层 | 剪贴板解析、内容分发、插入渲染 | [paste.ts](app/src/protyle/util/paste.ts) |
| 安全过滤层 | XSS 防护、文件名过滤、路径安全 | [file.go](kernel/util/file.go) |
| 消息传递层 | 错误码、WebSocket 广播、前端消息展示 | [websocket.go](kernel/util/websocket.go)、[processMessage.ts](app/src/util/processMessage.ts)、[fetch.ts](app/src/util/fetch.ts) |

---

## 二、关键流程分析

### 2.1 外部内容获取

#### 2.1.1 剪藏扩展接入（`/api/extension/copy`）

**入口函数**：`extensionCopy()` in [extension.go](kernel/api/extension.go#L43-L291)

**处理流程**：

```
浏览器扩展 → Multipart Form → 内核解析 → 资源提取 → DOM 转换
```

**核心参数**：
- `dom`: 剪藏的 HTML DOM 内容
- `assets`: 资源文件列表（图片等）
- `notebook`: 目标笔记本 ID
- `href`: 源页面 URL（用于链滴文章特殊处理）
- `clipType`: 剪藏类型（`part` 表示部分剪藏）

**链滴文章优化路径**：
当检测到 `href` 为链滴（ld246.com）或流云（liuyun.io）文章且非部分剪藏时，系统会直接调用原始 Markdown 接口获取内容，绕过 HTML 解析，提高内容质量。

```go
// 链滴文章直接获取 Markdown 源码
if strings.HasPrefix(symArticleHref, "https://ld246.com/article/") {
    baseURL = "https://ld246.com/article/raw/"
    // ...
}
```

#### 2.1.2 网络请求转发（`/api/network/forwardProxy`）

**入口函数**：`forwardProxy()` in [network.go](kernel/api/network.go#L155-L331)

**功能定位**：提供安全的 HTTP 代理服务，供前端或插件调用外部 API。

**安全机制**：
- 仅允许 `http/https` 协议
- 最大重定向次数限制：3 次
- 私有 IP 地址拦截（SSRF 防护）
- DNS 重绑定防御（Control 钩子）

**请求编码支持**：
- `base64-std` / `base64-url`
- `base32-std` / `base32-hex`
- `hex`
- `text` / `json`（默认）

**响应编码支持**：同请求编码，可灵活配置。

**默认超时**：7000ms，可通过 `timeout` 参数调整。

### 2.2 内容清洗转换

#### 2.2.1 HTML 转 Markdown 核心流程

**核心函数**：`HTML2Tree()` in [import.go](kernel/model/import.go#L59-L108)

**转换引擎**：基于 Lute（Markdown 解析渲染引擎）的 `HTML2Tree` 方法。

**清洗步骤**：

1. **PUA 字符移除**：使用 `gulu.Str.RemovePUA()` 清理私有区字符
2. **HTML 块特殊处理**：
   - SVG 图片提取与转换
   - Base64 图片解码与本地化
3. **表格内容转义**：表格单元格中的 `|` 进行转义处理
4. **数学公式检测**：标记是否包含数学公式（`withMath`）

**链滴文章专属处理**：
- 缩进代码块转换为围栏代码块
- 开启 `IndentCodeBlock` 解析选项
- AST 遍历修改代码块节点结构

#### 2.2.2 格式噪声过滤

**前端层面**（[paste.ts](app/src/protyle/util/paste.ts)）：

1. **剪贴板内容分类**：
   - `text/siyuan`：思源内部格式，最高优先级
   - `text/html`：HTML 格式，需解析转换
   - `text/plain`：纯文本，直接 Markdown 解析
   - `Files`：文件列表，走上传流程

2. **HTML 噪声清理**：
   - 移除 `<!--StartFragment-->` / `<!--EndFragment-->` 标记
   - 浏览器地址栏拷贝特殊处理（纯链接识别）
   - 空 `<a>` 标签移除
   - Windows 剪贴板格式规范化

3. **纯文本/HTML 智能判定**：
   - 检测文本换行数量和 HTML 标签复杂度
   - 豆包等 AI 工具复制内容的特殊兼容（外层有 HTML 标签但实际是纯文本）
   - 单图片/表格+图片的特殊场景识别

**后端层面**（[extension.go](kernel/api/extension.go)）：

1. **行首空白剔除**：段落首节点的行首空白字符移除
2. **iframe 换行清理**：正则移除 `<iframe>` 标签内的换行符，避免解析异常
3. **图片 alt/title 规范化**：
   - 检测并转换非文本格式的 alt 和 title
   - 使用 `parse.Inline()` 解析后提取纯文本
4. **TextMark 转 Inlines**：`parse.TextMarks2Inlines()` 统一行级标记格式
5. **嵌套行级扁平化**：`parse.NestedInlines2FlattedSpansHybrid()` 处理嵌套结构

#### 2.2.3 安全内容过滤（XSS 防护）

**Lute 引擎层面**：
- `SetSanitize(true)` in [lute.go](kernel/util/lute.go#L81)
- 启用 Lute 内置的 HTML 消毒功能
- 前端也调用 `Lute.Sanitize(textHTML)` 进行预处理

**文件名安全过滤**（[file.go](kernel/util/file.go#L225-L247)）：

`FilterUploadFileName()` 执行多层过滤：

| 过滤项 | 处理方式 | 风险点 |
|--------|---------|--------|
| 路径分隔符 `\ / :` | 替换为 `_` | 路径遍历攻击 |
| 通配符 `* ?` | 替换为 `_` | 文件系统注入 |
| 引号 `" '` | 替换为 `_` | 注入攻击 |
| 尖括号 `< >` | 替换为 `_` | XSS |
| 管道符 `\|` | 替换为 `_` | 命令注入 |
| Markdown 特殊字符 `~ [ ] ( ) ! \` & { } = # % $ ;` | 直接移除 | 语法注入 |
| 不可见字符 | `RemoveInvalid()` 移除 |  Unicode 攻击 |
| 文件名长度 | 截断至 189 字节 | 文件系统限制 |

**表情文件名特殊过滤**：
- `FilterUploadEmojiFileName()` 保留 `/` 分隔符（用于动态图标路径）
- 针对 XSS through emoji name 的防护（issue #15034）

**路径安全防护**：
- `IsSensitivePath()` 敏感路径检查
- `IsAbsPathInWorkspace()` 工作区内路径校验
- `IsSubPath()` 子路径验证

### 2.3 资源归档

#### 2.3.1 资源上传流程

**核心函数**：`Upload()` in [upload.go](kernel/model/upload.go#L131-L353)

**上传步骤**：

```
接收文件 → 文件名过滤 → 哈希计算 → 重复检测 → 写入磁盘 → 缓存更新
```

**资源目录查找策略**（`getAssetsDir()`）：
1. 优先使用文档同级 `assets` 目录
2. 其次使用笔记本级 `assets` 目录
3. 最后使用全局 `data/assets` 目录

#### 2.3.2 重复内容识别（资源去重）

**基于内容哈希的去重机制**：

1. **哈希计算**：使用 ETag 算法计算文件内容哈希（[etag.go](kernel/util/etag.go)）

2. **缓存查询**（[asset.go](kernel/cache/asset.go)）：
   - 内存缓存：`assetHashCache`（map 结构，sync.Mutex 保护）
   - 数据库查询：`sql.QueryAssetByHash()` 作为后备
   - 存在性验证：缓存命中后验证文件实际存在

3. **同名但不同文件处理**：
   - 哈希相同但文件名不同 → 使用随机哈希前缀强制重新保存
   - 防止文件名冲突导致的资源误引用

4. **特殊场景跳过重复**：
   - PDF 标注图片上传支持 `skipIfDuplicated` 参数
   - 通过文件名模式匹配（Glob）寻找已有文件

**重复资源命中后的用户可见结果**：
- **文件上传场景**（`Upload()`）：`succMap` 返回已存在的相对路径（`assets/xxx_id.ext`），前端无感知使用
- **网络资源本地化场景**（`NetAssets2LocalAssets()`）：同批次 URL 通过 `assetsMap` 内存缓存去重，后续直接复用已下载文件路径

**缓存同步机制**：
- 文件系统事件监听（assets_watcher）
- `HandleAssetsChangeEvent()` / `HandleAssetsRemoveEvent()` 实时更新缓存
- 启动时全量加载：`LoadAssets()`

#### 2.3.3 网络资源本地化完整路径

**API 端点**：
- `/api/format/netImg2LocalAssets`：仅图片本地化，入口 [format.go](kernel/api/format.go#L50-L73)
- `/api/format/netAssets2LocalAssets`：所有网络资源本地化，入口 [format.go](kernel/api/format.go#L28-L48)

**核心实现**：`NetAssets2LocalAssets()` + `netAssets2LocalAssets0()` in [assets.go](kernel/model/assets.go#L229-L445)

**完整处理链路**：

```
① 准备阶段
   ├─ syncingFiles.Store(rootID) 标记文档正在同步
   ├─ LoadTreeByBlockID() 加载文档 AST 树
   ├─ getAssetsDir() 确定 assets 目录（三级策略）
   └─ 创建 assets 目录（若不存在）

② 遍历网络链接
   ├─ getRemoteAssetsLinkDestsInTree() 提取文档中所有远程资源链接
   │
   └─ 对每个链接执行 ③~⑥

③ 下载阶段
   ├─ util.NewCustomReqClient() 使用自定义 TLS 指纹的 HTTP 客户端
   ├─ URL 规范化处理：
   │   ├─ // 前缀 → 补充 https:
   │   └─ qpic.cn 微信图片 → http 强制替换为 https
   ├─ SetRetryCount(1) + SetRetryFixedInterval(3s) 重试机制
   ├─ 设置 Referer 头（originalURL，提升防盗链站点下载成功率）
   ├─ 发送 GET 请求
   │
   ├─ 下载异常过滤（失败则 continue，跳过该资源）：
   │   ├─ reqErr != nil → 网络请求错误，记录日志
   │   ├─ statusCode=403/401 → forbiddenCount++，统计防盗链数量
   │   ├─ ContentType=text/html → 忽略，防止误下载网页
   │   ├─ statusCode!=200 → 非 200 状态码，记录日志
   │   └─ ContentLength>96MB → 超大文件跳过，记录警告
   │
   └─ resp.ToBytes() 将响应体转为字节数组

④ 命名阶段
   ├─ 从 URL 提取文件名：
   │   ├─ 去除 query string (?xxx)
   │   ├─ 去除 fragment (#xxx)
   │   └─ url.PathUnescape() URL 解码
   ├─ util.FilterUploadFileName() 文件名安全过滤
   ├─ 扩展名识别（四级回退策略）：
   │   ├─ ① 从 URL 中提取的扩展名
   │   ├─ ② util.IsCommonExt() 校验，无效则 mimetype.Detect(data) 内容检测
   │   ├─ ③ SVG 特殊识别（<svg ... </svg> 前缀后缀匹配）
   │   └─ ④ Content-Type 头 → mime.ExtensionsByType()
   ├─ util.AssetName(name, ast.NewNodeID()) 追加节点 ID 后缀
   └─ "network-asset-" 前缀标识

⑤ 去重与落盘阶段
   ├─ assetsMap[u] 同批次 URL 缓存（内存级去重）
   │   └─ 同一 URL 出现多次 → 复用首次下载结果
   ├─ writePath = filepath.Join(assetsDirPath, name)
   ├─ filelock.WriteFile(writePath, data) 写入磁盘
   ├─ setAssetsLinkDest() 将 AST 中的 URL 替换为 assets/xxx
   ├─ assetsMap[u] = name 写入缓存
   ├─ files++ 计数
   └─ size += len(data) 累计大小

⑥ 结果推送
   ├─ PushClearMsg(msgId) 清除"正在下载"提示
   ├─ files > 0 分支：
   │   ├─ PushMsg(Language(113)) "正在完成数据写入..."
   │   ├─ writeTreeUpsertQueue() 持久化文档树
   │   └─ PushUpdateMsg(Language(120), files, size) 成功提示
   │       "下载完毕，一共 [N] 个文件，共占用 [X] 磁盘空间"
   ├─ forbiddenCount > 0 分支：
   │   └─ PushErrMsg(Language(255), forbiddenCount) 防盗链警告
   │       "目标站点启用了防盗链，[N] 个资源无法下载"
   └─ files == 0 && forbiddenCount == 0 分支：
       └─ PushMsg(Language(121), 3000) "该文档中不存在网络文件"
```

**本地文件链接的特殊处理**（`file:///` 或本地绝对路径）：
- 检查文件存在性
- 检查是否为目录（忽略）
- `IsSensitivePath()` 敏感路径检查
- 同样执行文件名过滤 + ID 后缀
- 通过 `filelock.Copy()` 复制而非下载

#### 2.3.4 资源本地化替换覆盖面（五类资源差异对比）

资源本地化功能覆盖 **五类资源节点**，通过 `onlyImg` 参数控制处理范围。`netImg2LocalAssets` 仅处理图片，`netAssets2LocalAssets` 处理全部五类。

**核心提取函数**：`getRemoteAssetsLinkDests()` in [assets.go](kernel/model/assets.go#L1542-L1621)
**核心替换函数**：`setAssetsLinkDest()` in [assets.go](kernel/model/assets.go#L1494-L1540)

| 资源类型 | AST 节点类型 | onlyImg 处理 | 提取来源 | 实际替换字段 | 去重键 | 保存方式 | 计数统计 | 字段级一致性 |
|---------|-------------|-------------|---------|-------------|--------|---------|---------|-------------|
| **图片** | `NodeLinkDest` + `ParentIs(NodeImage)` | ✅ 处理 | `node.Tokens` | `node.Tokens`（字节替换） | URL 字符串 | HTTP 下载 / 文件复制 | ✅ 计入 files & size | ✅ 一致 |
| **普通链接** | `NodeLinkDest`（非图片父节点） | ❌ 跳过 | `node.Tokens` | `node.Tokens`（字节替换） | URL 字符串 | HTTP 下载 / 文件复制 | ✅ 计入 files & size | ✅ 一致 |
| **超链接 TextMark** | `IsTextMarkType("a")` | ❌ 跳过 | `node.TextMarkAHref` | `node.TextMarkAHref`（字符串替换） | URL 字符串 | HTTP 下载 / 文件复制 | ✅ 计入 files & size | ✅ 一致 |
| **音频/视频** | `NodeAudio` / `NodeVideo` | ❌ 跳过 | `GetNodeSrcTokens(node)` 解析 `Tokens` 中 `src="..."` | 仅 `node.Tokens`（字节替换）；**不更新 TextMarkAHref** | URL 字符串 | HTTP 下载 / 文件复制 | ✅ 计入 files & size | ❌ **严重不一致**，见下方分析 |
| **属性视图资源字段** | `NodeAttributeView` + `KeyTypeMAsset` | 仅 `AssetTypeImage` | `value.MAsset[].Content` | `asset.Content`（对象属性替换） | URL 字符串 | HTTP 下载 / 文件复制 | ✅ 计入 files & size | ✅ 一致 |

**各类型详细差异与字段级分析**：

1. **图片（Image）**
   - Markdown 语法：`![alt](url)`
   - 提取条件：`NodeLinkDest` 节点且父节点是 `NodeImage`
   - `onlyImg=true` 时**唯一**被处理的链接型资源
   - 属性视图中仅 `AssetTypeImage` 类型的资源被处理
   - **一致性验证**：
     - 提取：`string(node.Tokens)`
     - `//` 前缀规范化：`bytes.HasPrefix(node.Tokens, []byte("//"))` → 在 `Tokens` 前加 `https:`
     - 替换：`bytes.ReplaceAll(node.Tokens, oldDest, dest)`
     - ✅ **完全一致**，无字段级误差

2. **普通链接（Link）**
   - Markdown 语法：`[text](url)`
   - 提取条件：`NodeLinkDest` 节点且**非**图片父节点
   - 仅 `onlyImg=false`（`netAssets2LocalAssets`）时处理
   - 与图片共享相同的 `NodeLinkDest` 节点类型，通过父节点类型区分
   - **一致性验证**：同图片，✅ **完全一致**

3. **超链接 TextMark（TextMark A）**
   - SiYuan 特有的行级标记格式
   - 提取条件：`node.IsTextMarkType("a")` 为 true
   - 仅 `onlyImg=false` 时处理
   - 存储字段为 `node.TextMarkAHref`（字符串类型，非 Tokens）
   - **一致性验证**：
     - 提取：`node.TextMarkAHref`
     - `//` 前缀规范化：`strings.HasPrefix(node.TextMarkAHref, "//")` → 在 `TextMarkAHref` 前加 `https:`
     - 替换：`strings.ReplaceAll(node.TextMarkAHref, oldDest, dest)`
     - ✅ **完全一致**，无字段级误差

4. **音频/视频（Audio/Video）**
   - 块级媒体元素
   - 提取条件：`NodeAudio` 或 `NodeVideo` 节点类型
   - 仅 `onlyImg=false` 时处理
   - 通过 `treenode.GetNodeSrcTokens(node)` 提取源地址（从 `Tokens` 解析 `src="..."` 属性）
   - **节点数据结构**：
     - `Tokens`：完整 HTML 标签，如 `<audio controls="controls" src="URL1" data-src="URL2"></audio>`
     - `TextMarkAHref`：**创建时从未设置**（默认空字符串），可能在其他场景被赋值
     - `src` 和 `data-src` 可存储**不同 URL**（从 guide 文档实证：`src` 指向压缩后文件，`data-src` 指向原始文件）
   - **一致性验证（发现三个字段级 Bug）**：

     **Bug 1：`//` 前缀规范化字段错误（致命）**
     ```go
     // 实际代码
     } else if ast.NodeAudio == node.Type || ast.NodeVideo == node.Type {
         if strings.HasPrefix(node.TextMarkAHref, "//") {  // ❌ 检查的是 TextMarkAHref（空字符串）
             node.TextMarkAHref = "https:" + node.TextMarkAHref
         }
         node.Tokens = bytes.ReplaceAll(node.Tokens, []byte(oldDest), []byte(dest))
     }
     ```
     - 问题：`TextMarkAHref` 是**空字符串**，`strings.HasPrefix("", "//")` 永远为 false
     - 后果：`//` 开头的 URL 永远不会被规范化
     - 叠加效应：`netAssets2LocalAssets0` 中 `dest = "https:" + dest` 已规范化 oldDest，但 `Tokens` 中仍是 `//example.com/...`
     - **最终结果**：`//` 开头的音视频 URL **永远无法被替换**

     **Bug 2：`data-src` 属性未被提取和单独替换**
     - 提取：`GetNodeSrcTokens()` 只解析 `src="..."` 的值，忽略 `data-src="..."`
     - 替换：`bytes.ReplaceAll()` 只会替换与 `oldDest`（即 `src` 中的 URL）匹配的字符串
     - 问题：如果 `data-src` 中的 URL 与 `src` 不同（如 guide 文档所示，分别指向压缩版和原始版），则 `data-src` **不会被替换**
     - 后果：播放时可能仍从远程加载原始文件，本地化不彻底

     **Bug 3：`TextMarkAHref` 未被同步更新**
     - 如果 `TextMarkAHref` 在某些场景下被设置了 URL，替换时只更新了 `Tokens`，未更新 `TextMarkAHref`
     - 后果：两个字段数据不一致，可能导致引用错乱

5. **属性视图资源字段（Attribute View MAsset）**
   - 数据库视图中的资源类型列
   - 提取条件：`NodeAttributeView` 节点 + `KeyTypeMAsset` 键类型
   - `onlyImg=true` 时仅处理 `AssetTypeImage` 类型
   - `onlyImg=false` 时处理所有资源类型（含 `AssetTypeFile` 等）
   - 替换需调用 `av.SaveAttributeView()` 持久化属性视图数据
   - 替换时遍历 `keyValues → values → MAsset` 三层嵌套结构
   - **一致性验证**：
     - 提取：`asset.Content`
     - 替换：`asset.Content = dest`
     - 无 `//` 前缀规范化逻辑（属性视图中不会出现 `//` 开头的 URL）
     - ✅ **完全一致**，无字段级误差

**本地文件链接的特殊处理（五类通用）**：
- 由 `util.FileURLToLocalPath(dest)` 识别（`file:///` 协议或本地绝对路径）
- 三类跳过条件：文件不存在 / 是目录 / 敏感路径
- 复制使用 `filelock.Copy()` 而非 HTTP 下载
- 大小通过 `gulu.File.GetFileSize()` 获取
- 同样执行文件名过滤 + `network-asset-` 前缀 + ID 后缀

#### 2.3.5 四类场景对计数、提示和文档内容的影响

资源本地化过程中，不同的执行结果对 **成功计数**、**用户可见提示** 和 **文档内容** 三方面的影响各不相同：

| 场景 | 成功计数 (files/size) | 用户可见提示 | 文档内容变化 | 触发条件 |
|------|----------------------|-------------|-------------|---------|
| **重复资源命中**（同批次） | ❌ 不计入 | 无提示，静默复用 | ✅ 链接被替换为本地路径 | 同一 URL 在文档中出现多次 |
| **下载成功** | ✅ 计入 files++ & size+= | 最终汇总成功提示 | ✅ 链接被替换为本地路径 | HTTP 200 + 写入磁盘成功 |
| **防盗链失败** (403/401) | ❌ 不计入 | 🔴 红色错误提示："目标站点启用了防盗链，[N] 个资源无法下载" | ❌ 保留原始 URL | `forbiddenCount++` 统计，结束时统一提示 |
| **其他下载失败** (网络错误/非200/超大/HTML等) | ❌ 不计入 | ⚠️ 无显式提示（仅日志） | ❌ 保留原始 URL | reqErr / 非 200 / >96MB / text/html |
| **本地文件复制失败** | ❌ 不计入 | ⚠️ 无显式提示（仅日志 Error） | ❌ 保留原始路径 | Copy 出错 / 文件不存在 / 目录 / 敏感路径 |
| **无网络资源** | 0 | ℹ️ 信息提示："该文档中不存在网络文件"（3s） | ❌ 无变化 | `files==0 && forbiddenCount==0` |
| **部分成功 + 防盗链** | 仅成功数计入 | ✅ 成功提示 + 🔴 防盗链警告（两条消息） | ✅ 成功的替换，失败的保留 | 混合场景 |
| **音视频 `//` 前缀 URL** | ❌ 不计入 | ⚠️ 无提示，静默失败 | ❌ 保留原始 URL | 触发 **Bug 1**，`bytes.ReplaceAll` 不匹配 |
| **音视频 `data-src` 不同 URL** | 仅 `src` 成功计入 | ✅ 显示成功提示（但不完整） | ⚠️ `src` 被替换，`data-src` 仍为远程 | 触发 **Bug 2**，`data-src` 未被提取 |

**详细说明**：

1. **重复资源命中**
   - 去重缓存键：`assetsMap[url] = filename`（内存 map，单次调用内有效）
   - 命中后直接调用 `setAssetsLinkDest()` 替换链接
   - 不计入 `files` 和 `size`（避免重复统计）
   - 文档内容会被替换（所有出现处都指向同一个本地文件）

2. **防盗链统计**
   - 检测条件：`resp.StatusCode == 403 || resp.StatusCode == 401`
   - 计数器：`forbiddenCount++`（单独统计，不计入失败总数）
   - 提示时机：全部下载完成后统一弹出（红色错误样式，5 秒）
   - 防盗链资源**不替换**文档链接，保留原始 URL

3. **部分下载失败**
   - 非防盗链的其他失败（网络错误、超时、非 200、超大文件、HTML 内容）**无单独提示**
   - 仅通过 `logging.LogErrorf()` / `LogWarnf()` 记录日志
   - 用户需自行对比前后内容判断是否有遗漏
   - 失败的资源保留原始 URL，文档内容部分变化
   - **音视频 `//` 前缀失败**：属于此类，无任何提示，用户无法感知

4. **成功计数与提示**
   - 成功提示：`"下载完毕，一共 [N] 个文件，共占用 [X] 磁盘空间"`
   - 提示形式：`PushUpdateMsg()` 更新"正在写入"消息为成功消息
   - `files == 0 && forbiddenCount == 0` → 显示"不存在网络文件"
   - `files > 0 && forbiddenCount > 0` → 同时显示成功消息 + 防盗链警告（两条独立消息）
   - **注意**：音视频 `data-src` 未替换时，成功计数是准确的（只下载了 `src` 对应的文件），但文档内容不完整

#### 2.3.6 Base64 图片处理

**处理函数**：`processBase64Img()` in [import.go](kernel/model/import.go#L1250-L1328)

**支持格式**：
- `image/png`
- `image/jpeg`
- `image/svg+xml`

**处理流程**：
1. 解析 Base64 数据
2. 解码图片并重新编码（保证格式正确）
3. 生成安全文件名
4. 写入临时目录后拷贝到 assets
5. 替换 AST 节点中的链接地址

### 2.4 文档插入

#### 2.4.1 前端粘贴分发逻辑

**核心函数**：`paste()` in [paste.ts](app/src/protyle/util/paste.ts#L249-L640)

**内容分发路径**：

```
剪贴板数据
    ├─→ text/siyuan 存在 → 内部粘贴（重新生成ID）
    ├─→ 代码块上下文 → 纯文本插入
    ├─→ 文件列表 → 上传处理
    ├─→ HTML 内容 → /api/lute/html2BlockDOM → 插入
    └─→ 纯文本 → Markdown 解析 → 插入
```

**特殊处理场景**：

1. **单链接粘贴**：自动转换为链接格式（`pasteURLAutoConvert` 配置）
2. **块引用粘贴**：识别 `((id "text"))` 格式
3. **文件标注粘贴**：识别 `<<file "annotation">>` 格式
4. **动态引用粘贴**：识别块引用语法
5. **Base64 图片转换**：前端转换为 Blob URL 后上传

#### 2.4.2 后端 HTML 转块 DOM

**API 端点**：`/api/lute/html2BlockDOM` in [lute.go](kernel/api/lute.go#L78-L202)

**处理流程**：
1. 调用 `model.HTML2Tree()` 转换为 Markdown AST
2. 空列表项/空引用块清理
3. 单单元格表格转换为段落
4. 容器模式下本地资源文件复制
5. `TextMarks2Inlines` + `NestedInlines2FlattedSpansHybrid` 规范化
6. 格式化为 Markdown 后再解析为 Protyle DOM

### 2.5 失败提示与错误传递机制

#### 2.5.1 错误消息传递的三层架构

错误消息从内核到前端用户界面经过三层传递：

```
┌─────────────────────────────────────────────────────────────┐
│ 第一层：内核模型层（Go）                                      │
│  ├─ 生成错误码 + 多语言文案                                   │
│  ├─ Conf.Language(n) 从语言包读取提示文字                      │
│  └─ 两种传递通道：                                            │
│      ├─ 通道 A：HTTP 同步响应（ret.Code / ret.Msg / ret.Data）│
│      └─ 通道 B：WebSocket 广播（PushMsg/PushErrMsg 等）       │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│ 第二层：接口层 + 前端请求层                                    │
│  ├─ HTTP 响应：ret{code, msg, data:{closeTimeout, ...}}      │
│  ├─ fetchPost() / fetchSyncPost() 状态码处理                  │
│  │   ├─ 401: 3秒后刷新页面                                    │
│  │   ├─ 403/404: 返回错误信息                                 │
│  │   └─ 网络异常: 控制台警告 + 事务请求触发 kernelError       │
│  └─ WebSocket: BroadcastByType("main", "msg", code, msg, ...)│
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│ 第三层：前端消息展示层（TypeScript）                           │
│  ├─ processMessage() 统一处理                                 │
│  │   ├─ cmd="msg": showMessage(msg, timeout, type)           │
│  │   ├─ cmd="cmsg": hideMessage(id)                          │
│  │   └─ cmd="cprogress": 移除进度条                           │
│  ├─ Code 判定规则：                                           │
│  │   ├─ code < 0: 弹出提示                                    │
│  │   │   ├─ code == -1 → "error" 红色样式                     │
│  │   │   └─ code == -2 → "info" 普通样式                      │
│  │   └─ code == 0: 正常不提示（除非 cmd="msg" 强制）          │
│  └─ closeTimeout: 自动关闭毫秒数（0=不自动关闭）               │
└─────────────────────────────────────────────────────────────┘
```

#### 2.5.2 WebSocket 广播接口

**核心函数**（[websocket.go](kernel/util/websocket.go#L226-L321)）：

| 函数 | code | 用途 | 示例 |
|------|------|------|------|
| `PushMsg(msg, timeout)` | 0 | 普通提示 | "正在完成数据写入..." |
| `PushUpdateMsg(id, msg, timeout)` | 0 | 更新已存在的提示 | "正在下载网络文件 [url...]" |
| `PushErrMsg(msg, timeout)` | -1 | 错误提示（红色） | "目标站点启用了防盗链..." |
| `PushClearMsg(id)` | 0 | 关闭指定消息 | 下载完成后关闭进度 |
| `BroadcastByType(typ, cmd, code, msg, data)` | 可变 | 底层广播原语 | - |

#### 2.5.3 错误码体系

**HTTP 响应 ret.Code 约定**：

| Code | 含义 | 前端表现 |
|------|------|---------|
| 0 | 成功 | 正常执行回调，不弹提示（除非 `cmd="msg"`） |
| -1 | 错误 | `showMessage` 以红色 error 样式弹出，默认 5s 关闭 |
| -2 | 提示 | `showMessage` 以普通 info 样式弹出 |
| >0 | 特殊业务错误 | 由具体 API 自定义处理逻辑 |
| -401/-403/-404 | HTTP 状态码映射 | 直接返回状态文本 |

**内核语言包关键错误码**（`_kernel` in [zh_CN.json](app/appearance/langs/zh_CN.json#L1488-L1776)）：

| Code | 中文提示 | 触发场景 |
|------|---------|---------|
| 15 | 未找到 ID 为 [%s] 的内容块 | 剪藏目标块不存在 |
| 24 | 网络超时，请稍后再试 | 网络请求超时 |
| 27 | 正在上传 [%v] | 上传进度提示 |
| 28 | 网络异常，请稍后再试 | 通用网络错误 |
| 41 | 上传完毕 [%d] | 上传完成计数 |
| 70 | 正在处理 [%s]，请稍等... | 通用进度提示 |
| 71 | 插入资源文件失败，请重新打开文档 | BlockTree 为空（upload.go） |
| 72 | 内容已经复制到系统剪切板，请到思源中进行粘贴 | 剪藏扩展成功提示（extension.go） |
| 73 | 导入数据中... | 导入流程通用提示 |
| 113 | 正在完成数据写入... | 网络资源本地化开始写入 |
| 119 | 正在下载网络文件 [%s] | 单文件下载进度 |
| 120 | 下载完毕，一共 [%d] 个文件，共占用 [%s] 磁盘空间 | 网络资源本地化成功 |
| 121 | 该文档中不存在网络文件 | 本地化无文件可处理 |
| 179 | 磁盘空间可能不足... | 磁盘空间警告 |
| 255 | 目标站点启用了防盗链，[%d] 个资源无法下载 | 403/401 统计警告 |
| 258 | 操作失败，请稍后再试 | 通用操作失败 |

#### 2.5.4 网络异常处理

**转发代理异常**（[network.go](kernel/api/network.go)）：

| 错误场景 | 错误码 | 提示信息 |
|---------|--------|---------|
| URL 格式无效 | -1 | `invalid [url]` |
| 非 http/https 协议 | -1 | `only http/https is allowed` |
| 请求发送失败 | -1 | `forward request failed: {err}` |
| 响应体读取失败 | -1 | `read response body failed: {err}` |
| Payload 解码失败 | -2 | `decode {encoding} payload failed: {err}` |

**前端请求异常**（[fetch.ts](app/src/util/fetch.ts)）：

| HTTP 状态 | 处理方式 |
|----------|---------|
| 401 鉴权失败 | 3 秒后刷新页面 |
| 403/404 | 返回错误信息 |
| 网络错误 | 控制台警告，事务请求触发 kernelError |

**异常传播**：
- `processMessage()` 统一处理返回消息
- 失败回调 `failCallback` 支持自定义错误处理

#### 2.5.5 资源下载异常的用户可见结果

| 异常类型 | 用户可见提示 | 行为 |
|---------|-------------|------|
| 网络请求错误（reqErr） | 无显式提示，仅日志 | 跳过该资源，保留原始 URL |
| HTTP 403/401（防盗链） | "目标站点启用了防盗链，[N] 个资源无法下载" | 统计数量，结束时统一红色警告 |
| 返回 text/html | 无提示，仅日志 | 跳过，避免下载网页 |
| 非 200 状态码 | 无提示，仅日志 | 跳过 |
| 文件 >96MB | 无提示，仅日志（Warn 级别） | 跳过 |
| 写入磁盘失败 | 无提示，仅日志（Error 级别） | 跳过 |
| 全部下载成功 | "下载完毕，一共 [N] 个文件，共占用 [X] 磁盘空间" | 绿色信息提示，5s 关闭 |
| 文档无网络文件 | "该文档中不存在网络文件" | 信息提示，3s 关闭 |

#### 2.5.6 内容转换失败

- HTML 解析失败 → 日志记录，继续处理后续节点
- 图片解码失败 → 跳过该图片，保留原始链接
- Base64 解码失败 → 日志记录，返回原始内容

---

## 三、协作分工

### 3.1 前后端职责划分

| 层级 | 组件 | 主要职责 |
|------|------|---------|
| 前端表现层 | [paste.ts](app/src/protyle/util/paste.ts) | 剪贴板数据读取、格式判断、内容分发、渲染触发 |
| 前端工具层 | [fetch.ts](app/src/util/fetch.ts) | API 请求封装、HTTP 状态码映射、异常回调 |
| 前端消息层 | [processMessage.ts](app/src/util/processMessage.ts) | WebSocket 消息分发、提示框展示与关闭 |
| 后端 API 层 | [extension.go](kernel/api/extension.go) | 剪藏请求接收、参数解析、响应组装 |
| 后端 API 层 | [network.go](kernel/api/network.go) | 网络请求转发、SSRF 安全过滤、编码转换 |
| 后端 API 层 | [format.go](kernel/api/format.go) | 网络资源本地化 API 入口、错误码封装 |
| 后端模型层 | [import.go](kernel/model/import.go) | HTML 转 Markdown、AST 处理、Base64 图片 |
| 后端模型层 | [upload.go](kernel/model/upload.go) | 文件上传、资源去重、三级目录管理 |
| 后端模型层 | [assets.go](kernel/model/assets.go) | 网络资源下载、命名、落盘、链接替换 |
| 后端缓存层 | [cache/asset.go](kernel/cache/asset.go) | 资源哈希缓存、加速去重查询 |
| 后端工具层 | [file.go](kernel/util/file.go) | 文件名过滤、路径安全、文件操作 |
| 后端工具层 | [lute.go](kernel/util/lute.go) | Lute 引擎配置、Markdown 环境初始化 |
| 后端通信层 | [websocket.go](kernel/util/websocket.go) | WebSocket 广播、进度推送、错误提示 |

### 3.2 数据流协作

```
浏览器扩展
    ↓ (Multipart Form: dom + files)
/api/extension/copy
    ├─→ 资源文件 → 文件名过滤 → 哈希计算 → 写入 assets
    └─→ DOM 内容
          ├─→ 链滴文章? → 直接获取 Markdown → 缩进代码块转换
          └─→ 普通 HTML → HTML2Tree → 格式清洗 → 图片链接替换
                ↓
          Markdown 输出 + withMath 标记
                ↓
          ret{code:0, msg:Language(72), data:{...}}
                ↓
    fetchPost → processMessage → showMessage(L72: 内容已复制到剪切板)
                ↓
    前端 paste 处理 → insertHTML → 渲染更新

┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄

用户菜单: 网络图片转本地
    ↓
/api/format/netImg2LocalAssets
    ├─ 参数校验失败 → ret{code:-1, msg:err, data:{closeTimeout:5000}}
    └─ 正常 → NetAssets2LocalAssets()
          ├─ PushUpdateMsg(L119: 正在下载...) via WebSocket
          ├─ 下载循环 → 成功/跳过/防盗链统计
          ├─ PushClearMsg(id) 关闭进度
          ├─ 有成功 → PushMsg(L113: 正在写入) → PushUpdateMsg(L120)
          ├─ 防盗链 → PushErrMsg(L255) 红色警告
          └─ 无文件 → PushMsg(L121) 提示
                ↓
          processMessage WebSocket 消息 → showMessage 弹窗
```

---

## 四、特别关注机制详解

### 4.1 网络异常处理

#### 4.1.1 SSRF 防护体系

**三层防护**（[network.go](kernel/api/network.go#L333-L361)）：

1. **协议白名单**：仅允许 http/https
2. **IP 地址校验**：`isPrivateIP()` 过滤私有地址
   - 回环地址 `127.0.0.0/8`
   - 链路本地地址 `169.254.0.0/16`
   - 私有网络 `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
   - 未指定地址 `0.0.0.0`
3. **DNS 重绑定防御**：在 `Control` 钩子中验证解析后的 IP

#### 4.1.2 超时与重试

- 默认超时：7000ms（转发代理）
- 网络资源下载：`SetRetryCount(1)` + `SetRetryFixedInterval(3s)`，即失败后重试 1 次，间隔 3 秒
- 网络连通性检测：2 次重试，间隔 1 秒（`isOnline()` in [net.go](kernel/util/net.go#L159-L190)）
- 重定向限制：最多 3 次跳转

#### 4.1.3 本地地址验证工具集

- `IsLocalHostname()`：主机名本地检测
- `IsLocalHost()`：host:port 格式本地检测
- `IsLocalOrigin()`：Origin 头本地检测
- `GetPrivateIPv4s()`：获取本机私有 IPv4 列表（排除虚拟网卡）

### 4.2 格式噪声过滤

#### 4.2.1 多层过滤架构

```
原始 HTML
    ↓
第一层：浏览器剪贴板标准化（前端）
    ├─ 移除 Fragment 标记
    ├─ 移除空标签
    └─ 特殊来源适配（豆包等）
    ↓
第二层：Lute Sanitize（前端/后端）
    └─ XSS 过滤 + 危险标签移除
    ↓
第三层：HTML2Tree 转换（后端）
    ├─ PUA 字符移除
    ├─ 行首空白剔除
    └─ 特殊标签处理（iframe, svg, pre）
    ↓
第四层：AST 后处理（后端）
    ├─ 图片 alt/title 规范化
    ├─ 空块清理
    └─ 嵌套结构扁平化
```

#### 4.2.2 特殊场景兼容

| 场景 | 处理方式 | 相关 Issue |
|------|---------|-----------|
| 豆包复制粘贴 | 检测换行数量+简单标签，降级为纯文本 | #13265, #14313 |
| 单图片复制 | 识别 `<img>` 开头的 HTML，走文件上传 | #7021 |
| Excel 表格含图 | 走文件上传而非 HTML 解析 | - |
| 地址栏链接拷贝 | 检测单链接模式，直接文本处理 | - |
| PDF 标注复制 | 行尾零宽字符标识内部格式 | #11629 |
| Windows 剪贴板 | 处理 StartFragment 标记 | - |
| 微信 qpic.cn 图片 | http→https 协议升级 | #5052, #6431 |
| `//` 协议相对 URL | 自动补充 `https:` | #10598 |

### 4.3 重复内容识别

#### 4.3.1 两种去重机制对比

SiYuan 的资源去重分为两套独立机制，分别应用于不同场景：

| 机制 | 应用场景 | 去重键 | 缓存层 | 跨调用持久化 |
|------|---------|--------|--------|-------------|
| **内容哈希去重** | `Upload()` 文件上传 | ETag 内容哈希 | 内存 `assetHashCache` + SQL 数据库 | ✅ 跨调用生效 |
| **URL 内存去重** | `NetAssets2LocalAssets()` 网络资源本地化 | URL 字符串 | 内存 `assetsMap`（单次调用内） | ❌ 仅单次调用内 |

#### 4.3.2 资源上传去重（内容哈希）

```
文件上传
    ↓
计算内容哈希（ETag）
    ↓
┌────────────────────────────────────────┐
│ 内存缓存（assetHashCache）             │
│  ├─ 命中 → 验证文件存在 → 返回已有路径 │
│  └─ 未命中 → 查询 SQL 数据库           │
│        ├─ 命中 → 写入缓存 → 返回已有路径│
│        └─ 未命中 → 写入新文件 → 更新缓存│
└────────────────────────────────────────┘
```

**注意事项**：
- 哈希相同但文件名不同 → 强制重新保存（避免文件名误导）
- 空文件 → 使用随机哈希（避免所有空文件冲突）
- PDF 标注场景 → 支持文件名模式匹配去重（`skipIfDuplicated` + Glob）

#### 4.3.3 网络资源本地化去重（URL 内存级）

```
遍历文档所有远程资源链接
    ↓
每类节点调用 getRemoteAssetsLinkDests() 提取 URL
    ↓
assetsMap[url] 内存字典查询
    ├─ 命中 → 直接 setAssetsLinkDest() 替换，不计入计数
    └─ 未命中 → 下载/复制 → 写入 assetsMap → 替换链接
```

**五类资源节点共享同一去重字典**：
- 图片链接（`NodeLinkDest` + `NodeImage`）
- 普通链接（`NodeLinkDest` 非图片）
- TextMark 超链接（`IsTextMarkType("a")`）
- 音频/视频（`NodeAudio` / `NodeVideo`）
- 属性视图资源字段（`NodeAttributeView` + `MAsset`）

**去重特点**：
- 仅在单次 `NetAssets2LocalAssets()` 调用内有效
- 跨多次调用同一 URL 会被重复下载（无持久化 URL 去重）
- 同一份文件在不同 URL 下不会被识别为重复（无内容哈希校验）
- **音频/视频特殊问题**：
  - 去重键为 `src` 属性的 URL，`data-src` 中的不同 URL 不会去重
  - `//` 前缀的 URL 由于规范化 Bug，`oldDest` 被规范化为 `https://...`，但 `assetsMap` 键也为 `https://...`，理论上可命中去重；但由于 `Tokens` 仍为 `//...`，`setAssetsLinkDest` 中的 `bytes.ReplaceAll` 无法匹配，**导致去重命中但替换失败**

#### 4.3.4 缓存一致性保证

- 写入后立即更新缓存
- 文件系统事件监听实时同步
- 缓存命中后验证文件存在性
- 不存在的缓存条目自动清理

### 4.4 安全内容过滤

#### 4.4.1 XSS 防护层次

1. **Lute Sanitize**：HTML 标签白名单过滤
2. **文件名过滤**：移除特殊字符，防止 Markdown/HTML 注入
3. **路径安全**：工作区路径校验，防止目录遍历
4. **表情名称过滤**：防止通过自定义表情名注入 XSS

#### 4.4.2 文件系统安全

| 风险 | 防护措施 |
|------|---------|
| 路径遍历 | 过滤 `../`、`\`、`/` 等字符 |
| 符号链接 | `IsSymlinkPath()` 检测，`WalkWithSymlinks()` 循环检测 |
| 敏感路径访问 | `IsSensitivePath()` 黑名单检查 |
| 文件系统限制 | 文件名长度截断（189 字节） |
| 不可见字符 | `RemoveInvalid()` 清理 |

---

## 五、潜在风险与问题

### 5.1 安全风险

1. **SSRF 绕过可能**：
   - IPv6 地址可能绕过 `isPrivateIP()` 检测（当前仅处理 IPv4）
   - DNS 重绑定攻击时间窗口（DNS 解析与连接建立之间）

2. **XSS 注入面**：
   - 自定义属性（`custom-*`）可能被滥用
   - 数学公式、图表渲染器的安全边界需关注

3. **资源文件攻击面**：
   - SVG 文件可能包含脚本
   - HTML 附件中的脚本执行
   - 文件名中的 Unicode 隐藏字符

4. **网络资源本地化风险**：
   - 下载的资源文件未进行病毒扫描
   - 文件名虽然经过过滤，但内容类型未做严格校验（可伪装扩展名）

### 5.2 性能风险

1. **大文件处理**：
   - 大型 HTML 剪藏内容的内存占用
   - Base64 图片解码的 CPU 消耗
   - 资源哈希计算的 I/O 开销

2. **缓存一致性**：
   - 文件系统事件可能丢失
   - 并发写入时的缓存竞态条件

3. **网络依赖**：
   - 链滴文章直连失败时的回退逻辑
   - 网络图片本地化的超时处理（单个文件默认 HTTP 客户端超时）

### 5.3 功能完整性

1. **剪藏能力边界**：
   - 动态加载内容（JavaScript 渲染）无法捕获
   - 登录态内容无法访问
   - 反爬站点的验证码拦截

2. **格式保真度**：
   - 复杂 CSS 样式丢失
   - 交互式组件失效
   - 多栏布局退化

3. **错误处理完备性**：
   - 网络资源本地化中，除防盗链（403/401）外其他失败**无用户提示**，仅日志记录
   - 文件名被 `FilterUploadFileName` 过滤重命名后用户无感知
   - `onlyImg` 模式下仅图片被替换，普通链接/音视频/属性视图文件仍为远程链接，用户可能误判
   - 属性视图资源字段替换后需单独 `av.SaveAttributeView()` 持久化，失败无独立提示
   - **音视频 `//` 前缀 URL 完全静默失败**（Bug 1）：无提示、无日志、无法被替换

4. **去重机制覆盖不足**：
   - 网络资源本地化仅同批次 URL 去重，跨多次调用重复下载同一 URL
   - 同内容不同 URL 的资源不会被去重（无内容哈希校验）
   - 上传与本地化两套去重机制互不相通

5. **音视频本地化不彻底（已确认 Bug）**：
   - **Bug 1**：`//` 前缀的音视频 URL 永远无法被替换（规范化字段错误）
   - **Bug 2**：`data-src` 属性中的不同 URL 不会被提取和替换，可能仍从远程加载原始文件
   - **Bug 3**：`TextMarkAHref` 未被同步更新，存在字段不一致风险

### 5.4 错误处理缺陷

1. **部分失败处理**：
   - 批量图片下载中单张失败的处理策略
   - 剪藏部分成功时用户提示不足

2. **调试信息不足**：
   - 网络错误细节未完全暴露给用户
   - 转换失败原因难以定位

---

## 六、后续追查问题

### 6.1 待深入验证

1. **IPv6 SSRF 防护**：`isPrivateIP()` 是否覆盖了 IPv6 私有地址？
2. **SVG 安全**：上传的 SVG 文件是否会在预览时执行脚本？
3. **数学公式安全**：MathJax/KaTeX 渲染是否存在 XSS 风险？
4. **并发安全**：资源缓存的 `sync.Mutex` 是否足以应对高并发上传场景？
5. **网络资源下载超时**：`NewCustomReqClient()` 是否配置了合理的超时时间？
6. **属性视图资源替换原子性**：`av.SaveAttributeView()` 失败时是否会导致文档树与属性视图不一致？
7. **NodeAudio/NodeVideo 的 TextMarkAHref 使用场景**：该字段在哪些流程中会被赋值？（目前创建时未设置）
8. **本地文件链接识别**：`FileURLToLocalPath()` 的判定逻辑是否覆盖所有本地路径格式（Windows/Unix）？
9. **音视频 `data-src` 的实际用途**：前端播放时优先使用 `src` 还是 `data-src`？两者分别何时被更新？

### 6.2 优化方向

1. **增量去重**：当前网络资源本地化仅同批次 URL 去重，是否可增加跨批次基于 URL 的持久化去重？
2. **断点续传**：大文件/网络资源下载的断点续传支持
3. **预览机制**：剪藏内容插入前的预览与编辑
4. **模板支持**：剪藏内容的格式化模板选择
5. **失败详情**：网络资源本地化中，将单文件失败原因汇总展示给用户
6. **两类去重机制融合**：网络资源本地化完成后是否应写入内容哈希缓存，避免后续上传时重复保存？
7. **onlyImg 模式提示**：仅图片模式下，对未处理的链接/音视频/资源字段给予用户提示
8. **失败计数统计**：补充总失败数、按错误类型分类统计
9. **Bug 修复 - 音视频 `//` 前缀规范化**：将 `setAssetsLinkDest()` 中 `TextMarkAHref` 的规范化移至 `Tokens`，或在替换前先规范化 Tokens 中的 URL：
   ```go
   // 修复方案示例
   } else if ast.NodeAudio == node.Type || ast.NodeVideo == node.Type {
       // 先规范化 Tokens 中的 // 前缀
       if bytes.Contains(node.Tokens, []byte("src=\"//")) {
           node.Tokens = bytes.Replace(node.Tokens, []byte("src=\"//"), []byte("src=\"https://"), 1)
       }
       if bytes.Contains(node.Tokens, []byte("data-src=\"//")) {
           node.Tokens = bytes.Replace(node.Tokens, []byte("data-src=\"//"), []byte("data-src=\"https://"), 1)
       }
       node.Tokens = bytes.ReplaceAll(node.Tokens, []byte(oldDest), []byte(dest))
   }
   ```
10. **Bug 修复 - 音视频 `data-src` 提取与替换**：修改 `getRemoteAssetsLinkDests()` 同时提取 `src` 和 `data-src`，并确保两者都被替换
11. **Bug 修复 - 音视频 `TextMarkAHref` 同步更新**：在 `setAssetsLinkDest()` 中同步更新 `TextMarkAHref`（如果存在）
12. **静默失败场景提示**：对音视频 `//` 前缀等静默失败场景增加日志或用户提示

---

## 七、关键代码索引

### API 端点

| 端点 | 方法 | 文件 | 功能 |
|------|------|------|------|
| `/api/extension/copy` | POST | [extension.go](kernel/api/extension.go) | 浏览器剪藏扩展内容接收 |
| `/api/network/forwardProxy` | POST | [network.go](kernel/api/network.go) | 安全 HTTP 转发代理 |
| `/api/lute/html2BlockDOM` | POST | [lute.go](kernel/api/lute.go) | HTML 转块 DOM |
| `/api/asset/upload` | POST | [upload.go](kernel/model/upload.go) | 资源文件上传 |
| `/api/format/netImg2LocalAssets` | POST | [format.go](kernel/api/format.go#L50-L73) | 网络图片本地化 |
| `/api/format/netAssets2LocalAssets` | POST | [format.go](kernel/api/format.go#L28-L48) | 网络资源本地化 |

### 核心函数

| 函数名 | 文件 | 行号 | 功能 |
|--------|------|------|------|
| `extensionCopy` | [extension.go](kernel/api/extension.go) | L43 | 剪藏主入口 |
| `forwardProxy` | [network.go](kernel/api/network.go) | L155 | 网络转发代理 |
| `getSafeClient` | [network.go](kernel/api/network.go) | L339 | 安全 HTTP 客户端创建 |
| `HTML2Tree` | [import.go](kernel/model/import.go) | L59 | HTML 转 Markdown AST |
| `Upload` | [upload.go](kernel/model/upload.go) | L131 | 文件上传处理 |
| `NetAssets2LocalAssets` | [assets.go](kernel/model/assets.go) | L229 | 网络资源本地化主入口 |
| `netAssets2LocalAssets0` | [assets.go](kernel/model/assets.go) | L254 | 网络资源下载与替换核心实现 |
| `getRemoteAssetsLinkDestsInTree` | [assets.go](kernel/model/assets.go) | L1623 | 遍历整棵树提取远程资源节点 |
| `getRemoteAssetsLinkDests` | [assets.go](kernel/model/assets.go) | L1542 | 从单节点提取远程资源 URL（五类资源） |
| `setAssetsLinkDest` | [assets.go](kernel/model/assets.go) | L1494 | 替换节点中的资源链接地址 |
| `GetAssetPathByHash` | [assets.go](kernel/model/assets.go) | L72 | 哈希查找资源路径 |
| `FilterUploadFileName` | [file.go](kernel/util/file.go) | L225 | 上传文件名过滤 |
| `IsAssetLinkDest` | [path.go](kernel/util/path.go) | L293 | 判断是否为本地资源链接 |
| `NewLute` | [lute.go](kernel/util/lute.go) | L50 | Lute 引擎初始化 |
| `PushMsg` | [websocket.go](kernel/util/websocket.go) | L230 | 普通提示推送 |
| `PushErrMsg` | [websocket.go](kernel/util/websocket.go) | L246 | 错误提示推送 |
| `PushUpdateMsg` | [websocket.go](kernel/util/websocket.go) | L226 | 更新已存在提示 |
| `PushClearMsg` | [websocket.go](kernel/util/websocket.go) | L319 | 关闭指定提示 |
| `BroadcastByType` | [websocket.go](kernel/util/websocket.go) | L82 | WebSocket 底层广播原语 |
| `paste` | [paste.ts](app/src/protyle/util/paste.ts) | L249 | 前端粘贴主函数 |
| `fetchPost` | [fetch.ts](app/src/util/fetch.ts) | L8 | 前端 API 请求封装 |
| `processMessage` | [processMessage.ts](app/src/util/processMessage.ts) | L10 | 前端 WebSocket/HTTP 消息统一处理 |
| `Conf.Language` | [conf.go](kernel/model/conf.go) | L986 | 多语言文案读取 |

### 语言包

| 文件 | 说明 |
|------|------|
| [zh_CN.json](app/appearance/langs/zh_CN.json#L1488-L1776) | 中文语言包 `_kernel` 节点（内核错误码映射） |
