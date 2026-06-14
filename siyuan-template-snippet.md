# SiYuan 模板与代码片段引擎实现原理分析

> **文档说明**：本文所有源码引用均使用 **仓库相对路径**（格式如 `kernel/model/template.go`），便于跨环境复核。在实际代码库中，可将相对路径前置仓库根目录即可定位到具体文件。关键代码段同时标注了行号范围，便于 IDE/grep 快速跳转。

---

## 1. 系统架构概述

SiYuan（思源笔记）的模板与代码片段引擎是一个分层设计的系统，由后端核心引擎、API 接口层和前端交互层协同工作。该系统基于 Go 标准库 `text/template` 构建，通过自定义分隔符、函数注入和上下文管理，实现了强大的模板渲染能力。

### 1.1 核心模块组成

| 模块 | 主要职责 | 核心文件（仓库相对路径） |
|------|---------|------------------------|
| 模板引擎核心 | 模板解析、变量替换、函数执行 | `kernel/model/template.go` |
| 内置模板函数 | 通用工具函数、日期处理、统计函数 | `kernel/filesys/template.go` |
| SQL 模板函数 | 数据库查询相关模板函数 | `kernel/sql/database.go`（函数注册：第 1591–1621 行） |
| SQL 参数化执行 | SQL 语法解析、查询限制、行扫描 | `kernel/sql/block_query.go` |
| 代码片段管理 | CSS/JS 片段的加载、渲染、持久化 | `kernel/model/snippet.go` |
| API 接口层 | HTTP 接口定义与权限控制 | `kernel/api/template.go`, `kernel/api/snippet.go` |
| 角色与权限模型 | 角色定义、只读/管理员权限校验 | `kernel/model/role.go`, `kernel/model/session.go` |
| 前端交互层 | 模板选择、预览、插入渲染 | `app/src/protyle/hint/extend.ts`, `app/src/config/util/snippets.ts` |
| 内容插入层 | DOM 操作、事务处理、块更新 | `app/src/protyle/util/insertHTML.ts` |

### 1.2 跨环境复核注意事项（新增）

源码引用若使用本机绝对路径（如 `file:///d:/fz/...`），存在以下问题：

| 问题类型 | 影响范围 | 严重程度 |
|---------|---------|---------|
| 路径不可迁移 | Linux / macOS 开发者无法直接使用 `file://` 跳转 | 中 |
| Windows 盘符差异 | 其他机器上仓库路径可能在 `C:` / `E:` | 中 |
| 仓库结构改变 | 目录重命名 / 文件移动后绝对链接全部失效 | 高 |
| 安全信息泄露 | 暴露本机用户名 / 项目目录命名 | 低 |
| 代码审查可读性 | Reviewer 无法快速定位文件在仓库中的层级 | 中 |

**修正方案**：本文统一采用 **仓库相对路径 + 行号范围** 的标注方式（如 `kernel/sql/database.go#L1591-L1621`），并在第 12 章提供检索指引。

---

## 2. 模板引擎核心实现原理

### 2.1 模板解析流程

模板解析采用 Go 标准库 `text/template`，但使用了自定义的分隔符和函数扩展机制。

#### 2.1.1 双模式解析系统

SiYuan 实现了两种模板解析模式：

**模式一：标准 Go 模板语法（`{{ }}` 分隔符）**
```go
// 定义位置：kernel/model/template.go，函数 RenderGoTemplate
func RenderGoTemplate(templateContent string) (ret string, err error) {
    tmpl := template.New("")
    tplFuncMap := filesys.BuiltInTemplateFuncs()
    sql.SQLTemplateFuncs(&tplFuncMap)
    tmpl = tmpl.Funcs(tplFuncMap)
    tpl, err := tmpl.Parse(templateContent)
    // ...
    err = tpl.Execute(buf, nil)  // 无数据上下文
}
```
> 代码位置：`kernel/model/template.go`（RenderGoTemplate 函数，约第 51–69 行）

**模式二：SiYuan 动作模板（`.action{ }` 分隔符）**
```go
// 定义位置：kernel/model/template.go，函数 RenderTemplate
func RenderTemplate(p, id string, preview bool) (tree *parse.Tree, dom string, err error) {
    // ... 构建 dataModel
    goTpl := template.New("").Delims(".action{", "}")  // 自定义分隔符
    tplFuncMap := filesys.BuiltInTemplateFuncs()
    sql.SQLTemplateFuncs(&tplFuncMap)
    goTpl = goTpl.Funcs(tplFuncMap)
    tpl, err := goTpl.Funcs(tplFuncMap).Parse(gulu.Str.FromBytes(md))
    // ...
    err = tpl.Execute(buf, dataModel)  // 注入数据上下文
}
```
> 代码位置：`kernel/model/template.go`（RenderTemplate 函数，约第 309–499 行）

> **设计意图**：使用 `.action{ }` 作为分隔符可以避免与 Markdown 中的其他语法冲突，特别是在文档模板中。

### 2.2 变量替换与上下文注入

#### 2.2.1 数据模型（Data Model）

模板渲染时注入的上下文变量经过严格限制，仅包含 4 个字符串字段：

```go
dataModel := map[string]string{
    "title": titleVar,    // 文档标题或块内容（经过 strings.TrimSpace 清理）
    "id":    block.ID,    // 块 ID（23 位固定格式）
    "name":  block.Name,  // 块名称
    "alias": block.Alias, // 块别名
}
```
> 代码位置：`kernel/model/template.go`（RenderTemplate 内，约第 326–337 行）

> **作用域限制设计**：只注入 `map[string]string` 而非结构体，避免通过反射暴露更多字段。`title` 经过空白清理，降低模板变量直接引发注入的概率。

#### 2.2.2 模板函数注入

模板函数通过两层注入机制构建：

**第一层：Sprig 通用函数库 + 自定义函数**
```go
// 定义位置：kernel/filesys/template.go，函数 BuiltInTemplateFuncs
func BuiltInTemplateFuncs() (ret template.FuncMap) {
    ret = sprig.TxtFuncMap()

    // 安全移除危险函数（环境信息泄露类）
    delete(ret, "env")
    delete(ret, "expandenv")
    delete(ret, "getHostByName")

    // 日期处理
    ret["Weekday"] = util.Weekday
    ret["WeekdayCN"] = util.WeekdayCN
    ret["ISOWeek"] = util.ISOWeek
    ret["ISOYear"] = util.ISOYear

    // 数学运算
    ret["pow"] = pow
    ret["log"] = log

    // 内容处理（返回字符串，可继续参与变量替换）
    ret["getHPathByID"] = getHPathByID
    ret["statBlock"] = StatBlock
    ret["markdown2text"] = markdown2text
    ret["markdown2content"] = markdown2content
    // ...
}
```
> 代码位置：`kernel/filesys/template.go`（BuiltInTemplateFuncs 函数，约第 35–63 行）

**第二层：SQL 查询函数**
```go
// 定义位置：kernel/sql/database.go，函数 SQLTemplateFuncs
func SQLTemplateFuncs(templateFuncMap *template.FuncMap) {
    (*templateFuncMap)["queryBlocks"] = func(stmt string, args ...string) (retBlocks []*Block) {
        for _, arg := range args {
            stmt = strings.Replace(stmt, "?", arg, 1)  // ⚠ 字符串拼接式参数替换
        }
        retBlocks = SelectBlocksRawStmt(stmt, 1, 512)   // 限制：每页 512 条
        return
    }
    (*templateFuncMap)["getBlock"] = func(arg any) (retBlock *Block) { /* ... */ }
    (*templateFuncMap)["querySpans"] = func(stmt string, args ...string) (retSpans []*Span) {
        for _, arg := range args {
            stmt = strings.Replace(stmt, "?", arg, 1)  // ⚠ 字符串拼接式参数替换
        }
        retSpans = SelectSpansRawStmt(stmt, 512)        // 限制：512 条
        return
    }
    (*templateFuncMap)["querySQL"] = func(stmt string) (ret []map[string]any) {
        ret, _ = Query(stmt, 1024)                     // ⚠ 限制：1024 条（与前两者不同）
        return
    }
}
```
> 代码位置：`kernel/sql/database.go`（SQLTemplateFuncs 函数，第 1591–1621 行）

