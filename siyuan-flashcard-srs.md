# SiYuan 闪卡与间隔重复算法代码分析

## 一、概述

SiYuan（思源笔记）的闪卡系统基于 **FSRS（Free Spaced Repetition Scheduler）** 算法实现，采用独立的 `riff` 库作为卡包管理层，结合文档块属性（IAL）实现卡片与文档内容的双向关联。系统支持手动制卡、快速制卡、多卡包管理、复习调度、状态持久化等核心功能。

### 核心依赖

| 依赖库 | 作用 | 位置 |
|--------|------|------|
| `github.com/siyuan-note/riff` | 卡包管理与卡片抽象层 | [go.mod](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/go.mod#L72) |
| `github.com/open-spaced-repetition/go-fsrs/v3` | FSRS v3 间隔重复算法 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L32) |

---

## 二、架构与流程说明

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                        前端 (app/src)                         │
│  ┌────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ makeCard.ts│  │openCard.ts│  │viewCards.ts│ │flashcard.ts│ │
│  │  制卡交互   │  │ 复习交互  │  │ 卡片预览  │  │ 配置面板  │  │
│  └──────┬─────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘  │
└─────────┼───────────────┼──────────────┼──────────────┼───────┘
          │               │              │              │
          ▼               ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────┐
│                      API 层 (kernel/api)                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                     riff.go                            │  │
│  └───────────────────────┬────────────────────────────────┘  │
└──────────────────────────┼───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                    模型层 (kernel/model)                       │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                  flashcard.go                          │  │
│  │  卡片CRUD、复习调度、状态管理、事务操作                 │  │
│  └───────────────────────┬────────────────────────────────┘  │
└──────────────────────────┼───────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          ▼                                 ▼
┌──────────────────────┐        ┌──────────────────────┐
│   riff 库 (外部)      │        │  文档树 / SQL / 缓存  │
│  ┌────────────────┐  │        │  ┌────────────────┐  │
│  │ Deck (卡包)    │  │        │  │ 块 IAL 属性     │  │
│  │ Card (卡片)    │  │        │  │ custom-riff-    │  │
│  │ FSRS 调度      │  │        │  │ decks           │  │
│  └────────────────┘  │        │  └────────────────┘  │
└──────────────────────┘        └──────────────────────┘
```

### 2.2 卡片生成流程

#### 2.2.1 手动制卡流程

1. **用户触发**：选中块后点击制卡按钮或使用快捷键 `⌥⌘F`（快速制卡）
2. **前端请求**：调用 `/api/riff/addRiffCards` 接口，传入 `deckID` 和 `blockIDs`
3. **事务提交**：操作通过事务队列 `txQueue` 异步执行
4. **属性写入**：在块的 IAL（Inline Attribute List）中写入 `custom-riff-decks` 属性
5. **卡片创建**：调用 `deck.AddCard(cardID, blockID)` 创建卡片
6. **持久化保存**：调用 `deck.Save()` 将卡包数据写入文件

**关键代码**：
- 前端快速制卡：[quickMakeCard](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/makeCard.ts#L178-L225)
- 后端添加卡片：[doAddFlashcards](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949)

#### 2.2.2 卡片与文档的关联机制

卡片与文档块通过 **双向关联** 实现：

| 方向 | 存储位置 | 字段 | 说明 |
|------|----------|------|------|
| 块 → 卡包 | 文档 IAL 属性 | `custom-riff-decks` | 逗号分隔的卡包 ID 列表 |
| 卡包 → 块 | riff 卡包文件 | `blockID` | 每个卡片关联一个块 ID |

**关键常量**：
- 属性名：`NodeAttrRiffDecks = "custom-riff-decks"` [flashcard.go#L1200](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1200-L1201)
- 内置卡包 ID：`builtinDeckID = "20230218211946-2kw8jgx"` [flashcard.go#L987](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L987-L988)
- 前端常量：`QUICK_DECK_ID` [constants.ts#L326](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/constants.ts#L326-L327)

#### 2.2.3 制卡类型与答案隐藏

系统支持多种制卡类型，在复习时通过 CSS 隐藏答案部分：

| 类型 | 配置项 | 隐藏方式 | 样式类 |
|------|--------|----------|--------|
| 标记制卡 | `mark` | 隐藏标记内容 | `card__block--hidemark` |
| 列表制卡 | `list` | 隐藏子列表 | `card__block--hideli` |
| 超级块制卡 | `superBlock` | 隐藏超级块后续内容 | `card__block--hidesb` |
| 标题制卡 | `heading` | 隐藏标题后同级内容 | `card__block--hideh` |

**相关样式**：[_card.scss](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/assets/scss/business/_card.scss#L90-L101)

### 2.3 复习调度流程

#### 2.3.1 获取待复习卡片

```
用户打开复习界面
    ↓
