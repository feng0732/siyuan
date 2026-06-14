# SiYuan 模板与代码片段引擎实现原理分析

> **文档说明**：本文所有源码引用均使用 **仓库相对路径**（格式如 `kernel/model/template.go`），便于跨环境复核。关键代码段标注了行号范围，便于 IDE/grep 快速跳转。

---

## 1. 系统架构概述

SiYuan（思源笔记）的模板与代码片段引擎是一个分层设计的系统，由后端核心引擎、API 接口层和前端交互层协同工作。该系统基于 Go 标准库 `text/template` 构建，通过自定义分隔符、函数注入和上下文管理，实现了强大的模板渲染能力。

### 1.1 核心模块组成

| 模块 | 主要职责 | 核心文件（仓库相对路径） |
|------|---------|------------------------|
| 模板引擎核心 | 模板解析、变量替换、函数执行 | `kernel/model/template.go` |
| 内置模板函数 | 通用工具函数、日期处理、统计函数 | `kernel/filesys/template.go` |
| SQL 模板函数 | 数据库查询相关模板函数 | `kernel/sql/database.go`（L1591–L1621） |
| SQL 参数化执行 | SQL 语法解析、查询限制、行扫描 | `kernel/sql/block_query.go` |
| 代码片段管理 | CSS/JS 片段的加载、渲染、持久化 | `kernel/model/snippet.go` |
| API 接口层 | HTTP 接口定义与权限控制 | `kernel/api/template.go`, `kernel/api/snippet.go` |
| 角色与权限模型 | 角色定义、只读/管理员权限校验 | `kernel/model/role.go`, `kernel/model/session.go` |
| 前端交互层 | 模板选择、预览、插入渲染 | `app/src/protyle/hint/extend.ts`, `app/src/config/util/snippets.ts` |
| 内容插入层 | DOM 操作、事务处理、块更新 | `app/src/protyle/util/insertHTML.ts` |

---

## 2. 模板引擎核心实现原理

### 2.1 模板解析流程

模板解析采用 Go 标准库 `text/template`，但使用了自定义的分隔符和函数扩展机制。

#### 2.1.1 双模式解析系统

**模式一：标准 Go 模板语法（`{{ }}` 分隔符）**

```go
// kernel/model/template.go，函数 RenderGoTemplate（约 L51–L69）
func RenderGoTemplate(templateContent string) (ret string, err error) {
    tmpl := template.New("")
    tplFuncMap := filesys.BuiltInTemplateFuncs()
    sql.SQLTemplateFuncs(&tplFuncMap)
    tmpl = tmpl.Funcs(tplFuncMap)
    tpl, err := tmpl.Parse(templateContent)
    err = tpl.Execute(buf, nil)  // 无数据上下文
}
```

**模式二：SiYuan 动作模板（`.action{ }` 分隔符）**

```go
// kernel/model/template.go，函数 RenderTemplate（约 L309–L499）
func RenderTemplate(p, id string, preview bool) (tree *parse.Tree, dom string, err error) {
    goTpl := template.New("").Delims(".action{", "}")  // 自定义分隔符
    tplFuncMap := filesys.BuiltInTemplateFuncs()
    sql.SQLTemplateFuncs(&tplFuncMap)
    goTpl = goTpl.Funcs(tplFuncMap)
    tpl, err := goTpl.Funcs(tplFuncMap).Parse(gulu.Str.FromBytes(md))
    err = tpl.Execute(buf, dataModel)  // 注入数据上下文
}
```

> **设计意图**：使用 `.action{ }` 作为分隔符可以避免与 Markdown 中的其他语法冲突。

### 2.2 变量替换与上下文注入

#### 2.2.1 数据模型（Data Model）

模板渲染时注入的上下文变量经过严格限制，仅包含 4 个字符串字段：

```go
// kernel/model/template.go（约 L326–L337）
dataModel := map[string]string{
    "title": titleVar,    // 文档标题或块内容（经过 strings.TrimSpace 清理）
    "id":    block.ID,    // 块 ID（23 位固定格式）
    "name":  block.Name,  // 块名称
    "alias": block.Alias, // 块别名
}
```

> **作用域限制设计**：只注入 `map[string]string` 而非结构体，避免通过反射暴露更多字段。

#### 2.2.2 模板函数注入

模板函数通过两层注入机制构建：

**第一层：Sprig 通用函数库 + 自定义函数**

```go
// kernel/filesys/template.go，函数 BuiltInTemplateFuncs（约 L35–L63）
func BuiltInTemplateFuncs() (ret template.FuncMap) {
    ret = sprig.TxtFuncMap()
    delete(ret, "env")           // 防止获取环境变量
    delete(ret, "expandenv")     // 防止环境变量展开
    delete(ret, "getHostByName") // 防止 SSRF 相关 DNS 查询
    // 日期、数学、内容处理函数...
}
```

**第二层：SQL 查询函数**

```go
// kernel/sql/database.go，函数 SQLTemplateFuncs（L1591–L1621）
func SQLTemplateFuncs(templateFuncMap *template.FuncMap) {
    (*templateFuncMap)["queryBlocks"] = func(stmt string, args ...string) (retBlocks []*Block) {
        for _, arg := range args {
            stmt = strings.Replace(stmt, "?", arg, 1)
        }
        retBlocks = SelectBlocksRawStmt(stmt, 1, 512)
        return
    }
    (*templateFuncMap)["getBlock"] = func(arg any) (retBlock *Block) { /* ... */ }
    (*templateFuncMap)["querySpans"] = func(stmt string, args ...string) (retSpans []*Span) {
        for _, arg := range args {
            stmt = strings.Replace(stmt, "?", arg, 1)
        }
        retSpans = SelectSpansRawStmt(stmt, 512)
        return
    }
    (*templateFuncMap)["querySQL"] = func(stmt string) (ret []map[string]any) {
        ret, _ = Query(stmt, 1024)
        return
    }
}
```

