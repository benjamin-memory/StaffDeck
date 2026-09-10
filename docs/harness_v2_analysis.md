# Harness v2 执行引擎分析

> 分析对象：`backend/app/core/harness_v2_engine.py`（2156 行）及其协作模块
> 分析日期：2026-09-10
> 相关文档：[agent_loop.py 源码分析](agent_loop_analysis.md)

## 1. 一句话定义

**Harness v2 是一套"持久化任务调度 + 隔离执行"的对话执行引擎**，类注释原文为 "Outer planner + durable TaskFrame scheduler + isolated Harness runs"（harness_v2_engine.py:182）。

它取代了旧版 `AgentLoop` 里"一条消息进来 → 一个 agent 在一个内存循环里边想边做"的模式，把每轮对话拆成 **先规划（plan）→ 持久化成任务帧（TaskFrame）→ 逐帧调度执行** 的流水线，并通过租约 / 幂等回执 / 检查点保证并发安全与崩溃可恢复。

所有对外事件与消息元数据统一打标 `execution_engine: "harness_v2"`。

## 2. 设计动机：相对旧模式解决什么问题

- **多任务**：一轮消息可能拆成多个任务（普通问答 + 多个 SOP），需要依赖排序、分别执行、汇总回复；
- **长驻可恢复**：SOP 可能跨多轮（等用户补充、等外部业务系统回调），执行状态必须落库而非留在内存；
- **并发与重试安全**：同会话并发消息、客户端超时重发、worker 崩溃，都不能重复执行或跑乱状态；
- **隔离性**：每个任务只拿到自己需要的能力清单与上下文，外层对话历史不污染任务模型。

## 3. 核心概念

### 3.1 四张持久化表（`app/db/models.py`）

| 概念 | 表 / 模型 | 含义 |
|---|---|---|
| **Turn** | `harness_turns` / `HarnessTurnRecord`（models.py:1313） | 一次客户端可寻址请求的**幂等回执**。按 `(tenant_id, session_id, client_turn_id)` 唯一，并存 `request_digest` 与最终 `response_json`。重复请求直接重放响应，不重新执行 |
| **TaskFrame** | `harness_task_frames` / `HarnessTaskFrameRecord`（models.py:1246） | 规划产出的**任务帧**，最小调度单元。字段含 `kind`、`decision`、`status`、`skill_id/step_id`、`slots_json`、`depends_on_json`、`task_requirement_json`、`result_json`、`attempt_no` + `lease_owner/lease_expires_at` |
| **Run** | `harness_runs` / `HarnessRunRecord`（models.py:1285） | 一个 TaskFrame 的**一次执行尝试**（attempt），存任务需求快照、当轮能力清单快照（`capability_snapshot_json`）、动作计数与结果。重试 = 新 Run，旧 Run 废弃 |
| **AgentLoop** | `harness_agent_loops` / `HarnessAgentLoopRecord`（models.py:1219） | 跨多次激活共享的**逻辑内层循环**，以 `(session_id, loop_key)` 唯一。`checkpoint_json` 保存内层 transcript、引用、artifacts 等，使任务暂停后下一轮能接着跑而非重头来 |

### 3.2 两道并发屏障

- **Session Lock**（`harness_session_lock.acquire_harness_session`）：进程内会话互斥锁；
- **Session Lease**（`harness_session_leases` / `HarnessSessionLeaseStore`）：跨进程的 DB 租约栅栏，同一时刻只允许一个 worker 执行某会话；执行中周期性 `renew`，丢失即抛 `HarnessSessionLeaseLost` → `HarnessExecutionFenced`。

### 3.3 规划决策（RouterDecisionValue，session_schema.py:10）

共 9 种：

- `continue_active`：继续当前 SOP；
- `switch_to_pending`：恢复一个挂起任务；
- `start_new_task`：启动新 SOP；
- `create_pending` / `update_pending`：创建 / 更新挂起任务；
- `complete_task`：结束任务；
- `answer_only`：普通问答（不进 SOP）；
- `clarify`：向用户澄清；
- `handoff_human`：转人工。

### 3.4 TaskFrame 类型与状态机

- 两种 `kind`（session_schema.py:21）：`sop`（走 SOP 图节点）、`conversation`（普通问答/澄清/handoff）；
- 两种 `execution_target`：`self`（本会话执行）、`team_member`（派发给团队成员会话）；
- 状态（session_schema.py:23）：

```
queued → running → completed / handoff / failed / cancelled
                  → awaiting_user          （等用户补充槽位）
                  → waiting_external_task  （等外部业务系统）
                        → ready_to_resume  （外部回调后可恢复）
                  → blocked                （前置依赖未完成）
```

