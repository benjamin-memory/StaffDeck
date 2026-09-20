# `HarnessTaskAgent.run()` 逐句注释

> 注释对象：`backend/app/core/harness_agent.py` 的 `HarnessTaskAgent.run`，源文件 `:57-660`
> 分析日期：2026-09-15
> 阅读建议：先读 [harness_agent.py 分析](harness_agent_analysis.md) 了解整体设计；本篇只对 `run()` 做逐句中文注释，代码本身保持原样（仅新增 `#` 注释），不改变任何逻辑。
>
> 标注约定：
> - `# ▸ ...` 为本次新增的逐句讲解注释；源码原有注释原样保留。
> - `【返回点 N】` 标注函数的全部 8 个出口。
> - 行号沿用源文件，便于对照。

---

## 1. 函数签名与参数（`:57-70`）

```python
def run(
    self,
    requirement: TaskRequirement,          # ▸ 自包含任务：goal/requirements/槽位/完成标准/冻结能力清单/SOP 节点上下文
    model_config: ModelConfig,              # ▸ 模型配置；会被 _deadline_llm_client 按剩余截止时间收紧 timeout
    invoke_tool: ToolInvoker,               # ▸ 回调 (tool_name, arguments) -> result_dict，真正执行能力（含二次授权）
    *,                                      # ▸ 以下全部为仅限关键字参数，调用时必须写参数名
    max_actions: int = 6,                   # ▸ 本帧动作上限（tool/finish 各算动作的预算口径），下方会 clamp 到 [1,100]
    trace_sink: TraceSink | None = None,    # ▸ 可选轨迹回调 (event_type, payload) -> None；None 表示不上报
    is_cancelled: CancellationCheck | None = None,  # ▸ 可选取消探测回调，返回 True 即抛 HarnessExecutionCancelled
    image_payloads: list[ValidatedTaskImagePayload] | None = None,  # ▸ 已校验图片载荷，与 requirement.attachments 隔离物化
    step_deadline_monotonic: float | None = None,   # ▸ 单步截止的单调时钟时刻（time.monotonic 基准）；None 不限时
    step_timeout_seconds: int | None = None,        # ▸ 人类可读的单步超时秒数，仅用于错误文案
    checkpoint: dict[str, Any] | None = None,       # ▸ 上一轮持久化的 loop_checkpoint，用于跨回合续跑
) -> TaskExecutionResult:                  # ▸ 永远返回一个结果对象（含 status/reply/checkpoint），不返回 None
```

---

## 2. 初始化与 Checkpoint 恢复（`:71-132`）