---

## 3. SQL 模板查询限制：代码级深度分析（重点章节）

### 3.1 三个 SQL 模板函数的完整对照

| 维度 | `queryBlocks` | `querySpans` | `querySQL` |
|------|--------------|-------------|-----------|
| **签名** | `func(stmt string, args ...string) []*Block` | `func(stmt string, args ...string) []*Span` | `func(stmt string) []map[string]any` |
| **参数替换** | ✅ `strings.Replace(stmt, "?", arg, 1)` | ✅ `strings.Replace(stmt, "?", arg, 1)` | ❌ **无 args 参数，无替换** |
| **底层调用** | `SelectBlocksRawStmt(stmt, 1, 512)` | `SelectSpansRawStmt(stmt, 512)` | `Query(stmt, 1024)` |
| **默认 LIMIT** | 512 | 512 | **1024** |
| **返回类型** | `[]*Block`（21 个字段强类型结构体） | `[]*Span`（7 个字段强类型结构体） | `[]map[string]any`（动态任意列） |
| **SQL 解析器** | `sqlparser`（vitess） | `sqlparser`（vitess） | `sqlparser2` → `sqlparser` → `queryRawStmt`（三重回退） |
| **允许的语句类型** | SELECT / UNION | **仅 SELECT** | SELECT / UNION（sqlparser2 仅 SELECT） |
| **解析失败回退** | `selectBlocksRawStmt`（Go 代码层计数限行） | **直接返回空** | `queryRawStmt`（Go 代码层计数限行） |
| **代码位置** | `kernel/sql/database.go#L1592-L1598` | `kernel/sql/database.go#L1610-L1615` | `kernel/sql/database.go#L1617-L1619` |

### 3.2 关键发现：512 和 1024 是默认值而非硬性上限

经逐行阅读底层函数代码，发现三个函数的 LIMIT 参数**并非硬性上限**。当模板 SQL 中自带 LIMIT 子句时，解析器会**尊重用户指定的值**，覆盖默认的 512/1024。

#### 3.2.1 `SelectBlocksRawStmt` 的 LIMIT 处理逻辑

```go
// kernel/sql/block_query.go，函数 SelectBlocksRawStmt（约 L551–L644）
func SelectBlocksRawStmt(stmt string, page, limit int) (ret []*Block) {
    parsedStmt, err := sqlparser.Parse(stmt)
    if err != nil {
        return selectBlocksRawStmt(stmt, limit)  // 解析失败回退
    }

    switch parsedStmt.(type) {
    case *sqlparser.Select:
        slct := parsedStmt.(*sqlparser.Select)
        if nil == slct.Limit {
            // ❶ 无 LIMIT → 注入默认值 512 + OFFSET 0
            slct.Limit = &sqlparser.Limit{
                Rowcount: &sqlparser.SQLVal{Val: []byte(strconv.Itoa(limit))},
                Offset:   &sqlparser.SQLVal{Val: []byte(strconv.Itoa((page - 1) * limit))},
            }
        } else {
            // ❷ 有 LIMIT → 读取用户值，无上限校验！
            limit, _ = strconv.Atoi(string(slct.Limit.Rowcount.(*sqlparser.SQLVal).Val))
            if 0 >= limit {
                limit = 32  // 仅防护零值/负值
            }
            // 重新写入用户的 LIMIT 值（无上限封顶！）
            slct.Limit.Rowcount = &sqlparser.SQLVal{Val: []byte(strconv.Itoa(limit))}
            slct.Limit.Offset = &sqlparser.SQLVal{Val: []byte(strconv.Itoa((page - 1) * limit))}
        }
        stmt = sqlparser.String(slct)
    case *sqlparser.Union:
        // 同样逻辑：有 LIMIT → 读取用户值；无 → 注入默认
    default:
        return  // 非 SELECT/UNION 直接拒绝
    }
    // ...
}
```

**关键行为**：

| 条件 | 实际行为 | 示例 |
|------|---------|------|
| SQL 无 LIMIT | 注入默认 `LIMIT 512 OFFSET 0` | `SELECT * FROM blocks` → 返回 ≤512 行 |
| SQL 有 `LIMIT 100` | 使用用户值 `100`（≤512 则更低） | 正常场景，实际限制更低 |
| SQL 有 `LIMIT 99999` | 使用用户值 `99999`（远超 512！） | ⚠️ **默认 512 被绕过** |

#### 3.2.2 `SelectSpansRawStmt` 的 LIMIT 处理逻辑

```go
// kernel/sql/span.go，函数 SelectSpansRawStmt（约 L52–L93）
func SelectSpansRawStmt(stmt string, limit int) (ret []*Span) {
    parsedStmt, err := sqlparser.Parse(stmt)
    if err != nil {
        return  // ⚠️ 解析失败直接返回空（无回退路径）
    }
    switch parsedStmt.(type) {
    case *sqlparser.Select:
        slct := parsedStmt.(*sqlparser.Select)
        if nil == slct.Limit {
            // 无 LIMIT → 注入默认值 512
            slct.Limit = &sqlparser.Limit{
                Rowcount: &sqlparser.SQLVal{Val: []byte(strconv.Itoa(limit))},
            }
        }
        // ⚠️ 如果 slct.Limit 已存在（用户写了 LIMIT），则完全不修改！
        stmt = sqlparser.String(slct)
    default:
        return  // 非 SELECT 直接拒绝（UNION 也不支持）
    }
    // ...
}
```

