# 意图识别实现概览

> 主题：StaffDeck 中「意图识别」的实现机制
> 整理日期：2026-09-20
> 核心代码：`backend/app/core/turn_planner.py`
> 提示词：`backend/app/llm/prompts/turn_planner_prompt.md`
> 输出契约：`TURN_PLANNER_OUTPUT_SCHEMA`（`backend/app/llm/stage_protocol.py`）
> 详细逐函数拆解见：[turn_planner_analysis.md](turn_planner_analysis.md)

---

## 一句话结论

意图识别**不是关键词/规则分类器，而是一个专门的 LLM 阶段** —— `TurnPlanner`。它是 Harness v2 流水线中**唯一**负责「意图判定 + SOP 路由 + 本轮任务拆分」的环节，核心设计原则是：

> **模型负责「理解意图」，服务端负责「裁决什么合法」。**

模型产出的计划被视为**不可信的半成品**，再由服务端 `_normalize()` 对照真实状态做一次防御式重写后才落地。

---

## 一次意图识别的完整流程

```
用户消息 + 会话 / SOP / 在途任务 / 记忆 / 团队上下文
        │
        ▼
 TurnPlanner.plan()                 组装 payload + 系统提示词
        │
        ▼
 LLM generate_json（attempt 1）
        │  结构非法？→ 带校验错误让模型自修一次（attempt 2）
        ▼
 TurnPlan.model_validate            只校验「结构」合法
        │
        ▼
 _normalize()                       服务端权威重写，校验「业务」合法
        │
        ▼
 合法 TurnPlan → TaskFrame 落库 → 编译成任务执行
```

### 1. 给模型喂什么（`plan()`，turn_planner.py:56-78）

通过 `stage_payload(phase="TurnPlanner", ...)` 组装，关键 `stage_data`：

- **`current_session`**：当前 active SOP / 步骤、槽位、在途 task_frames、会话状态；
- **`available_sops`**：只给 `id / name / description / trigger_intents` 作为可路由候选。
  **刻意排除 GeneralSkill / 知识库 / 工具**，防止模型把「运行时能力」与「SOP 路由候选」混淆；
- **`interaction_mode` + `team_context`**：团队 TL 模式下注入唯一可信的成员目录。

判定规则主要由**系统提示词**（`turn_planner_prompt.md`）约束，例如：

- 明显在回答上一轮问题 → `continue_active`；
- 没有匹配 SOP 的闲聊 / 咨询 / 计算 → `kind=conversation`，不许伪造 SOP；
- `clarify` 只用于「想办 SOP 但多个 SOP 区分不了」，**缺槽位不算 clarify**；
- 只有用户明确继续某个 pending task 才用 `switch_to_pending`。

### 2. 模型输出什么：结构化 `TurnPlan`

- 顶层 `decision`：9 选 1 的路由决策
  （`continue_active / switch_to_pending / create_pending / update_pending / complete_task / start_new_task / answer_only / handoff_human / clarify`）
  ，外加 `confidence / user_intent / reason / clarification_question`；
- `task_frames[]`：本轮要执行的有序任务帧，每帧含 `kind=sop|conversation`、目标 SOP、
  `requirements`、`depends_on_task_ids`、派发目标等；
- `task_updates[]`：对既有在途任务的更新。

### 3. 结构校验 + 一次自修（`_generate_validated_plan`，:101-134）

- `SCHEMA_REPAIR_ATTEMPTS = 1`，最多尝试 2 次；
- 第一次输出不符合 pydantic schema 时，把压缩后的校验错误（`{path,type,message}`）
  作为 `_schema_repair` 回灌，要求「保留语义、只修字段类型」；
- 修复 payload 从原始入参深拷贝，避免失败输出污染基础上下文；
- 这一步只保证**结构合法**，不保证业务合法。
- 非 `LLMError` 的异常统一包装成 `LLMError("Turn Planner returned invalid JSON schema: ...")`。

### 4. 服务端权威重写（`_normalize()`，:136-372，核心）

不信任模型给的任何 id / skill / step / assignee，逐项对照服务端状态重写：

- **task_id**：只有命中「非终态在途帧」才允许复用，否则服务端新生成，
  防止伪造或复用终态 id；
- **SOP 白名单**：`target_skill_id` 不在可用 SOP 名单里 → **整帧丢弃**；
- **不许跳步**：新 SOP 强制从图首节点开始，继续中的 SOP 以服务端当前节点为准；
- **空帧兜底**：模型一帧没给，也按顶层 decision 补至少一个 conversation 帧，
  保证下游永不空转；
- **依赖闭包**：重映射依赖 id，剔除指向不存在任务或自环的依赖；
- **团队派发零臆测**：assignee 必须精确命中 `team_context.members`，
  远程帧只能依赖本批其他远程帧，否则降级为 `self`（避免永久阻塞）；
- **顶层 decision 定稿**：以重写后的第一帧反向修正顶层决策。

> 业务上不合法的帧**不报错**，而是被丢弃 / 降级 / 重写，
> 因此下游永远收到一份可执行计划。

---

## 新旧两套实现

| | 当前路径（v2） | 旧版 |
|---|---|---|
| 规划器 | `core/turn_planner.py` → `TurnPlanner` | `core/router.py` |
| 提示词 | `prompts/turn_planner_prompt.md` | `prompts/router_prompt.md` |
| 职责 | 意图 + SOP 路由 + 任务拆分一体 | 独立路由阶段 |

- v2 走 `TurnPlanner`，调用点为 `HarnessV2Engine.run()`（`harness_v2_engine.py:349`）。
- **斜杠指令**（`/skill`、`/tool`）完全绕过 LLM，由 `build_slash_turn_plan`
  直接构造计划（`harness_v2_engine.py:332-347`）。

---

## 设计要点小结

1. **单一规划阶段**：意图判定、SOP 路由、本轮任务拆分集中在一个 LLM 阶段；
   能力选择下沉到帧内 Harness Agent，规划与执行彻底解耦。
2. **模型出意图、服务端出裁决**：技能必须在白名单、节点由图状态决定、
   task_id 必须是在途非终态或服务端新生成、assignee 必须在目录内、依赖必须闭包。
3. **结构错误可修复、业务错误直接纠正**：schema 错误给一次自修；
   业务不合法的帧被丢弃 / 降级 / 重写而非报错。
4. **永不空转**：即使模型不给帧，也兜底出至少一个 conversation 帧。
5. **团队派发零臆测**：只接受目录内精确 agent_id，远程任务只能依赖本批远程任务。
6. **可观测与兼容**：规划 span 带 schema 重试属性；同时产出完整
   `turn_plan_created` 与旧版 `router_decision_created` 两类事件。
