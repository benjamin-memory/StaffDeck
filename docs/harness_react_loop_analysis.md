# ReAct / 技能循环实现分析

> 分析对象：
> - `backend/app/core/harness_agent.py`（内层 ReAct 循环，1140 行）
> - `backend/app/llm/prompts/harness_agent_prompt.md`（循环系统提示词）
> - `backend/app/core/harness_capability_invoker.py`（能力执行器）
> - `backend/app/core/task_request_compiler.py`（任务需求编译）
> - 以及 `harness_v2_engine.py::_run_frame`（外层 SOP 步骤循环）
>
> 分析日期：2026-09-10
> 相关文档：[AgentLoop 分析](agent_loop_analysis.md)、[Harness v2 执行引擎分析](harness_v2_analysis.md)

## 1. 总体结构：两层嵌套循环

Harness v2 的"技能循环"不是单层 ReAct，而是**两层嵌套**：

```
TaskFrame（一个 SOP 帧 / 一个会话帧）
└─ 外层：SOP 步骤循环  (_run_frame, harness_v2_engine.py:875 while)
    │   一个 SOP 帧可在一轮内连续推进多个图节点
    └─ 内层：单步 ReAct 循环  (HarnessTaskAgent.run, harness_agent.py:165 for)
        │   Thought 被隐式折叠，模型每轮只输出一个动作 JSON
        ├─ action="tool"   → 执行能力 → tool 结果回灌 → 继续
        └─ action="finish" → 结束本步骤，产出 TaskExecutionResult
```

- **内层循环**是标准 ReAct（Reason+Act）的严格协议化版本：没有自由文本 Thought，模型每轮必须输出且只能输出一个 JSON 动作（`tool` 或 `finish`），系统把工具结果追加进 transcript 后再次调用模型；
- **外层循环**负责 SOP 图推进：内层返回 `completed` 后，调用 `AgentLoop._apply_step_result` 迁移图节点，再在动作预算允许时把下一步重新编译成新的 `TaskRequirement` 喂给内层。

## 2. 内层 ReAct 循环：`HarnessTaskAgent.run`

### 2.1 输入：隔离的 TaskRequirement（而非对话历史）

类注释原文："Runs one isolated TaskRequirement without outer conversation messages"（harness_agent.py:55）。

`TaskRequestCompiler.compile`（task_request_compiler.py:110）把一个持久化帧 + 当前 SOP 节点编译成自包含的 `TaskRequirement`（:56）：

| 字段 | 内容 |
|---|---|
| `goal` / `requirements` | 当前节点 instruction + 待补槽位 + 帧需求合成的任务边界 |
| `required_slots` / `known_slots` | 当前节点 `expected_user_info` 中尚未满足的字段 / 已有槽位 |
| `completion_criteria` | 完成门槛：字段收集、技能 goal、**强制能力成功调用**、强制知识库检索 |
| `required_capability_names` / `required_knowledge_base_ids` | 当前节点标记为**强制执行**的能力与知识库 |
| `allowed_transitions` | 当前图节点允许的出边（`next_step_id` 白名单来源） |
| `sop_context` | 当前节点上下文 |
| `capability_manifest` | 冻结的能力清单（available 可直接调 / catalog 需先激活） |
| `source_user_message` | 驱动本轮的**最新**用户原话（截断 4000 字） |
| `out_of_scope_task_intents` | 兄弟帧接管的需求，禁止越界处理 |
| `prior_task_results` | 依赖帧结果 + 同会话被槽位引用的早先能力结果 |
| `memory_projection` | 长期记忆事实（与任务冲突时以任务为准） |
| `current_time` | 用户时区当前时间（含星期、UTC 偏移），防止模型用 shell 算日期出错 |
| `attachments` / `published_deliverables` | 已物化附件与历史交付物 |

### 2.2 动作协议（HarnessAction，harness_agent.py:42）

```python
action: Literal["tool", "finish"]
# tool 时：
tool_name: str | None
arguments: dict
# finish 时：
status: "completed" | "awaiting_user" | "handoff" | "failed"
reply_fragment: str       # 可直接展示的 Markdown 正文
slot_updates: dict        # 只允许稳定结构化字段，禁止存原文/message_content
next_step_id: str | None  # 必须出自 allowed_transitions
task_summary: str
structured_result: Any
```

系统提示词（harness_agent_prompt.md）的关键约束：

- **串行**：每轮至多一个 tool，禁止并行 tool_calls（prompt:91）；
- 能力名只能出现在 `tool_name`，绝不能写进 `action`（防模型把动作协议与工具调用混淆）；
- `finish.reply_fragment` 是面向用户的 Markdown（排版规则：多主题必须小标题分组、禁止"参考来源"页脚、首行直接给信息、禁套"结论/过程/交付物"三段式）；
- 除单个 JSON 对象外不得输出任何内容（无围栏、无推理过程、无 Markdown）；
- 证据足够立即结束，知识检索硬预算 **2 次有效检索**，禁止换同义词反复检索。