**关键行为**：

| 条件 | 实际行为 |
|------|---------|
| SQL 无 LIMIT | 注入 `LIMIT 512` |
| SQL 有 `LIMIT 99999` | ⚠️ **保留用户的 99999，完全不覆盖** |
| SQL 解析失败 | 直接返回空（与 queryBlocks 的回退不同） |

#### 3.2.3 `Query` 的 LIMIT 处理逻辑（querySQL 底层）

```go
// kernel/sql/block_query.go，函数 Query（约 L366–L445）
func Query(stmt string, limit int) (ret []map[string]any, err error) {
    originalStmt := stmt
    // ① 尝试 sqlparser2（支持 || 连接符）
    p := sqlparser2.NewParser(strings.NewReader(stmt))
    parsedStmt2, err := p.ParseStatement()
    if err != nil {
        if !strings.Contains(stmt, "||") {
            // ② 回退到 sqlparser（支持 UNION）
            parsedStmt, err2 := sqlparser.Parse(stmt)
            if nil != err2 {
                return queryRawStmt(stmt, limit)  // ③ 两个解析器都失败 → 无类型检查！
            }
            // sqlparser 成功：类型断言 + LIMIT 注入
            switch parsedStmt.(type) {
            case *sqlparser.Select:
                limitClause := getLimitClause(parsedStmt, limit)  // 读取用户 LIMIT 或注入默认
                slct := parsedStmt.(*sqlparser.Select)
                slct.Limit = limitClause
                stmt = sqlparser.String(slct)
            case *sqlparser.Union:
                // 同上
            default:
                return queryRawStmt(stmt, limit)  // 非 SELECT/UNION → 回退（⚠ 无拒绝！）
            }
        } else {
            return queryRawStmt(stmt, limit)  // 含 || 且 sqlparser2 失败 → 回退
        }
    } else {
        // sqlparser2 成功
        switch parsedStmt2.(type) {
        case *sqlparser2.SelectStatement:
            slct := parsedStmt2.(*sqlparser2.SelectStatement)
            if nil == slct.LimitExpr {
                slct.LimitExpr = &sqlparser2.NumberLit{Value: strconv.Itoa(limit)}
            }
            // ⚠️ 如果 LimitExpr 已存在，不覆盖！
            stmt = slct.String()
        default:
            return queryRawStmt(stmt, limit)  // 非 SELECT → 回退（⚠ 无拒绝！）
        }
    }
    // 执行查询...
}
```

`getLimitClause` 的行为：

```go
// kernel/sql/block_query.go，函数 getLimitClause（约 L479–L502）
func getLimitClause(parsedStmt sqlparser.Statement, limit int) (ret *sqlparser.Limit) {
    switch parsedStmt.(type) {
    case *sqlparser.Select:
        if nil != parsedStmt.(*sqlparser.Select).Limit {
            ret = parsedStmt.(*sqlparser.Select).Limit  // 保留用户 LIMIT
        }
    case *sqlparser.Union:
        // 同上
    }
    if nil == ret || nil == ret.Rowcount {
        ret = &sqlparser.Limit{Rowcount: &sqlparser.SQLVal{Val: []byte(strconv.Itoa(limit))}}
    }
    return
}
```

**Query 的三重回退路径**：

| 回退路径 | 触发条件 | 类型检查 | LIMIT 控制 | 风险等级 |
|---------|---------|---------|-----------|---------|
| sqlparser2 SELECT | `||` 操作符 + 解析成功 | 仅 SELECTStatement | 用户 LIMIT 优先，否则默认 1024 | 低 |
| sqlparser SELECT/UNION | 无 `\|\|` + 解析成功 | SELECT / UNION | 用户 LIMIT 优先，否则默认 1024 | 低 |
| `queryRawStmt` | **两个解析器都失败** 或 **非 SELECT** | ❌ **无类型检查** | `containsLimitClause` 检查 | **高** |

#### 3.2.4 `queryRawStmt` 回退路径的隐患

```go
// kernel/sql/block_query.go，函数 queryRawStmt（约 L504–L545）
func queryRawStmt(stmt string, limit int) (ret []map[string]any, err error) {
    rows, err := query(stmt)  // 直接执行，无类型检查！
    // ...
    noLimit := !containsLimitClause(stmt)
    var count int
    for rows.Next() {
        // ... scan ...
        ret = append(ret, m)
        count++
        if noLimit && limit < count {  // ⚠️ 仅在 noLimit 时才限行
            break
        }
    }
    return
}
```

`containsLimitClause` 实现：

```go
// kernel/sql/block_query.go（约 L945–L949）
func containsLimitClause(stmt string) bool {
    return strings.Contains(strings.ToLower(stmt), " limit ") ||
        strings.Contains(strings.ToLower(stmt), "\nlimit ") ||
        strings.Contains(strings.ToLower(stmt), "\tlimit ")
}
```

**`queryRawStmt` 的双重隐患**：

1. **无类型检查**：DML/DDL 语句（DELETE/DROP/INSERT 等）在两个解析器都失败时会**直接执行**，因为 `queryRawStmt` 没有 SELECT 类型断言。
2. **LIMIT 检测可绕过**：`containsLimitClause` 仅做字符串匹配（`" limit "` / `\nlimit ` / `\tlimit `），攻击者可通过注释符（`SELECT...FROM...WHERE 1=1/**/LIMIT 99999`）、无空格拼接（`...LIMIT`）等方式绕过，使 `noLimit=true` 判定为 `false`，从而跳过计数限行。

