# Plan v6: AITTK 战斗 AI 辅助决策系统开发计划

## TL;DR

为《杀戮尖塔2》开发战斗内 AI 辅助决策插件，专注 **出牌推荐** 单一决策域。
出牌推荐采用 **规则引擎实时计算**；战术复盘采用 **本地 LLM 异步辅助**。
遵循 **Collector → Snapshot → Recommender → Overlay** 四层架构。

候选搜索采用两阶段评分：Beam 展开阶段使用低成本 **FastHeuristic 粗排**；最终候选集合使用 **ActionSequenceEvaluator 精排**。
配置项分为 **UserConfig（用户可见）** 与 **DevConfig（调试模式可见）** 两层。
具备 **失败降级矩阵、Snapshot Schema Migration、性能三层预算（目标/告警/自适应退化）**。

---

## 1. 产品目标与范围

### 1.1 产品目标

在战斗回合内，基于当前状态输出：

- 推荐的出牌序列
- 推荐目标
- 推荐理由
- 推荐强度 / 置信度

结果以游戏内浮层 UI 呈现，不自动执行任何游戏操作。

### 1.2 首期范围

本期覆盖：

- 战斗状态采集与单回合出牌推荐
- 游戏内浮层展示
- 规则引擎推荐（出牌域）
- 异步战术复盘（LLM）

### 1.3 非目标

当前版本明确不做：

- 自动出牌 / 自动打牌
- 自动选择任何选项（纯建议）
- 多回合全局搜索
- 完整复刻战斗结算引擎
- 路线、选卡、事件、药水、遗物等非战斗域推荐
- 将 LLM 接入实时战斗推荐主链路
- 修改游戏逻辑或干预战斗执行

### 1.4 设计原则

- **实时性优先**：战斗内推荐必须立即可用
- **稳定性优先**：推荐结果不能高频抖动
- **低打扰优先**：UI 不遮挡关键游戏信息
- **解释性优先**：推荐必须能给出可读理由
- **规则优先**：规则引擎是推荐的即时基线
- **有限复刻**：只复刻少量核心数学修正项，不重建完整引擎
- **热路径低分配**：搜索与评估路径尽量近零分配，避免 GC 尖刺
- **优雅降级**：任何子系统失败不应导致游戏崩溃或 Mod 不可用

---

## 2. 当前基础与已确认信息

### 2.1 用户决策记录

- AI 策略：规则引擎为主，本地 LLM 为辅
- UI 方式：游戏内浮层 UI
- 首选功能：出牌推荐
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

**CombatManager / CombatState**

- `CombatManager.Instance`
  - Events: `CombatSetUp`, `TurnStarted`, `TurnEnded`, `CombatEnded`, `CombatWon`, `CreaturesChanged`
- `CombatState`: `CurrentSide`, `RoundNumber`, `Players`, `Enemies`, `Allies`, `Creatures`, `HittableEnemies`

**Player / PlayerCombatState**

- `Player`: `Relics`, `Potions`, `Gold`, `MaxEnergy`, `Deck`, `Character`
- `PlayerCombatState`: `Hand`, `DrawPile`, `DiscardPile`, `ExhaustPile`, `PlayPile`, `Energy`, `MaxEnergy`
  - `HasEnoughResourcesFor(card, out reason)`

**CardModel**

- `Title`, `EnergyCost`, `Type`, `TargetType`, `Rarity`, `Keywords`, `Tags`
- `IsUpgraded`, `CurrentUpgradeLevel`, `GainsBlock`, `IsBasicStrikeOrDefend`
- `DynamicVars`: `Damage`, `Block`, `Heal`, `Energy`, `PoisonPower`, `StrengthPower`, `DexterityPower`, `WeakPower`, `VulnerablePower`

**Creature / Powers**

- `Creature`: `CurrentHp`, `MaxHp`, `Block`, `Name`, `Monster`, `Powers`, `IsStunned`, `IsHittable`
- `PowerModel`: `Type`, `StackType`, `Amount`, `DisplayAmount`, `Title`, `Owner`, `DynamicVars`

**敌人意图**

- `MonsterModel.NextMove` → `MoveState.Intents` → `IReadOnlyList<AbstractIntent>`
- `AbstractIntent`: `IntentType`, `IntentTitle`
- `AttackIntent`: `GetSingleDamage(...)`, `GetTotalDamage(...)`, `Repeats`

**Relic / Potion**

