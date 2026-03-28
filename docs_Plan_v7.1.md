# Plan v7.1: AITTK 全域 AI 辅助决策系统开发计划

## TL;DR

为《杀戮尖塔2》开发全域 AI 辅助决策插件，覆盖 **出牌、路线、选卡、事件、药水、遗物** 六大决策域。
战斗内出牌推荐采用 **规则引擎实时计算**；非战斗域采用 **规则引擎即时基线 + LLM 异步辅助** 的双阶段模式。
各域遵循统一的 **Collector → Snapshot → Recommender → Overlay** 四层架构，通过 **RunContext** 共享全局运行状态，域间不直接耦合。

选卡推荐采用 **威胁驱动（Threat-Resolution）** 评估体系，而非流派匹配；RunContext 具备 **Hydration（状态重建）** 机制以应对存档恢复；知识库采用 **事实数据 + 策略类** 分离架构；UI 采用 **栈式状态机** 管理多层 Overlay。

---

## 1. 产品目标与范围

### 1.1 产品目标

在玩家游戏过程中的各个关键决策点，基于当前状态输出：

- 推荐的行动方案
- 推荐目标
- 推荐理由
- 推荐强度 / 置信度

结果以游戏内浮层 UI 呈现，不自动执行任何游戏操作。

**六大决策域：**

| 决策域 | 触发时机 | 决策内容 |
|---|---|---|
| 出牌推荐 | 战斗回合内 | 出哪些牌、什么顺序、打谁 |
| 路线推荐 | 地图界面选择下一节点时 | 走哪条路 |
| 卡牌奖励推荐 | 战斗胜利后选卡时 | 选哪张卡 / 是否跳过 |
| 事件推荐 | 进入事件房间时 | 选哪个选项 |
| 药水推荐 | 获得药水 / 战斗前 | 持有/使用/丢弃建议 |
| 遗物价值评估 | 获得遗物选择时 | 遗物对当前构筑的价值 |

### 1.2 首期范围

本期覆盖：

- 战斗状态采集与单回合出牌推荐
- 游戏内浮层展示（所有域）
- 规则引擎推荐（所有域）
- 异步战术复盘（LLM）
- RunContext 全局运行上下文（含 Hydration 机制）
- 路线推荐
- 卡牌奖励推荐（威胁驱动评估）
- 事件推荐（事实数据 + 策略类 + LLM 兜底）
- 药水推荐
- 遗物价值评估（标签 + 策略类）
- 非战斗域 LLM 异步辅助

### 1.3 非目标

当前版本明确不做：

- 自动出牌 / 自动打牌
- 自动选择任何选项（所有域均为纯建议）
- 多回合全局搜索
- 完整复刻战斗结算引擎
- 将 LLM 接入实时战斗推荐主链路（非战斗域允许 LLM 异步辅助）
- 修改游戏逻辑或干预战斗执行
- 完整复刻所有事件效果（首期基于知识库 + LLM 兜底）

### 1.4 设计原则

- **实时性优先**：战斗内推荐必须立即可用
- **稳定性优先**：推荐结果不能高频抖动
- **低打扰优先**：UI 不遮挡关键游戏信息
- **解释性优先**：推荐必须能给出可读理由
- **规则优先**：规则引擎是所有域的即时基线
- **威胁驱动**：选卡/遗物评估以生存缺口为首要考量，而非流派匹配
- **有限复刻**：只复刻少量核心数学修正项，不重建完整引擎
- **热路径低分配**：搜索与评估路径尽量近零分配，避免 GC 尖刺
- **优雅降级**：任何子系统失败不应导致游戏崩溃或 Mod 不可用
- **域间解耦**：各域通过 RunContext 间接通信，不直接耦合
- **数据与逻辑分离**：知识库 JSON 仅存储事实/标签，评估逻辑在 C# 策略类中

---

## 2. 当前基础与已确认信息

### 2.1 用户决策记录

- AI 策略：规则引擎为主，本地 LLM 为辅
- UI 方式：游戏内浮层 UI
- 首选功能：出牌推荐（扩展至全域）
- 当前状态：CombatCollector 日志验证通过

### 2.2 已完成内容

- Mod 骨架：csproj + Entry.cs + AITTK.json
- BaseLib 集成：NuGet 引用 + 游戏 mods 目录安装
- CombatCollector：已订阅 CombatManager 事件并验证日志
- 启动配置：--log + steam_appid.txt
- 敌人意图采集：IntentType + 攻击伤害
- 完整战斗数据采集：buff/debuff、卡牌伤害/格挡值、遗物、药水、运行上下文
- 全量 API 逆向完成：基于完整 ILSpy 反编译
- 编译状态：0 错误、0 警告

### 2.3 已验证游戏版本

| 字段 | 值 |
|---|---|
| 游戏版本 | *(填入当前验证通过的版本号)* |
| BaseLib 版本 | *(填入当前使用的 BaseLib 版本)* |
| 验证日期 | *(填入最近一次完整验证日期)* |

> 每次游戏更新后需重新验证关键 API 可用性，详见 §2.5。

### 2.4 已确认 API

以下 API 已确认可作为各域基础：

**CombatManager / CombatState**

- `CombatManager.Instance`
  - Events: `CombatSetUp`, `TurnStarted`, `TurnEnded`, `CombatEnded`, `CombatWon`, `CreaturesChanged`
- `CombatState`: `CurrentSide`, `RoundNumber`, `Players`, `Enemies`, `Allies`, `Creatures`, `HittableEnemies`
  - `RunState`

**Player / PlayerCombatState**

- `Player`: `Relics`, `Potions`, `Gold`, `MaxEnergy`, `Deck`, `Character`
- `PlayerCombatState`: `Hand`, `DrawPile`, `DiscardPile`, `ExhaustPile`, `PlayPile`, `Energy`, `MaxEnergy`, `Stars`, `OrbQueue`, `Pets`
  - `HasEnoughResourcesFor(card, out reason)`

**CardModel**

- `Title`, `EnergyCost`, `Type`, `TargetType`, `Rarity`, `Keywords`, `Tags`
- `IsUpgraded`, `CurrentUpgradeLevel`, `Enchantment`, `Affliction`, `GainsBlock`, `IsBasicStrikeOrDefend`
- `DynamicVars`:
  - 常见 key: `Damage`, `Block`, `Heal`, `Energy`, `Gold`, `HpLoss`
  - Power key: `PoisonPower`, `StrengthPower`, `DexterityPower`, `WeakPower`, `VulnerablePower`, `DoomPower`

**Creature / Powers**

- `Creature`: `CurrentHp`, `MaxHp`, `Block`, `Name`, `Monster`, `Powers`, `IsStunned`, `IsHittable`, `IsPet`
- `PowerModel`: `Type`, `StackType`, `Amount`, `DisplayAmount`, `Title`, `Owner`, `DynamicVars`

**敌人意图**

- `MonsterModel.NextMove` → `MoveState.Intents` → `IReadOnlyList<AbstractIntent>`
- `AbstractIntent`: `IntentType`, `IntentTitle`
- `AttackIntent`: `GetSingleDamage(...)`, `GetTotalDamage(...)`, `Repeats`

**Relic / Potion**

- `RelicModel`: `Title`, `Description`, `Status`, `DynamicVars`, `IsWax`, `IsMelted`
- `PotionModel`: `Title`, `Rarity`, `Usage`, `TargetType`, `DynamicVars`, `IsQueued`

**RunState / Map / Event / Reward**

- `IRunState`: `Acts`, `CurrentActIndex`, `Act`, `Map` (`ActMap`), `CurrentMapCoord`, `CurrentMapPoint`, `ActFloor`, `TotalFloor`, `AscensionLevel`, `Players`, `Modifiers`
- `ActMap`: `Grid[col,row]`, `GetAllMapPoints()`, `GetPointsInRow(row)`, `BossMapPoint`, `StartingMapPoint`
- `MapPoint`: `coord` (`MapCoord`), `PointType` (`MapPointType`), `Children`, `parents`
- `MapPointType` enum: `Unassigned`, `Unknown`, `Shop`, `Treasure`, `RestSite`, `Monster`, `Elite`, `Boss`, `Ancient`
- `MapCoord`: `col`, `row` (int)
- `EventModel`: `Title`, `Owner`, `CurrentOptions` (`IReadOnlyList<EventOption>`), `DynamicVars`, `IsFinished`
- `RewardType` enum: `None`, `Card`, `Gold`, `Potion`, `Relic`, `RemoveCard`, `SpecialCard`
- `RewardsSet`: `Rewards` (`List<Reward>`), `Player`, `Room`

### 2.5 游戏版本兼容性策略

《杀戮尖塔2》处于 Early Access 阶段，API 随版本更新可能发生不兼容变化。

**启动时兼容性检查：**

- 在 `Entry.cs` 初始化阶段，通过反射探测关键 API 的可用性
- 检查通过 → 正常加载所有模块
- 检查失败 → 禁用推荐功能，在日志和游戏内显示友好提示，Mod 本身不崩溃

**兼容性检查清单：**