调用 getDueFlashcards(deckID, reviewedCardIDs)
    ↓
调用 getDeckDueCards() 进行筛选
    ├─ 从 deck.Dues() 获取所有到期卡片
    ├─ 校验块是否存在（ExistBlockTrees）
    ├─ 过滤已跳过卡片（skipCardCache）
    ├─ 按新卡/复习卡分别计数
    ├─ 应用新卡上限（NewCardLimit）
    └─ 应用复习卡上限（ReviewCardLimit）
    ↓
按复习模式排序（混合/新卡优先/旧卡优先）
    ↓
返回 Flashcard 列表（含 nextDues 预测）
```

**关键函数**：
- [getDeckDueCards](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1093-L1197) - 核心调度逻辑
- [GetDueFlashcards](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L688-L701) - 对外接口

#### 2.3.2 复习评分流程

```
用户点击评分按钮（Again/Hard/Good/Easy）
    ↓
调用 ReviewFlashcard(deckID, cardID, rating, reviewedCardIDs)
    ├─ 检查 reviewCardCache（撤销支持）
    │   ├─ 命中：恢复缓存状态（撤销后重学）
    │   └─ 未命中：缓存当前状态
    ├─ 调用 deck.Review(cardID, rating) 执行 FSRS 算法
    ├─ 调用 deck.Save() 持久化卡包
    ├─ 调用 deck.SaveLog(log) 保存复习日志
    └─ 检查是否所有卡片复习完毕，清空缓存
    ↓