```python
        # ▸ 动作数归一化：至少 1 次、至多 100 次，防止上层传入 0/负数/超大值
        max_actions = max(1, min(int(max_actions), 100))

        # ▸ 复制一份 checkpoint，避免后续对入参字典的就地修改影响调用方；None 视为空 dict
        checkpoint = dict(checkpoint or {})

        # ▸ 从 requirement.sop_context.step 解析当前 SOP 节点 id（依次取 step_id/node_id/id）
        current_step_id = _requirement_step_id(requirement)

        # ▸ 判定层级 1：checkpoint 是否属于「同一个 TaskFrame」
        same_frame = (
            str(checkpoint.get("task_frame_id") or "") == requirement.task_frame_id
        )
        # ▸ 判定层级 2：在同帧前提下，是否还停留在「同一个 SOP 节点」
        same_step = (
            same_frame
            and str(checkpoint.get("step_id") or "") == current_step_id
        )

        # ▸ 帧级历史：仅同帧可继承；换帧则从空 transcript 开始
        transcript = _dict_items(checkpoint.get("transcript")) if same_frame else []

        # 从暂停恢复时，把用户的最新回复显式写进内层对话记录：仅靠
        # requirement.source_user_message 字段太不显眼，实测模型会忽略它
        # 而重复上一轮的提问（真实案例：确认暂停点反复索要确认）。
        # ▸ 取本轮驱动用户消息的原文（去掉首尾空白）
        latest_user_message = str(requirement.source_user_message or "").strip()

        # ▸ 仅在「同帧且已有历史且有新回复」时才考虑补写
        if same_frame and transcript and latest_user_message:
            # ▸ 从后往前找 transcript 中最近一条 role=user 的文本
            previous_user_entry = next(
                (
                    str(item.get("content") or "")
                    for item in reversed(transcript)
                    if item.get("role") == "user"
                ),
                None,
            )
            # ▸ 若最新回复与历史最后一条 user 不同，显式追加一条 user 消息，
            # ▸ 防止模型恢复后忽略新回复、重复上一轮提问
            if previous_user_entry != latest_user_message:
                transcript.append({"role": "user", "content": latest_user_message})

        # ▸ —— 以下按「帧级 / 步骤级」两种作用域恢复 ——

        # ▸ 帧级：知识引用、证据包、产物，跨节点保留（同帧）
        citations = _dict_items(checkpoint.get("citations")) if same_frame else []
        evidence_results = (
            _dict_items(checkpoint.get("evidence_results")) if same_frame else []
        )

        # ▸ 步骤级：本节点已完成的能力结果，仅同步骤保留；换到新节点要重新统计
        capability_results = (
            _dict_items(checkpoint.get("capability_results")) if same_step else []
        )
        # ▸ 步骤级：本节点「强制知识库」中已被满足的 id 集合，仅同步骤保留
        satisfied_required_knowledge_ids = set(
            _string_list(checkpoint.get("satisfied_required_knowledge_ids"))
            if same_step
            else []
        )
        # ▸ 步骤级：本节点「有效知识检索」成功次数（受每帧 2 次预算约束），仅同步骤保留
        successful_knowledge_searches = (
            int(checkpoint.get("successful_knowledge_searches") or 0)
            if same_step
            else 0
        )

        # ▸ 帧级：产物清单、已加载的 GeneralSkill 名称，跨节点保留
        artifacts = _dict_items(checkpoint.get("artifacts")) if same_frame else []
        loaded_general_skill_names = (
            _string_list(checkpoint.get("loaded_general_skill_names"))
            if same_frame
            else []
        )

        # ▸ 跨步骤：最近任务摘要总是读取（不看 same_frame/same_step），只保留最近 8 条
        recent_task_summaries = _string_list(
            checkpoint.get("recent_task_summaries")
        )[-8:]

        # A non-retryable failure only blocks an identical call inside this
        # invocation of the AgentLoop.  Persisting the signature in the
        # checkpoint made a later user turn inherit an obsolete failure even
        # after its inputs or external state had changed.
        # ▸ 「不可重试失败」的工具指纹集合：故意不持久化，只在本次调用内生效，
        # ▸ 避免后续回合在输入/外部状态已变化后仍继承一个过时失败
        non_retryable_action_signatures: set[str] = set()

        # 强制能力从未尝试就 finish(failed) 只拦一次，避免与固执模型互相死锁。
        # ▸ 守卫 B 是否已触发过的开关（整轮最多打回一次）
        failed_finish_without_attempt_blocked = False

        # ▸ 当前可直接调用的能力名集合（冻结清单中 available 的项）；
        # ▸ 循环内还会因 capability_describe 成功而动态扩充
        allowed_names = requirement.capability_manifest.allowed_names()

        # ▸ 读取帧内 Agent 的系统提示词（harness_agent_prompt.md），每次 run 读一次
        system_prompt = PROMPT_PATH.read_text(encoding="utf-8").strip()

        # ▸ 待执行动作队列：LLM 一次可返回多个动作，首个立即执行，其余缓存到这里
        pending_actions: list[HarnessAction] = []
```

---

## 3. `finish()` 收口闭包（`:134-163`）

```python
        # ▸ 所有终态都经此闭包返回：维护任务摘要 + 写回 loop_checkpoint
        def finish(result: TaskExecutionResult) -> TaskExecutionResult:
            # ▸ 摘要优先取 task_summary，缺省取 reply_fragment；用 split/join 折叠多余空白为单行
            summary = " ".join(
                str(result.task_summary or result.reply_fragment or "").split()
            ).strip()

            # ▸ 记录摘要的条件：不是 action_budget、摘要非空、且不与上一条重复
            if (
                result.status != "action_budget"
                and summary
                and (
                    not recent_task_summaries
                    or recent_task_summaries[-1] != summary
                )
            ):
                # ▸ 单条摘要截断 1000 字符后追加（最终在 checkpoint 里再截到 8 条）
                recent_task_summaries.append(summary[:1_000])

            # ▸ 写回版本化的 checkpoint，供下回合按帧/步骤两级恢复（各列表均做长度上限）
            result.loop_checkpoint = {
                "version": 1,
                "task_frame_id": requirement.task_frame_id,
                "step_id": current_step_id,
                "transcript": _transcript_for_model(transcript),  # ▸ 有界投影（最近6条+技能常留+回执）
                "citations": citations[-20:],
                "evidence_results": evidence_results[-10:],
                "capability_results": capability_results[-20:],
                "satisfied_required_knowledge_ids": sorted(
                    satisfied_required_knowledge_ids
                ),
                "successful_knowledge_searches": successful_knowledge_searches,
                "artifacts": artifacts[-20:],
                "loaded_general_skill_names": loaded_general_skill_names[-20:],
                "recent_task_summaries": recent_task_summaries[-8:],
            }
            return result
```

