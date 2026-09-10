# HarnessV2Engine 代码走读与函数级参考

> 分析对象：`backend/app/core/harness_v2_engine.py`（2156 行）
> 分析日期：2026-09-10
> 视角：逐段代码走读、状态字段、事务/租约/事件/错误码清单、owner 回调耦合点
> 互补文档：[Harness v2 执行引擎分析](harness_v2_analysis.md)（架构与设计动机）、[agent_loop.py 源码分析](agent_loop_analysis.md)

---

## 1. 文件职责一句话

`HarnessV2Engine` 是一次对话 turn（`ChatTurnRequest → ChatTurnResponse`）的**总编排器**，类注释原文：

> "Outer planner + durable TaskFrame scheduler + isolated Harness runs"（harness_v2_engine.py:182）

它自己**不做** LLM 规划、不跑 ReAct、不推进 SOP 图，而是通过组合的组件与 `self.owner`（`AgentLoop` 实例）回调完成：

```
ChatTurnRequest
      │
      ▼
加锁/租约 → turn 幂等 claim → 入账 → 上下文准备
      │
      ▼
TurnPlanner（LLM 规划）或斜杠指令计划
      │
      ▼
TaskFrame 持久化 + 依赖排序 + 团队派发
      │
      ▼
逐帧调度 _run_frame（帧内隔离 ReAct）
      │
      ▼
会话投影恢复 → 引用/产物聚合 → 回复合成 → 落账
      │
      ▼
ChatTurnResponse（并存为可重放回执）
```

## 2. 依赖地图

### 2.1 引擎组合的组件（`__init__`，:184-206）

| 属性 | 类型 | 职责 |
|---|---|---|
| `planner` | `TurnPlanner` | LLM 外层规划，产出 `TurnPlan` |
| `compiler` | `TaskRequestCompiler` | 把帧 + 会话状态编译成帧内 `TaskRequirement` |
| `manifests` | `CapabilityManifestBuilder` | 构建某步骤可见的能力清单 |
| `task_agent` | `HarnessTaskAgent` | 帧内隔离 ReAct 小循环（仅 `tool` / `finish` 两种 action） |
| `store` | `TaskFrameStore` | TaskFrame / Run / AgentLoop 的持久化与状态迁移 |
| `turn_store` | `HarnessTurnStore` | turn 幂等 claim、续约、终结 |
| `session_leases` | `HarnessSessionLeaseStore` | 跨进程会话租约（fencing 栅栏） |

### 2.2 关键导入模块

| 模块 | 用途 |
|---|---|
| `harness_session_lock` | 进程内会话互斥锁（`acquire_harness_session` / `release_harness_session`） |
| `harness_session_lease` | DB 会话租约；`HarnessSessionLeaseLost` 丢租约异常、`HarnessSessionLeaseToken` 令牌 |
| `task_frame_store` | TaskFrame CRUD；`TaskFrameClaimConflict`（帧租约冲突）、`planned_frame_from_record`（记录→计划帧） |
| `task_request_compiler` | `TaskRequestCompiler`、`TaskExecutionResult`（帧执行结果统一结构） |
| `harness_capability_invoker` | 工具调用的鉴权 / 续约 / 取消 / trace 包装 |
| `harness_agent` | `HarnessTaskAgent`；`HarnessExecutionCancelled` / `HarnessExecutionFenced` |
| `slash_commands` | 斜杠指令解析与强制能力：`parse_slash_command`、`build_slash_turn_plan`、`resolve_capability`、`force_capability_for_requirement` 等 |
| `turn_planner` | `TurnPlanner.plan`、`turn_plan_router_decision`（plan → 对外路由决策投影） |
| `slot_hydration_policy` | 用记忆自动水合 plan 槽位 |
| `cancellation.is_chat_turn_cancelled` | 协作式取消标志查询（支持 message / client 两种身份） |
| `capability_discovery.project_capability_manifest` | 完整清单 → 给模型看的安全投影 |
| `harness_attachments` | 附件物化、图片载荷校验 |
| `published_deliverables` | 列出本会话已发布交付物（供后续帧引用） |
| `skills.nesting` | `expand_visible_sops` / `discoverable_sops`：嵌套 SOP 展开与可发现性 |
| `memory.service.memory_read` | 长期记忆读取 |
| `knowledge.citations.compact_knowledge_citation_labels` | 回复内引用标签压缩对齐 |
| `tools.external_tasks.update_external_task_checkpoint` | 外部业务任务回调检查点 |
| 懒加载 `app.teams.wakeup` | `build_team_planner_context` / `publish_team_planner_frames`（仅 team_tl 模式导入） |