| 检查项 | 检查方式 | 影响域 |
|---|---|---|
| `CombatManager.Instance` 可访问 | 反射检查属性存在性 | 出牌 |
| `CombatState` 核心属性完整 | 反射检查 `Players`, `Enemies`, `RoundNumber` | 出牌 |
| `PlayerCombatState.Hand` 可访问 | 反射检查属性 | 出牌 |
| `MonsterModel.NextMove` 可访问 | 反射检查属性 | 出牌 |
| `CardModel.DynamicVars` 可访问 | 反射检查属性 | 出牌/选卡 |
| `IRunState.Map` 可访问 | 反射检查属性 | 路线 |
| `ActMap.GetAllMapPoints()` 可访问 | 反射检查方法 | 路线 |
| `EventModel.CurrentOptions` 可访问 | 反射检查属性 | 事件 |
| `RewardsSet.Rewards` 可访问 | 反射检查属性 | 选卡 |
| `Player.Deck` 可访问 | 反射检查属性 | 选卡/遗物/RunContext |

**按域降级：** 若某个域的 API 检查失败，仅禁用该域的推荐，其他域正常工作。

**维护策略：**

- 在仓库 README 中维护"已验证游戏版本"列表
- 游戏大版本更新后，优先运行兼容性检查 + Golden Cases 回归

新文件：

- `Scripts/Compatibility/ApiCompatibilityChecker.cs`

---

## 3. 总体架构

### 3.1 分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                          UI 层                                   │
│  CombatOverlay / RouteOverlay / CardRewardOverlay / EventOverlay │
│  PotionOverlay / RelicOverlay / ReviewPanel                      │
│  OverlayManager (栈式状态机)                                     │
├───────────────────────────────────────────���─────────────────────┤
│                        Manager 层                                │
│              RecommendationManager (出牌)                        │
│              DomainRecommendationManager (非战斗域统一管理)       │
├─────────────────────────────────────────────────────────────────┤
│                     Recommendation 层                            │
│  [出牌] CandidateGenerator → Evaluator → Ranker                 │
│  [路线] RouteRecommender                                        │
│  [选卡] CardRewardRecommender (威胁驱动)                         │
│  [事件] EventRecommender (事实数据 + 策略类)                     │
│  [药水] PotionRecommender                                       │
│  [遗物] RelicRecommender (标签 + 策略类)                         │
├─────────────────────────────────────────────────────────────────┤
│                      Analysis 层                                 │
│                    CombatAnalyzer                                │
│                  DeckFunctionalAnalyzer                          │
│                  ThreatWindowAnalyzer                            │
├─────────────────────────────────────────────────────────────────┤
│                      Snapshot 层                                 │
│  CombatSnapshot / RouteSnapshot / CardRewardSnapshot             │
│  EventSnapshot / PotionSnapshot / RelicSnapshot                  │
├─────────────────────────────────────────────────────────────────┤
│                      RunContext 层                                │
│        RunContextCollector + RunContext (含 Hydration)            │
├────────��────────────────────────────────────────────────────────┤
│                      Collector 层                                │
│  CombatCollector / RouteCollector / CardRewardCollector           │
│  EventCollector / PotionCollector / RelicCollector                │
├─────────────────────────────────────────────────────────────────┤
│                    Compatibility 层                               │
│                  ApiCompatibilityChecker                         │
├─────────────────────────────────────────────────────────────────┤
│                      Logging 层                                  │
│                      AITTKLogger                                │
└─────────────────────────────────────────────────────────────────┘

    [异步旁路] Review 层：LLMReviewClient / LLMDomainAdvisor / TacticalReviewService
```

### 3.2 多域统一架构原则

所有推荐域遵循统一的四层模式：

1. **Collector**：从游戏 runtime 采集该域的原始数据
2. **Snapshot**：转换为只读纯数据对象
3. **Analyzer / Recommender**：生成推荐
4. **Overlay**：在游戏内展示

各域之间**不直接耦合**。需要跨域信息时，统一通过 `RunContext` 获取。

**性能分级：**

| 域 | 时间压力 | 计算位置 | LLM 参与 |
|---|---|---|---|
| 出牌推荐 | 实时（<20ms） | 后台线程 | ❌ |
| 路线推荐 | 低压（秒级） | 主线程或后台 | ✅ 异步辅助 |
| 卡牌奖励 | 低压（秒级） | 主线程或后台 | ✅ 异步辅助 |
| 事件推荐 | 低压（秒级） | 主线程或后台 | ✅ 异步辅助 |
| 药水推荐 | 低压 | 主线程 | ⚠️ 可选 |
| 遗物评估 | 低压（秒级） | 主线程或后台 | ✅ 异步辅助 |

### 3.3 实时推荐数据流（出牌）

```
CombatManager Events
   ↓
CombatCollector
   ↓
CombatSnapshot (Raw, 含 SchemaVersion)
   ↓
CombatAnalyzer
   ↓
DerivedCombatAnalysis
   ↓
CandidateSequenceGenerator (Beam Search + 粗排)
   ↓
ActionSequenceEvaluator (精排)
   ↓
RuleBasedRecommender
   ↓
RecommendationManager
   ↓
主线程 UI 轮询并刷新
   ↓
CombatOverlay
```

### 3.4 非战斗域推荐数据流（通用）

```
游戏场景事件（进入地图/选卡/事件/获得药水/获得遗物）
   ↓
域 Collector
   ↓
域 Snapshot + RunContext
   ↓
域 Recommender（规则引擎，即时出结果）
   ↓
域 Overlay（立即显示规则推荐）
   ↓  （同时，异步旁路）
LLMDomainAdvisor（后台异步调用）
   ↓
LLM 返回后更新 Overlay（叠加 LLM 分析，标注"AI 分析"标签）
```

### 3.5 异步复盘数据流

```
CombatSnapshot / ActualPlayedSequence / Recommendation
   ↓
LLMPromptBuilder (构建精简 Prompt)
   ↓
LLMReviewClient
   ↓
TacticalReviewService
   ↓
主线程 UI 显示复盘点评
```

---

## 4. 线程模型��并发约束

### 4.1 总原则

- 游戏 runtime object 只允许主线程访问
- Godot SceneTree / Node 只允许主线程访问
- 后台线程只处理纯 C# 数据对象
- 后台线程禁止直接修改 UI 节点

### 4.2 推荐计算线程边界

**主线程负责：**

- 监听游戏事件
- 采样并构建所有域的 Snapshot
- 管理 UI 生命周期
- 非战斗域的规则推荐计算（低压，可在主线程完成）
- RunContext Hydration（需要访问游戏 runtime object）

**后台线程可负责：**

- 基于 CombatSnapshot 的分析与出牌推荐
- LLM 异步调用（所有域）
- 路线前瞻搜索（如计算量较大）

### 4.3 结果发布模型

采用 **不可变结果对象 + 原子交换** 模式：

- 推荐结果封装为不可变对象
- 后台线程完成计算后，通过 `Interlocked.Exchange` 发布最新结果
- UI 主线程在 `_Process` 中读取最新引用并比对版本号
- 若版本变化，则刷新 UI

**UI 引用捕获规则：**

UI 线程在 `_Process` 方法的**第一行**必须将全局引用捕获为局部变量，后续所有读取操作均基于此局部引用，**禁止在同一帧内多次读取全局字段**。

`RecommendationSnapshot` 及所有域的推荐结果对象内部集合使用 `ImmutableArray<T>`，确保真正不可变。

### 4.4 并发实现约束

- 不使用高频 `lock(obj)` 保护 UI 读取路径
- 推荐结果对象必须不可变
- 结果对象需携带：`Version`、`GenerationId`、`Timestamp`
- 若存在多次并发计算，仅接受最新 `GenerationId` 的结果
- 后台推荐计算通过 `CancellationToken` 支持取消

### 4.5 失败与降级策略矩阵

| 失败场景 | 处理方式 | 用户可见行为 | 日志级别 |
|---|---|---|---|
| API 兼容性检查失败（全局） | 禁用全部推荐模块 | 提示"AITTK: 当前游戏版本不兼容" | Error |
| API 兼容性检查失败（单域） | 禁用该域推荐，其他域正常 | 该域 Overlay 不显示 | Warn |
| Snapshot 构建异常 | 捕获异常，跳过本次推荐 | UI 保持上次结果；连续 3 次失败则清空 | Warn / Error |
| 后台推荐计算超时 | 返回当前最优候选 | 轻微延迟，推荐正常显示 | Warn |
| 后台推荐线程未处理异常 | 捕获异常，丢弃本次结果 | UI 不更新，保持上次结果 | Error |
| LLM（Ollama）不可用 | 禁用 LLM 辅助（所有域） | 复盘显示"不可用"；非战斗域仅显示规则推荐 | Warn（首次） |
| LLM 单次超时 | 丢弃本次 LLM 结果 | 仅显示规则推荐 | Debug |
| Snapshot JSON 反序列化失败 | 跳过该 case | 回归测试输出跳过原因 | Warn |
| Snapshot SchemaVersion 不兼容 | 尝试 migration；不可迁移则拒绝 | 回归测试报警 | Error |
| UI 节点不存在或已被释放 | `IsInstanceValid()` 检查后安全退出 | 无崩溃 | Warn |
| 推荐计算连续 5 次超过性能目标 | 触发自适应退化（见 §7.5） | 推荐可能略简化 | Warn |
| 事件知识库未收录 + LLM 不可用 | 显示事件选项的客观效果描述（如果有） | 显示"暂无评估建议" | Info |
| RunContext 状态同步失败 | 使用上次有效的 RunContext | 非战斗域推荐可能略过时 | Warn |
| RunContext 为空或未初始化 | 触发全量 Hydration 重建 | 非战斗域推荐短暂延迟 | Warn |
| RunContext 一致性校验失败 | 触发全量 Hydration 重建 | 推荐结果可能短暂刷新 | Warn |
| Hydration 重建失败 | 禁用非战斗域推荐 | 仅出牌推荐可用 | Error |
| Overlay 栈状态异常（栈深度超限） | 清空栈并重新 Push 当前场景 Overlay | Overlay 短暂闪烁后恢复 | Warn |

**实现原则：**

- 所有后台线程入口必须包裹 `try-catch`，禁止异常逃逸
- UI 层所有节点操作前必须 `IsInstanceValid()` 检查
- 连续失败需要计数器追踪，避免无限重试或日志洪泛
- 首次失败打日志，后续相同类型失败做节流（60 秒内同类型最多 3 条）

---

## 5. 数据模型设计

### 5.1 分层

中间模型拆分为两层：

- **Raw Snapshot**：忠实记录当前原始状态，不混入策略判断
- **Derived Analysis**：从 Raw Snapshot 中计算推荐所需的衍生特征

### 5.2 Snapshot 版本化

所有域的 Snapshot 均携带 `SchemaVersion` 字段（整数递增）。

- 序列化时写入当前 `SchemaVersion`
- 反序列化时检查版本兼容性
- 每次修改 Snapshot 字段结构时，必须递增 `SchemaVersion`

**轻量 Migration 机制：**

`SnapshotMigrator` 负责逐版本递增迁移。

**Migration 规则：**

| 变更类型 | 是否可自动迁移 | 处理方式 |
|---|---|---|
| 新增可选字段 | ✅ | 填默认值 |
| 枚举新增成员 | ✅ | 历史数据回退 `Unknown` |
| 字段重命名 | ✅ | Migration hook 映射 |
| 字段类型变更 | ❌ | 拒绝加载 |
| 移除必需字段 | ❌ | 拒绝加载 |
| 嵌套结构重组 | 视情况 | 按具体情况编写 migration |

新文件：`Scripts/Debug/SnapshotMigrator.cs`

### 5.3 出牌域模型

```
CombatSnapshot                      (含 SchemaVersion)
├── PlayerStateInfo
├── EnemyStateInfo[]
├── CardInstanceInfo[]
├── PowerInfo[]
├── RelicInfo[]
├── PotionInfo[]
├── IncomingIntentInfo[]
└── CombatContextInfo