---

## 4. 主循环（`:165-660`）

### 4.1 循环顶端：取消与单步截止（`:165-177`）

```python
        # ▸ iteration 从 1 计到 max_actions（含），用于让模型感知剩余预算与 trace 关联
        for iteration in range(1, max_actions + 1):
            # ▸ 检查点 1：若外部已请求取消，立即抛 HarnessExecutionCancelled（向上穿透，不在本函数捕获）
            _raise_if_cancelled(is_cancelled)

            # ▸ 检查点 2：单步软截止。已超时则直接构造 SOP_STEP_TIMEOUT 失败结果返回
            if _deadline_expired(step_deadline_monotonic):
                # 【返回点 1】循环开始即超时；本轮尚未消耗动作，故 action_count=iteration-1
                return finish(_step_timeout_result(
                    requirement,
                    action_count=iteration - 1,
                    timeout_seconds=step_timeout_seconds,
                    capability_results=capability_results,
                    citations=citations,
                    evidence_results=evidence_results,
                    artifacts=artifacts,
                    trace_sink=trace_sink,
                ))
```

### 4.2 组装本轮 LLM Payload（`:178-204`）

```python
            # ▸ 把任务需求整体序列化为 JSON 安全的 dict
            requirement_payload = requirement.model_dump(mode="json")

            # ▸ 将原始附件 + 已校验图片，转成「描述符」与「隔离上下文」两部分
            # ▸ （isolated_attachment_context 负责隔离物化，避免模型直接触碰原始路径）
            attachment_descriptors, attachment_context = isolated_attachment_context(
                requirement.attachments,
                image_payloads,
            )
            # ▸ 用安全的描述符替换 requirement 里的原始附件字段
            requirement_payload["attachments"] = attachment_descriptors

            # ▸ 组装给模型的完整 payload
            payload = {
                "task_requirement": requirement_payload,
                "harness_transcript": _transcript_for_model(transcript),  # ▸ 有界历史，非全量
                "iteration": iteration,                                  # ▸ 当前第几轮
                "remaining_actions": max_actions - iteration + 1,        # ▸ 含本轮在内的剩余动作数
                # ▸ 知识检索预算：每帧最多 2 次有效检索，实时回传已用/剩余，约束模型不要刷检索
                "knowledge_search_budget": {
                    "maximum_successful_calls": MAX_SUCCESSFUL_KNOWLEDGE_SEARCHES_PER_TASK,
                    "successful_calls": successful_knowledge_searches,
                    "remaining_successful_calls": max(
                        0,
                        MAX_SUCCESSFUL_KNOWLEDGE_SEARCHES_PER_TASK
                        - successful_knowledge_searches,
                    ),
                },
            }

            # ▸ 若存在历史任务摘要，作为 agent_loop_memory 注入，帮助长循环记住做过的子任务
            if recent_task_summaries:
                payload["agent_loop_memory"] = {
                    "recent_task_summaries": list(recent_task_summaries),
                }

            # ▸ 若有图片等附件隔离上下文，挂到 conversation_context 提供给模型
            if attachment_context is not None:
                payload["conversation_context"] = attachment_context
```

### 4.3 获取动作：队列优先 + 最多两次协议修复（`:205-297`）