## 3. 实例状态字段（:195-206）

| 字段 | 含义 | 清理点 |
|---|---|---|
| `turn_record` | 当前 turn 的幂等记录 | 终结后保留 |
| `session_lease` | 会话租约令牌 | `close()` 释放 |
| `user_message_id` / `current_source_turn_id` | 本轮用户消息 ID（取消判定、来源绑定） | — |
| `session` | 当前会话 ORM 对象 | — |
| `active_frame_id` | 正在执行的帧行 ID | 帧结束/取消/中断时清空 |
| `active_frame_lease_owner` / `active_frame_attempt_no` | 当前帧租约 owner 与尝试号（fencing 双因子） | 同上 |
| `active_run_id` | 当前 Run 行 ID | run 结束清空 |
| `slash_command` | 本轮解析出的斜杠选择（可能为 None） | — |
| `_session_lock` / `_session_lock_id` | 进程内会话锁及其键 | `close()` |

## 4. 模块级函数（按出现顺序）

### 4.1 SOP 投影与快照

- **`_turn_skill_projection(source_skills, *, interaction_mode)`** (:79)
  调 `expand_visible_sops` 展开嵌套 SOP，返回 `(可执行 skills, 可路由 skills=discoverable_sops)`。`interaction_mode` 目前被显式忽略（`_ = interaction_mode`）：团队 TL 会话是独立 ChatSession，无需为状态隔离隐藏 TL 自己的 SOP。

- **`_apply_forced_sop_snapshot(source_skills, forced_sop_id, snapshot)`** (:104)
  定时任务可携带创建时冻结的 SOP 快照。校验：快照 `skill_id` 必须等于强制目标，`content_json` 必须是 dict，否则抛 `SlashCommandError("FORCED_SOP_SNAPSHOT_INVALID")`。**安全底线**：当前技能列表里找不到该 SOP 时原样返回——历史快照永不复活已下线/解绑的 SOP（继续走常规能力访问报错）。命中则构造一个 pinned `Skill`（status 固定 `published`，并透传 `agent_branch_meta`）。

- **`_turn_slash_selection(request)`** (:156)
  统一解析两条"强制路由"来源：
  1. 用户文本里的斜杠指令（`parse_slash_command`）；
  2. 服务端钉住的 `forced_sop_id`（定时任务）——合成一个 `kind="sop"` 的选择。
  互斥规则：两者同时存在 → `FORCED_SOP_COMMAND_CONFLICT`；定时任务模式（`interaction_mode == "scheduled_task"`）文本里带斜杠指令 → `SLASH_COMMAND_MODE_CONFLICT`（必须走结构化 SOP 选择）。

### 4.2 结果/载荷加工

- **`_step_result(result)`** (:1607)：`TaskExecutionResult` → 对外 `StepAgentResult`；状态到 action 的映射：`completed→advance`、`awaiting_user→ask_user`、`handoff→handoff`，其余（failed/blocked/action_budget/未知）→ `reply`。

- **`_is_recoverable_action_protocol_failure(result)`** (:1628)：仅当 failed 且错误码为 `HARNESS_ACTION_INVALID`（模型 action 信封非法）时视为可恢复——帧保留可续跑。

- **`_defer_failed_step_after_completed_checkpoint(result, completed_results)`** (:1637)：SOP 帧一轮可连走多个节点；若某节点已完成（过渡已持久化、且已产出回复），启动下一节点时模型/协议失败，**不得抹掉已给用户的结果**。把失败改写为 `action_budget`（帧继续排队），并在回复后固定追加"（本轮执行到此暂停，剩余步骤已排队；回复任意消息即可继续。）"——注释里记录了真实事故：用户看到"将进入正式提交步骤"误以为会自动提交。

- **`_enforce_required_slots(result, requirement, session)`** (:1686)：模型声称 completed 但必填槽位（合并 session 槽位与本轮更新后）仍为空/空容器 → 强制降级 `awaiting_user`，无回复时生成"还需要您补充：X、Y。"

- **`_combine_results(task_frame_id, results)`** (:1711)：合并帧内多步结果。**只取最后一个非空回复作为终态回复**（中间节点的过渡回复不拼接，避免重复）；槽位更新、引用、证据、能力结果、artifacts 全部展平合并；`task_summary` 去重用"；"连接；action_count 求和；空结果列表产生 `EMPTY_TASK_RESULT` 失败。