动作预算耗尽或进程崩溃时，帧会被重新置回 `queued`，等下一条用户消息继续。

## 4. 完整执行流程

入口：`AgentLoop.handle_turn` → `HarnessV2Engine(self).run(request)`（harness_v2_engine.py:208）。

### 阶段 0：加锁与会话恢复

1. 首轮无 session_id 时，用 `sha256(tenant_id + user_id + client_turn_id)` 派生**确定性会话 ID**（`_with_recoverable_first_session`，:1990），让客户端重试可落到同一会话；
2. 获取进程内会话锁 → `get_or_create_harness_session` 创建/恢复会话（捕获两 worker 并发建行的 `IntegrityError`，:2020），并做 tenant/user/agent 归属校验，不匹配抛 `HarnessExecutionFenced`；
3. 获取 DB 会话租约。

### 阶段 1：幂等领取（:221）

`turn_store.claim(session, request)`：

- 相同 `client_turn_id` + 请求摘要已完成 → 直接返回 `turn_claim.replay`（重放 `response_json`）；
- 锁冲突 / 租约冲突 → 抛 `HarnessTurnConflict` / `HarnessSessionBusy`，由 `AgentLoop.handle_turn` 翻译成"并发或重复请求已阻止"。

### 阶段 2：入账与上下文准备（:225–320）

- 会话置 running；落 user 消息，绑定到 turn 记录（`bind_user_message` + `events.bind_turn`）；
- 解析斜杠指令 / 定时任务强制 SOP；定时任务可携带**不可变 SOP 快照**（`_apply_forced_sop_snapshot`，:104），快照不会复活已下线的 SOP；
- `context_injection`（服务端注入上下文）拼进执行消息后从请求剥离，不进入 Planner；
- 解析模型配置（无默认模型直接报错）；加载可见 SOP 并 `expand_visible_sops` 展开嵌套；`_drop_unavailable_skill_state` 自愈脏技能状态；
- 读取长期记忆（`memory.context_memories`，记 `memory_recalled`）；
- `_conversation_context` 构建历史上下文（可能触发 LLM 压缩）；
- 各阶段之间穿插 `_renew_session_lease()` 续约与 `_raise_if_cancelled()` 协作式取消检查。

### 阶段 3：规划（:332–432）

- 斜杠指令走 `build_slash_turn_plan`；否则 `TurnPlanner.plan()` 让 LLM 产出唯一的 `TurnPlan`：
  - 本轮总体 `decision`、置信度、意图；
  - 若干 `PlannedTaskFrame`：kind、目标 SOP/步骤、`requirements`、`slot_hints`、`depends_on_task_ids`、执行目标；
  - 对存量任务的 `task_updates`；
- 规划结果经 pydantic 校验与 `_normalize` 归一化（修正 LLM 决策与实际会话状态矛盾的情况）；
- **挂起任务优先规则**（:360）：存在 `ready_to_resume` 的 SOP 帧且规划不是 complete/handoff 时，强制改写 plan 为 `switch_to_pending`；
- `SlotHydrationPolicy.hydrate_plan` 用记忆自动水合槽位（记 `slots_hydrated`）；
- 落 `turn_plan_created` 与兼容旧 trace 的 `router_decision_created` 事件。

### 阶段 4：团队派发与计划落库（:433–499）

- `execution_target="team_member"` 的帧仅在 `interaction_mode == "team_tl"` 且有可信 team 时允许，经 `publish_team_planner_frames` 持久化派发到成员会话；派发失败直接报错（"已停止本轮虚假分发"），随后从本轮本地执行列表剔除；
- `complete_task` 决策在此关闭活动帧并可顺带完成当前 SOP；
- `persist_plan` 把帧 upsert 为 `HarnessTaskFrameRecord` 并应用 task_updates；再捞出因依赖满足而就绪的帧；
- `_dependency_order`（:2131）按依赖做拓扑排序（成环时剩余帧按原序追加）。

### 阶段 5：逐帧调度（:514–641）

受**动作预算**约束（agent 的 `harness_max_actions` 优先，否则租户 `UIConfig.agent_loop_max_actions`，默认 32、硬上限 100）：

