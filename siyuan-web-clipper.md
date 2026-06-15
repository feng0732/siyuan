# SiYuan 剪藏与网页抓取功能实现分析

## 一、概述

SiYuan（思源笔记）的剪藏与网页抓取功能是一个多层次的内容处理系统，涵盖了从外部内容获取、清洗转换、资源归档到文档插入的完整链路。该功能主要由前端（TypeScript/Electron）和后端内核（Go）协同完成，通过 HTTP API 进行通信。

### 核心模块定位

| 模块 | 主要职责 | 关键文件 |
|------|---------|---------|
| 剪藏接入层 | 浏览器扩展内容接收、链滴文章直取 | [extension.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/extension.go) |
| 网络转发层 | 安全的 HTTP 请求转发、SSRF 防护 | [network.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/network.go) |
| 内容转换层 | HTML 转 Markdown、格式清洗 | [import.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/import.go)、[lute.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/util/lute.go) |
| 资源管理层 | 上传、去重、哈希缓存 | [upload.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/upload.go)、[asset.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/cache/asset.go) |
| 前端粘贴层 | 剪贴板解析、内容分发、插入渲染 | [paste.ts](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/app/src/protyle/util/paste.ts) |
| 安全过滤层 | XSS 防护、文件名过滤、路径安全 | [file.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/util/file.go) |

---

## 二、关键流程分析

### 2.1 外部内容获取

#### 2.1.1 剪藏扩展接入（`/api/extension/copy`）