### 3.3 `querySQL` 无参数替换的设计含义

三个模板函数中，只有 `querySQL` 不接受 `args ...string` 参数：

```go
// kernel/sql/database.go#L1592-L1598
(*templateFuncMap)["queryBlocks"] = func(stmt string, args ...string) (retBlocks []*Block) {
    for _, arg := range args {
        stmt = strings.Replace(stmt, "?", arg, 1)  // 字符串替换
    }
    retBlocks = SelectBlocksRawStmt(stmt, 1, 512)
    return
}

// kernel/sql/database.go#L1617-L1619
(*templateFuncMap)["querySQL"] = func(stmt string) (ret []map[string]any) {
    ret, _ = Query(stmt, 1024)  // 无 args，无字符串替换
    return
}
```

**这一差异的含义**：

| 维度 | queryBlocks / querySpans | querySQL |
|------|-------------------------|----------|
| 参数来源 | `stmt` + `args`，args 来自模板变量（如 `.title`） | 仅 `stmt`，整条 SQL 在模板中直接构造 |
| 注入风险 | `.title` 经 `strings.Replace` 进入 SQL，**无转义** | 整条 SQL 由模板作者构造，无运行时变量注入点 |
| 安全影响 | 模板变量中的特殊字符可改变 SQL 语义 | 安全边界更清晰：SQL 在模板源码中可见，无隐式拼接 |
| 使用场景 | `queryBlocks "SELECT * FROM blocks WHERE content LIKE ?" .title` | `querySQL "SELECT COUNT(*) FROM blocks"` |

> `querySQL` 不接受 args 实际上是**更安全**的设计：它消除了运行时变量通过 `strings.Replace` 注入 SQL 的攻击面。但代价是模板作者必须在模板中硬编码查询条件，或使用 Go 模板的字符串拼接功能自行构造 SQL（这同样无转义保护）。

### 3.4 三个函数的完整执行路径对比

```
queryBlocks(stmt, args...)
    │
    ├─ strings.Replace 循环替换 ?
    │
    └─ SelectBlocksRawStmt(stmt, 1, 512)
         │
         ├─ sqlparser.Parse 成功
         │    ├─ SELECT → 用户 LIMIT 优先 / 注入默认 512 + OFFSET
         │    ├─ UNION → 同上
         │    └─ 其他 → 返回空
         │
         └─ sqlparser.Parse 失败
              └─ selectBlocksRawStmt(stmt, 512)
                   └─ query(stmt) 直接执行
                        └─ containsLimitClause → noLimit 时计数限行

querySpans(stmt, args...)
    │
    ├─ strings.Replace 循环替换 ?
    │
    └─ SelectSpansRawStmt(stmt, 512)
         │
         ├─ sqlparser.Parse 成功
         │    ├─ SELECT → 用户 LIMIT 优先 / 注入默认 512（无 OFFSET）
         │    └─ 其他 → 返回空（不支持 UNION！）
         │
         └─ sqlparser.Parse 失败
              └─ 直接返回空（无回退执行路径！）

querySQL(stmt)
    │
    └─ Query(stmt, 1024)
         │
         ├─ sqlparser2.Parse 成功
         │    └─ SelectStatement → 用户 LIMIT 优先 / 注入默认 1024
         │
         ├─ sqlparser2 失败 && 不含 ||
         │    ├─ sqlparser.Parse 成功
         │    │    ├─ SELECT → getLimitClause → 用户 LIMIT 优先 / 默认 1024
         │    │    ├─ UNION → 同上
         │    │    └─ 其他 → queryRawStmt（⚠ 无类型检查！）
         │    │
         │    └─ sqlparser.Parse 失败
         │         └─ queryRawStmt(stmt, 1024)（⚠ 无类型检查！）
         │
         └─ sqlparser2 失败 && 含 ||
              └─ queryRawStmt(stmt, 1024)（⚠ 无类型检查！）
```

---

## 4. 512 vs 1024 对核心链路的三阶段影响分析

模板引擎的数据流可拆解为三个阶段：**变量替换 → 上下文注入 → 内容插入**。SQL 限制的差异在每个阶段产生不同级别的传导效应。

### 4.1 阶段一：变量替换（Template Execute）

```
模板文本                                    Go text/template 执行
.action{ queryBlocks "SELECT ... ?" .title }
         │                     │
         └── stmt 参数 ────────┤
         └── .title → args ────┤
                               ▼
                    strings.Replace(stmt, "?", .title, 1)
                               │
                               ▼
                    SelectBlocksRawStmt(拼接后的 stmt, 1, 512)
```

**512 vs 1024 在此阶段的影响**：

- LIMIT 参数**不影响变量替换本身**——无论限制是 512 还是 1024，`strings.Replace` 对 `?` 的替换行为完全相同（无转义、无校验）。
- 但 LIMIT 决定了**替换后的 SQL 能返回多少数据**。`querySQL` 的 1024 上限意味着替换（虽然它不使用 args 替换，但模板字符串拼接的效果等价）成功后可获取两倍于 `queryBlocks` 的记录量。
- **更关键的是**：由于 512/1024 是默认值而非硬性上限（见 3.2 节），模板作者若在 SQL 中显式写 `LIMIT 99999`，三个函数都会**尊重用户值**，默认限制形同虚设。

### 4.2 阶段二：上下文注入（range 循环渲染）

SQL 返回结果成为模板 `range` 循环的数据源，注入到渲染上下文中：

