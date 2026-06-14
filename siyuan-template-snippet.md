# SiYuan 模板与代码片段引擎实现原理分析

## 1. 系统架构概述

SiYuan（思源笔记）的模板与代码片段引擎是一个分层设计的系统，由后端核心引擎、API 接口层和前端交互层协同工作。该系统基于 Go 标准库 `text/template` 构建，通过自定义分隔符、函数注入和上下文管理，实现了强大的模板渲染能力。

### 1.1 核心模块组成

| 模块 | 主要职责 | 核心文件 |
|------|---------|---------|
| 模板引擎核心 | 模板解析、变量替换、函数执行 | [template.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go) |
| 内置模板函数 | 通用工具函数、日期处理、统计函数 | [template.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/filesys/template.go) |
| SQL 模板函数 | 数据库查询相关模板函数 | [database.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/database.go#L1591-L1621) |
| SQL 参数化执行 | SQL 语法解析、查询限制、行扫描 | [block_query.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/block_query.go#L366-L445) |
| 代码片段管理 | CSS/JS 片段的加载、渲染、持久化 | [snippet.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/snippet.go) |
| API 接口层 | HTTP 接口定义与权限控制 | [template.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/template.go), [snippet.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/snippet.go) |
| 角色与权限模型 | 角色定义、只读/管理员权限校验 | [role.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/role.go), [session.go](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/session.go#L207-L260) |
| 前端交互层 | 模板选择、预览、插入渲染 | [extend.ts](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/hint/extend.ts), [snippets.ts](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/config/util/snippets.ts) |
| 内容插入层 | DOM 操作、事务处理、块更新 | [insertHTML.ts](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/util/insertHTML.ts) |

---

## 2. 模板引擎核心实现原理

### 2.1 模板解析流程

模板解析采用 Go 标准库 `text/template`，但使用了自定义的分隔符和函数扩展机制。

#### 2.1.1 双模式解析系统

SiYuan 实现了两种模板解析模式：

**模式一：标准 Go 模板语法（`{{ }}` 分隔符）**
```go
func RenderGoTemplate(templateContent string) (ret string, err error) {
    tmpl := template.New("")
    tplFuncMap := filesys.BuiltInTemplateFuncs()
    sql.SQLTemplateFuncs(&tplFuncMap)
    tmpl = tmpl.Funcs(tplFuncMap)
    tpl, err := tmpl.Parse(templateContent)
    err = tpl.Execute(buf, nil)  // 无数据上下文
}
```
[template.go:51-69](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go#L51-L69)

**模式二：SiYuan 动作模板（`.action{ }` 分隔符）**
```go
func RenderTemplate(p, id string, preview bool) (tree *parse.Tree, dom string, err error) {
    // ... 构建 dataModel
    goTpl := template.New("").Delims(".action{", "}")
    tplFuncMap := filesys.BuiltInTemplateFuncs()
    sql.SQLTemplateFuncs(&tplFuncMap)
    goTpl = goTpl.Funcs(tplFuncMap)
    tpl, err := goTpl.Funcs(tplFuncMap).Parse(gulu.Str.FromBytes(md))
    err = tpl.Execute(buf, dataModel)  // 注入数据上下文
}
```
[template.go:309-499](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go#L309-L499)

> **设计意图**：使用 `.action{ }` 作为分隔符可以避免与 Markdown 中的其他语法冲突，特别是在文档模板中。

### 2.2 变量替换与上下文注入

#### 2.2.1 数据模型（Data Model）

模板渲染时注入的上下文变量经过严格限制，仅包含 4 个字符串字段：

```go
dataModel := map[string]string{
    "title": titleVar,    // 文档标题或块内容（经过空白清理）
    "id":    block.ID,    // 块 ID（23 位固定格式）
    "name":  block.Name,  // 块名称
    "alias": block.Alias, // 块别名
}
```
[template.go:326-337](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go#L326-L337)

> **作用域限制设计**：只注入 `map[string]string` 而非结构体，避免通过反射暴露更多字段。
> `title` 经过 `strings.TrimSpace` 清理，降低模板变量直接引发注入的概率。

#### 2.2.2 模板函数注入

模板函数通过两层注入机制构建：

**第一层：Sprig 通用函数库 + 自定义函数**
```go
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
[template.go:35-63](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/filesys/template.go#L35-L63)

**第二层：SQL 查询函数**
```go
func SQLTemplateFuncs(templateFuncMap *template.FuncMap) {
    (*templateFuncMap)["queryBlocks"] = func(stmt string, args ...string) (retBlocks []*Block) {
        for _, arg := range args {
            stmt = strings.Replace(stmt, "?", arg, 1)  // ⚠ 字符串拼接式参数替换
        }
        retBlocks = SelectBlocksRawStmt(stmt, 1, 512)
        return
    }
    (*templateFuncMap)["getBlock"] = func(arg any) (retBlock *Block) { /* ... */ }
    (*templateFuncMap)["querySpans"] = func(stmt string, args ...string) (retSpans []*Span) {
        for _, arg := range args {
            stmt = strings.Replace(stmt, "?", arg, 1)  // ⚠ 字符串拼接式参数替换
        }
        retSpans = SelectSpansRawStmt(stmt, 512)
        return
    }
    (*templateFuncMap)["querySQL"] = func(stmt string) (ret []map[string]any) {
        ret, _ = Query(stmt, 512)
        return
    }
}
```
[database.go:1591-1621](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/database.go#L1591-L1621)

### 2.3 SQL 模板参数处理详解（修正补充）

这是模板引擎安全设计中**最关键也最容易被忽略的环节**，需要深入分析。

#### 2.3.1 伪参数化：字符串替换的实现细节

`queryBlocks` 和 `querySpans` 采用了**模拟参数化**而非真实的 SQL 预编译：

```go
// 模板函数内部实现
for _, arg := range args {
    stmt = strings.Replace(stmt, "?", arg, 1)  // 每次只替换第一个问号
}
```

这意味着：
- **替换顺序敏感**：`args` 的顺序必须与 SQL 中 `?` 出现的顺序严格一致
- **无转义处理**：`arg` 中的单引号、反斜杠等特殊字符不会被转义
- **无法区分字面量问号与占位符**：如果 SQL 字面量中包含 `?`，会被误替换

**风险传递路径**：
```
模板作者在 {{ queryBlocks "SELECT * FROM blocks WHERE content LIKE '%?%'" .title }}
                                    ↓
                    .title 可能是 "abc' OR '1'='1"
                                    ↓
        strings.Replace 后得到带有注入语句的完整 SQL
                                    ↓
                SelectBlocksRawStmt 执行（仅做 SELECT 语法校验）
```

#### 2.3.2 SQL 语法解析层的防护作用

虽然参数替换不安全，但 SQL 执行前会经过 `sqlparser` 的语法解析，这是**核心防护**：

```go
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
[block_query.go:551-644](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/block_query.go#L551-L644)

**防护能力分析**：

| 攻击类型 | 是否被拦截 | 说明 |
|---------|-----------|------|
| DML 语句（DROP/DELETE/INSERT） | ✅ 是 | 语法解析非 SELECT/UNION 直接返回空 |
| DDL 语句（CREATE/ALTER） | ✅ 是 | 同样被类型断言拦截 |
| UNION 注入 | ⚠️ 部分 | UNION 本身被允许，但整体仍被 LIMIT 限制 |
| 时间盲注 | ⚠️ 否 | 可通过 `LIKE` + 子查询 + `SLEEP()` 构造 |
| 报错注入 | ⚠️ 部分 | 语法错误静默返回空，难以利用错误信息 |
| 批量数据导出 | ⚠️ 限制 | 被强制 LIMIT 512，但可分页或多次查询绕过 |

**`querySQL` 的双解析器策略**：
```go
func Query(stmt string, limit int) (ret []map[string]any, err error) {
    p := sqlparser2.NewParser(strings.NewReader(stmt))  // 支持 || 连接符
    parsedStmt2, err := p.ParseStatement()
    if err != nil {
        if !strings.Contains(stmt, "||") {
            parsedStmt, err2 := sqlparser.Parse(stmt)  // 回退到支持 UNION 的解析器
            // ... 同样进行类型断言 + LIMIT 注入
        }
    }
    // 解析失败则直接执行（回退路径）
    rows, err := query(stmt)
    if err != nil {
        rows, err = query(originalStmt + " LIMIT " + strconv.Itoa(limit))  // 最后兜底
    }
}
```
[block_query.go:366-445](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/block_query.go#L366-L445)

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
│     - ⚠ SQL 函数执行            │
│       - strings.Replace 替换 ?  │
│       - sqlparser 语法校验      │
│       - 强制 LIMIT 注入         │
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
[extend.ts:542-564](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/hint/extend.ts#L542-L564)

### 3.2 insertHTML 核心逻辑

`insertHTML` 是内容插入的核心，处理多种场景：

```typescript
export const insertHTML = (html: string, protyle: IProtyle, isBlock = false,
                           useProtyleRange = false, insertByCursor = false) => {
    // 1. 获取编辑范围
    const range = usePrortle ? protyle.toolbar.range : getEditorRange(protyle.wysiwyg.element);

    // 2. 特殊场景分发
    // 2.1 数据库单元格 → processAV
    if (blockElement.classList.contains("av")) { processAV(...); return }
    // 2.2 表格单元格批量填充 → processTable
    if (blockElement.classList.contains("table") && processTable(...)) { return }
    // 2.3 代码块 → 保留换行符的纯文本插入
    if (!isBlock && isNodeCodeBlock) { /* codeblock 处理逻辑 */ return }

    // 3. 行级插入
    if (!isBlock) {
        range.insertNode(tempElement.content.cloneNode(true));
        input(protyle, blockElement, range);  // 触发输入事件，更新 IAL
        return;
    }

    // 4. 块级插入：构造事务
    const doOperation: IOperation[] = [];
    const undoOperation: IOperation[] = [];
    Array.from(tempElement.content.children).reverse().find((item) => {
        if (addId === id) {
            doOperation.push({ action: "update", data: item.outerHTML, id: addId });
        } else {
            doOperation.push({
                action: "insert",
                data: item.outerHTML,
                id: addId,
                previousID: id
            });
        }
    });
    transaction(protyle, doOperation, undoOperation);
};
```
[insertHTML.ts:264-590](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/util/insertHTML.ts#L264-L590)

### 3.3 权限与 SQL 处理对内容插入链路的影响（补充）

模板渲染到内容插入之间存在**三段数据传递**，每一段都可能被 SQL 查询结果影响：

```
┌──────────────────────┐    ┌──────────────────────┐    ┌──────────────────────┐
│  模板 Execute 阶段   │ →  │  AST 后处理阶段      │ →  │ insertHTML 阶段      │
│  (Go 后端)           │    │  (lute 引擎)         │    │  (前端 DOM)          │
└──────────────────────┘    └──────────────────────┘    └──────────────────────┘
         │                           │                           │
         ▼                           ▼                           ▼
 queryBlocks 返回的 *Block      HTML 实体转义规则            insertAdjacentHTML
 Content / Markdown 字段        与 标签白名单                / range.insertNode
         │                           │                           │
         └────── 未经过滤的用户数据可沿此链路传递 ────────────────┘
```

**关键风险节点**：

| 链路节点 | 风险来源 | 影响 |
|---------|---------|------|
| 模板变量 → SQL | `{{ queryBlocks "..." .title }}` | `.title` 中的特殊字符进入 SQL |
| SQL 结果 → Markdown | `Block.Content` / `Block.Markdown` | 返回的内容包含原始 HTML 或 Markdown 注入 |
| Markdown → HTML（lute） | 不受信任的 Markdown 语法 | lute 解析生成 DOM（需依赖其 XSS 防护） |
| HTML → 前端 DOM | `insertHTML` 直接插入 `response.data.content` | 若后端 HTML 转义不完整，触发存储型 XSS |
| 事务提交 → 数据库 | `action: "insert"` 的 data 字段 | 恶意内容被持久化，后续所有打开该文档的用户受影响 |

**结论**：SQL 模板函数的输出直接进入渲染流程，是**变量替换 → 上下文注入 → 内容插入**整个链路中**数据量最大、来源最不可控**的入口。

---

## 4. 代码片段（Snippet）引擎

### 4.1 代码片段数据结构

```go
type Snippet struct {
    ID                string `json:"id"`
    Name              string `json:"name"`
    Type              string `json:"type"`              // "js" 或 "css"
    Enabled           bool   `json:"enabled"`           // 是否启用
    DisabledInPublish bool   `json:"disabledInPublish"` // 发布模式下是否禁用
    Content           string `json:"content"`           // 完整内容
}
```
[snippet.go:31-38](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/conf/snippet.go#L31-L38)

### 4.2 读取权限深度分析（修正补充）

这是之前分析中需要**重点修正**的部分。

#### 4.2.1 API 路由权限矩阵

```go
// 路由定义
ginServer.Handle("POST", "/api/snippet/getSnippet",
    model.CheckAuth,            // ✅ 只需要登录
    getSnippet)

ginServer.Handle("POST", "/api/snippet/setSnippet",
    model.CheckAuth,            // ① 登录
    model.CheckAdminRole,       // ② 管理员角色 ❌ Reader/Editor 无权限
    model.CheckReadonly,        // ③ 非只读模式
    setSnippet)

ginServer.Handle("POST", "/api/snippet/removeSnippet",
    model.CheckAuth,
    model.CheckAdminRole,       // ❌ Reader/Editor 无权限
    model.CheckReadonly,
    removeSnippet)
```
[router.go:472-474](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/router.go#L472-L474)

#### 4.2.2 CheckAuth 可通过的角色范围

```go
func CheckAuth(c *gin.Context) {
    // ① JWT 认证：接受 3 种角色
    if role := GetGinContextRole(c); IsValidRole(role, []Role{
        RoleAdministrator,  // 管理员
        RoleEditor,         // 编辑者
        RoleReader,         // 👀 只读用户
    }) {
        c.Next()
        return
    }
    // ② API Token → RoleAdministrator
    // ③ Session 认证（localhost + 空授权码 → RoleAdministrator）
}
```
[session.go:207-213](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/session.go#L207-L213)

#### 4.2.3 发布模式下的二次过滤

```go
func getSnippet(c *gin.Context) {
    // ... 解析请求参数 ...
    confSnippets, err := model.LoadSnippets()
    
    isPublish := model.IsReadOnlyRoleContext(c)  // Reader / Visitor
    var snippets []*conf.Snippet
    for _, s := range confSnippets {
        if isPublish && s.DisabledInPublish {
            continue  // 👀 发布模式下，标记"发布时禁用"的片段被过滤
        }
        if "all" != typ && s.Type != typ { continue }
        if 2 != enabledArg && s.Enabled != enabled { continue }
        snippets = append(snippets, s)
    }
    // ... 返回
}
```
[snippet.go:63-76](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/snippet.go#L63-L76)

其中 `IsReadOnlyRoleContext` 判定逻辑：
```go
func IsReadOnlyRole(role Role) bool {
    return IsValidRole(role, []Role{ RoleReader, RoleVisitor })
}
```
[role.go:43-47](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/role.go#L43-L47)

#### 4.2.4 权限分析结论

| 角色 | 读取片段 `/getSnippet` | 修改片段 `/setSnippet` | 删除片段 `/removeSnippet` |
|------|----------------------|----------------------|------------------------|
| Administrator | ✅ 全部内容 | ✅ 是 | ✅ 是 |
| Editor | ✅ 全部内容（因 CheckAuth 接受） | ❌ 被 CheckAdminRole 拦截 | ❌ 被 CheckAdminRole 拦截 |
| Reader | ✅ 全部内容（但 `DisabledInPublish=true` 被过滤） | ❌ | ❌ |
| Visitor | ❌ 被 CheckAuth 拦截（CheckAuth 不接受 Visitor） | ❌ | ❌ |
| 未认证 / Token 错误 | ❌ 401 | ❌ | ❌ |

**关键发现**：
1. **Editor 角色可读不可改**：Editor 能下载所有 JS/CSS 片段内容，但无法修改。这在团队场景中意味着片段内容对所有成员透明。
2. **Reader 角色读取受 DisabledInPublish 限制**：发布环境中敏感片段（如含内部地址的脚本）可被标记为发布时禁用，Reader 无法获取。
3. **Visitor 无任何 API 访问权**：必须至少是 Reader 级别才能通过 `/api/snippet/*`。

### 4.3 代码片段加载与渲染

**后端加载逻辑**：
```go
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
[snippet.go:68-111](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/snippet.go#L68-L111)

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
[snippet.go:32,54-60](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/snippet.go#L32-L60)

**前端渲染逻辑**：
```typescript
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
[snippets.ts:7-42](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/config/util/snippets.ts#L7-L42)

**触发时机**：`renderSnippet` 在 `onGetConfig` 中尽早调用，确保片段在界面渲染前注入：
```typescript
renderSnippet();  // 在 onGetConfig.ts:78
```
[onGetConfig.ts:78](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/boot/onGetConfig.ts#L78)

### 4.4 代码片段对变量替换、上下文注入、内容插入的影响（补充）

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
1. **变量替换的可见性**：模板渲染时 SQL 查询得到的敏感数据，一旦插入 DOM，代码片段的 JS 可通过 `textContent`、`dataset` 读取。
2. **上下文注入的跨域**：代码片段与模板使用不同机制（前端 vs 后端），但共享 `window.siyuan` API，片段可调用 `fetchPost` 再次请求模板渲染接口，形成循环。
3. **内容插入的后置修改**：`insertHTML` 提交事务后，代码片段可通过 JS 进一步修改 DOM（绕过编辑器校验），这些修改**不会**反映到数据库，刷新后丢失。

---

## 5. 作用域限制机制

### 5.1 模板执行作用域

模板引擎的作用域限制体现在以下层面：

1. **数据上下文限制**：
   - `RenderGoTemplate`：`Execute(buf, nil)` — 无任何数据注入
   - `RenderTemplate`：仅注入 `title/id/name/alias` 四个 `string` 类型变量

2. **函数执行沙箱**：
   - 所有函数均为纯函数或只读操作
   - 移除 `env/expandenv/getHostByName` 等环境信息函数
   - SQL 查询仅通过 `SelectBlocksRawStmt` 等只读接口，语法层仅允许 SELECT/UNION

3. **文件系统限制**：
   - 模板文件必须位于 `workspace/templates/` 目录下
   - 通过 `IsAbsPathInWorkspace` 验证路径合法性
   ```go
   func IsAbsPathInWorkspace(absPath string) bool {
       return gulu.File.IsSubPath(WorkspaceDir, absPath)
   }
   ```
   [path.go:368-370](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/util/path.go#L368-L370)

### 5.2 代码片段作用域

代码片段在浏览器全局作用域执行，但有分层限制：

1. **全局开关控制**：
   ```typescript
   if ((!window.siyuan.config.snippet.enabledCSS && item.type === "css") ||
       (!window.siyuan.config.snippet.enabledJS && item.type === "js")) {
       return;
   }
   ```

2. **单片段启用状态**：`item.enabled` 为 `false` 时不注入。

3. **发布环境隔离**：
   ```go
   isPublish := model.IsReadOnlyRoleContext(c)
   for _, s := range confSnippets {
       if isPublish && s.DisabledInPublish {
           continue;
       }
   }
   ```
   [snippet.go:63-68](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/snippet.go#L63-L68)

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
[template.go:57-58](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go#L57-L58)

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
    return
    // ret.Msg = "invalid block id"
}

// 模板渲染错误
_, content, err := model.RenderTemplate(p, id, preview)
if err != nil {
    ret.Code = -1
    ret.Msg = util.EscapeHTML(err.Error())  // HTML 转义防止错误信息中的 XSS
    return
}
```
[template.go:77-99](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/template.go#L77-L99)

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
[block_query.go:630-637](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/block_query.go#L630-L637)

> **设计权衡**：语法错误静默失败减少了报错注入的信息泄露，但也增加了模板作者的调试难度。

### 6.4 前端错误处理

```typescript
export const previewTemplate = (pathString: string, element: Element, parentId: string) => {
    if (!pathString) {
        element.innerHTML = "";  // 路径为空时静默清空
        return;
    }
    // fetchPost 失败时由上层统一弹 toast
};
```
[util.ts:6-18](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/toolbar/util.ts#L6-L18)

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

[router.go:368-370,472-474](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/router.go#L368-L370)

#### 7.1.1 CheckAuth 完整实现

```go
func CheckAuth(c *gin.Context) {
    // ① JWT → 接受 Admin/Editor/Reader
    if role := GetGinContextRole(c); IsValidRole(role, [...] Role{...}) { c.Next(); return }

    // ② Authorization Header → Token/Bearer 相等则设为 Admin
    if authHeader := c.GetHeader("Authorization"); "" != authHeader {
        if Conf.Api.Token == token {
            c.Set(RoleContextKey, RoleAdministrator)
            c.Next(); return
        }
        c.JSON(401, ...); c.Abort(); return
    }

    // ③ Query 参数 ?token= → 相等则设为 Admin
    if token := c.Query("token"); "" != token {
        if Conf.Api.Token == token {
            c.Set(RoleContextKey, RoleAdministrator)
            c.Next(); return
        }
        c.JSON(401, ...); c.Abort(); return
    }

    // ④ Session + localhost 检查（空授权码场景）
    localhost := util.IsLocalHost(c.Request.RemoteAddr)
    session := util.GetSession(c)
    workspaceSession := util.GetWorkspaceSession(session)
    if "" == Conf.AccessAuthCode {
        if util.SiYuanAccessAuthCodeBypass {
            c.Set(RoleContextKey, RoleAdministrator); c.Next(); return
        }
        // Origin/Host/X-Forwarded-Host 非本地时强制鉴权
    }
    if workspaceSession.AccessAuthCode == Conf.AccessAuthCode {
        c.Next(); return
    }
    // 全部失败 → 401
}
```
[session.go:207-260](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/session.go#L207-L260)

### 7.2 输入验证

**路径验证**：
```go
func IsAbsPathInWorkspace(absPath string) bool {
    return gulu.File.IsSubPath(WorkspaceDir, absPath)
}
```
[path.go:368-370](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/util/path.go#L368-L370)

**ID 格式验证**：
```go
func InvalidIDPattern(idArg string, result *gulu.Result) bool {
    if ast.IsNodeIDPattern(idArg) { return false }
    result.Code = -1
    result.Msg = "invalid block id"
    return true
}
```
[net.go:314-322](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/util/net.go#L314-L322)

**错误信息转义**：
```go
ret.Msg = util.EscapeHTML(err.Error())
```

### 7.3 危险函数移除

```go
delete(ret, "env")           // 防止获取环境变量（DB 密码、API Key 等）
delete(ret, "expandenv")     // 防止环境变量展开
delete(ret, "getHostByName") // 防止 SSRF 相关 DNS 查询
```
[template.go:38-41](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/filesys/template.go#L38-L41)

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
[snippet.go:32,54-60](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/snippet.go#L32-L60)

---

## 8. 扩展性设计

### 8.1 模板函数扩展

当前模板函数通过两层注入实现扩展：

1. **BuiltInTemplateFuncs**：通用工具函数
2. **SQLTemplateFuncs**：数据库查询函数

> **扩展点**：可通过新增 `XxxTemplateFuncs` 函数 + 修改 `SQLTemplateFuncs` 调用点来引入插件函数。

### 8.2 插件扩展点

前端通过插件系统扩展 `/` 命令菜单：

```typescript
protyle.app.plugins.forEach((plugin) => {
    plugin.protyleSlash.forEach(slash => {
        allList.push({
            filter: slash.filter,
            id: slash.id,
            value: `plugin${Constants.ZWSP}${plugin.name}${Constants.ZWSP}${slash.id}`,
            html: slash.html
        });
    });
});
```
[extend.ts:356-366](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/hint/extend.ts#L356-L366)

### 8.3 多种使用场景

模板引擎在多个场景中复用：

1. **文档模板插入**：`/api/template/render`
2. **动态图标**：`RenderDynamicIconContentTemplate`
3. **PDF 页脚**：`/api/template/renderSprig`
4. **数据库计算列**：Attribute View 计算列使用模板渲染
5. **代码片段**：用户自定义 CSS/JS（独立但共享鉴权模型）

---

## 9. 模块间协作关系

### 9.1 模板渲染数据流（完整版本）

```
┌────────────────────────────────────────────────────────────────────┐
│                           用户前端操作                              │
│  斜杠命令选模板 / 插入按钮 / 预览请求                               │
└──────────────────┬─────────────────────────────────────────────────┘
                   │  POST /api/template/render
                   │  需通过: CheckAuth → CheckAdminRole → CheckReadonly
                   ▼
┌────────────────────────────────────────────────────────────────────┐
│                     API 接口层 (api/template.go)                   │
│  - p: 模板绝对路径 (IsAbsPathInWorkspace 校验)                     │
│  - id: 目标块 ID (InvalidIDPattern 校验)                           │
│  - preview: 是否预览模式                                           │
└──────────────────┬─────────────────────────────────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────────────────────────────┐
│                     模板模型层 (model/template.go)                  │
│  Step 1: LoadTreeByBlockID(id) → 取目标块                          │
│  Step 2: 构造 dataModel = {title,id,name,alias}                    │
│  Step 3: os.ReadFile(p) → 读模板内容                               │
│          ┌─────────────── 函数注入 ────────────────┐                │
│          │  BuiltInTemplateFuncs() + SQLTemplateFuncs              │
│          │  - Sprig TxtFuncMap (剔除 env 等)                        │
│          │  - Weekday/pow/getHPathByID/statBlock/...                │
│          │  - queryBlocks / querySpans / getBlock / querySQL        │
│          └───────────────────────────────────────────────┘          │
│  Step 4: template.New("").Delims(".action{", "}").Parse(content)   │
│  Step 5: Execute(buf, dataModel)                                   │
│          ┌────────── SQL 函数调用路径（⚠ 关键）────────────┐        │
│          │  模板中 {{ queryBlocks "SELECT ... ?" .title }}          │
│          │    → strings.Replace(stmt, "?", .title, 1)               │
│          │    → SelectBlocksRawStmt(stmt, 1, 512)                   │
│          │       → sqlparser.Parse 类型断言(SELECT/UNION)           │
│          │       → 强制 LIMIT / OFFSET 注入                        │
│          │       → 转义清理 → query(stmt) 执行                      │
│          │       → rows.Scan 填充 []*Block 返回                     │
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
│                   前端交互层 (extend.ts / insertHTML.ts)            │
│  ① 渲染模式判断（空行→块级，否则行级）                               │
│  ② 场景分发：av / table / codeblock / 普通                          │
│  ③ 块级插入：构建 doOperation / undoOperation 事务数组              │
│     action ∈ {insert, update, delete} +  prevID/parentID 定位      │
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

### 9.2 代码片段数据流（修正版，含角色过滤）

```
┌─────────────┐
│ onGetConfig │  (应用启动 / 设置改变时)
└──────┬──────┘
       │ POST /api/snippet/getSnippet
       │ 权限: 仅 CheckAuth → Admin / Editor / Reader 均可
       │ 请求: { type:"all"|"js"|"css", enabled: 2|0|1 }
       ▼
┌──────────────────────────────────────────────────┐
│ getSnippet (api/snippet.go)                      │
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
│ renderSnippet (snippets.ts)                      │
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

### 10.1 安全风险（修正与补充）

#### 10.1.1 SQL 注入：分层防护但不完美

| 防护层 | 机制 | 绕过可能性 | 严重程度 |
|--------|------|-----------|---------|
| API 鉴权 | 模板渲染需 CheckAdminRole | Reader 无法直接调接口，但 Reader 可通过已插入模板的结果看到注入输出 | 中 |
| 字符串替换 | `strings.Replace(stmt, "?", arg, 1)` | **无转义**：`'`、`;`、注释符均可原样进入 | 高 |
| sqlparser 类型断言 | 仅允许 `*sqlparser.Select` / `*sqlparser.Union` | 无法直接 DML，但 UNION / 子查询 / 函数（如 `load_extension`）仍可探索 | 中高 |
| LIMIT 强制注入 | 自动加 LIMIT 512 + OFFSET | 可分页或用 UNION ALL 聚合 | 低 |
| 语法错误静默 | 错误不报给前端 | 增加盲注难度，但仍可基于响应内容长度判断 | 中 |

**高危场景示例（需管理员权限构造模板）**：
```
.action{ range queryBlocks "SELECT 1,id,content,type,box,path,hpath,root_id,parent_id,hash FROM blocks WHERE type = 'd' AND substr(content,1,1) = substr('" .title . "',1,1)" }
.action{ .Content }}.action{ end }}
```
上述做法配合布尔判断可实现逐字符提取（时间盲注变体）。

#### 10.1.2 代码片段权限：Reader 可读取全部未屏蔽片段

- 场景：发布站点开启了 Reader 账号（公开但需登录）
- 影响：
  - 含内部 API 地址、调试逻辑的 JS 片段暴露
  - CSS 中泄露的 DOM 结构信息可被爬虫利用
- 缓解：`DisabledInPublish` 标记 + 前端全局开关 `enabledJS` / `enabledCSS`
- **隐患**：读取操作**无审计日志**，无法追溯谁下载了片段内容。

#### 10.1.3 XSS 风险

- **模板侧**：SQL 查询结果（如 `Block.Content`）包含用户输入的 HTML/Markdown，经 lute 渲染后仍可能存在标签过滤漏洞。
- **片段侧**：片段本身是完全信任的，一旦管理员账号被盗，攻击者可植入持久型 XSS。
- **错误信息**：`EscapeHTML` 仅在 API 返回 `Msg` 时生效，`Content` 字段不做额外转义。

#### 10.1.4 路径遍历

- `IsAbsPathInWorkspace` 基于 `gulu.File.IsSubPath`，需要确认其对 Windows 符号链接、`..\` 重复拼接等边界场景的处理。

### 10.2 性能问题

1. **每次渲染都重新 Parse 模板**：无编译缓存，大模板或高频场景下 CPU 压力明显。
2. **AST 全量后处理**：`GenBlockIDs` / `ProcessDatabaseView` 遍历整棵树。
3. **SQL 无执行超时**：复杂子查询 + 大量数据可阻塞 Go 协程。
4. **代码片段全量传输**：前端一次性拉所有片段的 `content`，包含大量未启用的也会传输（但前端 `enabled=2` 参数默认只拉已启用的）。

### 10.3 功能缺陷

1. **错误信息不精确**：缺少行号、列号、函数签名提示。
2. **模板调试缺失**：无法 print 变量、无法断点。
3. **片段版本管理缺失**：无历史版本、无 diff、无回滚。
4. **SQL 函数文档不足**：每个函数的返回结构（Block/Span 字段）需查源码。
5. **参数替换顺序无法校验**：`?` 数量与 `args` 数量不匹配不会报错。

---

## 11. 后续研究方向

### 11.1 安全性增强

1. **真实参数化查询**：
   ```go
   // 目标：将 args 传入 db.Query，而非字符串替换
   // rows, err := db.Query(stmt, convertToAny(args)...)
   // 需要解决类型转换（Go template 传参均为 string）与 ? 数量校验
   ```

2. **SQL 函数沙箱**：
   - `context.WithTimeout` 包裹 `db.Query`（需改造底层 query 函数）
   - 扫描行数限制 + 列白名单（Block/Span 可选字段级过滤）
   - SQLite `PRAGMA trusted_schema=OFF` 等运行时加固

3. **代码片段审计**：
   - `/getSnippet` 请求写操作日志（IP / 角色 / 时间 / 筛选条件）
   - `/setSnippet` 变更写历史（保留 N 个版本）
   - 新增 `sha256(content)` 字段，前端可做完整性校验

4. **Reader 角色细分**：
   - 新增 "ReaderNoSnippet" 角色或细粒度配置项
   - 片段级 "minRole" 属性（哪些角色能读取）

### 11.2 性能优化

1. **模板编译缓存（LRU）**：
   ```go
   type templateCacheEntry struct {
       tmpl *template.Template
       usedAt time.Time
   }
   var templateCache = sync.Map{} // key: content hash
   ```

2. **增量后处理**：对 AST 中未修改子树跳过 `GenBlockIDs`。

3. **异步渲染**：复杂模板在 goroutine 中执行，前端显示进度条。

4. **片段增量传输**：HTTP 响应头 `ETag` + `If-None-Match`，`content` 不变则返回 304。

### 11.3 功能扩展

1. **模板调试工具**：
   - IDE 语法高亮 + 错误下划线
   - 内置 `debug()` 函数：输出 context / 可用函数
   - 执行时间 & 调用栈面板

2. **变量作用域扩展（按需开关）**：
   - `dataModel` 增加 `doc.updated`、`user.role`、`workspace.name` 等字段
   - 明确字段来源文档，防止误用

3. **片段沙箱**：
   - Web Worker 包裹 JS 片段，使用受控代理 API
   - CSS 前缀隔离（Shadow DOM / 命名空间）

4. **模板生命周期钩子**：
   - `preRender(p, id) → modifyContext(ctx)`
   - `postRender(md) → transform(md)`
   - `postInsert(blockIDs) → hook()`

### 11.4 架构演进

1. **插件化模板函数注册**：
   ```go
   // 设计提案
   type TemplateFuncProvider interface {
       FuncMap() template.FuncMap
       RequiredRole() Role  // 返回可使用此函数组的最小角色
   }
   ```

2. **多模板引擎**：除 `text/template` 外引入极简引擎（无循环、无函数）供 Reader 级场景使用。

3. **可视化模板编辑器**：
   - 拖拽块结构 + 绑定变量
   - SQL 构建器（参数占位符可视化 + 类型标注）
   - 实时预览 & 风险提示（高亮可能的注入点）

---

## 12. 关键代码索引

### 12.1 模板引擎核心

| 功能 | 文件位置 |
|------|---------|
| 带上下文模板渲染入口 | [template.go:309-499](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go#L309-L499) |
| 标准 Sprig 模板渲染 | [template.go:51-69](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go#L51-L69) |
| 动态图标内容模板 | [template.go:264-307](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go#L264-L307) |
| 文档另存为模板 | [template.go:183-262](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/template.go#L183-L262) |
| 内置模板函数 + 危险函数移除 | [template.go:35-63](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/filesys/template.go#L35-L63) |
| SQL 模板函数（含 strings.Replace 参数替换） | [database.go:1591-1621](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/database.go#L1591-L1621) |

### 12.2 SQL 执行层（含语法解析防护）

| 功能 | 文件位置 |
|------|---------|
| SelectBlocksRawStmt（sqlparser 类型断言 + LIMIT 注入） | [block_query.go:551-644](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/block_query.go#L551-L644) |
| SelectBlocksRawStmtNoParse（绕过解析器的回退路径） | [block_query.go:547-549](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/block_query.go#L547-L549) |
| Query（双解析器策略，querySQL 底层） | [block_query.go:366-445](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/block_query.go#L366-L445) |
| SelectSpansRawStmt（spans 表等效逻辑） | [span.go:52-93](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/span.go#L52-L93) |
| 底层 query 函数（最终 db.Query 调用点） | [database.go:1359-1369](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/sql/database.go#L1359-L1369) |

### 12.3 代码片段核心

| 功能 | 文件位置 |
|------|---------|
| 片段加载（含自动补 ID） | [snippet.go:68-111](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/snippet.go#L68-L111) |
| 片段保存（含互斥锁） | [snippet.go:32,54-60](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/snippet.go#L32-L60) |
| 前端 DOM 渲染（完整内容直接注入） | [snippets.ts:7-42](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/config/util/snippets.ts#L7-L42) |
| 片段数据结构（含 DisabledInPublish） | [snippet.go:31-38](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/conf/snippet.go#L31-L38) |
| 启动时 renderSnippet 调用点 | [onGetConfig.ts:78](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/boot/onGetConfig.ts#L78) |

### 12.4 API 接口 + 权限矩阵

| 接口 | 路由位置 | 鉴权中间件链 |
|------|---------|-------------|
| 渲染模板 | [router.go:368-370](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/router.go#L368-L370) | CheckAuth + CheckAdminRole + CheckReadonly |
| 渲染 Sprig 模板 | [template.go:28-45](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/template.go#L28-L45) | 同上 |
| 文档另存为模板 | [template.go:47-66](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/template.go#L47-L66) | 同上 |
| 获取片段 **（仅 CheckAuth）** | [router.go:472](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/router.go#L472) | CheckAuth（Reader 可过） |
| 获取片段 + 角色过滤实现 | [snippet.go:63-76](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/snippet.go#L63-L76) | IsReadOnlyRoleContext + DisabledInPublish |
| 保存片段 | [router.go:473](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/router.go#L473) | CheckAuth + CheckAdminRole + CheckReadonly |
| 删除片段 | [router.go:474](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/api/router.go#L474) | CheckAuth + CheckAdminRole + CheckReadonly |

### 12.5 角色与鉴权实现

| 功能 | 文件位置 |
|------|---------|
| CheckAuth 完整流程（4 种认证方式 + 3 种角色） | [session.go:207-260](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/session.go#L207-L260) |
| CheckAdminRole（仅 Admin 通过） | [session.go:185-193](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/session.go#L185-L193) |
| CheckReadonly（非 ReadOnly 角色 + 非只读开关） | [session.go:195-205](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/session.go#L195-L205) |
| 角色定义（Administrator/Editor/Reader/Visitor） | [role.go:1-64](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/role.go#L1-L64) |
| IsReadOnlyRoleContext 判定 | [role.go:62-64](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/model/role.go#L62-L64) |

### 12.6 前端交互与内容插入

| 功能 | 文件位置 |
|------|---------|
| 模板插入渲染（斜杠命令回调） | [extend.ts:542-564](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/hint/extend.ts#L542-L564) |
| 模板预览入口 | [util.ts:6-18](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/toolbar/util.ts#L6-L18) |
| 内容插入核心 insertHTML（事务构建） | [insertHTML.ts:264-590](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/util/insertHTML.ts#L264-L590) |
| 斜杠命令菜单（模板/插件入口） | [extend.ts:33-39](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/app/src/protyle/hint/extend.ts#L33-L39) |

### 12.7 输入验证与路径检查

| 功能 | 文件位置 |
|------|---------|
| IsAbsPathInWorkspace（路径子目录校验） | [path.go:368-370](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/util/path.go#L368-L370) |
| InvalidIDPattern（ID 格式校验） | [net.go:314-322](file:///d:/fz/0601/solo-dogfeeding/code/293-siyuan/kernel/util/net.go#L314-L322) |

---

## 总结

SiYuan 的模板与代码片段引擎是一个**分层明确、协作紧密**的系统，本次补充分析修正并深化了以下关键结论：

### ✅ 设计亮点
1. **双模板模式**：`{{ }}` 与 `.action{ }` 分隔符适应不同场景，减少语法冲突。
2. **SQL 语法层防护**：`sqlparser` 类型断言仅允许 SELECT/UNION，是防御 DML 攻击的核心屏障。
3. **分级权限模型**：`/getSnippet` 仅需 CheckAuth（Reader 可访问），`/setSnippet` 需 Admin，平衡了可用性与安全性。
4. **事务化内容插入**：所有块级修改走 `doOperation/undoOperation`，确保可撤销与数据一致性。
5. **角色二次过滤**：发布环境中 `DisabledInPublish + IsReadOnlyRoleContext` 保护敏感片段。

### ⚠️ 需重点关注的问题
1. **SQL 伪参数化**：`strings.Replace` 无转义是当前最大隐患，应尽快改为真实 `db.Query(stmt, args...)`。
2. **解析器回退路径**：双解析器均失败时跳过类型断言直接执行，存在被构造"解析器盲区"突破的风险。
3. **Reader 片段读取无审计**：建议为 `/getSnippet` 增加操作日志。
4. **SQL 结果 → Markdown → HTML → DOM → DB 整条链路**：未对 `Block.Content` 做二次输出编码，依赖 lute 单点防护。

### 🔗 各因素对核心链路的影响汇总

| 关键因素 | 变量替换阶段 | 上下文注入阶段 | 内容插入阶段 |
|---------|------------|-------------|------------|
| **CheckAuth 接受 Reader** | 不直接影响（模板渲染仍需 Admin） | 代码片段可间接读取已渲染结果 | Reader 可下载片段分析 DOM 结构 → 构造更精准注入 |
| **strings.Replace 替换 ?** | `.title` 等变量进入 SQL 时可改变语义 | SQL 返回数据成为模板循环上下文 | 注入输出的内容经 lute 渲染后直接入事务 |
| **sqlparser 仅允许 SELECT** | 防止 DML 但不防 UNION/盲注 | SQL 结果字段集合受限为 Block/Span 结构 | 输出数据仍可通过语义内容实现钓鱼或误导 |
| **DisabledInPublish** | — | Reader 上下文下敏感片段不返回 | 前端不执行敏感 JS，减少后续二次攻击面 |
| **insertHTML 事务化** | — | 不受影响 | 即使内容异常也可撤销；但已持久化的存储型 XSS 需额外机制检测 |

以上分析揭示了 SiYuan 模板系统在**安全性、可用性、可扩展性**之间的精心权衡，同时指出了在真实参数化查询、审计日志、沙箱化执行等方面的持续演进方向。