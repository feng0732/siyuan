# SiYuan 块数据库视图与字段类型代码分析

## 概述

SiYuan（思源笔记）的块数据库（Attribute View，简称 AV）是其结构化能力的核心。数据库以独立的 JSON 文件形式存储，支持多种字段类型和多种视图布局（表格、画廊、看板），通过前后端协作完成数据的读取、渲染、编辑和持久化。

本文档从代码组织角度分析字段定义、视图配置、数据读取及界面展示间的状态传递机制，重点关注类型扩展、校验规则、排序过滤和持久化边界。

---

## 1. 职责划分

### 1.1 整体架构

SiYuan 采用前后端分离架构，数据库视图相关代码分布在三个主要层次：

| 层次 | 目录 | 主要职责 |
|------|------|----------|
| 前端渲染层 | [app/src/protyle/render/av/](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/) | 界面渲染、用户交互、状态管理 |
| 后端 API 层 | [kernel/api/av.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/api/av.go) | HTTP 接口定义、参数校验 |
| 后端核心层 | [kernel/av/](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/) | 数据结构定义、过滤排序、视图渲染逻辑 |
| 后端业务层 | [kernel/model/attribute_view.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/model/attribute_view.go) | 事务处理、持久化操作、业务规则 |

### 1.2 前端模块职责

| 模块文件 | 职责 |
|----------|------|
| [render.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/render.ts) | 渲染入口、视图调度、搜索处理 |
| [col.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/col.ts) | 字段（列）管理、类型切换、字段编辑 |
| [view.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/view.ts) | 视图管理、视图切换、视图配置 |
| [cell.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/cell.ts) | 单元格渲染、值解析、单元格编辑 |
| [filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/filter.ts) | 过滤规则配置、过滤条件编辑 |
| [sort.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/sort.ts) | 排序规则配置 |
| [action.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/action.ts) | 点击事件分发、交互处理 |
| [openMenuPanel.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/openMenuPanel.ts) | 配置面板统一入口 |
| [layout.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/layout.ts) | 布局类型切换 |
| [groups.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/groups.ts) | 分组配置与管理 |
| [row.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/row.ts) | 行操作、行选择、分页 |
| [calc.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/calc.ts) | 列计算（统计行） |
| [relation.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/relation.ts) | 关联字段处理 |
| [rollup.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/rollup.ts) | 汇总字段处理 |
| [select.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/select.ts) | 单选/多选字段处理 |
| [date.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/date.ts) | 日期字段处理 |
| [number.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/number.ts) | 数字字段格式化 |
| [asset.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/asset.ts) | 资源字段处理 |
| [blockAttr.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/blockAttr.ts) | 块属性格式渲染 |

### 1.3 后端模块职责

| 模块文件 | 职责 |
|----------|------|
| [av.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go) | 核心数据结构定义、AV 加载/保存、视图管理 |
| [value.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/value.go) | 字段值结构、值格式化、值比较 |
| [filter.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/filter.go) | 过滤逻辑、操作符实现 |
| [sort.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/sort.go) | 排序逻辑 |
| [group.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/group.go) | 分组逻辑 |
| [layout.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/layout.go) | 布局基础结构、接口定义 |
| [layout_table.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/layout_table.go) | 表格布局实现 |
| [layout_gallery.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/layout_gallery.go) | 画廊布局实现 |
| [layout_kanban.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/layout_kanban.go) | 看板布局实现 |
| [calc.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/calc.go) | 计算逻辑 |
| [relation.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/relation.go) | 关联字段处理 |
| [mirror.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/mirror.go) | 镜像数据库处理 |

---

## 2. 数据流向

### 2.1 整体数据流图

```
用户交互 → 前端事件处理 → 事务队列 → 后端 API → 业务逻辑 → 持久化
     ↑                                                        ↓
     └──────────────── 渲染更新 ←── WS 推送 ←───────────────┘
```

### 2.2 数据读取流程

#### 前端渲染入口