### 2.3 每轮迭代做什么（harness_agent.py:165–648）

```
for iteration in 1..max_actions:
  1. 取消检查；步骤截止时间检查（超时 → SOP_STEP_TIMEOUT，failed）
  2. 组装 payload：
       task_requirement + harness_transcript + iteration
       + remaining_actions + knowledge_search_budget
       (+ agent_loop_memory 最近任务摘要)
       (+ 附件隔离视觉上下文)
  3. 取动作：pending_actions 队列优先，否则调 LLM
       - 支持一次返回动作序列（{"actions":[...]}），finish 必须是序列最后一个
       - Schema 校验失败：注入 protocol_repair 给 1 次自我修复机会
  4. finish → 完成门槛校验（见 2.4）→ 返回 TaskExecutionResult
  5. tool → 三道前置校验（见 2.5）→ invoke_tool 执行
  6. 结果回灌 transcript，更新预算/引用/证据/产物/已激活能力
循环耗尽 → status="action_budget"（帧保留排队，下轮继续）
```

调用 LLM 时用 `llm_operation("harness.task_action", ...)` 打可观测 span（带 task_frame_id/iteration/protocol_attempt，:215）。有步骤截止时间时，用 `_deadline_llm_client`（:902）把剩余时间压进模型 timeout，避免超时后还在等模型。

### 2.4 finish 的完成门槛（防"假完成"）

模型想 `finish` 时，代码做两道服务端拦截：

1. **强制能力未成功执行**（:350，`_missing_required_capabilities` :814）：
   `finish(completed)` 时若 `required_capability_names` 里有能力没在本帧成功过、或强制知识库没检索过 → 不接受结束，往 transcript 注入一条 `REQUIRED_CAPABILITY_NOT_INVOKED` 的 tool 错误让模型重做；
2. **未尝试就宣告失败**（:392）：
   `finish(failed)` 但强制能力在本节点**一次都没调用过** → 拦截一次（仅一次，防死锁），注入 `REQUIRED_CAPABILITY_NOT_ATTEMPTED`，要求"先实际调用，再依据真实结果决定成败"。这是真实事故驱动的：模型带着上一步的工具报错直接在提交节点宣告失败。

通过门槛后 `_finish_result`（:842）再做两层归一化：

- `next_step_id` 不在 `allowed_transitions` 白名单 → 置空（防止模型幻觉跳步，图拓扑才是权威）；
- 当前节点 `type == "handoff"` 且未指定下一步 → 状态强制归一化为 `handoff`（可选 handoff 动作不会把成功步骤误判成 handoff，但专用 handoff 终节点拥有路由决定权）。

### 2.5 tool 调用的三道前置校验

1. **冻结清单校验**（:459）：`tool_name` 不在 manifest 的 `allowed_names()` → 不执行，回 `TOOL_NOT_AVAILABLE`（"不在当前 TaskFrame 的冻结清单中"）；
2. **知识检索预算**（:475）：同一帧已有 2 次有效检索 → 不执行，回 `KNOWLEDGE_SEARCH_BUDGET_EXHAUSTED`；
3. **不可重试去重**（:491）：相同 tool+arguments 此前失败且错误标 `retryable=false` → 不执行，回 `NON_RETRYABLE_ACTION_REPEATED`。签名集合只活在本次调用内（不持久化），避免后续轮次继承已过时的失败。

工具异常被捕获为标准错误结果（`HARNESS_TOOL_ERROR`），不会炸穿循环；取消与围栏异常直接上抛。

### 2.6 能力渐进式披露（Progressive Disclosure）

模型**不能直接调用全部工具**，分三档：

- `available`：帧开始就已授权、可直接调用的能力；
- `catalog`：只含名称/类型/描述的紧凑目录（约 8000 字符预算，可能截断），需先调 `capability_describe` 拉取完整 input schema 并**激活**后才能用；
- 目录截断或无候选时，调 `capability_search` 搜索完整冻结目录。

激活在两处生效：invoker 侧维护 `_activated_names`，未激活调用返回 `CAPABILITY_NOT_ACTIVATED`（harness_capability_invoker.py:178）；agent 侧 `_activate_described_capabilities`（:680）校验 describe 返回的 `snapshot_revision` 与本帧一致后才把能力并入 manifest，防止用陈旧 schema 提权。

能力种类（invoker 分发，harness_capability_invoker.py:223）：