DerivedCombatAnalysis
├── TotalExpectedIncomingDamage
├── PlayableCards[]
├── EstimatedDamageMap
├── EstimatedBlockMap
├── HighThreatTargets[]
├── AoeOpportunity
├── NondeterministicActionFlags
└── CoreModifierSummary

PlayRecommendation
├── ActionSequence: RecommendedAction[]
├── Score
├── Confidence
├── Reasons: RecommendationReason[]
└── NondeterministicWarnings[]

RecommendationSnapshot
├── Version
├── GenerationId
├── Timestamp
├── Recommendations: ImmutableArray<PlayRecommendation>
└── ComputationTimeMs
```

### 5.4 RunContext 模型

```
RunContext                           (含 SchemaVersion)
├── DeckComposition
│   ├── TotalCardCount: int
│   ├── AttackCount / SkillCount / PowerCount: int
│   ├── AverageEnergyCost: float
│   ├── ScalingCardCount: int
│   └── FunctionalProfile
│       ├── SingleTargetDamagePerTurn: float
│       ├── AoeDamagePerTurn: float
│       ├── BlockPerTurn: float
│       ├── ScalingPotential: float
│       ├── DrawPower: float
│       ├── HasReliableAoe: bool
│       ├── HasBurstDamage: bool
│       └── HasSufficientDefense: bool
├── RelicInventory: RelicInfo[]
├── PotionInventory: PotionInfo[]
├── PlayerRunState
│   ├── CurrentHp / MaxHp: int
│   ├── Gold: int
│   └── AscensionLevel: int
├── RunProgress
│   ├── CurrentAct / CurrentFloor: int
│   ├── FloorsToNextBoss: int
│   └── IsEliteNext / IsBossNext: bool
├── MapState
│   ├── VisiblePaths: MapPath[]
│   └── VisitedPoints: MapCoord[]
├── HistoricalPerformance
│   ├── AverageHpLossPerCombat: float
│   ├── AverageTurnsPerCombat: float
│   └── EliteWinRate: float
├── ThreatWindow
│   ├── NextEliteDistance: int?
│   ├── NextBossDistance: int
│   ├── KnownUpcomingEliteTypes: string[]
│   ├── SurvivalGaps: SurvivalGapAnalysis
│   │   ├── NeedMoreBlock: bool
│   │   ├── NeedMoreSingleTargetDamage: bool
│   │   ├── NeedMoreAoe: bool
│   │   ├── NeedMoreScaling: bool
│   │   └── NeedBurstDamage: bool
│   └── ThreatLevel: ThreatLevel (Low / Medium / High / Critical)
├── IsHydrated: bool
├── HydrationSource: string ("EventDriven" / "FullRebuild")
├── LastValidatedTimestamp: long
└── IsHistoricalDataApproximate: bool
```

### 5.5 非战斗域模型

```
RouteSnapshot
├── CurrentCoord: MapCoord
├── AvailableNextPoints: MapPointInfo[]
├── LookaheadPaths: MapPathInfo[]      (2-3 层前瞻)
└── RunContext (引用)

CardRewardSnapshot
├── AvailableCards: CardInfo[]
├── CanSkip: bool
└── RunContext (引用)

EventSnapshot
├── EventTitle: string
├── EventOptions: EventOptionInfo[]
└── RunContext (引用)

PotionSnapshot
├── CurrentPotions: PotionInfo[]
├── NewPotion: PotionInfo?
├── PotionSlotsFull: bool
└── RunContext (引用)

