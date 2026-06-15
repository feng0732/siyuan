# SiYuan 闪卡制卡失败场景代码分析

## 一、核心结论修正

### 1.1 之前结论的错误

> ❌ 旧结论："deck.Save() 失败后，复习时找不到卡片"

**错误原因**：忽略了 `deck.AddCard()` 修改的是**内存卡包** `Decks[deckID]`，而复习接口 `getDeckDueCards` 中的 `deck.Dues()` 读取的也是**同一份内存卡包**。因此，`deck.Save()` 失败并不影响运行期复习。

### 1.2 正确结论

> ✅ 新结论："deck.Save() 失败后，**运行期复习正常**（卡片在内存卡包中可被找到）；**重启后卡片丢失**（从磁盘加载卡包，未持久化的卡片不在文件中）"

### 1.3 关键机制：ReviewFlashcard 的 Save 可能自动修复

`ReviewFlashcard` 在每次评分后都会调用 `deck.Save()` [flashcard.go#L492](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L492)，此 Save 会将**整个卡包**（包括之前 AddCard 未持久化的卡片）一起写入磁盘。如果磁盘故障是暂时的，复习评分时的 Save 成功，则之前的未持久化卡片被一并补存。

---

## 二、制卡失败场景的完整状态矩阵

### 2.1 场景 S1：deck.Save() 失败

#### 执行轨迹

代码位置：[flashcard.go#L857-L949](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949)

```
doAddFlashcards 执行时序：
├─ ① deckLock.Lock()
├─ ② isSyncingStorages() → false
├─ ③ 查找/创建卡包 → deck = Decks[deckID]
├─ ④ 遍历 blockIDs 设置块属性
│     ├─ node.SetIALAttr("custom-riff-decks", val)
│     ├─ tx.writeTree(tree)          → 标记待写入
│     ├─ cache.PutBlockIAL(...)      → 内存缓存更新
│     └─ pushBlockAttrs(...)         → 搜索索引推送
├─ ⑤ 遍历 blockIDs 添加卡片
│     └─ deck.AddCard(cardID, blockID)  → ⚠️ 内存卡包修改
├─ ⑥ deck.Save()  → ❌ 失败，仅记日志，返回 nil
└─ ⑦ return nil   → 事务继续 → commit
```

#### 运行期状态（未重启）

| 数据层 | 状态 | 说明 | 复习是否可找到 |
|--------|------|------|--------------|
| 块属性缓存 | ✅ 已更新 | `cache.PutBlockIAL()` 先于 Save 执行 | - |
| 块属性磁盘 | ✅ 已写入 | commit 在 Save 之后执行，不受影响 | - |
| **卡包内存** (`Decks[deckID]`) | **✅ 已包含卡片** | `deck.AddCard()` 在 Save 之前已执行 | **✅ 可以找到** |
| 卡包磁盘 | ❌ 未写入 | Save 失败 | - |
| 搜索索引 | ✅ 已推送 | pushBlockAttrs 先执行 | - |

**运行期复习验证**：

1. `getDeckDueCards` 调用 `deck.Dues()` [L1097](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1097)
2. `deck.Dues()` 遍历 `deck.Cards`（内存）找到 `due ≤ now` 的卡片
3. 由于 `deck.AddCard()` 已将卡片加入内存，**卡片出现在复习列表中** ✅
4. 用户评分 → `ReviewFlashcard` → `deck.Review()` → **`deck.Save()`** [L492](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L492)
5. 如果此次 Save 成功 → 卡片（含复习后状态）被持久化 → **自动修复** ✅
6. 如果此次 Save 仍然失败 → 卡片仍仅在内存中

#### 重启后状态

| 数据层 | 状态 | 说明 |
|--------|------|------|
| 块属性缓存 | ✅ 已加载 | 从磁盘重新加载块属性 |
| 块属性磁盘 | ✅ 已写入 | 不受卡包 Save 失败影响 |
| **卡包内存** | **❌ 不含该卡片** | 从磁盘 .deck 文件加载，未持久化的卡片不存在 |
| 卡包磁盘 | ❌ 未写入 | Save 失败 |

**重启后后果**：
- `LoadFlashcards()` [L951-L985](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L951-L985) 从 `storage/riff/` 读取 `.deck` 文件
- 未持久化的卡片不在 `.deck` 文件中 → 加载后 `Decks[deckID]` 不含该卡片
- 块属性有 `custom-riff-decks`，但卡包无对应卡片 → **重启后不一致**
- 但：如果在运行期已成功复习过该卡片（且 Save 成功），则重启后一致

#### 关键时间线

```
deck.Save() 失败
    │
    ├───── 运行期 ─────────────────────────────────────────┐
    │   卡包内存有卡片 → 复习可找到 → 可能通过 Review 补存    │
    │                                                      │
    ├───── 未补存就重启 ────────────────────────────────────┐
    │   卡包从磁盘加载 → 卡片丢失 → 块属性孤立              │
    │                                                      │
    └───── 已通过 Review 补存后重启 ────────────────────────┐
        卡包从磁盘加载 → 卡片存在（含复习状态）→ 一致 ✅     │
```

---

### 2.2 场景 S2：后续操作失败 → rollback

#### 执行轨迹

1. ✅ `doAddFlashcards` 全部成功（含 `deck.Save()` 成功），返回 nil
2. ❌ 后续 op 失败 → `ret != nil`
3. ❌ `tx.rollback()` 被调用 → 仅清空 `tx.trees`

#### 运行期状态（未重启）

| 数据层 | 状态 | 说明 | 复习是否可找到 |
|--------|------|------|--------------|
| 块属性缓存 | ✅ 已更新 | rollback 不清理缓存 | - |
| 块属性磁盘 | ❌ 未写入 | tx.trees 被丢弃 | - |
| **卡包内存** | **✅ 已包含卡片** | rollback 不影响卡包 | **✅ 可以找到** |
| 卡包磁盘 | ✅ 已写入 | deck.Save 在 rollback 之前已成功 | - |

**运行期复习验证**：
- 卡片在内存卡包中 → 复习可找到 ✅
- 块属性缓存有标记 → 编辑器显示闪卡图标 ✅
- 但块属性磁盘未写入 → 刷新页面后图标消失

#### 重启后状态

| 数据层 | 状态 | 说明 |
|--------|------|------|
| 块属性缓存 | ✅ 从磁盘重新加载 | 磁盘无该属性 → 缓存也无 |
| 块属性磁盘 | ❌ 未写入 | rollback 丢弃了 tx.trees |
| **卡包内存** | **✅ 含该卡片** | 从磁盘 .deck 文件加载（Save 成功过） |
| 卡包磁盘 | ✅ 已写入 | deck.Save 已成功 |

**重启后后果**：
- 卡包有卡片，块属性无标记 → **反向不一致**
- 复习时卡片出现，但编辑器中不显示闪卡标记
- 用户无法通过 UI 移除该卡片（因为块上没有 `custom-riff-decks` 属性）

---

### 2.3 场景 S3：tx.commit() 失败

#### 运行期状态

| 数据层 | 状态 |
|--------|------|
| 块属性缓存 | ✅ 已更新 |
| 块属性磁盘 | ⚠️ 部分写入 |
| 卡包内存 | ✅ 已包含卡片 |
| 卡包磁盘 | ✅ 已写入 |

#### 重启后状态

| 数据层 | 状态 |
|--------|------|
| 块属性缓存 | ⚠️ 取决于磁盘写入结果 |
| 块属性磁盘 | ⚠️ 部分写入 |
| 卡包内存 | ✅ 从磁盘加载 |
| 卡包磁盘 | ✅ 已写入 |

---

### 2.4 场景 S0：全部成功

#### 运行期/重启后状态

| 数据层 | 运行期 | 重启后 |
|--------|--------|--------|
| 块属性缓存 | ✅ | ✅ |
| 块属性磁盘 | ✅ | ✅ |
| 卡包内存 | ✅ | ✅ |
| 卡包磁盘 | ✅ | ✅ |

---

## 三、ReviewFlashcard 的自动修复机制

### 3.1 机制说明

代码位置：[ReviewFlashcard#L467-L509](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L467-L509)

```go
func ReviewFlashcard(deckID, cardID string, rating riff.Rating, reviewedCardIDs []string) (err error) {
    deckLock.Lock()
    defer deckLock.Unlock()
    waitForSyncingStorages()

    deck := Decks[deckID]           // 从全局内存卡包读取
    card := deck.GetCard(cardID)     // 内存卡包中有该卡 → 正常获取

    // ... 撤销缓存逻辑 ...

    deck.Review(cardID, rating)      // 更新内存卡包中的卡片状态
    deck.Save()                      // 保存整个卡包到磁盘（含之前 AddCard 的卡片）
    deck.SaveLog(log)
    return
}
```

**关键点**：
- `deck.Save()` 保存的是**整个卡包**的所有数据，不是增量保存
- 如果 `doAddFlashcards` 中 Save 失败，但 `ReviewFlashcard` 中 Save 成功，则 AddCard 添加的卡片会被一并持久化
- 这是一种**隐式修复**机制：只要用户在运行期成功复习了该卡片，数据就不会丢失

### 3.2 自动修复的前提条件

| 条件 | 说明 |
|------|------|
| 卡片在内存卡包中 | `deck.AddCard()` 已执行，卡片在 `Decks[deckID]` 中 |
| 用户在运行期复习该卡 | 触发 `ReviewFlashcard` → `deck.Save()` |
| 复习时的 Save 成功 | 磁盘故障是暂时性的，不是持久性的 |

### 3.3 自动修复的时间窗口

```
deck.Save() 失败（制卡时）
    │
    │  ← 自动修复窗口：从现在到重启之前
    │     用户只要复习到这张卡片，Save 就会补存
    │
    ├─ 用户复习到该卡 → ReviewFlashcard → deck.Save() 成功 → ✅ 修复
    │
    └─ 重启 → LoadFlashcards → 卡片丢失 → ❌ 无法自动修复
```

### 3.4 自动修复的局限

| 局限 | 说明 |
|------|------|
| 仅限运行期 | 重启后内存卡包丢失，无法触发修复 |
| 依赖用户行为 | 如果用户不复习该卡片，Save 不会被触发 |
| 依赖磁盘恢复 | 如果磁盘故障持续，ReviewFlashcard 的 Save 也会失败 |
| 不修复块属性 | 如果是 S2 场景（rollback），块属性已丢失，卡包有卡片但块无标记 |

---

## 四、一致性风险的修正判定

### 4.1 会发生的一致性风险（修正版）

| ID | 风险 | 运行期表现 | 重启后表现 | 严重度 |
|----|------|-----------|-----------|--------|
| **R1** | deck.Save() 失败 | ✅ 复习正常（内存有卡片），可能自动修复 | ❌ 卡片丢失，块属性孤立 | 中 |
| **R2** | 事务 rollback | ✅ 复习正常（卡包已 Save），缓存与磁盘不一致 | ❌ 块无属性，卡包有卡片 | 高 |
| **R3** | commit 部分失败 | ✅ 复习正常（卡包已 Save） | ⚠️ 部分块有属性，部分无 | 中 |
| **R4** | pushBlockAttrs 时序差 | 搜索短暂超前 | 无影响 | 低 |
| **R5** | 已删除块卡片残留 | 调度时被 ExistBlockTrees 过滤 | 同左 | 低 |
| **R6** | 多端同步冲突 | 取决于同步时序 | 可能丢失/重复 | 中 |

### 4.2 不会发生的情况（修正版）

| ID | 常见误解 | 不会发生的原因 |
|----|----------|----------------|
| ❌ "deck.Save 失败后复习找不到卡片" | `deck.AddCard()` 已将卡片加入内存卡包 `Decks[deckID]`，复习接口 `deck.Dues()` 读取同一内存卡包，运行期一定能找到 |
| ❌ "deck.Save 失败后立即导致数据丢失" | 运行期数据完整，只有重启后才丢失。且复习时的 `deck.Save()` 可能自动补存 |
| ❌ "卡包写入失败会触发事务回滚" | `deck.Save()` 失败不返回错误，事务继续 |
| ❌ "rollback 会回滚卡包操作" | `rollback()` 仅清空 `tx.trees` 和 `tx.nodes` |
| ❌ "每日新卡配额按自然日重置" | 配额基于 `reviewCardCache`，回合级 |
| ❌ "块属性缓存未更新但卡包已写入" | `cache.PutBlockIAL()` 在 `deck.Save()` 之前 |

### 4.3 R1 的完整影响链

```
doAddFlashcards 中 deck.Save() 失败
│
├── 运行期影响：
│   ├── 卡包内存有卡片 → deck.Dues() 找到 → 复习正常 ✅
│   ├── 块属性缓存有标记 → 编辑器显示闪卡图标 ✅
│   ├── 块属性磁盘有标记 → 刷新页面后仍显示 ✅
│   └── ReviewFlashcard → deck.Save() → 可能补存 ✅/❌
│
├── 未补存 + 重启后影响：
│   ├── 卡包内存无卡片（从磁盘加载）→ deck.Dues() 找不到 ❌
│   ├── 块属性缓存有标记（从磁盘加载）→ 编辑器仍显示闪卡图标 ✅
│   ├── 块属性磁盘有标记 → 但卡包无对应卡片
│   └── 结果：块显示制卡标记，但复习列表中不出现该卡片
│
└── 已通过 Review 补存 + 重启后影响：
    ├── 卡包内存有卡片（从磁盘加载，含复习状态）→ 复习正常 ✅
    ├── 块属性缓存有标记 → 编辑器显示 ✅
    └── 完全一致 ✅
```

---

## 五、复习回合缓存配额 vs 间隔重复到期时间

### 5.1 核心区别

| 维度 | 复习回合缓存配额 | FSRS 间隔重复到期时间 |
|------|----------------|----------------------|
| **本质** | 内存计数器，控制单回合卡片吞吐量 | 基于算法的绝对时间点，决定卡片何时进入复习队列 |
| **数据来源** | `reviewCardCache`（全局 Map） | `fsrs.Card.Due` 字段 |
| **存储位置** | 内存 | 持久化到 `{deckID}.deck` JSON 文件 |
| **读取时机** | `getDeckDueCards` 每次获取到期卡片时 | `deck.Dues()` 筛选 `due ≤ now` |
| **生命周期** | 回合开始时清零；关闭窗口时丢失 | 卡片生命周期内持续，每次 Review 后更新 |
| **统计依据** | 缓存中卡片的**初始状态**（New/非 New） | FSRS 算法参数（Stability、Difficulty 等） |
| **影响范围** | 单次 API 返回多少张卡片 | 卡片是否出现在到期列表中 |
| **持久化** | 否，重启即失 | 是，写入磁盘 |
| **"每日"含义** | 实为"每回合"，关闭窗口即重置 | 真正的时间间隔（分钟/小时/天/月） |

### 5.2 reviewCardCache 的双重作用

代码位置：[flashcard.go#L488](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L488)

```go
reviewCardCache[cardID] = card.Clone()  // 首次评分前克隆
```

存储的是**首次评分前的状态快照**，服务于：
1. **撤销恢复**：重评时恢复到初始状态 [L479-L485](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L479-L485)
2. **配额统计**：按初始状态统计已用配额 [L1126-L1134](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1126-L1134)

**统计特性**：
- 按初始状态统计，不按当前状态
- 新卡复习后状态变为 Learning，但配额统计仍计入 newCount
- 配额统计**稳定**，不会因状态变化而波动

### 5.3 两者的协作流程

```
getDeckDueCards(deck, reviewedCardIDs, blockIDs, newCardLimit, reviewCardLimit, reviewMode)
│
├── ① deck.Dues()  ─────────────── FSRS 层：获取所有 due ≤ now 的卡片
│     └── 从内存卡包 Decks[deckID] 读取（非磁盘）
│
├── ② ExistBlockTrees 过滤 ─────── 过滤已删除块
│
├── ③ 配额统计 ─────────────────── 回合层：从 reviewCardCache 统计
│     ├── newCount = 缓存中初始状态为 New 的卡数
│     └── reviewCount = 缓存中初始状态非 New 的卡数
│
├── ④ 遍历 dues 按配额筛选 ────── 两者交汇
│     ├── newCount < newCardLimit → 接受 New 卡
│     ├── reviewCount < reviewCardLimit → 接受 Review/Learning/Relearning 卡
│     └── 跳过 skipCardCache 中的卡片
│
└── ⑤ 按 reviewMode 排序返回 ──── 新旧混合/新卡优先/旧卡优先
```

### 5.4 配额协作示例（新卡上限 20）

| 时间点 | 操作 | newCount | Due 总数 | 返回新卡数 |
|--------|------|----------|---------|-----------|
| T0 | 回合开始 | 0 | 50 | 20 |
| T1 | 复习 15 张新卡 | 15 | 35 | - |
| T2 | 翻页 | 15 | 35 | 补充 5 张 |
| T3 | 复习完 20 张 | 20 | 30 | 配额满 |
| T4 | 关闭窗口再打开 | 0 | 30 | 再返回 20 张 |

---

## 六、同步状态检测的两种策略

### 6.1 策略对照

| 策略 | 函数 | 行为 | 适用操作 |
|------|------|------|----------|
| 阻塞等待 | `waitForSyncingStorages()` | 每秒轮询直到同步完成 | 复习类操作（7 个） |
| 立即失败 | `isSyncingStorages()` | 立即返回 bool，不等待 | 制卡/删卡类操作（2 个） |

代码位置：[repository.go#L1210-L1217](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1210-L1217)

### 6.2 触发顺序问题

两种策略都是**先获取 deckLock，再检查同步状态**：

- [doAddFlashcards#L858-L861](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L858-L861)
- [ReviewFlashcard#L468-L471](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L468-L471)

同步期间所有闪卡操作先阻塞在 `deckLock.Lock()` 上，可能形成操作队列堆积。

---

## 七、删卡操作的一致性分析

### 7.1 删卡 S1：deck.Save() 失败

代码位置：[flashcard.go#L837-L855](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L837-L855)

**运行期**：
- 卡包内存已移除卡片 → `deck.Dues()` 不再返回 → 复习正常（卡片不再出现）✅
- 块属性缓存已移除 → 编辑器不显示闪卡图标 ✅
- 但卡包磁盘仍有卡片

**重启后**：
- 卡包从磁盘加载 → 卡片恢复出现 → 复习时该卡片重新出现 ❌
- 块属性已从磁盘移除 → 编辑器不显示闪卡图标 ❌
- **反向不一致**：复习时出现"已删除"的卡片

### 7.2 删卡 S2：rollback

**运行期**：
- 卡包内存已移除、磁盘已移除 → 复习不再出现 ✅
- 块属性缓存有标记 → 编辑器仍显示图标（缓存未清）

**重启后**：
- 卡包磁盘无卡片 → 复习不出现 ✅
- 块属性磁盘有标记 → 编辑器显示图标 → 但无对应卡片
- **反向不一致**：编辑器显示闪卡标记，但复习找不到

---

## 八、风险评估汇总（修正版）

### 8.1 数据一致性风险

| ID | 风险 | 运行期 | 重启后 | 自愈可能 |
|----|------|--------|--------|----------|
| **R1** | 制卡 Save 失败 | 正常（内存有卡片） | 卡片丢失，块属性孤立 | Review 的 Save 可补存 |
| **R2** | 事务 rollback | 正常（卡包已存） | 块无属性，卡包有卡片 | 无法自愈 |
| **R3** | commit 部分失败 | 正常 | 部分块关联断裂 | 无法自愈 |
| **R4** | 删卡 Save 失败 | 正常（内存已移除） | 卡片恢复出现 | 删卡无 Review 补存机制 |
| **R5** | 删卡 rollback | 缓存有标记 | 块有属性但卡包无卡片 | 无法自愈 |
| **R6** | 已删除块卡片残留 | 调度时过滤 | 卡包膨胀 | 无法自愈 |

### 8.2 设计风险

| ID | 风险 | 严重度 | 说明 |
|----|------|--------|------|
| **D1** | deck.Save() 失败静默处理 | 中 | 仅记录日志，不返回错误 |
| **D2** | 闪卡操作不在事务原子性范围内 | 高 | rollback 不回滚卡包操作 |
| **D3** | 删卡无 Review 补存机制 | 低 | 制卡有隐式修复（Review Save 补存），删卡没有 |

### 8.3 时间边界风险

| ID | 风险 | 严重度 | 说明 |
|----|------|--------|------|
| **T1** | "每日配额"实为"每回合配额" | 低 | 关闭窗口即重置 |
| **T2** | 配额按初始状态统计 | 低 | 设计如此，稳定性优先 |
| **T3** | 跨时区 due 显示差异 | 低 | 服务端时区计算 |

---

## 九、后续验证要点

### 9.1 一致性验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V1 | R1 运行期验证 | 制卡 → 模拟 Save 失败 → 复习 | 复习能找到卡片（内存卡包有） |
| V2 | R1 重启后验证 | V1 后重启 → 复习 | 卡片丢失（磁盘无） |
| V3 | R1 自动修复验证 | V1 后 → 复习该卡（Review Save 成功）→ 重启 | 卡片存在（Review 补存） |
| V4 | R2 重启后验证 | 制卡 → rollback → 重启 | 块无属性，卡包有卡片 |
| V5 | R4 重启后验证 | 删卡 → Save 失败 → 重启 | 卡片恢复出现 |
| V6 | D1 静默处理验证 | 模拟 Save 失败 → 检查 API 返回 | API 返回成功 |

### 9.2 配额 vs 到期时间验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V7 | 配额按初始状态统计 | 复习 10 张新卡 → 翻页 | newCount 仍为 10 |
| V8 | 关闭窗口配额重置 | 复习 15 张 → 关闭 → 重开 | 返回 20 张 |
| V9 | 到期时间持久化 | 复习记 due → 重启 → 查 due | due 不变 |
| V10 | 内存卡包 vs 磁盘卡包 | 制卡 Save 失败 → 复习正常 → 重启 → 卡片丢失 | 运行期读内存，重启读磁盘 |

### 9.3 同步状态验证

| 编号 | 验证项 | 测试方法 | 预期结果 |
|------|--------|----------|----------|
| V11 | 制卡同步中立即失败 | 启动同步 → 制卡 | 返回错误 |
| V12 | 复习同步中阻塞等待 | 启动同步 → 评分 | 等待后成功 |

---

## 十、关键代码索引

### 10.1 制卡核心

| 函数 | 文件 | 行号 |
|------|------|------|
| `doAddFlashcards` | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L857-L949) | L857-L949 |
| `doRemoveFlashcards` | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L747-L771) | L747-L771 |
| `removeFlashcardsByBlockIDs` | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L837-L855) | L837-L855 |
| `deck.Save()` 失败静默处理 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L944-L947) | L944-L947 |

### 10.2 复习与自动修复

| 函数 | 文件 | 行号 |
|------|------|------|
| `ReviewFlashcard`（含 deck.Save 补存） | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L467-L509) | L467-L509 |
| `getDeckDueCards`（读内存卡包） | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1093-L1197) | L1093-L1197 |
| `LoadFlashcards`（重启从磁盘加载） | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L951-L985) | L951-L985 |

### 10.3 事务与回滚

| 函数 | 文件 | 行号 |
|------|------|------|
| `performTx` | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L148-L349) | L148-L349 |
| `tx.commit()` | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1865-L1895) | L1865-L1895 |
| `tx.rollback()` | [transaction.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/transaction.go#L1897-L1902) | L1897-L1902 |

### 10.4 配额与到期时间

| 概念 | 代码位置 | 行号 |
|------|----------|------|
| 配额统计 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1126-L1134) | L1126-L1134 |
| 到期筛选 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L1097) | L1097 |
| 撤销缓存写入 | [flashcard.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/flashcard.go#L488) | L488 |

### 10.5 同步状态检测

| 函数 | 文件 | 行号 |
|------|------|------|
| `waitForSyncingStorages` | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1210-L1213) | L1210-L1213 |
| `isSyncingStorages` | [repository.go](file:///d:/fz/0601/solo-dogfeeding/code/294-siyuan/kernel/model/repository.go#L1216-L1217) | L1216-L1217 |