| kind | 执行路径 |
|---|---|
| `internal` | 内置：capability_search/describe、knowledge_search、lark_cli、list/read_published_deliverable 等 |
| `file` | typed 文件工具 + exec_command，运行在每帧隔离 workspace，受 OS 沙箱与网络策略约束 |
| `general_skill` | 物化并读取 GeneralSkill 包（SKILL.md 工作流说明），把说明载入 transcript，**不是第二套 runner** |
| `knowledge` | 知识库检索（元数据交集鉴权） |
| `tool` | HTTP/MCP/A2A 外部工具，大 JSON 自动落沙箱文件并支持引用自动解引用 |

### 2.7 隔离 transcript 与上下文裁剪

内层"记忆"是 `harness_transcript`——只含本帧的 assistant 动作与 tool 结果条目，**没有外层闲聊历史**。它同时是恢复载体：

- **checkpoint 作用域**（:74）：`same_frame`（同一 task_frame）才恢复完整 transcript；`same_step`（同一节点）才恢复 capability_results，换步即清空工具结果，避免旧步骤上下文污染；
- **恢复时补最新用户消息**（:85）：同帧恢复时若最新用户回复与 transcript 末尾的 user 条目不同，显式追加（真实事故：模型忽略 `source_user_message` 字段反复索要已给过的确认）；
- **投影裁剪** `_transcript_for_model`（:994）：
  - 默认只保留最近 6 条完整条目；更早的 tool 结果压缩成回执（success/error + 关键 data 字段 + `history_receipt`（omitted_chars + sha256））；
  - GeneralSkill 的**最新版说明始终保留**（它定义当前工作流），旧版与配套 assistant 动作丢弃；
  - 硬上限 40 条：先保技能说明，剩余预算给最新交互；
  - 工具结果在写入 transcript 前先经 `_bounded_capability_result`（:960）截断到 12000 字符。

checkpoint 结构（`finish()` 内联构造，:147）：

```python
{version, task_frame_id, step_id, transcript,
 citations[-20:], evidence_results[-10:], capability_results[-20:],
 satisfied_required_knowledge_ids, successful_knowledge_searches,
 artifacts[-20:], loaded_general_skill_names[-20:],
 recent_task_summaries[-8:]}
```

它存入 `harness_agent_loops.checkpoint_json`，帧暂停（awaiting_user/action_budget）后下一回合凭此续跑。

### 2.8 特殊出口

- **异步业务任务**（:542）：工具返回 `success + data.detached=true` 时立即 `finish(waiting_external_task)`，帧挂起等外部系统回调，之后由引擎桥接为 `ready_to_resume`；
- **结构化业务 JSON 适配** `_adapt_general_skill_structured_result`（:775）：GeneralSkill 要求固定 JSON 输出、模型直接吐出业务对象（而非 Harness 协议）时，若该技能确实已加载且对象里没有伪装的 action 字段，适配成 `finish(completed, structured_result=raw)`；RFC/MCP 形状的对象只当数据，绝不解释成工具调用；
- **产物收集**：工具返回的 artifacts 累积进结果；帧结束时 invoker 再做一次 workspace 快照 diff（`discover_artifacts`），把本轮新增/修改的用户文件自动发布。

## 3. 外层 SOP 步骤循环：`_run_frame` 中的 while（harness_v2_engine.py:875–1166）

内层每跑完一个步骤，外层决定"继续下一节点还是结束本帧"：

```
while remaining_actions > 0:
  取消检查；为本步设置 deadline（step_timeout_seconds，1–3600s）
  frame.target_step_id = 帧记录的 step / 会话当前 step
  1. 按当前节点构建 manifest（每步重新冻结能力与授权）
  2. compiler.compile → TaskRequirement
  3. 持久化 requirement；首次创建 HarnessRunRecord，后续步更新其 context
  4. HarnessTaskAgent.run → TaskExecutionResult
  5. 特殊处理：
     - human_handoff_resume 渠道下重复 handoff → 归一化为 completed
     - waiting_external_task → 记录恢复节点
     - 已提交节点后的续跑失败 → 保留已给用户的结果，剩余排队（不回滚）
  6. _enforce_required_slots：声明 completed 但必填槽仍缺
       → 强制改 awaiting_user 并生成追问
  7. completed 且无 next_step_id → 用图默认出边补一个下一步
  8. AgentLoop._apply_step_result：
       合并槽位、校验 next_step_id 合法性（幻觉跳步修复）、
       同步 awaiting_input、处理图分岔的 pending 兄弟分支
  9. TurnFinalizer.finalize（_finalize_execution_after_reply）：
       返回 continued / completed / handoff
       - handoff 但节点未声明该能力 → 降级 failed (HANDOFF_NOT_ALLOWED)
       - completed / handoff → 结束本帧
       - 预算耗尽 / 技能结束 / 步骤未推进 → action_budget 排队
       - 否则 frame.target_step_id 指向新节点，继续 while
多步结果 _combine_results 合并：只保留终态步骤回复，中间回复不拼接
finish_run / finish_frame / checkpoint 落库
```