- **`_single_task_reply(results)`** (:1779)：**省一次模型调用**的优化——本轮恰好一个帧且其终态回复有效时直接复用；`_structured_reply_requires_synthesis` (:1797) 检测回复看起来是 JSON 但与 structured_result 不一致（半截投影）时回退到 ResponseGenerator 合成。

- **`_response_task_payload(row, result, skill, step_result)`** (:1815)：构造传给 ResponseGenerator 的单帧 payload（任务文案、状态、SOP content、当前步、槽位、step_result、工具成功聚合、summary、structured_result、artifacts）。

- **`_inject_handoff_context(db, session, payloads, results, request=None)`** (:1849)：向 payload 注入 `handoff_info`。resume turn 判定**不依赖时间戳标记**（旧标记由 worker 写入、时序晚于 turn 执行导致永远 miss），改用显式 `request.channel == "human_handoff_resume"` + handoff 已 answered；本轮新触发 handoff 的帧也注入（无 human_reply，用于告知用户已转交）。

- **`_globalize_citations(results)`** (:1895)：跨帧引用全局重编号，同一身份（`concept_id` / `chunk_id` 优先，否则 source/section/title/excerpt 拼接）共用一个 `[n]` 标签，**上限 8 条**；同时原地改写每个 result 的标签。

- **`_citation_identity(citation)`** (:1918)：引用去重身份键，兜底指纹截断 2000 字符。

- **`_aggregate_artifacts(results)`** (:1931) / **`_merge_discovered_artifacts(...)`** (:1951)：产物按 `(type, task_frame_id, path, handoff_id)` 去重，**上限 20 个**；后者把 invoker 运行时发现（如下载的文件）并入当前结果，按 path 去重。

- **`_append_session_handoff_artifact(result, session)`** (:1970)：从会话 `awaiting_input_json.handoff_id` 补一个 `human_handoff` 产物。

### 4.3 会话与杂项

- **`_with_recoverable_first_session(request)`** (:1990)：首轮无 session_id 但带 client_turn_id 时，用 `sha256(tenant_id ␟ user_id ␟ client_turn_id)` 前 16 位派生确定性 ID `session_xxxxxxxx`，使客户端超时重发落到同一会话。
- **`get_or_create_harness_session(owner, request)`** (:2010)：建/取会话，**捕获两 worker 并发建行的 `IntegrityError`**（败者 rollback 后复用胜者行）；随后做 tenant / user / agent 三重归属校验，不匹配抛 `HarnessExecutionFenced`；历史会话无 agent_id 时补绑。
- **`_session_state(session)`** (:2055) / **`_restore_session_state(...)`** (:2067)：轮前快照 / 恢复会话七项投影字段（深拷贝，防 ORM 引用污染）。
- **`_prior_result(result)`** (:2079)：帧内步间上下文的精简投影。
- **`_source_user_message(db, row)`** (:2091)：兜底取帧来源用户消息文本。
- **`_sibling_task_intents(db, row)`** (:2098)：同一来源 turn 内**其他兄弟帧**的意图列表——作为 `out_of_scope_task_intents` 喂给编译器，防止帧间互相越界抢活。
- **`_skill_step_timeout_seconds(skill)`** (:2118)：从 SOP content 读 `step_timeout_seconds`，容错解析后夹在 **[1, 3600]** 秒。
- **`_dependency_order(records)`** (:2131)：按 `depends_on_json` 做拓扑排序；**成环/缺依赖**时不报错，剩余帧按原序直接追加（依赖不在本批中视为已满足）。

---

## 5. `run()` 主流程逐段走读（:208-770）

### 5.1 阶段 0｜锁与租约（:209-224）

1. `_with_recoverable_first_session` 派生确定性首轮 ID；
2. 有 session_id 先抢**进程内会话锁**；
3. `_get_or_create_session`（:782，委托模块函数）建/取会话后，**再用权威 session.id 抢一次锁**（覆盖首轮刚派生 ID 的情况）；
4. 获取 **DB 会话租约** `session_leases.acquire`；
5. `turn_store.claim` 做**幂等领取**：命中已完成同 turn → 直接 `return turn_claim.replay`（:223，重放响应，无任何副作用）。

### 5.2 阶段 1｜入账（:225-260）