**入口函数**：`extensionCopy()` in [extension.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/extension.go#L43-L291)

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

**入口函数**：`forwardProxy()` in [network.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/network.go#L155-L331)

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

**核心函数**：`HTML2Tree()` in [import.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/import.go#L59-L108)

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

**前端层面**（[paste.ts](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/app/src/protyle/util/paste.ts)）：

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

**后端层面**（[extension.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/extension.go)）：

1. **行首空白剔除**：段落首节点的行首空白字符移除
2. **iframe 换行清理**：正则移除 `<iframe>` 标签内的换行符，避免解析异常
3. **图片 alt/title 规范化**：
   - 检测并转换非文本格式的 alt 和 title
   - 使用 `parse.Inline()` 解析后提取纯文本
4. **TextMark 转 Inlines**：`parse.TextMarks2Inlines()` 统一行级标记格式
5. **嵌套行级扁平化**：`parse.NestedInlines2FlattedSpansHybrid()` 处理嵌套结构

#### 2.2.3 安全内容过滤（XSS 防护）

**Lute 引擎层面**：
- `SetSanitize(true)` in [lute.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/util/lute.go#L81)
- 启用 Lute 内置的 HTML 消毒功能
- 前端也调用 `Lute.Sanitize(textHTML)` 进行预处理

**文件名安全过滤**（[file.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/util/file.go#L225-L247)）：

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

**核心函数**：`Upload()` in [upload.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/upload.go#L131-L353)

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

1. **哈希计算**：使用 ETag 算法计算文件内容哈希（[etag.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/util/etag.go)）

2. **缓存查询**（[asset.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/cache/asset.go)）：
   - 内存缓存：`assetHashCache`（map 结构，sync.Mutex 保护）
   - 数据库查询：`sql.QueryAssetByHash()` 作为后备
   - 存在性验证：缓存命中后验证文件实际存在

3. **同名但不同文件处理**：
   - 哈希相同但文件名不同 → 使用随机哈希前缀强制重新保存
   - 防止文件名冲突导致的资源误引用

4. **特殊场景跳过重复**：
   - PDF 标注图片上传支持 `skipIfDuplicated` 参数
   - 通过文件名模式匹配（Glob）寻找已有文件

**缓存同步机制**：
- 文件系统事件监听（assets_watcher）
- `HandleAssetsChangeEvent()` / `HandleAssetsRemoveEvent()` 实时更新缓存
- 启动时全量加载：`LoadAssets()`

#### 2.3.3 Base64 图片处理

**处理函数**：`processBase64Img()` in [import.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/import.go#L1250-L1328)

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

**核心函数**：`paste()` in [paste.ts](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/app/src/protyle/util/paste.ts#L249-L640)

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

**API 端点**：`/api/lute/html2BlockDOM` in [lute.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/lute.go#L78-L202)

**处理流程**：
1. 调用 `model.HTML2Tree()` 转换为 Markdown AST
2. 空列表项/空引用块清理
3. 单单元格表格转换为段落
4. 容器模式下本地资源文件复制
5. `TextMarks2Inlines` + `NestedInlines2FlattedSpansHybrid` 规范化
6. 格式化为 Markdown 后再解析为 Protyle DOM

#### 2.4.3 网络图片/资源本地化

**API 端点**：
- `/api/format/netImg2LocalAssets`：仅图片本地化
- `/api/format/netAssets2LocalAssets`：所有网络资源本地化

**实现函数**：`NetAssets2LocalAssets(id, imgOnly, url)` in [assets.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/assets.go)

### 2.5 失败提示

#### 2.5.1 网络异常处理

**转发代理异常**（[network.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/network.go)）：

| 错误场景 | 错误码 | 提示信息 |
|---------|--------|---------|
| URL 格式无效 | -1 | `invalid [url]` |
| 非 http/https 协议 | -1 | `only http/https is allowed` |
| 请求发送失败 | -1 | `forward request failed: {err}` |
| 响应体读取失败 | -1 | `read response body failed: {err}` |
| Payload 解码失败 | -2 | `decode {encoding} payload failed: {err}` |

**前端请求异常**（[fetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/app/src/util/fetch.ts)）：

| HTTP 状态 | 处理方式 |
|----------|---------|
| 401 鉴权失败 | 3 秒后刷新页面 |
| 403/404 | 返回错误信息 |
| 网络错误 | 控制台警告，事务请求触发 kernelError |

**异常传播**：
- `processMessage()` 统一处理返回消息
- 失败回调 `failCallback` 支持自定义错误处理

#### 2.5.2 内容转换失败

- HTML 解析失败 → 日志记录，继续处理后续节点
- 图片解码失败 → 跳过该图片，保留原始链接
- Base64 解码失败 → 日志记录，返回原始内容

---

## 三、协作分工

### 3.1 前后端职责划分

| 层级 | 组件 | 主要职责 |
|------|------|---------|
| 前端表现层 | paste.ts | 剪贴板数据读取、格式判断、内容分发、渲染触发 |
| 前端工具层 | fetch.ts | API 请求封装、错误处理、状态码映射 |
| 后端 API 层 | extension.go | 剪藏请求接收、参数解析、响应组装 |
| 后端 API 层 | network.go | 网络请求转发、安全过滤、编码转换 |
| 后端模型层 | import.go | HTML 转 Markdown、AST 处理、Base64 图片 |
| 后端模型层 | upload.go | 文件上传、资源去重、目录管理 |
| 后端模型层 | assets.go | 资源管理、网络资源本地化 |
| 后端缓存层 | cache/asset.go | 资源哈希缓存、加速去重查询 |
| 后端工具层 | file.go | 文件名过滤、路径安全、文件操作 |
| 后端工具层 | lute.go | Lute 引擎配置、Markdown 环境初始化 |

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
          返回前端
                ↓
    前端 paste 处理 → insertHTML → 渲染更新
```

---

## 四、特别关注机制详解

### 4.1 网络异常处理

#### 4.1.1 SSRF 防护体系

**三层防护**（[network.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/network.go#L333-L361)）：

1. **协议白名单**：仅允许 http/https
2. **IP 地址校验**：`isPrivateIP()` 过滤私有地址
   - 回环地址 `127.0.0.0/8`
   - 链路本地地址 `169.254.0.0/16`
   - 私有网络 `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
   - 未指定地址 `0.0.0.0`
3. **DNS 重绑定防御**：在 `Control` 钩子中验证解析后的 IP

#### 4.1.2 超时与重试

- 默认超时：7000ms（转发代理）
- 网络连通性检测：2 次重试，间隔 1 秒（`isOnline()` in [net.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/util/net.go#L159-L190)）
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

### 4.3 重复内容识别

#### 4.3.1 资源去重机制

```
文件上传
    ↓
计算内容哈希（ETag）
    ↓
查询内存缓存（assetHashCache）
    ├─ 命中 → 验证文件存在 → 返回已有路径
    └─ 未命中 → 查询 SQL 数据库
          ├─ 命中 → 写入缓存 → 返回已有路径
          └─ 未命中 → 写入新文件 → 更新缓存
```

**注意事项**：
- 哈希相同但文件名不同 → 强制重新保存（避免文件名误导）
- 空文件 → 使用随机哈希（避免所有空文件冲突）
- PDF 标注场景 → 支持文件名模式匹配去重

#### 4.3.2 缓存一致性保证

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
   - 网络图片本地化的超时处理

### 5.3 功能完整性

1. **剪藏能力边界**：
   - 动态加载内容（JavaScript 渲染）无法捕获
   - 登录态内容无法访问
   - 反爬站点的验证码拦截

2. **格式保真度**：
   - 复杂 CSS 样式丢失
   - 交互式组件失效
   - 多栏布局退化

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

### 6.2 待补充分析

1. **网络资源本地化完整流程**：`NetAssets2LocalAssets` 的具体实现细节和错误处理
2. **插件扩展机制**：插件如何参与粘贴/剪藏流程（`paste` 事件）
3. **移动端剪藏**：移动端是否有特殊的剪藏实现路径
4. **错误码体系**：完整的错误码映射表和用户提示策略

### 6.3 优化方向

1. **增量去重**：当前基于内容哈希，是否可增加基于 URL 的去重？
2. **断点续传**：大文件/网络资源下载的断点续传支持
3. **预览机制**：剪藏内容插入前的预览与编辑
4. **模板支持**：剪藏内容的格式化模板选择

---

## 七、关键代码索引

### API 端点

| 端点 | 方法 | 文件 | 功能 |
|------|------|------|------|
| `/api/extension/copy` | POST | [extension.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/extension.go) | 浏览器剪藏扩展内容接收 |
| `/api/network/forwardProxy` | POST | [network.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/network.go) | 安全 HTTP 转发代理 |
| `/api/lute/html2BlockDOM` | POST | [lute.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/lute.go) | HTML 转块 DOM |
| `/api/asset/upload` | POST | [upload.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/upload.go) | 资源文件上传 |
| `/api/format/netImg2LocalAssets` | POST | [format.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/format.go) | 网络图片本地化 |
| `/api/format/netAssets2LocalAssets` | POST | [format.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/format.go) | 网络资源本地化 |

### 核心函数

| 函数名 | 文件 | 功能 |
|--------|------|------|
| `extensionCopy` | [extension.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/extension.go#L43) | 剪藏主入口 |
| `forwardProxy` | [network.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/network.go#L155) | 网络转发代理 |
| `getSafeClient` | [network.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/api/network.go#L339) | 安全 HTTP 客户端创建 |
| `HTML2Tree` | [import.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/import.go#L59) | HTML 转 Markdown AST |
| `Upload` | [upload.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/upload.go#L131) | 文件上传处理 |
| `GetAssetPathByHash` | [assets.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/model/assets.go#L72) | 哈希查找资源路径 |
| `FilterUploadFileName` | [file.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/util/file.go#L225) | 上传文件名过滤 |
| `NewLute` | [lute.go](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/kernel/util/lute.go#L50) | Lute 引擎初始化 |
| `paste` | [paste.ts](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/app/src/protyle/util/paste.ts#L249) | 前端粘贴主函数 |
| `fetchPost` | [fetch.ts](file:///d:/fz/0601/solo-dogfeeding/code/306-siyuan/app/src/util/fetch.ts#L8) | 前端 API 请求封装 |
