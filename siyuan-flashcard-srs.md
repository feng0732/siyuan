# SiYuan 闪卡制作与复习计划代码核对分析

## 一、同步状态检测的触发顺序问题修正

### 1.1 两种同步检测函数的本质区别

SiYuan 闪卡系统中存在**两种不同的同步状态检测策略**，分别用于不同的操作类型，这是之前分析中被混淆的关键点。

| 函数 | 行为 | 适用操作 | 代码位置 |
|------|------|----------|----------|
| `waitForSyncingStorages()` | **循环等待**同步完成，每秒轮询 | 复习类操作（读+写） | [repository.go#L1210-L1213](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1210-L1213) |
| `isSyncingStorages()` | **立即检查**并返回 bool，不等待 | 制卡/删卡类操作（事务写） | [repository.go#L1216-L1217](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1216-L1217) |

同步状态变量定义：
```go
var syncingStorages = atomic.Bool{}  // L1208
var syncingFiles = sync.Map{}        // L1207
var isBootSyncing = atomic.Bool{}    // 启动同步标记

func waitForSyncingStorages() {
    for isSyncingStorages() {
        time.Sleep(time.Second)  // 每秒轮询
    }
}

func isSyncingStorages() bool {
    return syncingStorages.Load() || isBootSyncing.Load()
}
```

### 1.2 触发顺序的实际分布

**复习类操作 - 使用 `waitForSyncingStorages()`（阻塞等待）**：

| 操作 | 调用位置 | 设计意图 |
|------|----------|----------|
| `GetFlashcardsByBlockIDs` | [flashcard.go#L46](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L46) | 确保读到最新数据 |
| `SetFlashcardsDueTime` | [flashcard.go#L87](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L87) | 批量设置到期时间 |
| `ReviewFlashcard` | [flashcard.go#L471](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L471) | 评分持久化 |
| `SkipReviewFlashcard` | [flashcard.go#L515](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L515) | 跳过标记 |
| `GetNotebookDueFlashcards` | [flashcard.go#L560](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L560) | 笔记本级获取 |
| `GetTreeDueFlashcards` | [flashcard.go#L612](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L612) | 文档级获取 |
| `GetDueFlashcards` | [flashcard.go#L692](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L692) | 全局/卡包级获取 |

**制卡/删卡类操作 - 使用 `isSyncingStorages()`（立即失败）**：

| 操作 | 调用位置 | 设计意图 |
|------|----------|----------|
| `doAddFlashcards` | [flashcard.go#L861](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L861) | 同步中直接返回 `TxErrCodeDataIsSyncing` |
| `doRemoveFlashcards` | [flashcard.go#L751](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L751) | 同上 |

### 1.3 触发顺序问题分析

#### ⚠️ 问题 1：同步检测时机在 deckLock 之后

**`doAddFlashcards` 执行顺序** [L857-L864](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L864)：
```go
func (tx *Transaction) doAddFlashcards(operation *Operation) (ret *TxErr) {
    deckLock.Lock()           // ① 先获取全局互斥锁
    defer deckLock.Unlock()

    if isSyncingStorages() {  // ② 后检查同步状态
        ret = &TxErr{code: TxErrCodeDataIsSyncing}
        return
    }
    // ... 后续操作
}
```

**风险**：
- 同步中，所有调用 `doAddFlashcards` 的协程都会先阻塞在 `deckLock.Lock()` 上
- 锁持有期间发现同步中 → 立即释放锁 → 返回错误
- 但此时**事务已经开始**（`tx.begin()` 在事务进入时调用），返回错误会触发 `tx.rollback()`

#### ⚠️ 问题 2：复习操作等待可能导致长时阻塞

**`ReviewFlashcard` 执行顺序** [L467-L472](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L467-L472)：
```go
func ReviewFlashcard(...) (err error) {
    deckLock.Lock()             // ① 先获取锁
    defer deckLock.Unlock()

    waitForSyncingStorages()    // ② 后循环等待（可能等待几秒~几分钟）
    // ... 同步完成后才继续
}
```

**风险**：
- `deckLock` 被一个复习操作持有并等待同步时，**所有其他闪卡操作都会被阻塞**
- 同步时间较长时，用户可能感受到明显卡顿
- 但这是设计使然，目的是确保评分不会写入正在被同步覆盖的文件

---

## 二、半成品缓存风险澄清

### 2.1 `cache.PutBlockIAL` 的调用时序

**`doAddFlashcards` 中块属性写入时序** [L911-L926](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L911-L926)：

```
① oldAttrs := parse.IAL2Map(node.KramdownIAL)
   ↓ 保存原始属性（用于 pushBlockAttrs）

② node.SetIALAttr(NodeAttrRiffDecks, val)
   ↓ 修改内存中的节点属性（仅在 tx.trees 中）

③ tx.writeTree(tree)
   ↓ 将修改后的 tree 标记为待写入（仍在内存）

④ cache.PutBlockIAL(blockID, parse.IAL2Map(node.KramdownIAL))
   ↓ ⚠️ 立即更新内存缓存！

⑤ pushBlockAttrs(oldAttrs, node)
   ↓ 推入属性同步队列（用于搜索索引等）
```

**关键点**：步骤④的 `cache.PutBlockIAL()` 在**事务 commit 之前**就更新了内存缓存。

### 2.2 `cache.PutBlockIAL` 实现

[cache/ial.go#L60-L63](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/cache/ial.go#L60-L63)：
```go
func PutBlockIAL(id string, ial map[string]string) {
    blockIALCache.Set(id, ial, 128)  // 写入 ristretto 缓存
}
```

### 2.3 半成品缓存的实际风险矩阵

| 场景 | 触发条件 | 缓存状态 | 磁盘状态 | 一致性 | 自愈方式 |
|------|----------|----------|----------|--------|----------|
| **C1: 事务正常 commit** | 流程完整执行 | ✅ 已更新 | ✅ 已写入 | 一致 | - |
| **C2: isSyncingStorages() 检测失败** | 同步中，L861 返回错误 | ❌ 未更新 | ❌ 未写入 | 一致 | - |
| **C3: 制卡中途某块 loadTree 失败** | 多块制卡时某块损坏，L901 `nil == tree`，`continue` | ❌ 该块未更新 | ❌ 该块未写入 | 部分块跳过，但一致 | 该块未制卡，下次可重试 |
| **C4: deck.Save() 失败** | L944 磁盘写入错误（权限/空间） | ✅ 缓存已更新 | ❌ 卡包未写入 | **不一致** | 块显示制卡标记，但复习找不到卡片 |
| **C5: 事务后续 op 失败触发 rollback** | 同一事务中后续操作失败，调用 `tx.rollback()` | ✅ 缓存已更新 | ❌ 文档树未写入 | **严重不一致** | 块属性缓存有标记，但磁盘文档无属性，卡包可能已写入 |
| **C6: 进程崩溃在 commit 之前** | L924 之后、`tx.commit()` 之前崩溃 | ✅ 缓存已更新（重启后丢失） | ❌ 均未写入 | 重启后一致 | 缓存是内存的，进程重启自动恢复 |

### 2.4 风险发生 vs 不发生的明确结论

#### ✅ 不会发生的风险

| ID | 之前假设 | 实际不会发生的原因 |
|----|----------|-------------------|
| ❌ R1_old | "块属性写入成功但卡包 Save 失败导致双向不一致" | 块属性**没有**在 L924 写入磁盘，只是更新了缓存。磁盘上块属性和卡包都未写入。不一致只存在于缓存层。 |
| ❌ R4_old | "同步中断导致缓存不一致且无法自愈" | 缓存是内存态，重启自动清零。运行期间的不一致仅影响显示，下次读取块时会从磁盘重新加载。 |
| ❌ "cache.PutBlockIAL 后崩溃导致数据丢失" | 缓存是内存的，崩溃即失。真正的数据一致性取决于磁盘写入，而非缓存。 |

#### ⚠️ 确实可能发生的风险

| ID | 风险 | 触发路径 | 实际影响 |
|----|------|----------|----------|
| **C4** | 缓存显示有标记，但卡包磁盘写入失败 | doAddFlashcards L924 更新缓存 → L944 deck.Save() 失败 | 块在编辑器中显示制卡标记（从缓存读），但复习时找不到卡片（从卡包读） |
| **C5** | 事务 rollback 不回滚缓存 + 不回滚卡包 | 同一事务中 addFlashcards 成功 → 后续 op 失败 → rollback | ① 缓存有标记 ② 卡包已 Save ③ 但文档树被 rollback 丢弃 → **三重不一致** |
| **C7** | `pushBlockAttrs` 与磁盘写入不同步 | L925 pushBlockAttrs 在 commit 前就推送属性变更 | 搜索索引可能先于磁盘写入被更新，短时间内搜索结果与磁盘不一致 |

---

## 三、复习回合缓存配额 vs FSRS 间隔重复到期时间

这是两个**完全独立、本质不同**的概念，之前的分析没有明确区分。

### 3.1 核心区别对照表

| 维度 | 复习回合缓存配额 | FSRS 间隔重复到期时间 |
|------|----------------|----------------------|
| **本质** | 内存计数器，控制单回合卡片吞吐量 | 基于算法的绝对时间点，决定卡片何时进入复习队列 |
| **存储位置** | `reviewCardCache`（全局 Map，内存） | `fsrs.Card.Due`（持久化到 .deck 文件） |
| **生命周期** | 回合开始（reviewedCardIDs 为空）时清零；全部复习完成或关闭窗口时清零 | 卡片生命周期内持续存在，每次复习后更新 |
| **计算依据** | 统计 `reviewCardCache` 中 New 状态和非 New 状态的卡片数量 | FSRS 算法：根据 Stability、Difficulty、Rating、RequestRetention 计算 |
| **影响范围** | 仅影响当前批次返回多少张卡片 | 影响卡片是否出现在 `deck.Dues()` 结果中 |
| **用户可见性** | 间接可见（进度条 "已复习/上限"） | 可见（"5 分钟后"、"3 天后" 等提示） |
| **持久化** | 否，重启即失 | 是，写入 `{deckID}.deck` JSON 文件 |
| **跨回合延续** | 否，每个回合独立统计 | 是，到期时间是绝对的，与何时打开复习无关 |

### 3.2 FSRS 到期时间计算原理

**调用链**：
```
deck.Review(cardID, rating)
  ↓ riff 库内部
  fsrs.Repeat(card, now)
    ↓
  for rating in [Again, Hard, Good, Easy]:
    计算 SchedulingCards[rating].card.Due
    ↓
  card.Due = SchedulingCards[actualRating].card.Due
```

**FSRS 核心公式**（`go-fsrs/v3` 内部实现）：
```
间隔计算:
  interval = stability / difficulty * f(rating)
  due = now + interval

稳定性更新 (Stability):
  S' = S * e^(w * (R_target - R)) * decay_factor

难度更新 (Difficulty):
  D' = D + w * (rating_mean - 3) * decay_factor
```

到期时间是**绝对时间点**，一旦计算并持久化，不受后续复习行为影响。

### 3.3 回合缓存配额统计原理

**`getDeckDueCards` 配额统计** [L1126-L1134](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1126-L1134)：
```go
newCount := 0
reviewCount := 0
for _, reviewedCard := range reviewCardCache {
    if riff.New == reviewedCard.GetState() {
        newCount++       // 缓存中状态为 New 的卡 → 计入新卡配额
    } else {
        reviewCount++    // 缓存中状态非 New 的卡 → 计入复习卡配额
    }
}
```

**⚠️ 关键观察**：
- 统计的是**缓存中卡片的当前状态**，而非卡片的原始状态
- 新卡复习后状态从 New → Learning，因此下次配额统计时 `newCount` 可能减少
- 这意味着配额统计是**动态的**，不是简单的"已复习 N 张"

### 3.4 两者的协作关系

```
┌─────────────────────────────────────────────────────────┐
│                    获取到期卡片流程                        │
├─────────────────────────────────────────────────────────┤
│  ① deck.Dues()                                           │
│     ↓ FSRS 层                                            │
│     筛选 due ≤ now 的所有卡片 → dues 列表                 │
│                                                          │
│  ② 遍历 dues，配额过滤                                    │
│     ↓ 回合层                                             │
│     newCount = reviewCardCache 中 New 状态卡数             │
│     reviewCount = reviewCardCache 中非 New 状态卡数        │
│     newCount ≥ NewCardLimit → 跳过后续 New 卡             │
│     reviewCount ≥ ReviewCardLimit → 跳过后续复习卡         │
│                                                          │
│  ③ 返回最终卡片列表 → 前端                                │
└─────────────────────────────────────────────────────────┘
```

**影响示例**：
- 配额 = 20 新卡/天，FSRS 有 50 张新卡 due
- 回合 A 开始：`newCount = 0` → 返回 20 张
- 用户复习 15 张后关闭窗口 → `reviewCardCache` 被丢弃
- 回合 B 重新打开：`newCount = 0` → **再返回 20 张**（不是 5 张）
- 结论：配额是**每回合**的，不是**每天**的

---

## 四、一致性风险的明确判定

### 4.1 制卡操作的执行步骤与风险点

**`doAddFlashcards` 完整时序及失败后果**：

| 步骤 | 操作 | 失败点 | 内存缓存状态 | 文档树状态 | 卡包状态 | 一致性判定 |
|------|------|--------|-------------|-----------|----------|-----------|
| 1 | `deckLock.Lock()` | 阻塞等待 | - | - | - | - |
| 2 | `isSyncingStorages()` | 同步中 → 返回错误 | ❌ 未变 | ❌ 未变 | ❌ 未变 | ✅ 一致 |
| 3 | 查找/创建卡包 | createDeck0 失败 → 后续 deck 为 nil | ❌ 未变 | ❌ 未变 | ❌ 未变 | ✅ 一致 |
| 4 | 遍历 blockIDs 处理属性 | 某块 loadTree 失败 → `continue` | ✅ 成功块已更新 | ⚠️ 待 commit | ❌ 未变 | 待观察 |
| 5 | 遍历 blockIDs 添加卡片 | deck 为 nil → return | ✅ 缓存已更新 | ⚠️ 待 commit | ❌ 未变 | ❌ 不一致（C4 变种） |
| 6 | `deck.Save()` | 磁盘错误 → return | ✅ 缓存已更新 | ⚠️ 待 commit | ❌ 未写入 | ❌ 不一致（C4） |
| 7 | 函数正常返回 | - | ✅ 缓存已更新 | ⚠️ 待 commit | ✅ 已写入 | ⚠️ 文档树仍待 commit |
| 8 | 事务后续操作 | 失败 → `tx.rollback()` | ✅ 缓存已更新 | ❌ 被丢弃 | ✅ 已写入 | ❌ 严重不一致（C5） |
| 9 | `tx.commit()` | - | ✅ 缓存已更新 | ✅ 已写入 | ✅ 已写入 | ✅ 一致 |

### 4.2 明确结论：哪些会发生，哪些不会

#### ✅ 可能发生的一致性问题

| ID | 风险场景 | 发生条件 | 概率 | 影响 |
|----|----------|----------|------|------|
| **C4** | 缓存显示制卡标记，但卡包未写入 | `deck.Save()` 磁盘错误（权限、空间满、文件锁冲突） | 低 | 块显示闪卡图标，但复习时不出现 |
| **C5** | 事务 rollback 三重不一致 | 同一事务包含 addFlashcards 和后续操作，后续操作失败 | 极低 | ① 缓存有标记 ② 卡包有卡片 ③ 文档无属性 |
| **C7** | 搜索索引与磁盘短暂不一致 | `pushBlockAttrs` 在 commit 前推送 | 中（每次制卡都发生） | 搜索可能短暂命中"已制卡"，但实际文档还未写入磁盘 |
| **C8** | 块存在性校验滞后 | 块已删除，但卡包中卡片未移除 | 中 | `ExistBlockTrees` 每次调度时过滤，但卡包中残留无效卡片 |
| **C9** | 多端同步版本冲突 | A 端制卡，B 端同时修改同一块 → 同步覆盖 | 中 | 可能出现块属性丢失或卡包卡片丢失 |

#### ❌ 不会发生的一致性问题

| ID | 之前担心的场景 | 不会发生的原因 |
|----|--------------|----------------|
| ❌ "块属性写入磁盘但卡包未写入" | 块属性写入磁盘发生在 `tx.commit()`（步骤 9），晚于 `deck.Save()`（步骤 6）。如果 deck.Save() 失败，事务会返回错误，不会走到 commit。**磁盘上**的块属性和卡包状态始终一致。不一致仅发生在缓存层。 |
| ❌ "卡包写入但块属性缓存未更新" | `cache.PutBlockIAL()` 在 `deck.Save()` 之前调用（步骤 4 vs 步骤 6），只要执行到步骤 6，缓存必然已更新。 |
| ❌ "复习评分后卡包未持久化" | `ReviewFlashcard` 中 `deck.Save()` 是同步调用，失败会返回 error 给前端，用户会看到错误提示。 |
| ❌ "翻页后配额统计错误" | 配额从 `reviewCardCache` 统计，翻页不清空缓存，配额连续累加。只要窗口不关闭，配额不会"重置"。 |

### 4.3 现有防御机制盘点

| 防御机制 | 位置 | 防御目标 | 局限性 |
|----------|------|----------|--------|
| `deckLock` 全局互斥 | 所有闪卡操作入口 | 防止卡包并发写入 | 无法防止与事务回滚的时序问题 |
| `ExistBlockTrees` 校验 | `getDeckDueCards` L1111 | 过滤已删除块的卡片 | 只在调度时过滤，不主动清理卡包 |
| `GetCardsByBlockID` 去重 | `doAddFlashcards` L935 | 防止同一块重复制卡 | 仅检查卡包内，不检查块属性 |
| `tx.rollback()` | 事务失败时 | 回滚文档树变更 | **不回滚卡包操作和缓存**（关键缺陷） |
| `isSyncingStorages()` | 制卡/删卡前 | 同步中拒绝写入 | 阻塞其他等待锁的操作 |
| `waitForSyncingStorages()` | 复习操作前 | 确保同步完成后再评分 | 可能长时阻塞 |

---

## 五、用户操作反馈的边界场景澄清

### 5.1 已确认的边界行为

| 场景 | 实际行为 | 设计预期 |
|------|----------|----------|
| 关闭窗口后配额重置 | ✅ 重置。`reviewCardCache` 是内存变量，新窗口打开时 `reviewedCardIDs` 为空 → 清空缓存 → 配额从 0 开始 | 预期内（"每日"是产品描述，技术上是"每回合"） |
| 全部完成后无法撤销 | ✅ 无法撤销。`ReviewFlashcard` L502-L507：`unreviewedCount == 0` 时清空缓存 | 设计边界 |
| 跳过操作重启后失效 | ✅ 失效。`SkipReviewFlashcard` 仅写内存 `skipCardCache`，不调用 `deck.Save()` | 设计使然，跳过是临时行为 |
| 切换筛选器丢失撤销 | ✅ 丢失。切换筛选器调用 `fetchNewRound()`，不带 `reviewedCards` → 后端清空缓存 | 设计边界 |
| 刷新丢失当前批次 | ✅ 丢失。前端不持久化 `options.cardsData` 和 `index`，刷新后重新拉取 | 预期内 |

### 5.2 进度显示的不一致边界

**前端进度统计** [genCardCount L30-L52](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/app/src/card/openCard.ts#L30-L52)：
```typescript
cardsData.cards.forEach((item, index) => {
    if (index > allIndex) return
    if (item.state === 0) newIndex++  // 仅统计当前批次
    else oldIndex++
})
```

**后端配额统计**：
```go
for _, reviewedCard := range reviewCardCache {
    if riff.New == reviewedCard.GetState() { newCount++ }
    else { reviewCount++ }  // 统计所有缓存中的卡（含翻页前的批次）
}
```

**不一致表现**：
- 翻页后，前端显示 "0 / 20"（新批次从 0 开始）
- 后端实际配额已消耗 15/20
- 这是显示设计，不是 Bug。用户感知的"已复习"与系统内部的"配额消耗"是两个维度。

---

## 六、修正后的风险评估汇总

### 6.1 数据一致性风险（修正版）

| ID | 风险 | 严重度 | 触发概率 | 影响范围 | 发生可能性 | 现有防御 |
|----|------|--------|----------|----------|------------|----------|
| **C4** | deck.Save() 失败导致缓存与卡包不一致 | 中 | 低 | 局部制卡不一致 | ✅ 可能 | 日志告警 |
| **C5** | 事务 rollback 不回滚卡包和缓存 | **高** | 极低 | **全局不一致** | ✅ 可能 | 无直接防御 |
| **C7** | pushBlockAttrs 与 commit 时序差导致搜索短暂不一致 | 低 | 中 | 搜索结果显示 | ✅ 可能 | 时间自愈（秒级） |
| **C8** | 已删除块的卡片残留卡包 | 低 | 中 | 卡包膨胀、调度时过滤 | ✅ 可能 | ExistBlockTrees 运行时过滤 |
| **C9** | 多端同步版本冲突 | 中 | 中 | 部分卡片重复/丢失 | ✅ 可能 | 时间戳先到先胜 |
| ❌ R1_old | 块属性磁盘写入成功但卡包 Save 失败 | - | - | - | ❌ 不会 | 事务时序保证磁盘写入一致 |
| ❌ R4_old | 同步中断导致缓存不一致无法自愈 | - | - | - | ❌ 不会 | 内存缓存重启自愈 |

### 6.2 同步检测风险

| ID | 风险 | 严重度 | 触发概率 | 说明 |
|----|------|--------|----------|------|
| **S1** | 复习操作持有 deckLock 等待同步 | 中 | 中（同步频繁时） | 所有闪卡操作被阻塞 |
| **S2** | 制卡操作同步中直接失败 | 低 | 中 | 用户体验不佳，需要手动重试 |

### 6.3 时间边界风险

| ID | 风险 | 严重度 | 触发概率 | 说明 |
|----|------|--------|----------|------|
| **T1** | 配额不按日历天重置 | 低 | 高（每次关闭窗口都发生） | "每日20新卡"实为"每回合20新卡" |
| **T2** | 跨时区 due 计算偏差 | 低 | 中（跨时区用户） | 服务器时区与客户端不同导致同一天到期数量不同 |
| **T3** | 长时复习跨天不刷新 | 低 | 低（持续复习 >24h） | 打开窗口时的 due 列表不随时间更新 |
| **T4** | 文档级配额仅文档入口生效 | 低 | 中（设置了文档级配额的用户） | Alt+0 全局入口忽略文档配置 |

---

## 七、后续验证要点（修正版）

### 7.1 一致性验证（核心）

| 编号 | 验证项 | 测试方法 | 预期结果 | 实际风险 |
|------|--------|----------|----------|----------|
| V1 | C5 场景复现 | 构造事务：addFlashcards + 后续注入失败操作 | 检查：① 块缓存属性 ② 块磁盘属性 ③ 卡包数据 | 预期三者不一致（C5 真实存在） |
| V2 | C4 场景复现 | 模拟 deck.Save() 失败（临时移除写入权限） | 检查：① 块缓存是否有标记 ② 块磁盘是否有属性 ③ 卡包是否有卡片 | 预期①有 ②无 ③无（磁盘层一致，缓存层不一致） |
| V3 | ❌ R1_old 证伪 | 同上场景 | 块磁盘属性**不应**存在（证明 R1_old 不会发生） | 验证事务时序正确性 |
| V4 | C7 场景观测 | 单步调试：在 L924 和 commit 之间断点 | 搜索"custom-riff-decks"是否命中该块 | 预期短暂命中（证明 C7 存在） |
| V5 | C8 场景复现 | 制卡 → 删除块 → 重启 → 检查卡包文件 | 卡包中应有残留卡片，但调度时不返回 | 验证 ExistBlockTrees 过滤有效 |

### 7.2 同步检测验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V6 | 制卡同步中失败 | 启动同步 → 立即制卡 | 返回 `TxErrCodeDataIsSyncing` 错误 |
| V7 | 复习同步中等待 | 启动同步 → 立即评分 | 操作阻塞直到同步完成，然后成功 |
| V8 | 同步中 deckLock 阻塞链 | 启动同步 → 评分（持有锁等待）→ 同时制卡 | 制卡先阻塞在 deckLock，同步完成后评分执行完成，制卡再执行 |

### 7.3 配额 vs 到期时间区分验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V9 | 关闭窗口配额重置 | 复习 15 张新卡 → 关闭窗口 → 重新打开 | 再次返回 20 张（不是 5 张） |
| V10 | 配额统计基于缓存状态 | 复习 10 张新卡（状态变为 Learning）→ 翻页 | newCount 从缓存统计，由于状态变化，newCount < 10 |
| V11 | 到期时间与回合无关 | 复习一张卡评 Good → 关闭窗口 → 立即重启 → 查询该卡 due | due 时间不变（证明持久化与回合无关） |
| V12 | 配额与 due 独立 | 50 张新卡 due → 设置配额 20 → 打开复习 → 关闭 → 再打开 | 两次各返回 20 张，due 列表始终有 50 张 |

### 7.4 边界行为验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V13 | 全部完成后撤销尝试 | 复习完最后一张 → 上一步 → 尝试评分 | 缓存已清空，撤销能力丢失（设计边界） |
| V14 | 跳过重启失效 | 跳过 3 张 → 重启 → 打开复习 | 3 张重新出现在列表中 |
| V15 | 切换筛选器丢失撤销 | 复习 5 张 → 切换筛选器 → 上一步 | 无法撤销（设计边界） |
| V16 | 翻页后进度显示重置 | 复习 15 张 → 翻页 | 前端显示 0/20，但后端实际配额已消耗 15 |

---

## 八、关键代码索引（修正版）

### 8.1 同步状态检测

| 函数 | 文件 | 行号 |
|------|------|------|
| `waitForSyncingStorages` (循环等待) | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1210-L1213) | L1210-L1213 |
| `isSyncingStorages` (立即检查) | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1216-L1217) | L1216-L1217 |
| `syncingStorages` 原子变量 | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1208) | L1208 |
| 复习操作使用 waitFor | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L471) | L471 |
| 制卡操作使用 isSyncing | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L861) | L861 |

### 8.2 半成品缓存时序

| 操作 | 文件 | 行号 |
|------|------|------|
| `cache.PutBlockIAL` 缓存更新 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L924) | L924 |
| `tx.writeTree` 标记待写入 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L922) | L922 |
| `pushBlockAttrs` 推送属性 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L925) | L925 |
| `deck.Save` 卡包持久化 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L944) | L944 |
| `PutBlockIAL` 实现 | [cache/ial.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/cache/ial.go#L60-L63) | L60-L63 |

### 8.3 配额与到期时间区分

| 概念 | 代码位置 | 行号 |
|------|----------|------|
| FSRS Card.Due 字段访问 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L388-L389) | L388-L389 |
| `deck.Dues()` 到期筛选 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1097) | L1097 |
| `reviewCardCache` 配额统计 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1126-L1134) | L1126-L1134 |
| `card.NextDues()` 预计算 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L540) | L540 |
| `deck.Review()` 调用 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L491) | L491 |

### 8.4 事务边界

| 函数 | 文件 | 行号 |
|------|------|------|
| `tx.rollback` 仅回滚文档树 | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1897-L1902) | L1897-L1902 |
| `tx.commit` 写入文档树 | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1865-L1894) | L1865-L1894 |
| `doAddFlashcards` 事务入口 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949) | L857-L949 |
| 事务操作分发 | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L211-L214) | L211-L214 |

### 8.5 关键全局变量

| 变量 | 类型 | 作用 | 生命周期 |
|------|------|------|----------|
| `reviewCardCache` | `map[string]riff.Card` | 撤销缓存 + 配额统计基数 | 回合开始/结束清空 |
| `skipCardCache` | `map[string]riff.Card` | 跳过标记（内存） | 回合开始/结束清空 |
| `Decks` | `map[string]*riff.Deck` | 已加载卡包（含 FSRS 卡片状态） | 进程生命周期，LoadFlashcards 填充 |
| `deckLock` | `sync.Mutex` | 闪卡操作全局互斥 | 进程生命周期 |
| `syncingStorages` | `atomic.Bool` | 存储同步状态标记 | 同步期间为 true |