- `_mark_session_running`；
- 落 user 消息（metadata 由 owner 构造），记录 `user_message_id` 并 `bind_user_message` 到 turn；
- `events.bind_turn(message_id, client_turn_id)`（若事件总线支持）；
- 记录 `user_message_received`（携带 channel/user_id/`execution_engine="harness_v2"`，非 visible 时带 `message_visibility`）。

### 5.3 阶段 2｜执行消息与上下文（:262-320）

- 解析斜杠/强制 SOP；执行消息 = 斜杠改写后的 prompt（`slash_command_message`）或原文；
- **`context_injection` 只拼进执行消息**（rstrip 后两换行拼接），随后从 execution_request 剥离——服务端注入上下文不进入 Planner 的任务描述（`_turn_planner_message` :96 同样保证只传原始 message）；
- 取模型配置，无默认模型直接 `RuntimeError("没有默认模型配置。")`；
- 加载已发布 SOP → 应用定时快照 → `_turn_skill_projection` 展开 → `_drop_unavailable_skill_state` 自愈失效技能的脏会话状态；
- 读长期记忆并记 `memory_recalled`；
- `commit + refresh` 后第一次取消检查 `_raise_if_cancelled`；
- 构建对话历史上下文（内部可能 LLM 压缩）。

### 5.4 阶段 3｜规划（:322-432）

- 续约；取 planner_state（存量挂起帧视图）；
- team_tl 模式懒加载 `build_team_planner_context`；
- **两条规划路径**：
  - 斜杠 skill/tool：先构建无技能上下文的 direct manifest 做 `resolve_capability` 校验，再 `build_slash_turn_plan`；
  - 否则 `planner.plan(...)`（传入：原始消息、会话、可路由 SOP、模型配置、对话上下文深拷贝、记忆、planner_state、interaction_mode、team_context）。
- **挂起帧优先接管规则**（:360-390）：planner_state 中存在 `ready_to_resume` 的 SOP 帧、且本轮决策不是 complete/handoff 时，直接把 plan **改写**为 `switch_to_pending`（从 DB 反查记录重建 planned_frame）——外部回调就绪的任务优先于新规划。
- 槽位记忆水合（`SlotHydrationPolicy.hydrate_plan`，记 `slots_hydrated`，source=memory）；
- 双事件留痕：`turn_plan_created`（新）+ `router_decision_created`（旧公共 trace，保持兼容）。

### 5.5 阶段 4｜团队派发与计划落库（:433-499）

- 筛出 `execution_target == "team_member"` 的帧：**必须** team_tl 模式且 team 存在，否则 `RuntimeError("团队成员 TaskFrame 缺少可信团队上下文。")`；
- `publish_team_planner_frames` 持久化派发到成员会话；返回 None → `RuntimeError("...已停止本轮虚假分发。")`（明确防止"嘴上派发、实际没存"）；派发成功后这些帧从本轮本地执行列表剔除；
- `complete_task`：关闭活动帧（`complete_active_frame`），必要时 `runtime.complete_current_skill`，记 `task_frame_completed`；
- 存轮前会话快照 `pre_turn_state = _session_state(session)`（:487，供轮末恢复）；
- `persist_plan` upsert 帧、应用 task_updates → 追加依赖刚满足的就绪帧 → `_dependency_order` 拓扑排序 → `commit`。

### 5.6 阶段 5｜逐帧调度循环（:501-641）

- 动作预算 `remaining_turn_actions`：owner 查询 agent 的 `harness_max_actions`，夹到 ≥0；
- 对每条记录：
  1. 续约；预算 ≤ 0：剩余帧整体 `defer_for_action_budget`，记 `turn_action_budget_exhausted`，commit 后跳出；
  2. 取消检查；
  3. 依赖未满足：`defer_for_dependencies` + blocked 结果（"前置任务完成后将自动继续该任务。"，错误码 `DEPENDENCY_WAITING`）+ `task_frame_dependency_waiting`，**continue**；
  4. `planned_frame_from_record` 转计划帧 → `_activate_frame` 恢复 SOP 运行时；
  5. SOP 帧但技能不可用：直接 `finish_frame(failed)`，错误码 `SOP_NOT_AVAILABLE`，continue；
  6. `_run_frame` 执行（先序结果 = 依赖帧结果 + 同会话引用帧结果，:600-603）；扣减动作预算（至少扣 1）；
  7. 帧 completed 后动态捞取被它释放的依赖帧，追加进本轮队列并记 `task_frame_dependencies_released`。

### 5.7 阶段 6｜轮末投影与回复（:643-770）