- `RelicModel`: `Title`, `Description`, `Status`, `DynamicVars`
- `PotionModel`: `Title`, `Rarity`, `Usage`, `TargetType`, `DynamicVars`

### 2.5 游戏版本兼容性策略

《杀戮尖塔2》处于 Early Access 阶段，API 随版本更新可能发生不兼容变化。

**启动时兼容性检查：**

在 `Entry.cs` 初始化阶段，通过反射探测关键 API 的可用性：
- 检查通过 → 正常加载推荐模块
- 检查失败 → 禁用推荐功能，在日志和游戏内显示友好提示，Mod 本身不崩溃

**兼容性检查清单：**

| 检查项 | 检查方式 |
|---|---|
| `CombatManager.Instance` 可访问 | 反射检查属性存在性 |
| `CombatState` 核心属性完整 | 反射检查 `Players`, `Enemies`, `RoundNumber` |
| `PlayerCombatState.Hand` 可访问 | 反射检查属性 |
| `MonsterModel.NextMove` 可访问 | 反射检查属性 |
| `CardModel.DynamicVars` 可访问 | 反射检查属性 |

新文件：`Scripts/Compatibility/ApiCompatibilityChecker.cs`

---

## 3. 总体架构

### 3.1 分层架构

```
┌─────────────────────────────────────────┐
│                  UI 层                   │
│         CombatOverlay / ReviewPanel      │
├─────────────────────────────────────────┤
│                Manager 层                │
│           RecommendationManager          │
├─────────────────────────────────────────┤
│              Recommendation 层           │
│   CandidateGenerator → Evaluator → Ranker│
├─────────────────────────────────────────┤
│               Analysis 层                │
│              CombatAnalyzer              │
├─────────────────────────────────────────┤
│               Snapshot 层                │
│             CombatSnapshot               │
├─────────────────────────────────────────┤
│               Collector 层               │
│             CombatCollector              │
├─────────────────────────────────────────┤
│             Compatibility 层             │
│           ApiCompatibilityChecker        │
├─────────────────────────────────────────┤
│               Logging 层                 │
│               AITTKLogger                │
└─────────────────────────────────────────┘

    [异步旁路] Review 层：LLMReviewClient / TacticalReviewService
```

### 3.2 实时推荐数据流

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

### 3.3 异步复盘数据流

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

## 4. 线程模型与并发约束

### 4.1 总原则

- 游戏 runtime object 只允许主线程访问
- Godot SceneTree / Node 只允许主线程访问
- 后台线程只处理纯 C# 数据对象
- 后台线程禁止直接修改 UI 节点

### 4.2 推荐计算线程边界

**主线程负责：**

- 监听游戏事件
- 采样并构建 CombatSnapshot
- 管理 UI 生命周期

**后台线程可负责：**

- 基于 CombatSnapshot 的分析与出牌推荐
- LLM 异步调用

### 4.3 结果发布模型

采用 **不可变结果对象 + 原子交换** 模式：

- 推荐结果封装为不可变对象
- 后台线程完成计算后，通过 `Interlocked.Exchange` 发布最新结果
- UI 主线程在 `_Process` 中读取最新引用并比对版本号
- 若版本变化，则刷新 UI

**UI 引用捕获规则：**

UI 线程在 `_Process` 方法的**第一行**必须将全局引用捕获为局部变量，后续所有读取操作均基于此局部引用，**禁止在同一帧内多次读取全局字段**。

`RecommendationSnapshot` 内部集合使用 `ImmutableArray<T>`，确保真正不可变。

### 4.4 并发实现约束

- 不使用高频 `lock(obj)` 保护 UI 读取路径
- 推荐结果对象必须不可变
- 结果对象需携带：`Version`、`GenerationId`、`Timestamp`
- 若存在多次并发计算，仅接受最新 `GenerationId` 的结果
- 后台推荐计算通过 `CancellationToken` 支持取消

### 4.5 失败与降级策略矩阵

