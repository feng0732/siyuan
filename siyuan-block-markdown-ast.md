# SiYuan 块级 Markdown 拆分规则与 AST 构建机制深度分析

## 1. 整体架构：解析分层与职责边界

SiYuan 的块级 Markdown 解析体系采用 **三层解析 + 一层编排** 的架构设计，通过明确的职责划分实现解析的稳定性和可扩展性。

### 1.1 四层架构总览

```
┌───────────────────────────────────────────────────────────────────┐
│  Layer 4: 应用编排层 (Kernel Model)                               │
│  [kernel/model/]                                                  │
│  职责: 事务管理、业务校验、数据一致性、多系统同步                  │
│  核心: transaction.go / block.go / file.go / tree.go             │
└──────────────────────────┬────────────────────────────────────────┘
                           │ Block DOM / parse.Tree / ast.Node
┌──────────────────────────▼────────────────────────────────────────┐
│  Layer 3: 数据持久化层 (Filesys / Treenode)                       │
│  [kernel/filesys/] + [kernel/treenode/]                           │
│  职责: .sy JSON 读写、版本升级、ID 修正、XSS 修复、块树索引        │
│  核心: LoadTree / WriteTree / fixTreeJSONData / UpsertBlockTree   │
└──────────────────────────┬────────────────────────────────────────┘
                           │ Block DOM ↔ Tree 转换
┌──────────────────────────▼────────────────────────────────────────┐
│  Layer 2: 结构转换层 (Lute Engine)                                │
│  [kernel/util/lute.go] + [github.com/88250/lute]                 │
│  职责: Markdown ↔ Block DOM ↔ AST Tree 双向转换                  │
│  核心: Md2BlockDOM / BlockDOM2Tree / Md2Tree / Tree2Md           │
└──────────────────────────┬────────────────────────────────────────┘
                           │ 纯文本 Markdown
┌──────────────────────────▼────────────────────────────────────────┐
│  Layer 1: 语法分词层 (Lute Parser)                                │
│  [github.com/88250/lute/parse]                                    │
│  职责: 词法分析、块级拆分、行级解析、IAL 提取                     │
│  核心: Block 识别器 / Inline 解析器 / IAL 解析器                  │
└───────────────────────────────────────────────────────────────────┘
```

### 1.2 核心转换路径

| 路径方向 | 入口函数 | 调用链路 |
|---------|---------|---------|
| **Markdown → Block DOM** | `Md2BlockDOM()` | `L1语法分词` → `L2 MarkdownAST` → `ProtyleRenderer` → `HTML(Block DOM)` |
| **Block DOM → AST Tree** | `BlockDOM2Tree()` | `HTML Parser` → `DOM AST` → `转换规则` → `parse.Tree` |
| **Markdown → AST Tree** | `Md2BlockDOMTree()` | `Md2BlockDOM()` → `BlockDOM2Tree()` |
| **AST Tree → Markdown** | `FormatNodeSync()` | `ast.Node` → `Markdown Renderer` → `纯文本 Markdown` |
| **AST Tree → Block DOM** | `RenderNodeBlockDOM()` | `ast.Node` → `ProtyleRenderer` → `HTML(Block DOM)` |