1. `_restore_visible_active_frame`（见 §8）+ `project_session` 回写会话投影，commit；
2. 取消检查；跨帧引用全局编号（上限 8）并回填 payload；注入 handoff 上下文；
3. 回复四分支（:669-707）：
   - 团队派发且本轮无本地执行帧 → 固定"已完成 N 个团队任务的拆分与派发。"；
   - 任一帧 handoff → 固定"已为你提交人工处理请求，请稍候…"，并走流式分片；
   - `_single_task_reply` 命中 → 直接用帧终态回复（省一次模型）；
   - 否则 ResponseGenerator 流式/非流式综合生成；
4. `compact_knowledge_citation_labels` 对齐回复正文与引用标签；聚合 artifacts（上限 20）；
5. 组装 assistant metadata：`execution_engine`、`task_frame_ids`、团队进度（phase=collecting）、client_turn_id、可见性、引用、artifacts、斜杠指令；
6. **终态竞争**（:738）：写终态 assistant 消息前最后一次取消检查——取消与正常投影竞争同一份持久化回执，只有赢家能落终态消息；
7. `stream_sink.finish()` 得到投递是否成功；`turn_store.begin_completion` 标记完成中；
8. owner `_finalize_turn` 落 assistant 消息 → commit；
9. visible 消息才异步入队记忆采集 `_enqueue_memory_capture`；
10. 构造 `ChatTurnResponse`；`turn_store.complete` 存响应为重放回执，返回。

### 5.8 `close()`（:772-780）

先释放 DB 会话租约（finally 兜底），再释放进程内锁并清空键。注意调用方需在 finally 中调用以避免锁泄漏。

---

## 6. `_run_frame()` 帧内执行走读（:815-1257）

签名输入：请求、会话、帧记录、计划帧、active_skill、模型配置、记忆、先序帧结果、本帧最大动作数；输出 `(合并后的 TaskExecutionResult, 最后一步 StepAgentResult)`。

### 6.1 初始化（:827-873）

- `mark_running(row)` + `ensure_agent_loop(row)`（取跨轮 checkpoint）；
- 记录当前帧租约双因子到实例（供取消/中断路径 fencing）；
- 物化附件（绑定 tenant/session/task_frame/user）、校验图片载荷、列出已发布交付物（排除当前帧）；
- SOP 帧才解析步骤超时；
- 记 `task_frame_started`（含 skill、step、超时、预算、agent_loop 信息）。

### 6.2 步骤循环（:875-1166，受 `remaining_actions > 0` 约束）

每轮迭代：

1. **取消检查**；SOP 帧计算本步单调时钟 deadline `step_deadline_monotonic`；
2. 同步 `frame.target_step_id`（取 row.step_id 或会话当前步）；
3. 构建能力清单：**完整 manifest 留服务端做鉴权**，`project_capability_manifest` 投影安全版给模型（:889-892 注释明确：`capability_describe` 可在后续激活 schema）；
4. `compiler.compile(...)` 生成 `TaskRequirement`，输入包括：
   - 先序帧结果 + **本帧此前各步结果**；
   - 附件描述符、已发布交付物；
   - **`source_user_message` 强制用当前用户最新消息**（:905-911 注释记录真实事故：长驻 SOP 帧恢复时若用建帧老消息，会顶掉用户最新的「提交」指令，导致 agent 重复提问；原始意图保留在 user_intent/slots 中不丢）；
   - `out_of_scope_task_intents` = 兄弟帧意图；客户端时区；
5. 斜杠 skill/tool 且当前是本 turn 新建 conversation 帧时，`resolve_capability` + `force_capability_for_requirement` 强制本次执行该能力；
6. `save_requirement`（带帧租约双因子）；首步 `start_run`（requirement + capability_snapshot 双快照落库）并把 agent_loop checkpoint 置 active 关联 run；后续步 `update_run_context`；commit；
7. 内部 `trace(event_type, payload)` 闭包（:956-972）：所有内层事件自动带上 `task_frame_id` / `harness_run_id` / `agent_loop_id` / engine 标记，**每次 trace 都 commit**——使流式中转能实时看到运行中的帧，而非轮末才暴露；
8. 构建 `HarnessCapabilityInvoker`：注入取消回调、租约保证回调（`ensure_execution_lease`）、trace、步骤 deadline、初始激活能力名白名单；
9. `task_agent.run(...)` 跑隔离 ReAct（传入图片、双超时参数、checkpoint），返回单步 `TaskExecutionResult`。