返回成功响应
```

**关键函数**：[ReviewFlashcard](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L467-L509)

#### 2.3.3 FSRS 算法参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `requestRetention` | 0.9（来自 fsrs.DefaultParam） | 请求保持率 |
| `maximumInterval` | 36500 天 | 最大间隔 |
| `weights` | fsrs.DefaultWeights() | 19 个权重值，逗号分隔字符串 |

**配置结构**：[Flashcard (conf)](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/conf/flashcard.go#L26-L40)

#### 2.3.4 卡片状态

FSRS 定义了四种卡片状态（`riff.State`）：

| 状态 | 数值 | 说明 |
|------|------|------|
| New | 0 | 新卡，从未学习过 |
| Learning | 1 | 学习中 |
| Review | 2 | 复习中（已进入长期记忆） |
| Relearning | 3 | 重新学习（遗忘后重新学习） |

**前端状态字段**：[ICard.state](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/types/index.d.ts#L363-L372)

### 2.4 持久化机制

#### 2.4.1 存储结构

卡包数据存储在工作区的 `storage/riff/` 目录下：

```
storage/riff/
├── {deckID}.deck      # 卡包元数据
└── {deckID}.cards     # 卡片数据
```

**存储路径函数**：[getRiffDir](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1082-L1084)

#### 2.4.2 加载流程

系统启动时调用 `LoadFlashcards()` 加载所有卡包：

1. 读取 `storage/riff/` 目录下所有 `.deck` 文件
2. 调用 `riff.LoadDeck()` 加载卡包和卡片
3. 传入 FSRS 参数（requestRetention, maximumInterval, weights）
4. 将卡包存入内存映射 `Decks map[string]*riff.Deck`

**加载函数**：[LoadFlashcards](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L951-L985)

#### 2.4.3 保存时机

- 每次复习后：`deck.Save()` + `deck.SaveLog(log)`
- 添加/移除卡片后：`deck.Save()`
- 重命名卡包后：`deck.Save()`

---

## 三、模块协作关系

### 3.1 核心模块交互

| 模块 | 文件 | 职责 |
|------|------|------|
| 制卡模块 | [makeCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/makeCard.ts) | 制卡对话框、卡包管理、快速制卡 |
| 复习模块 | [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | 复习界面、卡片渲染、用户交互 |
| 预览模块 | [viewCards.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/viewCards.ts) | 卡片列表预览、分页、重置 |
| 配置模块 | [flashcard.ts (config)](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/config/flashcard.ts) | 闪卡设置面板 |
| API 层 | [riff.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/api/riff.go) | REST API 接口 |
| 模型层 | [flashcard.go (model)](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | 业务逻辑、事务处理 |
| 配置层 | [flashcard.go (conf)](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/conf/flashcard.go) | 配置结构与默认值 |

### 3.2 数据结构关系

#### 3.2.1 前端数据结构

```typescript
interface ICard {
    deckID: string;        // 卡包 ID
    cardID: string;        // 卡片 ID
    blockID: string;       // 关联块 ID
    nextDues: IObject;     // 各评级的下次到期时间（预测）
    lapses: number;        // 遗忘次数
    lastReview: number;    // 最后复习时间（时间戳）
    reps: number;          // 复习次数
    state: number;         // 卡片状态（0=新卡）
}

interface ICardData {
    cards: ICard[];                    // 卡片列表
    unreviewedCount: number;           // 未复习总数
    unreviewedNewCardCount: number;    // 未复习新卡数
    unreviewedOldCardCount: number;    // 未复习旧卡数
}
```

**定义位置**：[index.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/types/index.d.ts#L356-L379)

#### 3.2.2 后端数据结构

```go
type Flashcard struct {
    DeckID     string                 // 卡包 ID
    CardID     string                 // 卡片 ID
    BlockID    string                 // 块 ID
    Lapses     int                    // 遗忘次数
    Reps       int                    // 复习次数
    State      riff.State             // 卡片状态
    LastReview int64                  // 上次复习时间
    NextDues   map[riff.Rating]string // 预测下次到期时间
}

