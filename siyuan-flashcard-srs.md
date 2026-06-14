# SiYuan 闪卡制卡代码核对分析：返回值、事务与一致性

## 一、核心结论速览

| 失败场景 | 块属性缓存 | 块属性磁盘 | 卡包内存 | 卡包磁盘 | 一致性 |
|----------|-----------|-----------|----------|----------|--------|
| **S1: deck.Save() 失败** | ✅ 已更新 | ✅ 已写入 | ✅ 已修改 | ❌ 未写入 | **磁盘级不一致** ⚠️ |
| **S2: 后续操作失败 → rollback** | ✅ 已更新 | ❌ 被丢弃 | ✅ 已修改 | ✅ 已写入 | **三重不一致** 🚨 |
| **S3: tx.commit() 失败** | ✅ 已更新 | ⚠️ 部分/未写入 | ✅ 已修改 | ✅ 已写入 | **部分不一致** ⚠️ |
| **S0: 全部成功** | ✅ 已更新 | ✅ 已写入 | ✅ 已修改 | ✅ 已写入 | ✅ 一致 |

> **关键发现**：`doAddFlashcards` 在 `deck.Save()` 失败时**不返回错误**（仅记录日志），事务继续执行并 commit，导致块属性磁盘已写入但卡包磁盘未写入的**磁盘级不一致**。

---

## 二、制卡代码执行时序详解

### 2.1 `doAddFlashcards` 完整执行路径