### 2.3 SQL 模板参数处理与返回限制详解（重点修正）

这是模板引擎安全设计中**最关键也最容易被忽略**的环节，本次重点核实了 **返回行数限制（512 vs 1024）** 的实际差异。

#### 2.3.1 三个 SQL 模板函数的返回限制对比（核实结果）

| 模板函数 | 底层调用 | 限制参数 | 实际上限 | 代码位置 |
|---------|---------|---------|---------|---------|
| `queryBlocks` | `SelectBlocksRawStmt(stmt, 1, 512)` | page=1, limit=512 | **512 条/页** | `kernel/sql/database.go#L1596` |
| `querySpans` | `SelectSpansRawStmt(stmt, 512)` | limit=512 | **512 条** | `kernel/sql/database.go#L1614` |
| `querySQL` | `Query(stmt, 1024)` | limit=1024 | **1024 条** | `kernel/sql/database.go#L1618` |

> **非模板函数场景对比**：
> - `kernel/model/search.go#L226` 使用 `SelectBlocksRawStmtNoParse(stmt, 1024)`（内部嵌入块查询）
> - `kernel/model/index.go#L327` 使用 `SelectBlocksRawStmtNoParse(stmt, 102400)`（索引构建，超大值）

**设计意图推断**：
- `queryBlocks` / `querySpans` 返回强类型结构体（`*Block` / `*Span`），每个对象字段较多，512 条足以覆盖常见模板场景。
- `querySQL` 返回 `map[string]any` 灵活结构，用于复杂聚合场景，因此放宽到 1024。
- 1024 / 102400 仅用于内核内部搜索和索引，不作为模板 API 暴露。

#### 2.3.2 伪参数化：字符串替换的实现细节

`queryBlocks` 和 `querySpans` 采用了**模拟参数化**而非真实的 SQL 预编译：

```go
for _, arg := range args {
    stmt = strings.Replace(stmt, "?", arg, 1)  // 每次只替换第一个问号
}
```

这意味着：
- **替换顺序敏感**：`args` 的顺序必须与 SQL 中 `?` 出现的顺序严格一致。
- **无转义处理**：`arg` 中的单引号、反斜杠等特殊字符不会被转义。
- **无法区分字面量问号与占位符**：如果 SQL 字面量中包含 `?`，会被误替换。
- **数量不匹配无告警**：`?` 数与 `args` 数不一致时不会报错，多余 `?` 保留原样。

**风险传递路径**：
```
模板作者在 .action{ range queryBlocks "SELECT * FROM blocks WHERE content LIKE '%?%'" .title }
                                    ↓
                .title 可能是 "abc' OR '1'='1"
                                    ↓
        strings.Replace 后得到带有注入语句的完整 SQL
                                    ↓
                SelectBlocksRawStmt 执行（仅做 SELECT 语法校验 + LIMIT 512）
                                    ↓
         返回结果被模板 range 循环渲染为 Markdown → HTML → 插入 DOM
```

#### 2.3.3 SQL 语法解析层的防护作用

虽然参数替换不安全，但 SQL 执行前会经过 `sqlparser` 的语法解析，这是**核心防护**：

```go
// 定义位置：kernel/sql/block_query.go，函数 SelectBlocksRawStmt
func SelectBlocksRawStmt(stmt string, page, limit int) (ret []*Block) {
    parsedStmt, err := sqlparser.Parse(stmt)
    if err != nil {
        return selectBlocksRawStmt(stmt, limit)  // 解析失败时回退到无解析执行
    }

    switch parsedStmt.(type) {
    case *sqlparser.Select:
        slct := parsedStmt.(*sqlparser.Select)
        // 强制注入 LIMIT 子句
        slct.Limit = &sqlparser.Limit{
            Rowcount: &sqlparser.SQLVal{Val: []byte(strconv.Itoa(limit))},
            Offset:   &sqlparser.SQLVal{Val: []byte(strconv.Itoa((page - 1) * limit))},
        }
        stmt = sqlparser.String(slct)
    case *sqlparser.Union:
        union := parsedStmt.(*sqlparser.Union)
        // 同样强制注入 LIMIT
        union.Limit = &sqlparser.Limit{ /* ... */ }
        stmt = sqlparser.String(union)
    default:
        return  // ❌ 非 SELECT/UNION 直接拒绝（DROP/DELETE/INSERT/UPDATE 被拦截）
    }

    // 转义修正（应对解析器再序列化后的特殊字符问题）
    stmt = strings.ReplaceAll(stmt, "\\'", "''")
    stmt = strings.ReplaceAll(stmt, "\\\"", "\"")
    stmt = strings.ReplaceAll(stmt, "\\\\*", "\\*")
    stmt = strings.ReplaceAll(stmt, "from dual", "")

    rows, err := query(stmt)  // 直接执行
    // ...
}
```
> 代码位置：`kernel/sql/block_query.go`（SelectBlocksRawStmt 函数，约第 551–644 行）

**防护能力分析**：

| 攻击类型 | 是否被拦截 | 说明 |
|---------|-----------|------|
| DML 语句（DROP/DELETE/INSERT） | ✅ 是 | 语法解析非 SELECT/UNION 直接返回空 |
| DDL 语句（CREATE/ALTER） | ✅ 是 | 同样被类型断言拦截 |
| UNION 注入 | ⚠️ 部分 | UNION 本身被允许，但整体仍被 LIMIT 限制 |
| 时间盲注 | ⚠️ 否 | 可通过 `LIKE` + 子查询 + SQLite 函数构造 |
| 报错注入 | ⚠️ 部分 | 语法错误静默返回空，难以利用错误信息 |
| 批量数据导出 | ⚠️ 限制 | queryBlocks 限 512，querySpans 限 512，querySQL 限 1024 |

**`querySQL` 的双解析器策略**：
```go
// 定义位置：kernel/sql/block_query.go，函数 Query
func Query(stmt string, limit int) (ret []map[string]any, err error) {
    originalStmt := stmt
    // ① 先用支持 || 连接符的解析器（sqlparser2）
    p := sqlparser2.NewParser(strings.NewReader(stmt))
    parsedStmt2, err := p.ParseStatement()
    if err != nil {
        if !strings.Contains(stmt, "||") {
            // ② 回退到支持 UNION 的解析器（sqlparser）
            parsedStmt, err2 := sqlparser.Parse(stmt)
            // ... 同样进行类型断言 + LIMIT 注入
        }
    }
    // ③ 解析失败则直接执行（回退路径）
    rows, err := query(stmt)
    if err != nil {
        rows, err = query(originalStmt + " LIMIT " + strconv.Itoa(limit))  // 最后兜底
    }
    // ...
}
```
> 代码位置：`kernel/sql/block_query.go`（Query 函数，约第 366–445 行）

> **隐患点**：当两个解析器都失败时，代码会**跳过类型断言直接执行 SQL**。如果攻击者构造出能绕过解析器但 SQLite 仍能执行的语句，可能突破 SELECT 限制。

### 2.4 完整渲染流程