- 预算耗尽 → 剩余帧 `defer_for_action_budget` 重新排队，记 `turn_action_budget_exhausted`，本轮结束；
- 依赖未满足 → `defer_for_dependencies` 置 blocked，产出"前置任务完成后将自动继续该任务"片段；
- 就绪帧 → `_activate_frame` 恢复 SOP 运行时状态（`restore_task_frame`，并按 decision 补记 `skill_started` / `skill_resumed` 事件供管理端调用统计）→ 进入 `_run_frame`；
- SOP 帧执行前发现技能已下线 → 帧直接 failed（`SOP_NOT_AVAILABLE`）；
- 某帧完成后调用 `ready_dependency_frames` 动态释放被它阻塞的帧，追加到本轮队列。

### 阶段 6：帧内执行 `_run_frame`（:815）

旧 ReAct 循环现在居住于此，每次只处理一个 SOP 步骤或一个会话任务：

1. `mark_running` + 确保 AgentLoop 记录，读取 checkpoint；
2. 物化附件（`materialize_task_attachments`）、图片载荷、已发布交付物；SOP 帧解析步骤超时（`step_timeout_seconds`，1–3600 秒封顶）；
3. 循环（受剩余动作预算约束）：
   1. 构建本步骤能力清单 `CapabilityManifest`：**完整清单留服务端做鉴权**，`project_capability_manifest` 投影出安全版本给模型；
   2. `TaskRequestCompiler.compile` 生成 `TaskRequirement`，输入包含：依赖帧结果、同会话引用帧结果、本帧此前各步结果、附件、已发布交付物、**当前用户最新消息**（避免长驻帧恢复时老消息顶掉新回复）、兄弟帧意图（`out_of_scope_task_intents` 防越界）、客户端时区；
   3. 持久化 requirement；首次创建 `HarnessRunRecord`（含能力快照），后续步更新 run context；
   4. `HarnessTaskAgent.run`（harness_agent.py:54）执行**隔离的内层 ReAct 循环**：模型只允许输出两种 action —— `tool` 或 `finish`（harness_agent.py:42）；工具调用统一经 `HarnessCapabilityInvoker`（鉴权、租约续约、取消检查、trace 落库）；
   5. 产出 `TaskExecutionResult`；
4. 结果后处理：
   - 等外部任务时写 `waiting_external_task` 并记录恢复节点（:1012）；
   - `_enforce_required_required_slots`：声明完成但必填槽位缺失 → 强制改 `awaiting_user`；
   - 无显式下一节点时按图的默认出边补 `next_step_id`；
   - 调 `AgentLoop._apply_step_result` 推进 SOP 图（槽位合并、防幻觉跳步、挂起分支调度）；
   - 调 `_finalize_execution_after_reply`（`TurnFinalizer`）判定 continued/completed/handoff；当前节点未声明 handoff 却想转人工 → 降级 failed（`HANDOFF_NOT_ALLOWED`）；
   - 预算耗尽 / 技能结束 / 步骤未推进时结束本帧，否则带着新 step 继续循环；
5. `_combine_results`（:1711）合并多步结果：**只保留终态步骤的回复**，中间过渡回复不拼接（避免重复），槽位/引用/证据/产物聚合；
6. 外部任务桥接（:1169）：若等待期间外部任务已终态，SOP 帧转 `ready_to_resume` 并合并外部结果；
7. `finish_run` / `finish_frame` / 保存 AgentLoop checkpoint，落 `task_frame_finished`。

### 阶段 7：会话投影与回复合成（:643–770）

1. `_restore_visible_active_frame`（:1490）决定轮末会话"停在哪"，优先级：
   1. handoff 帧（会话保持 handoff 状态）；
   2. 最新 awaiting 的 conversation 帧（清空 SOP 状态，挂起会话帧）；
   3. 本轮前原 SOP 的未完成帧（技能优先、sequence 次之）；
   4. 无候选则清空 SOP 状态；
2. `project_session` 回写会话投影；引用全局重新编号（`_globalize_citations`，上限 8 条）；artifacts 聚合去重（上限 20 个）；注入 handoff 上下文（`_inject_handoff_context`，含人工回复）；
3. 回复合成三分支：
   - 单帧且回复有效 → `_single_task_reply` **直接用帧终态回复，省一次模型调用**（JSON 投影不完整时回退合成）；
   - 多帧 / 纯问答 → `ResponseGenerator.generate` 做一次综合；
   - 任一帧 handoff → 固定文案"已为你提交人工处理请求…"；
4. 终态竞争：`begin_completion` 先把 turn 置完成中（与取消竞争，只有赢家能写终态 assistant 消息）；`stream_sink.finish()` 判定流式投递是否成功；
5. `AgentLoop._finalize_turn` 落 assistant 消息（元数据含 `execution_engine`、`task_frame_ids`、团队进度、引用、产物、斜杠指令等），未走流式时登记渠道投递；
6. commit 后异步入队记忆采集（仅 visible 消息）；
7. `turn_store.complete` 存入最终响应作为可重放回执，返回 `ChatTurnResponse`。