要点：

- **能力与授权按步重新冻结**：每个节点重新 build manifest 并在 RunRecord 存 `capability_snapshot`，撤权/归档的能力在执行前复检（`CAPABILITY_AUTHORIZATION_REVOKED`）；
- **图拓扑权威**：模型给的 `next_step_id` 先经内层白名单（allowed_transitions）、再经外层 `_apply_step_result` 的 `_skill_has_step` 双重校验，非法跳转记 `step_agent_result_repaired` 并忽略；
- **一帧多步**：一个 SOP 帧可在一轮内连续走多个节点，每步都是一次完整内层 ReAct；动作预算（默认 32、上限 100）是内外两层共享的同一计数器（外层 `remaining_turn_actions` 扣减内层 `action_count`）。

## 4. 能力执行的可靠性设计（HarnessCapabilityInvoker）

每次 `invoke`（harness_capability_invoker.py:168）：

1. 取消检查 → **执行租约续约**（会话租约 + turn 租约 + 帧租约，丢失即 `HarnessExecutionFenced`）；
2. 三道鉴权：在冻结清单中 → 已激活 → 当前仍授权（执行前复检 DB）；
3. **副作用幂等**：外部写类工具按 `ToolReplayPolicy` 生成 `logical_action_key`（租户+帧+步骤+工具+关键参数的哈希），落 `HarnessInvocationRecord`：
   - 已有 completed 成功记录 → 直接重放缓存结果，不再触发外部副作用；
   - 已有记录但结果未知（超时/连接重置，可能已写入）→ 拒绝自动重试（`TOOL_CALL_OUTCOME_UNKNOWN`），要求人工核对；
   - 确认未发出的配置/授权类失败 → 释放 claim 允许修好后重试；
   - 并发 insert 撞唯一约束时回查重放；
4. 每次调用落完整审计（参数脱敏、结果审计）与 trace（`harness_tool_completed` 逐次 commit，供流式中转实时看到帧内进度）。

## 5. 一次典型 ReAct 迭代的时序

```
模型 → {"action":"tool","tool_name":"knowledge_search","arguments":{...}}
invoker: 续约租约 → 鉴权 → 幂等检查 → 执行 → 落 HarnessInvocationRecord
agent:   transcript += assistant(tool 动作) + tool(截断后结果)
         更新 evidence/citations/成功检索计数
         trace: harness_tool_completed（立即 commit）
模型 → {"action":"tool","tool_name":"capability_describe", ...}  # 激活业务工具
模型 → {"action":"tool","tool_name":"submit_order", ...}         # 强制能力
模型 → {"action":"finish","status":"completed",
        "reply_fragment":"## 办理结果\n...","next_step_id":"node_confirm"}
agent:   校验强制能力已成功 ✓ → next_step_id 在白名单 ✓
外层:    图迁移到 node_confirm → TurnFinalizer=continued
         → 编译 node_confirm 的新 Requirement → 内层开新一轮 ReAct
…
外层 completed → finish_frame(queued/completed/awaiting_user/…)
```

## 6. 设计要点小结

1. **协议极简**：只有 tool/finish 两个动作、串行一轮一个工具，靠 Schema + 一次 protocol_repair 兜底，解析失败以 `HARNESS_ACTION_INVALID` 结束且帧可恢复；
2. **隔离**：内层只见编译后的 TaskRequirement 与本帧 transcript，不见原始对话；能力按帧冻结、按步重新授权，workspace 按帧隔离；
3. **服务端强制完成门槛**：强制能力必须真实成功调用、必填槽位服务端二次校验，模型无法"假完成"；
4. **预算双限**：动作数（max_actions，默认 32/上限 100）防失控循环；知识检索每帧 2 次防发散；
5. **状态全持久化**：transcript/checkpoint 入 `harness_agent_loops`，需求与能力快照入 `harness_runs`，工具调用入 `harness_invocations`，使 awaiting_user、action_budget、崩溃、外部任务等待都能跨轮/跨进程精确恢复；
6. **副作用安全**：逻辑动作键 + 结果缓存实现写操作重放/阻断，超时不自动重试，避免重复下单之类事故；
7. **图权威高于模型**：`next_step_id` 白名单 + 图校验双层拦截幻觉跳转，分岔分支由 pending steps 调度；
8. **真实事故驱动的防护**遍布注释：最新用户消息补录、未尝试不许宣告失败、时区注入防 shell 算错日期、续跑失败不覆盖已提交结果等。