```
SQL 返回结果                         模板 range 循环
[]*Block (≤512 或用户 LIMIT)   →   .action{ range queryBlocks "..." }
                                   .action{ .Content }  ← 每个块的 Content 字段
[]*Span  (≤512 或用户 LIMIT)    →   .action{ range querySpans "..." }
                                   .action{ .Content }  ← 每个行内元素的 Content
[]map[string]any (≤1024 或用户 LIMIT) → .action{ range querySQL "..." }
                                        .action{ .column_name }  ← 任意列
```

**512 vs 1024 在此阶段的影响**：

| 影响维度 | queryBlocks (512) / querySpans (512) | querySQL (1024) |
|---------|--------------------------------------|-----------------|
| **循环迭代次数** | ≤512 次（默认） | ≤1024 次（默认），翻倍 |
| **单次迭代数据量** | Block: 21 字段（含 Content, Markdown, IAL 等） | map: 任意列，取决于 SELECT 的列数 |
| **总渲染输出量** | 512 × Block 平均大小 ≈ 数十 KB ~ 数百 KB | 1024 × 动态列数 ≈ 可能达 MB 级 |
| **内存占用** | Block 结构体内存在栈上分配，GC 压力可控 | map[string]any 每行独立分配，GC 压力较大 |
| **模板超时风险** | 低 | 中高（1024 次循环 + 每次可能触发嵌套查询） |
| **XSS 触发面** | Block.Content/Markdown 可能含用户输入 | 任意列可能含用户输入，字段不可预测 |

**核心风险**：`querySQL` 返回 `map[string]any`，模板作者可直接访问**任意列**，包括其他表的敏感字段（如 `SELECT * FROM attributes` 或跨表 JOIN）。而 `queryBlocks` / `querySpans` 的返回类型固定，字段集合是已知且可控的。

### 4.3 阶段三：内容插入（insertHTML → DOM → 事务）

渲染后的 Markdown/HTML 通过 `insertHTML` 进入事务系统，最终写入数据库：

```
模板渲染输出（Markdown）        lute 引擎转换             insertHTML
".action{ .Content }..."  →  HTML 字符串  →  range.insertNode / transaction
                                                    │
                                                    ▼
                                              /api/transactions
                                                    │
                                                    ▼
                                           写入 blocks / spans 表
```

**512 vs 1024 在此阶段的影响**：

| 影响维度 | 512 限制 | 1024 限制 |
|---------|---------|----------|
| **DOM 节点数** | 512 个 Block × 每个 Block 可能生成 5-20 个 DOM 节点 ≈ 2500-10000 | 1024 × 动态内容 → 可能生成 **2-4 倍** DOM 节点 |
| **事务体积** | `doOperation` 数组大小与 Block 数成正比 | 1024 行的 `querySQL` 可能产生 1024+ 个 insert 操作 |
| **前端性能** | 中等规模 DOM 操作，浏览器可承受 | 大规模 DOM 操作可能导致**渲染卡顿** |
| **持久化影响** | 存储型内容在合理范围内 | 大量恶意内容被持久化，后续所有打开该文档的用户受影响 |
| **撤销栈** | 单次 undo 可回退 | 超大事务的 undo 操作本身也可能卡顿 |

### 4.4 三阶段综合影响图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     变量替换阶段                                     │
│  .title ──→ strings.Replace ──→ SQL 语句                            │
│                                                                      │
│  ⚠ 512/1024 不影响替换过程本身                                       │
│  ⚠ 但决定注入成功后可获取的数据量                                     │
│  ⚠ 用户自定义 LIMIT 可覆盖默认值                                     │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ SQL 返回结果
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     上下文注入阶段                                    │
│  range 循环迭代 × N 条结果                                           │
│                                                                      │
│  queryBlocks: ≤512 × 21字段     ┃  querySQL: ≤1024 × 动态列数        │
│  ─────────────────────────────  ┃  ──────────────────────────────    │
│  渲染输出量: 中                 ┃  渲染输出量: 高（2x+）              │
│  GC 压力: 低                    ┃  GC 压力: 高（每行独立 map 分配）   │
│  字段可预测: 是                 ┃  字段可预测: 否（任意列可访问）      │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ 渲染后的 HTML
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     内容插入阶段                                      │
│  insertHTML → transaction → /api/transactions → 写入数据库            │
│                                                                      │
│  DOM 节点数: 与 SQL 返回行数正相关                                    │
│  事务体积: 与 Block/map 数量成正比                                    │
│  持久化影响: 存储型内容影响所有后续读者                                │
│  性能风险: 1024 行可能导致渲染卡顿和事务超时                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. 代码片段（Snippet）引擎

### 5.1 代码片段数据结构

```go
// kernel/conf/snippet.go（约 L31–L38）
type Snippet struct {
    ID                string `json:"id"`
    Name              string `json:"name"`
    Type              string `json:"type"`              // "js" 或 "css"
    Enabled           bool   `json:"enabled"`           // 是否启用
    DisabledInPublish bool   `json:"disabledInPublish"` // 发布模式下是否禁用
    Content           string `json:"content"`           // 完整内容
}
```

### 5.2 读取权限深度分析

#### 5.2.1 API 路由权限矩阵

```go
// kernel/api/router.go（约 L472–L474）
ginServer.Handle("POST", "/api/snippet/getSnippet",
    model.CheckAuth,            // 仅需登录（Admin/Editor/Reader 均通过）
    getSnippet)

ginServer.Handle("POST", "/api/snippet/setSnippet",
    model.CheckAuth,            // ① 登录
    model.CheckAdminRole,       // ② 管理员角色
    model.CheckReadonly,        // ③ 非只读模式
    setSnippet)
```