### 6.3 步结果后处理（:1008-1166）

- **handoff resume 通道**（:1008）：`channel == "human_handoff_resume"` 且结果仍是 handoff → 改写 completed（人工回复本身就是终态信号，禁止 resume turn 再进一次终态 handoff 节点）；
- **等外部任务**（:1012-1037）：查进行中的 `ExternalBusinessTask`，写入 `resume_step_id`（取默认下一节点），供回调后从该节点恢复；
- SOP 帧的"已提交检查点后续步失败"延迟处理（:1039-1055，见 §4.2），命中则保存 checkpoint 后**跳出**，保留排队；
- 保存 checkpoint（合并发现的 artifacts，上限 20）；扣减动作预算；
- **conversation 帧**（:1076-1097）：handoff 决策/结果 → owner 建人工工单并追加 `human_handoff` 产物；否则普通结束，break（conversation 帧一轮只跑一次）；
- **SOP 帧**：
  1. `_enforce_required_slots` 必填槽位闸门；
  2. completed 但无显式 next_step → 用图的默认出边补；
  3. owner `_apply_step_result` 推进 SOP 图（槽位合并、防幻觉跳步、挂起分支调度在 owner 内部）；
  4. owner `_finalize_execution_after_reply` 给出终态判定：
     - `handoff` → 帧 handoff + 会话 handoff 产物；
     - 结果自称 handoff 但节点未声明该能力 → **降级 failed，`HANDOFF_NOT_ALLOWED`**；
     - `completed` → 帧结束；
     - 预算耗尽 / SOP 已退出 / 步骤没推进 → `action_budget` 结束；
     - 否则更新 `frame.target_step_id` 继续下一轮迭代。

### 6.4 收尾与外部任务桥接（:1168-1257）

- `_combine_results` 合并多步；
- 合并结果为 `waiting_external_task` 时复查外部任务是否**在执行期间已到达终态**（:1169-1205）：
  - SOP 帧 → `ready_to_resume`，外部任务 completed 时跳转到其 `resume_step_id`，结果并入 slots，写 structured_result 与检查点；
  - conversation 帧 → completed / failed；
- `finish_run`（带租约双因子）；
- 帧状态落定：`action_budget` 或可恢复协议失败 → 帧置回 **queued**（agent_loop 同样保留可续跑）；其余按 combined.status；
- `finish_frame` + `finish_agent_loop_for_frame`（checkpoint、last_run_id）；
- 记 `task_frame_finished`（含 action_count、error、agent_loop 状态）；commit；清空活动帧字段。

---

## 7. 并发、租约与取消体系

引擎里共有**四层**互不替代的防护：

| 层 | 机制 | 作用域 | 失败表现 |
|---|---|---|---|
| 1. 进程内会话锁 | `acquire_harness_session`（:212） | 单进程内同会话互斥 | 阻塞/冲突异常 |
| 2. DB 会话租约 | `HarnessSessionLeaseStore`，周期 renew（`_renew_session_lease` :788，每次还顺带 renew turn） | 跨 worker 同会话互斥（fencing） | `HarnessSessionLeaseLost` → `HarnessExecutionFenced` |
| 3. Turn 幂等 claim | `turn_store.claim`，按 client_turn_id + 请求摘要 | 重发/重放 | 直接重放 response_json |
| 4. TaskFrame 租约 | `lease_owner` + `attempt_no` 双因子，随每次写帧/写 run 校验 | 同帧并发 attempt（崩溃接管） | `TaskFrameClaimConflict` → `HarnessExecutionFenced`（`_renew_execution_leases` :793-813） |

**取消**（协作式，非抢占）：

- `_is_cancelled`（:1389）支持两个身份：本轮 message ID（identity_kind=message）或 client_turn_id（identity_kind=client）；
- 检查点散布在：主流程各阶段边界、帧循环顶部、`_run_frame` 步骤循环顶部，并以回调形式注入 invoker 与内层 agent；
- `mark_cancelled()`（:1259）：先 rollback 清理未提交状态；有来源 turn 时 `cancel_source_turn` 统一收尾，否则尽力取消活动 run/帧（**带租约双因子校验**，旧 owner 不能取消新 attempt）；turn 记 `CANCELLED`；
- `mark_interrupted(code, message)`（:1315）：崩溃路径——进行中的 run 标 failed，帧**重新排队 queued**（agent_loop 置 action_budget 保留 checkpoint），turn 标 failed；同样用租约双因子防旧执行者误改新 attempt。