| 失败场景 | 处理方式 | 用户可见行为 | 日志级别 |
|---|---|---|---|
| API 兼容性检查失败 | 禁用全部推荐模块，不注册事件监听 | 提示"AITTK: 当前游戏版本不兼容，推荐功能已禁用" | Error |
| Snapshot 构建异常 | 捕获异常，跳过本次推荐 | UI 保持上次结果；连续 3 次失败则清空并提示"数据采集异常" | Warn（单次）/ Error（连续） |
| 后台推荐计算超时 | 返回当前搜索到的最优候选 | 轻微延迟，推荐正常显示（质量可能略低） | Warn |
| 后台推荐线程未处理异常 | 捕获异常，丢弃本次结果，不传播 | UI 不更新，保持上次结果 | Error |
| LLM（Ollama）不可用 | 禁用复盘模块 | 复盘面板显示"复盘服务不可用" | Warn（首次）/ 后续静默 |
| LLM 单次超时 | 丢弃本次 LLM 结果 | 实时推荐不受影响 | Debug |
| Snapshot JSON 反序列化失败 | 跳过该 case | 回归测试输出跳过原因 | Warn |
| Snapshot SchemaVersion 不兼容 | 尝试 migration；不可迁移则拒绝加载 | 回归测试报警并列出不兼容版本 | Error |
| UI 节点不存在或已被释放 | `IsInstanceValid()` 检查后安全退出 | 无崩溃，无视觉异常 | Warn |
| 推荐计算连续 5 次超过性能目标 | 触发自适应退化（见 §7.5） | 推荐可能略简化，用户无感 | Warn |

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

`CombatSnapshot` 携带 `SchemaVersion` 字段（整数递增）。

- 序列化时写入当前 `SchemaVersion`
- 反序列化时检查版本兼容性
- 每次修改 Snapshot 字段结构时，必须递增 `SchemaVersion`

**轻量 Migration 机制：**

`SnapshotMigrator` 负责逐版本递增迁移。对于可自动升级的历史快照，提供轻量 migration hook；仅在结构性破坏变更时拒绝加载。

```csharp
// Scripts/Debug/SnapshotMigrator.cs
public static class SnapshotMigrator
{
    public static CombatSnapshot? TryMigrate(JsonObject raw, int sourceVersion)
    {
        if (sourceVersion == 1) raw = MigrateV1ToV2(raw);
        if (sourceVersion <= 2) raw = MigrateV2ToV3(raw);
        // ...
        return Deserialize(raw);
    }

    private static JsonObject MigrateV1ToV2(JsonObject raw)
    {
        // 新增字段 → 填默认值
        raw.TryAdd("NewField", JsonValue.Create(0));
        // 新增枚举值 → 历史数据回退 Unknown
        // 缺失字段 → 警告但不报错
        return raw;
    }
}
```

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

### 5.4 字段设计原则

**原始字段**（卡牌示例）：

- 标题、费用、类型、目标类型、keywords、DynamicVars、PreviewValue
- 当前所在区域（手牌/弃牌/抽牌堆等）

**推导字段**（卡牌示例）：

- EstimatedBaseDamage / EstimatedBaseBlock
- IsPlayable / UnplayableReason
- RequiresTarget / IsZeroCost
- ProvidesDefense / ProvidesDraw
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
- 将 runtime object 转换为纯数据对象
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
- 不兼容版本尝试 migration（通过 `SnapshotMigrator`）
- 新文件：
  - `Scripts/Debug/CombatSnapshotSerializer.cs`
  - `Scripts/Debug/SnapshotMigrator.cs`

**步骤 1.6：建立统一日志层**

- 统一管理日志级别（Trace / Debug / Info / Warn / Error）
- 同类型失败日志节流（60 秒内同类型最多 3 条）
- 新文件：`Scripts/Debug/AITTKLogger.cs`

**步骤 1.7：启动时 API 兼容性检查**

- 执行 §2.5 兼容性检查
- 检查失败则禁用推荐模块，Mod 本身不崩溃
- 新文件：`Scripts/Compatibility/ApiCompatibilityChecker.cs`

### 6.3 验证标准

- 相同状态可生成稳定结构快照
- 快照可序列化 / 反序列化，含 SchemaVersion 校验
- 历史版本快照可通过 migration 自动升级
- 推荐层不依赖 runtime object
- 日志层可正常输出各级别日志
- API 兼容性检查可正常通过/拒绝

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
| `RuleBasedRecommender` | 编排 Generator + Evaluator，产出推荐列表 | 不做 UI、不做线程控制、不做生命周期管理 |
| `RecommendationManager` | 生命周期、缓存、取消、发布、去抖 | 不做具体评分、不做 UI 操作 |
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

权重在 DevConfig 可配置。

新文件：`Scripts/AI/Evaluation/ValueFunction.cs`

**步骤 2.3：候选序列生成器（两阶段搜索）**

