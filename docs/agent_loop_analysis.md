# `agent_loop.py` 源码分析

> 分析对象：`backend/app/core/agent_loop.py`（1719 行）
> 分析日期：2026-09-10

## 1. 定位与总体结论

`AgentLoop` 是聊天一轮对话（turn）的**外层门面与协调层**。从历史上看它是核心 ReAct/技能循环；引入 Harness v2 后，真正的执行流水线（加锁 → 幂等领取 → 规划 → TaskFrame 调度 → 隔离的 Harness 执行）已下沉到 `HarnessV2Engine`（`harness_v2_engine.py`）。

当前文件保留的职责是：

1. **入口适配**：对外暴露同步 `handle_turn` 与 SSE 流式 `handle_turn_stream`；
2. **异常归一化**：把引擎层各类异常翻译成统一的 `ChatTurnResponse` / 错误事件；
3. **SOP 图状态机辅助**：以薄封装方式委托 `GraphRules`，判定终态、管理挂起分支（pending steps）、跳转节点；
4. **人工转接（handoff）编排**：解析处理人、通知渠道并触发渠道私聊通知；
5. **会话/技能状态自愈**：清理不可见技能、修复失效步骤；
6. **收尾（finalize）**：落库 assistant 消息、引用规整、会话标题/摘要、渠道投递、事件记录；
7. **上下文与记忆配置**：会话历史上下文构建、LLM 摘要压缩参数、异步记忆采集入队。

构造时依赖：`db: Session`、可选 `event_sink`（事件外发回调）、可选 `stream_sink`；内部持有 `EventLog`、`SkillRuntime`、`ResponseGenerator`、`MemoryService`。

## 2. 两个入口与执行流程

### 2.1 `handle_turn(request) -> ChatTurnResponse`（agent_loop.py:166）

```
handle_turn
 └─ HarnessV2Engine(self).run(request)   # 成功时直接返回，引擎负责全部主流程
     成功 → ChatTurnResponse（直接返回）
     并发冲突 / 取消 / 前置错误 / LLM 错误 / 未知异常 → 本层捕获并归一化
```

- 引擎通过 `owner`（即 AgentLoop 实例）反向调用本类的大量辅助方法（如 `_get_request_model`、`_append_message`、`_drop_unavailable_skill_state`、`_finalize_turn` 等），两者是**双向协作**关系。
- 注意：本方法里的 `step_result = StepAgentResult(action="reply")` 只在**异常/兜底响应**中使用；成功路径由引擎内部构造响应。

异常处理矩阵：

| 异常 | 处理 | 用户侧表现 |
|---|---|---|
| `HarnessTurnConflict` / `HarnessSessionBusy` | rollback，记录 `turn_rejected`，commit | "Harness 并发或重复请求已阻止"，错误码 `HARNESS_TURN_CONFLICT` / `HARNESS_SESSION_BUSY` |
| `HarnessExecutionCancelled` | `engine.mark_cancelled()`，幂等落库一条"已停止生成"assistant 消息，清理取消标记 | 返回固定文案 `已停止生成` |
| `AgentLoopPreconditionError` / `SlashCommandError` | `mark_interrupted` 后走 `_finish_with_error` | "系统配置错误"，附带原始 code（如 `invalid_model_config`） |
| `LLMError` | 记录 `error_occurred`，`format_runtime_failure_reply` + 模型失败建议 | "模型调用失败"，code=`LLM_ERROR` |
| 其他 `Exception` | 同上，code=`HARNESS_V2_ERROR` | "Harness v2 执行出错" |

`finally` 块负责：turn 记录进入终态（completed/failed/cancelled）时清理取消标记，并调用 `engine.close()`。LLM/未知异常在捕获后统一落到方法尾部的 `_finalize_turn` → commit → 刷新会话 → 返回。

### 2.2 `handle_turn_stream(request)`（agent_loop.py:298）

委托给 `_handle_turn_stream_v2`，事件序列：