type RiffCard struct {
    Due        time.Time  // 下次到期时间
    Reps       uint64     // 复习次数
    Lapses     uint64     // 遗忘次数
    State      fsrs.State // 卡片状态
    LastReview time.Time  // 上次复习时间
}
```

**定义位置**：
- [Flashcard](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L527-L536)
- [RiffCard](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/block.go#L77-L81)

### 3.3 事务协作

闪卡操作（添加/移除）通过 SiYuan 的事务系统执行：

1. **事务提交**：`PerformTransactions()` 将事务放入 `txQueue` 通道
2. **异步执行**：`flushQueue()` 协程消费队列，调用 `performTx()`
3. **操作分发**：根据 `Action` 字段调用对应的 `do*` 方法
4. **双向更新**：同时更新块 IAL 属性和 riff 卡包数据

**事务操作类型**：
- `addFlashcards` - 添加闪卡
- `removeFlashcards` - 移除闪卡

**相关代码**：
- 事务队列：[transaction.go#L64-L132](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L64-L132)
- 添加闪卡事务：[doAddFlashcards](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949)
- 移除闪卡事务：[doRemoveFlashcards](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L747-L771)

### 3.4 并发控制

系统使用 `deckLock sync.Mutex` 保护卡包的并发访问：

- 所有卡包读写操作都需要先获取 `deckLock`
- 复习缓存（`reviewCardCache`、`skipCardCache`）在锁内操作
- 同步期间拒绝写入操作（返回 `TxErrCodeDataIsSyncing`）

---

## 四、时间边界与状态管理

### 4.1 到期时间计算

FSRS 算法根据用户评分计算下次复习时间：

| 评分 | 对应值 | 效果 |
|------|--------|------|
| Again | 1 | 忘记，重置为学习状态，间隔大幅缩短 |
| Hard | 2 | 困难，间隔小幅增加 |
| Good | 3 | 良好，正常间隔增长 |
| Easy | 4 | 简单，间隔大幅增长 |

**预测下次到期时间**：`card.NextDues()` 返回四种评级对应的预测到期时间

### 4.2 新卡与复习卡限制

系统按日限制新卡和复习卡数量：

- **新卡上限**（`NewCardLimit`）：默认 20 张/天
- **复习卡上限**（`ReviewCardLimit`）：默认 200 张/天
- **文档级限制**：可通过块属性 `custom-riff-new-card-limit` 和 `custom-riff-review-card-limit` 单独设置

**文档级限制代码**：[GetTreeDueFlashcards](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L608-L651)

### 4.3 复习模式

| 模式 | 值 | 说明 |
|------|----|------|
| 新旧混合 | 0 | 默认，按到期时间排序 |
| 新卡优先 | 1 | 先复习新卡，再复习旧卡 |
| 旧卡优先 | 2 | 先复习旧卡，再复习新卡 |

**模式排序代码**：[getDeckDueCards#L1186-L1195](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1186-L1195)

### 4.4 撤销与跳过机制

#### 4.4.1 撤销机制

系统使用 `reviewCardCache` 缓存已复习的卡片状态以支持撤销：

1. 首次复习时缓存卡片原始状态
2. 撤销后再次复习时，从缓存恢复状态
3. 所有卡片复习完毕后清空缓存

**关键代码**：[ReviewFlashcard#L479-L489](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L479-L489)

#### 4.4.2 跳过机制

使用 `skipCardCache` 记录跳过的卡片，在获取待复习卡片时过滤：

1. 用户点击"跳过"时将卡片加入缓存
2. `getDeckDueCards()` 过滤掉已跳过的卡片
3. 复习结束后清空缓存

**关键代码**：
- [SkipReviewFlashcard](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L511-L525)
- [getDeckDueCards#L1136-L1139](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1136-L1139)

---

## 五、批量更新与用户操作反馈

### 5.1 批量操作 API

| 操作 | API 端点 | 说明 |
|------|----------|------|
| 批量设置到期时间 | `/api/riff/batchSetRiffCardsDueTime` | 批量修改卡片到期时间 |
| 批量重置卡片 | `/api/riff/resetRiffCards` | 按笔记本/文档/卡包重置学习进度 |
| 批量添加卡片 | `/api/riff/addRiffCards` | 批量添加块到卡包 |
| 批量移除卡片 | `/api/riff/removeRiffCards` | 批量从卡包移除块 |

#### 5.1.1 批量设置到期时间

输入格式：`YYYYMMDDHHmmss` 格式的时间字符串

**函数**：[SetFlashcardsDueTime](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L81-L114)

#### 5.1.2 批量重置卡片

支持按三种维度重置：
- `notebook` - 整个笔记本
- `tree` - 指定文档及其子文档
- `deck` - 指定卡包

实现方式：先删除再重建卡片（`removeFlashcards` + `addFlashcards`）

**函数**：[ResetFlashcards](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L116-L181)

### 5.2 用户操作反馈

#### 5.2.1 键盘快捷键

| 操作 | 快捷键 |
|------|--------|
| 打开闪卡复习 | `⌃0` (Ctrl+0) |
| 快速制卡 | `⌥⌘F` (Option+Command+F) |
| 显示答案 / Good | `空格` / `回车` |
| Again (忘记) | `1` / `j` / `a` |
| Hard (困难) | `2` / `k` / `s` |
| Good (良好) | `3` / `l` / `d` |
| Easy (简单) | `4` / `;` / `f` |
| 跳过 | `0` / `x` |
| 上一张（撤销） | `p` / `q` |

**快捷键定义**：[constants.ts#L450-L451](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/constants.ts#L450-L451), [constants.ts#L499](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/constants.ts#L499)

#### 5.2.2 进度显示

复习界面实时显示进度：
- 新卡进度：`当前新卡数 / 未复习新卡总数`
- 复习卡进度：`当前复习卡数 / 未复习旧卡总数`
- 未复习总数统计

**生成函数**：[genCardCount](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L30-L52)

#### 5.2.3 完成态

- **全部复习完成**：显示"没有待复习的卡片"
- **达每日上限但还有未复习**：显示"继续复习"按钮，开启新一轮

**相关函数**：
- [allDone](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L884-L895)
- [newRound](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L897-L908)

---

## 六、风险评估

### 6.1 数据一致性风险

| 风险点 | 等级 | 说明 | 缓解措施 |
|--------|------|------|----------|
| 块属性与卡包数据不一致 | 中 | 事务可能部分失败（如 IAL 写入成功但卡包保存失败） | 事务机制保证，单事务内操作 |
| 并发修改冲突 | 低 | 多客户端同时修改同一卡包 | `deckLock` 互斥锁 |
| 同步期间数据丢失 | 中 | 同步过程中进行闪卡操作可能被覆盖 | 同步期间拒绝写入（`isSyncingStorages` 检查） |
| 块删除后残留卡片 | 低 | 块被删除后，卡包中仍有对应卡片 | 获取待复习卡片时校验块是否存在（`ExistBlockTrees`） |

**同步检查代码**：[doAddFlashcards#L861-L864](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L861-L864)

### 6.2 性能风险

| 风险点 | 等级 | 说明 |
|--------|------|------|
| 大量卡片加载慢 | 中 | 卡包文件过大时，启动加载时间长 |
| 频繁磁盘写入 | 低 | 每次复习都调用 `deck.Save()`，可能频繁 IO |
| 复习列表排序 | 低 | 每次获取都排序，卡片数量大时可能有性能问题 |

### 6.3 算法与逻辑风险

| 风险点 | 等级 | 说明 |
|--------|------|------|
| FSRS 参数配置错误 | 中 | 用户修改 weights 参数可能导致算法异常 |
| 时间边界问题 | 低 | 跨时区、跨日期的到期时间计算 |
| 新卡/复习卡计数偏差 | 低 | `reviewCardCache` 和 `skipCardCache` 可能影响计数准确性 |
| 撤销状态不一致 | 低 | 撤销后状态可能与实际持久化状态有偏差 |

### 6.4 安全与兼容性风险

| 风险点 | 等级 | 说明 |
|--------|------|------|
| 卡包文件损坏 | 中 | 文件损坏或格式不兼容可能导致数据丢失 |
| 版本升级兼容性 | 中 | riff 库或 FSRS 算法升级可能导致数据不兼容 |
| 导出/导入数据丢失 | 低 | 复制文档时移除闪卡属性（设计行为，但可能出乎用户意料） |

**导出移除属性**：[export.go#L2667](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/export.go#L2667)
**副本移除属性**：[transaction.go#L1334](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1334)

---

## 七、后续验证要点

### 7.1 功能验证

- [ ] **卡片创建**：验证手动制卡、快速制卡能否正确创建卡片并写入块属性
- [ ] **多卡包**：验证一个块能否加入多个卡包，以及各卡包状态是否独立
- [ ] **复习流程**：验证四种评分（Again/Hard/Good/Easy）的间隔计算是否符合 FSRS 算法
- [ ] **新卡上限**：验证新卡上限能否正确限制每日新卡数量
- [ ] **复习卡上限**：验证复习卡上限能否正确限制每日复习数量
- [ ] **复习模式**：验证三种复习模式（混合/新卡优先/旧卡优先）的排序是否正确
- [ ] **撤销功能**：验证撤销操作能否正确恢复卡片状态
- [ ] **跳过功能**：验证跳过的卡片不会再次出现在当前轮复习中
- [ ] **批量重置**：验证按笔记本/文档/卡包批量重置能否正确清零学习进度
- [ ] **批量设置到期时间**：验证批量修改到期时间是否生效
- [ ] **文档级限制**：验证 `custom-riff-new-card-limit` 等自定义属性能否生效

### 7.2 边界条件验证

- [ ] **跨天复习**：在午夜前后复习，验证新卡/复习卡计数是否正确重置
- [ ] **时区问题**：修改系统时区，验证到期时间计算是否正确
- [ ] **最大间隔**：验证 `maximumInterval` 限制是否生效
- [ ] **零卡片**：验证没有卡片时的界面表现
- [ ] **删除块**：删除关联块后，验证卡包中卡片的处理逻辑
- [ ] **大数量级**：验证 1000+ 卡片的加载和复习性能

### 7.3 持久化与一致性验证

- [ ] **异常退出**：模拟进程崩溃，验证重启后数据是否完整
- [ ] **并发操作**：多窗口同时复习同一卡包，验证数据一致性
- [ ] **同步冲突**：同步过程中进行闪卡操作，验证错误提示和数据保护
- [ ] **导出导入**：验证导出文档后再导入，闪卡状态是否丢失（预期丢失）
- [ ] **复制副本**：验证复制文档后是否正确移除闪卡属性

### 7.4 配置验证

- [ ] **FSRS 参数**：修改 `requestRetention`、`weights` 等参数，验证是否影响调度
- [ ] **制卡类型开关**：分别开关 mark/list/superBlock/heading，验证答案隐藏效果
- [ ] **配置持久化**：修改配置后重启，验证配置是否正确保存

### 7.5 集成验证

- [ ] **插件扩展**：验证插件能否通过 `updateCards` 钩子修改卡片数据
- [ ] **事件总线**：验证 `click-flashcard-action` 事件能否正确触发
- [ ] **移动端适配**：验证移动端界面布局和交互是否正常
- [ ] **键盘操作**：验证所有快捷键是否正常工作

---

## 八、关键代码索引

### 后端核心文件

| 文件 | 主要功能 |
|------|----------|
| [kernel/model/flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | 闪卡核心业务逻辑 |
| [kernel/api/riff.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/api/riff.go) | 闪卡 API 接口 |
| [kernel/conf/flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/conf/flashcard.go) | 闪卡配置结构 |
| [kernel/model/transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go) | 事务处理框架 |

### 前端核心文件

| 文件 | 主要功能 |
|------|----------|
| [app/src/card/makeCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/makeCard.ts) | 制卡与卡包管理界面 |
| [app/src/card/openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | 复习界面与交互逻辑 |
| [app/src/card/viewCards.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/viewCards.ts) | 卡片预览列表 |
| [app/src/card/util.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/util.ts) | 卡片工具函数 |
| [app/src/config/flashcard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/config/flashcard.ts) | 闪卡设置面板 |
| [app/src/types/index.d.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/types/index.d.ts) | TypeScript 类型定义 |
| [app/src/constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/constants.ts) | 常量定义 |

### 样式文件

| 文件 | 主要功能 |
|------|----------|
| [app/src/assets/scss/business/_card.scss](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/assets/scss/business/_card.scss) | 闪卡相关样式 |
| [app/src/assets/scss/protyle/_wysiwyg.scss](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/assets/scss/protyle/_wysiwyg.scss#L69-L71) | 编辑器内闪卡块样式 |

---

*本文档基于 SiYuan v3.6.x 版本代码分析生成*