```python
            try:
                # ▸ 本轮最终要执行的动作；validation_error 记录最近一次协议错误
                action: HarnessAction | None = None
                validation_error: Exception | None = None

                # ▸ 路径 A：上一次 LLM 返回了多动作且还有缓存 → 直接取队首，不再调用 LLM
                if pending_actions:
                    action = pending_actions.pop(0)
                else:
                    # ▸ 路径 B：需要调用 LLM。最多 2 次尝试（第 2 次带 protocol_repair）
                    for protocol_attempt in range(2):
                        # Persist a stable link between this LLM span and the Harness
                        # iteration that consumes it.  Timing projections must not
                        # infer this relationship from overlapping wall-clock windows.
                        # ▸ 用 llm_operation span 记录可观测性：把「某次 LLM 调用」与
                        # ▸ 「消费它的 Harness iteration / 第几次协议尝试」稳定关联
                        with llm_operation(
                            "harness.task_action",
                            task_frame_id=requirement.task_frame_id,
                            iteration=iteration,
                            protocol_attempt=protocol_attempt + 1,
                        ):
                            # ▸ 构造一个 timeout 被收紧到「剩余截止时间」的 LLM 客户端
                            # ▸ （硬超时，保证阻塞中的模型调用也能被单步时限打断）
                            client = _deadline_llm_client(
                                model_config,
                                step_deadline_monotonic,
                            )
                            # ▸ 调模型拿原始 JSON：优先 generate_json_sequence（可一次返回多动作），
                            # ▸ 老客户端/测试桩回退 generate_json
                            raw = _generate_harness_action_json(
                                client,
                                system_prompt,
                                payload,
                            )

                        # ▸ —— 解析模型输出 ——
                        try:
                            # ▸ 规范化为动作列表：支持单动作 / {"actions":[...]} / 裸数组；
                            # ▸ 空序列报错；finish 必须位于序列末尾
                            actions = _harness_actions_from_raw(raw)
                            action = actions[0]              # ▸ 首个动作本轮执行
                            pending_actions.extend(actions[1:])  # ▸ 其余入队，后续轮次直接取用
                            # ▸ 一次返回多个动作时上报轨迹
                            if len(actions) > 1 and trace_sink:
                                trace_sink(
                                    "harness_action_sequence_accepted",
                                    {
                                        "iteration": iteration,
                                        "action_count": len(actions),
                                        "actions": [item.action for item in actions],
                                    },
                                )
                        except (ValidationError, ValueError) as exc:
                            # ▸ 动作信封不合法。先尝试「指令型 GeneralSkill 裸业务 JSON」适配：
                            # ▸ 若已加载某技能且模型直接吐了业务 JSON（无伪造 action 字段），
                            # ▸ 将其安全包装成 finish(completed, structured_result=raw)
                            action = _adapt_general_skill_structured_result(
                                raw,
                                loaded_general_skill_names=loaded_general_skill_names,
                            )
                            if action is not None:
                                # ▸ 适配成功：上报后跳出协议重试循环，进入动作分发
                                if trace_sink:
                                    trace_sink(
                                        "harness_structured_result_adapted",
                                        {
                                            "iteration": iteration,
                                            "source": loaded_general_skill_names[-1],
                                            "result_type": type(raw).__name__,
                                        },
                                    )
                                break

                            # ▸ 适配失败，记下原始校验错误
                            validation_error = exc

                            # ▸ 第 1 次失败：注入 protocol_repair 反馈后重试
                            if protocol_attempt == 0:
                                payload = {
                                    **payload,
                                    "protocol_repair": {
                                        "message": (
                                            "上一次输出不符合 HarnessAction Schema。请只修正动作"
                                            "协议，不要改变任务意图。"
                                        ),
                                        "invalid_output": raw,                 # ▸ 回传非法输出
                                        "validation_error": str(exc),          # ▸ 回传校验报错
                                        # ▸ 给出合法 tool 信封样例
                                        "required_tool_envelope": {
                                            "action": "tool",
                                            "tool_name": (
                                                "capability_manifest 中的已授权能力名称"
                                            ),
                                            "arguments": {},
                                        },
                                        # ▸ 给出合法 finish 信封样例
                                        "required_finish_envelope": {
                                            "action": "finish",
                                            "status": "completed | awaiting_user | handoff | failed",
                                        },
                                    },
                                }
                                if trace_sink:
                                    trace_sink(
                                        "harness_action_repair_requested",
                                        {
                                            "iteration": iteration,
                                            "error": str(exc),
                                        },
                                    )
                                continue  # ▸ 带反馈进入第 2 次协议尝试
                            # ▸ 第 2 次仍失败：抛出，交由外层 except 统一处理
                            raise
                        # ▸ 解析/适配成功，退出 protocol_attempt 循环
                        break

                # ▸ 理论上走到这里 action 应已就绪；为空则把保存的错误抛出，否则报通用错误
                if action is None:
                    if validation_error is not None:
                        raise validation_error
                    raise RuntimeError("Harness action generation returned no action.")

            except (ValidationError, ValueError, LLMError) as exc:
                # ▸ 动作生成/解析阶段的可预期失败（含两次修复仍失败、LLM 调用异常）

                # ▸ 若此刻恰好已超时，优先按超时终态返回
                if _deadline_expired(step_deadline_monotonic):
                    # 【返回点 2】生成动作期间超时
                    return finish(_step_timeout_result(
                        requirement,
                        action_count=iteration - 1,
                        timeout_seconds=step_timeout_seconds,
                        capability_results=capability_results,
                        citations=citations,
                        evidence_results=evidence_results,
                        artifacts=artifacts,
                        trace_sink=trace_sink,
                    ))

                # ▸ 上报动作失败轨迹
                if trace_sink:
                    trace_sink(
                        "harness_action_failed",
                        {
                            "iteration": iteration,
                            "error": str(exc),
                        },
                    )

                # 【返回点 3】协议级失败终态。上层凭 HARNESS_ACTION_INVALID 把帧重新置为
                # queued 并保留 AgentLoop，使下一轮可恢复，而不是把帧判死
                return finish(TaskExecutionResult(
                    task_frame_id=requirement.task_frame_id,
                    status="failed",
                    reply_fragment="当前任务的执行模型没有返回有效动作。",
                    task_summary="Harness 动作解析失败。",
                    capability_results=capability_results,
                    action_count=iteration,
                    error={"code": "HARNESS_ACTION_INVALID", "message": str(exc)},
                ))
```