```
session_created?（仅新建会话）
user_message_received
status(planning, "正在规划本轮任务")
  └─ 内部完整调用 handle_turn() 拿到最终 reply
stream_cancelled | error （互斥终止分支）
stream_delta* （把完整 reply 用 ResponseGenerator.chunk_text 切块）
stream_end
complete（携带完整 ChatTurnResponse）
```

**关键事实：这是"伪流式"** —— 先一次性执行完整轮次拿到全文，再按块回放。事件中统一带 `execution_engine: "harness_v2"`。turn 归属通过 `HarnessTurnRecord.client_turn_id → user_message_id` 反查；无 client_turn_id 时退化为取最近一条 user 消息。

## 3. SOP 图（Graph）状态机相关

SOP 技能内容是一张节点/边图，规则全部由 `app.core.graph_rules.GraphRules` 提供，本类只做会话状态读写与事件记录。

- **挂起分支**：图的多分支待办节点列表存在 `slots_json["_graph_pending_steps"]`（常量 `GRAPH_PENDING_STEPS_SLOT`）中，与业务槽位共用一个 JSON 字段。
- `_apply_step_result`（agent_loop.py:869）是状态迁移核心：
  1. 合并 `slot_updates` 并记 `slot_updated`；
  2. LLM 返回的 `next_step_id` 若不在技能内 → 记 `step_agent_result_repaired` 修复事件并忽略（防幻觉跳转）；
  3. 同步 `awaiting_input_json`（问用户/澄清且有缺失字段时挂起；推进/工具/完成时清除）；
  4. 目标节点在 pending 中 → 合并激活；不在 → 入队 pending 并激活兄弟节点；首次分岔时 `_queue_graph_sibling_steps` 把其余出边节点排队；
  5. 无 next_step 但步骤完成时，尝试激活下一个 pending 节点。
- **完成判定** `_should_complete_skill`（agent_loop.py:586）优先级：
  1. 图上仍有未完成工作（pending 未清 / 自环 / 当前节点仍有出边）→ **不完成**（图拓扑权威，防止中间节点误结束整个 SOP）；
  2. 工具成功且当前节点允许收尾 → 完成；
  3. answer-ready（动作允许最终回复且 `required_info` 全满足）/ 终态节点 / 无下一步且无工具调用 → 完成。
- 终态判定结合 `terminal_node_ids` 与 `GraphRules.terminal_position_from_step`（动作 + 槽位）。
- 技能完成走 `_complete_active_skill` → `SkillRuntime.complete_current_skill`（支持技能栈恢复），记 `skill_completed`（含恢复到的 skill/step）。
- 轮末收尾 `_finalize_execution_after_reply` 整体委托 `TurnFinalizer.finalize`，以回调形式注入本类的 handoff/完成判定方法，返回 `"continued" | "completed" | "handoff"`。

## 4. 人工转接（Human Handoff）链路

入口在 `TurnFinalizer` 中通过回调触发，核心方法：

- `_maybe_route_to_handoff_node`（agent_loop.py:510）：step_result 要求 handoff 但当前节点未声明时，用 `GraphRules.find_handoff_node_id` 做 **BFS 找可达 handoff 节点**并路由过去（而非取数组第一个），以便从该节点读取处理人。
- `_create_human_handoff_request`（agent_loop.py:733）处理人解析优先级（在 `HumanHandoffService.create` 内）：
  1. 当前 SOP 节点的 `assignee_user_id`（handoff 类型节点或 `allowed_actions` 含 `handoff_human`）；
  2. 会话所属渠道绑定 `ChannelBinding.config_json.default_handoff_assignee_user_id`（按 `chat_session.channel_binding_id` 反查，而非 agent 挂载列表取首个）；
  3. 服务内进一步回退（agent/租户管理员）。
- 通知渠道 `assignee_notify_channel`：`None`=默认（网页 + 可达渠道私聊）、`"web"`=仅网页、具体渠道（如 feishu）=按渠道投递。
- `_maybe_notify_handoff_assignee`（agent_loop.py:802）：指定渠道时优先会话 binding，不匹配则在租户内找该渠道任一 active 员工绑定（`resolve_handoff_notify_binding`）；无可用绑定记 warning 跳过，**通知失败不影响转接主流程**（网页收件箱兜底）。