## 8. `_restore_visible_active_frame` 轮末投影（:1490-1604）

决定会话轮末"停在哪个帧/什么状态"，优先级从上到下：

1. **会话 handoff**：选 handoff 帧设为活动帧，保持 handoff 状态；
2. **最新 awaiting 的 conversation 帧**：清空所有 SOP 投影字段，写 `awaiting_input_json`（kind=conversation、expected_fields 取自 requirement.required_slots、requirements、question_summary），设活动帧；
3. 若轮前活动 SOP 在本批 records 中**没有对应帧**（说明本轮没碰它）→ 整体恢复轮前快照 `_restore_session_state`，避免投影被无关任务串走；
4. 选未终态 SOP 帧候选：排序优先原技能（0/1），再按 `sequence`；无原技能帧时纯按 sequence；
5. 无 SOP 候选：取未终态 conversation 帧中 sequence 最小者，清空 SOP 状态后设活动帧；都没有则活动帧置空；
6. 选中 SOP 帧：`runtime.restore_task_frame` 恢复技能/步骤/slots；`awaiting_user` 状态额外构造 awaiting_input（expected_fields、question_summary），`set_active_task_frame` 落库。

`_activate_frame`（:1424）是调度侧的对称操作：仅 SOP 帧，按 skill_id 在当前可执行 skills 中查找；找不到返回 None（调用方据此判 SOP_NOT_AVAILABLE）；找到则恢复运行时并补记统计事件。

### 8.1 `_record_skill_activation_event`（:1451）

只有 `start_new_task → skill_started`、`switch_to_pending → skill_resumed` 落事件，`continue_active` 不计新调用。注释说明原因：管理端 SOP 调用次数统计（`api/skills._skill_stats`）只认这两个事件，legacy 运行时移除后事件断流导致统计恒为 0；payload 保持旧字段结构（to/from skill/step/version）。

---

## 9. 事务提交点与事件清单

### 9.1 `commit` 点（按执行顺序）

| 位置 | 时机 |
|---|---|
| `get_or_create_harness_session` :2019/:2050 | 会话建/绑 |
| :315 | 记忆读取后、规划前 |
| `_renew_session_lease` :791 | 每次续约（含 turn renew） |
| `_renew_execution_leases` :813 | 帧执行中租约续期 |
| :498 | plan 持久化 + 依赖帧捞出后 |
| :530 | 动作预算耗尽延期后 |
| `_run_frame` :954 | 首步 start_run / 后续 update_run_context 后 |
| trace 闭包 :972 | 每条内层 trace（实时流式可见性） |
| :1037 | 外部任务 resume_step 写入后 |
| :1253 | task_frame_finished 后 |
| :649 | 轮末投影后 |
| :751 | assistant 终态消息落库后 |
| `mark_cancelled` :1301 / `mark_interrupted` :1375 | 异常收尾 |

### 9.2 本文件发出的事件类型

| 事件 | 行号 | 备注 |
|---|---|---|
| `user_message_received` | :245 | 含 channel/可见性 |
| `memory_recalled` | :308 | 有记忆才有 |
| `slots_hydrated` | :402 | source=memory |
| `turn_plan_created` | :414 | plan 全量 dump |
| `router_decision_created` | :425 | 旧公共 trace 兼容 |
| `task_frame_completed` | :478 | complete_task 决策 |
| `turn_action_budget_exhausted` | :521 | 含被延期帧 ID |
| `task_frame_dependency_waiting` | :544 | 含 depends_on |
| `task_frame_dependencies_released` | :632 | 含释放帧 ID 列表 |
| `task_frame_started` | :855 | 含超时/预算/agent_loop |
| `harness_step_continuation_deferred` | :1047 | 检查点后续跑失败转排队 |
| `task_frame_finished` | :1237 | 含状态/动作数/错误 |
| `skill_started` / `skill_resumed` | :1467/:1469 | 仅新任务/恢复任务 |
| 内层 trace（透传） | :956 | ReAct/工具事件统一附加 run/frame/loop 标识并逐次 commit |

## 10. 错误码与异常一览