候选搜索采用两阶段评分：Beam 展开阶段使用低成本启发式进行粗排；最终候选集合使用 `ActionSequenceEvaluator` 做精排与置信度计算。

**阶段一：粗排（Fast Heuristic）**

在 Beam Search 展开每个节点时，使用极简特征快速筛掉明显差的分支：

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

仅对 Beam Search 输出的最终候选集合（10~15 个）执行完整评估：

- 核心修正项前向追踪（Strength / Dexterity / Vulnerable / Weak）
- 出牌顺序收益评估
- 不确定性降权
- 置信度前置特征计算

精排使用 `ActionSequenceEvaluator` + `EvaluationScratchpad` + `ValueFunction` 完整链路。

**等价目标剪枝（Symmetry Breaking）：**

当多个目标在以下维度完全等价时，视为等价目标并合并：当前 HP、当前 Block、本回合意图类型、意图伤害值、关键 Debuff 状态（Vulnerable / Weak / Poison 的 Amount）。

**CancellationToken 批次检查：** 每 32 个节点检查一次，控制轮询开销。

**首期参数默认值（后续可根据 profiling 与 Golden Cases 调整）：**

| 参数 | 默认值 | 说明 |
|---|---|---|
| Beam Width | 5 | 每层保留的 Top-K 候选（粗排决定） |
| 最大搜索深度 | 10 | 通常等于手牌上限 |
| 最大候选节点数 | 500 | 含等价剪枝后的安全冗余 |
| 搜索内部时间预算 | 15ms | 超时强制返回当前最优 |
| 精排候选上限 | 15 | Beam 输出的最终候选数量 |
| 最终 Top-K 输出 | 3 | 返回给 UI 的推荐数量 |
| 取消检查间隔 | 32 节点 | 批次检查 CancellationToken |

**搜索预算与全链路关系：**
- 搜索内部预算 15ms = Beam Search + 粗排
- 全链路目标 < 20ms = 搜索 15ms + 精排 ~3ms + 管理器开销 ~2ms

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
| Dexterity | 修正防御牌格挡估值 |
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
| 单次推荐全链路 | < 20ms（搜索 15ms + 精排 3ms + 管理器 2ms） |
| 推荐重算频率 | ≤ 4 次/秒 |
| 热路径堆分配 | 近零（粗排阶段零分配） |
| UI 刷新 | 不阻塞战斗动画 |

**告警阈值：**

| 条件 | 行为 | 日志 |
|---|---|---|
| 单次 > 20ms | 记录但不干预 | Debug |
| 单次 > 50ms | 记录 + 触发退化评估 | Warn |
| 连续 5 次 > 20ms | 触发自适应退化 | Warn |
| 单帧内检测到 GC Gen1+ | 记录 | Warn（仅 Debug 模式） |

**自适应退化策略：**

当连续超过性能目标时，`RecommendationManager` 自动启动退化模式，按以下优先级逐级降级：

| 退化级别 | 措施 | 恢复条件 |
|---|---|---|
| Level 1 | 精排候选上限从 15 降至 8 | 连续 10 次正常后恢复 |
| Level 2 | Beam Width 从 5 降至 3 | 连续 10 次正常后恢复 |
| Level 3 | 防抖窗口从 80ms 升至 200ms | 连续 10 次正常后恢复 |
| Level 4 | 跳过精排，直接输出粗排 Top-K | 连续 10 次正常后恢复 |

退化期间 UI 正常显示，用户无感知（推荐质量可能略降）。

### 7.6 验证标准

- 可输出合法推荐序列
- 能识别基础连招顺序收益
- 非确定性截断稳定
- 同一快照下结果稳定
- 热路径无明显 GC 尖刺
- 满足性能预算（目标值 + 告警阈值）
- 等价目标剪枝正确
- 粗排/精排两阶段正常协作
- 自适应退化可正确触发与恢复

---

## 8. Phase 3：游戏内浮层 UI

### 8.1 目标

以稳定、低打扰、主线程安全的方式展示出牌推荐结果。

### 8.2 UI 原则

- 默认简洁，可展开解释
- 不遮挡关键游戏信息
- 结果稳定，不频繁闪烁
- 不伪装成绝对正确答案

### 8.3 工作项

**步骤 3.1：Overlay 架构**

- 创建 CanvasLayer
- Godot Anchor 系统自适应布局
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