RelicSnapshot
├── AvailableRelics: RelicInfo[]
├── IsShopPurchase: bool
├── Cost: int?
└── RunContext (引用)
```

### 5.6 字段设计原则

**原始字段**（卡牌示例）：

- 标题、费用、类型、目标类型、keywords、DynamicVars、PreviewValue
- 当前所在区域（手牌/弃牌/抽牌堆等）

**推导字段**（卡牌示例）：

- EstimatedBaseDamage / EstimatedBaseBlock
- IsPlayable / UnplayableReason
- RequiresTarget / IsZeroCost
- ProvidesDefense / ProvidesDraw / ProvidesScaling
- HasOrderDependency / ChangesHandStructure / IntroducesNonDeterminism

**推导字段**（敌人示例）：

- ExpectedIncomingDamage / WillAttackThisTurn
- ThreatScore / KillPriority
- HasCoreModifierRelevantState

---

## 6. Phase 1：状态标准化与采集完善

### 6.1 目标

将已有 CombatCollector 升级为稳定的标准状态层，为推荐器、UI、回放和异步复盘提供统一输入。

### 6.2 工作项

**步骤 1.1：API 逆向确认** — ✅ 已完成

**步骤 1.2：全量战斗采集** — ✅ 已完成

**步骤 1.3：建立标准化快照**

- 创建 `CombatSnapshot`（含 `SchemaVersion`）
- 将 runtime object 转换��纯数据对象
- 禁止推荐层直接依赖游戏对象
- 新文件：
  - `Scripts/Models/CombatSnapshot.cs`
  - `Scripts/Models/PlayerStateInfo.cs`
  - `Scripts/Models/EnemyStateInfo.cs`
  - `Scripts/Models/CardInstanceInfo.cs`
  - `Scripts/Models/PowerInfo.cs`
  - `Scripts/Models/CombatContextInfo.cs`

**步骤 1.4：建立分析层**

- 分析内容：总预计承伤、可出牌集合、基础伤害/格挡估值、高威胁目标、AOE 机会、非确定性动作标记、核心修正项状态摘要
- 新文件：
  - `Scripts/Models/DerivedCombatAnalysis.cs`
  - `Scripts/AI/CombatAnalyzer.cs`

**步骤 1.5：快照序列化与离线回放支持**

- CombatSnapshot JSON 导出 / 导入
- 序列化/反序列化时校验 `SchemaVersion`
- 不兼容版本尝试 migration
- 新文件：
  - `Scripts/Debug/CombatSnapshotSerializer.cs`
  - `Scripts/Debug/SnapshotMigrator.cs`

**步骤 1.6：建立统一日志层**

- 统一管理日志级别（Trace / Debug / Info / Warn / Error）
- 同类型失败日志节流（60 秒内同类型最多 3 条）
- 新文件：`Scripts/Debug/AITTKLogger.cs`

**步骤 1.7：启动时 API 兼容性检查**

- 执行 §2.5 兼容性检查，支持按域降级
- 新文件：`Scripts/Compatibility/ApiCompatibilityChecker.cs`

### 6.3 验证标准

- 相同状态可生成稳定结构快照
- 快照可序列化 / 反序列化，含 SchemaVersion 校验
- 历史版本快照可通过 migration 自动升级
- 推荐层不依赖 runtime object
- 日志层可正常输出各级别日志
- API 兼容性检查可按域通过/拒绝

---

## 7. Phase 2：规则推荐核心（出牌）

### 7.1 目标

实现一个 **稳定、合法、低 GC、可调参、可回归测试** 的实时规则推荐器。

### 7.2 核心原则

- 推荐对象是 **候选出牌序列**
- 不实现完整战斗引擎复刻
- 优先使用游戏已暴露的 PreviewValue / DynamicVars
- 对少数高频核心乘区做 **有限复刻**
- 对非确定性未来状态采用 **截断 + 启发式未来收益**
- 搜索热路径遵守 **近零分配**
- 评估器定位为**近似前向评估器（Approximate Forward Evaluator），仅追踪 N 个核心修正项（首期 N ≤ 4）**

### 7.3 明确不做

- 不在推荐器中复刻全部遗物 / 怪物 / 稀有卡牌逻辑
- 不尝试在实时路径中处理全量随机未来状态
- 不在热路径频繁分配 `List<>` / `Dictionary<>`
- 评估器中追踪的核心修正项不超过 4 个，新增需显式评审

### 7.4 工作项

**步骤 2.1：定义推荐接口**

输入：`CombatSnapshot` + `DerivedCombatAnalysis`
输出：`List<PlayRecommendation>`

每条推荐包含：出牌序列、目标、分数、置信度、理由列表、非确定性警告列表

新文件：

- `Scripts/AI/IPlayRecommender.cs`
- `Scripts/AI/PlayRecommendation.cs`

**步骤 2.2：回合价值函数**

**组件职责边界（全局适用）：**

| 组件 | 唯一职责 | 明确不做 |
|---|---|---|
| `CandidateSequenceGenerator` | 在合法性约束下生成候选出牌序列 | 不做评分、不做 UI、不做线程控制 |
| `ActionSequenceEvaluator` | 对候选序列打分（精排） | 不做生成、不做排序、不做缓存 |
| `FastHeuristic` | 搜索展开阶段快速打分（粗排） | 不做完整评估、不追踪修正项 |
| `ValueFunction` | 定义和计算各评分维度 | 不做搜索、不做截断判断 |
| `RuleBasedRecommender` | 编排 Generator + Evaluator，产出推荐列表 | 不做 UI、不做线程控制 |
| `RecommendationManager` | 生命周期、缓存、取消、发布、去抖 | 不做具体评分、不做 UI |
| `CombatOverlay` | 读取并展示推荐结果 | 不做计算、不做数据采集 |

评分维度：

| 维度 | 说明 | 默认权重 |
|---|---|---|
| 生存价值 | 减少本回合预期承伤、避免致死 | 1.0 |
| 输出价值 | 对高威胁目标造成有效伤害 | 0.8 |
| 斩杀价值 | 本回合击杀敌人 | 1.2 |
| 状态价值 | Buff/Debuff/易伤/虚弱等收益 | 0.6 |
| 资源价值 | 抽牌、回能、0费连锁、保留关键资源 | 0.5 |
| 惩罚项 | 过量防御、能量浪费、无效伤害、错误顺序等 | -（各项独立） |

权重在 DevConfig 可配置。考虑战斗阶段差异（Boss 战生存权重上调等）。

新文件：`Scripts/AI/Evaluation/ValueFunction.cs`

**步骤 2.3：候选序列生成器（两阶段搜索）**

**阶段一：粗排（Fast Heuristic）**

| 粗排特征 | 计算成本 | 说明 |
|---|---|---|
| 能量是否用满 | 极低 | 剩余能量越多越差 |
| 是否防止致死 | 低 | 承伤 > HP 时防御牌加分 |
| 是否达成斩杀 | 低 | 敌人 HP 归零大幅加分 |
| 是否明显浪费 | 极低 | 过量格挡、对死亡敌人出牌 |
| 是否触发非确定性截断 | 极低 | 截断节点给予保守估分 |

粗排目标：**单次调用 < 0.5μs**，零堆分配。

新文件：`Scripts/AI/Evaluation/FastHeuristic.cs`

**阶段二：精排（Detailed Evaluation）**

仅对 Beam Search 输出的最终候选集合（10~15 个）执行完整评估。

**等价目标剪枝（Symmetry Breaking）：**

| 维度 | 条件 |
|---|---|
| 当前 HP | 相同 |
| 当前 Block | 相同 |
| 本回合意图类型 | 相同 |
| 意图伤害值 | 相同 |
| 关键 Debuff 状态 | Vulnerable / Weak / Poison 的 Amount 相同 |

**CancellationToken 批次检查：** 每 32 个节点检查一次。

**首期参数默认值（后续可根据 profiling 调整）：**

| 参数 | 默认值 | 说明 |
|---|---|---|
| Beam Width | 5 | 每层保留的 Top-K 候选 |
| 最大搜索深度 | 10 | 通常等于手牌上限 |
| 最大候选节点数 | 500 | 含等价剪枝后的安全冗余 |
| 搜索内部时间预算 | 15ms | 超时返回当前最优 |
| 精排候选上限 | 15 | Beam 输出的最终候选数量 |
| 最终 Top-K 输出 | 3 | 返回给 UI 的推荐数量 |
| 取消检查间隔 | 32 节点 | 批次检查 CancellationToken |

搜索预算与全链路关系：搜索 15ms + 精排 ~3ms + 管理器 ~2ms = 全链路 < 20ms

新文件：

- `Scripts/AI/Generation/CandidateSequenceGenerator.cs`
- `Scripts/AI/Generation/TargetEquivalenceComparer.cs`

**步骤 2.4：非确定性截断机制**

截断触发条件：抽牌且洗牌、随机生成卡牌、未知候选集、评估器无法建模的手牌结构变化。

处理：终止该分支、给予保守启发式分、作为叶子序列参与排序。

**步骤 2.5：候选序列评估器（精排）**

近似前向评估器，仅追踪 ≤4 核心修正项。其余使用 PreviewValue + 不确定性降权。

新文件：`Scripts/AI/Evaluation/ActionSequenceEvaluator.cs`

**步骤 2.6：核心乘区有限复刻**

| 修正项 | 用途 |
|---|---|
| Strength | 修正攻击牌伤害估值 |
| Dexterity | 修正防��牌格挡估值 |
| Vulnerable | 修正目标受伤倍率 |
| Weak | 修正自身攻击倍率 |

扩展规则：新增需满足高频（>30%）、数学清晰、普适性强，并在 CHANGELOG 记录。

新文件：`Scripts/AI/Evaluation/EvaluationScratchpad.cs`

**步骤 2.7：热路径近零分配实现**

固定上限数组、预分配 scratch buffer、对象池 / `ArrayPool<T>`。粗排阶段零堆分配。

**步骤 2.8：推荐置信度定义**

基于 Top1/Top2 分差的 RelativeDelta，结合不确定性修正。输出：High / Medium / Low。

**步骤 2.9：推荐管理器**

| 参数 | 默认值 | 说明 |
|---|---|---|
| 防抖窗口 | 80ms | 状态变化后等待此时间再触发重算 |
| 最大重算频率 | 每秒 4 次 | 即 250ms 最小间隔 |
| 过期计算取消 | 启用 | 通过 CancellationToken 取消 |

dirty flags + snapshot hash 去重 + generation id + 结果粘性 + 自适应退化管理。

新文件：`Scripts/AI/Recommendation/RecommendationManager.cs`

### 7.5 性能预算

**目标值：**

| 指标 | 目标 |
|---|---|
| 单次推荐全链路 | < 20ms |
| 推荐重算频率 | ≤ 4 次/秒 |
| 热路径堆分配 | 近零 |
| UI 刷新 | 不阻塞战斗动画 |

**告警阈值：**

| 条件 | 行为 | 日志 |
|---|---|---|
| 单次 > 20ms | 记录但不干预 | Debug |
| 单次 > 50ms | 记录 + 触发退化评估 | Warn |
| 连续 5 次 > 20ms | 触发自适应退化 | Warn |
| GC Gen1+ 检测 | 记录 | Warn（仅 Debug 模式） |

**自适应退化策略：**

| 退化级别 | 措施 | 恢复条件 |
|---|---|---|
| Level 1 | 精排候选从 15 降至 8 | 连续 10 次正常后恢复 |
| Level 2 | Beam Width 从 5 降至 3 | 连续 10 次正常后恢复 |
| Level 3 | 防抖窗口从 80ms 升至 200ms | 连续 10 次正常后恢复 |
| Level 4 | 跳过精排，直接输出粗排 Top-K | 连续 10 次正常后恢复 |

### 7.6 验证标准

- 可输出合法推荐序列
- 能识别基础连招顺序收益
- 非确定性截断稳定
- 同一快照下结果稳定
- 热路径无明显 GC 尖刺
- 满足性能预算
- 等价目标剪枝正确
- 粗排/精排两阶段正常协作
- 自适应退化可正确触发与恢复

---

## 8. Phase 3：游戏内浮层 UI

### 8.1 目标

以稳定、低打扰、主线程安全的方式展示所有域的推荐结果。

### 8.2 UI 原则

- 默认简洁，可展开解释
- 不遮挡关键游戏信息
- 结果稳定，不频繁闪烁
- 不伪装成绝对正确答案
- 支持多层 Overlay 并存，通过栈式 UI 状态机管理层级

**栈式 UI 状态机模型：**

`OverlayManager` 采用**栈���状态机（Stack-based State Machine）**管理 Overlay 层级，而非互斥开关。这是因为游戏 UI 支持叠加操作（如战斗中打开药水面板、地图界面查看牌组）。

**状态栈行为：**

| 操作 | 栈行为 | 示例 |
|---|---|---|
| 进入战斗 | Push CombatOverlay | 栈：[Combat] |
| 战斗中打开药水面板 | Push PotionOverlay | 栈：[Combat, Potion] |
| 关闭药水面板 | Pop PotionOverlay | 栈：[Combat] |
| 战斗结束进入选卡 | Pop CombatOverlay, Push CardRewardOverlay | 栈：[CardReward] |
| 进入地图 | Push RouteOverlay | 栈：[Route] |

**层级管理：**

- 每个 Overlay 有一个 `BaseZIndex`，入栈时根据栈深度自动调整 Z-Index
- 栈顶 Overlay 获得完整交互权
- 非栈顶 Overlay 降低透明度但保持可见（半透明提示），或根据配置隐藏
- 栈为空时无 Overlay 显示
- 栈深度上限为 4，超限时清空并重建

**场景检测驱动：**

- OverlayManager 订阅游戏场景变化事件
- 主场景切换（战斗→地图→事件）时清空栈并 Push 对应主 Overlay
- 子面板打开/关闭时 Push/Pop 辅助 Overlay

### 8.3 工作项

**步骤 3.1：Overlay 架构**

- 创建 CanvasLayer
- Godot Anchor 系统自适应布局
- 栈式状态机实现
- 浮层默认位置：屏幕右侧，避开核心游戏 UI

新文件：`Scripts/UI/OverlayManager.cs`

**步骤 3.2：出牌推荐展示**

推荐顺序数字、目标标记、推荐面板、理由摘要、置信度等级。

新文件：`Scripts/UI/CombatOverlay.cs`

**步骤 3.3：主线程刷新实现**

- `_Process` 第一行捕获局部引用
- 仅 Version 变化时刷新
- 所有节点操作前 `IsInstanceValid()` 检查

**步骤 3.4：配置项**

**UserConfig（用户可见）：**

| 配置��� | 类型 | 默认值 |
|---|---|---|
| AI 辅助总开关 | bool | true |
| 是否显示出牌顺序 | bool | true |
| 是否显示详细理由 | bool | false |
| Top-N 显示数量 | int | 3 |
| 浮层位置 | enum | Right |
| 最低显示置信度 | enum | Low |
| 是否启用异步复盘 | bool | false |
| 是否启用路线推荐 | bool | true |
| 是否启用选卡推荐 | bool | true |
| 是否启用事件推荐 | bool | true |
| 是否启用药水推荐 | bool | true |
| 是否启用遗物评估 | bool | true |
| 非战斗域是否使用 LLM 辅助 | bool | false |

**DevConfig（调试模式可见）：**

| 配置项 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| Beam Width | int | 5 | 搜索宽度 |
| 最大搜索深度 | int | 10 | 搜索深度上限 |
| 最大候选节点数 | int | 500 | 搜索节点预算 |
| 搜索时间预算 | ms | 15 | 内部搜索时限 |
| 精排候选上限 | int | 15 | 精排阶段候选数 |
| 防抖窗口 | ms | 80 | 状态变化防抖 |
| 价值函数权重组 | float[] | 见 §7.4 | 各评分维度权重 |
| 日志级别 | enum | Info | 日志输出级别 |
| 是否导出 Snapshots | bool | false | 调试用快照导出 |
| 是否开启性能 Profiling | bool | false | 计时日志 |
| 结果粘性时长 | ms | 500 | 推荐结果最短显示时间 |
| LLM 超时时间 | ms | 5000 | 非战斗域 LLM 调用超时 |
| 路线前瞻深度 | int | 3 | 路线推荐前瞻层数 |
| Overlay 栈深度上限 | int | 4 | 超限时自动清空重建 |

新文件：`Scripts/Config/AITTKConfig.cs`

### 8.4 验证标准

- 各域 UI 可稳定显示推荐
- 无跨线程 UI 崩溃风险
- 关闭功能后无残留节点
- 推荐更新不明显抖动
- 不同分辨率下浮层位置正确
- 场景切换时正确显示对应域 Overlay
- 多层 Overlay 并存时 Z-Index 正确
- 栈异常时可自动恢复

---

## 9. Phase 4：异步战术复盘与 LLM 解释增强

### 9.1 目标

将本地 LLM 用于 **非实时** 的战术复盘与解释增强，并为非战斗域提供异步辅助分析。

### 9.2 定位

**LLM 负责：**

- 回合结束后的战术点评
- 战斗结束后的总结
- 对规则推荐进行自然语言解释
- 回答"为什么不推荐另一个动作"
- 路线/选卡/事件/遗物域的异步辅助分析
- 未收录事件的效果推断

**LLM 不负责：**

- 战斗内实时推荐排序
- 战斗内实时合法性判断
- 任何域的自动执行决策

**非战斗域 LLM 调用模式：**

```
用户进入决策界面
        ↓