```
用户触发模板插入（斜杠菜单 / 插入按钮）
        ↓
前端调用 /api/template/render  [需 CheckAuth + CheckAdminRole + CheckReadonly]
        ↓
┌─────────────────────────────────┐
│  1. 加载目标块上下文            │
│     - LoadTreeByBlockID(id)     │
│     - 验证 IsAbsPathInWorkspace │
│     - 构建 dataModel (4 字段)   │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  2. 模板解析                    │
│     - 读取模板文件内容          │
│     - 创建 template 实例        │
│     - 注入 BuiltInTemplateFuncs │
│     - 注入 SQLTemplateFuncs     │
│     - Parse 模板内容            │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  3. 变量替换与函数执行          │
│     - Execute(dataModel)        │
│     - ⚠ SQL 函数执行（3 种限制）│
│       queryBlocks → LIMIT 512   │
│       querySpans  → LIMIT 512   │
│       querySQL    → LIMIT 1024  │
│       - strings.Replace 替换 ?  │
│       - sqlparser 语法校验      │
│       - 强制 LIMIT/OFFSET 注入  │
│       - db.Query 直接执行       │
│     - 渲染为 Markdown 文本      │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  4. 后处理（AST 遍历）          │
│     - GenBlockIDs() 重新生成ID  │
│     - TranscludeRef() 处理引用  │
│     - ProcessDatabaseView()     │
│     - 处理折叠标题              │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  5. 转换为 Block DOM            │
│     - lute.Tree2BlockDOM()      │
└─────────────────────────────────┘
        ↓
前端接收 content HTML
        ↓
insertHTML 插入（行级 / 块级事务）
        ↓
事务提交 → 持久化到 blocks 表 → 重建索引
```

---

## 3. 内容插入机制

### 3.1 前端渲染与插入流程

模板渲染完成后，前端通过 `hintRenderTemplate` 处理插入：

```typescript
// 定义位置：app/src/protyle/hint/extend.ts，函数 hintRenderTemplate
export const hintRenderTemplate = (value: string, protyle: IProtyle, nodeElement: Element) => {
    fetchPost("/api/template/render", {
        id: protyle.block.parentID,
        path: value
    }, (response) => {
        focusByRange(protyle.toolbar.range);
        const editElement = getContenteditableElement(nodeElement);
        if (editElement && editElement.textContent.trim() === "") {
            insertHTML(response.data.content, protyle, true);   // 空行 → 块级插入
        } else {
            insertHTML(response.data.content, protyle);        // 非空 → 行级插入
        }
        // 后渲染管线
        blockRender(protyle, protyle.wysiwyg.element);
        processRender(protyle.wysiwyg.element);
        highlightRender(protyle.wysiwyg.element);
        avRender(protyle.wysiwyg.element, protyle);
    });
};
```
> 代码位置：`app/src/protyle/hint/extend.ts`（hintRenderTemplate 函数，约第 542–564 行）

### 3.2 insertHTML 核心逻辑

`insertHTML` 是内容插入的核心，处理多种场景：

```typescript
// 定义位置：app/src/protyle/util/insertHTML.ts，函数 insertHTML
export const insertHTML = (html: string, protyle: IProtyle, isBlock = false,
                           useProtyleRange = false, insertByCursor = false) => {
    // 1. 获取编辑范围
    const range = useProtyleRange ? protyle.toolbar.range : getEditorRange(protyle.wysiwyg.element);

    // 2. 特殊场景分发
    //    2.1 数据库单元格 → processAV
    //    2.2 表格单元格批量填充 → processTable
    //    2.3 代码块 → 保留换行符的纯文本插入

    // 3. 行级插入
    if (!isBlock) {
        range.insertNode(tempElement.content.cloneNode(true));
        input(protyle, blockElement, range);  // 触发输入事件，更新 IAL
        return;
    }

    // 4. 块级插入：构造事务
    const doOperation: IOperation[] = [];
    const undoOperation: IOperation[] = [];
    // 遍历插入块的 DOM 子节点，构建 insert/update 操作
    transaction(protyle, doOperation, undoOperation);
};
```
> 代码位置：`app/src/protyle/util/insertHTML.ts`（insertHTML 函数，约第 264–590 行）

### 3.3 权限与 SQL 处理对内容插入链路的影响

模板渲染到内容插入之间存在**三段数据传递**，SQL 返回限制（512 / 1024）直接影响每一段的数据量上限：

```
┌──────────────────────┐    ┌──────────────────────┐    ┌──────────────────────┐
│  模板 Execute 阶段   │ →  │  AST 后处理阶段      │ →  │ insertHTML 阶段      │
│  (Go 后端)           │    │  (lute 引擎)         │    │  (前端 DOM)          │
└──────────────────────┘    └──────────────────────┘    └──────────────────────┘
         │                           │                           │
         ▼                           ▼                           ▼
 queryBlocks: ≤512 个 *Block     HTML 实体转义规则            insertAdjacentHTML
 querySpans : ≤512 个 *Span     与 标签白名单                / range.insertNode
 querySQL   : ≤1024 个 map
 Content / Markdown 字段                                        │
         │                           │                           │
         └────── 未经过滤的用户数据可沿此链路传递 ────────────────┘
```

**关键风险节点**：

| 链路节点 | 风险来源 | 数据量上限 | 影响 |
|---------|---------|-----------|------|
| 模板变量 → SQL | `{{ queryBlocks "..." .title }}` | — | `.title` 中的特殊字符进入 SQL |
| SQL 结果 → Markdown | `Block.Content` / `Block.Markdown` | queryBlocks:512<br>querySQL:1024 | 返回内容包含原始 HTML 或 Markdown 注入 |
| Markdown → HTML（lute） | 不受信任的 Markdown 语法 | 取决于内容长度 | lute 解析生成 DOM（需依赖其 XSS 防护） |
| HTML → 前端 DOM | `insertHTML` 直接插入 `response.data.content` | 取决于 HTML 体积 | 若后端转义不完整，触发存储型 XSS |
| 事务提交 → 数据库 | `action: "insert"` 的 data 字段 | 块级大小限制 | 恶意内容被持久化，后续所有打开该文档的用户受影响 |

**结论**：
- `querySQL` 的 1024 上限比 `queryBlocks` 的 512 高出一倍，意味着**单次注入可携带的数据量更大**，在存储型 XSS 或数据窃取场景中风险更高。
- SQL 模板函数的输出直接进入渲染流程，是**变量替换 → 上下文注入 → 内容插入**整个链路中**数据量最大、来源最不可控**的入口。

---

## 4. 代码片段（Snippet）引擎

### 4.1 代码片段数据结构

```go
// 定义位置：kernel/conf/snippet.go
type Snippet struct {
    ID                string `json:"id"`
    Name              string `json:"name"`
    Type              string `json:"type"`              // "js" 或 "css"
    Enabled           bool   `json:"enabled"`           // 是否启用
    DisabledInPublish bool   `json:"disabledInPublish"` // 发布模式下是否禁用
    Content           string `json:"content"`           // 完整内容
}
```
> 代码位置：`kernel/conf/snippet.go`（Snippet 结构体，约第 31–38 行）

### 4.2 读取权限深度分析

#### 4.2.1 API 路由权限矩阵

```go
// 定义位置：kernel/api/router.go
ginServer.Handle("POST", "/api/snippet/getSnippet",
    model.CheckAuth,            // ✅ 只需要登录（Admin/Editor/Reader 均通过）
    getSnippet)

ginServer.Handle("POST", "/api/snippet/setSnippet",
    model.CheckAuth,            // ① 登录
    model.CheckAdminRole,       // ② 管理员角色（❌ Reader/Editor 无权限）
    model.CheckReadonly,        // ③ 非只读模式
    setSnippet)

ginServer.Handle("POST", "/api/snippet/removeSnippet",
    model.CheckAuth,
    model.CheckAdminRole,       // ❌ Reader/Editor 无权限
    model.CheckReadonly,
    removeSnippet)
```
> 代码位置：`kernel/api/router.go`（路由注册，约第 472–474 行）

