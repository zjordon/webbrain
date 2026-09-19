# 🧠 Phase 3：主循环 — LLM 决策

> 📍 源码范围：`_processMessageInner`（L25214-25432 组装 + L25385-25556 循环前半）
> 🎯 目标：ReAct 循环的"思考半边"——每轮迭代组装工具目录与消息、发起 LLM 调用（含 Ask 流式与回退）、处理 LLM 错误（溢出/瞬时/成本）、解析响应并把文本形态的工具调用救回来。

## 📋 主要逻辑流程

| # | 子步骤 | 源码 | 职责 |
|---|---|---|---|
| 1 | [工具目录组装](3-主循环内部逻辑/1-工具目录组装.md) | L25214-25244 + L25398-25423 | `getToolsForMode` + 技能工具 + 每迭代重建 + 选区/standalone 清空 |
| 2 | [LLM 调用与流式回退](3-主循环内部逻辑/2-LLM调用与流式回退.md) | L25249-25364 + L25428-25479 | `chatMainTurn` → 流式资格判定 → `_chatStreamWithCostAllowance` → 失败回退非流式 |
| 3 | [LLM 错误处理与重试](3-主循环内部逻辑/3-LLM错误处理与重试.md) | L25480-25543 | 成本耗尽即停；上下文溢出→紧急裁剪重试一次；瞬时错误→2s 后重试一次 |
| 4 | [响应解析与文本兜底](3-主循环内部逻辑/4-响应解析与文本兜底.md) | L25546-25622 | abort 检查点②、LFM 原生搜索标记、文本形态工具调用解析、恢复标志重置、成本消息拦截 |

## 🔄 数据流图

```
messages + provider + mode
  ↓ [子步骤1] 每迭代：
  │   _maybeReinjectAdapter（steps>0 跨站点）
  │   skillTools = _skillToolDefinitions(...)（活跃技能动态）
  │   tools = getToolsForMode(mode, {tier, skillTools, webMcp, cloudRun, watchBeep})
  │   allowedToolNames / toolSchemas 重建
  │   _manageContext（途中压缩，用上一步报告的 prompt_tokens）
  ↓ [子步骤2] steps++ → onUpdate('thinking')
  │   chatMainTurn(prunedMessages, {tools, temperature, maxTokens:4096})
  │     └─ Ask 流式资格 → text_delta 实时广播 → 终端事件 → {content, toolCalls, usage}
  │        └─ 传输失败 → 清空已发文本 + 禁流 + 一次非流式重试
  ↓ [子步骤3] 异常路径
  │   cost_limit → break；overflow → _emergencyTrim + 重试；transient → 2s + 重试
  ↓ [子步骤4] result
  ├─ usage.prompt_tokens → _lastInputTokens（下轮压缩依据）
  ├─ trace.recordLLMResponse（延迟按 shouldOrderInteractiveAskTrace 排队）
  └─ 文本工具调用兜底：_tryParseToolCallsFromText → result.toolCalls
→ Phase 4（toolCalls 非空）/ Phase 5（纯文本）
```

## 💡 设计要点

1. **工具表每迭代重建**（L25402-25416）：技能在循环中被 `load_skill` 激活后，下一次模型调用立刻看到新工具表——动态暴露与循环结构解耦。
2. **温度确定性**：动作模式 0.15、Ask 0.3（L25235 `plannerTemperature`）——浏览器控制决策偏确定性。
3. **流式是集成不是旁路**（阶段一 5.1）：Ask 流式聚合在 `processMessage` 生产生命周期内，传输失败静默重试一次而非整轮报废。
4. **错误分类三分法**：成本（立即停）/ 溢出（裁剪重试）/ 瞬时（延时重试）——每类最多重试一次，不无限重试。
5. **token 反馈环**：`usage.prompt_tokens` 记入 `_lastInputTokens`，下轮 `_manageContext` 用"增量字符"判断压缩时机（L25452-25457 注释）。

## 📊 总结

| 维度 | 结论 |
|---|---|
| 循环条件 | `while (steps < this.maxSteps)`，maxSteps=130（可配 195） |
| 每迭代开销 | 工具表重建 + 途中压缩 + abort 检查 |
| LLM 调用点 | L25451（主）+ L25498（溢出重试）+ L25527（瞬时重试） |
| 流式 | 仅 Ask 交互回合；失败自动降级 |
| 详细文档 | 4 篇（见上表链接） |