#### 5.2.2 权限分析结论

| 角色 | 读取片段 `/getSnippet` | 修改片段 `/setSnippet` | 删除片段 `/removeSnippet` |
|------|----------------------|----------------------|------------------------|
| Administrator | ✅ 全部内容 | ✅ 是 | ✅ 是 |
| Editor | ✅ 全部内容 | ❌ 被 CheckAdminRole 拦截 | ❌ |
| Reader | ✅ 但 `DisabledInPublish=true` 被过滤 | ❌ | ❌ |
| Visitor | ❌ CheckAuth 不接受 | ❌ | ❌ |

### 5.3 前端渲染逻辑

```typescript
// app/src/config/util/snippets.ts（约 L7–L42）
export const renderSnippet = () => {
    fetchPost("/api/snippet/getSnippet", {type: "all", enabled: 2}, (response) => {
        response.data.snippets.forEach((item: ISnippet) => {
            // CSS → <style id="snippetCSS{id}">.insertAdjacentHTML("beforeend")
            // JS  → <script id="snippetJS{id}">.text = content → appendChild
            // ⚠️ 完全信任内容，直接注入 DOM
        });
    });
};
```

### 5.4 代码片段对变量替换、上下文注入、内容插入的影响

代码片段与模板引擎分属两个独立系统，但在运行时存在间接耦合：

1. **变量替换的可见性**：模板渲染时 SQL 查询得到的敏感数据（最多 1024 行 `querySQL` 结果），一旦插入 DOM，代码片段的 JS 可通过 `textContent`、`dataset` 读取。
2. **上下文注入的跨域**：代码片段与模板使用不同机制（前端 vs 后端），但共享 `window.siyuan` API，片段可调用 `fetchPost` 再次请求模板渲染接口，形成循环。
3. **内容插入的后置修改**：`insertHTML` 提交事务后，代码片段可通过 JS 进一步修改 DOM（绕过编辑器校验），这些修改不会反映到数据库，刷新后丢失。

---

## 6. 完整渲染流程（含 SQL 限制标注）

```
用户触发模板插入（斜杠菜单 / 插入按钮）
        ↓
前端调用 /api/template/render  [需 CheckAuth + CheckAdminRole + CheckReadonly]
        ↓
┌─────────────────────────────────┐
│  1. 加载目标块上下文            │
│     - LoadTreeByBlockID(id)     │
│     - 构建 dataModel (4 字段)   │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  2. 模板解析                    │
│     - 注入 BuiltInTemplateFuncs │
│     - 注入 SQLTemplateFuncs     │
│     - Parse 模板内容            │
└─────────────────────────────────┘
        ↓
┌──────────────────────────────────────────────────┐
│  3. 变量替换与函数执行                            │
│     - Execute(dataModel)                          │
│     - SQL 函数执行路径：                           │
│       queryBlocks → Replace(?) → SelectBlocksRaw  │
│         → sqlparser → 用户 LIMIT 优先 / 默认 512  │
│       querySpans  → Replace(?) → SelectSpansRaw   │
│         → sqlparser → 用户 LIMIT 优先 / 默认 512  │
│       querySQL    → 无替换 → Query                │
│         → 双解析器 → 用户 LIMIT 优先 / 默认 1024   │
│         → 解析失败 → queryRawStmt（⚠ 无类型检查）  │
│     - 渲染为 Markdown 文本                        │
└──────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│  4. 后处理（AST 遍历）          │
│     - GenBlockIDs() 重新生成ID  │
│     - TranscludeRef() 处理引用  │
│     - ProcessDatabaseView()     │
└─────────────────────────────────┘
        ↓
前端接收 content HTML → insertHTML 插入 → 事务提交 → 持久化
```

---

## 7. 安全机制分析

### 7.1 API 鉴权体系

| API 端点 | CheckAuth | CheckAdminRole | CheckReadonly | 可访问角色 |
|----------|:---------:|:--------------:|:-------------:|-----------|
| `/api/template/render` | ✅ | ✅ | ✅ | Administrator |
| `/api/snippet/getSnippet` | ✅ | ❌ | ❌ | Administrator, Editor, Reader |
| `/api/snippet/setSnippet` | ✅ | ✅ | ✅ | Administrator |

### 7.2 SQL 注入防护分层

| 防护层 | 机制 | 绕过可能性 |
|--------|------|-----------|
| API 鉴权 | 模板渲染需 CheckAdminRole | Reader 无法直接调接口 |
| 字符串替换 | `strings.Replace(stmt, "?", arg, 1)` | **无转义**：`'`、`;`、注释符均可原样进入 |
| sqlparser 类型断言 | 仅允许 SELECT / UNION | 无法直接 DML |
| LIMIT 默认注入 | 无 LIMIT 时注入 512/1024 | **用户自定义 LIMIT 可覆盖默认值** |
| 语法错误静默 | 错误不报给前端 | 增加盲注难度，但可基于响应长度判断 |
| queryRawStmt 回退 | 双解析器失败时直接执行 | ⚠️ **无类型检查**，DML 可能执行 |

### 7.3 危险函数移除

```go
// kernel/filesys/template.go（约 L38–L41）
delete(ret, "env")           // 防止获取环境变量
delete(ret, "expandenv")     // 防止环境变量展开
delete(ret, "getHostByName") // 防止 SSRF 相关 DNS 查询
```

---

## 8. 作用域限制机制

### 8.1 模板执行作用域