规则引擎立即输出基线推荐（毫秒级）→ 立即显示
        ↓  （同时）
后台异步发送 LLM 请求
        ↓
LLM 返回后更新 UI（叠加 LLM 分析，标注"AI 分析"标签）
        ↓
LLM 超时或不可用 → 仅显示规则推荐，无影响
```

**隐私与数据安全：**

默认所有数据仅发送至本地 Ollama 服务（`localhost:11434`），不上传任何云端服务器。AITTK 不收集、不传输、不存储任何用户游戏数据至外部。

### 9.3 工作项

**步骤 4.1：LLM 复盘客户端**

- 调用本地 Ollama API，后台异步执行
- 新文件：`Scripts/AI/Review/LLMReviewClient.cs`

**步骤 4.2：LLM Prompt 工程**

模型选型：

| 模型 | 参数量 | 中文能力 | 推荐场景 |
|---|---|---|---|
| Qwen2.5-7B | 7B | ★★★★★ | 首选 |
| Llama3-8B | 8B | ★★★☆☆ | 备选 |
| Phi-3-mini | 3.8B | ★★☆☆☆ | 低配备选 |

Prompt 设计：精简结构化输入（< 800 Token），结构化输出（< 500 Token），角色为"杀戮尖塔战术分析师"。

新文件：`Scripts/AI/Review/LLMPromptBuilder.cs`

**���骤 4.3：非战斗域 LLM 辅助**

- 统一的域级 LLM 调用接口
- 为路线/选卡/事件/遗物提供异步补充分析
- 新文件：`Scripts/AI/Review/LLMDomainAdvisor.cs`

**步骤 4.4：复盘输入/输出**

输入：CombatSnapshot（精简版）、实际出牌序列、规则推荐结果、回合结果摘要。
输出：战术点评、替代思路、潜在失误分析、风险提示。

**步骤 4.5：复盘结果展示**

独立点评面板或气泡，不覆盖实时推荐 UI。

### 9.4 验证标准

- LLM 可异步返回复盘内容
- 失败或超时不影响实时推荐和规则推荐
- 非战斗域 LLM 辅助可正确叠加显示
- Prompt 输入 Token 数 < 800
- Ollama 不可用时所有域优雅降级

---

## 10. Phase 5：RunContext 与路线推荐

### 10.1 目标

建立贯穿整局的全局运行上下文（RunContext），并实现路线推荐。

### 10.2 RunContext 工作项

**步骤 5.1：RunContext 数据采集与 Hydration**

**增量更新（Event-driven）：**

- 订阅所有影响全局状态的游戏事件（战斗结束、获得卡牌/遗物/药水、楼层变化等）
- 维护 `RunContext` 对象，在每次状态变化时增量更新
- 更新后标记 `HydrationSource = "EventDriven"`

**全量重建（Hydration）：**

RunContextCollector 不能仅依赖增量事件更新。必须实现完整的状态重建能力，覆盖以下场景：

| 触发场景 | 行为 |
|---|---|
| Mod 首次加载（游戏已在进行中） | 从当前 Player 对象全量重建 RunContext |
| 玩家 Save & Quit 后重新加载 | 检测到 RunContext 为空/过期，触发全量重建 |
| 进入任何新场景 | 校验 RunContext 与实际状态一致性，不一致则重建 |
| RunContext 一致性校验失败 | 丢弃旧数据，全量重建 |

**全量重建流程：**

```
检测触发条件（首次加载 / 存档恢复 / 一致性校验失败）
   ↓
从 Player.Deck 重建 DeckComposition + FunctionalProfile
   ↓
从 Player.Relics 重建 RelicInventory
   ↓
从 Player.Potions 重建 PotionInventory
   ↓
从 Player（HP/Gold）重建 PlayerRunState
   ↓
从 IRunState（Act/Floor/Map）重建 RunProgress + MapState
   ↓
从 MapState 重建 ThreatWindow
   ↓
HistoricalPerformance 无法重建 → 填默认值 + 标记 IsHistoricalDataApproximate = true
   ↓
