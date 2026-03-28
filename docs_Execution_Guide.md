# AITTK 执行指南：VS Code + GitHub Copilot 开发实战

> **定位**：本文档是 `docs_Plan_v7.1.md` 的执行层配套文档，专注于"怎么做"而非"做什么"。
> 所有架构决策和验收标准以 Plan v7.1 为准，本文档补充具体操作步骤、Copilot 提示词和可执行检查清单。

---

## 目录

1. [开发环境搭建](#1-开发环境搭建)
2. [GitHub Copilot 使用策略](#2-github-copilot-使用策略)
3. [Phase 1 执行：状态标准化与采集完善](#3-phase-1-执行状态标准化与采集完善)
4. [Phase 2 执行：规则推荐核心（出牌）](#4-phase-2-执行规则推荐核心出牌)
5. [Phase 3 执行：游戏内浮层 UI](#5-phase-3-执行游戏内浮层-ui)
6. [Phase 4 执行：异步 LLM 复盘](#6-phase-4-执行异步-llm-复盘)
7. [Phase 5 执行：RunContext 与路线推荐](#7-phase-5-执行runcontext-与路线推荐)
8. [Phase 6 执行：选卡 / 事件 / 药水 / 遗物推荐](#8-phase-6-执行选卡--事件--药水--遗物推荐)
9. [回归测试与质量保障工作流](#9-回归测试与质量保障工作流)
10. [立即可执行的下一步](#10-立即可执行的下一步)

---

## 1. 开发环境搭建

### 1.1 必装 VS Code 扩展

| 扩展 | 用途 |
|---|---|
| **C# Dev Kit** (`ms-dotnettools.csdevkit`) | C# 语言服务、IntelliSense、调试 |
| **GitHub Copilot** (`github.copilot`) | AI 代码补全 |
| **GitHub Copilot Chat** (`github.copilot-chat`) | 对话式代码生成 |
| **Godot Tools** (`geequlim.godot-tools`) | GDScript/场景文件支持（可选，当前为纯 C# Mod） |
| **Todo Tree** (`gruntfuggly.todo-tree`) | 追踪代码中的 `// TODO:` 标记 |
| **EditorConfig** (`editorconfig.editorconfig`) | 统一代码格式 |

安装命令（在 VS Code 终端执行）：

```bash
code --install-extension ms-dotnettools.csdevkit
code --install-extension github.copilot
code --install-extension github.copilot-chat
code --install-extension gruntfuggly.todo-tree
```

### 1.2 推荐 VS Code 工作区设置

在项目根目录创建 `.vscode/settings.json`：

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "ms-dotnettools.csdevkit",
  "omnisharp.enableRoslynAnalyzers": true,
  "omnisharp.enableEditorConfigSupport": true,
  "csharp.suppressDotnetRestoreNotification": false,
  "todo-tree.general.tags": ["TODO", "FIXME", "HACK", "PERF", "THREAD"],
  "todo-tree.highlights.defaultHighlight": {
    "foreground": "white",
    "background": "red",
    "icon": "alert",
    "type": "tag"
  },
  "github.copilot.enable": {
    "*": true,
    "plaintext": false,
    "markdown": true
  }
}
```

### 1.3 推荐 `.editorconfig`

在项目根目录创建 `.editorconfig`：

```ini
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = crlf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

# 命名规范
dotnet_naming_rule.private_fields.symbols = private_fields
dotnet_naming_rule.private_fields.style = underscore_prefix
dotnet_naming_rule.private_fields.severity = suggestion

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_style.underscore_prefix.required_prefix = _
dotnet_naming_style.underscore_prefix.capitalization = camel_case

[*.json]
indent_style = space
indent_size = 2
```

### 1.4 调试启动配置

在 `.vscode/launch.json` 中添加（可与现有启动配置合并）：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to STS2",
      "type": "coreclr",
      "request": "attach",
      "processName": "SlayTheSpire2"
    },
    {
      "name": "Run Tests (xUnit)",
      "type": "coreclr",
      "request": "launch",
      "program": "dotnet",
      "args": ["test", "--logger", "console;verbosity=detailed"],
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

### 1.5 目录结构规范

按 Plan v7.1 §14 文件清单，在项目中预创建以下子目录（每个放置 `.gitkeep`）：

```
Scripts/
├── AI/
│   ├── CardReward/
│   ├── Evaluation/
│   ├── Event/
│   ├── Generation/
│   ├── Potion/
│   ├── Recommendation/
│   ├── Relic/
│   ├── Review/
│   └── Route/
├── Collectors/
├── Compatibility/
├── Config/
├── Data/
├── Debug/
├── Models/
├── Tests/
│   ├── GoldenCases/
│   └── Snapshots/
└── UI/
```

---

## 2. GitHub Copilot 使用策略

### 2.1 核心原则

- **给上下文，不给指令**：在文件顶部写清楚"这个文件的职责是什么、输入是什么、输出是什么"，Copilot 的补全质量会显著提升。
- **用 Chat 做架构，用补全做实现**：复杂接口设计用 `Ctrl+I`（Inline Chat）或侧边栏 Chat，具体方法体实现用 Tab 补全。
- **分步生成**：每次只让 Copilot 写一个方法，验证后再继续，避免累积错误。

### 2.2 高效 Copilot Chat 提示词模板

**生成数据模型**：

```
根据以下字段列表，生成 C# record 类型，要求不可变（init-only），
包含 SchemaVersion 静态常量（值为 1），
所有集合字段使用 ImmutableArray<T>，
添加 [System.Serializable] 特性：

[粘贴字段列表]
```

**生成接口 + 实现骨架**：

```
创建接口 I{Name} 和默认实现 {Name}，
接口方法签名为：{方法签名}，
实现类构造函数接受 {依赖列表}，
所有异常必须在 try-catch 中捕获并通过 AITTKLogger 记录，
不要抛出到调用方。
```

**生成单元测试**：

```
为 {ClassName}.{MethodName} 生成 xUnit 测试，
测试场景：{场景描述}，
使用 Arrange-Act-Assert 格式，
断言使用 FluentAssertions。
```

**重构现有代码（热路径优化）**：

```
以下方法在热路径中被调用（每帧最多 4 次），
请将所有 List<T>/Dictionary<T> 分配替换为：
- 固定大小数组（上限已知的情况）
- ArrayPool<T> 租用（临时缓冲区）
目标：零堆分配。
```

### 2.3 Copilot 的局限性与补救

| 场景 | Copilot 表现 | 补救方法 |
|---|---|---|
| 游戏 API（`CombatManager` 等） | 可能臆造不存在的 API | 在文件顶部用注释声明"以下是已验证的 API 列表" |
| 跨线程安全 | 可能生成不安全的写法 | 在注释中明确"必须使用 `Interlocked.Exchange`，禁止 `lock`" |
| 性能敏感代码 | 默认生成带分配的写法 | 在注释中标注 `// HOT PATH: zero allocation required` |
| Godot Node 生命周期 | 可能忽略 `IsInstanceValid()` | 在文件顶部添加"所有节点访问前必须 `IsInstanceValid()` 检查" |

---

## 3. Phase 1 执行：状态标准化与采集完善

> **目标**：可运行的 Snapshot + 序列化 + 日志 + API 兼容检查
> **参考**：Plan v7.1 §6

### 3.1 任务清单（P1）

- [ ] **P1-T1** 创建 `Scripts/Debug/AITTKLogger.cs`（无外部依赖，先做）
- [ ] **P1-T2** 创建 `Scripts/Compatibility/ApiCompatibilityChecker.cs`
- [ ] **P1-T3** 创建 Snapshot 数据模型（6 个文件，见下）
- [ ] **P1-T4** 创建 `Scripts/AI/CombatAnalyzer.cs`
- [ ] **P1-T5** 创建 `Scripts/Debug/CombatSnapshotSerializer.cs`
- [ ] **P1-T6** 创建 `Scripts/Debug/SnapshotMigrator.cs`
- [ ] **P1-T7** 修改 `Scripts/Collectors/CombatCollector.cs`，集成 Snapshot 构建
- [ ] **P1-T8** 修改 `Scripts/Entry.cs`，调用 ApiCompatibilityChecker

### 3.2 P1-T1：AITTKLogger — 实现要点

**文件**：`Scripts/Debug/AITTKLogger.cs`

实现要点（告知 Copilot）：

```csharp
// 职责：统一日志层
// - 支持 Trace / Debug / Info / Warn / Error 五个级别
// - 日志节流：60 秒内同类型（相同 tag + level）最多记录 3 条
// - 输出方式：调用游戏的 GD.Print / GD.PrintErr（或 Console.WriteLine 作为降级）
// - 线程安全：节流计数器使用 ConcurrentDictionary 或 lock 保护
// - 静态类，无需实例化
// 示例：AITTKLogger.Warn("CombatCollector", "Snapshot build failed", ex);
```

**节流实现提示**：

```csharp
// 节流 key = $"{tag}:{level}:{message的前32字符}"
// 节流状态 = (Count: int, FirstSeenAt: long)
// 检查：若 Count < 3 或 (now - FirstSeenAt) > 60_000ms，则允许输出并更新计数
```

### 3.3 P1-T3：Snapshot 数据模型 — 实现顺序

按依赖关系从底向上创建，每个文件先写，后让 Copilot 填充字段：

1. `PowerInfo.cs` — 无依赖
2. `CardInstanceInfo.cs` — 无依赖
3. `PlayerStateInfo.cs` — 依赖 PowerInfo, CardInstanceInfo
4. `EnemyStateInfo.cs` — 依赖 PowerInfo
5. `CombatContextInfo.cs` — 无依赖
6. `CombatSnapshot.cs` — 聚合以上所有

**CombatSnapshot 文件头注释模板**（帮助 Copilot 理解上下文）：

```csharp
// CombatSnapshot — 单回合战斗状态的纯数据快照
// 职责：忠实记录当前原始状态，不混入策略判断
// 不变性：构建后只读，所有集合为 ImmutableArray<T>
// 版本化：SchemaVersion 在字段结构变更时递增，当前为 1
// 序列化：支持 System.Text.Json，通过 CombatSnapshotSerializer 导出/导入
// 线程安全：纯数据对象，创建后线程安全
// 来源：由 CombatCollector 在主线程构建，传递给后台推荐线程
```

### 3.4 P1-T4：CombatAnalyzer — 分析逻辑要点

**文件头注释**：

```csharp
// CombatAnalyzer — 从 CombatSnapshot 计算推荐所需的衍生特征
// 输入：CombatSnapshot（纯数据，线程安全）
// 输出：DerivedCombatAnalysis（也是纯数据，不可变）
// 性能：在后台线程运行，非热路径，允许适量分配
// 不做：不访问游戏 runtime 对象，不修改任何状态
```

**核心计算项**（告知 Copilot 需要实现的内容）：

- `TotalExpectedIncomingDamage`：对所有敌人 `AttackIntent` 求和（减去当前 Block）
- `PlayableCards`：手牌中 `IsPlayable = true` 的集合（基于能量 >= 费用）
- `EstimatedDamageMap`：每张攻击牌对每个目标的估算伤害（含 Strength/Vulnerable 修正）
- `HighThreatTargets`：意图伤害高且 HP 低的敌人
- `AoeOpportunity`：存活敌人 >= 2 且有 AOE 牌时为 true

### 3.5 P1 验证方法

在游戏中触发战斗，检查日志输出：

```
[AITTK][Info][CombatCollector] CombatSnapshot built: SchemaVersion=1, Cards=5, Enemies=2
[AITTK][Info][CombatAnalyzer] Analysis: TotalIncoming=12, Playable=4, HighThreat=1
```

导出快照到文件并重新导入，验证字段完整性：

```csharp
// 在 DevConfig.ExportSnapshots = true 时，每回合将快照写入：
// %AppData%/AITTK/snapshots/combat_{timestamp}.json
```

---

## 4. Phase 2 执行：规则推荐核心（出牌）

> **目标**：可运行的出牌推荐，满足 <20ms 全链路性能预算
> **参考**：Plan v7.1 §7

### 4.1 任务清单（P2）

- [ ] **P2-T1** 创建 `IPlayRecommender.cs` + `PlayRecommendation.cs`
- [ ] **P2-T2** 创建 `ValueFunction.cs`（评分维度定义）
- [ ] **P2-T3** 创建 `FastHeuristic.cs`（粗排，零分配目标）
- [ ] **P2-T4** 创建 `EvaluationScratchpad.cs`（四项核心乘区状态载体）
- [ ] **P2-T5** 创建 `CandidateSequenceGenerator.cs`（Beam Search）
- [ ] **P2-T6** 创建 `TargetEquivalenceComparer.cs`（等价剪枝）
- [ ] **P2-T7** 创建 `ActionSequenceEvaluator.cs`（精排）
- [ ] **P2-T8** 创建 `RuleBasedRecommender.cs`（编排）
- [ ] **P2-T9** 创建 `RecommendationManager.cs`（生命周期管理）
- [ ] **P2-T10** 编写 Golden Cases 离线测试

### 4.2 P2-T3：FastHeuristic — 零分配实现要点

**文件头注释**（用于 Copilot）：

```csharp
// FastHeuristic — 搜索阶段快速粗排打分器
// HOT PATH: 在 Beam Search 的每个节点扩展时调用
// 性能约束：单次调用 < 0.5μs，零堆分配
// 输入：固定大小的栈数据（候选序列状态，用 ref struct 传递）
// 输出：float 分数（在调用方的栈上）
// 禁止：new List<>()、new Dictionary<>()、LINQ、装箱、string 拼接
// 技术：所有临时数据存放于 ref struct 或固定大小的 stackalloc 数组
```

**避免分配的常见技巧**（可直接贴给 Copilot）：

```csharp
// 替代 List<int>：使用固定大小数组或 Span<int> + stackalloc
// 替代 LINQ .Where().Select()：使用 for 循环 + 局部变量
// 替代 string.Format()：仅在非热路径的日志中使用
// 替代 new CardState()：使用 readonly struct 或传递 ref 参数
```

### 4.3 P2-T5：Beam Search 实现要点

**文件头注释**：

```csharp
// CandidateSequenceGenerator — 两阶段 Beam Search 候选出牌序列生成器
// 职责：在合法性约束下生成候选出牌序列（不评分、不排序）
// 参数（来自 DevConfig）：
//   BeamWidth = 5, MaxDepth = 10, MaxNodes = 500, TimeBudgetMs = 15
// 非确定性截断：抽牌+洗牌、随机生成牌时终止该分支
// CancellationToken 检查：每 32 个节点批次检查一次
// 等价剪枝：通过 TargetEquivalenceComparer 跳过等价序列
// 热路径：后台线程，接近零分配（使用 ArrayPool + 固定数组）
```

**Beam Search 框架代码骨架**（Copilot Inline Chat 提示）：

```
实现 BFS/Beam Search，层级为手牌出牌序列，
每层保留 Top BeamWidth 个节点（通过 FastHeuristic 打分），
搜索停止条件：
  1. 所有叶子均已无牌可出或能量耗尽
  2. 达到 MaxDepth
  3. 节点数超过 MaxNodes
  4. CancellationToken 触发（每 32 节点检查）
  5. 超过 TimeBudgetMs
输出：List<CandidateSequence>（最多 15 个，进入精排）
```

### 4.4 P2-T6：等价目标剪枝

当手牌有多张相同的牌（或有多个等价目标），Beam Search 会产生重复路径。`TargetEquivalenceComparer` 需要：

```csharp
// 两个候选节点被认为等价的条件（ALL 满足）：
// 1. 当前 Player HP 相同
// 2. 当前 Player Block 相同
// 3. 所有敌人的 (HP, Block, Vulnerable, Weak, Poison) 元组集合相同（顺序无关）
// 4. 剩余手牌集合相同（不考虑顺序）
// 5. 剩余能量相同
// 实现：重写 GetHashCode() 基于以上字段的组合 hash
```

### 4.5 P2-T9：RecommendationManager — 关键参数

```csharp
// RecommendationManager — 推荐计算的生命周期管理器
// 防抖窗口：80ms（游戏事件触发后等待此时间再触发重算）
// 最大重算频率：4 次/秒（250ms 最小间隔）
// 结果粘性：500ms（推荐结果最短显示时间，避免闪烁）
// 并发模式：后台 Task + CancellationTokenSource
//           新计算启动时取消上一次未完成的计算
// 结果发布：Interlocked.Exchange 原子替换，UI 主线程轮询
// Dirty flags：订阅 TurnStarted / HandChanged 事件时置 dirty
// Snapshot hash：通过简单字段 hash 去重，避免无变化时重算
```

### 4.6 P2 性能验证方法

在 DevConfig 中启用 `EnablePerformanceProfiling = true`，观察日志：

```
[AITTK][Debug][RecommendationManager] Calc #42: search=11ms, eval=2ms, total=13ms ✓
[AITTK][Debug][RecommendationManager] Calc #43: search=18ms, eval=3ms, total=21ms ⚠ >20ms
```

使用 VS Code 的 `.NET Memory Dump` 工具检查 Gen1+ GC：
- `dotnet-counters monitor --process-id <PID> System.Runtime`
- 关注 `gen-1-gc-count` 在战斗中的增长速率

---

## 5. Phase 3 执行：游戏内浮层 UI

> **目标**：稳定、低打扰的栈式多域 Overlay
> **参考**：Plan v7.1 §8

### 5.1 任务清单（P3）

- [ ] **P3-T1** 创建 `AITTKConfig.cs`（UserConfig + DevConfig，无 Godot 依赖）
- [ ] **P3-T2** 创建 `OverlayManager.cs`（栈式状态机核心）
- [ ] **P3-T3** 创建 `CombatOverlay.cs`（出牌推荐展示）
- [ ] **P3-T4** 在 `Entry.cs` 中注册 OverlayManager + 订阅场景变化事件

### 5.2 P3-T2：OverlayManager 线程安全要点

**文件头注释**：

```csharp
// OverlayManager — 栈式 Overlay 状态机
// 必须在主线程操作（Push/Pop），因为涉及 Godot Node 操作
// 栈最大深度：4（来自 DevConfig.OverlayStackDepth）
// 超过上限：清空栈并重建当前场景对应的主 Overlay
// 场景检测：订阅游戏场景变化事件，主场景切换时清栈
// IsInstanceValid()：所有 Node 引用访问前必须检查
```

**_Process 方法规范**（务必告知 Copilot）：

```csharp
public override void _Process(double delta)
{
    // 规则：第一行必须将全局引用捕获为局部变量
    var result = Interlocked.CompareExchange(ref _latestResult, null, null);
    if (result == null) return;

    // 后续所有读取都基于局部变量 result，不再读取全局字段
    if (result.Version == _displayedVersion) return;

    UpdateDisplay(result);  // 仅在版本变化时更新
    _displayedVersion = result.Version;
}
```

### 5.3 P3 验证检查清单

- [ ] 游戏内浮层默认显示在屏幕右侧
- [ ] 分辨率切换后位置正确（Godot Anchor 系统）
- [ ] 关闭 AI 辅助总开关后，浮层完全消失（无残留节点）
- [ ] 战斗→地图场景切换时，正确切换 Overlay 类型
- [ ] 多层 Overlay（如战斗中打开药水面板）Z-Index 正确
- [ ] 模拟栈超限：正确触发清空并重建

---

## 6. Phase 4 执行：异步 LLM 复盘

> **目标**：Ollama 本地 LLM 异步复盘，不影响主流程
> **参考**：Plan v7.1 §9

### 6.1 任务清单（P4）

- [ ] **P4-T1** 创建 `LLMReviewClient.cs`（Ollama HTTP 客户端）
- [ ] **P4-T2** 创建 `LLMPromptBuilder.cs`（Prompt 构建）
- [ ] **P4-T3** 创建 `LLMDomainAdvisor.cs`（非战斗域统一接口）
- [ ] **P4-T4** 创建 `TacticalReviewService.cs`（编排复盘流程）

### 6.2 P4-T1：LLMReviewClient 关键实现

**HTTP 客户端配置**：

```csharp
// LLMReviewClient — 调用本地 Ollama HTTP API
// 基地址：http://localhost:11434（来自 DevConfig.OllamaBaseUrl）
// 超时：5000ms（来自 DevConfig.LlmTimeoutMs）
// 接口：POST /api/generate（Ollama REST API）
// 错误处理：HttpRequestException / OperationCanceledException → 记录 Warn，返回 null
// 绝不向云端发送数据，仅访问 localhost
// 使用 static readonly HttpClient（不要在每次调用时创建 HttpClient）
```

**Prompt 结构**（帮助 Copilot 生成 LLMPromptBuilder）：

```
系统角色：你是一位《杀戮尖塔2》战术分析师，擅长分析牌组和战斗决策。

输入（< 800 Token）：
- 回合状态摘要（HP/Block/能量/手牌）
- 敌人意图摘要
- 实际出牌序列
- 规则引擎推荐序列（如有）
- 结果（实际效果）

输出要求（< 500 Token）：
- 3 条战术点评（每条 1-2 句）
- 1 个替代思路（如果有更优解）
- 1 个风险提示（如果有）
- 格式：JSON，键名为 "comments", "alternative", "risk"
```

### 6.3 P4 验证方法

1. 启动 Ollama：`ollama run qwen2.5:7b`
2. 在 UserConfig 中启用 `EnableAsyncReview = true`
3. 战斗结束后，等待 3-5 秒，复盘面板应出现
4. 关闭 Ollama，验证 Mod 正常运行（降级显示"LLM 不可用"）

---

## 7. Phase 5 执行：RunContext 与路线推荐

> **目标**：稳定的跨局全局状态，含 Hydration 重建能力
> **参考**：Plan v7.1 §10

### 7.1 任务清单（P5）

- [ ] **P5-T1** 创建 RunContext 相关数据模型（5 个文件）
- [ ] **P5-T2** 创建 `RunContextCollector.cs`（含 Hydration 逻辑）
- [ ] **P5-T3** 创建 `DeckFunctionalAnalyzer.cs`
- [ ] **P5-T4** 创建 `ThreatWindowAnalyzer.cs`
- [ ] **P5-T5** 创建路线推荐相关文件（5 个文件）

### 7.2 P5-T2：RunContextCollector Hydration 实现要点

Hydration 是 P5 最关键也最易出错的部分，务必按以下顺序实现：

**步骤一：实现全量重建（FullRebuild）**

```csharp
// 全量重建流程（必须在主线程执行，因为需要访问 Player 等游戏对象）：
// 1. 从 Player.Deck 重建 DeckComposition
// 2. 从 Player.Relics 重建 RelicInventory
// 3. 从 Player.Potions 重建 PotionInventory
// 4. 从 Player（HP/Gold）重建 PlayerRunState
// 5. 从 IRunState（Act/Floor/Map）重建 RunProgress + MapState
// 6. 调用 ThreatWindowAnalyzer 重建 ThreatWindow
// 7. HistoricalPerformance → 填默认值，标记 IsHistoricalDataApproximate = true
// 8. 标记 IsHydrated = true, HydrationSource = "FullRebuild"
```

**步骤二：实现一致性校验**

```csharp
// 校验方法（进入每个新场景时调用）：
// - RunContext.DeckComposition.TotalCardCount == Player.Deck.Count？
// - RunContext.RelicInventory.Length == Player.Relics.Count？
// - RunContext.PlayerRunState.CurrentHp == Player.CurrentHp？
// 任何不一致 → AITTKLogger.Warn + 触发全量重建
// 注意：Save&Quit 后 RunContext 可能为 null，此时也触发全量重建
```

### 7.3 P5-T3/T4：功能分析器实现要点

**DeckFunctionalAnalyzer 关键指标计算**（告知 Copilot）：

```csharp
// SingleTargetDamagePerTurn：
//   sum(攻击牌.EstimatedBaseDamage) * (5 / 平均牌组抽取速度)
//   简化：假设每回合抽 5 张，计算期望能出的攻击牌数量 × 平均伤害

// BlockPerTurn：
//   sum(防御牌.EstimatedBaseBlock) * (5 / 平均牌组抽取速度)

// HasReliableAoe：
//   牌组中有 AOE 牌（TargetType = AllEnemies）且数量 >= 2

// ScalingPotential：
//   含 Strength/Dexterity/Poison 等 Power 关键词的牌的数量
```

---

## 8. Phase 6 执行：选卡 / 事件 / 药水 / 遗物推荐

> **目标**：六大决策域全部可用
> **参考**：Plan v7.1 §11

### 8.1 任务清单（P6）

- [ ] **P6-T1** 选卡推荐（4 个文件 + 威胁驱动评估）
- [ ] **P6-T2** 事件推荐（6 个文件 + EventEffects.json 初始数据）
- [ ] **P6-T3** 药水推荐（4 个文件）
- [ ] **P6-T4** 遗物评估（6 个文件 + RelicTags.json 初始数据）

### 8.2 P6-T1：威胁驱动选卡推荐 — 评估流程

**CardRewardRecommender 评估逻辑**（用于 Copilot Chat）：

```
为每张候选卡按以下优先级顺序打分（高优先级决策不被低优先级覆盖）：

1. 生存底线检查（ThreatLevel == Critical）：
   若该牌直接解决 SurvivalGaps 中的缺口 → +200 分（大幅加分）

2. 威胁解决价值：
   遍历 SurvivalGaps 中的每个 gap，
   若该牌与 gap 匹配（如 NeedMoreBlock → 防御牌）→ +100 × 匹配程度

3. 地板价值（通用强度）：
   基于 CardRarity 和 CardType 的基础分（Rare=60, Uncommon=40, Common=20）

4. 牌组膨胀惩罚：
   若 DeckComposition.TotalCardCount > 20 → 所有牌 -30 分，"跳过" +50 分
   若 > 25 → 所有牌 -50 分，"跳过" +80 分

5. 协同加分（权重上限 20%，即最高加分 = 总分 × 20%）：
   与当前遗物/已有牌的组合 → 适量加分，但不能超过上限
```

### 8.3 P6-T2：EventEffects.json 初始数据填充建议

首期至少收录以下高频事件（使用 Copilot Chat 生成 JSON）：

```
生成以下杀戮尖塔2事件的 EventEffects.json 条目，
格式参考（见 Plan §11.2），
只记录客观效果（不含评估逻辑）：
- Big Fish（大鱼）
- The Cleric（牧师）
- Dead Adventurer（死去的冒险者）
- The Mausoleum（陵墓）
- Hypnotizing Colored Mushrooms（彩色蘑菇）
```

### 8.4 P6-T4：RelicTags.json 初始标签填充建议

首期至少为以下遗物类型添加标签：

```
生成以下遗物类型的 RelicTags.json 条目，
格式参考（见 Plan §11.4），
标签分类：
- 触发条件标签（OnAttackPlayed, OnSkillPlayed, OnTurnStart, Passive）
- 效果标签（GrantsStrength, GrantsDexterity, GrantsEnergy, DrawCards）
- 需求标签（NeedsHighAttackCount, NeedsLowHp, NeedsFullEnergy）
```

---

## 9. 回归测试与质量保障工作流

> **参考**：Plan v7.1 §12

### 9.1 测试项目设置

在解决方案中创建独立的测试项目：

```xml
<!-- AITTK.Tests/AITTK.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="xunit" Version="2.9.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.*" />
    <PackageReference Include="FluentAssertions" Version="6.*" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.*" />
  </ItemGroup>
  <ItemGroup>
    <!-- 只引用不依赖游戏 runtime 的纯 C# 文件 -->
    <Compile Include="..\Scripts\AI\**\*.cs" />
    <Compile Include="..\Scripts\Models\**\*.cs" />
    <Compile Include="..\Scripts\Debug\**\*.cs" />
  </ItemGroup>
</Project>
```

> **注意**：依赖 Godot Node / 游戏 runtime API 的代码无法在独立测试项目中运行，
> 仅对纯 C# 数据模型和算法层编写单元测试。

### 9.2 Golden Cases 文件结构

```
Scripts/Tests/GoldenCases/
├── Combat/
│   ├── must_defend_lethal.json          # 必须优先防御场景
│   ├── lethal_opportunity.json          # 可斩杀场景
│   ├── aoe_optimal.json                 # AOE 最优场景
│   ├── combo_order_buff_then_attack.json
│   └── nondeterministic_truncation.json
├── Route/
│   ├── low_hp_prefer_rest.json
│   └── gold_available_prefer_shop.json
└── CardReward/
    ├── archetype_trap_test.json         # 流派陷阱测试
    ├── skip_thick_deck.json             # 跳过测试
    └── survival_gap_test.json           # 生存缺口测试
```

**Golden Case JSON 格式**（出牌域示例）：

```json
{
  "description": "必须优先防御：敌人致死攻击，手牌有防御牌",
  "schemaVersion": 1,
  "snapshot": { "...": "CombatSnapshot 的 JSON 内容" },
  "expectedTopAction": {
    "cardTitle": "Defend",
    "targetId": null,
    "reason": "prevent_lethal"
  },
  "tolerance": "exact_card_type"
}
```

### 9.3 VS Code 测试运行工作流

```bash
# 运行所有单元测试
dotnet test AITTK.Tests/ --logger "console;verbosity=normal"

# 运行特定 Golden Case
dotnet test AITTK.Tests/ --filter "Category=GoldenCase&Domain=Combat"

# 性能测试（BenchmarkDotNet）
dotnet run --project AITTK.Benchmarks/ --configuration Release
```

在 VS Code 中安装 `.NET Test Explorer` 扩展可以在侧边栏直接运行和查看测试结果。

---

## 10. 立即可执行的下一步

> **当前状态**：CombatCollector 日志验证通过，编译 0 错误 0 警告
> **下一个 PR 目标**：完成 Phase 1（M1 里程碑）

### 10.1 今天可以开始的 3 个任务

**任务 A（30 分钟）：创建 AITTKLogger.cs**

1. 打开 VS Code，新建文件 `Scripts/Debug/AITTKLogger.cs`
2. 在文件顶部写入[第 3.2 节的注释](#32-p1-t1aittkklogger--实现要点)
3. 按 `Ctrl+I` 调用 Copilot Inline Chat，输入："基于以上注释，实现 AITTKLogger 静态类，包含 Trace/Debug/Info/Warn/Error 五个静态方法，带节流逻辑"
4. 验证：确认节流逻辑使用 `ConcurrentDictionary`，确认没有未处理的线程安全问题

**任务 B（45 分钟）：创建 CombatSnapshot 数据模型**

1. 按[第 3.3 节](#33-p1-t3snapshot-数据模型--实现顺序)的顺序，从 PowerInfo.cs 开始
2. 每个文件：先写文件头注释，再用 Copilot Chat 生成字段
3. 参考 Plan v7.1 §5.3 中的字段树确认字段完整性
4. 最后验证：所有集合字段使用 `ImmutableArray<T>`，包含 `SchemaVersion = 1` 常量

**任务 C（20 分钟）：创建 ApiCompatibilityChecker.cs**

1. 新建文件 `Scripts/Compatibility/ApiCompatibilityChecker.cs`
2. Copilot Chat 提示："实现 ApiCompatibilityChecker，遍历 Plan §2.5 中的兼容性检查清单，每项用 Type.GetProperty()/GetMethod() 反射探测，失败则调用 AITTKLogger.Error 并按域禁用推荐"
3. 在 `Entry.cs` 中调用 `ApiCompatibilityChecker.RunChecks()` 并处理返回结果

### 10.2 本周目标（M1 完成）

| 日期 | 目标 |
|---|---|
| Day 1 | AITTKLogger + ApiCompatibilityChecker（任务 A + C） |
| Day 2 | CombatSnapshot 全部数据模型（任务 B）|
| Day 3 | CombatAnalyzer（DerivedCombatAnalysis 计算逻辑） |
| Day 4 | CombatSnapshotSerializer + SnapshotMigrator |
| Day 5 | 修改 CombatCollector 集成 Snapshot 构建 + 端到端验证 |

### 10.3 常见踩坑与提醒

| 踩坑 | 预防方法 |
|---|---|
| Copilot 生成的 API 名不存在 | 每次生成后立即编译检查，不要累积错误 |
| 后台线程访问 Godot Node 崩溃 | 所有 Snapshot 构建放主线程，后台线程只接受纯 C# 对象 |
| 热路径无意引入 LINQ | 在 `FastHeuristic.cs` / `CandidateSequenceGenerator.cs` 顶部加 `// HOT PATH` 注释警示 |
| SchemaVersion 忘记递增 | 在 PR checklist 中加一条"是否修改了 Snapshot 字段？是否递增了 SchemaVersion？" |
| Overlay 在场景切换后崩溃 | 所有 `_Process` 开头必须 `IsInstanceValid()` 检查，加入代码审查 checklist |
| LLM 超时拖慢 UI 刷新 | LLM 调用必须在独立 Task 中，不得 `await` 在主线程 |

---

*本文档与 `docs_Plan_v7.1.md` 配套使用。架构决策、验收标准和字段定义以 Plan v7.1 为权威来源，本文档仅提供执行层的补充指导。*
