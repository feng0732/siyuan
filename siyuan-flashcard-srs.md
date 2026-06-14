# SiYuan 闪卡复习调度：时间边界、状态持久化与一致性风险深度分析

## 一、概述

本文档深入分析 SiYuan 闪卡系统的 **时间边界处理**、**状态持久化细节**、**每日配额与缓存协作关系**、**块属性与卡包数据的一致性风险**，以及 **用户操作反馈的边界场景**。分析基于实际代码执行路径，覆盖前后端完整链路。

### 核心代码定位

| 模块 | 文件 | 关键行数 |
|------|------|----------|
| 调度算法与配额控制 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1093-L1197) | `getDeckDueCards` L1093-L1197 |
| 复习评分与撤销缓存 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L467-L525) | `ReviewFlashcard` L467-L525 |
| 制卡事务（块属性+卡包） | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949) | `doAddFlashcards` L857-L949 |
| 删卡事务（块属性+卡包） | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L747-L855) | `doRemoveFlashcards` L747-L855 |
| 事务提交与回滚 | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1856-L1902) | `begin/commit/rollback` L1856-L1902 |
| 前端已复习列表传递 | [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L715-L773) | 评分请求 L715-L773 |
| API 层 reviewedCards 解析 | [riff.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/api/riff.go#L285-L297) | `getReviewedCards` L285-L297 |

---

## 二、时间边界与调度计算细节

### 2.1 到期卡片筛选（Dues 计算）

#### 2.1.1 `getDeckDueCards` 核心执行流程

函数签名：
```go
func getDeckDueCards(deck *riff.Deck, reviewedCardIDs, blockIDs []string,
    newCardLimit, reviewCardLimit, reviewMode int) (ret []riff.Card,
    unreviewedCount, unreviewedNewCardCountInRound, unreviewedOldCardCountInRound int)
```

**完整执行时序**（L1093-L1197）：

| 步骤 | 操作 | 代码位置 | 说明 |
|------|------|----------|------|
| 1 | `deck.Dues()` 获取全部到期卡片 | L1097 | riff 库内部按 FSRS 算法计算 due ≤ now 的卡片 |
| 2 | 按 blockIDs 过滤（文档/笔记本级复习） | L1100-L1105 | 非全局复习时只返回指定范围内块 |
| 3 | `ExistBlockTrees` 验证块存在性 | L1111-L1117 | **关键防御**：过滤已删除的块，避免悬挂卡片 |
| 4 | 检查 reviewedCardIDs 是否为空 | L1119-L1124 | 空 → 新回合开始，清空撤销/跳过缓存 |
| 5 | 从 `reviewCardCache` 统计已消耗配额 | L1126-L1134 | **关键逻辑**：从缓存而非参数统计已用新卡/复习卡数 |
| 6 | 遍历 dues，按配额选择卡片 | L1136-L1184 | 跳过缓存中的卡片，配额满后停止累加 |
| 7 | 按 reviewMode 排序新卡/旧卡 | L1186-L1195 | 0=混合 1=新卡优先 2=旧卡优先 |

#### 2.1.2 时间边界关键问题

**Q1: FSRS due 时间的时区与精度**

- riff 库内部 `due` 字段为 `time.Time` 类型，精度为秒级
- `deck.Dues()` 返回 `due ≤ time.Now()` 的所有卡片
- **跨天边界**：由于配额不按日历天重置（见 2.2），跨天不会导致配额清零
- **时区风险**：所有时间使用服务器本地时区，客户端与服务端时区不一致时，同一天到期的卡片数量在不同时区可能显示不同

**Q2: 块存在性校验的性能影响**

```go
checkResult := treenode.ExistBlockTrees(toCheckBlockIDs)  // L1111
```
- 每次获取到期卡片都会**批量校验块存在性**
- 卡片量大时（>1000），此步骤可能成为性能瓶颈
- 已删除块对应的卡片不会从卡包中移除，仅在调度时被临时过滤

---

### 2.2 每日新卡/复习上限与已复习列表、缓存的协作关系

这是系统最复杂的协作逻辑之一，涉及**前端 reviewedCards 参数**、**后端 reviewCardCache**、**跳过缓存 skipCardCache** 三方联动。

#### 2.2.1 三方数据结构定义

| 数据项 | 位置 | 结构 | 生命周期 |
|--------|------|------|----------|
| `reviewedCards` 参数 | 前端 → 后端 | `ICard[]`（完整卡片数组） | 每次 API 请求携带，前端 `options.cardsData.cards` |
| `reviewCardCache` | 后端全局变量 | `map[string]riff.Card` | 新回合开始（reviewedCardIDs 为空）或全部复习完成时清空 |
| `skipCardCache` | 后端全局变量 | `map[string]riff.Card` | 同上 |
| `Conf.Flashcard.NewCardLimit` | 配置 | int，默认 20 | 全局默认，可被文档级 IAL 覆盖 |
| `Conf.Flashcard.ReviewCardLimit` | 配置 | int，默认 200 | 同上 |

#### 2.2.2 配额统计的真相：从缓存统计，不从参数统计

**关键代码 L1126-L1134**：
```go
newCount := 0
reviewCount := 0
for _, reviewedCard := range reviewCardCache {
    if riff.New == reviewedCard.GetState() {
        newCount++
    } else {
        reviewCount++
    }
}
```

> **⚠️ 核心发现**：配额统计**完全依赖后端缓存** `reviewCardCache`，而非前端传入的 `reviewedCardIDs`。
>
> `reviewedCardIDs` 的唯一作用是：
> 1. **判断是否为新回合**（空 → 清空缓存）
> 2. **计算 unreviewedCount**（跳过已在列表中的卡片）

#### 2.2.3 前端 reviewedCards 的传递链

**前端构建** [openCard.ts#L720](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L720)：
```typescript
fetchPost("/api/riff/reviewRiffCard", {
    deckID: currentCard.deckID,
    cardID: currentCard.cardID,
    rating: parseInt(type),
    reviewedCards: options.cardsData.cards  // 当前回合的全部卡片列表
}, ...)
```

**后端解析** [riff.go#L285-L297](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/api/riff.go#L285-L297)：
```go
func getReviewedCards(arg map[string]any) (ret []string) {
    reviewedCardsArg := arg["reviewedCards"].([]any)
    for _, card := range reviewedCardsArg {
        c := card.(map[string]any)
        cardID := c["cardID"].(string)
        ret = append(ret, cardID)  // 只提取 cardID
    }
    return
}
```

**结论**：前端每次请求都携带**当前回合的全部卡片**（不仅仅是已复习的），后端仅提取 cardID 用于判断。

#### 2.2.4 配额控制完整协作流程

以新卡上限 = 20 为例：

```
时间轴 ──────────────────────────────────────────────────────────►

[回合开始]
  前端: 首次调用 getRiffDueCards(reviewedCards=[])
    ↓
  后端 getDeckDueCards:
    L1120: reviewedCardIDs 为空 → 清空 reviewCardCache 和 skipCardCache
    L1126: newCount = reviewCardCache 中 New 状态 = 0
    L1127: reviewCount = reviewCardCache 中非 New 状态 = 0
    L1167-L1173: 选出 20 张 New 卡后停止，放入 retNew
  返回: 20 新卡 + 200 复习卡（如有的话）

[第 1-10 张卡复习]
  前端: 每复习一张，调用 reviewRiffCard(reviewedCards=[全部220张卡片])
    ↓
  后端 ReviewFlashcard:
    L488: 首次复习 → reviewCardCache[cardID] = card.Clone()  // 缓存原始状态
    L491: deck.Review() 变更卡片状态和 due
    L492: deck.Save() 持久化
  此时 reviewCardCache 增长到 10 张

[获取下一轮卡片（翻页）]
  前端: index 越界 → 再次调用 getRiffDueCards(reviewedCards=[全部220张卡片])
    ↓
  后端 getDeckDueCards:
    L1120: reviewedCardIDs 非空 → 不清空缓存 ✓
    L1126: newCount = reviewCardCache 中 New 状态卡数量
           注意：New 卡复习后状态会变 Learning/Review，
                 所以 newCount 此时可能 < 10
    L1167-L1173: 继续补满 20 张新卡（如 dues 中还有）
  返回: 补充后的卡片列表

[全部复习完成]
  后端 ReviewFlashcard L502-L507:
    unreviewedCount == 0 → 清空 reviewCardCache 和 skipCardCache
```

#### 2.2.5 文档级配额覆盖机制

代码位置：[GetTreeDueFlashcards L608-L651](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L608-L651)

```go
ial := sql.GetBlockAttrs(rootID)
if newCardLimitStr := ial["custom-riff-new-card-limit"]; "" != newCardLimitStr {
    newCardLimit, _ = strconv.Atoi(newCardLimitStr)
}
if reviewCardLimitStr := ial["custom-riff-review-card-limit"]; "" != reviewCardLimitStr {
    reviewCardLimit, _ = strconv.Atoi(reviewCardLimitStr)
}
```

**文档级 IAL 属性**：
- `custom-riff-new-card-limit`：文档级新卡上限
- `custom-riff-review-card-limit`：文档级复习卡上限

**⚠️ 注意**：文档级配额**仅在文档级复习入口生效**（`GetTreeDueFlashcards`），全局复习入口（Alt+0）使用全局配置，不读取文档 IAL。

---

## 三、状态持久化细节

### 3.1 复习评分的持久化时序

#### 3.1.1 `ReviewFlashcard` 完整执行时序

代码位置：[flashcard.go#L467-L509](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L467-L509)

| 步骤 | 操作 | 代码行 | 持久化点 | 失败后果 |
|------|------|--------|----------|----------|
| 1 | `deckLock.Lock()` | L468 | - | 阻塞等待 |
| 2 | `waitForSyncingStorages()` | L471 | - | 同步中阻塞 |
| 3 | **缓存检查与恢复**（撤销逻辑核心） | L479-L489 | 内存 | 仅影响撤销功能 |
| 3.1 | 命中缓存 → 恢复缓存状态 | L482 | `deck.SetCard(cachedCard)` | 后续评分基于原始状态重算 |
| 3.2 | 未命中 → 克隆入缓存 | L488 | `reviewCardCache[cardID] = card.Clone()` | 下次撤销无原始状态 |
| 4 | `deck.Review(cardID, rating)` | L491 | 内存（riff 内部） | 评分未写入，内存状态不变 |
| 5 | **`deck.Save()`** | L492 | ✅ **写入 .deck 文件** | 卡包状态丢失（仅内存） |
| 6 | **`deck.SaveLog(log)`** | L497 | ✅ **写入复习日志文件** | 日志丢失（不影响调度） |
| 7 | 检查是否全部完成 → 清空缓存 | L502-L507 | 内存 | 缓存残留不影响数据正确性 |

#### 3.1.2 撤销机制的工作原理

**撤销触发链**：
1. 用户按 `p` 或 `q` 键 → 前端 `index--` → 显示上一张卡片
2. 用户对同一张卡再次评分（1/2/3/4）→ 发送 `reviewRiffCard` 请求
3. **关键**：由于同一张 cardID 再次被评分，后端 L479 命中 `reviewCardCache`

**命中缓存后的恢复逻辑** L480-L485：
```go
// 将缓存的卡片重新覆盖回卡包中，以恢复最开始复习前的状态
deck.SetCard(cachedCard)
// 从跳过缓存中移除（如果上一次点的是跳过的话）
delete(skipCardCache, cardID)
```

恢复后再次执行 `deck.Review()`，相当于**从初始状态重新应用新的评分**。

> **⚠️ 设计要点**：
> - 撤销不需要专门的"撤销 API"，通过对同一张卡重复评分隐式触发
> - 每次评分前都会先恢复到缓存中的**初始状态**（即第一次评分前的快照），而非"撤销上一步"
> - 因此支持对同一张卡**多次撤销重评**（每次都回到最初状态）

#### 3.1.3 跳过（Skip）的持久化特性

代码位置：[SkipReviewFlashcard L511-L525](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L511-L525)

```go
func SkipReviewFlashcard(deckID, cardID string) (err error) {
    deckLock.Lock()
    defer deckLock.Unlock()
    waitForSyncingStorages()
    deck := Decks[deckID]
    card := deck.GetCard(cardID)
    if nil == card { return }
    skipCardCache[cardID] = card  // 仅内存操作，不持久化！
    return
}
```

**⚠️ 关键发现：跳过操作不进行任何持久化**
- 仅写入 `skipCardCache` 内存 Map
- `deck.Save()` 从未调用
- 卡片状态（due、reps、lapses）完全不变
- **重启后跳过失效**：下次打开复习界面，被跳过的卡片重新出现

### 3.2 存储结构与文件组织

```
data/storage/riff/
├── {deckID}.deck          # 卡包元数据 + 卡片状态 JSON
├── {deckID}.cards         # 卡片附加数据（如复习日志引用）
├── {deckID}_YYYYMMDD.log  # 每日复习日志（FSRS 优化训练用）
└── 20230218211946-2kw8jgx.deck   # 内置快速制卡包
```

**加载流程** [LoadFlashcards L951-L985](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L951-L985)：
- 启动时遍历 `storage/riff/` 目录，读取所有 `.deck` 文件
- 调用 `riff.LoadDeck()` 反序列化 JSON
- 注入 FSRS 参数：`RequestRetention`、`MaximumInterval`、`Weights`

### 3.3 持久化文件的格式

`.deck` 文件由 riff 库序列化，核心字段：

| 字段 | 说明 | 持久化时机 |
|------|------|-----------|
| `Deck.ID / Name` | 卡包标识 | CreateDeck / RenameDeck |
| `Deck.Created / Updated` | 创建/更新时间戳 | 每次 Save |
| `Card.ID` | 卡片 ID（`ast.NewNodeID()` 生成） | AddCard 时 |
| `Card.BlockID` | 关联的文档块 ID | AddCard 时 |
| `Card.State` | New(0)/Learning(1)/Review(2)/Relearning(3) | Review 后 Save |
| `Card.Due` | 下次到期时间 | Review 后 Save |
| `Card.ScheduledDays` | 本次调度间隔天数 | Review 后 Save |
| `Card.Reps` | 累计复习次数 | Review 后 Save |
| `Card.Lapses` | 忘记录（进入 Relearning 次数） | Review 后 Save |
| `Card.Stability / Difficulty` | FSRS 稳定性/难度参数 | Review 后 Save |
| `Card.LastReview` | 上次复习时间 | Review 后 Save |

---

## 四、块属性写入与卡包保存的不一致风险

### 4.1 `doAddFlashcards` 的事务边界分析

代码位置：[flashcard.go#L857-L949](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949)

#### 4.1.1 分步执行与持久化点

```
doAddFlashcards 执行路径：
├─ ① deckLock.Lock() ─────────────────────────────────── 全局互斥锁
├─ ② isSyncingStorages() 检查 ──────────────────────── 同步中直接返回错误
├─ ③ 查找/创建卡包 ──────────────────────── 仅内存操作，不立即 Save
│     └─ 如找不到 → createDeck0() → 内部会 deck.Save() ✅
├─ ④ 遍历 blockIDs，逐个写入块属性 ─────────────────── ⚠️ 文档树事务
│     ├─ loadTree() ──────────────────────── 加载到内存 tx.trees
│     ├─ node.SetIALAttr("custom-riff-decks", val) ─── 内存修改
│     ├─ tx.writeTree(tree) ──────────────── 放入 tx.trees（待 commit）
│     ├─ cache.PutBlockIAL(...) ──────────── 立即更新内存缓存
│     └─ pushBlockAttrs(...) ─────────────── 推入属性同步队列
├─ ⑤ 遍历 blockIDs，逐个添加卡片到 riff
│     └─ deck.AddCard(ast.NewNodeID(), blockID) ─────── 仅内存操作
└─ ⑥ deck.Save() ─────────────────────────── ✅ 卡包持久化（文件写入）
```

#### 4.1.2 不一致风险点矩阵

| 风险场景 | 触发条件 | 后果 | 影响范围 |
|----------|----------|------|----------|
| **R1: 块属性写入成功，卡包 Save 失败** | 步骤④全部成功后，步骤⑥ `deck.Save()` 因磁盘/权限错误返回 | 块有 `custom-riff-decks` 属性，但卡包中无对应卡片 | 显示不一致：制卡标记存在，但复习时找不到卡片 |
| **R2: 部分块属性写入，部分未写入** | 多块制卡时，某块的 loadTree 失败（文件损坏/锁定），循环继续 | 部分块有属性，部分无；卡包中仅对有属性的块（或全部块）AddCard | 部分双向关联断裂 |
| **R3: 事务 rollback 不回滚卡包操作** | `tx.rollback()` 仅清空 `tx.trees` 和 `tx.nodes`，不回滚 `deck.AddCard()` 和 `deck.Save()` | 文档属性被回滚，但卡包已持久化新卡片 | **严重不一致**：卡包中有卡片，但块无属性 |
| **R4: 同步中断导致半成品状态** | 同步中 `isSyncingStorages()` 返回 true，doAddFlashcards 返回 `TxErrCodeDataIsSyncing`，但块属性已写入内存缓存（L924） | cache 中的 IAL 已更新，但文档树和卡包均未持久化 | 重启后恢复一致，但运行期间缓存显示错误 |

#### 4.1.3 R3 深度分析：rollback 无法回滚卡包操作

事务回滚实现 [transaction.go#L1897-L1902](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1897-L1902)：

```go
func (tx *Transaction) rollback() {
    tx.trees, tx.nodes = nil, nil   // 仅丢弃文档树变更
    tx.state.Store(3)
    tx.m.Unlock()
    return
}
```

**闪卡操作在事务中的执行顺序**（transaction.go L183-L341）：
```
事务执行循环 for _, op := range tx.DoOperations:
  switch op.Action:
    case "addFlashcards":
      ret = tx.doAddFlashcards(op)
        └─ 内部：
           ├─ 先修改文档树（块属性）→ 写入 tx.trees
           └─ 后修改卡包 → deck.Save() 立即写入磁盘
    case "removeFlashcards":
      ret = tx.doRemoveFlashcards(op)
        └─ 同上结构
```

**R3 触发路径**：
1. `doAddFlashcards` 执行成功 → 文档树修改在内存，卡包已落盘
2. **后续操作**（如另一个 op）失败 → `tx.rollback()` 被调用
3. `rollback()` 丢弃文档树变更 → 块属性**未写入文档**
4. 但卡包文件**已包含新卡片** → 双向关联断裂

### 4.2 `doRemoveFlashcards` 的事务边界

代码位置：[flashcard.go#L747-L855](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L747-L855)

```go
func (tx *Transaction) doRemoveFlashcards(operation *Operation) (ret *TxErr) {
    deckLock.Lock()
    defer deckLock.Unlock()
    if isSyncingStorages() { /* 返回错误 */ }

    deckID := operation.DeckID
    blockIDs := operation.BlockIDs

    // ① 先移除块属性（文档树事务）
    if err := tx.removeBlocksDeckAttr(blockIDs, deckID); err != nil {
        return &TxErr{...}
    }

    // ② 后移除卡包中的卡片
    if "" == deckID {
        // 从所有卡包中移除
        for _, deck := range Decks {
            removeFlashcardsByBlockIDs(blockIDs, deck)
        }
    } else {
        removeFlashcardsByBlockIDs(blockIDs, Decks[deckID])
    }
    return
}
```

`removeFlashcardsByBlockIDs` [L837-L855](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L837-L855)：
```go
func removeFlashcardsByBlockIDs(blockIDs []string, deck *riff.Deck) {
    cards := deck.GetCardsByBlockIDs(blockIDs)
    for _, card := range cards {
        deck.RemoveCard(card.ID())
    }
    deck.Save()  // ✅ 立即持久化
}
```

删卡操作与制卡**结构相同**，风险点 R1/R2/R3 同样存在。

### 4.3 风险缓解与现有防御机制

| 机制 | 位置 | 作用 |
|------|------|------|
| `deckLock` 全局互斥 | 所有闪卡操作入口 | 防止卡包并发写入 |
| `waitForSyncingStorages()` | 持久化操作前 | 同步期间拒绝写入 |
| `ExistBlockTrees` 校验 | `getDeckDueCards` L1111 | 调度时过滤已删除块的卡片 |
| `GetCardsByBlockID` 去重 | `doAddFlashcards` L935-L938 | 防止同一块重复制卡 |

> **⚠️ 关键缺失**：缺少**主动一致性修复**机制。当不一致发生时（如 R3 场景），系统无自动检测和修复逻辑。需要手动通过以下方式修复：
> 1. 重新对问题块执行一次"移除卡片→添加卡片"操作
> 2. 或直接编辑块的 `custom-riff-decks` 属性

---

## 五、用户操作反馈的边界场景

### 5.1 前端操作状态机

代码位置：[bindCardEvent L239-L776](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L239-L776)

```
┌─────────────┐   空格/-1    ┌──────────────┐   1/2/3/4    ┌───────────┐
│   显示题目   │ ───────────► │   显示答案    │ ───────────► │  下一张卡  │
│ (答案隐藏)   │              │ (四个评分按钮) │              │ (或完成)   │
└──────┬──────┘              └──────┬───────┘              └─────┬─────┘
       │                            │                            │
       │ -2/上一步(仅 index>0)      │ -2/上一步                   │ 全部完成
       ▼                            ▼                            ▼
  上一张卡片                   上一张卡片                  ┌─────────────┐
                                                            │ allDone()   │
                                                            │ 显示无待复习 │
                                                            └──────┬──────┘
                                                                   │
                                                                   │ unreviewedCount>0
                                                                   ▼
                                                            ┌─────────────┐
                                                            │ newRound()  │
                                                            │ 按钮: 继续复习│
                                                            └─────────────┘
```

### 5.2 边界场景详述

#### 5.2.1 撤销重评的边界

| 场景 | 用户操作 | 系统行为 | 预期/实际 |
|------|----------|----------|-----------|
| B1: 跨页撤销 | 复习 30 张 → 翻页 → 按上一步回到第 30 张 → 重评 | **可用**：reviewCardCache 跨请求保留，cardID 命中即可恢复 | ✓ 正确 |
| B2: 全部完成后再撤销 | 复习完最后一张 → `reviewCardCache` 被清空 (L505) → 按上一步 | **不可用**：缓存已清空，恢复原始状态失败 | ⚠️ 边界限制 |
| B3: 跳过的卡片重评 | 跳过卡 → 按上一步 → 重新评分 | **可用**：重评时 `delete(skipCardCache, cardID)` (L485) 移除跳过标记 | ✓ 正确 |
| B4: 多次撤销重评 | 评分为 1 → 撤销 → 评分为 4 → 撤销 → 评分为 3 | **每次都恢复初始状态**：因为 L482 总是用缓存中的克隆（第一次评分前的快照）覆盖 | ✓ 设计如此 |

#### 5.2.2 配额耗尽与翻页边界

代码位置：[openCard.ts#L731-L763](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L731-L763)

```
index 超出 cardsData.cards 长度时:
  ├─ 再次请求 getRiffDueCards(reviewedCards=当前全部卡片)
  ├─ 后端: reviewedCardIDs 非空 → 不清空缓存 → 从缓存统计配额
  │   ├─ 配额未满 + dues 还有卡 → 返回补充卡片 → 继续翻页
  │   ├─ 配额已满 / 无更多 dues → 返回空卡片列表
  │       ├─ unreviewedCount > 0 → 显示 newRound 界面
  │       │   （原因：配额满，或有跳过的卡，或跨天到期的新卡）
  │       └─ unreviewedCount == 0 → 显示 allDone 界面
  └─ 注意: 新请求替换了 options.cardsData，已复习的卡片信息在前端丢失
```

**B5: newRound 与配额的关系**

`newRound` 出现意味着：
- 当前批次卡片已全部评分
- 但后端判断仍有 `unreviewedCount > 0`
- 常见原因：
  1. 配额已满（如 20 张新卡已用完，但 dues 中还有更多新卡明天才能复习）
  2. 有被跳过的卡片（在 skipCardCache 中被排除）
  3. 复习过程中跨天，新卡片刚好到期

用户点击"继续复习"按钮后，前端发送不带 reviewedCards 的新请求，后端清空缓存重新计算配额。

#### 5.2.3 筛选模式切换边界

代码位置：[openCard.ts#L599-L657](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L599-L657)

筛选模式：全部 / 笔记本 / 文档 / 指定卡包

切换筛选模式的行为：
- `fetchNewRound()` 被调用
- **不传 reviewedCards** → 后端清空 reviewCardCache 和 skipCardCache
- **B6: 切换筛选器 = 隐式开始新回合**，已复习的进度（撤销能力）全部丢失

#### 5.2.4 进度显示的边界

代码位置：[genCardCount L30-L52](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L30-L52)

显示格式：`新卡已复习 / 新卡未复习 + 复习卡已复习 / 复习卡未复习`

**B7: 进度统计的不一致风险**
- 前端进度：`genCardCount()` 仅统计**当前批次 cardsData.cards**
- 后端配额：基于 `reviewCardCache` 统计（含翻页前的批次）
- **翻页后进度显示重置**：新批次 cardsData 不含已复习卡片，前端显示"0 / 20"，但后端实际配额已消耗

#### 5.2.5 键盘快捷键冲突边界

| 快捷键 | 动作 | 冲突风险 |
|--------|------|----------|
| `空格` | 显示答案 / 直接 Good (3) | 与编辑器空格输入冲突 |
| `1/j/a` | Again | 与 Vim 模式 j 键冲突 |
| `2/k/s` | Hard | 与 Vim 模式 k/s 键冲突 |
| `3/l/d` | Good | 与 Vim 模式 l/d 键冲突 |
| `4/;/f` | Easy | 与 Vim 模式 f 键冲突 |
| `p/q` | 上一步（撤销） | 无明显冲突 |
| `0/x` | 跳过 | 无明显冲突 |

### 5.3 完成态与本地持久化

前端仅持久化以下状态到 localStorage（非复习进度）：

| 存储项 | 键 | 说明 |
|--------|-----|------|
| 全屏状态 | `LOCAL_FLASHCARD.fullscreen` | [openCard.ts#L249/L342](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L249) |
| 窗口/标签参数 | `customModelData` | 包含 cardsData、index、cardType |

**⚠️ B8: 复习状态在前端不持久化**
- 刷新页面或关闭窗口后，当前复习进度（index、当前批次卡片列表）全部丢失
- 但后端已保存的卡片状态（due、reps 等）不受影响
- 重新打开后从新的批次开始复习（未复习的卡片会重新出现）

---

## 六、风险评估汇总

### 6.1 数据一致性风险

| ID | 风险 | 严重度 | 触发概率 | 影响范围 | 现有防御 |
|----|------|--------|----------|----------|----------|
| R1 | 块属性写入成功但卡包 Save 失败 | 中 | 低（磁盘异常） | 局部制卡不一致 | 日志告警 |
| R2 | 多块批量制卡部分失败 | 中 | 中（个别块损坏） | 部分块关联断裂 | 循环内 continue |
| R3 | 事务 rollback 不回滚卡包操作 | **高** | 低（后续操作失败） | **全局不一致** | 无直接防御 |
| R4 | 同步中断导致缓存不一致 | 低 | 中 | 运行期显示异常 | 重启后自愈 |
| R5 | 跨设备同步后卡包与块属性版本不匹配 | 中 | 中（多端同时编辑） | 部分卡片重复/丢失 | 时间戳 + 先到先胜 |

### 6.2 时间边界风险

| ID | 风险 | 说明 |
|----|------|------|
| T1 | 时区不一致导致跨天到期偏差 | 服务器与客户端时区不同时，due 计算基准不一致 |
| T2 | 配额不按日历天重置，"每日"概念模糊 | 配额基于缓存回合，非自然日；关闭窗口后配额立即重置 |
| T3 | 长时复习跨天到期卡片插入 | 复习过程中（未关闭窗口），时钟过 0 点，新卡片不会自动进入待复习列表 |
| T4 | 文档级配额仅文档入口生效 | Alt+0 全局入口不读取文档 IAL，配额不同 |

### 6.3 操作反馈风险

| ID | 风险 | 说明 |
|----|------|------|
| U1 | 全部完成后无法撤销 | L505 清空缓存，最后一张卡无法重评 |
| U2 | 切换筛选器丢失撤销能力 | 隐式开始新回合 |
| U3 | 进度显示翻页后重置 | 前端统计与后端配额不同步 |
| U4 | 跳过操作重启后失效 | 仅内存缓存，不持久化 |
| U5 | 刷新丢失当前批次 | 前端不保存复习进度 |

---

## 七、后续验证要点

### 7.1 一致性验证（核心）

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V1 | R3 场景复现 | 构造事务包含 addFlashcards + 后续失败操作 | 检查块属性与卡包是否一致 |
| V2 | R1 场景复现 | 模拟 deck.Save() 失败（权限/磁盘） | 检查块是否有孤立的 custom-riff-decks 属性 |
| V3 | 多端同步一致性 | A 端制卡 → 同步 → B 端检查 | 块属性与卡包数据均存在且一致 |
| V4 | 批量制卡原子性 | 100 块制卡，中途注入块损坏 | 验证成功块的双向关联完整性 |
| V5 | 删除文档块联动 | 删除有闪卡的块 → 复习 → 调度 | ExistBlockTrees 正确过滤，无崩溃 |

### 7.2 时间与配额验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V6 | 配额统计正确性 | 复习 N 张新卡 → 翻页 → 获取新批次 | 新卡补充不超过 20-N |
| V7 | 新回合配额重置 | 关闭复习窗口 → 重新打开 | 配额从 0 重新统计 |
| V8 | 跨时区到期 | 客户端与服务端时区差 8h | 到期卡片数量在两端可解释（不要求一致） |
| V9 | 跨天到期不刷新 | 打开复习 → 等过 0 点 → 检查列表 | 不主动刷新（当前设计） |
| V10 | 文档级配额 | 文档设置 custom-riff-new-card-limit=5 → 文档级复习 | 最多返回 5 张新卡 |
| V11 | 全局入口忽略文档级配额 | 同上文档 → Alt+0 全局复习 | 返回全局配额 20 张 |

### 7.3 撤销与跳过验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V12 | 单次撤销重评 | 评 Again → 撤销 → 评 Easy | 最终 due 与首次评 Easy 一致 |
| V13 | 跨批次撤销 | 复习完第一页 → 翻页 → 上一步 → 重评第一页卡片 | 状态正确恢复 |
| V14 | 多次撤销重评 | 连续撤销重评 3 次 | 每次都恢复初始状态 |
| V15 | 跳过 → 撤销 → 评分 | 跳过 → 上一步 → 评分 Good | skipCardCache 被正确移除，状态更新 |
| V16 | 全部完成后撤销尝试 | 复习完最后一张 → 上一步 | 尝试撤销（边界，当前设计不支持） |
| V17 | 跳过重启后失效 | 跳过 3 张 → 重启 SiYuan → 打开复习 | 3 张卡片重新出现在列表中 |

### 7.4 用户操作边界验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V18 | 筛选器切换后撤销 | 复习 5 张 → 切换筛选器 → 上一步 | 无崩溃（撤销能力丢失是预期） |
| V19 | 快速连续评分 | 连续按 3 键 10 次/秒 | 后端 deckLock 排队，不丢失评分 |
| V20 | 同步期间评分 | 评分操作与云同步并行 | waitForSyncingStorages 正确等待或返回 |
| V21 | 新回合按钮交互 | 完成批次 → 显示 newRound → 点击继续 | 清空缓存，开始新回合 |
| V22 | 窗口刷新恢复 | 复习中途刷新页面 | 已复习卡片状态持久化，未复习重新排队 |

---

## 八、关键代码索引

### 8.1 后端核心文件

| 文件 | 关键函数/结构 | 行号 |
|------|--------------|------|
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `getDeckDueCards` | L1093-L1197 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `ReviewFlashcard` | L467-L509 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `SkipReviewFlashcard` | L511-L525 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `doAddFlashcards` | L857-L949 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `doRemoveFlashcards` | L747-L771 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `removeBlocksDeckAttr` | L773-L835 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `removeFlashcardsByBlockIDs` | L837-L855 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `GetTreeDueFlashcards`（文档级配额） | L608-L651 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `LoadFlashcards` | L951-L985 |
| [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go) | `getAllDueFlashcards`（全局仅内置卡包） | L724-L745 |
| [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go) | `begin / commit / rollback` | L1856-L1902 |
| [riff.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/api/riff.go) | `getReviewedCards`（参数解析） | L285-L297 |

### 8.2 前端核心文件

| 文件 | 关键函数/逻辑 | 行号 |
|------|--------------|------|
| [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | `genCardCount`（进度显示） | L30-L52 |
| [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | `bindCardEvent`（全部交互） | L239-L776 |
| [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | 评分请求（reviewedCards 传递） | L715-L773 |
| [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | `nextCard` | L860-L882 |
| [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | `allDone` | L884-L895 |
| [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | `newRound` | L897-L908 |
| [openCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts) | 筛选器切换逻辑 | L599-L657 |
| [makeCard.ts](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/makeCard.ts) | `quickMakeCard`（快速制卡） | L178-L225 |

### 8.3 关键全局变量

| 变量 | 类型 | 作用 | 生命周期 |
|------|------|------|----------|
| `Decks` | `map[string]*riff.Deck` | 全部已加载卡包 | 进程生命周期，LoadFlashcards 填充 |
| `deckLock` | `sync.Mutex` | 闪卡操作全局互斥 | 进程生命周期 |
| `reviewCardCache` | `map[string]riff.Card` | 撤销缓存 | 回合开始/结束清空 |
| `skipCardCache` | `map[string]riff.Card` | 跳过缓存 | 回合开始/结束清空 |
| `Conf.Flashcard` | `conf.Flashcard` | 全局配置 | 启动时加载，配置面板修改 |