## 5. 容错与可靠性机制

| 场景 | 处理 | 代码位置 |
|---|---|---|
| 客户端超时重发 | turn 幂等回执重放，不重复执行 | `HarnessTurnStore.claim` |
| 同会话并发 | 进程内锁 + DB 会话租约双重屏障 | `harness_session_lock` / `harness_session_lease` |
| 两 worker 并发建首会话 | 捕获 `IntegrityError`，败者复用胜者行 | :2020 |
| 进程崩溃 | `mark_interrupted`：run 标 failed、帧重新排队（queued）、turn 标 failed，下轮凭 checkpoint 续跑 | :1315 |
| 用户取消 | 散布在帧循环与内层 agent 的协作式检查点；`mark_cancelled` 收尾，落"已停止生成" | :1259 |
| 租约丢失（worker 卡死被接管） | 续约失败抛 `HarnessExecutionFenced`，旧执行者写入全部被围栏拒绝 | :800 |
| 模型输出非法 action 信封 | `HARNESS_ACTION_INVALID` 视为可恢复，帧保留 queued | :1628 |
| 已提交节点后续跑失败 | `_defer_failed_step_after_completed_checkpoint` 保留已给用户的结果，剩余步骤排队，回复追加"本轮执行到此暂停…回复任意消息即可继续" | :1637 |
| 步骤超时 | 每步 deadline 传入内层循环与 invoker | :848、:2118 |
| 外部业务任务回调 | `waiting_external_task` ↔ `ready_to_resume` 状态桥接，记录恢复节点与外部结果 | :1012、:1169 |
| SOP 中途下线 | 执行前检查，帧 failed（`SOP_NOT_AVAILABLE`） | :565 |
| 模型幻觉跳步 | `next_step_id` 不在图中则忽略并修复（AgentLoop 侧 `step_agent_result_repaired`） | agent_loop.py:894 |

## 6. 与 `AgentLoop` 的分工

| 层 | 职责 |
|---|---|
| `HarnessV2Engine` | 锁/租约、turn 幂等、规划编排、TaskFrame 持久化与调度、Run/checkpoint 生命周期、依赖排序、动作预算、回复合成编排 |
| `TurnPlanner` | LLM 规划：产出 TurnPlan（决策 + 帧 + 任务更新），并做归一化校验 |
| `TaskRequestCompiler` | 把 PlannedTaskFrame + 会话/槽位/能力清单/历史结果编译成内层 `TaskRequirement` |
| `HarnessTaskAgent` | 帧内隔离 ReAct 小循环（仅 tool/finish 两动作），读写 checkpoint |
| `HarnessCapabilityInvoker` | 工具/能力调用的鉴权、租约续约、取消、trace |
| `TaskFrameStore` / `HarnessTurnStore` / `HarnessSessionLeaseStore` | 三类持久化状态的领取、租约、状态迁移、依赖查询 |
| `AgentLoop`（owner） | 退化为"能力提供者 + 状态机回调库"：会话/模型/技能查询、SOP 图推进（`_apply_step_result`）、完成判定（`TurnFinalizer`）、handoff、消息落库收尾（`_finalize_turn`）；引擎通过 `self.owner.xxx` 反向调用 |

## 7. 关键事件流（EventLog）

一次典型 turn 可观察到的主要事件：

```
user_message_received
memory_recalled?
turn_plan_created / router_decision_created
slots_hydrated?
skill_started | skill_resumed?            （仅新任务/恢复任务）
task_frame_started
  general_skill_trace / tool_result*      （内层执行 trace，逐次 commit 供流式中转）
task_frame_finished
task_frame_dependency_waiting | dependencies_released?
turn_action_budget_exhausted?
assistant_message_created
session_state_changed
```

## 8. 小结

Harness v2 的本质是把"一次对话"建模成**可持久化、可调度、可恢复的任务执行系统**：

- **Turn** 解决 exactly-once；
- **TaskFrame** 解决多任务拆解、依赖与跨轮挂起；
- **Run + attempt/lease** 解决崩溃重试与并发围栏；
- **AgentLoop checkpoint** 解决内层执行连续性；
- **隔离的 HarnessTaskAgent** 解决上下文污染与能力最小化授权；
- 旧的 SOP 图状态机（GraphRules）、handoff、消息收尾仍复用 `AgentLoop` 中的成熟逻辑，新旧两层通过 owner 回调衔接。