### 4.4 动作执行前的取消/截止复查与轨迹（`:327-348`）

```python
            # ▸ 检查点 3：拿到动作后、执行前再查一次取消
            _raise_if_cancelled(is_cancelled)

            # ▸ 检查点 4：LLM 耗时可能已越过截止时间，再查一次（这次本轮已消耗一个动作）
            if _deadline_expired(step_deadline_monotonic):
                # 【返回点 4】动作生成完成后发现超时，action_count 计为 iteration
                return finish(_step_timeout_result(
                    requirement,
                    action_count=iteration,
                    timeout_seconds=step_timeout_seconds,
                    capability_results=capability_results,
                    citations=citations,
                    evidence_results=evidence_results,
                    artifacts=artifacts,
                    trace_sink=trace_sink,
                ))

            # ▸ 上报「一个动作已通过校验、开始处理」
            if trace_sink:
                trace_sink(
                    "harness_action_created",
                    {
                        "iteration": iteration,
                        "action": action.action,
                        "tool_name": action.tool_name,
                    },
                )
```

### 4.5 `finish` 分支与两道强制能力守卫（`:349-456`）

```python
            if action.action == "finish":
                # ▸ 计算本节点「尚未成功完成」的强制能力 / 强制知识库（缺失项形如 knowledge_search:<id>）
                missing_capabilities = _missing_required_capabilities(
                    requirement,
                    capability_results,
                    satisfied_required_knowledge_ids,
                )

                # —— 守卫 A：想 completed（或未指定状态）但强制能力没做 ——
                if action.status in {None, "completed"} and missing_capabilities:
                    # ▸ 不接受完成：往 transcript 灌一对「assistant finish / tool 校验失败」，
                    # ▸ 让模型在同一循环内看到拒绝原因后真正去执行
                    transcript.extend(
                        [
                            {
                                "role": "assistant",
                                "action": "finish",
                                "status": action.status or "completed",
                            },
                            {
                                "role": "tool",
                                "tool_name": "harness_requirement_check",
                                "result": {
                                    "success": False,
                                    "error": {
                                        "code": "REQUIRED_CAPABILITY_NOT_INVOKED",
                                        "message": (
                                            "当前 SOP 节点尚未成功执行强制能力："
                                            + "、".join(missing_capabilities)
                                        ),
                                    },
                                },
                            },
                        ]
                    )
                    if trace_sink:
                        trace_sink(
                            "harness_completion_blocked",
                            {
                                "iteration": iteration,
                                "reason": "required_capability_not_invoked",
                                "missing_capabilities": missing_capabilities,
                            },
                        )
                    continue  # ▸ 打回，进入下一 iteration 强迫模型调用能力

                # 真实案例：提交节点的模型带着上一步骤的工具报错直接
                # finish(failed)，全程没在本节点尝试过强制能力。失败结论
                # 必须建立在真实尝试之上，先打回一次要求实际调用。
                # —— 守卫 B：想 failed 但强制能力在本节点从未尝试（整轮仅拦一次）——
                if (
                    action.status == "failed"
                    and missing_capabilities
                    and not failed_finish_without_attempt_blocked
                ):
                    # ▸ 收集本节点 capability_results 里出现过的工具名（尝试过的）
                    attempted = {
                        str(item.get("tool_name") or "")
                        for item in capability_results
                        if isinstance(item, dict)
                    }
                    # ▸ 在缺失项里筛出「非知识检索、且连尝试都没尝试过」的强制能力
                    unattempted = [
                        name
                        for name in missing_capabilities
                        if not name.startswith("knowledge_search:")
                        and name not in attempted
                    ]
                    if unattempted:
                        # ▸ 标记本守卫已用，避免反复打回与固执模型死锁
                        failed_finish_without_attempt_blocked = True
                        transcript.extend(
                            [
                                {
                                    "role": "assistant",
                                    "action": "finish",
                                    "status": "failed",
                                },
                                {
                                    "role": "tool",
                                    "tool_name": "harness_requirement_check",
                                    "result": {
                                        "success": False,
                                        "error": {
                                            "code": (
                                                "REQUIRED_CAPABILITY_NOT_ATTEMPTED"
                                            ),
                                            "message": (
                                                "当前 SOP 节点的强制能力在本节点内"
                                                "尚未尝试调用，不能直接宣告失败："
                                                + "、".join(unattempted)
                                                + "。请先实际调用，再依据其真实"
                                                "结果决定完成或失败。"
                                            ),
                                        },
                                    },
                                },
                            ]
                        )
                        if trace_sink:
                            trace_sink(
                                "harness_completion_blocked",
                                {
                                    "iteration": iteration,
                                    "reason": "required_capability_not_attempted",
                                    "missing_capabilities": unattempted,
                                },
                            )
                        continue  # ▸ 打回一次，要求先真实调用

                # ▸ 两道守卫都通过（或无强制能力）：构造最终结果并返回
                # 【返回点 5】模型主动 finish（completed/awaiting_user/handoff/failed 之一）
                return finish(_finish_result(
                    requirement,
                    action,
                    citations,
                    evidence_results,
                    capability_results,
                    artifacts,
                    action_count=iteration,
                ))
```