#### 4.2.2 CheckAuth 可通过的角色范围

```go
// 定义位置：kernel/model/session.go，函数 CheckAuth
func CheckAuth(c *gin.Context) {
    // ① JWT 认证：接受 3 种角色
    if role := GetGinContextRole(c); IsValidRole(role, []Role{
        RoleAdministrator,  // 管理员
        RoleEditor,         // 编辑者
        RoleReader,         // 只读用户
    }) {
        c.Next()
        return
    }
    // ② API Token → RoleAdministrator
    // ③ Session 认证（localhost + 空授权码 → RoleAdministrator）
}
```
> 代码位置：`kernel/model/session.go`（CheckAuth 函数，约第 207–260 行）

#### 4.2.3 发布模式下的二次过滤

```go
// 定义位置：kernel/api/snippet.go，函数 getSnippet
func getSnippet(c *gin.Context) {
    // ... 解析请求参数 ...
    confSnippets, err := model.LoadSnippets()

    isPublish := model.IsReadOnlyRoleContext(c)  // Reader / Visitor
    var snippets []*conf.Snippet
    for _, s := range confSnippets {
        if isPublish && s.DisabledInPublish {
            continue  // 发布模式下，标记"发布时禁用"的片段被过滤
        }
        if "all" != typ && s.Type != typ { continue }
        if 2 != enabledArg && s.Enabled != enabled { continue }
        snippets = append(snippets, s)
    }
    // ... 返回
}
```
> 代码位置：`kernel/api/snippet.go`（getSnippet 函数，约第 63–76 行）

其中 `IsReadOnlyRoleContext` 判定逻辑：
```go
// 定义位置：kernel/model/role.go
func IsReadOnlyRole(role Role) bool {
    return IsValidRole(role, []Role{ RoleReader, RoleVisitor })
}
```
> 代码位置：`kernel/model/role.go`（IsReadOnlyRole 函数，约第 43–47 行）

#### 4.2.4 权限分析结论

| 角色 | 读取片段 `/getSnippet` | 修改片段 `/setSnippet` | 删除片段 `/removeSnippet` |
|------|----------------------|----------------------|------------------------|
| Administrator | ✅ 全部内容 | ✅ 是 | ✅ 是 |
| Editor | ✅ 全部内容（CheckAuth 通过） | ❌ 被 CheckAdminRole 拦截 | ❌ 被 CheckAdminRole 拦截 |
| Reader | ✅ 全部内容（但 `DisabledInPublish=true` 被过滤） | ❌ | ❌ |
| Visitor | ❌ 被 CheckAuth 拦截（CheckAuth 不接受 Visitor） | ❌ | ❌ |
| 未认证 / Token 错误 | ❌ 401 | ❌ | ❌ |

**关键发现**：
1. **Editor 角色可读不可改**：Editor 能下载所有 JS/CSS 片段内容，但无法修改。
2. **Reader 角色读取受 DisabledInPublish 限制**：发布环境中敏感片段可被标记为发布时禁用。
3. **Visitor 无任何 API 访问权**：必须至少是 Reader 级别。

### 4.3 代码片段加载与渲染

**后端加载逻辑**：
```go
// 定义位置：kernel/model/snippet.go，函数 loadSnippets
func loadSnippets() (ret []*conf.Snippet, err error) {
    confPath := filepath.Join(util.SnippetsPath, "conf.json")
    data, err := filelock.ReadFile(confPath)
    gulu.JSON.UnmarshalJSON(data, &ret)

    needRewrite := false
    for _, snippet := range ret {
        if "" == snippet.ID {
            snippet.ID = ast.NewNodeID()   // 自动补齐缺失 ID
            needRewrite = true
        }
    }
    if needRewrite { writeSnippetsConf(ret) }
    return
}
```
> 代码位置：`kernel/model/snippet.go`（loadSnippets 函数，约第 68–111 行）

**并发安全**：
```go
var snippetsLock = sync.Mutex{}

func SetSnippet(snippets []*conf.Snippet) (err error) {
    snippetsLock.Lock()
    defer snippetsLock.Unlock()
    err = writeSnippetsConf(snippets)
    return
}
```
> 代码位置：`kernel/model/snippet.go`（SetSnippet 函数 + 互斥锁，约第 32、54–60 行）

**前端渲染逻辑**：
```typescript
// 定义位置：app/src/config/util/snippets.ts，函数 renderSnippet
export const renderSnippet = () => {
    fetchPost("/api/snippet/getSnippet", {type: "all", enabled: 2}, (response) => {
        response.data.snippets.forEach((item: ISnippet) => {
            const id = `snippet${item.type === "css" ? "CSS" : "JS"}${item.id}`;
            let exitElement = document.getElementById(id);

            // 全局开关检查
            if ((!window.siyuan.config.snippet.enabledCSS && item.type === "css") ||
                (!window.siyuan.config.snippet.enabledJS && item.type === "js")) {
                exitElement?.remove();
                return;
            }
            // 单片段启用检查
            if (!item.enabled) {
                exitElement?.remove();
                return;
            }

            // 直接注入 DOM（⚠️ 完整信任内容）
            if (item.type === "css") {
                document.head.insertAdjacentHTML("beforeend",
                    `<style id="${id}">${item.content}</style>`);
            } else if (item.type === "js") {
                exitElement = document.createElement("script");
                exitElement.type = "text/javascript";
                exitElement.text = item.content;
                exitElement.id = id;
                document.head.appendChild(exitElement);
            }
        });
    });
};
```
> 代码位置：`app/src/config/util/snippets.ts`（renderSnippet 函数，约第 7–42 行）

**触发时机**：`renderSnippet` 在应用启动 `onGetConfig` 中尽早调用，确保片段在界面渲染前注入：
```typescript
renderSnippet();  // app/src/boot/onGetConfig.ts，约第 78 行
```

### 4.4 代码片段对变量替换、上下文注入、内容插入的影响

代码片段与模板引擎**分属两个独立系统**，但在运行时存在**间接耦合**：

```
┌───────────────────────────────────┐
│  模板渲染内容插入 DOM             │
│  (insertHTML → transaction)      │
└─────────────────┬─────────────────┘
                  │
                  ▼
         blockRender() 等后渲染钩子
                  │
                  ▼
┌───────────────────────────────────┐    ┌──────────────────────────────────┐
│  用户代码片段 CSS / JS            │ ←──┤  选择器可匹配模板新插入的块     │
│  - 全局样式覆盖                   │    │  - MutationObserver 可监听新节点 │
│  - 全局事件监听（冒泡）           │    │  - window.siyuan 共享 API       │
│  - DOM 读写                       │    └──────────────────────────────────┘
└─────────────────┬─────────────────┘
                  │
                  ▼
         内容插入后的二次修改（不在事务内）
```

**具体影响**：
1. **变量替换的可见性**：模板渲染时 SQL 查询得到的敏感数据（最多 1024 行 `querySQL` 结果），一旦插入 DOM，代码片段的 JS 可通过 `textContent`、`dataset` 读取。
2. **上下文注入的跨域**：代码片段与模板使用不同机制（前端 vs 后端），但共享 `window.siyuan` API，片段可调用 `fetchPost` 再次请求模板渲染接口，形成循环。
3. **内容插入的后置修改**：`insertHTML` 提交事务后，代码片段可通过 JS 进一步修改 DOM（绕过编辑器校验），这些修改**不会**反映到数据库，刷新后丢失。

---

## 5. 作用域限制机制

### 5.1 模板执行作用域

模板引擎的作用域限制体现在以下层面：