代码位置：[flashcard.go#L857-L949](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949)

```
doAddFlashcards 执行时序：
┌─ ① deckLock.Lock()  ────────────────────────────── 获取全局互斥锁
├─ ② isSyncingStorages()  ───────────────────────── 同步中直接返回错误
│     └─ 返回 &TxErr{code: TxErrCodeDataIsSyncing}
├─ ③ 查找/创建卡包  ────────────────────────────────
│     ├─ Decks[deckID] 存在 → 直接使用
│     └─ 不存在 → createDeck0() → 内部 deck.Save()
├─ ④ 遍历 blockIDs，逐个设置块属性  ────────────────
│     ├─ loadTree(rootID)  → 加载文档树到内存
│     ├─ node.SetIALAttr("custom-riff-decks", val)  → 修改节点属性
│     ├─ tx.writeTree(tree)  → 标记为待写入（仍在内存 tx.trees）
│     ├─ cache.PutBlockIAL(blockID, ...)  →  ⚠️ 立即更新内存缓存
│     └─ pushBlockAttrs(oldAttrs, node)  →  ⚠️ 推送属性变更（搜索索引等）
├─ ⑤ 遍历 blockIDs，逐个 AddCard  ────────────────
│     ├─ 去重检查：deck.GetCardsByBlockID(blockID)
│     └─ deck.AddCard(ast.NewNodeID(), blockID)  → 仅内存修改
├─ ⑥ deck.Save()  ─────────────────────────────── 写入 .deck 文件
│     └─ ❌ 失败：logging.LogErrorf(...) → return （返回 nil，不报错！）
└─ ⑦ return nil  ──────────────────────────────── 正常返回
```

### 2.2 关键代码：返回值分析

**`deck.Save()` 失败时的处理** [flashcard.go#L944-L947](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L944-L947)：
```go
if err := deck.Save(); err != nil {
    logging.LogErrorf("save deck [%s] failed: %s", deckID, err)
    return   // 返回 nil，不返回 TxErr
}
```

**函数签名**：
```go
func (tx *Transaction) doAddFlashcards(operation *Operation) (ret *TxErr)
```

**推论**：`deck.Save()` 失败时，`ret` 保持 nil，函数正常返回。

### 2.3 事务层的处理逻辑

代码位置：[transaction.go#L183-L341](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L183-L341)

```go
for _, op := range tx.DoOperations {
    switch op.Action {
    case "addFlashcards":
        ret = tx.doAddFlashcards(op)  // 返回 nil
    // ... 其他操作
    }

    if nil != ret {  // ret 为 nil，不触发 rollback
        tx.rollback()
        return
    }
}
// 循环全部通过后
if cr := tx.commit(); nil != cr {  // 执行 commit
    ...
}
```

**结论**：`deck.Save()` 失败 → 返回 nil → 不触发 rollback → 继续 commit → **块属性写入磁盘，但卡包未写入**。

---

## 三、三种失败场景的详细分析

### 3.1 场景 S1：卡包保存失败（deck.Save error）

**触发条件**：磁盘满、权限不足、文件锁定、I/O 错误等导致 `deck.Save()` 返回 error。

**执行轨迹**：
1. ✅ `cache.PutBlockIAL()` → 缓存更新
2. ✅ `pushBlockAttrs()` → 搜索索引推送
3. ✅ `deck.AddCard()` → 卡包内存修改
4. ❌ `deck.Save()` → 卡包磁盘写入失败
5. ✅ `doAddFlashcards` 返回 nil → 不触发 rollback
6. ✅ `tx.commit()` → 块属性写入磁盘

**最终状态**：

| 数据层 | 状态 | 说明 |
|--------|------|------|
| 块属性缓存 | ✅ 已更新 | cache.PutBlockIAL 先于 deck.Save 执行 |
| 块属性磁盘 | ✅ 已写入 | commit 在 deck.Save 之后执行，且不受 Save 失败影响 |
| 卡包内存 | ✅ 已修改 | AddCard 已执行 |
| 卡包磁盘 | ❌ 未写入 | Save 失败 |
| 搜索索引 | ✅ 已推送 | pushBlockAttrs 先执行 |

**后果**：
- 块在编辑器中显示闪卡标记
- 块的 `custom-riff-decks` 属性在磁盘上存在
- 但复习时找不到该卡片
- 重启后依然不一致（磁盘级）
- 只能通过"先移除再重新制卡"手动修复

**发生概率**：低（依赖磁盘异常）
**严重度**：中

---

### 3.2 场景 S2：后续操作失败 → rollback

**触发条件**：同一事务中，`addFlashcards` 之后的其他操作（如 update/delete/insert 等）执行失败，触发 `tx.rollback()`。

**执行轨迹**：
1. ✅ `doAddFlashcards` 全部执行成功，返回 nil
   - 缓存已更新
   - pushBlockAttrs 已推送
   - deck.AddCard 已执行
   - deck.Save 已成功写入磁盘
2. ❌ 后续 op 执行失败 → `ret != nil`
3. ❌ `tx.rollback()` 被调用

**rollback 的实际行为** [transaction.go#L1897-L1902](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1897-L1902)：
```go
func (tx *Transaction) rollback() {
    tx.trees, tx.nodes = nil, nil   // 仅丢弃文档树变更
    tx.state.Store(3)
    tx.m.Unlock()
    return
}
```

**rollback 不回滚的内容**：
- ❌ 不回滚 `cache.PutBlockIAL()`（缓存已更新）
- ❌ 不回滚 `pushBlockAttrs()`（搜索索引已推送）
- ❌ 不回滚 `deck.AddCard()` + `deck.Save()`（卡包已落盘）

**最终状态**：

| 数据层 | 状态 | 说明 |
|--------|------|------|
| 块属性缓存 | ✅ 已更新 | rollback 不清理缓存 |
| 块属性磁盘 | ❌ 未写入 | tx.trees 被丢弃，未 commit |
| 卡包内存 | ✅ 已修改 | rollback 不影响卡包 |
| 卡包磁盘 | ✅ 已写入 | deck.Save 在 rollback 之前已成功 |
| 搜索索引 | ✅ 已推送 | rollback 不撤回推送 |

**后果**：
- 三重不一致：缓存 ✅ / 块磁盘 ❌ / 卡包磁盘 ✅
- 编辑器中可能显示闪卡标记（从缓存读），但刷新页面后消失（从磁盘重新加载）
- 卡包中有卡片，但块没有属性，无法通过块 UI 管理
- 复习时卡片会出现，但点击"定位"可能找不到对应块

**发生概率**：极低（需要 addFlashcards + 后续操作失败同时发生）
**严重度**：高

---

### 3.3 场景 S3：tx.commit() 失败

**触发条件**：`tx.commit()` 中 `writeTreeUpsertQueue()` 失败（如磁盘满、文件损坏）。

**执行轨迹**：
1. ✅ `doAddFlashcards` 全部成功
2. ✅ 其他 op 全部成功
3. ❌ `tx.commit()` 执行到某 tree 的 `writeTreeUpsertQueue()` 失败
4. ❌ commit 提前返回 error

**commit 的行为** [transaction.go#L1865-L1894](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1865-L1894)：
```go
func (tx *Transaction) commit() (err error) {
    for _, tree := range tx.trees {
        if err = writeTreeUpsertQueue(tree); err != nil {
            return   // 中途失败，直接返回
        }
        // ... 其他处理
    }
    // ...
}
```

**最终状态**：

| 数据层 | 状态 | 说明 |
|--------|------|------|
| 块属性缓存 | ✅ 已更新 | commit 前已更新 |
| 块属性磁盘 | ⚠️ 部分写入 | 失败点之前的 tree 已写入，之后的未写入 |
| 卡包内存 | ✅ 已修改 | commit 前已修改 |
| 卡包磁盘 | ✅ 已写入 | deck.Save 在 commit 之前 |
| 搜索索引 | ✅ 已推送 | pushBlockAttrs 先执行 |

**后果**：
- 部分块的属性已写入，部分未写入
- 卡包中所有卡片都已写入
- 部分双向关联断裂

**发生概率**：极低
**严重度**：中

---

### 3.4 场景 S0：全部成功（对照）

**执行轨迹**：
1. ✅ `cache.PutBlockIAL()` → 缓存更新
2. ✅ `pushBlockAttrs()` → 搜索索引推送
3. ✅ `deck.AddCard()` → 卡包内存修改
4. ✅ `deck.Save()` → 卡包磁盘写入
5. ✅ `doAddFlashcards` 返回 nil
6. ✅ 其他 op 全部成功
7. ✅ `tx.commit()` → 块属性磁盘写入

**最终状态**：

| 数据层 | 状态 |
|--------|------|
| 块属性缓存 | ✅ 一致 |
| 块属性磁盘 | ✅ 一致 |
| 卡包内存 | ✅ 一致 |
| 卡包磁盘 | ✅ 一致 |

---

## 四、一致性风险的明确判定

### 4.1 会发生的一致性风险

| ID | 风险场景 | 对应场景 | 发生可能性 | 严重度 | 触发条件 |
|----|----------|----------|-----------|--------|----------|
| **R1** | 块属性磁盘已写入，但卡包磁盘未写入 | S1 | ✅ 确认存在 | 中 | `deck.Save()` 失败 |
| **R2** | 事务 rollback 三重不一致 | S2 | ✅ 确认存在 | 高 | addFlashcards 成功 + 后续 op 失败 |
| **R3** | commit 部分失败导致部分不一致 | S3 | ✅ 确认存在 | 中 | commit 中途磁盘错误 |
| **R4** | pushBlockAttrs 与磁盘写入时序差 | 所有场景 | ✅ 确认存在 | 低 | commit 前推送属性，搜索索引短暂超前 |
| **R5** | 已删除块的卡片残留卡包 | 长期运行 | ✅ 确认存在 | 低 | 块删除后未同步清理卡包 |
| **R6** | 多端同步版本冲突 | 多端使用 | ✅ 确认存在 | 中 | 两端同时修改同一块/卡包 |

### 4.2 不会发生的一致性风险

| ID | 常见误解 | 不会发生的原因 |
|----|----------|----------------|
| ❌ "块属性缓存未更新但卡包已写入" | - | `cache.PutBlockIAL()`（L924）在 `deck.Save()`（L944）之前调用，时序上不可能 |
| ❌ "卡包写入失败会触发事务回滚，块属性不会落盘" | - | `deck.Save()` 失败不返回错误，事务不会回滚，块属性依然会 commit |
| ❌ "rollback 会回滚卡包操作" | - | `rollback()` 仅清空 `tx.trees` 和 `tx.nodes`，不涉及卡包 |
| ❌ "制卡操作同步中会等待同步完成" | - | 制卡使用 `isSyncingStorages()`（立即失败），不是 `waitForSyncingStorages()`（等待） |
| ❌ "每日新卡配额按自然日重置" | - | 配额基于 `reviewCardCache`，回合结束/窗口关闭即重置，不是日历天 |
| ❌ "删卡的 deck.Save 失败会返回错误" | - | `removeFlashcardsByBlockIDs` 返回 void，Save 失败仅记日志 |

### 4.3 风险发生路径图

```
                        开始制卡
                           │
                           ▼
                ┌──────────────────────┐
                │  isSyncingStorages?  │── true ──→ 返回错误（一致）
                └──────────────────────┘
                           │ false
                           ▼
                ┌──────────────────────┐
                │  cache.PutBlockIAL   │── ✅ 缓存更新
                └──────────────────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │   pushBlockAttrs     │── ✅ 搜索推送
                └──────────────────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │   deck.AddCard       │── ✅ 内存修改
                └──────────────────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │     deck.Save        │
                └──────────────────────┘
                           │
                ┌──────────┴──────────┐
                │ 成功                │ 失败
                ▼                     ▼
         后续 op 循环          返回 nil（不报错）───→ R1：磁盘级不一致
                │
      ┌─────────┴──────────┐
      │ 全部成功            │ 某个失败
      ▼                     ▼
   tx.commit           tx.rollback ─────→ R2：三重不一致
      │
      ├─ 成功 → S0：完全一致
      └─ 失败 → R3：部分不一致
```

---

## 五、复习回合缓存配额 vs 间隔重复到期时间

这是两个**完全独立、本质不同**的概念，必须严格区分。

### 5.1 核心区别对照表

| 维度 | 复习回合缓存配额 | FSRS 间隔重复到期时间 |
|------|----------------|----------------------|
| **本质** | 内存计数器，控制单回合卡片吞吐量 | 基于算法的绝对时间点，决定卡片何时进入复习队列 |
| **数据来源** | `reviewCardCache`（全局 Map） | `fsrs.Card.Due` 字段 |
| **存储位置** | 内存 | 持久化到 `{deckID}.deck` JSON 文件 |
| **生命周期** | 回合开始（reviewedCardIDs 为空）时清零；全部复习完成或关闭窗口时清零 | 卡片生命周期内持续存在 |
| **计算依据** | 统计 reviewCardCache 中卡片的初始状态（New / 非 New） | FSRS 算法：Stability、Difficulty、Rating、RequestRetention |
| **影响范围** | 单次 API 返回多少张卡片 | 是否出现在 deck.Dues() 结果中 |
| **用户感知** | 间接（进度条 "已复习/上限"） | 直接（"3 天后复习" 提示） |
| **持久化** | 否，重启即失 | 是，写入磁盘 |
| **跨回合延续** | 否，每个回合独立统计 | 是，绝对时间，与何时打开无关 |
| **"每日"含义** | 实为"每回合"，关闭窗口即重置 | 真正的时间间隔，与"日"无关（可以是分钟/小时/天/月） |

### 5.2 FSRS 到期时间计算

**调用链**：
```
用户评分
  ↓
ReviewFlashcard(deckID, cardID, rating)
  ↓
deck.Review(cardID, rating)  [riff 库]
  ↓
fsrs.Repeat(card, now)       [go-fsrs 库]
  ↓
计算 SchedulingCards → 选择对应 rating 的 Card
  ↓
card.Due = 计算出的下次到期时间
card.Stability / card.Difficulty = 更新后的参数
  ↓
deck.Save()  →  持久化到 .deck 文件
```

**关键代码**：
- 评分入口：[ReviewFlashcard#L491](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L491)
- 下次到期预览：[NextDues#L540](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L540)
- 到期筛选：[getDeckDueCards#L1097](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1097)

**到期时间特性**：
- 是**绝对时间点**（如 2026-06-18 14:30:00）
- 计算后持久化，不受后续复习行为影响
- 与"天"没有必然联系，可以是分钟、小时、天、月
- 服务端时区计算，客户端显示时可能有时差

### 5.3 回合缓存配额统计

#### 5.3.1 reviewCardCache 的双重作用

`reviewCardCache` 存储的是**首次评分前的状态快照**，同时服务于两个目的：
1. **撤销恢复**：重评时恢复到初始状态
2. **配额统计**：按初始状态统计已用配额

代码位置：[flashcard.go#L488](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L488)：
```go
reviewCardCache[cardID] = card.Clone()  // 首次评分前克隆
```

#### 5.3.2 配额统计逻辑

代码位置：[flashcard.go#L1126-L1134](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1126-L1134)：
```go
newCount := 0
reviewCount := 0
for _, reviewedCard := range reviewCardCache {
    if riff.New == reviewedCard.GetState() {
        newCount++       // 初始状态为 New → 计入新卡配额
    } else {
        reviewCount++    // 初始状态非 New → 计入复习卡配额
    }
}
```

#### 5.3.3 配额统计特性

| 卡片初始状态 | 加入缓存时机 | 计入配额 |
|-------------|-------------|----------|
| New | 首次评分后 | newCount++ |
| Learning | 首次评分后 | reviewCount++ |
| Review | 首次评分后 | reviewCount++ |
| Relearning | 首次评分后 | reviewCount++ |

**关键点**：
- 统计的是**初始状态**，不是当前状态
- 一张新卡一旦被复习，始终计入 newCount，不会因为状态变为 Learning 而转移
- 配额统计是**稳定的**，不会因为状态变化而波动

#### 5.3.4 配额协作示例（新卡上限 20）

| 时间点 | 操作 | newCount | 返回/补充新卡数 |
|--------|------|----------|----------------|
| T0 | 打开复习窗口（回合开始） | 0 | 20 张 |
| T1 | 复习 15 张 | 15 | - |
| T2 | 翻页 | 15 | 补充 5 张，批次共 25 张 |
| T3 | 复习完 20 张 | 20 | 配额满，不再返回新卡 |
| T4 | 关闭窗口 → 再打开 | 0 | 再返回 20 张（如果 due 足够） |

---

## 六、删卡操作的一致性分析

### 6.1 `doRemoveFlashcards` 执行路径

代码位置：[flashcard.go#L747-L771](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L747-L771)

```
doRemoveFlashcards 执行时序：
┌─ ① deckLock.Lock()
├─ ② isSyncingStorages() → 同步中返回错误
├─ ③ tx.removeBlocksDeckAttr(blockIDs, deckID)  ── 移除块属性
│     ├─ 遍历 blockIDs，逐个 loadTree
│     ├─ node.RemoveIALAttr / SetIALAttr
│     ├─ tx.writeTree(tree)
│     ├─ cache.PutBlockIAL(blockID, ...)  ← 更新缓存
│     └─ pushBlockAttrs(...)
├─ ④ removeFlashcardsByBlockIDs(blockIDs, deck)  ── 移除卡包卡片
│     ├─ deck.GetCardsByBlockIDs(blockIDs)
│     ├─ 逐个 deck.RemoveCard(card.ID())
│     └─ deck.Save()  ← 失败只记日志，不返回错误
└─ ⑤ return nil
```

### 6.2 `removeFlashcardsByBlockIDs` 的返回值

代码位置：[flashcard.go#L837-L855](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L837-L855)：
```go
func removeFlashcardsByBlockIDs(blockIDs []string, deck *riff.Deck) {
    // ...
    err := deck.Save()
    if err != nil {
        logging.LogErrorf("save deck [%s] failed: %s", deck.ID, err)
        // 不返回错误，函数返回 void
    }
}
```

**结论**：删卡操作的 `deck.Save()` 失败同样**不返回错误**，事务继续 commit。

### 6.3 删卡的失败场景

| 失败场景 | 块属性 | 卡包磁盘 | 一致性 |
|----------|--------|----------|--------|
| deck.Save() 失败 | ✅ 已移除（commit 后） | ❌ 卡片还在 | **反向不一致**：块无属性，但卡包有卡片 |
| 后续操作失败 → rollback | ❌ 未移除（rollback） | ✅ 已移除 | **反向不一致**：块有属性，但卡包无卡片 |
| commit 失败 | ⚠️ 部分移除 | ✅ 已移除 | 部分不一致 |

**反向不一致的后果**：
- 块不显示闪卡标记，但复习时卡片会出现
- 用户困惑："我明明移除了，怎么还在复习列表里？"

---

## 七、同步状态检测的两种策略

### 7.1 策略对照表

| 策略 | 函数 | 行为 | 适用操作 | 设计意图 |
|------|------|------|----------|----------|
| 阻塞等待 | `waitForSyncingStorages()` | 每秒轮询，直到同步完成 | 复习类操作（7 个） | 确保评分写入时文件不被同步覆盖 |
| 立即失败 | `isSyncingStorages()` | 立即返回 bool，不等待 | 制卡/删卡类操作（2 个） | 同步中避免写入冲突，直接提示用户 |

### 7.2 触发顺序问题

两种策略都有同样的触发顺序问题：**先获取 deckLock，再检查同步状态**。

**`doAddFlashcards`** [L858-L864](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L858-L864)：
```go
func (tx *Transaction) doAddFlashcards(operation *Operation) (ret *TxErr) {
    deckLock.Lock()           // ① 先获取锁
    defer deckLock.Unlock()
    if isSyncingStorages() {  // ② 后检查同步
        ret = &TxErr{code: TxErrCodeDataIsSyncing}
        return
    }
```

**`ReviewFlashcard`** [L468-L472](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L468-L472)：
```go
func ReviewFlashcard(...) (err error) {
    deckLock.Lock()             // ① 先获取锁
    defer deckLock.Unlock()
    waitForSyncingStorages()    // ② 后等待同步
```

**风险**：
- 同步期间，所有闪卡操作都先阻塞在 `deckLock.Lock()` 上
- 锁持有期间才发现同步中 → 等待/失败
- 同步时间长时，可能形成操作队列堆积

---

## 八、风险评估汇总

### 8.1 数据一致性风险

| ID | 风险 | 严重度 | 发生概率 | 影响范围 | 确认状态 |
|----|------|--------|----------|----------|----------|
| **R1** | deck.Save() 失败导致磁盘级不一致 | 中 | 低 | 块显示标记但复习找不到 | ✅ 确认存在 |
| **R2** | 事务 rollback 三重不一致 | **高** | 极低 | 缓存/块磁盘/卡包三方不一致 | ✅ 确认存在 |
| **R3** | commit 部分失败 | 中 | 极低 | 部分块关联断裂 | ✅ 确认存在 |
| **R4** | pushBlockAttrs 时序差（搜索短暂超前） | 低 | 高 | 搜索结果短暂不一致 | ✅ 确认存在 |
| **R5** | 已删除块卡片残留 | 低 | 中 | 卡包膨胀 | ✅ 确认存在 |
| **R6** | 多端同步版本冲突 | 中 | 中 | 卡片重复/丢失 | ✅ 确认存在 |

### 8.2 设计风险

| ID | 风险 | 严重度 | 说明 |
|----|------|--------|------|
| **D1** | deck.Save() 失败静默处理 | 中 | 仅记录日志，不返回错误，上层无感知 |
| **D2** | 闪卡操作不在事务原子性范围内 | 高 | rollback 不回滚卡包操作，破坏事务原子性 |
| **D3** | 缓存更新早于磁盘写入 | 低 | 写穿式缓存正常设计，但失败时不一致 |

### 8.3 时间边界风险

| ID | 风险 | 严重度 | 说明 |
|----|------|--------|------|
| **T1** | "每日配额"实为"每回合配额" | 低 | 关闭窗口即重置，与产品描述的"每日"有偏差 |
| **T2** | 配额按初始状态统计 | 低 | 新卡复习后状态变化，但配额计数不变（设计如此，稳定性优先） |
| **T3** | 跨时区 due 显示差异 | 低 | 服务端时区计算，客户端可能显示不同 |

---

## 九、后续验证要点

### 9.1 一致性验证（核心）

| 编号 | 验证项 | 测试方法 | 预期结果 | 验证目标 |
|------|--------|----------|----------|----------|
| V1 | R1 复现：deck.Save 失败 | 临时移除 riff 目录写入权限 → 制卡 | ① 块属性磁盘已写入 ② 卡包文件未更新 ③ 编辑器显示制卡标记 | 证明磁盘级不一致真实存在 |
| V2 | R2 复现：rollback 不回滚卡包 | 构造复合事务：addFlashcards + 注入失败的 update op | ① 卡包有卡片 ② 块无属性 ③ 缓存有标记 | 证明三重不一致真实存在 |
| V3 | D1 验证：Save 失败无错误返回 | 模拟 deck.Save 失败，检查 API 返回 | API 返回成功（或无错误），仅日志有记录 | 证明失败被静默处理 |
| V4 | R5 验证：删除块后卡片残留 | 制卡 → 删除块 → 重启 → 检查 .deck 文件 | 卡包中仍有该 blockID 的卡片 | 证明 ExistBlockTrees 只过滤不清理 |
| V5 | 删卡 R1 反向验证 | 模拟删卡时 deck.Save 失败 | 块属性已移除，但卡包中卡片仍在 | 证明反向不一致 |

### 9.2 配额 vs 到期时间验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V6 | 配额按初始状态统计 | 复习 10 张新卡（状态变 Learning）→ 翻页 | newCount 仍为 10，还能补充 10 张 |
| V7 | 关闭窗口配额重置 | 复习 15 张新卡 → 关闭 → 重开 | 再次返回 20 张（不是 5 张） |
| V8 | 到期时间持久化 | 复习一张卡评 Good → 记录 due 时间 → 重启 → 查询该卡 due | due 时间不变 |
| V9 | 配额与到期时间独立 | 50 张新卡 due → 复习 20 张 → 关闭 → 重开 | 因配额重置，再次返回 20 张（证明配额回合级、due 持久化） |

### 9.3 同步状态验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V10 | 制卡同步中立即失败 | 启动同步 → 立即制卡 | 返回同步中错误，不等待 |
| V11 | 复习同步中阻塞等待 | 启动同步 → 立即评分 | 操作阻塞直到同步完成，然后成功 |
| V12 | deckLock 顺序问题 | 启动同步 → 评分（持有锁等待）→ 同时制卡 | 制卡先阻塞在 deckLock，同步完成后评分执行，制卡再执行 |

---

## 十、关键代码索引

### 10.1 制卡核心

| 函数 | 文件 | 行号 |
|------|------|------|
| `doAddFlashcards` | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949) | L857-L949 |
| `doRemoveFlashcards` | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L747-L771) | L747-L771 |
| `removeBlocksDeckAttr` | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L773-L835) | L773-L835 |
| `removeFlashcardsByBlockIDs` | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L837-L855) | L837-L855 |
| `deck.Save()` 失败静默处理 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L944-L947) | L944-L947 |

### 10.2 事务与回滚

| 函数 | 文件 | 行号 |
|------|------|------|
| `performTx`（事务主循环） | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L148-L349) | L148-L349 |
| `tx.commit()` | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1865-L1895) | L1865-L1895 |
| `tx.rollback()` | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1897-L1902) | L1897-L1902 |
| 事务操作分发（含闪卡） | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L211-L214) | L211-L214 |

### 10.3 同步状态检测

| 函数 | 文件 | 行号 |
|------|------|------|
| `waitForSyncingStorages` | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1210-L1213) | L1210-L1213 |
| `isSyncingStorages` | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1216-L1217) | L1216-L1217 |
| `syncingStorages` 变量 | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1208) | L1208 |

### 10.4 配额与到期时间

| 概念 | 代码位置 | 行号 |
|------|----------|------|
| 配额统计（reviewCardCache） | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1126-L1134) | L1126-L1134 |
| 到期筛选（deck.Dues()） | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1097) | L1097 |
| 评分入口（deck.Review()） | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L491) | L491 |
| 撤销缓存写入 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L488) | L488 |
| `reviewCardCache` 声明 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L61-L62) | L61-L62 |

### 10.5 缓存与推送

| 操作 | 文件 | 行号 |
|------|------|------|
| `cache.PutBlockIAL` 调用 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L924) | L924 |
| `pushBlockAttrs` 调用 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L925) | L925 |
| `tx.writeTree` 调用 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L922) | L922 |
| `PutBlockIAL` 实现 | [cache/ial.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/cache/ial.go#L60-L63) | L60-L63 |