## 5. 会话与技能状态自愈

- `_drop_unavailable_skill_state`（agent_loop.py:1296）：每轮开始时基于"当前可见技能集合"修复脏状态——
  - 清空遗留的 `skill_stack_json` / `resume_after_answer_json`；
  - active 技能不可见 → 清空 active skill/step/slots/awaiting；
  - active 技能在但 step 已不存在 → 重置到首步；
  - `pending_tasks_json` 中按 skill_id 过滤失效帧；awaiting 指向失效技能则清除；
  - 有变更时记 `skill_state_pruned`（含移除技能列表与修复步骤）。
- `_finish_stale_completed_skill`：对处于终态的陈旧技能做收尾。
- `_get_or_create_session`：会话 channel 只保留 `PILOTDECK_GROUP_CHAT_CHANNEL` 与 `skill_test` 两个内部值，其余强制为 `None`。

## 6. 收尾与消息落库（`_finalize_turn`，agent_loop.py:1630）

1. 更新会话时间戳与状态（`handoff` 状态不被覆盖为 active）；
2. 组装 assistant 消息元数据（经 `ConversationProjection`，含知识引用去重）；
3. 引用后处理链：恢复被截断的原子引用 → 规范化引用标号 → 去除尾部引用清单 → 压缩引用标号（可能重写正文）；
4. 无标题会话用首条用户消息生成兜底标题；写 `summary = 最近回复：{reply[:120]}`；
5. 落库 assistant 消息（幂等 turn 关联：`user_message_id`/`turn_id`）；
6. **渠道投递**：仅当 `self.stream_delivery_succeeded == False` 时 `stage_channel_delivery`（流式已实时投递则不重复）；
7. 记 `assistant_message_created` 与 `session_state_changed` 事件。

取消场景另有 `_persist_cancelled_assistant_message`（agent_loop.py:1486）：通过扫描同会话全部 assistant 消息的 turn_id/user_message_id/client_turn_id 做**幂等**去重。

## 7. 上下文、模型与记忆

- **模型解析** `_get_request_model`：显式 `model_config_id` 校验租户归属与启用状态（失败抛 `AgentLoopPreconditionError`），否则 `model_for_agent` 取 agent/租户默认；均经 `resolve_model_config_for_runtime`。
- **每轮动作上限** `_get_agent_loop_max_actions`：agent 级 `harness_max_actions` 优先，否则 `UIConfig.agent_loop_max_actions`，默认 32，硬上限 100（`MAX_TOOL_ACTIONS_PER_TURN` / `_LIMIT`）。
- **上下文** `_conversation_context`：取全部消息 → `visible_message_rows` 过滤 → `build_conversation_context`（预算/压缩触发比/近轮数等由 `UIConfig` 驱动，默认 32k token、0.70 触发、近 6 轮），并把压缩状态写回 `context_state_json`。压缩通过 LLM 生成中文事实摘要（`_context_summary_builder`，span 名 `context.compact`）。
- **记忆**：`MemoryService.context_memories` 供引擎读取；`_enqueue_memory_capture` 把异步记忆采集任务入队，失败仅记 `memory_error`，不阻断主流程。
- **人格提示词** `_get_persona_prompt`：非总体 agent 用 `AgentIdentityPrompt.render`；总体 agent 优先自身 `persona_prompt`；最后回退租户级 `PersonaConfig`。

## 8. 流式事件机制（`_stream_event`，agent_loop.py:465）

- 统一外层信封 `{event, data:{kind, sessionId, timestamp, provider:"skill", ...payload}}`；
- 白名单事件（`stream_delta`/`stream_end`/`tool_result`/`step_result`/`knowledge_result` 等）且带 turn 标识时**持久化到 EventLog 并立即 commit**；其余只下发不入库。
- `_stream_status` 在非 received 阶段也会持久化状态事件。

## 9. 值得注意的问题与风险