1. **数据上下文限制**：
   - `RenderGoTemplate`：`Execute(buf, nil)` — 无任何数据注入。
   - `RenderTemplate`：仅注入 `title/id/name/alias` 四个 `string` 类型变量。

2. **函数执行沙箱**：
   - 所有函数均为纯函数或只读操作。
   - 移除 `env/expandenv/getHostByName` 等环境信息函数。
   - SQL 查询仅通过 `SelectBlocksRawStmt` 等只读接口，语法层仅允许 SELECT/UNION，并限制返回 512 / 1024 条。

3. **文件系统限制**：
   - 模板文件必须位于 `workspace/templates/` 目录下。
   - 通过 `IsAbsPathInWorkspace` 验证路径合法性。
   ```go
   func IsAbsPathInWorkspace(absPath string) bool {
       return gulu.File.IsSubPath(WorkspaceDir, absPath)
   }
   ```
   > 代码位置：`kernel/util/path.go`（IsAbsPathInWorkspace 函数，约第 368–370 行）

### 5.2 代码片段作用域

代码片段在浏览器全局作用域执行，但有分层限制：

1. **全局开关控制**：`window.siyuan.config.snippet.enabledCSS` / `enabledJS`。
2. **单片段启用状态**：`item.enabled` 为 `false` 时不注入。
3. **发布环境隔离**：`IsReadOnlyRoleContext(c) && s.DisabledInPublish → skip`。
4. **身份分级读取**：见 4.2.4 权限矩阵，不同角色可见范围不同。

---

## 6. 错误提示机制

### 6.1 模板错误处理

模板解析和执行错误统一使用国际化错误提示：
```go
tpl, err := tmpl.Parse(templateContent)
if err != nil {
    return "", fmt.Errorf(Conf.Language(44), err.Error())
    // Conf.Language(44) = "渲染模板失败：%s"
}
```
> 代码位置：`kernel/model/template.go`（RenderGoTemplate 内，约第 57–58 行）

### 6.2 API 层错误返回

```go
// 路径验证
if !util.IsAbsPathInWorkspace(p) {
    ret.Code = -1
    ret.Msg = "Path [" + p + "] is not in workspace"
    return
}

// ID 格式验证
if util.InvalidIDPattern(id, ret) {
    return  // ret.Msg = "invalid block id"
}

// 模板渲染错误
_, content, err := model.RenderTemplate(p, id, preview)
if err != nil {
    ret.Code = -1
    ret.Msg = util.EscapeHTML(err.Error())  // HTML 转义防止错误信息中的 XSS
    return
}
```
> 代码位置：`kernel/api/template.go`（renderTemplate 处理函数，约第 77–99 行）

### 6.3 SQL 静默失败策略

```go
rows, err := query(stmt)
if err != nil {
    if strings.Contains(err.Error(), "syntax error") {
        return  // 语法错误静默返回空，不暴露错误细节
    }
    logging.LogWarnf("sql query [%s] failed: %s", stmt, err)
    return
}
```
> 代码位置：`kernel/sql/block_query.go`（SelectBlocksRawStmt 内，约第 630–637 行）

> **设计权衡**：语法错误静默失败减少了报错注入的信息泄露，但也增加了模板作者的调试难度。

### 6.4 前端错误处理

```typescript
// 定义位置：app/src/protyle/toolbar/util.ts，函数 previewTemplate
export const previewTemplate = (pathString: string, element: Element, parentId: string) => {
    if (!pathString) {
        element.innerHTML = "";  // 路径为空时静默清空
        return;
    }
    // fetchPost 失败时由上层统一弹 toast
};
```
> 代码位置：`app/src/protyle/toolbar/util.ts`（previewTemplate 函数，约第 6–18 行）

---

## 7. 安全机制分析

### 7.1 API 鉴权体系

所有模板和片段 API 都经过多层鉴权（完整矩阵）：

| API 端点 | CheckAuth | CheckAdminRole | CheckReadonly | 可访问角色 |
|----------|:---------:|:--------------:|:-------------:|-----------|
| `/api/template/render` | ✅ | ✅ | ✅ | Administrator |
| `/api/template/renderSprig` | ✅ | ✅ | ✅ | Administrator |
| `/api/template/docSaveAsTemplate` | ✅ | ✅ | ✅ | Administrator |
| `/api/snippet/getSnippet` | ✅ | ❌ | ❌ | Administrator, Editor, Reader |
| `/api/snippet/setSnippet` | ✅ | ✅ | ✅ | Administrator |
| `/api/snippet/removeSnippet` | ✅ | ✅ | ✅ | Administrator |
| `/api/query/sql`（间接给模板用） | ✅ | ✅ | ✅ | Administrator |

> 路由注册位置：`kernel/api/router.go`（约第 368–370、472–474 行）

### 7.2 输入验证

**路径验证**（`kernel/util/path.go`）：通过 `gulu.File.IsSubPath(WorkspaceDir, absPath)` 确保路径位于工作区内。

**ID 格式验证**（`kernel/util/net.go`）：通过 `ast.IsNodeIDPattern` 正则校验块 ID 格式。

**错误信息转义**：`util.EscapeHTML(err.Error())` 对返回给前端的错误消息做 HTML 实体转义。

### 7.3 危险函数移除

```go
delete(ret, "env")           // 防止获取环境变量（DB 密码、API Key 等）
delete(ret, "expandenv")     // 防止环境变量展开
delete(ret, "getHostByName") // 防止 SSRF 相关 DNS 查询
```
> 代码位置：`kernel/filesys/template.go`（BuiltInTemplateFuncs 内，约第 38–41 行）

> **仍需警惕**：Sprig 的 `dict`、`list`、`get` 等元编程函数与 SQL 函数结合，可能构造出意外数据结构。

### 7.4 并发安全

```go
var snippetsLock = sync.Mutex{}

func SetSnippet(snippets []*conf.Snippet) (err error) {
    snippetsLock.Lock()
    defer snippetsLock.Unlock()
    err = writeSnippetsConf(snippets)
    return
}
```
> 代码位置：`kernel/model/snippet.go`（约第 32、54–60 行）

---

## 8. 扩展性设计

### 8.1 模板函数扩展

当前模板函数通过两层注入实现扩展：
1. **BuiltInTemplateFuncs**（`kernel/filesys/template.go`）：通用工具函数。
2. **SQLTemplateFuncs**（`kernel/sql/database.go`）：数据库查询函数。

> **扩展点**：可通过新增 `XxxTemplateFuncs` 函数 + 修改 `SQLTemplateFuncs` 调用点来引入插件函数。

### 8.2 插件扩展点

前端通过插件系统扩展 `/` 命令菜单（位置：`app/src/protyle/hint/extend.ts`，约第 356–366 行）。

### 8.3 多种使用场景

模板引擎在多个场景中复用：
1. 文档模板插入：`/api/template/render`
2. 动态图标：`RenderDynamicIconContentTemplate`
3. PDF 页脚：`/api/template/renderSprig`
4. 数据库计算列：Attribute View 计算列使用模板渲染
5. 代码片段：用户自定义 CSS/JS（独立但共享鉴权模型）

---

## 9. 模块间协作关系

### 9.1 模板渲染数据流（完整版本，含 SQL 限制标注）