1. **数据上下文限制**：`RenderGoTemplate` 无数据注入；`RenderTemplate` 仅注入 `title/id/name/alias` 四个 `string` 类型变量。
2. **函数执行沙箱**：所有函数均为纯函数或只读操作；移除 `env/expandenv/getHostByName`；SQL 查询仅允许 SELECT/UNION。
3. **文件系统限制**：模板文件必须位于 `workspace/templates/` 目录下，通过 `IsAbsPathInWorkspace` 验证。

### 8.2 代码片段作用域

1. **全局开关控制**：`window.siyuan.config.snippet.enabledCSS` / `enabledJS`。
2. **单片段启用状态**：`item.enabled` 为 `false` 时不注入。
3. **发布环境隔离**：`IsReadOnlyRoleContext(c) && s.DisabledInPublish → skip`。

---

## 9. 错误提示机制

### 9.1 模板错误处理

```go
// kernel/model/template.go（约 L57–L58）
tpl, err := tmpl.Parse(templateContent)
if err != nil {
    return "", fmt.Errorf(Conf.Language(44), err.Error())  // "渲染模板失败：%s"
}
```

### 9.2 SQL 静默失败策略

```go
// kernel/sql/block_query.go（约 L630–L637）
rows, err := query(stmt)
if err != nil {
    if strings.Contains(err.Error(), "syntax error") {
        return  // 语法错误静默返回空，不暴露错误细节
    }
    logging.LogWarnf("sql query [%s] failed: %s", stmt, err)
    return
}
```

> **设计权衡**：语法错误静默失败减少了报错注入的信息泄露，但也增加了模板作者的调试难度。

---

## 10. 潜在问题与风险

### 10.1 SQL LIMIT 默认值可被用户覆盖

**这是本次分析发现的最关键问题**：

三个 SQL 模板函数的 512/1024 限制仅是**无 LIMIT 子句时的默认注入值**，并非硬性上限。当模板 SQL 中包含 `LIMIT N` 时：

- `SelectBlocksRawStmt`：读取用户值并使用（仅防护零值/负值，改为 32）
- `SelectSpansRawStmt`：保留用户值，完全不覆盖
- `Query` / `getLimitClause`：保留用户值，完全不覆盖
- `queryRawStmt`（回退路径）：若 SQL 含 `LIMIT` 字样则跳过计数限行

**风险场景**：

```
.action{ range queryBlocks "SELECT * FROM blocks LIMIT 99999" }
.action{   .Content }
.action{ end }
```

此模板将绕过 512 默认限制，一次返回最多 99999 条 Block 记录。

### 10.2 `querySQL` 的 `queryRawStmt` 回退路径无类型检查

当 `sqlparser2` 和 `sqlparser` 均解析失败时，`Query` 函数回退到 `queryRawStmt`，后者：
- 无 SELECT/UNION 类型断言
- 直接调用 `query(stmt)` 执行任意 SQL
- LIMIT 仅通过 `containsLimitClause` 字符串匹配检测

### 10.3 SQL 伪参数化

`queryBlocks` / `querySpans` 的 `strings.Replace` 无转义是持续的隐患。

### 10.4 `querySQL` 返回 `map[string]any` 字段不可控

模板作者可通过 `querySQL` 查询任意表、访问任意列，包括敏感字段。

### 10.5 代码片段权限

- `/getSnippet` 仅需 CheckAuth，Editor/Reader 可读取全部未屏蔽片段
- 读取操作无审计日志

### 10.6 性能问题

1. 每次渲染都重新 Parse 模板，无编译缓存
2. SQL 无执行超时，复杂子查询可阻塞 Go 协程
3. `querySQL` 的 1024 行 × 动态列数可能导致模板渲染输出膨胀

---

## 11. 后续研究方向

### 11.1 安全性增强

1. **将 LIMIT 默认值改为硬性上限**：
   ```go
   // 建议修改：在 SelectBlocksRawStmt / SelectSpansRawStmt / Query 中
   // 读取用户 LIMIT 后，与硬性上限取较小值
   if userLimit > hardCap {
       userLimit = hardCap  // 硬性封顶
   }
   ```

2. **统一 SQL 限制常量**：
   ```go
   const (
       TemplateQueryBlocksLimit = 512
       TemplateQuerySpansLimit  = 512
       TemplateQuerySQLLimit    = 1024
   )
   ```

3. **真实参数化查询**：将 `strings.Replace` 改为 `db.Query(stmt, args...)`。

4. **queryRawStmt 增加类型检查**：在回退路径中增加正则检测，拒绝非 SELECT 语句。

5. **querySQL 返回字段白名单**：对 `map[string]any` 的 key 做白名单过滤。

6. **代码片段审计日志**：为 `/getSnippet` 增加操作日志。

### 11.2 性能优化

1. 模板编译缓存（LRU）
2. SQL 执行超时（`context.WithTimeout`）
3. 代码片段增量传输（ETag + 304）

### 11.3 功能扩展

1. 模板调试工具（语法高亮 + 错误定位）
2. SQL 构建器（参数占位符可视化 + 类型标注）
3. 片段沙箱（Web Worker / Shadow DOM）

---

## 12. 源码检索指引（仓库相对路径）

> 跨环境复核说明：下表中的所有路径均为相对仓库根目录的路径。文件后的行号区间为该功能的大致代码范围，不同版本可能略有差异。

### 12.1 模板引擎核心

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| 带上下文模板渲染 RenderTemplate | `kernel/model/template.go`（约 L309–L499） |
| 标准 Sprig 模板渲染 RenderGoTemplate | `kernel/model/template.go`（约 L51–L69） |
| 内置模板函数 + 危险函数移除 BuiltInTemplateFuncs | `kernel/filesys/template.go`（约 L35–L63） |
| SQL 模板函数注册 SQLTemplateFuncs | `kernel/sql/database.go`（L1591–L1621，精确） |