1. **类体量与间接层**：1719 行、约 60 个方法，其中相当一部分是对 `GraphRules` / `HumanHandoffService` / `ConversationProjection` 的一行透传（如 `_edge_condition`、`_skill_slot_satisfied`、`_step_actions`、`_human_handoff_*`），增加了跳转成本；这些回调多为适配 `TurnFinalizer` 的注入接口而保留。
2. **死代码**（全 backend 无调用方）：
   - `_pace_stream`（agent_loop.py:499）与常量 `STREAM_CHUNK_INTERVAL_SECONDS`；
   - 模块级 `_knowledge_scope_ids`（agent_loop.py:89）；
   - `_message_context_entry`（agent_loop.py:1452）。
3. **伪流式的代价**：`handle_turn_stream` 内部同步跑完整个 turn 才开始回放 token，首字延迟等于整轮耗时；且每个 `stream_delta` 都触发一次 `db.commit()`（agent_loop.py:422），长回复会产生高频提交。
4. **控制流分叉不对称**：前置错误分支在 `handle_turn` 内直接 `return self._finish_with_error(...)`，而 LLM/未知异常走方法尾部的统一 `_finalize_turn`，两条路径的元数据/可见性处理存在差异，维护时需分别验证。
5. **`slots_json` 复用**：图调度内部状态 `_graph_pending_steps` 与业务槽位混存，存在键名碰撞风险，也使槽位数据需要感知内部保留键。
6. **防御性 `hasattr(self.db, "get"/"exec")`**（agent_loop.py:1221、1239、1399）：为测试假对象兜底的分支散落在生产代码中，建议以测试夹具或 Protocol 约束替代。
7. **流式中会话消失静默返回**：`_handle_turn_stream_v2` 在 `db.get` 取不到会话时直接 `return`（agent_loop.py:347），不下发错误事件，客户端可能只看到 planning 后流中断。
8. **重复实例化服务**：`_human_handoff_assignee_user_id` / `_human_handoff_tenant_admin_user_id` / `_human_handoff_context_summary` 各自 new 一个 `HumanHandoffService`，可在一轮中产生多个实例（无状态，影响小）。
9. **取消幂等扫描 O(n)**：`_persist_cancelled_assistant_message` 每次取消都加载会话全部 assistant 消息做去重，超长会话下有轻微性能压力。

## 10. 关键常量

| 常量 | 值 | 含义 |
|---|---|---|
| `STREAM_CHUNK_INTERVAL_SECONDS` | 0.045 | 流块节流（当前仅被死代码 `_pace_stream` 引用） |
| `MAX_TOOL_ACTIONS_PER_TURN` | 32 | 每轮工具动作默认上限 |
| `MAX_TOOL_ACTIONS_PER_TURN_LIMIT` | 100 | 每轮工具动作硬上限 |
| `GRAPH_PENDING_STEPS_SLOT` | `_graph_pending_steps` | 图挂起分支在 slots_json 中的保留键 |
| `CANCELLED_ASSISTANT_REPLY` | `已停止生成` | 取消时落库/返回的固定文案 |

## 11. 主要协作模块

| 模块 | 职责 |
|---|---|
| `harness_v2_engine.HarnessV2Engine` | turn 主流水线：会话锁/租约、turn 幂等领取与重放、规划、TaskFrame 调度、隔离执行 |
| `harness_session_lock` / `harness_turn_store` | 会话级互斥锁、turn 记录与 client_turn_id 幂等 |
| `graph_rules.GraphRules` | SOP 图的全部纯规则：节点/边、终态、动作、槽位满足、handoff BFS |
| `turn_finalizer.TurnFinalizer` | 轮末收尾：handoff、技能完成判定与执行 |
| `human_handoff_service.HumanHandoffService` | 转接单创建、处理人解析、上下文/待问摘要 |
| `channels/service_outbox` | 渠道投递与转接私聊通知（发件箱模式） |
| `conversation_context` / `conversation_projection` | 历史上下文构建与压缩、消息元数据/引用投影 |
| `response_generator.ResponseGenerator` | 失败文案、模型失败建议、回复切块 |
| `skill_runtime.SkillRuntime` | 技能完成与技能栈管理 |
| `memory` | 上下文记忆读取与异步采集入队 |
| `observability.EventLog` / `spans.llm_operation` | 事件落库/外发、LLM 可观测 span |