[avRender](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/prototype/render/av/render.ts#L452-L617) 是前端渲染的总入口，流程如下：

1. **定位 AV 元素**：查找所有 `[data-type="NodeAttributeView"]` 元素
2. **保存当前状态**：在重新渲染前，从 DOM 中提取当前状态（选中单元格、滚动位置、分页大小等）
3. **请求后端数据**：调用 `/api/av/renderAttributeView` 接口
4. **视图分发**：根据 `viewType` 分发到不同的渲染函数
   - `table` → 表格视图
   - `gallery` → 画廊视图
   - `kanban` → 看板视图
5. **渲染后恢复**：将之前保存的状态恢复到新渲染的 DOM 上

#### 后端渲染流程

[RenderAttributeView](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/model/attribute_view_render.go#L38-L67) 是后端渲染入口：

1. **加载 AV 数据**：从 `storage/av/{id}.json` 解析 JSON
2. **获取目标视图**：根据 `viewID` 或默认视图查找
3. **数据兼容升级**：`upgradeAttributeViewSpec` 处理旧版本数据
4. **SQL 层渲染**：`sql.RenderView` 执行数据查询和组装
5. **视图实例渲染**：`renderViewableInstance` 生成分页、过滤、排序后的视图
6. **分组视图渲染**：`renderAttributeViewGroups` 生成分组数据

### 2.3 数据修改流程（事务机制）

SiYuan 使用事务机制确保数据一致性和撤销/重做能力。

#### 前端事务

前端通过 [transaction](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/prototype/wysiwyg/transaction.ts) 函数提交操作：

```typescript
// 示例：更新字段名
transaction(protyle, [
    {
        action: "updateAttrViewCol",
        id: colId,
        avID: avID,
        name: newValue,
        type: colData.type,
    }
], [
    // 撤销操作
    {
        action: "updateAttrViewCol",
        id: colId,
        avID: avID,
        name: colData.name,
        type: colData.type,
    }
]);
```

事务特点：
- **批量提交**：多个操作可合并为一个事务
- **乐观更新**：前端先更新本地状态（动画效果），再异步提交后端
- **队列管理**：事务按顺序执行，前一个完成后才执行下一个
- **撤销支持**：每个 doOperation 对应一个 undoOperation

#### 后端事务

后端通过 `/api/transactions` 接口接收事务，在 `kernel/model/transaction.go` 中执行。每个操作对应一个处理函数，如：
- `updateAttrViewCol` → 更新字段
- `setAttrViewSorts` → 设置排序
- `setAttrViewColHidden` → 设置字段隐藏

### 2.4 状态更新机制

修改提交后，状态通过两种方式同步到前端：

1. **直接刷新**：前端提交后调用 `avRender` 重新渲染
2. **增量更新**：`refreshAV` 函数根据 `operation.action` 类型执行局部更新，避免全量重渲染
3. **WebSocket 推送**：多端同步时通过 WS 推送更新

---

## 3. 字段定义与类型扩展

### 3.1 字段类型体系

SiYuan 共支持 **17 种** 字段类型，定义在 [KeyType](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go#L90-L111) 常量和 [TAVCol](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/types/index.d.ts#L96-L113) 类型中。

| 类型 | KeyType 常量 | 说明 |
|------|-------------|------|
| 主键 | `block` | 数据库的主键字段，关联到具体块 |
| 文本 | `text` | 普通文本内容 |
| 数字 | `number` | 数值，支持格式化 |
| 日期 | `date` | 日期/时间，支持范围 |
| 单选 | `select` | 单个选项 |
| 多选 | `mSelect` | 多个选项 |
| URL | `url` | 链接地址 |
| 邮箱 | `email` | 电子邮件地址 |
| 电话 | `phone` | 电话号码 |
| 资源 | `mAsset` | 文件/图片资源 |
| 模板 | `template` | 模板渲染内容 |
| 创建时间 | `created` | 行创建时间（自动） |
| 更新时间 | `updated` | 行更新时间（自动） |
| 复选框 | `checkbox` | 布尔值 |
| 关联 | `relation` | 关联到其他数据库 |
| 汇总 | `rollup` | 基于关联字段的汇总计算 |
| 行号 | `lineNumber` | 自动编号 |

### 3.2 字段数据结构

#### 后端字段定义

[Key](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go#L114-L146) 结构体定义了字段的元数据：

```go
type Key struct {
    ID   string  `json:"id"`   // 字段 ID
    Name string  `json:"name"` // 字段名
    Type KeyType `json:"type"` // 字段类型
    Icon string  `json:"icon"` // 字段图标
    Desc string  `json:"desc"` // 字段描述

    // 类型特有属性
    Options      []*SelectOption `json:"options,omitempty"`  // 单选/多选选项
    NumberFormat NumberFormat    `json:"numberFormat"`       // 数字格式化
    Template     string          `json:"template"`           // 模板内容
    Relation     *Relation       `json:"relation,omitempty"` // 关联信息
    Rollup       *Rollup         `json:"rollup,omitempty"`   // 汇总信息
    Date         *Date           `json:"date,omitempty"`     // 日期设置
    Created      *Created        `json:"created,omitempty"`  // 创建时间设置
    Updated      *Updated        `json:"updated,omitempty"`  // 更新时间设置
}
```

#### 前端字段定义

[IAVColumn](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/types/index.d.ts#L1011-L1042) 接口定义了前端使用的字段结构，增加了视图相关属性：

```typescript
interface IAVColumn {
    width: string;      // 列宽
    icon: string;       // 图标
    id: string;         // ID
    name: string;       // 名称
    desc: string;       // 描述
    wrap: boolean;      // 是否换行
    pin: boolean;       // 是否固定
    hidden: boolean;    // 是否隐藏
    type: TAVCol;       // 类型
    numberFormat: string; // 数字格式
    // ... 其他类型特有属性
}
```

### 3.3 类型扩展机制

#### 扩展方式

字段类型采用 **"基础结构 + 类型特有属性"** 的扩展模式：

1. **统一的 Key 结构**：所有字段类型共享相同的基础字段（ID、名称、图标、描述）
2. **可选的类型配置**：每种类型有自己的配置子结构（如 `Options`、`Relation`、`Rollup`）
3. **值结构多态**：[Value](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/value.go#L35-L61) 结构体为每种类型提供独立的字段

#### 值的多态实现

后端 [Value](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/value.go#L35-L61) 结构体采用"胖接口"模式：

```go
type Value struct {
    ID         string  `json:"id,omitempty"`
    KeyID      string  `json:"keyID,omitempty"`
    BlockID    string  `json:"blockID,omitempty"`
    Type       KeyType `json:"type,omitempty"`
    IsDetached bool    `json:"isDetached,omitempty"`

    Block    *ValueBlock    `json:"block,omitempty"`
    Text     *ValueText     `json:"text,omitempty"`
    Number   *ValueNumber   `json:"number,omitempty"`
    Date     *ValueDate     `json:"date,omitempty"`
    MSelect  []*ValueSelect `json:"mSelect,omitempty"`
    // ... 其他类型的值结构
}
```

前端 [IAVCellValue](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/types/index.d.ts#L1064-L1110) 也采用相同模式。

#### 类型切换

在 [col.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/prototype/render/av/col.ts#L237-L243) 中，所有类型通过 `genUpdateColItem` 函数生成切换选项：

```typescript
${genUpdateColItem("text", colData.type)}
${genUpdateColItem("number", colData.type)}
${genUpdateColItem("select", colData.type)}
// ... 共 17 种类型
```

类型切换时，数据会尽量保留兼容部分，但不兼容的部分会丢失。

---

## 4. 视图配置

### 4.1 视图类型

SiYuan 支持三种视图布局类型，定义在 [TAVView](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/types/index.d.ts#L95) 中：

- `table` - 表格视图
- `gallery` - 画廊视图
- `kanban` - 看板视图

### 4.2 视图数据结构

#### 后端视图结构

[View](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go#L204-L229) 结构体是视图的核心定义：

```go
type View struct {
    ID               string         `json:"id"`
    Icon             string         `json:"icon"`
    Name             string         `json:"name"`
    HideAttrViewName bool           `json:"hideAttrViewName"`
    Desc             string         `json:"desc"`
    Filters          []*ViewFilter  `json:"filters,omitempty"`
    Sorts            []*ViewSort    `json:"sorts,omitempty"`
    PageSize         int            `json:"pageSize"`
    LayoutType       LayoutType     `json:"type"`
    Table            *LayoutTable   `json:"table,omitempty"`
    Gallery          *LayoutGallery `json:"gallery,omitempty"`
    Kanban           *LayoutKanban  `json:"kanban,omitempty"`
    ItemIDs          []string       `json:"itemIds,omitempty"`

    // 分组相关
    Group        *ViewGroup `json:"group,omitempty"`
    Groups       []*View    `json:"groups,omitempty"`
    GroupKey     *Key       `json:"groupKey,omitempty"`
    GroupVal     *Value     `json:"groupVal,omitempty"`
    GroupFolded  bool       `json:"groupFolded"`
    GroupHidden  int        `json:"groupHidden"`
}
```

#### 视图的"配置-实例"分离模式

值得注意的是，SiYuan 采用了**配置与渲染实例分离**的设计：

- **配置层**：`View` 结构体，存储在 JSON 文件中，包含用户配置
- **实例层**：`BaseInstance`、`TableInstance` 等，由渲染过程生成，包含分页后的数据

[BaseLayout](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/layout.go#L20-L35) 中可以看到废弃字段的注释，表明了从"视图直接持有所有数据"向"配置与实例分离"的演进。

### 4.3 视图共享与视图特有配置

**数据库级共享配置**（存储在 `keyValues` 中）：
- 字段定义（Key）
- 字段值（Value）

**视图级特有配置**（存储在 `views` 中）：
- 字段顺序
- 字段宽度（表格视图）
- 字段隐藏/显示
- 字段固定（pin）
- 过滤规则
- 排序规则
- 分组规则
- 分页大小
- 布局类型特有配置（卡片大小、封面设置等）

---

## 5. 排序与过滤

### 5.1 排序机制

#### 排序数据结构

[ViewSort](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/sort.go#L28-L39) 定义排序规则：

```go
type ViewSort struct {
    Column string    `json:"column"` // 字段 ID
    Order  SortOrder `json:"order"`  // ASC / DESC
}
```

支持多级排序，按数组顺序优先级递减。

#### 排序算法

[Sort](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/sort.go#L41) 函数实现排序逻辑：

1. **已编辑/未编辑分离**：未编辑的项目按块创建时间排序，已编辑的项目按字段值排序
   - 目的：保持新添加但未编辑的项目在列表底部
   - 判断依据：`val.IsEdited()` 检查值是否被手动编辑过

2. **类型特定排序**：每种值类型实现自己的比较逻辑
   - 文本：字典序
   - 数字：数值比较
   - 日期：时间戳比较
   - 复选框：false < true

3. **多级排序**：按排序规则数组依次比较

### 5.2 过滤机制

#### 过滤数据结构

[ViewFilter](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/filter.go#L29-L36) 定义过滤规则：

```go
type ViewFilter struct {
    Column        string           `json:"column"`
    Qualifier     FilterQuantifier `json:"quantifier,omitempty"` // Any/All/None
    Operator      FilterOperator   `json:"operator"`
    Value         *Value           `json:"value"`
    RelativeDate  *RelativeDate    `json:"relativeDate,omitempty"`
    RelativeDate2 *RelativeDate    `json:"relativeDate2,omitempty"`
}
```

#### 操作符类型

共支持 **15 种** 过滤操作符：

| 操作符 | 说明 |
|--------|------|
| `=` | 等于 |
| `!=` | 不等于 |
| `>` | 大于 |
| `>=` | 大于等于 |
| `<` | 小于 |
| `<=` | 小于等于 |
| `Contains` | 包含 |
| `Does not contains` | 不包含 |
| `Is empty` | 为空 |
| `Is not empty` | 不为空 |
| `Starts with` | 开头是 |
| `Ends with` | 结尾是 |
| `Is between` | 在...之间 |
| `Is true` | 为真 |
| `Is false` | 为假 |

不同字段类型支持的操作符不同，由前端 [getDefaultOperatorByType](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/prototype/render/av/filter.ts#L17-L27) 提供默认操作符。

#### 量词（Qualifier）

用于多选等多值字段：
- `Any`：任意一个满足（默认）
- `All`：全部满足
- `None`：都不满足

#### 相对日期过滤

日期类型支持相对日期过滤，如"过去 7 天"、"下个月"等。[RelativeDate](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/filter.go#L55-L59) 结构：

```go
type RelativeDate struct {
    Count     int                   `json:"count"`     // 数量
    Unit      RelativeDateUnit      `json:"unit"`      // 天/周/月/年
    Direction RelativeDateDirection `json:"direction"` // 前/当前/后
}
```

#### 过滤执行流程

[Filter](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/filter.go#L90) 函数执行过滤：

1. 找到过滤字段对应的列索引
2. 遍历所有项目，对每个项目：
   - 获取字段值
   - 根据操作符调用值的 `Filter` 方法
   - 所有过滤条件都通过则保留

3. 类型不匹配时不进行过滤（兼容字段类型变更的情况）

---

## 6. 持久化边界

### 6.1 存储位置与格式

- **存储路径**：`{工作空间}/storage/av/{avID}.json`
- **存储格式**：JSON（支持单行或缩进格式，由 `UseSingleLineSave` 控制）
- **文件锁**：使用 `filelock.WriteFile` 原子写入，保证并发安全

### 6.2 保存时的数据清理

[SaveAttributeView](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go#L579-L652) 函数在保存前执行多项清理：

1. **格式版本升级**：`UpgradeSpec(av)` 处理数据格式兼容
2. **块值去重**：检查主键字段值，移除重复的块绑定
3. **视图值去重**：`ItemIDs` 去重，保证项目 ID 列表唯一
4. **分页大小校验**：`PageSize < 1` 时设置为默认值
5. **清理渲染回填值**：移除 `IsRenderAutoFill` 标记的值

> **关键点**：`IsRenderAutoFill` 是渲染过程中自动填充的值（如创建时间、更新时间的显示值），这些值只用于展示，不持久化到磁盘。

### 6.3 持久化边界划分

| 数据类别 | 持久化 | 说明 |
|----------|--------|------|
| 字段定义（Key） | ✅ | 存储在 `keyValues[].key` |
| 字段值（Value） | ✅ | 用户编辑过的值持久化 |
| 视图配置 | ✅ | 存储在 `views[]` 中 |
| 过滤规则 | ✅ | 视图级配置 |
| 排序规则 | ✅ | 视图级配置 |
| 分组规则 | ✅ | 视图级配置 |
| 项目 ID 列表 | ✅ | 用于维护自定义排序 |
| 分页大小 | ✅ | 视图级配置 |
| 渲染结果数据 | ❌ | 每次请求重新生成 |
| 分页后的数据 | ❌ | 只返回当前页 |
| 格式化后的值 | ❌ | 运行时计算，不存储 |
| 自动填充值（IsRenderAutoFill） | ❌ | 渲染时生成，保存时清理 |
| 临时 UI 状态 | ❌ | 选中单元格、滚动位置等 |

### 6.4 数据库与块的关系

- **数据库独立存储**：每个数据库是一个独立的 JSON 文件
- **块引用数据库**：块通过 `custom-av-id` 属性引用数据库
- **主键绑定块**：数据库的 `block` 类型字段值关联到具体的块 ID
- **一对多关系**：一个数据库可被多个块引用（镜像），各块可独立设置当前视图

---

## 7. 界面展示与状态传递

### 7.1 前端渲染流程

[avRender](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/prototype/render/av/render.ts#L452-L617) 函数的核心流程：

```
1. 从 DOM 提取当前状态（resetData）
   ├─ 选中单元格 ID
   ├─ 选中行 ID 列表
   ├─ 拖拽填充位置
   ├─ 滚动位置
   ├─ 表头/表尾 transform（固定表头）
   ├─ 搜索框内容
   └─ 各分组分页大小

2. 调用 /api/av/renderAttributeView 获取数据

3. 根据 viewType 分发到具体渲染函数
   ├─ 表格 → renderGroupTable / 直接渲染
   ├─ 画廊 → renderGallery
   └─ 看板 → renderKanban

4. 渲染后恢复状态（afterRenderTable）
   ├─ 恢复滚动位置
   ├─ 恢复选中状态
   ├─ 恢复分页大小
   ├─ 恢复拖拽填充
   ├─ 绑定事件监听器
   └─ 聚焦处理
```

### 7.2 状态传递机制

#### DOM 数据属性（data-*）

SiYuan 大量使用 HTML 元素的 `data-*` 属性传递状态：

```html
<!-- 视图标签 -->
<div class="item item--focus" 
     data-id="view-id" 
     data-av-type="table"
     data-page="50">

<!-- 单元格 -->
<div class="av__cell av__cell--select" 
     data-id="cell-id"
     data-col-id="col-id"
     data-dtype="text"
     data-wrap="false">
```

特点：
- **状态即 DOM**：UI 状态直接绑定在 DOM 元素上
- **重新渲染时提取-恢复**：渲染前从 DOM 提取状态，渲染后恢复
- **便于事件委托**：事件处理时直接从 target 元素读取数据

#### 前端内存状态

除了 DOM 状态，还有少量内存状态：

- `searchTimeout`：搜索输入防抖计时器
- `refreshTimeouts`：刷新防抖计时器（每个 protyle 一个）
- `window.siyuan.transactions`：事务队列

### 7.3 增量更新机制

为了避免频繁全量重渲染，[refreshAV](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/prototype/render/av/render.ts#L647-L899) 函数实现了增量更新：

- **简单操作**：直接操作 DOM 更新（如修改列宽、修改名称、切换冻结等）
- **复杂操作**：延迟 100ms 后全量重渲染（如添加/删除行、修改排序等）
- **事务合并**：快速连续操作会被合并，只触发一次重渲染

### 7.4 配置面板

[openMenuPanel](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/prototype/render/av/openMenuPanel.ts#L62-L74) 是所有配置面板的统一入口：

- 类型包括：`select`、`properties`、`config`、`sorts`、`filters`、`edit`、`date`、`asset`、`switcher`、`relation`、`rollup`
- 每次打开面板前会重新请求后端数据，确保配置基于最新状态
- 面板中的修改通过 `transaction` 提交

---

## 8. 校验规则

### 8.1 前端校验

前端校验主要在用户输入时进行：

1. **选项重复校验**：添加选择项时检查是否已存在
2. **空值校验**：字段名为空时不提交
3. **数字格式校验**：数字字段输入时验证
4. **日期格式校验**：日期字段验证日期合法性

### 8.2 后端校验/清理

后端更侧重于数据清理和一致性保证：

1. **ID 格式校验**：`InvalidIDPattern` 检查 ID 格式
2. **值去重**：保存时移除重复的块值
3. **分页大小修正**：小于 1 时设为默认值
4. **视图值去重**：`ItemIDs` 去重
5. **类型不兼容降级**：过滤时字段类型不匹配则跳过该过滤

### 8.3 异常情况处理

#### 已知异常场景及处理

| 异常场景 | 处理方式 | 代码位置 |
|----------|----------|----------|
| 数据库文件不存在 | 自动创建空数据库 | [RenderAttributeView](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/model/attribute_view_render.go#L41-L57) |
| 字段类型变更后过滤值不匹配 | 跳过该过滤条件 | [Value.Filter](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/filter.go#L145-L148) |
| 关联数据库不存在 | 返回空关联 | [relation.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/relation.go) |
| JSON 解析失败 | 记录错误日志，返回空 | [ParseAttributeView](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go) |
| 大文件警告 | 超过阈值时推送警告消息 | [SaveAttributeView](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go#L647-L650) |
| 视图 ID 不存在 | 回退到默认视图 | [GetCurrentView](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go#L664-L680) |
| 并发写入 | 文件锁保证原子性 | `filelock.WriteFile` |
| 未找到指定分组 | 使用默认分组 | 分组渲染逻辑 |

#### 潜在风险点

1. **事务乐观更新失败**：前端先更新 UI，若后端失败可能出现状态不一致
2. **类型转换数据丢失**：字段类型切换时，旧类型的值可能无法转换
3. **关联循环引用**：两个数据库互相关联可能导致递归渲染问题
4. **大数据量性能**：所有数据加载到内存处理，数据量过大时可能有性能问题
5. **镜像同步延迟**：多块引用同一数据库时，更新可能需要逐块刷新

---

## 9. 后续调查清单

以下是值得进一步深入调查的方向：

### 9.1 性能相关

- [ ] 数据库数据量增大时的渲染性能瓶颈分析
- [ ] 分页机制的具体实现（前端分页 vs 后端分页）
- [ ] 虚拟滚动的实现情况
- [ ] SQL 层渲染的具体查询逻辑

### 9.2 数据一致性

- [ ] 事务失败时的回滚机制细节
- [ ] 多端同步时的冲突处理策略
- [ ] 关联字段的双向同步机制
- [ ] 汇总字段的计算时机和缓存策略

### 9.3 扩展性

- [ ] 插件系统如何扩展数据库字段类型
- [ ] 视图布局的扩展点
- [ ] 公式/计算字段的实现机制
- [ ] 数据库 API 的完整列表

### 9.4 数据迁移

- [ ] 不同 spec 版本间的升级逻辑
- [ ] 废弃字段的清理计划（BaseLayout 中的废弃 filters/sorts/pageSize）
- [ ] 历史数据兼容的完整链路

### 9.5 安全边界

- [ ] 数据库文件的权限控制
- [ ] 关联字段的跨数据库访问边界
- [ ] 模板字段的注入风险评估
- [ ] 资源字段的路径安全检查

---

## 附录：核心文件速查表

### 前端核心文件

| 功能 | 文件路径 |
|------|----------|
| 类型定义 | [app/src/types/index.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/types/index.d.ts) |
| 渲染入口 | [app/src/protyle/render/av/render.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/render.ts) |
| 字段管理 | [app/src/protyle/render/av/col.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/col.ts) |
| 视图管理 | [app/src/protyle/render/av/view.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/view.ts) |
| 单元格渲染 | [app/src/protyle/render/av/cell.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/cell.ts) |
| 过滤逻辑 | [app/src/protyle/render/av/filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/filter.ts) |
| 排序逻辑 | [app/src/protyle/render/av/sort.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/sort.ts) |
| 事件处理 | [app/src/protyle/render/av/action.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/render/av/action.ts) |
| 事务机制 | [app/src/protyle/wysiwyg/transaction.ts](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/app/src/protyle/wysiwyg/transaction.ts) |

### 后端核心文件

| 功能 | 文件路径 |
|------|----------|
| 核心结构 | [kernel/av/av.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/av.go) |
| 值结构 | [kernel/av/value.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/value.go) |
| 过滤逻辑 | [kernel/av/filter.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/filter.go) |
| 排序逻辑 | [kernel/av/sort.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/sort.go) |
| 分组逻辑 | [kernel/av/group.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/group.go) |
| 布局接口 | [kernel/av/layout.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/av/layout.go) |
| API 接口 | [kernel/api/av.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/api/av.go) |
| 业务逻辑 | [kernel/model/attribute_view.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/model/attribute_view.go) |
| 渲染入口 | [kernel/model/attribute_view_render.go](file:///d:/fz/0601/solo-dogfeeding/code/286-siyuan/kernel/model/attribute_view_render.go) |