| 码 / 异常 | 触发点 | 含义 |
|---|---|---|
| `SlashCommandError: FORCED_SOP_SNAPSHOT_INVALID` | :116/:122 | 定时任务 SOP 快照 skill_id 不符或 content 非法 |
| `FORCED_SOP_COMMAND_CONFLICT` | :163 | 强制 SOP 与用户斜杠指令并存 |
| `SLASH_COMMAND_MODE_CONFLICT` | :174 | 定时任务文本里夹带斜杠指令 |
| `DEPENDENCY_WAITING` | :540 | 前置帧未完成 |
| `SOP_NOT_AVAILABLE` | :571 | 执行前 SOP 已下线/解绑 |
| `HANDOFF_NOT_ALLOWED` | :1147 | 当前 SOP 节点未声明转人工能力 |
| `EMPTY_TASK_RESULT` | :1720 | 帧未产出任何结果（防御性） |
| `HARNESS_ACTION_INVALID`（识别） | :1633 | 模型 action 信封非法 → 可恢复，帧保 queued |
| `CANCELLED` | :1307 | 用户协作式取消收尾 |
| `HarnessExecutionFenced` | :800、:2033-2046 | 租约丢失 / 帧 claim 冲突 / 会话 tenant-user-agent 不匹配 |
| `HarnessExecutionCancelled` | :1420 | 取消检查点抛出 |
| `RuntimeError("没有默认模型配置。")` | :276 | 模型配置缺失 |
| `RuntimeError("团队成员 TaskFrame 缺少可信团队上下文。")` | :439 | 远程帧脱离 team_tl 上下文 |
| `RuntimeError("...已停止本轮虚假分发。")` | :451 | 团队帧持久化失败 |

## 11. `self.owner`（AgentLoop）耦合点清单

引擎通过 owner 反向复用 legacy 能力，边界如下：

| 回调 | 用途 |
|---|---|
| `owner.db` / `owner.events` | DB 会话、事件总线 |
| `owner._mark_session_running` / `_get_or_create_session` | 会话生命周期 |
| `owner._append_message` / `_user_message_metadata` | 用户消息入账 |
| `owner._get_request_model` / `_get_persona_prompt` | 模型配置与人格 prompt |
| `owner._list_published_skills` / `_drop_unavailable_skill_state` / `_get_active_skill` | SOP 查询与会话脏状态自愈 |
| `owner.memory.context_memories` | 长期记忆 |
| `owner._conversation_context` | 历史上下文（含压缩） |
| `owner.runtime.restore_task_frame` / `complete_current_skill` | SOP 运行时状态恢复/完成 |
| `owner._get_agent_loop_max_actions` | 动作预算配置 |
| `owner._default_next_step` | SOP 图默认出边 |
| `owner._apply_step_result` / `_finalize_execution_after_reply` | **SOP 图推进 + 终态判定（核心状态机仍在 owner 侧）** |
| `owner._create_human_handoff_request` | 转人工工单 |
| `owner.response_generator.generate(_stream)` / `chunk_text` | 回复合成与流式分片 |
| `owner.stream_sink` / `stream_delivery_succeeded` | 流式投递 |
| `owner._finalize_turn` | assistant 消息落库收尾 |
| `owner._enqueue_memory_capture` | 异步记忆采集（仅 visible） |

## 12. 设计要点速查（面试/复习用）

1. **exactly-once**：Turn 幂等 claim + 响应重放；客户端重发零副作用。
2. **确定性首轮 ID**：`hash(tenant+user+client_turn)` 让"还没 session_id 的首轮重试"可复聚合同一会话，且用 IntegrityError 兜底并发建行。
3. **双快照**：Run 保存 requirement 快照与 capability_snapshot 快照——重试新 Run 不复用旧授权面。
4. **fencing 双因子**：帧/Run 每次写入带 lease_owner + attempt_no，崩溃后被接管的旧执行者写不进来。
5. **预算两级**：turn 级预算（帧间）与帧内 action 预算；耗尽一律回 queued，等下条消息续跑，不丢 checkpoint。
6. **已承诺结果不可回滚**：`_defer_failed_step_after_completed_checkpoint` 把"连走多步时后半段失败"显式建模为排队暂停，并给用户明确的继续提示。
7. **最小授权**：完整 manifest 仅服务端鉴权，模型只见投影；兄弟帧意图作为 out-of-scope 边界输入。
8. **最新消息优先**：恢复帧的编译输入强制用当前用户消息，避免长驻帧用创建时旧消息顶掉新指令。
9. **终态竞争单点裁决**：写 assistant 终态消息前做最后一次取消检查，`begin_completion` 占位，保证取消与正常完成只有一个赢家。
10. **可观测性优先**：内层 trace 逐条 commit，使运行中状态可被流式中转实时观察；所有事件带统一 `execution_engine` 标记。