### 4.6 `tool` 分支：名称白名单与知识预算（`:458-489`）

```python
            # ▸ 以下为 action.action == "tool" 的处理
            # ▸ 规范化工具名（去空白）
            tool_name = str(action.tool_name or "").strip()

            # ▸ 闸门 1：空名或不在冻结清单（含本轮动态激活集合）内 → 不执行，
            # ▸ 伪造一条 TOOL_NOT_AVAILABLE 工具错误回灌 transcript，让模型自纠
            if not tool_name or tool_name not in allowed_names:
                transcript.append(
                    {
                        "role": "tool",
                        "tool_name": tool_name,
                        "result": {
                            "success": False,
                            "error": {
                                "code": "TOOL_NOT_AVAILABLE",
                                "message": "该能力不在当前 TaskFrame 的冻结清单中。",
                            },
                        },
                    }
                )
                continue  # ▸ 进入下一 iteration

            # ▸ 闸门 2：knowledge_search 已用满每帧 2 次有效检索 → 不再实际检索，
            # ▸ 返回预算耗尽的工具错误，要求用已有证据完成、不要换同义查询
            if (
                tool_name == "knowledge_search"
                and successful_knowledge_searches
                >= MAX_SUCCESSFUL_KNOWLEDGE_SEARCHES_PER_TASK
            ):
                result = {
                    "success": False,
                    "error": {
                        "code": "KNOWLEDGE_SEARCH_BUDGET_EXHAUSTED",
                        "message": (
                            "当前 TaskFrame 已完成两次有效知识检索。请使用已有证据完成"
                            "原始需求；不要扩展相邻主题或继续改写同义查询。"
                        ),
                    },
                }
            else:
```

### 4.7 不可重试去重与实际调用（`:490-539`）

```python
                # ▸ 计算本次调用指纹（工具名 + 参数的规范化 JSON），用于不可重试去重
                action_signature = _action_signature(
                    tool_name,
                    dict(action.arguments or {}),
                )

                # ▸ 闸门 3：相同指纹此前已失败且被标记 retryable=False → 本次不再执行
                if action_signature in non_retryable_action_signatures:
                    result = {
                        "success": False,
                        "error": {
                            "code": "NON_RETRYABLE_ACTION_REPEATED",
                            "message": (
                                "相同工具与参数此前已失败且不可重试，本次未再次执行。"
                                "请根据前一次错误更换工具或参数，或结束当前任务。"
                            ),
                            "retryable": False,
                        },
                    }
                    if trace_sink:
                        trace_sink(
                            "harness_action_failed",
                            {
                                "iteration": iteration,
                                "tool_name": tool_name,
                                "error": {
                                    "code": "NON_RETRYABLE_ACTION_REPEATED",
                                    "message": (
                                        "模型重复提交了已标记为不可重试的相同工具调用；"
                                        "调用未执行，AgentLoop 将继续重新规划。"
                                    ),
                                    "retryable": False,
                                },
                            },
                        )
                else:
                    # ▸ 闸门通过，真正调用注入的能力执行器
                    try:
                        _raise_if_cancelled(is_cancelled)      # ▸ 检查点 5：调用前取消检查
                        result = invoke_tool(tool_name, dict(action.arguments or {}))
                        _raise_if_cancelled(is_cancelled)      # ▸ 检查点 6：调用后取消检查
                    except (HarnessExecutionCancelled, HarnessExecutionFenced):
                        # ▸ 取消 / 租约栅栏失效属于控制流信号，必须原样上抛，不能吞成工具错误
                        raise
                    except Exception as exc:
                        # ▸ 其他工具异常归一为 HARNESS_TOOL_ERROR 工具结果喂回模型
                        result = {
                            "success": False,
                            "error": {
                                "code": "HARNESS_TOOL_ERROR",
                                "message": str(exc),
                            },
                        }

                    # ▸ 若该结果是「不可重试失败」，登记指纹，使后续相同调用直接短路
                    if _is_non_retryable_failure(result):
                        non_retryable_action_signatures.add(action_signature)
```