标记 IsHydrated = true, HydrationSource = "FullRebuild"
```

**一致性校验：**

每次进入新场景时，抽样校验 RunContext 与实际游戏状态的一致性：
- `RunContext.DeckComposition.TotalCardCount` == `Player.Deck.Count`
- `RunContext.RelicInventory.Length` == `Player.Relics.Count`
- `RunContext.PlayerRunState.CurrentHp` == `Player.CurrentHp`

任何一项不一致 → 触发全量重建并记录 Warn 日志。

新文件：

- `Scripts/Models/RunContext.cs`
- `Scripts/Models/DeckComposition.cs`
- `Scripts/Models/FunctionalProfile.cs`
- `Scripts/Models/SurvivalGapAnalysis.cs`
- `Scripts/Collectors/RunContextCollector.cs`

**步骤 5.2：牌组功能分析器 + 威胁窗口分析器**

**DeckFunctionalAnalyzer：**

不识别"流派"，而是分析牌组的**功能性指标**：

| 指标 | 计算方式 |
|---|---|
| 单体输出/回合 | 基于攻击牌的平均伤害 × 每回合可出牌数估算 |
| AOE 输出/回合 | 基于 AOE 牌的可用性和伤害 |
| 格挡/回合 | 基于防御牌的平均格挡值 |
| 成长潜力 | 力量/灵巧/毒等缩放牌的数量和质量 |
| 抽牌能力 | 抽牌牌数量和每回合额外抽牌估值 |

**ThreatWindowAnalyzer：**

基于 `MapState` 和 `RunProgress`，分析近期面临的威胁和对应的生存缺口：

- 根据前方节点类型（精英/Boss）和距离，判断威胁等级
- 将牌组功能指标与威胁需求对比，识别**具体的生存缺口**
- 例如："距离精英还有 2 层，单体输出估值仅 12/回合，缺口：需要高伤害攻击牌"

> **设计原则：永远不要因为"流派匹配"而推荐一张不解决当前问题的牌。**

新文件：

- `Scripts/AI/CardReward/DeckFunctionalAnalyzer.cs`
- `Scripts/AI/CardReward/ThreatWindowAnalyzer.cs`

### 10.3 路线推荐工作项

**步骤 5.3：路线数据采集**

- 订阅地图场景事件
- 采集 `ActMap`、当前坐标、可选子节点
- 新文件：`Scripts/Collectors/RouteCollector.cs`

**步骤 5.4：路线快照**

- 包含：当前位置、可选路径（2-3 层前瞻）、各节点类型
- 新文件：
  - `Scripts/Models/RouteSnapshot.cs`
  - `Scripts/Models/RouteAnalysis.cs`

**步骤 5.5：路线评估器**

对每个可选节点打分，评估维度：

| 维度 | 说明 |
|---|---|
| 战斗风险 | 精英/Boss 风险 vs 当前 HP 和牌组强度 |
| 资源收益 | 商店（有金币时价值高）、宝箱、休息点 |
| 成长价值 | 精英的遗物奖励对当前构筑的提升 |
| 路径规划 | 综合后续 2-3 层的节点类型 |
| HP 安全边际 | 当前 HP 是否允许冒险 |
| Boss 准备度 | 距离 Boss 还有几层，是否需要休息/强化 |

支持 2-3 层前瞻搜索。

新文件：

- `Scripts/AI/Route/IRouteRecommender.cs`
- `Scripts/AI/Route/RouteRecommender.cs`

**步骤 5.6：路线 UI**

- 地图界面上高亮推荐路径，显示各节点评分和理由
- 新文件：`Scripts/UI/RouteOverlay.cs`

### 10.4 验证标准

- RunContext 可在整局过程中稳定维护
- **Save & Quit 后重新加载，RunContext 可通过 Hydration 正确重建**
- **一致性校验可检测到不一致并触发自动重建**
- **Hydration 失败时非战斗域推荐被正确禁用**
- DeckFunctionalAnalyzer 可输出功能性指标
- ThreatWindowAnalyzer 可识别生存缺口
- 路线推荐可显示各节点评分
- 2-3 层前瞻正常工作

---

## 11. Phase 6：选卡 / 事件 / 药水 / 遗物推荐

### 11.1 卡牌奖励推荐（威胁驱动）

**触发时机：** 战斗胜利后出现卡牌选择界面

**评估维度：**

| 维度 | 说明 | 优先级 |
|---|---|---|
| 威胁解决价值 | 该卡牌是否直接解决 ThreatWindow 中识别的生存缺口 | 最高 |
| 功能补缺价值 | 该卡牌是否弥补 FunctionalProfile 中的弱项 | 高 |
| 地板价值 | 该卡牌在任何牌组中的基础强度（独立于构筑的通用价值） | 中 |
| 稀有度价值 | 稀有卡通常更值得拿 | 中 |
| 费用曲线 | 拿了之后平均费用是否合理 | 中 |
| 牌组膨胀惩罚 | 牌组太厚时"跳过"加分 | 全局 |
| 协同加分 | 与当前遗物/已有卡牌的组合潜力 | 低（权重上限不超过总分的 20%） |

**评估优先级（显式排序）：**

1. **生存底线**：如果 ThreatLevel = Critical 且该牌直接解决缺口 → 大幅加分
2. **威胁解决**：牌的功能与 SurvivalGaps 匹配 → 主要加分
3. **地板价值**：牌的通用强度 → 基础分
4. **膨胀��罚**：牌组过厚 → 全局扣分，"跳过"加分
5. **协同加分**：与已有牌/遗物的配合 → 次要加分

> **设计原则：永远不要因为"流派匹配"而推荐一张不解决当前问题的牌。协同价值是锦上添花，不是决策主因。**

新文件：

- `Scripts/Collectors/CardRewardCollector.cs`
- `Scripts/Models/CardRewardSnapshot.cs`
- `Scripts/AI/CardReward/CardRewardRecommender.cs`
- `Scripts/UI/CardRewardOverlay.cs`

### 11.2 事件推荐（事实数据 + 策略类 + LLM 兜底）

**触发时机：** 进入事件房间，事件选项可选时

**三层分离架构：**

| 层 | 职责 | 格式 |
|---|---|---|
| 事实层（JSON） | 描述事件选项的客观效果（如"失去 X 血量，获得 Y 遗物"） | `EventEffects.json` |
| 策略层（C#） | 根据 RunContext 动态计算选项价值 | `EventOptionEvaluator.cs` |
| 兜底层（LLM） | 处理未收录事件 | `LLMDomainAdvisor.cs` |

**EventEffects.json 仅存储事实，不存储评估逻辑：**

```json
{
  "BigFish": {
    "options": [
      {
        "id": "eat",
        "effects": [
          { "type": "Heal", "value": 5 }
        ]
      },
      {
        "id": "banana",
        "effects": [
          { "type": "MaxHpUp", "value": 5 }
        ]
      },
      {
        "id": "donut",
        "effects": [
          { "type": "GainRelic", "relicPool": "random" },
          { "type": "MaxHpDown", "value": 5 }
        ]
      }
    ]
  }
}
```

**EventOptionEvaluator.cs 负责动态评估：**

根据 RunContext 动态计算每个效果的价值：
- "Heal 5" 在满血时价值为 0，在血量低于 30% 时价值极高
- "MaxHpDown 5" 在 Boss 前价值为负，在 Act 1 早期惩罚较轻
- "GainRelic random" 的期望价值基于当前遗物数量和运行阶段

新文件：

- `Scripts/Collectors/EventCollector.cs`
- `Scripts/Models/EventSnapshot.cs`
- `Scripts/AI/Event/EventRecommender.cs`
- `Scripts/AI/Event/EventEffectRegistry.cs`
- `Scripts/AI/Event/EventOptionEvaluator.cs`
- `Scripts/Data/EventEffects.json`
- `Scripts/UI/EventOverlay.cs`

### 11.3 药水推荐

**触发时机：** 获得药水时 + 战斗前/中

**评估维度：**

| 维度 | 说明 |
|---|---|
| 当前战斗价值 | 该药水在本场战斗的即时收益 |
| 保留价值 | 留到精英/Boss 的预期收益（基于 ThreatWindow） |
| 槽位稀缺性 | 药水栏是否已满 |
| 替换价值 | 新药水 vs 现有药水 |

新文件：

- `Scripts/Collectors/PotionCollector.cs`
- `Scripts/Models/PotionSnapshot.cs`
- `Scripts/AI/Potion/PotionRecommender.cs`
- `Scripts/UI/PotionOverlay.cs`

### 11.4 遗物价值评估（标签 + 策略类）

**触发时机：** 获得遗物选择时

**标签 + 策略类模式：**

`RelicTags.json` 仅存储遗物的功能标签（不含评估逻辑）：

```json
{
  "Shuriken": {
    "tags": ["OnAttackPlayed", "GrantsStrength", "NeedsHighAttackCount"],
    "triggerCondition": "Play3Attacks"
  },
  "Vajra": {
    "tags": ["Passive", "GrantsStrength"],
    "triggerCondition": "Always"
  }
}
```

`RelicValueEvaluator.cs` 根据标签 + RunContext 动态计算价值：
- "NeedsHighAttackCount" 标签 → 检查 DeckComposition 中攻击牌占比
- "GrantsStrength" 标签 → 检查是否有多段攻击牌可以放大收益
- 基于 FunctionalProfile 和 ThreatWindow 综合评估

**评估维度：**

| 维度 | 说明 |
|---|---|
| 功能协同 | 遗物标签与当前牌组 FunctionalProfile 的匹配度 |
| 通用价值 | 遗物基础强度 |
| 阶段价值 | 在当前 Act/楼层的价值 |
| 金币机会成本 | 商店购买时金币是否有更好用途 |
| 威胁解决 | 遗物是否弥补 SurvivalGaps 中的缺口 |

新文件：

- `Scripts/Collectors/RelicCollector.cs`
- `Scripts/Models/RelicSnapshot.cs`
- `Scripts/AI/Relic/RelicRecommender.cs`
- `Scripts/AI/Relic/RelicTagRegistry.cs`
- `Scripts/AI/Relic/RelicValueEvaluator.cs`
- `Scripts/Data/RelicTags.json`
- `Scripts/UI/RelicOverlay.cs`

### 11.5 验证标准

- 选卡界面可显示各卡评分和"跳过"评分
- **威胁驱动优先级正确：高威胁场景下优先推荐解决缺口的牌而非流派牌**
- 事件知识库覆盖常见事件，动态评估根据 RunContext 变化
- LLM 兜底可处理未收录事件
- 药水建议可用
- 遗物评分可用，标签与策略类正确联动

---

## 12. 测试、评估与回归

### 12.1 目标

建立稳定的离线回放与回归测试体系，保障所有域的推荐质量与性能。

### 12.2 工作项

**步骤 12.1：Snapshot 回放**

所有域的 Snapshot 均可 JSON 导出/导入。回放前校验 SchemaVersion。

**步骤 12.2：Golden Cases**

**出牌域：**
- 必须优先防御、本回合可斩杀、AOE 最优、先上状态再攻击
- 0费链路优先、非确定性截断、高不确定降权
- 多等价目标、高能量多手牌

**路线域：**
- 低血量时应优先休息点
- 有金币时商店路线价值高
- Boss 前应避免连续精英

**选卡域：**
- 牌组过厚时应推荐跳过
- 缺 AOE 时 AOE 卡应加分
- **【流派陷阱测试】牌组有毒牌基础，但下一层是高伤精英，应推荐防御牌而非更多毒牌**
- **【跳过测试】牌组功能完整且偏厚，三张卡均为中等质量，应推荐跳过**
- **【生存缺口测试】血量低于 30%，应大幅提升治疗/防御相关卡牌的评分**

**事件域：**
- 常见事件选项正确评估
- 低血量时治疗选项评分上升

**RunContext 域：**
- **【Hydration 测试】模拟 Save & Quit 后重新加载，验证全量重建正确性**
- **【一致性校验测试】手动修改 RunContext 使其与实际状态不一致，验证自动重建触发**

**步骤 12.3：质量指标**

| 指标 | 说明 | 适用域 |
|---|---|---|
| 推荐合法率 | 推荐中无非法操作 | 出牌 |
| 必防场景正确率 | 致死威胁下优先防御 | 出牌 |
| Lethal 场景正确率 | 可斩杀时成功识别 | 出牌 |
| Combo 顺序识别率 | 先buff后攻击等 | 出牌 |
| Top-1 人工接受率 | 人工评审接受比例 | 所有域 |
| 平均耗时 | 单次推荐计算时间 | 出牌 |
| P99 耗时 | 第 99 百分位计算时间 | 出牌 |
| GC 次数 / 峰值 | 热路径 GC 情况 | 出牌 |
| 高置信推荐正确率 | High 置信度的实际正确比例 | 所有域 |
| 退化触发频率 | 自适应退化的触发次数 | 出牌 |
| 事件知识库覆盖率 | 已收录事件占总事件比例 | 事件 |
| 功能分析准确率 | FunctionalProfile 判断的正确比例 | 选卡/遗物 |
| 威胁驱动正确率 | 高威胁场景下推荐是否解决缺口 | 选卡 |
| Hydration 重建成功率 | 存档恢复后 RunContext 重建正确比例 | RunContext |

**步骤 12.4：回归测试**

- ��次改规则后运行 replay
- 对比 Golden Cases 结果并输出 diff 日志
- SchemaVersion 不兼容时尝试 migration

---

## 13. Phase 7：后续扩展

后续可扩展：

- 商店购买推荐
- 休息点选择推荐（休息 vs 升级 vs 其他选项）
- 多角色构筑模板库
- 社区贡献的事件/遗物知识库更新机制
- 跨局统计与玩家画像

原则：各模块独立，不与现有域强耦合。

---

## 14. 关键文件清单

| 文件 | 操作 | 阶段 |
|---|---|---|
| `Scripts/Collectors/CombatCollector.cs` | 修改/完善 | P1 |
| `Scripts/Models/CombatSnapshot.cs` | 新建 | P1 |
| `Scripts/Models/PlayerStateInfo.cs` | 新建 | P1 |
| `Scripts/Models/EnemyStateInfo.cs` | 新建 | P1 |
| `Scripts/Models/CardInstanceInfo.cs` | 新建 | P1 |
| `Scripts/Models/PowerInfo.cs` | 新建 | P1 |
| `Scripts/Models/CombatContextInfo.cs` | 新建 | P1 |
| `Scripts/Models/DerivedCombatAnalysis.cs` | 新建 | P1 |
| `Scripts/AI/CombatAnalyzer.cs` | 新建 | P1 |
| `Scripts/Debug/CombatSnapshotSerializer.cs` | 新建 | P1 |
| `Scripts/Debug/SnapshotMigrator.cs` | 新建 | P1 |
| `Scripts/Debug/AITTKLogger.cs` | 新建 | P1 |
| `Scripts/Compatibility/ApiCompatibilityChecker.cs` | 新建 | P1 |
| `Scripts/AI/IPlayRecommender.cs` | 新建 | P2 |
| `Scripts/AI/PlayRecommendation.cs` | 新建 | P2 |
| `Scripts/AI/RecommendationSnapshot.cs` | 新建 | P2 |
| `Scripts/AI/Generation/CandidateSequenceGenerator.cs` | 新建 | P2 |
| `Scripts/AI/Generation/TargetEquivalenceComparer.cs` | 新建 | P2 |
| `Scripts/AI/Evaluation/ActionSequenceEvaluator.cs` | 新建 | P2 |
| `Scripts/AI/Evaluation/EvaluationScratchpad.cs` | 新建 | P2 |
| `Scripts/AI/Evaluation/ValueFunction.cs` | 新建 | P2 |
| `Scripts/AI/Evaluation/FastHeuristic.cs` | 新建 | P2 |
| `Scripts/AI/Recommendation/RuleBasedRecommender.cs` | 新建 | P2 |
| `Scripts/AI/Recommendation/RecommendationManager.cs` | 新建 | P2 |
| `Scripts/UI/OverlayManager.cs` | 新建 | P3 |
| `Scripts/UI/CombatOverlay.cs` | 新建 | P3 |
| `Scripts/Config/AITTKConfig.cs` | 新建 | P3 |
| `Scripts/AI/Review/LLMReviewClient.cs` | 新建 | P4 |
| `Scripts/AI/Review/LLMPromptBuilder.cs` | 新建 | P4 |
| `Scripts/AI/Review/TacticalReviewService.cs` | 新建 | P4 |
| `Scripts/AI/Review/LLMDomainAdvisor.cs` | 新建 | P4 |
| `Scripts/Models/RunContext.cs` | 新建 | P5 |
| `Scripts/Models/DeckComposition.cs` | 新建 | P5 |
| `Scripts/Models/FunctionalProfile.cs` | 新建 | P5 |
| `Scripts/Models/SurvivalGapAnalysis.cs` | 新建 | P5 |
| `Scripts/Collectors/RunContextCollector.cs` | 新建 | P5 |
| `Scripts/AI/CardReward/DeckFunctionalAnalyzer.cs` | 新建 | P5 |
| `Scripts/AI/CardReward/ThreatWindowAnalyzer.cs` | 新建 | P5 |
| `Scripts/Collectors/RouteCollector.cs` | 新建 | P5 |
| `Scripts/Models/RouteSnapshot.cs` | 新建 | P5 |
| `Scripts/Models/RouteAnalysis.cs` | 新建 | P5 |
| `Scripts/AI/Route/IRouteRecommender.cs` | 新建 | P5 |
| `Scripts/AI/Route/RouteRecommender.cs` | 新建 | P5 |
| `Scripts/UI/RouteOverlay.cs` | 新建 | P5 |
| `Scripts/Collectors/CardRewardCollector.cs` | 新建 | P6 |
| `Scripts/Models/CardRewardSnapshot.cs` | 新建 | P6 |
| `Scripts/AI/CardReward/CardRewardRecommender.cs` | 新建 | P6 |
| `Scripts/UI/CardRewardOverlay.cs` | 新建 | P6 |
| `Scripts/Collectors/EventCollector.cs` | 新建 | P6 |
| `Scripts/Models/EventSnapshot.cs` | 新建 | P6 |
| `Scripts/AI/Event/EventRecommender.cs` | 新建 | P6 |
| `Scripts/AI/Event/EventEffectRegistry.cs` | 新建 | P6 |
| `Scripts/AI/Event/EventOptionEvaluator.cs` | 新建 | P6 |
| `Scripts/Data/EventEffects.json` | 新建 | P6 |
| `Scripts/UI/EventOverlay.cs` | 新建 | P6 |
| `Scripts/Collectors/PotionCollector.cs` | 新建 | P6 |
| `Scripts/Models/PotionSnapshot.cs` | 新建 | P6 |
| `Scripts/AI/Potion/PotionRecommender.cs` | 新建 | P6 |
| `Scripts/UI/PotionOverlay.cs` | 新建 | P6 |
| `Scripts/Collectors/RelicCollector.cs` | 新建 | P6 |
| `Scripts/Models/RelicSnapshot.cs` | 新建 | P6 |
| `Scripts/AI/Relic/RelicRecommender.cs` | 新建 | P6 |
| `Scripts/AI/Relic/RelicTagRegistry.cs` | 新建 | P6 |
| `Scripts/AI/Relic/RelicValueEvaluator.cs` | 新建 | P6 |
| `Scripts/Data/RelicTags.json` | 新建 | P6 |
| `Scripts/UI/RelicOverlay.cs` | 新建 | P6 |
| `Scripts/Entry.cs` | 修改 | P1-P6 |

---

## 15. 里程碑与验收标准

| 里程碑 | 验收标准 | 预估工期 |
|---|---|---|
| **M1：标准状态层** | CombatSnapshot 稳定；JSON 导出/回放；日志层；API 兼容性检查；SnapshotMigrator | ~1 周 |
| **M2：出牌推荐** | 合法推荐；combo 识别；置信度；性能预算；粗排/精排；等价剪枝 | ~2-3 周 |
| **M2.5：回归体系** | 离线 replay；Golden Cases；质量指标；自适应退化 | 与 M2 并行 |
| **M3：UI** | 栈式状态机；各域 Overlay 稳定；分辨率自适应；UserConfig/DevConfig | ~1 周 |
| **M4：LLM** | 异步复盘；LLMDomainAdvisor 框架；降级 | ~1 周 |
| **M5：RunContext + 路线** | RunContext 含 Hydration 与一致性校验；FunctionalProfile 与 ThreatWindow；路线推荐 2-3 层前瞻 | ~1-2 周 |
| **M6a：选卡推荐** | 威胁驱动评估正确；流派陷阱测试通过；"跳过"评分合理 | ~1 周 |
| **M6b：事件推荐** | 事实+策略类分离；知识库覆盖常见事件；LLM 兜底 | ~1-2 周 |
| **M6c：药水+遗物** | 药水建议可用；遗物标签+策略类正确联动 | ~1 周 |
| **M7：全域 LLM** | "规则先出+LLM 后补"双阶段正常工作 | ~1 周 |

> 工期为单人开发粗估。

---

## 16. 风险与应对

| # | 风险 | 应对策略 |
|---|---|---|
| 1 | 评估器膨胀成伪战斗引擎 | ≤4 核心乘区；准入标准；术语"近似前向评估器" |
| 2 | 抽牌/生成牌导致搜索失控 | 非确定性截断 + 保守启发式 |
| 3 | 热路径 GC 尖刺 | 固定数组 / scratch buffer / 对象池 |
| 4 | Godot 跨线程 UI 崩溃 | 主线程轮询 + 不可变对象 + 引用捕获 |
| 5 | 置信度误导玩家 | top1/top2 分差；高不确定性降级 |
| 6 | LLM 拖累主流程 | 战斗域不参与；非战斗域异步+超时 |
| 7 | 游戏版本 API 不兼容 | 启动反射探测；按域降级 |
| 8 | Golden Cases Schema 变更 | SchemaVersion + SnapshotMigrator |
| 9 | 多目标分支爆炸 | 等价目标剪枝 + 节点预算 500 + 粗排淘汰 |
| 10 | CancellationToken 轮询开销 | 批次检查（每 32 节点） |
| 11 | 性能超标 | 三层预算 + 自适应退化 + 自动恢复 |
| 12 | 事件效果数据维护成本 | JSON 仅存事实；评估逻辑在 C# 策略类；LLM 兜底未收录事件 |
| 13 | RunContext 读档后为空 | Hydration 全量重建；一致性校验；重建失败禁用非战斗域 |
| 14 | UI 叠加操作导致 Overlay 冲突 | 栈式状态机；Z-Index 管理；栈异常自动恢复 |
| 15 | 功能分析误判 | 保守阈值；威胁驱动优先；Golden Cases 覆盖流派陷阱场景 |
| 16 | 选卡陷入流派陷阱 | 协同加分权重上限 20%；威胁解决为首要评估维度；显式设计原则禁止流派主导 |
| 17 | 知识库数据腐化 | JSON 仅存标签/事实；动态逻辑在策略类；版本更新后标签可增量补充 |
| 18 | 多域 Overlay 遮挡画面 | 栈式管理；非栈顶半透明或隐藏；栈深度上限 |

---

## 17. 建议执行顺序

```
P1  ：Snapshot + Analysis + 日志层 + API 兼容性检查 + SnapshotMigrator
         ↓