```
┌────────────────────────────────────────────────────────────────────┐
│                           用户前端操作                              │
│  斜杠命令选模板 / 插入按钮 / 预览请求                               │
└──────────────────┬─────────────────────────────────────────────────┘
                   │  POST /api/template/render
                   │  需通过: CheckAuth → CheckAdminRole → CheckReadonly
                   ▼
┌────────────────────────────────────────────────────────────────────┐
│                     API 接口层 (kernel/api/template.go)             │
│  - p: 模板绝对路径 (IsAbsPathInWorkspace 校验)                     │
│  - id: 目标块 ID (InvalidIDPattern 校验)                           │
│  - preview: 是否预览模式                                           │
└──────────────────┬─────────────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────────────┐
│                     模板模型层 (kernel/model/template.go)           │
│  Step 1: LoadTreeByBlockID(id) → 取目标块                          │
│  Step 2: 构造 dataModel = {title,id,name,alias}                    │
│  Step 3: os.ReadFile(p) → 读模板内容                               │
│          ┌─────────────── 函数注入 ────────────────┐                │
│          │  BuiltInTemplateFuncs() + SQLTemplateFuncs              │
│          │  - Sprig TxtFuncMap (剔除 env 等)                        │
│          │  - Weekday/pow/getHPathByID/...                          │
│          │  - queryBlocks (LIMIT 512)                               │
│          │  - querySpans  (LIMIT 512)                               │
│          │  - querySQL    (LIMIT 1024)                              │
│          └───────────────────────────────────────────────┘          │
│  Step 4: template.New("").Delims(".action{", "}").Parse(content)   │
│  Step 5: Execute(buf, dataModel)                                   │
│          ┌────────── SQL 函数调用路径（⚠ 关键）────────────┐        │
│          │  模板中 .action{ queryBlocks "SELECT ... ?" .title }    │
│          │    → strings.Replace(stmt, "?", .title, 1)               │
│          │    → SelectBlocksRawStmt / Query                         │
│          │       → sqlparser.Parse 类型断言(SELECT/UNION)           │
│          │       → 强制 LIMIT 512 / 1024 + OFFSET 注入             │
│          │       → 转义清理 → query(stmt) 执行                      │
│          │       → rows.Scan 填充返回值                             │
│          └────────────────────────────────────────────────┘         │
│  Step 6: lute 引擎重新 Parse Markdown → AST                        │
│  Step 7: AST 后处理                                                 │
│          - GenBlockIDs() → 避免 ID 冲突                             │
│          - TranscludeRef() → 块引用文本替换                         │
│          - ProcessDatabaseView() → 渲染数据库块                     │
│  Step 8: Tree2BlockDOM(tree, ProtyleMaxTypeBlockCount)             │
└──────────────────┬─────────────────────────────────────────────────┘
                   │  JSON: {content: string, operations: ...}
                   ▼
┌────────────────────────────────────────────────────────────────────┐
│                   前端交互层 (app/src/protyle/*)                    │
│  ① 渲染模式判断（空行→块级，否则行级）                               │
│  ② 场景分发：av / table / codeblock / 普通                          │
│  ③ 块级插入：构建 doOperation / undoOperation 事务数组              │
│     action ∈ {insert, update, delete} + prevID/parentID 定位       │
│  ④ transaction(protyle, do, undo) → 调用 /api/transactions         │
│  ⑤ 后渲染管线：blockRender → processRender → highlight → av        │
└──────────────────┬─────────────────────────────────────────────────┘
                   │  (持久化触发)
                   ▼
┌────────────────────────────────────────────────────────────────────┐
│                     数据层 (kernel/sql)                             │
│  - 新块写入 blocks 表 + 全文索引                                    │
│  - spans 表同步更新 inline 元素                                     │
│  - filetree 等派生表按需更新                                        │
└────────────────────────────────────────────────────────────────────┘
```

### 9.2 代码片段数据流（含角色过滤）

```
┌─────────────┐
│ onGetConfig │  (应用启动 / 设置改变时)
└──────┬──────┘
       │ POST /api/snippet/getSnippet
       │ 权限: 仅 CheckAuth → Admin / Editor / Reader 均可
       │ 请求: { type:"all"|"js"|"css", enabled: 2|0|1 }
       ▼
┌──────────────────────────────────────────────────┐
│ getSnippet (kernel/api/snippet.go)               │
│  ① LoadSnippets() → 读 conf.json                 │
│  ② 角色过滤:                                     │
│     if IsReadOnlyRoleContext(c)                  │
│        && snippet.DisabledInPublish → skip       │
│  ③ 类型过滤: type != "all" && type 不匹配 → skip │
│  ④ 启用状态过滤: enabled!=2 && 不匹配 → skip     │
│  ⑤ 返回 []*Snippet（含完整 content 字段）         │
└──────┬───────────────────────────────────────────┘
       │ JSON Response
       ▼
┌──────────────────────────────────────────────────┐
│ renderSnippet (app/src/config/util/snippets.ts)  │
│  遍历每个片段:                                    │
│  ① 全局开关检查（enabledCSS / enabledJS）        │
│  ② 单片段 enabled 检查                           │
│  ③ CSS → <style id="snippetCSS{id}">             │
│                   .insertAdjacentHTML("beforeend")│
│     JS → <script id="snippetJS{id}">             │
│                   .text = content → appendChild  │
└──────┬───────────────────────────────────────────┘
       │
       ▼
  document.head 中生效，样式立即作用于所有已渲染 DOM
  JS 在全局作用域执行，window.siyuan API 可用
```

---

## 10. 潜在问题与风险

### 10.1 安全风险

#### 10.1.1 SQL 注入：分层防护但不完美（含 512 vs 1024 差异影响）

| 防护层 | 机制 | 绕过可能性 | 数据量上限 | 严重程度 |
|--------|------|-----------|-----------|---------|
| API 鉴权 | 模板渲染需 CheckAdminRole | Reader 无法直接调接口，但可通过已插入模板的结果看到注入输出 | — | 中 |
| 字符串替换 | `strings.Replace(stmt, "?", arg, 1)` | **无转义**：`'`、`;`、注释符均可原样进入 | — | 高 |
| sqlparser 类型断言 | 仅允许 `*sqlparser.Select` / `*sqlparser.Union` | 无法直接 DML，但 UNION / 子查询 / SQLite 函数仍可探索 | — | 中高 |
| LIMIT 强制注入 | queryBlocks/Spans=512, querySQL=1024 | 可分页或用 UNION ALL 聚合 | 512 / 1024 | 低（querySQL 风险略高） |
| 语法错误静默 | 错误不报给前端 | 增加盲注难度，但仍可基于响应内容长度判断 | — | 中 |

> **querySQL 风险更高的原因**：1024 上限意味着攻击者在单次请求中可获取两倍于 `queryBlocks` 的数据量，在时间盲注或 UNION 聚合场景下效率更高。

#### 10.1.2 代码片段权限：Reader 可读取全部未屏蔽片段

- 场景：发布站点开启了 Reader 账号（公开但需登录）。
- 影响：含内部 API 地址、调试逻辑的 JS 片段暴露；CSS 中泄露的 DOM 结构信息可被爬虫利用。
- 缓解：`DisabledInPublish` 标记 + 前端全局开关 `enabledJS` / `enabledCSS`。
- **隐患**：读取操作**无审计日志**，无法追溯谁下载了片段内容。

#### 10.1.3 XSS 风险

- **模板侧**：SQL 查询结果（`Block.Content`）包含用户输入的 HTML/Markdown，经 lute 渲染后仍可能存在标签过滤漏洞。
- **片段侧**：片段本身是完全信任的，一旦管理员账号被盗，攻击者可植入持久型 XSS。
- **错误信息**：`EscapeHTML` 仅在 API 返回 `Msg` 时生效，`Content` 字段不做额外转义。

#### 10.1.4 路径遍历

