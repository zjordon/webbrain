# 📐 Phase 2：Plan-before-Act 规划门

> 📍 源码范围：`_processMessageInner`（L25159-25213）+ 规划器族（`_maybeRunPlannerGate` L10034、`_runPlannerGate` L11037、`_runPlannerIntentGate` L10839、`_runReadScopeClassifier` L10662、`_waitForPlanReview` L9552）
> 🎯 目标：在 Act/Dev 运行的**第一个浏览器工具**执行之前，用一次结构化 LLM 调用产出可审批的计划；批准后计划钉入 scratchpad、技能被激活、进度会话被建立——拒绝/超时/中止则运行在工具执行前停止。

## 📋 主要逻辑流程

| # | 子步骤 | 源码 | 职责 |
|---|---|---|---|
| 1 | [Trace 启动](2-规划门内部逻辑/1-Trace启动.md) | L25159-25177 | 规划门路径下 trace 先行启动，规划 LLM 调用记入本次运行 |
| 2 | [规划门执行](2-规划门内部逻辑/2-规划门执行.md) | L25179-25187 + L10034/L10839/L11037 | 模式选择（Off/Try/Strict）→ 规划 LLM 调用 → JSON 校验与修复 → 审批卡挂起 |
| 3 | [门后处置](2-规划门内部逻辑/3-门后处置.md) | L25188-25213 | 语言策略、responseOnly 短路、`_startPlanExecutionGuard`、进度会话建立 |

## 🔄 数据流图

```
enriched 首条消息 + messages 历史
  ↓ [子步骤1] _startTraceRun（规划路径先行）→ runId
  ↓ [子步骤2] _maybeRunPlannerGate
  │   ├─ readScopePreflight? → _runReadScopeClassifier（L10662）
  │   ├─ planBeforeAct=off  → _runPlannerIntentGate（L10839，紧凑意图 schema）
  │   └─ try/strict        → _runPlannerGate（L11037，完整规划 schema）
  │        ↓ 一次规划 LLM 调用（planner.js 提示词，phase:'planner' 追踪）
  │        ↓ normalizePlan 边界清洗 → _waitForPlanReview（L9552）
  │        ↓ 用户批准/编辑 → formatPlanScratchpad 钉入
  ↓ GateOutcome {proceed, reason?, message?, responseOnly?, skillIds,
  │              progressLedgerPolicy, progressAction, responseLanguagePolicy}
  ├─ proceed=false → finalResponse = message（cancelled/plan_only/cost_limit）
  ├─ responseOnly → _completeResponseOnlyTurn（只读短回合）
  └─ proceed → [子步骤3] _startPlanExecutionGuard + _ensureProgressSessionForCurrentTask
       ↓ 激活 skillIds（_activateSkillsForRun L13559）
→ Phase 3（主循环）
```

## 💡 设计要点

1. **门在工具前，不在回合前**：拒绝/超时/中止都能在"任何浏览器工具执行前"停止运行（阶段一 4.3 状态机）。
2. **三模式降级语义**：Off 用紧凑意图 schema；Try（默认）完整 schema、无效 JSON 只降级本回合为 Ask；Strict 无效即停——失败的规划**不能授权动作**。
3. **计划抗压缩**：批准的计划经 `formatPlanScratchpad()` 变成 `[Approved plan]` scratchpad 条目，`_manageContext` 压缩时始终保留。
4. **规划器输入最小化**：用户任务 + 消毒 URL/标题 + 1500 字符历史摘要（`_buildPlannerHistoryDigest` L9726）；页面上下文包为不可信数据、图像块丢弃。
5. **调度任务可自动批准**：`scheduledRunPolicies.autoApprovePlanReview`（L1576）允许无人值守计划直接钉入。

## 📊 总结

| 维度 | 结论 |
|---|---|
| 触发条件 | Act/Dev 手动运行（非云运行、非 standalone）或读范围预检 |
| 规划模式 | off（意图门）/ try（默认，降级 Ask）/ strict（停） |
| 审批交互 | `plan_review` 卡片 → `submitPlanResponse`（L9528）resolve Promise |
| 产物 | 批准计划 scratchpad + 激活技能 + 进度会话 + GateOutcome |
| 详细文档 | 3 篇（见上表链接） |