P2  ：出牌推荐 baseline（ValueFunction + FastHeuristic + Beam Search + 等价剪枝）
         ↓
P2.5：非确定性截断、核心乘区修正、性能 profiling、自适应退化
         ↓  (并行)
P2.6：replay / Golden Cases / GC 指标
         ↓
P3  ：栈式 UI 状态机 + 出牌 Overlay + 场景检测框架
         ↓
P4  ：异步 LLM 复盘 + LLMDomainAdvisor 框架
         ↓
P5  ：RunContext (含 Hydration) + DeckFunctionalAnalyzer + ThreatWindowAnalyzer + 路线推荐
         ↓
P6  ：选卡（威胁驱动）/ 事件（事实+策略类）/ 药水 / 遗物（标签+策略类）
         ↓
P7  ：全域 LLM 异步辅助联调
```

---

## 18. 最终定义

> 一个面向《杀戮尖塔2》的全域 AI 辅助决策插件，覆盖出牌、路线、选卡、事件、药水、遗物六大决策域；
> 以标准化快照和 RunContext（含 Hydration 状态重建）为数据基础，各域遵循统一的 Collector → Snapshot → Recommender → Overlay 四层架构，域间通过 RunContext 间接通信；
> 战斗内出牌推荐采用规则引擎��时计算（≤4 核心修正项、两阶段 Beam Search、等价目标剪枝、近零分配热路径）；
> 选卡/遗物评估采用威胁驱动（Threat-Resolution）体系，以生存缺口而非流派匹配为首要考量；
> 事件/遗物知识库采用事实数据与策略类分离架构，JSON 仅存标签和客观效果，评估逻辑在 C# 中动态计算；
> 非战斗域采用规则引擎即时基线 + LLM 异步辅���的双阶段模式；
> UI 采用栈式状态机管理多层 Overlay 并存；
> 具备按域降级的 API 兼容性检查、Schema 版本化与 Migration、失败降级矩阵、自适应性能退化和离线回归测试体系；
> 配置项分为用户层与开发层，所有数据仅在本地处理，不上传云端。

---

## 附录 A：版本变更历史

### v4 → v5 变更摘要

| # | 变更 |
|---|---|
| 1 | 游戏版本兼容性策略 + API 探测 |
| 2 | Snapshot SchemaVersion + 迁移策略 |
| 3 | 统一日志层 AITTKLogger |
| 4 | Beam Search + 参数初始值 |
| 5 | 权重表 + 调参机制 + 战斗阶段差异 |
| 6 | "近似前向评估器" + 修正项准入标准 |
| 7 | 节流参数 + CancellationToken |
| 8 | 分辨率自适应 + Anchor 布局 |
| 9 | PromptBuilder + 模型选型 + Token 预算 |
| 10 | 里程碑时间估算 |
| 11 | 风险 #7 (API 不兼容), #8 (Schema 变更) |
| 12 | 文件结构子目录组织 |
| 13 | 设计原则"优雅降级" |
| 14 | 配置项扩展 |

### v5 → v6 变更摘要

| # | 变更 |
|---|---|
| 1 | 失败与降级策略矩阵 |
| 2 | UserConfig / DevConfig 两层 |
| 3 | 粗排/精排两阶段 |
| 4 | Migration 机制 + SnapshotMigrator |
| 5 | 性能三层（目标/告警/退化）+ 自适应退化 |
| 6 | 组件职责边界声明 |
| 7 | "轻量级前向模拟器"→"近似前向评估器" |
| 8 | 搜索预算 15ms vs 全链路 20ms 说明 |
| 9 | LLM 隐私声明 |
| 10 | 等价目标剪枝 + 节点预算 500 |
| 11 | CancellationToken 批次检查（每 32 节点） |
| 12 | UI 引用捕获规则 + ImmutableArray |
| 13 | 风险 #9-#11 |

### v6 → v7 变更摘要

| # | 变更 |
|---|---|
| 1 | 范围扩展为六大决策域 |
| 2 | 新增 RunContext 全局运行上下文 |
| 3 | 多域统一架构原则 |
| 4 | 非战斗域推荐数据流 |
| 5 | 路线推荐（Phase 5） |
| 6 | 选卡/事件/药水/遗物推荐（Phase 6） |
| 7 | LLM 升级为非战斗域异步辅助 |
| 8 | API 兼容性检查按域降级 |
| 9 | UserConfig 新增各域开关 |
| 10 | 里程碑 M5-M7 |
| 11 | 风险 #12-#15 |
| 12 | 新增数据文件与 ~30 个新文件 |

### v7 → v7.1 变更摘要

| # | 变更 | 位置 |
|---|---|---|
| 1 | 选卡评估从"流派匹配"重构为"威胁驱动（Threat-Resolution）" | §1.4, §5.4, §10.2, §11.1 |
| 2 | ArchetypeSignals 替换为 FunctionalProfile + SurvivalGapAnalysis | §5.4 |
| 3 | DeckArchetypeAnalyzer 替换为 DeckFunctionalAnalyzer + ThreatWindowAnalyzer | §10.2 |
| 4 | RunContext 新增 ThreatWindow（近期威胁窗口） | §5.4 |
| 5 | RunContext 新增 Hydration（状态重建）机制 | §10.2 步骤5.1 |
| 6 | RunContext 新增一致性校验 + 元数据字段 | §10.2 步骤5.1 |
| 7 | 事件知识库从单层 JSON 重构为三层：事实(JSON) + 策略(C#) + 兜底(LLM) | §11.2 |
| 8 | EventKnowledgeBase 替换为 EventEffectRegistry + EventOptionEvaluator | §11.2 |
| 9 | 遗物知识库从 RelicSynergies.json 重构为 RelicTags.json + RelicValueEvaluator | §11.4 |
| 10 | UI 从互斥开关改为栈式状态机（Stack-based State Machine） | §8.2, §8.3 |
| 11 | 失败矩阵新增 Hydration 失败、栈异常、知识库未收录等场景 | §4.5 |
| 12 | Golden Cases 新增流派陷阱测试、Hydration 测试 | §12.2 |
| 13 | 质量指标新增威胁驱动正确��、Hydration 成功率 | §12.3 |
| 14 | 新增设计原则"威胁驱动"和"数据与逻辑分离" | §1.4 |
| 15 | 风险新增 #16 (流派陷阱)、#17 (知识库腐化)、#18 (Overlay 冲突) | §16 |
| 16 | 选卡协同加分权重上限硬性约束为 20% | §11.1 |