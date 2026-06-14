# SiYuan 块数据库视图与字段类型代码分析

> 本文档基于 SiYuan 源码仓库结构，从代码组织角度分析块数据库（Attribute View，简称 AV）的字段定义、视图配置、数据读取及界面展示间的状态传递机制。所有引用均使用仓库相对路径，便于在任意环境下复查。

---

## 目录

- [1. 代码结构与职责划分](#1-代码结构与职责划分)
- [2. 字段定义与类型扩展](#2-字段定义与类型扩展)
- [3. 视图配置与数据流向](#3-视图配置与数据流向)
- [4. 排序与过滤机制](#4-排序与过滤机制)
- [5. 持久化边界](#5-持久化边界)
- [6. 界面状态传递](#6-界面状态传递)
- [7. 类型切换异常边界](#7-类型切换异常边界)
- [8. 渲染结果异常边界](#8-渲染结果异常边界)
- [9. 后续调查清单](#9-后续调查清单)

---

## 1. 代码结构与职责划分

### 1.1 三层架构

SiYuan 数据库视图采用典型的前后端分离三层架构：

```
┌─────────────────────────────────────────────────────┐
│  前端渲染层  (app/src/protyle/render/av/)            │
│  负责 DOM 渲染、用户交互、状态暂存                    │
├─────────────────────────────────────────────────────┤
│  后端核心层  (kernel/av/)                            │
│  负责数据结构、过滤/排序/分组、视图实例构建            │
├─────────────────────────────────────────────────────┤
│  后端业务层  (kernel/model/attribute_view*.go)       │
│  负责事务处理、持久化、跨库联动                        │
└─────────────────────────────────────────────────────┘
```

### 1.2 前端模块清单

路径：`app/src/protyle/render/av/`

| 文件 | 核心职责 | 可复查入口 |
|------|----------|------------|
| `render.ts` | 渲染总入口、视图分发、搜索、防抖刷新 | `avRender` 函数（L452）、`refreshAV` 函数（L647） |
| `col.ts` | 列（字段）管理：类型切换、添加、删除、重命名 | `genUpdateColItem` 函数（L1263）、`addCol` 函数（L1271） |
| `view.ts` | 视图管理：切换、重命名、分组 | - |
| `cell.ts` | 单元格渲染与编辑 | - |
| `filter.ts` | 过滤规则 UI 与默认操作符 | `getDefaultOperatorByType` 函数 |
| `sort.ts` | 排序规则 UI | - |
| `row.ts` | 行操作与分页 | - |
| `action.ts` | 点击事件分发 | - |
| `openMenuPanel.ts` | 所有配置面板的统一入口 | `openMenuPanel` 函数（L62） |
| `layout.ts` | 布局类型切换 | - |
| `groups.ts` | 分组配置 | - |
| `calc.ts` | 列计算（统计行） | - |
| `select.ts` | 单选/多选字段编辑 | - |
| `date.ts` | 日期字段编辑 | - |
| `number.ts` | 数字格式设置 | - |
| `asset.ts` | 资源字段 | - |
| `relation.ts` | 关联字段配置 | - |
| `rollup.ts` | 汇总字段配置 | - |

### 1.3 后端模块清单

路径：`kernel/av/`

| 文件 | 核心职责 | 可复查入口 |
|------|----------|------------|
| `av.go` | 核心数据结构、AV 加载/保存、视图管理 | `AttributeView` 结构体（L38）、`View` 结构体（L204）、`Key` 结构体（L114）、`SaveAttributeView` 函数（L579） |
| `value.go` | 字段值结构、格式化、判空、判编辑 | `Value` 结构体（L35）、`IsEdited` 方法（L207）、`IsEmpty` 方法（L306） |
| `filter.go` | 过滤执行逻辑、操作符实现 | `Filter` 函数（L90）、`Value.Filter` 方法（L140） |
| `sort.go` | 排序执行逻辑 | `Sort` 函数（L41）、`ViewSort` 结构体（L29） |
| `group.go` | 分组逻辑 | - |
| `layout.go` | 布局基础结构与接口 | `BaseLayout` 结构体（L20）、`Collection` 接口（L188）、`Item` 接口（L236） |
| `layout_table.go` | 表格布局定义与实例 | `LayoutTable` 结构体（L24） |
| `layout_gallery.go` | 画廊布局定义与实例 | `LayoutGallery` 结构体（L24） |
| `layout_kanban.go` | 看板布局定义与实例 | `LayoutKanban` 结构体（L24） |
| `calc.go` | 计算逻辑 | - |
| `relation.go` | 关联字段处理 | - |
| `av_fix.go` | 版本升级与数据兼容 | `UpgradeSpec` 函数（L29） |
| `mirror.go` | 镜像数据库 | - |

### 1.4 业务层与 API

| 路径 | 职责 |
|------|------|
| `kernel/model/attribute_view.go` | 事务处理器：20+ 个 `do*` 函数对应不同操作 |
| `kernel/model/attribute_view_render.go` | 渲染入口：组装数据、调用 SQL 层、返回结果 |
| `kernel/api/av.go` | HTTP API 路由定义 |
| `kernel/model/transaction.go` | 事务调度：根据 `action` 分发到对应 `do*` 函数 |

---

## 2. 字段定义与类型扩展

### 2.1 字段类型全集

共 **17 种** 字段类型，定义在 `kernel/av/av.go` L90-L111：

| 类型常量 | 字符串值 | 说明 |
|----------|----------|------|
| `KeyTypeBlock` | `block` | 主键（绑定到块） |
| `KeyTypeText` | `text` | 文本 |
| `KeyTypeNumber` | `number` | 数字 |
| `KeyTypeDate` | `date` | 日期 |
| `KeyTypeSelect` | `select` | 单选 |
| `KeyTypeMSelect` | `mSelect` | 多选 |
| `KeyTypeURL` | `url` | 链接 |
| `KeyTypeEmail` | `email` | 邮箱 |
| `KeyTypePhone` | `phone` | 电话 |
| `KeyTypeMAsset` | `mAsset` | 资源（多文件） |
| `KeyTypeTemplate` | `template` | 模板 |
| `KeyTypeCreated` | `created` | 创建时间（自动） |
| `KeyTypeUpdated` | `updated` | 更新时间（自动） |
| `KeyTypeCheckbox` | `checkbox` | 复选框 |
| `KeyTypeRelation` | `relation` | 关联 |
| `KeyTypeRollup` | `rollup` | 汇总 |
| `KeyTypeLineNumber` | `lineNumber` | 行号 |

前端类型别名：`TAVCol`，定义在 `app/src/types/index.d.ts` L96-L113。

### 2.2 字段（Key）结构

字段的元数据定义在 `kernel/av/av.go` L114-L146：

```go
type Key struct {
    ID   string  `json:"id"`
    Name string  `json:"name"`
    Type KeyType `json:"type"`
    Icon string  `json:"icon"`
    Desc string  `json:"desc"`

    // 类型特有属性（均为可选）
    Options      []*SelectOption `json:"options,omitempty"`  // 单选/多选
    NumberFormat NumberFormat    `json:"numberFormat"`       // 数字
    Template     string          `json:"template"`           // 模板
    Relation     *Relation       `json:"relation,omitempty"` // 关联
    Rollup       *Rollup         `json:"rollup,omitempty"`   // 汇总
    Date         *Date           `json:"date,omitempty"`     // 日期
    Created      *Created        `json:"created,omitempty"`  // 创建时间
    Updated      *Updated        `json:"updated,omitempty"`  // 更新时间
}
```

**扩展模式**：采用「基础结构 + 可选类型属性」的胖接口模式。所有字段类型共享同一结构体，通过 `Type` 字段标记当前类型，只有对应类型的子结构会被填充和使用。

前端对应结构：`IAVColumn`，定义在 `app/src/types/index.d.ts` L1011-L1042，增加了视图级属性（`width`、`pin`、`hidden`、`wrap`）。

### 2.3 字段值（Value）结构

值的多态结构定义在 `kernel/av/value.go` L35-L62：

```go
type Value struct {
    ID         string  `json:"id,omitempty"`
    KeyID      string  `json:"keyID,omitempty"`
    BlockID    string  `json:"blockID,omitempty"`
    Type       KeyType `json:"type,omitempty"`
    IsDetached bool    `json:"isDetached,omitempty"`

    CreatedAt int64 `json:"createdAt,omitempty"`
    UpdatedAt int64 `json:"updatedAt,omitempty"`

    Block    *ValueBlock    `json:"block,omitempty"`
    Text     *ValueText     `json:"text,omitempty"`
    Number   *ValueNumber   `json:"number,omitempty"`
    Date     *ValueDate     `json:"date,omitempty"`
    MSelect  []*ValueSelect `json:"mSelect,omitempty"`
    URL      *ValueURL      `json:"url,omitempty"`
    // ... 其他类型的值结构（共 17 种）

    IsRenderAutoFill bool `json:"-"` // 仅运行时使用，不序列化
}
```

**关键设计点**：
- 所有类型的值共享同一 `Value` 结构体，通过 `Type` 字段区分
- 每种类型有自己的子结构（`Text`、`Number` 等），为 `nil` 或空时表示空值
- `IsRenderAutoFill` 使用 `json:"-"` 标签，标记渲染期自动填充的值，保存时会被清理（见第 5 节）

前端对应结构：`IAVCellValue`，定义在 `app/src/types/index.d.ts` L1064-L1110。

### 2.4 键值组织方式

数据库采用「键-值分离」的列式存储结构，定义在 `kernel/av/av.go` L50-L54：

```go
type KeyValues struct {
    Key    *Key     `json:"key"`              // 字段定义
    Values []*Value `json:"values,omitempty"` // 该字段的所有值
}
```

每个字段有自己的值列表，值通过 `BlockID` 与行（项目）关联。取值时遍历匹配：

```go
func (kValues *KeyValues) GetValue(blockID string) (ret *Value) {
    for _, v := range kValues.Values {
        if v.BlockID == blockID {
            ret = v
            return
        }
    }
    return
}
```

---

## 3. 视图配置与数据流向

### 3.1 视图结构

视图的核心配置定义在 `kernel/av/av.go` L203-L229：

```go
type View struct {
    ID               string
    Icon             string
    Name             string
    HideAttrViewName bool
    Desc             string
    Filters          []*ViewFilter  `json:"filters,omitempty"` // 过滤规则
    Sorts            []*ViewSort    `json:"sorts,omitempty"`   // 排序规则
    PageSize         int            `json:"pageSize"`          // 分页大小
    LayoutType       LayoutType     `json:"type"`              // 布局类型
    Table            *LayoutTable   `json:"table,omitempty"`   // 表格布局配置
    Gallery          *LayoutGallery `json:"gallery,omitempty"` // 画廊布局配置
    Kanban           *LayoutKanban  `json:"kanban,omitempty"`  // 看板布局配置
    ItemIDs          []string       `json:"itemIds,omitempty"` // 项目排序 ID 列表

    // 分组相关
    Group        *ViewGroup `json:"group,omitempty"`
    Groups       []*View    `json:"groups,omitempty"`
    GroupKey     *Key       `json:"groupKey,omitempty"`
    GroupVal     *Value     `json:"groupVal,omitempty"`
    GroupFolded  bool       `json:"groupFolded"`
    GroupHidden  int        `json:"groupHidden"`
}
```

**注意**：`filters`、`sorts`、`pageSize` 在 `BaseLayout` 中也存在，但标记为废弃（计划 2026 年 6 月删除，参见 `kernel/av/layout.go` L27-L35）。这些字段是历史遗留，当前已迁移到 `View` 层级。

### 3.2 三种布局配置

#### 表格布局（`LayoutTable`）

路径：`kernel/av/layout_table.go` L24-L32

- 特有属性：`Columns`（`ViewTableColumn` 列表），每个列有 `Pin`、`Width`、`Calc`
- 废弃字段：`RowIDs`（已迁移到 `View.ItemIDs`）

#### 画廊布局（`LayoutGallery`）

路径：`kernel/av/layout_gallery.go` L24-L39

- 特有属性：封面来源、卡片宽高比、卡片大小、是否适应图片、是否显示字段名、卡片字段列表
- 废弃字段：`CardIDs`（已迁移到 `View.ItemIDs`）

#### 看板布局（`LayoutKanban`）

路径：`kernel/av/layout_kanban.go` L24-L37

- 特有属性：与画廊类似，额外有 `FillColBackgroundColor`（是否填充列背景色）

### 3.3 数据库级 vs 视图级

| 配置级别 | 存储位置 | 内容 |
|----------|----------|------|
| 数据库级 | `AttributeView.KeyValues` / `KeyIDs` | 字段定义、字段值、字段顺序 |
| 视图级 | `AttributeView.Views[]` | 过滤、排序、分页、布局、分组、字段显示顺序、列宽、固定列、隐藏列 |

一个数据库可以有多个视图，共享字段和数据，但各自独立配置显示方式。

### 3.4 数据读取全链路

```
前端 avRender()
    ↓
[提取 DOM 状态] 选中单元格、滚动位置、分页、搜索词
    ↓
fetchSyncPost("/api/av/renderAttributeView")
    ↓
后端 RenderAttributeView()  [kernel/model/attribute_view_render.go]
    ├─ ParseAttributeView()  // 从 JSON 加载
    ├─ upgradeAttributeViewSpec()  // 版本兼容
    ├─ sql.RenderView()  // SQL 层数据组装
    ├─ renderViewableInstance()  // 过滤/排序/分页
    └─ renderAttributeViewGroups()  // 分组
    ↓
返回 JSON（IAV 结构）
    ↓
前端渲染到 DOM
    ↓
[恢复 DOM 状态] 滚动位置、选中状态、分页
```

#### 前端入口：`avRender`

路径：`app/src/protyle/render/av/render.ts` L452

核心流程：
1. 找到所有 `[data-type="NodeAttributeView"]` 元素
2. **从 DOM 提取当前状态**（`resetData`）：选中单元格、选中行、拖拽填充、滚动位置、搜索词、分页大小、表头 transform
3. 调用 `/api/av/renderAttributeView` 接口获取数据
4. 根据 `data-av-type` 分发：`table` / `gallery` / `kanban`
5. **渲染后恢复状态**：滚动、选中、分页、事件绑定

#### 后端渲染入口：`RenderAttributeView`

路径：`kernel/model/attribute_view_render.go` L38

核心步骤：
1. 加载 AV JSON
2. 数据版本升级
3. SQL 层渲染（从块表查询补充数据）
4. 视图实例构建（应用过滤、排序、分页）
5. 分组视图生成

### 3.5 数据修改全链路

```
用户交互 → 前端事件处理
    ↓
transaction(protyle, [doOps], [undoOps])
    ↓ （乐观更新：先更新 UI 动画）
POST /api/transactions
    ↓
Transaction 执行 [kernel/model/transaction.go]
    ↓ 根据 action 分发
doUpdateAttrViewColumn() 等
    ↓
SaveAttributeView()  [kernel/av/av.go L579]
    ↓ （原子写入 filelock.WriteFile）
刷新页面 / 推送 WS 消息
```

事务特点：
- 每个事务包含 `doOperations`（执行操作）和 `undoOperations`（撤销操作）
- 前端可批量提交多个操作
- 后端按顺序执行，全部成功或全部回滚

---

## 4. 排序与过滤机制

### 4.1 排序机制

#### 排序规则结构

路径：`kernel/av/sort.go` L28-L39

```go
type ViewSort struct {
    Column string    `json:"column"` // 字段 ID
    Order  SortOrder `json:"order"`  // "ASC" / "DESC"
}
```

支持多级排序，按数组顺序优先级递减。

#### 排序算法：已编辑/未编辑分离

路径：`kernel/av/sort.go` L41-L106 起

排序的核心设计是**「已编辑/未编辑分离」**：

1. **判断编辑状态**：遍历每个项目的排序字段值，调用 `val.IsEdited()` 判断
   - 复选框特殊处理：如果主键编辑过，则复选框也算编辑过（L68-L73）
2. **分组排序**：
   - 未编辑的项目：按块创建时间升序排列（`val1.CreatedAt < val2.CreatedAt`，L94-L104）
   - 已编辑的项目：按排序规则进行多级比较（L106-L152）
3. **结果合并**：**已编辑在前，未编辑在后**

   证据：`kernel/av/sort.go` L154-L155：

   ```go
   // 将包含未编辑的项目放在最后
   collection.SetItems(append(editedItems, uneditedItems...))
   ```

   `append(editedItems, uneditedItems...)` 将未编辑列表拼接到已编辑列表之后，注释也明确写了「将未编辑的项目放在最后」。

> **设计意图**：未编辑（即新增但未填值）的项目保持按创建时间排列在底部，避免新行"乱飞"。

#### `IsEdited` 判断逻辑

路径：`kernel/av/value.go` L207-L226

```
IsEdited() = true 的条件：
1. CreatedAt < 1709740800000（2024-03-07 前的旧数据，全部视为已编辑）
2. 类型是 created / updated（自动字段始终视为已编辑）
3. 类型是 checkbox：CreatedAt != UpdatedAt
4. 其他类型：!IsEmpty() || CreatedAt != UpdatedAt
```

### 4.2 过滤机制

#### 过滤规则结构

路径：`kernel/av/filter.go` L29-L59

```go
type ViewFilter struct {
    Column        string           `json:"column"`
    Qualifier     FilterQuantifier `json:"quantifier,omitempty"` // Any / All / None
    Operator      FilterOperator   `json:"operator"`
    Value         *Value           `json:"value"`               // 过滤参考值
    RelativeDate  *RelativeDate    `json:"relativeDate,omitempty"`
    RelativeDate2 *RelativeDate    `json:"relativeDate2,omitempty"`
}
```

#### 过滤操作符

共 **15 种** 操作符（`FilterOperator*` 常量）：

| 操作符 | 说明 |
|--------|------|
| `=` | 等于 |
| `!=` | 不等于 |
| `>` `>=` `<` `<=` | 数值比较 |
| `Contains` / `Does not contains` | 包含/不包含 |
| `Is empty` / `Is not empty` | 空值判断 |
| `Starts with` / `Ends with` | 开头/结尾 |
| `Is between` | 区间 |
| `Is true` / `Is false` | 布尔判断 |

不同字段类型支持的操作符不同，前端通过 `getDefaultOperatorByType` 提供默认操作符（`app/src/protyle/render/av/filter.ts` L17）。

#### 量词（Qualifier）

用于多选等多值字段：
- `Any`（默认）：任意一个值满足
- `All`：所有值都满足
- `None`：没有值满足

#### 相对日期

`RelativeDate` 结构（`kernel/av/filter.go` L55-L59）：
- `Count`：数量
- `Unit`：天/周/月/年
- `Direction`：前/当前/后

#### 过滤执行流程

路径：`kernel/av/filter.go` L90-L138

```
Filter(viewable, attrView):
  1. 找到每个过滤字段对应的列索引（L97-L105）
  2. 遍历所有项目（L108-L136）：
     ├─ 对每个过滤条件 j：
     │   ├─ values[index] 为 nil 时（L114-L126）：
     │   │   ├─ operator = IsNotEmpty → pass=false，但未 break，落入 L122 → 见下文 bug 分析
     │   │   ├─ operator = IsEmpty    → pass=true，break → 安全
     │   │   └─ operator = 其他       → 落入 L122 → 见下文 bug 分析
     │   └─ values[index] 非 nil → 调用 values[index].Filter(filter, ...)
     └─ 所有条件通过则保留
```

#### 空值过滤分支 bug：L122 的 nil 指针解引用

路径：`kernel/av/filter.go` L114-L126

```go
if nil == values[index] {
    if FilterOperatorIsNotEmpty == operator {
        pass = false          // L116: 设为不通过，但没有 break！
    } else if FilterOperatorIsEmpty == operator {
        pass = true           // L118
        break                 // L119: 只有这里安全退出
    }

    if KeyTypeText != values[index].Type {  // L122: values[index] 为 nil！
        pass = false          // L123
    }
    break                     // L125
}
```

**判断：这是确认的 bug。** 当 `values[index]` 为 nil 且操作符不是 `IsEmpty` 时，L122 对 nil 指针访问 `.Type` 字段会导致 Go 运行时 panic（nil pointer dereference）。具体场景：

| 操作符 | 执行路径 | 结果 |
|--------|----------|------|
| `IsEmpty` | L118 pass=true, L119 break | ✅ 安全 |
| `IsNotEmpty` | L116 pass=false, 无 break, 落入 L122 | ❌ panic |
| 其他（`=`、`Contains` 等） | 跳过两个 if, 落入 L122 | ❌ panic |

**修复方式**：应在 L116 后添加 `break`，或在 L122 前添加 nil 守卫。

**实际影响**：此 bug 在正常使用中可能很难触发，因为渲染管线通常为每个字段创建 Value 对象（即使是空值）。但 `values[index]` 的 nil 检查（L114）表明开发者预期此路径可达，可能在字段动态增删的竞态条件下出现。

#### 类型不匹配的降级处理

路径：`kernel/av/filter.go` L145-L148

```go
if nil != filter.Value && value.Type != filter.Value.Type {
    // 由于字段类型被用户编辑过导致和过滤规则值类型不匹配，该情况下不过滤
    return true
}
```

**行为**：当过滤值的类型和字段类型不匹配时，**直接通过（不过滤）**，即跳过这条过滤条件。这是为了兼容字段类型切换后过滤规则尚未更新的情况。

#### 汇总字段的特殊过滤

汇总字段的过滤需要先动态计算汇总值（`kernel/av/filter.go` L160-L192）：
1. 找到关联字段
2. 加载目标数据库
3. 调用 `value.Rollup.BuildContents()` 计算结果
4. 再进行过滤比较

---

## 5. 持久化边界

### 5.1 存储位置与格式

- **路径模式**：`{工作空间}/storage/av/{avID}.json`
- **格式**：JSON，由 `UseSingleLineSave` 控制是否单行
- **原子性**：使用 `filelock.WriteFile` 保证写入原子性

### 5.2 保存前的清理流程

路径：`kernel/av/av.go` L579-L652

`SaveAttributeView` 函数在写入磁盘前执行以下清理：

| 步骤 | 代码位置 | 说明 |
|------|----------|------|
| 版本升级 | L587 `UpgradeSpec(av)` | 确保数据格式为最新版本 |
| 块值去重 | L590-L608 | 主键字段值中重复的 BlockID 只保留一个 |
| 视图 ItemIDs 去重 | L613 | 项目 ID 列表去重 |
| 分页大小修正 | L616-L618 | PageSize < 1 时设为默认值 |
| 清理渲染回填值 | L622-L628 | 删除所有 `IsRenderAutoFill = true` 的值 |

### 5.3 持久化 vs 运行时

| 数据 | 持久化？ | 说明 |
|------|----------|------|
| 字段定义（Key） | ✅ | `keyValues[].key` |
| 字段值（Value） | ✅ | 用户编辑的所有值 |
| 视图配置（过滤/排序/分页） | ✅ | `views[]` |
| 分组配置 | ✅ | `views[].group` |
| 布局配置 | ✅ | `table` / `gallery` / `kanban` |
| 项目 ID 列表 | ✅ | `itemIds`，用于维护自定义排序 |
| **渲染自动填充值** | ❌ | `IsRenderAutoFill` 标记的值，保存时删除 |
| 视图实例数据 | ❌ | 每次请求重新计算 |
| 格式化后的值 | ❌ | 运行时生成 |
| 汇总计算结果 | 部分 | `RollupCalc.Result` 可能缓存 |
| 前端 UI 状态 | ❌ | 选中、滚动、搜索等，存在 DOM 上 |

### 5.4 `IsRenderAutoFill` 的作用

路径：`kernel/av/value.go` L61

这是一个运行时标记（`json:"-"`），用于区分：
- **用户编辑的值**：持久化到磁盘
- **渲染时自动填充的值**：如创建时间、更新时间的显示值，只用于展示，不持久化

保存时会清理所有带此标记的值（`kernel/av/av.go` L622-L628）。

### 5.5 版本升级机制

路径：`kernel/av/av_fix.go` L29-L38

```go
func UpgradeSpec(av *AttributeView) {
    if CurrentSpec <= av.Spec {
        return
    }
    upgradeSpec1(av)
    upgradeSpec2(av)
    upgradeSpec3(av)
    upgradeSpec4(av)
}
```

每次保存前都会调用，确保数据格式升级到最新版本。例如：
- `upgradeSpec4`：为 `created`/`updated` 字段补上 `IncludeTime` 默认值（L40-L59）

---

## 6. 界面状态传递

### 6.1 状态即 DOM

SiYuan 数据库视图的前端状态管理核心原则是：**状态存储在 DOM 上**。

所有交互状态通过 HTML 元素的 `data-*` 属性承载：

```html
<!-- 容器 -->
<div data-type="NodeAttributeView"
     data-av-id="2024xxxx"
     data-av-type="table"
     data-node-id="...">

<!-- 单元格 -->
<div class="av__cell"
     data-col-id="col123"
     data-dtype="text"
     data-wrap="false">

<!-- 行 -->
<div class="av__row"
     data-id="row456">
```

### 6.2 重渲染时的状态提取与恢复

路径：`app/src/protyle/render/av/render.ts` L481-L542

每次重新渲染前，从 DOM 提取状态到 `resetData` 对象：

| 状态项 | 提取方式 |
|--------|----------|
| 选中单元格 | `.av__cell--select` + 父级 groupId / rowId |
| 选中行 | `.av__row--select` |
| 拖拽填充 | `.av__drag-fill` |
| 激活单元格 | `.av__cell--active` |
| 滚动位置 | `.av__scroll.scrollLeft` |
| 表头/表尾 transform | `style^="transform"` |
| 搜索词 | `[data-type="av-search"].textContent` |
| 分页大小 | `.av__body[data-page-size]` |

渲染完成后，`afterRenderTable` 等函数将这些状态恢复到新 DOM 上。

### 6.3 增量更新：`refreshAV`

路径：`app/src/protyle/render/av/render.ts` L647

为避免频繁全量重渲染，`refreshAV` 根据 `operation.action` 类型执行**增量 DOM 更新**：

| 操作类型 | 更新方式 |
|----------|----------|
| `setAttrViewName` | 修改标题文本 |
| `setAttrViewColWidth` | 遍历所有行设置列宽 style |
| `setAttrViewCardSize` | 切换卡片大小 CSS 类 |
| `setAttrViewCardAspectRatio` | 切换封面比例 CSS 类 |
| `hideAttrViewName` | 切换标题显示 CSS 类 |
| `setAttrViewWrapField` | 设置 data-wrap 属性 |
| `setAttrViewFillColBackgroundColor` | 切换看板背景色 CSS 类 |
| ... 其他简单操作 | ... |

**复杂操作**（如添加/删除行、修改排序、修改过滤）则延迟 100ms 后调用 `avRender` 全量重渲染，期间的多次操作会被合并。

### 6.4 配置面板统一入口

路径：`app/src/protyle/render/av/openMenuPanel.ts` L62

所有配置面板（字段属性、过滤、排序、关联、日期选择器等）通过 `openMenuPanel` 函数统一打开。面板类型包括：
`select`、`properties`、`config`、`sorts`、`filters`、`edit`、`date`、`asset`、`switcher`、`relation`、`rollup`

打开面板前会重新请求后端数据，确保配置基于最新状态。

### 6.5 前端内存状态

少量状态存在内存中（不在 DOM 上）：

| 状态 | 位置 | 说明 |
|------|------|------|
| 搜索防抖 | `searchTimeout`（闭包） | 搜索输入延迟触发重渲染 |
| 刷新防抖 | `refreshTimeouts`（Map） | 每个 protyle 一个延迟刷新计时器 |
| 事务队列 | `window.siyuan.transactions` | 待提交的事务队列 |

---

## 7. 类型切换异常边界

字段类型切换是数据库中最容易产生边界问题的操作。以下从代码事实出发，梳理类型切换时可能影响渲染结果和数据一致性的异常场景。

### 7.1 类型切换的核心处理

路径：`kernel/model/attribute_view.go` L4656-L4721

`updateAttributeViewColumn` 函数处理类型切换：

```go
// 切换类型
changeType = keyValues.Key.Type != colType
keyValues.Key.Type = colType

// 同步修改所有值的 Type 标记
for _, value := range keyValues.Values {
    value.Type = colType
}
```

**行为**：
1. 修改 `Key.Type`（字段的类型标记）
2. 修改该字段所有 `Value.Type`（每个值的类型标记）
3. **但不修改值的具体内容结构**（`Text`、`Number` 等子结构保持原样）

### 7.2 类型切换的数据残留问题

#### 现象

类型切换后，旧类型的值结构（如 `text.content`）仍然存在于 JSON 中，只是 `Type` 标记变了。

#### 影响分析

1. **渲染层**：渲染时通过 `Type` 决定读取哪个子结构。切换后旧结构不被读取，相当于"死数据"
2. **序列化**：旧结构仍然占用 JSON 存储空间（虽有 `omitempty`，但如果指针非 nil 就会被序列化）
3. **切回原类型**：如果切回原来的类型，旧数据会"复活"，可能带来意外

> **可复查点**：`Value` 结构体中的子字段均使用指针且标注 `omitempty`。如果类型切换时旧子结构指针不为 nil，它仍会被序列化到 JSON。

### 7.3 类型切换触发的级联清理

类型切换时（`changeType = true`），代码明确处理了两种级联：

#### 7.3.1 分组清理

```go
if groupKey := view.GetGroupKey(attrView); nil != groupKey && groupKey.ID == operation.ID {
    removeAttributeViewGroup0(view)
}
```

**行为**：如果切换的字段是当前分组字段，**清空分组配置**。

#### 7.3.2 关联汇总清理

```go
for _, keyValues := range destAv.KeyValues {
    if av.KeyTypeRollup == keyValues.Key.Type && keyValues.Key.Rollup.KeyID == operation.ID {
        // 置空关联过来的汇总
        for _, val := range keyValues.Values {
            val.Rollup.Contents = nil
        }
        keyValues.Key.Rollup.Calc = &av.RollupCalc{Operator: av.CalcOperatorNone}
    }
}
```

**行为**：如果其他数据库的汇总字段引用了此字段，**清空汇总结果**并将计算方式设为 None。

### 7.4 类型切换未处理的边界

以下场景在类型切换时**没有明确处理**，可能导致异常渲染或数据不一致：

#### 7.4.1 过滤规则残留

- **现象**：过滤规则仍引用旧类型的值和操作符
- **代码证据**：`updateAttributeViewColumn` 中没有清理过滤规则的逻辑
- **降级处理**：过滤执行时，若 `value.Type != filter.Value.Type`，会跳过该过滤条件（见第 4.2 节）
- **用户感知**：过滤条件还在 UI 上显示，但实际上不起作用，可能造成困惑

#### 7.4.2 排序规则残留

- **现象**：排序规则引用的字段还在，但字段类型变了
- **判断：排序不会崩溃，类型不匹配的值在比较时返回 0（视为相等），最终按块创建时间回退排序。**

  证据：`kernel/av/sort.go` L123 调用 `val1.Compare(val2, attrView)`。`Compare` 方法定义在 `kernel/av/sort.go` L161-L479，结构如下：

  ```go
  func (value *Value) Compare(other *Value, attrView *AttributeView) int {
      switch value.Type {
      case KeyTypeBlock:
          if nil != value.Block && nil != other.Block { ... }
      case KeyTypeText:
          if nil != value.Text && nil != other.Text { ... }
      case KeyTypeNumber:
          if nil != value.Number && nil != other.Number { ... }
      // ... 所有 17 种类型都有分支
      }
      return 0   // L478：默认返回 0
  }
  ```

  每个类型分支的入口条件是**两个值的对应子结构都非 nil**。类型切换后：
  - 从 `number` 切换为 `text`：`value.Type = text`，但 `value.Text` 仍为 nil → 不进入 `KeyTypeText` 分支的 if 块 → 落到 L478 `return 0`
  - 从 `text` 切换为 `number`：`value.Type = number`，但 `value.Number` 仍为 nil → 同样落到 L478 `return 0`

  返回 0 表示两值相等，触发 `sort.go` L124-L126 的 `sorted = false; continue`，继续下一级排序规则。若所有排序规则都返回 0，则最终按块创建时间回退排列。不会崩溃，但排序结果不反映用户预期的字段值顺序。

#### 7.4.3 视图字段配置残留

- **现象**：表格列宽、隐藏/显示、固定列等配置在 `LayoutTable.Columns` 中，按 ID 关联，不受类型切换影响
- **影响**：列配置保留，这是预期行为，不是问题

#### 7.4.4 汇总字段引用的目标字段类型变更

**跨数据库汇总**：类型切换时已处理。`updateAttributeViewColumn`（`kernel/model/attribute_view.go` L4698-L4718）会遍历所有引用此字段的其他数据库汇总，清空其 `Rollup.Contents` 并将 `Calc.Operator` 设为 `CalcOperatorNone`。

**本数据库内汇总**：类型切换时未显式清理，但 `BuildContents` 在渲染时做了运行时降级。

证据：`kernel/av/value.go` L864-L866：

```go
if val := destVal.GetValByType(destKey.Type); nil == val || reflect.ValueOf(val).IsNil() {
    // 目标字段因为修改类型导致空值
    continue
}
```

`GetValByType`（`kernel/av/value.go` L421-L465）根据当前 `destKey.Type` 读取对应的值子结构。例如：
- 目标字段类型从 `number` 切换为 `text` 后，`destKey.Type = text`
- `GetValByType(text)` 返回 `value.Text`
- 但原始值存储在 `value.Number` 中，`value.Text` 为 nil
- 条件 `nil == val` 为 true → `continue`（跳过该值）

**判断：本数据库内的汇总在目标字段类型变更后，渲染时自动降级为空值，不会崩溃。** 具体表现：
1. `BuildContents` 跳过类型不匹配的值（L866 continue），汇总结果为空
2. `calcContents` 对空 Contents 计算，结果为零值/空值
3. 汇总单元格显示为空
4. 注释（L865）明确说明这是预期行为：「目标字段因为修改类型导致空值」

#### 7.4.5 看板/画廊封面字段失效

- **现象**：画廊或看板的 `CoverFrom = CoverFromAssetField`，且 `CoverFromAssetKeyID` 指向的字段类型从 `mAsset` 切换为其他类型
- **判断：封面降级为空，不会崩溃，且不会显示旧资源数据。**

  完整链路（以画廊为例，`kernel/sql/av_gallery.go` L196-L212，看板在 `kernel/sql/av_kanban.go` L191-L207）：

  ```go
  case av.CoverFromAssetField:
      if "" == view.Gallery.CoverFromAssetKeyID {
          break                                 // L197-L198：无资源字段 ID → 无封面
      }

      assetValue := attrView.GetValue(view.Gallery.CoverFromAssetKeyID, cardID)
      if nil == assetValue || 1 > len(assetValue.MAsset) {
          break                                 // L201-L203：值为 nil 或 MAsset 为空 → 无封面
      }

      for _, asset := range assetValue.MAsset {
          if asset.Type == av.AssetTypeImage && util.IsPossiblyImage(asset.Content) {
              galleryCard.CoverURL = asset.Content  // L207-L208：找到图片 → 设置封面
              break
          }
      }
      return
  ```

  **降级为空的精确触发条件**（满足任一即不显示封面）：

  | 条件 | 代码行 | 说明 |
  |------|--------|------|
  | `CoverFromAssetKeyID` 为空字符串 | L197-L198 | 未配置资源字段 |
  | `attrView.GetValue()` 返回 nil | L201 | 该单元格从未被写入过任何值 |
  | `len(assetValue.MAsset) < 1` | L202 | `MAsset` 切片为空或 nil |
  | `MAsset` 中所有资源 `Type != AssetTypeImage` | L207 遍历无匹配 | 没有图片资源 |
  | `MAsset` 中图片资源 `util.IsPossiblyImage(content) == false` | L207 | URL 不是合法图片路径 |

  **为什么不会显示旧资源数据**：类型切换时（`kernel/model/attribute_view.go` L4672-L4677），`changeType` 触发后虽然只改 `Key.Type` 和 `Value.Type`、不清空子结构，但封面代码**不检查值的 Type 字段**，直接读 `assetValue.MAsset`。旧 `MAsset` 子结构理论上仍在内存中，但：

  1. 若该单元格在资源字段类型下从未填过值 → `MAsset` 本就为 nil → L202 break
  2. 若该单元格曾填过资源值 → `MAsset` 非 nil，理论上仍会被读到并显示旧封面 → 这是**唯一可能显示旧资源的场景**

  **用户可见结果**：
  - 绝大多数情况下卡片不显示封面（静默降级）
  - 仅当该单元格在原资源字段类型下曾填过图片资源时，切换类型后可能短暂显示旧图片，直到用户编辑该单元格触发值清理

### 7.5 主键（block）类型的特殊性

主键字段类型固定为 `block`，不能切换。所有其他字段的值通过 `BlockID` 与主键关联。

### 7.6 自动字段（created/updated）的特殊性

- `created` 和 `updated` 字段的值由系统自动填充
- `IsEdited()` 始终返回 `true`（`kernel/av/value.go` L213-L215）
- 这些字段参与排序时始终被视为"已编辑"

---

## 8. 渲染结果异常边界

以下梳理可能影响渲染结果正确性的异常场景和降级策略。

### 8.1 数据加载异常

| 场景 | 处理方式 | 代码位置 |
|------|----------|----------|
| JSON 文件不存在 | 自动创建空数据库 | `RenderAttributeView` 入口判断 |
| JSON 解析失败 | 记录错误日志，返回 nil | `ParseAttributeView` |
| 视图 ID 不存在 | 回退到 `av.ViewID` 指向的视图；都找不到时用第一个视图 | `av.GetCurrentView`（`kernel/av/av.go` L664） |
| 关联数据库不存在 | 返回空关联，汇总结果为空 | `relation.go` 中 destAv 为 nil 时的处理 |
| 大文件警告 | 超过阈值时推送错误消息（7 秒自动消失） | `SaveAttributeView` L647-L650 |

### 8.2 字段值异常

#### 8.2.1 值为 nil

- **渲染**：显示为空单元格
- **排序**：nil 被视为空值，排在非空值后面（见 8.3.1 分析）
- **过滤**：见 4.2 节「空值过滤分支 bug」分析。当 `values[index]` 为 nil 且操作符不是 `IsEmpty` 时，存在 nil 指针解引用 bug

#### 8.2.2 类型不匹配

- **过滤**：跳过该过滤条件（返回 true，即通过），见第 4.2 节
- **排序**：直接按当前类型比较，可能产生意外排序结果
- **渲染**：按 `Type` 字段读取对应子结构，旧类型的数据不显示

#### 8.2.3 重复块绑定

- **现象**：主键字段中同一个 `BlockID` 出现多次
- **处理**：保存时去重，只保留一个（`SaveAttributeView` L590-L608）

#### 8.2.4 模板渲染错误

模板字段使用 Go template 语法，渲染入口在 `kernel/sql/av.go` L604-L650：

1. **模板渲染失败**：`renderTemplateField` 返回 error 时（L636），构造错误信息但继续执行，失败单元格的 `Template.Content` 仍为空字符串（L645：`content` 为空）
2. **错误上报**：渲染完成后，调用方（`kernel/sql/av_table.go` L128-L131、`av_kanban.go` L130-L133、`av_gallery.go` L135-L138）通过 `util.PushErrMsg` 推送错误消息，显示 30000ms（30 秒）
3. **用户可见结果**：模板渲染失败的单元格显示为空，同时右上角弹出红色错误提示「数据库模板字段渲染失败：xxx」

### 8.3 排序异常

#### 8.3.1 空值排序

路径：`kernel/av/sort.go` L106-L152

已编辑项目中的空值排序逻辑：

```go
if nil == val1 || val1.IsEmpty() {
    if nil != val2 && !val2.IsEmpty() {
        return false    // val1 空 val2 非空 → val1 排在后面
    }
    sorted = false      // 两者都空，无法区分，继续下一级排序
    continue
} else {
    if nil == val2 || val2.IsEmpty() {
        return true     // val1 非空 val2 空 → val1 排在前面
    }
}
```

**判断**：无论升序还是降序，空值始终排在非空值后面。具体行为：
- val1 空 + val2 非空 → `return false`（val1 排后）
- val1 非空 + val2 空 → `return true`（val1 排前）
- 两者都空 → `sorted = false`，按下一级排序规则比较，最终回退到块创建时间

这意味着降序排序时空值也排在后面，而非排在前面。

#### 8.3.2 复选框排序特殊性

- 复选框的 `IsEdited` 判断使用 `CreatedAt != UpdatedAt`，而不是 `IsEmpty`
- 因为"复选框不会为空，即使未勾选也不算是空"（代码注释 L217-L219）
- 如果主键编辑过，复选框即使未编辑过也算编辑过（L68-L73）

### 8.4 过滤异常

#### 8.4.1 参考值为空

- 过滤参考值为空或 nil 时，直接通过（返回 true）
- `kernel/av/filter.go` L141-L143

#### 8.4.2 字段找不到

- 过滤字段 ID 在视图字段中找不到时，该过滤条件被忽略（不加入 `colIndexes`）
- `kernel/av/filter.go` L98-L105

#### 8.4.3 相对日期计算

- 相对日期基于当前时间计算
- 每次渲染重新计算，所以"过去 7 天"是动态的

### 8.5 分组异常

- 分组字段类型切换时，分组被清空（见第 7.3.1 节）
- 分组折叠状态保存在视图配置中
- 空白分组可设置隐藏（`GroupHidden = 1` 表示空白隐藏）

### 8.6 并发与一致性

#### 8.6.1 写入原子性

- 使用 `filelock.WriteFile` 保证文件写入原子性
- 但数据库级别的并发修改仍可能冲突（后写覆盖先写）

#### 8.6.2 乐观更新风险

- 前端在提交事务前先更新 UI（乐观更新）
- 如果后端事务失败，UI 已经变了，可能造成短暂不一致
- 前端 `transaction` 函数会在失败时调用 `undoOperations` 回滚 UI 状态

#### 8.6.3 多端同步

- 修改通过 WebSocket 推送到其他端
- 其他端收到推送后刷新对应 AV 块
- 潜在冲突：两端同时修改同一字段

### 8.7 大数据量边界

- 所有数据加载到内存中处理（过滤、排序都在内存中完成）
- 数据量过大时可能有性能问题
- 分页是**后端分页**（返回指定页的数据），但过滤排序是全量后再分页

### 8.8 关联与汇总的级联

#### 8.8.1 双向关联

- 双向关联时，修改一端会同步另一端
- 级联修改通过 `relatedAvIDs` 机制处理

#### 8.8.2 循环引用风险

- 两个数据库互相关联不会导致递归渲染
- `rollupFurtherCollections` 参数用于传递已经渲染过的集合实例，避免重复计算
- 关联字段的渲染只读取目标数据库的值，不会触发目标数据库的完整渲染流程
- 因此不存在递归深度问题

### 8.9 镜像数据库

- 镜像引用同一个 AV JSON，但不同块可设置不同当前视图
- 修改数据时所有镜像都会反映（共享同一份 JSON）
- 视图配置是数据库级的（存储在 `views[]` 中），所有镜像共享
- 但每个块通过 `custom-av-view` 属性指定当前显示哪个视图 ID
- 因此不同镜像可以显示同一数据库的不同视图

---

## 9. 后续调查清单

以下方向值得进一步深入验证。

### 9.1 类型切换相关

- [ ] 类型切换后旧值结构是否真的残留？构造测试用例验证 JSON 输出
- [ ] 类型切换时是否应该清理过滤规则？当前跳过过滤的用户体验是否合理
- [ ] 切回原类型时，旧数据"复活"是预期行为还是 bug？
- [ ] 封面字段类型切换后，旧 `MAsset` 指针非 nil 时显示过期封面的具体复现条件

### 9.2 渲染正确性

- [ ] `Filter` 函数 L122 nil 指针解引用 bug 的修复和测试
- [ ] 模板字段渲染失败的具体降级表现验证
- [ ] 分组字段值为 nil 时，项目会被分到哪一组？

### 9.3 性能边界

- [ ] 全量内存过滤排序在大数据量下的性能表现
- [ ] 汇总字段的计算缓存策略（每次过滤都重新计算？）
- [ ] 前端全量重渲染 vs 增量更新的分界是否合理
- [ ] 搜索功能的实现：前端过滤还是后端过滤？

### 9.4 数据一致性

- [ ] 双向关联的修改事务是否原子
- [ ] `filelock.WriteFile` 的具体实现和重试策略

### 9.5 版本与兼容性

- [ ] `BaseLayout` 中废弃字段的清理计划（2026 年 6 月）
- [ ] 各 spec 版本的具体变更内容
- [ ] 历史数据的完整升级链路覆盖

---

## 附录：文件相对路径速查

### 前端核心文件

| 功能 | 相对路径 |
|------|----------|
| 类型定义 | `app/src/types/index.d.ts` |
| 渲染总入口 | `app/src/protyle/render/av/render.ts` |
| 字段管理 | `app/src/protyle/render/av/col.ts` |
| 过滤 UI | `app/src/protyle/render/av/filter.ts` |
| 配置面板入口 | `app/src/protyle/render/av/openMenuPanel.ts` |
| 事务提交 | `app/src/protyle/wysiwyg/transaction.ts` |

### 后端核心文件

| 功能 | 相对路径 |
|------|----------|
| 核心数据结构 | `kernel/av/av.go` |
| 值结构与工具 | `kernel/av/value.go` |
| 过滤逻辑 | `kernel/av/filter.go` |
| 排序逻辑 | `kernel/av/sort.go` |
| 布局基础 | `kernel/av/layout.go` |
| 版本升级 | `kernel/av/av_fix.go` |
| 事务处理 | `kernel/model/attribute_view.go` |
| 渲染入口 | `kernel/model/attribute_view_render.go` |
| API 路由 | `kernel/api/av.go` |
| 事务调度 | `kernel/model/transaction.go` |