配置项分为 UserConfig 与 DevConfig 两层。默认 UI 仅暴露 UserConfig；DevConfig 仅在调试模式下可见。

**UserConfig（用户可见）：**

| 配置项 | 类型 | 默认值 |
|---|---|---|
| AI 辅助总开关 | bool | true |
| 是否显示出牌顺序 | bool | true |
| 是否显示详细理由 | bool | false |
| Top-N 显示数量 | int | 3 |
| 浮层位置 | enum | Right |
| 最低显示置信度 | enum | Low |
| 是否启用异步复盘 | bool | false |

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

新文件：`Scripts/Config/AITTKConfig.cs`

### 8.4 验证标准

- UI 可稳定显示推荐
- 无跨线程 UI 崩溃风险
- 关闭功能后无残留节点
- 推荐更新不明显抖动
- 不同分辨率下浮层位置正确

---

## 9. Phase 4：异步战术复盘与 LLM 解释增强

### 9.1 目标

将本地 LLM 用于 **非实时** 的战术复盘与解释增强。

### 9.2 定位

**LLM 负责：**

- 回合结束后的战术点评
- 战斗结束后的总结
- 对规则推荐进行自然语言解释
- 回答"为什么不推荐另一个动作"

**LLM 不负责：**

- 战斗内实时推荐排序
- 战斗内实时合法性判断
- 任何自动执行决策

**隐私与数据安全：**

默认所有复盘数据仅发送至本地 Ollama 服务（`localhost:11434`），不上传任何云端服务器。AITTK 不收集、不传输、不存储任何用户游戏数据至外部。

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

**步骤 4.3：复盘输入/输出**

输入：CombatSnapshot（精简版）、实际出牌序列、规则推荐结果、回合结果摘要。
输出：战术点评、替代思路、潜在失误分析、风险提示。

**步骤 4.4：复盘结果展示**

独立点评面板或气泡，不覆盖实时推荐 UI。

新文件：`Scripts/AI/Review/TacticalReviewService.cs`

### 9.4 验证标准

- LLM 可异步返回复盘内容
- 失败或超时不影响实时推荐
- Prompt 输入 Token 数 < 800
- Ollama 不可用时优雅降级

---

## 10. 测试、评估与回归

### 10.1 目标

建立稳定的离线回放与回归测试体系，保障推荐质量与性能。

### 10.2 工作项

**步骤 10.1：Snapshot 回放**

CombatSnapshot 可 JSON 导出/导入。回放前校验 SchemaVersion，不兼容时尝试 migration。

**步骤 10.2：Golden Cases**

- 必须优先防御（致死威胁）
- 本回合可斩杀时优先输出
- AOE vs 单体选择
- 先上状态再攻击
- 0费链路优先
- 非确定性截断场景
- 高不确定降权场景
- 多等价目标
- 高能量多手牌

**步骤 10.3：质量指标**

| 指标 | 说明 |
|---|---|
| 推荐合法率 | 推荐中无非法操作 |
| 必防场景正确率 | 致死威胁下优先防御 |
| Lethal 场景正确率 | 可斩杀时成功识别 |
| Combo 顺序识别率 | 先 buff 后攻击等 |
| Top-1 人工接受率 | 人工评审接受比例 |
| 平均耗时 | 单次推荐计算时间 |
| P99 耗时 | 第 99 百分位计算时间 |
| GC 次数 / 峰值 | 热路径 GC 情况 |
| 高置信推荐正确率 | High 置信度的实际正确比例 |
| 退化触发频率 | 自适应退化的触发次数 |

**步骤 10.4：回归测试**

- 每次改规则后运行 replay
- 对比 Golden Cases 结果并输出 diff 日志
- SchemaVersion 不兼容时尝试 migration

---

## 11. 关键文件清单

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
| `Scripts/Entry.cs` | 修改 | P1-P4 |

---

## 12. 里程碑与验收标准

| 里程碑 | 验收标准 | 预估工期 |
|---|---|---|
| **M1：标准状态层** | CombatSnapshot 稳定；JSON 导出/回放；日志层；API 兼容性检查；SnapshotMigrator | ~1 周 |
| **M2：出牌推荐** | 合法推荐；combo 识别；置信度；性能预算；粗排/精排；等价剪枝 | ~2-3 周 |
| **M2.5：回归体系** | 离线 replay；Golden Cases；质量指标；自适应退化 | 与 M2 并行 |
| **M3：UI** | CombatOverlay 稳定；分辨率自适应；UserConfig/DevConfig | ~1 周 |
| **M4：LLM** | 异步复盘；降级 | ~1 周 |