`IsAbsPathInWorkspace` 基于 `gulu.File.IsSubPath`，需确认其对 Windows 符号链接、`..\` 重复拼接等边界场景的处理。

### 10.2 性能问题

1. **每次渲染都重新 Parse 模板**：无编译缓存，大模板或高频场景下 CPU 压力明显。
2. **AST 全量后处理**：`GenBlockIDs` / `ProcessDatabaseView` 遍历整棵树。
3. **SQL 无执行超时**：复杂子查询 + 大量数据可阻塞 Go 协程。
4. **代码片段全量传输**：前端一次性拉取所有片段的 `content`（默认 `enabled=2` 只拉已启用）。

### 10.3 功能缺陷

1. **错误信息不精确**：缺少行号、列号、函数签名提示。
2. **模板调试缺失**：无法 print 变量、无法断点。
3. **片段版本管理缺失**：无历史版本、无 diff、无回滚。
4. **SQL 函数文档不足**：每个函数的返回结构需查源码。
5. **参数替换顺序无法校验**：`?` 数量与 `args` 数量不匹配不会报错。
6. **SQL 限制不一致**：`queryBlocks` / `querySpans` = 512，`querySQL` = 1024，缺少统一常量定义，易在后续维护中引发误用。

---

## 11. 后续研究方向

### 11.1 安全性增强

1. **真实参数化查询**：
   ```go
   // 目标：将 args 传入 db.Query，而非字符串替换
   // rows, err := db.Query(stmt, convertToAny(args)...)
   // 需要解决类型转换（Go template 传参均为 string）与 ? 数量校验
   ```

2. **统一 SQL 限制常量**：
   ```go
   // 建议新增
   const (
       TemplateQueryBlocksLimit = 512
       TemplateQuerySpansLimit  = 512
       TemplateQuerySQLLimit    = 1024  // 或统一为同一值并文档说明差异
   )
   ```

3. **SQL 函数沙箱**：
   - `context.WithTimeout` 包裹 `db.Query`。
   - 扫描行数限制 + 列白名单（Block/Span 字段级过滤）。
   - SQLite `PRAGMA trusted_schema=OFF` 等运行时加固。

4. **代码片段审计**：
   - `/getSnippet` 请求写操作日志（IP / 角色 / 时间 / 筛选条件）。
   - `/setSnippet` 变更写历史（保留 N 个版本）。
   - 新增 `sha256(content)` 字段，前端可做完整性校验。

5. **Reader 角色细分**：新增 "ReaderNoSnippet" 角色或细粒度配置项；片段级 `minRole` 属性。

### 11.2 性能优化

1. **模板编译缓存（LRU）**：基于内容哈希缓存 `*template.Template`。
2. **增量后处理**：对 AST 中未修改子树跳过 `GenBlockIDs`。
3. **异步渲染**：复杂模板在 goroutine 中执行，前端显示进度条。
4. **片段增量传输**：HTTP 响应头 `ETag` + `If-None-Match`，`content` 不变则返回 304。

### 11.3 功能扩展

1. **模板调试工具**：语法高亮 + 错误下划线；内置 `debug()` 函数；执行时间面板。
2. **变量作用域扩展（按需开关）**：`dataModel` 增加 `doc.updated`、`user.role`、`workspace.name` 等字段。
3. **片段沙箱**：Web Worker 包裹 JS 片段，使用受控代理 API；CSS 前缀隔离（Shadow DOM）。
4. **模板生命周期钩子**：`preRender` / `postRender` / `postInsert`。

### 11.4 架构演进

1. **插件化模板函数注册**：定义 `TemplateFuncProvider` 接口（含 `RequiredRole()` 权限声明）。
2. **多模板引擎**：除 `text/template` 外引入极简引擎（无循环、无函数）供 Reader 级场景使用。
3. **可视化模板编辑器**：拖拽块结构 + 绑定变量；SQL 构建器（参数占位符可视化 + 类型标注）；风险提示高亮。

---

## 12. 源码检索指引（仓库相对路径）

> 跨环境复核说明：下表中的所有路径均为**相对仓库根目录**的路径。在你的本地环境中，可通过 `cd <仓库根>` 后用 `grep` / IDE 全局搜索快速定位。文件后的行号区间为该功能的大致代码范围，不同版本可能略有差异。

### 12.1 模板引擎核心

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| 带上下文模板渲染入口 RenderTemplate | `kernel/model/template.go`（约 L309–L499） |
| 标准 Sprig 模板渲染 RenderGoTemplate | `kernel/model/template.go`（约 L51–L69） |
| 动态图标内容模板 RenderDynamicIconContentTemplate | `kernel/model/template.go`（约 L264–L307） |
| 文档另存为模板 RenderDocSaveAsTemplate | `kernel/model/template.go`（约 L183–L262） |
| 内置模板函数 + 危险函数移除 BuiltInTemplateFuncs | `kernel/filesys/template.go`（约 L35–L63） |
| SQL 模板函数注册（含 512/1024 限制） SQLTemplateFuncs | `kernel/sql/database.go`（L1591–L1621，精确） |

### 12.2 SQL 执行层（含语法解析防护）

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| SelectBlocksRawStmt（sqlparser 类型断言 + LIMIT 注入） | `kernel/sql/block_query.go`（约 L551–L644） |
| SelectBlocksRawStmtNoParse（绕过解析器的回退路径） | `kernel/sql/block_query.go`（约 L547–L549） |
| Query（双解析器策略，querySQL 底层，LIMIT 1024） | `kernel/sql/block_query.go`（约 L366–L445） |
| SelectSpansRawStmt（spans 表等效逻辑，LIMIT 512） | `kernel/sql/span.go`（约 L52–L93） |
| 底层 query 函数（最终 db.Query 调用点） | `kernel/sql/database.go`（约 L1359–L1369） |

### 12.3 代码片段核心

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| 片段加载（含自动补 ID） loadSnippets | `kernel/model/snippet.go`（约 L68–L111） |
| 片段保存（含互斥锁） SetSnippet | `kernel/model/snippet.go`（约 L32、L54–L60） |
| 前端 DOM 渲染 renderSnippet | `app/src/config/util/snippets.ts`（约 L7–L42） |
| 片段数据结构 Snippet（含 DisabledInPublish） | `kernel/conf/snippet.go`（约 L31–L38） |
| 启动时 renderSnippet 调用点 | `app/src/boot/onGetConfig.ts`（约 L78） |

### 12.4 API 接口 + 权限矩阵

| 接口 | 路由位置（相对路径） | 鉴权中间件链 |
|------|-------------------|-------------|
| 渲染模板 | `kernel/api/router.go`（约 L368–L370） | CheckAuth + CheckAdminRole + CheckReadonly |
| 渲染 Sprig 模板 | `kernel/api/template.go`（约 L28–L45） | 同上 |
| 文档另存为模板 | `kernel/api/template.go`（约 L47–L66） | 同上 |
| 获取片段（仅 CheckAuth，Reader 可过） | `kernel/api/router.go`（L472，精确） | CheckAuth |
| 获取片段 + 角色过滤实现 | `kernel/api/snippet.go`（约 L63–L76） | IsReadOnlyRoleContext + DisabledInPublish |
| 保存片段 | `kernel/api/router.go`（L473，精确） | CheckAuth + CheckAdminRole + CheckReadonly |
| 删除片段 | `kernel/api/router.go`（L474，精确） | CheckAuth + CheckAdminRole + CheckReadonly |

### 12.5 角色与鉴权实现

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| CheckAuth（4 种认证方式 + 3 种角色） | `kernel/model/session.go`（约 L207–L260） |
| CheckAdminRole（仅 Admin 通过） | `kernel/model/session.go`（约 L185–L193） |
| CheckReadonly（非 ReadOnly 角色 + 非只读开关） | `kernel/model/session.go`（约 L195–L205） |
| 角色定义（Administrator/Editor/Reader/Visitor） | `kernel/model/role.go`（约 L1–L64） |
| IsReadOnlyRoleContext 判定 | `kernel/model/role.go`（约 L62–L64） |

### 12.6 前端交互与内容插入

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| 模板插入渲染 hintRenderTemplate（斜杠命令回调） | `app/src/protyle/hint/extend.ts`（约 L542–L564） |
| 模板预览入口 previewTemplate | `app/src/protyle/toolbar/util.ts`（约 L6–L18） |
| 内容插入核心 insertHTML（事务构建） | `app/src/protyle/util/insertHTML.ts`（约 L264–L590） |
| 斜杠命令菜单（模板/插件入口） hintSlash | `app/src/protyle/hint/extend.ts`（约 L33–L39） |

### 12.7 输入验证与路径检查

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| IsAbsPathInWorkspace（路径子目录校验） | `kernel/util/path.go`（约 L368–L370） |
| InvalidIDPattern（ID 格式校验） | `kernel/util/net.go`（约 L314–L322） |

---

## 13. 路径格式与 SQL 限制对核心链路的综合影响（新增总结）

### 13.1 本机绝对路径 → 仓库相对路径的改进收益

| 维度 | 本机绝对路径（旧） | 仓库相对路径（新） | 收益 |
|------|-----------------|----------------|------|
| 跨平台可复核 | ❌ Windows 盘符 / macOS 路径不兼容 | ✅ 所有平台一致 | 代码审查、Issue 追溯效率显著提升 |
| 仓库迁移 | ❌ 移动目录后全部失效 | ✅ 不受影响 | 文档长期可维护性 |
| 安全脱敏 | ❌ 暴露开发者目录结构 | ✅ 不包含本机信息 | 可直接用于公开文档 |
| IDE 跳转 | ✅ `file://` 可一键跳转 | ⚠️ 需手动拼接或配合 IDE 搜索 | 第 12 章提供行号范围做补偿 |
| 版本差异 | ❌ 行号偏移后难以重新定位 | ✅ 配合行号范围可快速 grep | 跨版本可复核性增强 |