**代码参考**:
- Lute 引擎配置: [util/lute.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/util/lute.go)
- Markdown 导入入口: [model/file.go#L1018-L1042](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/file.go#L1018-L1042)
- 文档创建核心: [model/file.go#L1744-L1859](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/file.go#L1744-L1859)

---

## 2. 块级 Markdown 拆分规则详解

### 2.1 块拆分的底层机制

#### 2.1.1 块识别触发条件

Lute 解析器按以下优先级识别块边界（从高到低）：

`````
1. 空行分隔（连续 2 个换行符 \n\n）
   └── 绝大多数块的显式边界

2. 行首标记识别（行首 0-3 空格后的特殊字符）
   ├── # + 空格          → 标题块 (NodeHeading)
   ├── ``` / ~~~         → 代码块开始 (NodeCodeBlock)
   ├── > + 空格          → 引用块 (NodeBlockquote)
   ├── -/*/+ + 空格      → 无序列表项 (NodeListItem, subtype=u)
   ├── 数字 + . + 空格   → 有序列表项 (NodeListItem, subtype=o)
   ├── - [ ] + 空格      → 任务列表项 (NodeListItem, subtype=t)
   ├── | 开头            → 表格块 (NodeTable)
   ├── --- / *** / ___   → 分隔线块 (NodeThematicBreak)
   ├── {{{ / }}}         → 超级块开始/结束 (NodeSuperBlock)
   ├── {: ... }          → 属性列表 (NodeKramdownBlockIAL)
   └── <tag              → HTML 块开始 (NodeHTMLBlock)

3. 容器块嵌套规则（递归解析）
   ├── 引用块: 每行 > 前缀递增
   ├── 列表块: 缩进 2-4 空格为子级
   ├── 超级块: {{{ / }}} 匹配对
   └── Callout: > [!NOTE] 变体

4. 兜底规则
   └── 无匹配 → 段落块 (NodeParagraph)
`````

#### 2.1.2 WYSIWYG 模式对拆分的影响

`SetProtyleWYSIWYG(true)`（SiYuan 强制启用）会改变以下拆分行为：

| 选项 | 标准 Markdown | WYSIWYG 模式 |
|------|-------------|-------------|
| 缩进代码块 | 4 空格 = 代码块 | **禁用**（仅围栏代码块） |
| Setext 标题 | `===` / `---` | **禁用**（仅 ATX `#` 标题） |
| YAML Front Matter | `---` 元数据 | **禁用** |
| 链接引用定义 | `[id]: url` | **禁用** |
| 段落首空格 | 忽略 | **保留**（`SetParagraphBeginningSpace(true)`） |
| 自动空格 | 中英文间插空格 | **禁用** |

**代码参考**: [util/lute.go#L50-L88](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/util/lute.go#L50-L88)

---

### 2.2 各块类型拆分与 AST 转换规则

#### 2.2.1 标题块 (Heading → `ast.NodeHeading`)

**Markdown 语法**:
```markdown
# 一级标题
## 二级标题 {#custom-id .custom-class name="别名"}
### 三级标题 ^blockref-id
```

**拆分规则**:
1. 行首 0-3 空格后匹配 `#{1,6}\s` 正则
2. 提取 `#` 数量 → `HeadingLevel` (1-6)
3. 移除行尾 IAL（属性列表），单独解析为 `NodeKramdownBlockIAL`
4. 剩余内容作为行级元素递归解析（`heading → inline nodes`）

**AST 节点结构**:
```go
&ast.Node{
    Type:         ast.NodeHeading,
    ID:           "20250101120000-abc123",          // SiYuan 块 ID
    HeadingLevel: 1,                               // 1-6
    HeadingMarker: "#",                            // 标记字符
    Tokens:       []byte("一级标题"),              // 原始文本
    KramdownIAL:  [][]string{                      // IAL 属性
        {"id", "20250101120000-abc123"},
        {"updated", "20250101120000"},
        {"custom-id", "自定义 ID"},
    },
    Children:     []*ast.Node{                     // 行级子节点
        {Type: ast.NodeText, Tokens: []byte("一级标题")},
    },
}
```

**应用层处理**:
- 创建空文档时自动补空段落 → [treenode/tree.go#L77-L80](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/tree.go#L77-L80)
- 折叠标题下方子块移动 → `MoveFoldHeading()`
- 标题层级变化触发展开/插入逻辑 → [model/transaction.go#L1514-L1556](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L1514-L1556)

---

#### 2.2.2 段落块 (Paragraph → `ast.NodeParagraph`)

**Markdown 语法**:
```markdown
这是普通段落文本，支持**加粗**、*斜体*、[链接](url)等行级语法。
{: #paragraph-id name="段落别名" style="color:red"}
```

**拆分规则**:
1. 不能被识别为其他块类型的文本行 → 段落
2. 连续非空行合并为同一段落（空行终止）
3. 行尾 IAL 单独提取
4. 内容进入行级解析管道（Inline Parser）

**AST 节点结构**:
```go
&ast.Node{
    Type:     ast.NodeParagraph,
    ID:       "20250101120000-def456",
    Tokens:   []byte("这是普通段落文本..."),
    Children: []*ast.Node{
        {Type: ast.NodeText, Tokens: []byte("这是普通段落文本，支持")},
        {Type: ast.NodeTextMark, TextMarkType: "strong"},  // **加粗**
        {Type: ast.NodeTextMark, TextMarkType: "em"},      // *斜体*
        {Type: ast.NodeLinkText, ...},                     // [链接]
        {Type: ast.NodeLinkDest, ...},                     // (url)
    },
}
```

**应用层特殊处理** ([model/file.go#L1823-L1853](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/file.go#L1823-L1853)):
- 段落中仅包含 `.mp3` 链接 → 自动转换为 `NodeAudio` 块
- 段落中仅包含 `.mp4` 链接 → 自动转换为 `NodeVideo` 块
- 空文档自动补空段落 → `NewParagraph("")`

**段落创建辅助函数**:
```go
func NewParagraph(id string) (ret *ast.Node) {
    newID := id
    if "" == newID {
        newID = ast.NewNodeID()  // 生成 YYYYMMDDHHmmss-xxxxxx 格式 ID
    }
    ret = &ast.Node{ID: newID, Type: ast.NodeParagraph}
    ret.SetIALAttr("id", newID)
    ret.SetIALAttr("updated", newID[:14])  // ID 前 14 位 = 时间戳
    return
}
```

**代码参考**: [treenode/tree.go#L116-L125](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/tree.go#L116-L125)

---

#### 2.2.3 列表块 (List / ListItem)

**Markdown 语法**:
```markdown
- 无序列表项 1
- 无序列表项 2
  - 嵌套列表项
  - [ ] 任务项未完成
  - [x] 任务项已完成

1. 有序列表项 1
2. 有序列表项 2
```

**拆分规则**:
1. 行首匹配列表标记（`-/*/+` 或 `数字.`）后接 1 空格
2. 连续缩进相同的列表项合并为同一个 `NodeList`
3. 缩进增加 → 创建子级 `NodeList`（嵌套）
4. 方括号 `[ ]` / `[x]` → `ListData.Typ = 3` (任务列表)

**AST 双层结构**:
```
NodeList (l, subtype=u/o/t)
├── ListData.Typ: 0=无序列表, 1=有序列表, 3=任务列表
├── ListData.OrderedListStart: 起始序号
└── NodeListItem (i) × N
    ├── ListData.Typ: 继承自父级
    ├── ListData.Task: 0=未完成, 1=已完成
    └── 内容节点 (Paragraph / CodeBlock / 嵌套 List ...)
```

**子类型缩写**: [treenode/node.go#L416-L427](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/node.go#L416-L427)
- `u` = Unordered (无序)
- `o` = Ordered (有序)
- `t` = Task (任务)

**应用层容器约束** ([model/transaction.go#L687-L703](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L687-L703)):

插入块到列表时的强制规则：
```go
if ast.NodeList == node.Type {
    // 列表下只能挂列表项！
    if ast.NodeList == toInsert.Type {
        // 插入的是列表 → 提取所有 ListItem 逐个追加
        for childLi := toInsert.FirstChild; nil != childLi; childLi = childLi.Next {
            node.AppendChild(childLi)
        }
    } else {
        // 插入的是非列表 → 自动包装一个新 ListItem
        newLi := &ast.Node{
            ID: ast.NewNodeID(),
            Type: ast.NodeListItem,
            ListData: &ast.ListData{Typ: node.ListData.Typ},  // 继承类型
        }
        node.AppendChild(newLi)
        newLi.AppendChild(toInsert)  // 内容放入 ListItem
    }
}
```

**删除列表项时的清理**:
```go
if srcNode.Parent.FirstChild == srcNode.Parent.LastChild {
    // 列表中唯一的列表项被移除 → 删除整个空列表
    srcEmptyList = srcNode.Parent
}
// ...
if nil != srcEmptyList {
    srcEmptyList.Unlink()
}
```

**代码参考**: [model/transaction.go#L869-L915](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L869-L915)

---

#### 2.2.4 代码块 (CodeBlock → `ast.NodeCodeBlock`)

**Markdown 语法**:

使用 ``` + 语言标识开启围栏，内容原样保留，``` 关闭围栏（围栏反引号数量 ≥ 3 且内外数量匹配）：

> 示例如下（使用缩进代码块展示，避免嵌套围栏冲突）：
>
>     ```go
>     package main
>
>     func main() {
>         fmt.Println("Hello SiYuan")
>     }
>     ```

围栏后可附加 IAL 属性，如 `{#code-block-id .custom-class}`。

**拆分规则**:
1. 围栏代码块（仅在 WYSIWYG 模式下支持）
2. 行首 0-3 空格 + 连续 3 个及以上反引号或波浪号
3. 开始围栏后可指定语言标识（`go` / `python` 等）
4. 结束围栏必须与开始围栏字符相同，数量 ≥ 开始围栏
5. 围栏后 IAL 提取为属性
6. 围栏间内容 **原样保留**，不进行行级解析

**AST 节点结构**（外层使用 5 反引号围栏，避免内层 ``` 冲突）:

`````go
&ast.Node{
    Type:              ast.NodeCodeBlock,
    ID:                "20250101120000-code12",
    CodeBlockMarker:   []byte("```"),       // 开始围栏标记（原始字符）
    CodeBlockOpenMarker: []byte("```"),     // 结束围栏标记（原始字符）
    CodeBlockInfo:     []byte("go"),        // 语言标识
    IsFencedCodeBlock: true,                // 是否围栏代码块
    Tokens:            []byte("package main\n\nfunc main() {\n    fmt.Println(\"Hello SiYuan\")\n}"),
    KramdownIAL:       [][]string{
        {"id", "20250101120000-code12"},
        {"custom-class", "custom-class"},
    },
}
`````

**注意**: 缩进代码块（4 空格缩进）在 WYSIWYG 模式下被 `SetIndentCodeBlock(false)` 禁用。

---

#### 2.2.5 属性列表 (IAL → `ast.NodeKramdownBlockIAL`)

**Markdown 语法**:
```markdown
## 标题块
{: #heading-id name="标题别名" icon="📝" tags="tag1,tag2" custom-key="自定义值"}

段落内容行。
{: #para-id updated="20250101120000"}
```

**拆分规则**:
1. 块后紧邻的 `{: ... }` 行（可前接 0-3 空格）
2. 键值对格式: `key=value`、`#id`（简写 `id=xxx`）、`.class`（简写 `class=xxx`）
3. 无引号值、单引号值、双引号值均支持
4. 所有键值对解析为 `[][]string` 二维数组
5. **不独立存在** → IAL 属性被合并到前一个块的 `KramdownIAL` 字段

**应用层 IAL 处理**:

IAL 在系统中承担元数据存储的核心角色，常用属性:

| 属性键 | 用途 | 示例值 |
|-------|------|-------|
| `id` | 块唯一标识（必填） | `20250101120000-abc123` |
| `title` | 文档标题（仅根节点） | `我的笔记` |
| `updated` | 更新时间戳（秒） | `20250101120000` |
| `name` | 块命名/别名 | `重要段落` |
| `alias` | 块别名（用于引用锚点） | `定义-1` |
| `memo` | 块备注信息 | `待补充细节` |
| `bookmark` | 书签描述 | `**第3章**重点` |
| `icon` | 块图标 | `📝` |
| `tags` | 标签列表（逗号分隔） | `工作,优先级高` |
| `fold` | 是否折叠（仅标题/超级块） | `1` |
| `heading-fold` | 折叠标题子块标记 | `1` |
| `custom-*` | 用户自定义属性前缀 | `custom-复习次数=3` |
| `custom-avs` | 属性视图关联数据 | JSON 字符串 |
| `custom-hidden` | 文档隐藏标记 | `true` |

**IAL ↔ Map 转换**:
```go
// IAL 二维数组 → Map（便于属性查询）
ialMap := parse.IAL2Map(node.KramdownIAL)
val := ialMap["custom-key"]

// 属性写入（自动去重/更新）
node.SetIALAttr("updated", util.CurrentTimeSecondsStr())

// 缓存热点 IAL（避免频繁遍历 AST）
cache.PutBlockIAL(node.ID, parse.IAL2Map(node.KramdownIAL))
```

**XSS 防护修复** (GHSA-ff66-236v-p4fg):
```go
func escapeNodeAttributeValues(node *ast.Node) (escaped bool) {
    for _, kv := range node.KramdownIAL {
        // 解码 → 重新编码 → 比对
        // 不一致 = 未正确转义 / 恶意拼接
        canonical := html.EscapeAttrVal(html.UnescapeAttrVal(kv[1]))
        if canonical != kv[1] {
            kv[1] = canonical
            escaped = true
        }
    }
    return
}
```

**代码参考**: [filesys/tree.go#L494-L507](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/filesys/tree.go#L494-L507)

---

#### 2.2.6 超级块 (SuperBlock → `ast.NodeSuperBlock`)

**Markdown 语法**:
```markdown
{{{row
  :PROPERTIES:
  :layout: col2
  :END:

  ## 左栏内容

---

  ## 右栏内容
}}}
{: #super-id layout="col2"}
```

**拆分规则**:
1. 行首 `{{{` 标记超级块开始（可后接布局标识）
2. 内部 `---` 分隔符划分子块区域
3. 行首 `}}}` 标记超级块结束
4. 结束后 IAL 合并到超级块节点
5. 内部块按标准块规则递归解析

**应用层插入规则** ([model/transaction.go#L704-L710](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L704-L710)):
```go
if ast.NodeSuperBlock == node.Type {
    // 超级块插入需跳过布局标记节点
    layout := node.ChildByType(ast.NodeSuperBlockLayoutMarker)
    if nil != layout {
        // Prepend: 布局标记后插入
        layout.InsertAfter(toInsert)
    } else {
        // Append: 最后一个子节点前插入
        node.LastChild.InsertBefore(toInsert)
    }
}
```

---

#### 2.2.7 其他块类型速查

| 块类型 | 缩写 | 触发语法 | AST 关键字段 |
|-------|------|---------|-------------|
| 引用块 (Blockquote) | `b` | 行首 `> ` | 纯容器，子块递归 |
| Callout | `callout` | `> [!NOTE]` | `CalloutType: "NOTE/TIP/IMPORTANT/CAUTION/WARNING"` |
| 表格 (Table) | `t` | 行首 `\|` | `TableAligns[]`, `NodeTableHead/Body/Row/Cell` |
| 数学公式块 (MathBlock) | `m` | `$$ ... $$` | `MathBlockScript: TeX 内容` |
| 嵌入查询块 | `query_embed` | 代码块语言 `query` | `NodeBlockQueryEmbedScript: SQL 语句` |
| 属性视图 (AttributeView) | `av` | `av` 代码块 | `AttributeViewID`, `AttributeViewType` |
| HTML 块 | `html` | HTML 标签 | `Tokens: 原始 HTML` |
| 分隔线 | `tb` | `---` / `***` / `___` | 无子节点 |
| 视频块 | `video` | `.mp4` 链接段落 | `Tokens: <video> HTML` |
| 音频块 | `audio` | `.mp3` 链接段落 | `Tokens: <audio> HTML` |
| 文档根 (Document) | `d` | 整个 `.sy` 文件 | `Spec: "2"`, `Box`, `Path`, `HPath` |

**完整类型映射表**: [treenode/node.go#L370-L398](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/node.go#L370-L398)

---

### 2.3 行级元素解析 (Inline Elements)

行级元素在块内容内部进行二次解析（Paragraph / Heading / TableCell 等）：

| 行级类型 | `NodeTextMark` 子类型 | 语法示例 |
|---------|---------------------|---------|
| 加粗 | `strong` | `**text**` / `__text__` |
| 斜体 | `em` | `*text*` / `_text_` |
| 删除线 | `strikethrough` | `~~text~~` |
| 标记 | `mark` | `==text==` |
| 行级代码 | `code` | 用单个反引号包裹 code |
| 行级公式 | `inline-math` | `$E=mc^2$` |
| 上标 | `sup` | `^text^` |
| 下标 | `sub` | `~text~` |
| 标签 | `tag` | `#tag#` |
| 块引用 | `block-ref` | `((block-id))` |
| 文件标注引用 | `file-annotation-ref` | `((file.pdf$page=1))` |
| 超链接 | `a` | `[text](url)` |
| 图片 | `img` | `![alt](url)` |
| 行级 HTML | `inline-html` | `<span>text</span>` |

**行级结构扁平化**:

Spec 版本升级 ""→1 时执行 `NestedInlines2FlattedSpans()`，将嵌套的行级节点转换为扁平的 `TextMark` 跨度序列，简化 WYSIWYG 编辑器处理。

---

## 3. 解析引擎与应用层职责划分

### 3.1 Lute 解析引擎 (L1-L2) 的纯职责

| 维度 | Lute 引擎职责 | 明确不做的事 |
|-----|-------------|------------|
| **结构化** | Markdown 语法规则识别、块/行级拆分、AST 构建、IAL 提取 | 不生成块 ID、不设置业务属性 |
| **校验** | 语法层面校验（围栏闭合、列表完整性）、HTML 白名单过滤 | 不校验 ID 格式、不校验业务约束 |
| **转换** | Markdown ↔ Block DOM ↔ Tree 格式双向转换 | 不做类型升级（段落→音视频块） |
| **渲染** | 纯文本输出、HTML 渲染、JSON 序列化 | 不处理索引、不广播事件 |
| **安全** | `SetSanitize(true)` HTML 恶意脚本过滤 | 不处理属性值二次转义 |

---

### 3.2 Kernel 应用层 (L3-L4) 的扩展职责

#### 3.2.1 结构化增强 (Structural Enhancement)

**ID 注入与补全**:
```go
// 插入块时检查并补全缺失的 ID
if "" == insertedNode.ID {
    insertedNode.ID = ast.NewNodeID()  // YYYYMMDDHHmmss-random
    insertedNode.SetIALAttr("id", insertedNode.ID)
}
```

**容器约束强制**（参见列表/超级块规则）:
- `NodeList` 下只能有 `NodeListItem`，自动包装
- `NodeHeading` 下方块使用逻辑父子（HeadingChildren），非 AST 直接父子
- `NodeSuperBlock` 布局标记节点的插入位置特殊处理

**特殊类型转换**:
```go
// 段落 → 音频块 / 视频块
if ast.NodeParagraph == n.Type {
    link := n.FirstChild
    if nil != link && link.IsTextMarkType("a") {
        if strings.HasSuffix(link.TextMarkAHref, ".mp3") {
            audio := &ast.Node{Type: ast.NodeAudio, ID: n.ID, ...}
            n.InsertBefore(audio)
            n.Unlink()
        }
    }
}
```

**代码参考**: [model/file.go#L1823-L1853](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/file.go#L1823-L1853)

---

#### 3.2.2 业务校验 (Business Validation)

**文档级校验** ([model/file.go#L1744-L1804](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/file.go#L1744-L1804)):

| 校验项 | 规则 | 失败处理 |
|-------|------|---------|
| 标题长度 | ≤ 512 个 Unicode 字符 | 返回错误语言包 (106) |
| ID 合法性 | 文件名必须包含合法 ID | 返回语言 (16) |
| 隐藏文件 | 文件名不能以 `.` 开头 | 返回语言 (13) |
| 笔记本存在 | BoxID 必须已注册 | 返回语言 (0) |
| 路径深度 | `/` 分隔 ≤ 7 层 (可配置) | 返回语言 (118) |
| 文件存在 | 目标 `.sy` 不能已存在 | 返回语言 (1) |
| 标题规范化 | 移除 `/`、非法字符、ZWJ 保护 | 自动修正 |

**事务级校验** ([model/transaction.go#doUpdate](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L1410-L1594)):

| 校验项 | 规则 | 失败处理 |
|-------|------|---------|
| 目标树加载 | `tx.loadTree(id)` 必须成功 | `TxErrCodeBlockNotFound` |
| 节点存在性 | `GetNodeInTree(tree, id)` 非 nil | `TxErrCodeBlockNotFound` |
| 更新数据非空 | `operation.Data` 去除光标标记后非空 | `TxErrCodeBlockNotFound` |
| 子树有效性 | `BlockDOM2Tree` 返回的根至少有一子节点 | `TxErrCodeBlockNotFound` |

---

#### 3.2.3 编辑流程编排 (Editing Orchestration)

**事务执行矩阵**（`performTx` 中的 25+ 种操作类型）:

| 操作类别 | 操作类型 | 核心处理函数 |
|---------|---------|-------------|
| **基础 CRUD** | `create` / `update` / `delete` | `doCreate` / `doUpdate` / `doDelete` |
| **位置插入** | `insert` / `appendInsert` / `prependInsert` | `doInsert` / `doAppendInsert` / `doPrependInsert` |
| **位置移动** | `move` / `moveOutlineHeading` / `append` | `doMove` / `doMoveOutlineHeading` / `doAppend` |
| **折叠控制** | `foldHeading` / `unfoldHeading` | `doFoldHeading` / `doUnfoldHeading` |
| **属性修改** | `setAttrs` / `updateAttrs` / `doUpdateUpdated` | `doSetAttrs` / 属性更新 API |
| **AV 操作** | `addAttrViewCol` / `updateAttrViewCell` 等 30+ 种 | AV 子系统专门处理 |
| **闪卡操作** | `addFlashcards` / `removeFlashcards` | 闪卡子系统处理 |

**事务状态机**:
```
tx.begin()           → 初始化 Lute、准备 nodes/trees 缓存
     │
processLargeInsert() → 大块插入优化（跳过逐节点处理）
processLargeDelete() → 批量删除优化
     │
逐个执行 DoOperations
  for op in tx.DoOperations:
      switch op.Action: do*() handlers
     │
tx.commit()          → 写 .sy 文件 / 写 SQLite / 写 FTS 索引 / 推送 WS
tx.state = 完成
```

**代码参考**: [model/transaction.go#L148-L247](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L148-L247)

---

#### 3.2.4 存储与同步 (Storage & Sync)

**三层写入管道**（事务提交阶段）:

```
tx.trees（变更中的树缓存）
    │
    ▼  indexWriteTreeUpsertQueue()
┌─────────────────────────────────────────────┐
│ 1. filesys.WriteTree()                      │
│    ├─ tree → JSON（JSONRenderer）          │
│    ├─ 可选：json.Indent 格式化              │
│    ├─ filelock.WriteFile（原子写）          │
│    └─ 写入 data/{box}/{path}.sy            │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│ 2. treenode.UpsertBlockTree()               │
│    └─ SQLite → blocktrees 表（ID → 路径映射）│
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│ 3. sql.UpsertTreeQueue()                    │
│    ├─ blocks 表（块内容、类型、属性）        │
│    ├─ block_refs 表（引用关系）             │
│    └─ blocks_fts 表（FTS5 全文索引）        │
└─────────────────────────────────────────────┘
```

**同步标记**:
- 每次写入后调用 `IncSync()` 递增同步计数器
- 云同步模块监听计数器变化，增量推送变更

---

## 4. 解析异常检查顺序与处理流程

### 4.1 导入/加载路径的异常检查链

#### 4.1.1 Markdown 导入检查顺序 (CreateDocByMd)

```
Step 1: 环境校验
  ├─ createDocLock 互斥锁（串行化文档创建）
  ├─ BoxID 存在性检查
  └─ └─ 不存在 → 返回 Conf.Language(0)

Step 2: Lute 解析层（内部）
  ├─ Md2BlockDOM(md) → Block DOM
  ├─ 语法错误 → Lute 内部容错（通常不报错，降级为段落）
  └─ 围栏不闭合 → 合并到下一个块 / 直到文档结束

Step 3: 标题规范化
  ├─ normalizeDocTitle(title)
  │   ├─ 移除 '/' 字符
  │   ├─ 非法字符过滤（RemoveInvalid）
  │   ├─ ZWJ 保护（避免 Emoji 变形）
  │   └─ TrimSpace
  ├─ 长度 ≤ 512 Unicode 字符
  │   └─ 超长 → Conf.Language(106)
  └─ 空标题 → 替换为 Conf.Language(16) "未命名文档"

Step 4: 路径合法性
  ├─ 文件名提取合法 ID（GetTreeID）
  ├─ 不以 '.' 开头（隐藏文件）
  ├─ 路径深度 ≤ 7 层（可配置 AllowCreateDeeper）
  ├─ 父级文档存在（否则 ErrBlockNotFound）
  └─ 目标 .sy 文件不存在（否则 Conf.Language(1) 已存在）

Step 5: Block DOM → Tree 转换
  ├─ BlockDOM2Tree(dom) → parse.Tree
  ├─ Root 设置 ID / Box / Path / HPath / Spec
  ├─ 空文档补空段落（NewParagraph）
  └─ 特殊节点转换（MP3→Audio / MP4→Video）

Step 6: 事务提交
  ├─ doCreate(tree) → 通过
  ├─ FlushTxQueue() → 强制同步执行
  └─ 建立排序索引
```

**代码参考**: [model/file.go#L1018-L1042](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/file.go#L1018-L1042)

---

#### 4.1.2 .sy 文件加载检查顺序 (LoadTree)

```
Step 1: 物理读取
  ├─ filelock.ReadFile 读取 .sy 文件
  └─ IO 错误 → 记录日志 + 返回 err（上层降级处理）

Step 2: 二进制清理
  └─ removeUnescapedUnicodeNull() → 移除未转义 \0 避免 JSON 解析失败

Step 3: JSON 解析（含自动修复）
  ├─ dataparser.ParseJSON()
  ├─ 缺失字段 → 补默认值（needFix = true）
  ├─ 类型错误 → 自动转换（字符串→数字等）
  ├─ 语法错误 → 尽量修复 + needFix = true
  └─ 不可修复 → 返回 err，终止加载

Step 4: 版本兼容性
  ├─ CheckSpec(tree) → 比较 tree.Root.Spec 与 CurrentSpec("2")
  ├─ Spec > Current → ErrSpecTooNew（数据来自未来版本，拒绝加载）
  ├─ Spec 非法（非数字）→ 记录日志 + 跳过升级
  └─ Spec < Current → 标记 needFix = true，进入升级流程

Step 5: 版本升级（向前兼容）
  ├─ UpgradeSpec("" → 1 → 2)
  │   ├─ Spec "" → 1: NestedInlines2FlattedSpans（行级扁平化）
  │   └─ Spec 1  → 2: 增加 Callout 块类型支持
  └─ 升级后 tree.Root.Spec = CurrentSpec

Step 6: 安全修复（XSS）
  ├─ escapeAttributeValues() → 全树遍历
  │   ├─ 解码属性值 html.UnescapeAttrVal()
  │   ├─ 重新编码 html.EscapeAttrVal()
  │   └─ 前后不同 → 修正 + hasEscaped = true
  └─ 适用版本: v3.5.2+（修复 #16686 XSS 漏洞）

Step 7: ID 一致性
  ├─ 从路径提取 ID（文件名去 .sy）
  ├─ 与 tree.Root.ID 比对
  └─ 不一致 → 强制重置为路径 ID（重命名后常见情况）

Step 8: 修复持久化（needFix = true 时）
  ├─ JSONRenderer 重新序列化
  ├─ 可选 json.Indent 格式化（UseSingleLineSave 控制）
  ├─ os.MkdirAll 确保目录存在
  └─ filelock.WriteFile 原子写回
```

**代码参考**:
- 加载入口: [filesys/tree.go#L398-L456](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/filesys/tree.go#L398-L456)
- XSS 修复: [filesys/tree.go#L474-L508](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/filesys/tree.go#L474-L508)
- 版本升级: [treenode/tree.go#L158-L192](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/tree.go#L158-L192)

---

### 4.2 编辑/更新路径的异常检查链

#### 4.2.1 块更新异常顺序 (doUpdate)

```
输入: operation { id, data (Block DOM 字符串) }

Step 1: 目标定位
  ├─ tx.loadTree(id) → 加载所在树
  └─ 失败 → TxErrCodeBlockNotFound + 日志

Step 2: 输入清洗
  ├─ 移除前端光标标记: strings.Replace(data, FrontEndCaret, "")
  └─ 空数据 → TxErrCodeBlockNotFound

Step 3: DOM → AST 转换
  ├─ luteEngine.BlockDOM2Tree(data) → 子树
  │   ├─ DOM 解析失败 → 返回空树（后续捕获）
  │   └─ 非法标签 → 按段落 / 文本降级
  └─ 继承原树 ID / Box / Path（子树根信息）

Step 4: 旧节点验证
  ├─ GetNodeInTree(tree, id) → 找到被替换的节点
  └─ 不存在 → TxErrCodeBlockNotFound + msg

Step 5: 引用关系收集
  ├─ ast.Walk 全遍历新子树
  │   ├─ 空白 inline-math → 标记删除（n.Unlink()）
  │   ├─ block-ref → sql.CacheRef() 缓存 + 收集 defID
  │   └─ 文档标题引用 → DynamicRefTexts 缓存覆盖锚文本
  └─ 引用变更 → 异步延迟刷新计数（SQLFlushInterval）

Step 6: 子节点有效性
  ├─ subTree.Root.FirstChild 必须存在
  │   └─ 不存在 → TxErrCodeBlockNotFound
  ├─ 列表特殊处理（List 父下更新 ListItem）
  │   └─ ast.NodeList == updatedNode.Type → 取 FirstChild（跳过包装 List）
  └─ 容器块折叠迁移
      └─ oldNode.IsContainerBlock() → MoveFoldHeading(new, old)

Step 7: 类型特定处理
  ├─ HTML 块 → 剔除连续空行（含纯空格行）
  ├─ AV 节点 → 解析视图 + 设置 ViewType/ViewID
  └─ 删除节点同步 → syncDelete2AvBlock()

Step 8: 折叠标题层级
  ├─ needUnfoldParentHeading（标题降级）
  │   └─ 展开父折叠标题 + 推送 unfold WS 消息
  └─ needInsertAfterParentHeading（标题升级）
      └─ 主动推送 insert WS 消息插入到原折叠标题后

Step 9: 原子替换 + 元数据
  ├─ oldNode.InsertAfter(updatedNode)
  ├─ oldNode.Unlink()
  ├─ CreatedUpdated(updatedNode) → 递归更新所有父节点 updated
  ├─ cache.PutBlockIAL() → 更新属性缓存
  ├─ tx.nodes[] / tx.trees[] → 事务变更缓存
  └─ 提交时统一写入（管道 1-2-3）
```

**代码参考**: [model/transaction.go#L1410-L1594](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L1410-L1594)

---

### 4.3 事务执行级别的异常分类处理

**位置**: [model/transaction.go#L83-L125](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L83-L125)

```go
func flushTx(tx *Transaction) {
    defer logging.Recover()                    // 第 0 层：Panic 捕获
    flushLock.Lock()                           // 第 0 层：全局互斥（串行保证）
    isFlushing = true
    defer flushLock.Unlock()

    start := time.Now()
    if txErr := performTx(tx); nil != txErr {
        switch txErr.code {
        case TxErrCodeBlockNotFound (0):
            // 可恢复错误：推送消息 + 继续服务
            → util.PushTxErr(msg, 0, nil)
            return

        case TxErrCodeDataIsSyncing (1):
            // 临时状态：用户稍后重试
            → util.PushMsg(Conf.Language(222), 5000)
            // 不返回，不崩溃

        case TxErrHandleAttributeView (3):
            // 子系统错误：日志 + 提示，不影响主流程
            → util.PushMsg(Conf.Language(258), 5000)
            logging.LogErrorf(...)
            // 不返回，不崩溃

        case TxErrCodeWriteTree (2):
            // 致命错误：数据一致性无法保证
            → logging.LogFatalf(ExitCodeFatal, ...)
            // os.Exit(1) 终止进程！

        default / TxErrCodePushMsg (4):
            // 同 0，可恢复
            → util.PushTxErr(...)
            return
        }
    }

    // 性能监控
    elapsed := time.Since(start).Milliseconds()
    if 2000 < elapsed {
        logging.LogWarnf("op tx [%dms] > 2000ms")  // >2秒告警
    }
}
```

**Panic 恢复（内层）** ([model/transaction.go#L168-L178](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go#L168-L178)):
```go
defer func() {
    if e := recover(); nil != e {
        logging.LogErrorf("PANIC RECOVERED: %v\n\t%s", e, logging.ShortStack())
        if 1 == tx.state.Load() {
            tx.rollback()  // 事务状态=执行中 → 尝试回滚
            return
        }
    }
}()
```

---

### 4.4 索引损坏的降级恢复链

当块 ID 在数据库索引中找不到（索引与文件不一致）时的自动修复：

```
GetBlockTree(id) → nil（索引缺失）
     │
     ▼
ErrBlockNotFound 触发
     │
     ▼
indexTreeInFilesystem() 从文件系统恢复
     ├─ 1. 全局搜索：在 data/ 所有 .sy 文件中 grep 块 ID
     ├─ 2. 找到匹配 → filesys.LoadTree(box, path) 加载完整树
     ├─ 3. treenode.UpsertBlockTree(tree) → 重建 blocktrees 索引
     └─ 4. sql.IndexTreeQueue(tree) → 重建 blocks / block_refs / FTS 索引
          │
          ▼ 仍失败
     用户触发：设置 → 搜索 → 重建索引
          │
          ▼
     /api/filetree/reindexTree API
     → 全量遍历 data/ → 逐文件重建所有索引
```

---

## 5. 附录：关键数据结构与转换表

### 5.1 Block DOM → AST Node 映射表

前端编辑后产生的 Block DOM（HTML）通过 `BlockDOM2Tree()` 转换为 AST 节点：

| data-type | → ast.Node.Type | 缩写 |
|-----------|----------------|------|
| `NodeDocument` | `ast.NodeDocument` | `d` |
| `NodeHeading` | `ast.NodeHeading` | `h` |
| `NodeParagraph` | `ast.NodeParagraph` | `p` |
| `NodeList` | `ast.NodeList` | `l` |
| `NodeListItem` | `ast.NodeListItem` | `i` |
| `NodeCodeBlock` | `ast.NodeCodeBlock` | `c` |
| `NodeMathBlock` | `ast.NodeMathBlock` | `m` |
| `NodeTable` | `ast.NodeTable` | `t` |
| `NodeBlockquote` | `ast.NodeBlockquote` | `b` |
| `NodeSuperBlock` | `ast.NodeSuperBlock` | `s` |
| `NodeCallout` | `ast.NodeCallout` | `callout` |
| `NodeBlockQueryEmbed` | `ast.NodeBlockQueryEmbed` | `query_embed` |
| `NodeAttributeView` | `ast.NodeAttributeView` | `av` |
| `NodeHTMLBlock` | `ast.NodeHTMLBlock` | `html` |
| `NodeThematicBreak` | `ast.NodeThematicBreak` | `tb` |
| `NodeVideo` | `ast.NodeVideo` | `video` |
| `NodeAudio` | `ast.NodeAudio` | `audio` |
| `NodeIFrame` | `ast.NodeIFrame` | `iframe` |
| `NodeWidget` | `ast.NodeWidget` | `widget` |
| `NodeText` | `ast.NodeText` | `text`（行级） |
| `NodeTextMark` | `ast.NodeTextMark` | `textmark`（行级） |

**代码参考**: [treenode/node.go#L370-L414](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/node.go#L370-L414)

### 5.2 双 Lute 引擎对比

| 配置项 | 编辑引擎 NewLute() | 导入引擎 NewStdLute() |
|-------|-------------------|---------------------|
| `SetProtyleWYSIWYG` | **true** | false |
| `SetSanitize` | **true** | false |
| `SetIndentCodeBlock` | **false**（禁用 4 空格） | **true**（支持缩进代码块） |
| `SetGFMAutoLink` | - | **false**（导入时不自动链接） |
| `SetCodeSyntaxHighlight` | false | false |
| `SetCallout` | **true** | - |
| `SetDataTask` | **true** | - |
| `SetKramdownIAL` | **true** | - |
| `SetBlockRef` | **true** | - |
| `SetSuperBlock` | **true** | - |

**代码参考**: [util/lute.go#L50-L115](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/util/lute.go#L50-L115)

### 5.3 关键代码路径速查

| 功能 | 文件 | 行号 |
|-----|------|-----|
| Lute 初始化 | [util/lute.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/util/lute.go) | L50-L115 |
| Markdown → 文档 | [model/file.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/file.go) | L1018-L1042 |
| 文档创建核心 | [model/file.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/file.go) | L1744-L1859 |
| 块更新 doUpdate | [model/transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go) | L1410-L1594 |
| 列表插入约束 | [model/transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go) | L687-L703 |
| 超级块插入规则 | [model/transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go) | L704-L710 |
| 事务执行错误分类 | [model/transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/model/transaction.go) | L83-L125 |
| 类型缩写映射 | [treenode/node.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/node.go) | L370-L414 |
| 空段落创建 | [treenode/tree.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/tree.go) | L116-L125 |
| 文件加载+修复 | [filesys/tree.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/filesys/tree.go) | L398-L456 |
| XSS 属性修复 | [filesys/tree.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/filesys/tree.go) | L474-L508 |
| 版本兼容性 | [treenode/tree.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/treenode/tree.go) | L139-L192 |
| 前端事务队列 | [transaction.ts](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/app/src/protyle/wysiwyg/transaction.ts) | L64-L275 |
| HTML→BlockDOM API | [api/lute.go](file:///d:/fz/0601/solo-dogfeeding/code/283-siyuan/kernel/api/lute.go) | L78-L202 |

---

## 6. 总结

SiYuan 的块级 Markdown 解析体系设计遵循以下核心原则：

1. **解析引擎纯粹性**：Lute 专注语法规则与格式转换，不掺杂业务逻辑
2. **应用层兜底**：Kernel 在 Lute 输出基础上进行 ID 补全、容器约束、类型升级、属性注入
3. **多层校验纵深**：语法校验→业务校验→事务校验→数据修复→索引恢复，每层有降级策略
4. **数据自我修复**：加载时 9 步检查流程自动修正 ID、转义、版本、格式等不一致问题
5. **错误分级处理**：可恢复错误推送消息，致命错误终止进程避免数据进一步损坏
6. **最终一致性优先**：通过串行事务队列、原子文件写入、延迟索引刷新保证数据最终一致