### 4.8 结果裁剪、异步外部任务、历史登记（`:540-598`）

```python
            # ▸ 把工具结果裁剪到约 12k 字符，超出则变 {truncated, preview}，防止撑爆上下文
            bounded_result = _bounded_capability_result(tool_name, result)

            # ▸ 取出 data 负载，用于判断「异步外部任务」
            result_data = result.get("data")

            # ▸ 特殊终态：成功且 data.detached=True，表示异步业务任务已受理、需挂起 SOP
            if (
                result.get("success") is True
                and isinstance(result_data, dict)
                and result_data.get("detached") is True
            ):
                # ▸ 记录这次能力结果与成对 transcript
                capability_results.append(bounded_result)
                transcript.extend(
                    [
                        {
                            "role": "assistant",
                            "action": "tool",
                            "tool_name": tool_name,
                            "arguments": action.arguments,
                        },
                        {
                            "role": "tool",
                            "tool_name": tool_name,
                            "result": bounded_result,
                        },
                    ]
                )
                # ▸ 对用户的可见回复取 data.user_reply
                reply = str(result_data.get("user_reply") or "").strip()

                # 【返回点 6】等待外部异步任务；上层据此挂起帧并写 ExternalBusinessTask.resume_step_id
                return finish(TaskExecutionResult(
                    task_frame_id=requirement.task_frame_id,
                    status="waiting_external_task",
                    reply_fragment=reply,
                    capability_results=capability_results,
                    action_count=iteration,
                    task_summary=(
                        "异步业务任务已受理，等待完成后恢复 SOP。"
                        if requirement.kind == "sop"
                        else "异步业务任务已受理，可通过任务号查询进度。"
                    ),
                    structured_result={
                        "task_id": result_data.get("task_id"),
                        "status": result_data.get("status"),
                    },
                ))

            # ▸ 若本次是成功的 general_skill.* read，登记技能名（用于裸 JSON 适配与有界历史保留）
            if _is_loaded_general_skill_result(tool_name, result):
                loaded_general_skill_names.append(tool_name)

            # ▸ 成对写入 transcript：assistant 的工具调用 + tool 的（裁剪后）结果
            transcript.extend(
                [
                    {
                        "role": "assistant",
                        "action": "tool",
                        "tool_name": tool_name,
                        "arguments": action.arguments,
                    },
                    {
                        "role": "tool",
                        "tool_name": tool_name,
                        "result": bounded_result,
                    },
                ]
            )

            # ▸ 「发现类」元工具不计入业务能力结果；其余能力才累计
            if tool_name not in {"capability_search", "capability_describe"}:
                capability_results.append(bounded_result)
```

### 4.9 工具后截止检查、动态激活、证据/产物累积与 trace（`:599-648`）