### 13.2 SQL 返回限制（512 vs 1024）对注入链路的具体影响

```
变量替换阶段               上下文注入阶段                 内容插入阶段
   .title ──┐                │                          │
            ▼                ▼                          ▼
    strings.Replace    SQL 返回结果进入                insertHTML
    (无转义)           range 循环 / 变量渲染            事务提交到 DB
            │                │                          │
            │                │ queryBlocks: ≤512 行     │ 单次渲染最多插入
            │                │ querySpans : ≤512 行     │ 受 LIMIT 间接限制
            │                │ querySQL   : ≤1024 行    │ 的块数量
            │                │                          │
            └───────────  512 与 1024 的差异  ────────────┘
                           │
                           ▼
            querySQL 返回行数翻倍 →
              ① 注入 Payload 可携带的数据更多
              ② 时间盲注单次提取的位数更多
              ③ 存储型 XSS 在 DOM 中的触发面更广
```

**具体影响分析**：

1. **对变量替换的影响**：SQL 限制本身不影响 `.title` 等变量进入 SQL 时的转义缺失问题，但决定了**注入成功后能拿到多少数据**。`querySQL` 的 1024 行上限使得攻击者在一次模板渲染中可提取更多记录。

2. **对上下文注入的影响**：SQL 返回结果成为模板 `range` 循环的数据源。512 限制意味着 `range` 最多迭代 512 次，减少了渲染时的计算量和输出长度。但 `querySQL` 的 1024 上限放宽了这一约束，在模板递归或嵌套渲染时可能造成内存或输出膨胀。

3. **对内容插入的影响**：渲染后的 HTML 通过 `insertHTML` 进入事务系统。SQL 返回行数越多，生成的 DOM 节点和事务操作越多。`querySQL` 的 1024 行上限意味着在极端情况下，单次模板插入可能产生数千个 DOM 节点，对前端性能和事务大小构成压力。

### 13.3 建议行动项

| 优先级 | 行动项 | 关联章节 |
|--------|--------|---------|
| P0 | 将 `queryBlocks` / `querySpans` 的 `strings.Replace` 改为真实参数化查询 | 2.3.2, 11.1 |
| P1 | 统一 SQL 限制常量（`TemplateQueryBlocksLimit` 等），消除硬编码 512/1024 | 10.3, 11.1 |
| P1 | 为 `/api/snippet/getSnippet` 增加操作审计日志 | 4.2.4, 11.1 |
| P2 | 在文档/注释中明确说明 `querySQL` 使用 1024 的设计理由 | 2.3.1 |
| P2 | 为 `querySQL` 返回的 `map[string]any` 增加字段级白名单 | 7.1, 11.1 |
| P3 | 评估引入模板编译缓存（LRU）的可行性 | 10.2, 11.2 |

---

## 总结

SiYuan 的模板与代码片段引擎是一个**分层明确、协作紧密**的系统。经过本次核实与修正，文档形成以下最终结论：

### ✅ 设计亮点
1. **双模板模式**：`{{ }}` 与 `.action{ }` 分隔符适应不同场景，减少语法冲突。
2. **SQL 语法层防护**：`sqlparser` 类型断言仅允许 SELECT/UNION，是防御 DML 攻击的核心屏障。
3. **分级权限模型**：`/getSnippet` 仅需 CheckAuth（Reader 可访问），`/setSnippet` 需 Admin，平衡了可用性与安全性。
4. **事务化内容插入**：所有块级修改走 `doOperation/undoOperation`，确保可撤销与数据一致性。
5. **角色二次过滤**：发布环境中 `DisabledInPublish + IsReadOnlyRoleContext` 保护敏感片段。

### ⚠️ 需重点关注的问题（本次核实修正）
1. **SQL 返回限制不一致**：`queryBlocks` / `querySpans` = **512** 条，`querySQL` = **1024** 条。差异本身并非 bug，但缺少统一常量定义和文档说明，维护时容易混淆。
2. **SQL 伪参数化**：`strings.Replace` 无转义是当前最大隐患，应尽快改为真实 `db.Query(stmt, args...)`。
3. **解析器回退路径**：双解析器均失败时跳过类型断言直接执行，存在被构造"解析器盲区"突破的风险。
4. **Reader 片段读取无审计**：建议为 `/getSnippet` 增加操作日志。
5. **源码引用格式**：本文已全部改为**仓库相对路径 + 行号范围**，解决了本机绝对路径跨环境不可复用的问题。

### 🔗 关键因素对核心链路的影响汇总

| 关键因素 | 变量替换阶段 | 上下文注入阶段 | 内容插入阶段 |
|---------|------------|-------------|------------|
| **CheckAuth 接受 Reader** | 不直接影响（模板渲染仍需 Admin） | 片段可间接读取已渲染结果 | Reader 可下载片段分析 DOM 结构 → 构造更精准注入 |
| **strings.Replace 替换 ?** | `.title` 等变量进入 SQL 时可改变语义 | SQL 返回数据成为模板循环上下文 | 注入输出的内容经 lute 渲染后直接入事务 |
| **sqlparser 仅 SELECT** | 防 DML 但不防盲注 | SQL 结果字段集合受限（Block/Span/map） | 输出数据仍可通过语义内容实现钓鱼或误导 |
| **LIMIT 512 vs 1024** | 不直接影响变量 | querySQL 返回 2 倍数据量 → 模板循环压力增大 | 单次插入的 DOM 节点数可能翻倍 |
| **DisabledInPublish** | — | Reader 上下文下敏感片段不返回 | 前端不执行敏感 JS，减少后续二次攻击面 |
| **insertHTML 事务化** | — | 不受影响 | 即使内容异常也可撤销；但存储型 XSS 仍需额外检测 |

以上分析揭示了 SiYuan 模板系统在**安全性、可用性、可扩展性、可复核性**之间的精心权衡，同时指出了在真实参数化查询、统一 SQL 限制常量、审计日志、沙箱化执行、文档引用规范化等方面的持续演进方向。