### 12.2 SQL 执行层（含 LIMIT 处理逻辑）

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| SelectBlocksRawStmt（用户 LIMIT 优先 / 注入默认 512） | `kernel/sql/block_query.go`（约 L551–L644） |
| SelectBlocksRawStmtNoParse（绕过解析器） | `kernel/sql/block_query.go`（约 L547–L549） |
| selectBlocksRawStmt（解析失败回退，containsLimitClause 限行） | `kernel/sql/block_query.go`（约 L698–L724） |
| Query（双解析器 + queryRawStmt 回退，默认 1024） | `kernel/sql/block_query.go`（约 L366–L445） |
| getLimitClause（用户 LIMIT 优先 / 注入默认） | `kernel/sql/block_query.go`（约 L479–L502） |
| queryRawStmt（无类型检查，containsLimitClause 限行） | `kernel/sql/block_query.go`（约 L504–L545） |
| containsLimitClause（字符串匹配 LIMIT 子句） | `kernel/sql/block_query.go`（约 L945–L949） |
| SelectSpansRawStmt（用户 LIMIT 优先 / 注入默认 512） | `kernel/sql/span.go`（约 L52–L93） |
| 底层 query 函数（最终 db.Query 调用点） | `kernel/sql/database.go`（约 L1359–L1369） |

### 12.3 代码片段核心

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| 片段加载 loadSnippets | `kernel/model/snippet.go`（约 L68–L111） |
| 片段保存 SetSnippet（含互斥锁） | `kernel/model/snippet.go`（约 L32、L54–L60） |
| 前端 DOM 渲染 renderSnippet | `app/src/config/util/snippets.ts`（约 L7–L42） |
| 片段数据结构 Snippet | `kernel/conf/snippet.go`（约 L31–L38） |

### 12.4 API 接口 + 权限矩阵

| 接口 | 路由位置 | 鉴权中间件链 |
|------|---------|-------------|
| 渲染模板 | `kernel/api/router.go`（约 L368–L370） | CheckAuth + CheckAdminRole + CheckReadonly |
| 获取片段 | `kernel/api/router.go`（约 L472） | CheckAuth |
| 保存/删除片段 | `kernel/api/router.go`（约 L473–L474） | CheckAuth + CheckAdminRole + CheckReadonly |

### 12.5 前端交互与内容插入

| 功能 | 相对路径 + 行号指引 |
|------|-------------------|
| 模板插入渲染 hintRenderTemplate | `app/src/protyle/hint/extend.ts`（约 L542–L564） |
| 内容插入核心 insertHTML | `app/src/protyle/util/insertHTML.ts`（约 L264–L590） |

---

## 13. 总结

### 13.1 核心发现

1. **512 和 1024 是默认值而非硬性上限**：三个 SQL 模板函数在 SQL 无 LIMIT 子句时注入默认限制值（512/1024），但当 SQL 中自带 LIMIT 时，解析器会**尊重用户指定的值**，默认限制形同虚设。这是本次分析最重要的发现。

2. **三个函数的限制机制完全不同**：
   - `queryBlocks`：sqlparser AST 注入 + `selectBlocksRawStmt` Go 计数回退
   - `querySpans`：sqlparser AST 注入，解析失败直接返回空
   - `querySQL`：双解析器三重回退，最终 `queryRawStmt` **无类型检查**

3. **querySQL 不接受 args 参数**：这实际上是更安全的设计——消除了运行时变量通过 `strings.Replace` 注入 SQL 的攻击面。但返回 `map[string]any` 的动态列特性增加了下游风险。

4. **queryRawStmt 是最危险的回退路径**：当两个解析器都失败时，SQL 直接执行且无类型检查，DML 语句可能被执行。`containsLimitClause` 的字符串匹配也可被绕过。

### 13.2 512 vs 1024 对三阶段链路的影响汇总

| 阶段 | 512 限制的影响 | 1024 限制的影响（querySQL） | 用户自定义 LIMIT 的风险 |
|------|--------------|---------------------------|---------------------|
| 变量替换 | 不影响替换过程 | 不影响替换过程 | 不影响替换过程 |
| 上下文注入 | ≤512 次循环，输出量中等 | ≤1024 次循环，输出量翻倍 | ⚠️ 无上限，可能产生 MB 级输出 |
| 内容插入 | DOM 节点数可控 | DOM 节点数翻倍，性能压力增大 | ⚠️ 可能产生数万 DOM 节点 |

### 13.3 优先级排序的建议行动项

| 优先级 | 行动项 | 说明 |
|--------|--------|------|
| P0 | **将 LIMIT 从默认值改为硬性上限** | 在 `SelectBlocksRawStmt` / `SelectSpansRawStmt` / `Query` 中，读取用户 LIMIT 后与硬性上限取 `min` |
| P0 | **queryRawStmt 增加类型检查** | 在回退路径中拒绝非 SELECT 语句，或改为直接返回空 |
| P1 | 真实参数化查询 | 将 `strings.Replace` 改为 `db.Query(stmt, args...)` |
| P1 | 统一 SQL 限制常量 | 定义 `TemplateQueryBlocksLimit` 等常量，消除硬编码 |
| P2 | querySQL 返回字段白名单 | 对 `map[string]any` 的 key 做过滤 |
| P2 | `/getSnippet` 增加审计日志 | 记录 IP、角色、筛选条件 |
| P3 | 模板编译缓存评估 | 基于 LRU 缓存 `*template.Template` |