```python
            # ▸ 检查点 7：工具执行后若已超时，按超时终态返回（本轮动作已消耗）
            if _deadline_expired(step_deadline_monotonic):
                # 【返回点 7】工具执行后超时
                return finish(_step_timeout_result(
                    requirement,
                    action_count=iteration,
                    timeout_seconds=step_timeout_seconds,
                    capability_results=capability_results,
                    citations=citations,
                    evidence_results=evidence_results,
                    artifacts=artifacts,
                    trace_sink=trace_sink,
                ))

            # ▸ 若本次是 capability_describe 成功，按 snapshot_revision 校验后
            # ▸ 把返回的能力描述符激活（加入 manifest.available 与 allowed_names），
            # ▸ 这是「目录中的能力 → 可实际调用」的唯一合法通道
            activated_names = _activate_described_capabilities(
                requirement,
                tool_name,
                result,
            )
            allowed_names.update(activated_names)

            # ▸ 累积工具返回的产物
            _extend_dict_list(artifacts, result.get("artifacts"))

            # ▸ 知识检索成功且确有可用证据：有效计数 +1，并登记「已满足的知识库 id」
            if tool_name == "knowledge_search" and bool(result.get("success")):
                if _has_usable_knowledge_evidence(result):
                    successful_knowledge_searches += 1
                # ▸ 优先取本次参数指定的知识库 id；缺省回退到任务要求的知识库 id
                requested_knowledge_ids = _string_list(
                    (action.arguments or {}).get("knowledge_base_ids")
                )
                satisfied_required_knowledge_ids.update(
                    requested_knowledge_ids or requirement.required_knowledge_base_ids
                )

            # ▸ 累积引用与证据包：知识检索成功且 data 为 dict 时额外存 evidence_results
            if (
                tool_name == "knowledge_search"
                and bool(result.get("success"))
                and isinstance(result.get("data"), dict)
            ):
                _extend_dict_list(citations, result.get("citations"))
                evidence_results.append(dict(result["data"]))
            else:
                # ▸ 其他工具也可能带 citations，一并合并
                _extend_dict_list(citations, result.get("citations"))

            # ▸ 上报工具完成轨迹（结果再裁剪到约 4k；GeneralSkill 仅留少量关键字段）
            if trace_sink:
                trace_sink(
                    "harness_tool_completed",
                    {
                        "iteration": iteration,
                        "tool_name": tool_name,
                        "success": bool(result.get("success")),
                        "error": result.get("error"),
                        "result": _trace_capability_result(
                            tool_name,
                            result,
                        ),
                    },
                )
```

### 4.10 动作预算耗尽（`:649-660`）

```python
        # ▸ for 循环正常结束 = 跑满 max_actions 仍未 finish。
        # 【返回点 8】action_budget：上层据此把帧重新排队，下个用户回合凭 checkpoint 续跑
        return finish(TaskExecutionResult(
            task_frame_id=requirement.task_frame_id,
            status="action_budget",
            reply_fragment="当前任务已达到本轮自动执行上限，需要下一轮继续。",
            citations=citations,
            evidence_results=evidence_results,
            capability_results=capability_results,
            artifacts=artifacts,
            task_summary="Harness 达到 action budget。",
            action_count=max_actions,
            error={"code": "ACTION_BUDGET_EXHAUSTED"},
        ))
```

---

## 5. 八个返回点速查

| # | 位置 | 触发条件 | 返回 status | error.code | action_count |
|---|---|---|---|---|---|
| 1 | `:168` | 循环开头已超时 | `failed` | `SOP_STEP_TIMEOUT` | `iteration-1` |
| 2 | `:300` | 动作生成期间超时 | `failed` | `SOP_STEP_TIMEOUT` | `iteration-1` |
| 3 | `:318` | 动作协议两次修复仍失败 / LLM 异常 | `failed` | `HARNESS_ACTION_INVALID`（可恢复，帧重排队） | `iteration` |
| 4 | `:329` | 拿到动作后超时 | `failed` | `SOP_STEP_TIMEOUT` | `iteration` |
| 5 | `:448` | 模型 `finish` 通过两道强制能力守卫 | 由 `_finish_result` 决定（completed/awaiting_user/handoff/failed） | — | `iteration` |
| 6 | `:564` | 工具返回 `data.detached=True` | `waiting_external_task` | — | `iteration` |
| 7 | `:600` | 工具执行后超时 | `failed` | `SOP_STEP_TIMEOUT` | `iteration` |
| 8 | `:649` | 跑满 `max_actions` 未 finish | `action_budget` | `ACTION_BUDGET_EXHAUSTED`（帧重排队续跑） | `max_actions` |

> 另有一个**不经过返回点**的出口：取消。`_raise_if_cancelled` 在 4 个检查点（`:166/:327/:525/:527`）抛出 `HarnessExecutionCancelled`，工具调用里还可能抛 `HarnessExecutionFenced`；二者都不在本函数捕获，直接冒泡给 `HarnessV2Engine` → `AgentLoop.handle_turn` 的取消/栅栏分支。

## 6. 取消与截止检查点分布

- **取消检查（`_raise_if_cancelled`，共 4 处）**：循环顶 `:166`、动作执行前 `:327`、工具调用前 `:525` 与工具调用后 `:527`。
- **截止检查（`_deadline_expired`，共 4 处）**：循环顶 `:167`、动作生成异常分支 `:299`、动作执行前 `:328`、工具执行后 `:599`。
- **硬超时**：每次 LLM 调用前由 `_deadline_llm_client`（`:221`）把模型 timeout 收紧为 `min(模型配置, 剩余截止)`，与上面的软检查互补。