> 工期为单人开发粗估。

---

## 13. 风险与应对

| # | 风险 | 应对策略 |
|---|---|---|
| 1 | 评估器膨胀成伪战斗引擎 | ≤4 核心乘区；准入标准；术语"近似前向评估器" |
| 2 | 抽牌/生成牌导致搜索失控 | 非确定性截断 + 保守启发式 |
| 3 | 热路径 GC 尖刺 | 固定数组 / scratch buffer / 对象池 |
| 4 | Godot 跨线程 UI 崩溃 | 主线程轮询 + 不可变对象 + 引用捕获 |
| 5 | 置信度误导玩家 | top1/top2 分差；高不确定性降级 |
| 6 | LLM 拖累主流程 | 战斗域不参与；异步 + 超时 |
| 7 | 游戏版本 API 不兼容 | 启动反射探测；兼容性检查失败禁用推荐 |
| 8 | Golden Cases Schema 变更 | SchemaVersion + SnapshotMigrator |
| 9 | 多目标分支爆炸 | 等价目标剪枝 + 节点预算 500 + 粗排淘汰 |
| 10 | CancellationToken 轮询开销 | 批次检查（每 32 节点） |
| 11 | 性能超标 | 三层预算 + 自适应退化 + 自动恢复 |

---

## 14. 建议执行顺序

```
P1  ：Snapshot + Analysis + 日志层 + API 兼容性检查 + SnapshotMigrator
         ↓
P2  ：出牌推荐 baseline（ValueFunction + FastHeuristic + Beam Search + 等价剪枝）
         ↓
P2.5：非确定性截断、核心乘区修正、性能 profiling、自适应退化
         ↓  (并行)
P2.6：replay / Golden Cases / GC 指标
         ↓
P3  ：UI + 出牌 Overlay
         ↓
P4  ：异步 LLM 复盘
```

---

## 15. 最终定义

> 一个面向《杀戮尖塔2》的战斗 AI 辅助决策插件；
> 以标准化快照（CombatSnapshot）为数据基础，遵循 Collector → Snapshot → Recommender → Overlay 四层架构；
> 出牌推荐采用规则引擎实时计算（≤4 核心修正项、两阶段 Beam Search、等价目标剪枝、近零分配热路径）；
> 具备 API 兼容性检查、Snapshot Schema 版本化与 Migration、失败降级矩阵、自适应性能退化和离线回归测试体系；
> 配置项分为用户层（UserConfig）与开发层（DevConfig），所有数据仅在本地处理，不上传云端。

---

## 附录 A：版本变更历史

### v4 → v5 变更摘要

| # | 变更 |
|---|---|
| 1 | 游戏版本兼容性策略 + API 探测 |
| 2 | Snapshot SchemaVersion + 迁移策略 |
| 3 | 统一日志层 AITTKLogger |
| 4 | Beam Search + 参数初始值 |
| 5 | 权重表 + 调参机制 |
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

| # | 变更 | 位置 |
|---|---|---|
| 1 | 新增失败与降级策略矩阵 | §4.5 |
| 2 | 配置项分为 UserConfig / DevConfig 两层 | §8.3 步骤 3.4 |
| 3 | 候选搜索改为粗排/精排两阶段 + FastHeuristic | §7.4 步骤 2.3 |
| 4 | Snapshot 版本化补充 Migration 机制 + SnapshotMigrator | §5.2 |
| 5 | 性能预算改为三层（目标/告警/退化）+ 自适应退化策略 | §7.5 |
| 6 | 组件职责边界统一声明 | §7.4 步骤 2.2 |
| 7 | "轻量级前向模拟器"→"近似前向评估器（Approximate Forward Evaluator）" | 全文 |
| 8 | 参数标注"首期默认值" | §7.4 步骤 2.3 |
| 9 | 搜索预算 15ms vs 全链路 20ms 关系显式说明 | §7.4 步骤 2.3, §7.5 |
| 10 | LLM 隐私声明 | §9.2 |
| 11 | 等价目标剪枝 + 节点预算 500 | §7.4 步骤 2.3 |
| 12 | CancellationToken 批次检查（每 32 节点） | §7.4 步骤 2.3 |
| 13 | 风险 #9-#11 | §13 |
