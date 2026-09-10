# `chat_stream` 逐行注释详解

> 对象：`backend/app/api/chat.py` 的 `chat_stream`（`POST /api/chat/stream`）
>
> 本文只做一件事：把这个函数**逐行拆开讲清楚**，并在关键处配上示例数据。
> 架构层面的总览请看 `docs/chat-stream-api.md`，本文不重复。

---

## 0. 先建立一个心智模型

这个函数体内其实定义了**三个执行单元**，它们不是顺序执行的：

```
chat_stream()  ← HTTP 请求线程，做完校验后立刻返回 StreamingResponse
   │
   ├─ [同步] 校验租户/会话/附件/权限 ……失败直接抛 HTTPException（此时还没开始流）
   │
   ├─ [启动] threading.Thread(run_stream_worker)  ← 后台线程，跑真正的 Agent 逻辑
   │             写 AgentEvent 表
   │                  ↓
   └─ [返回] StreamingResponse(stream_events())  ← 生成器，被 Starlette 逐块拉取
                 轮询 AgentEvent 表 → yield SSE 文本
```

三者之间的通信全靠 4 个共享变量：

| 变量 | 类型 | 作用 |
|------|------|------|
| `relay_ready` | `threading.Event` | worker 告诉 relay「session_id 已确定，可以开始轮询了」 |
| `worker_done` | `threading.Event` | worker 告诉 relay「我干完了（无论成败）」 |
| `source_session_id` | `dict`（可变闭包盒） | 跨线程共享 session_id，新建会话时由 worker 回填 |
| `worker_terminal` | `dict`（可变闭包盒） | 记录是否已产生终端事件，供 `finally` 兜底判断 |

> **为什么用 `dict` 而不是普通变量？**
> Python 闭包里给外层变量赋值需要 `nonlocal`，但 `nonlocal` 无法跨越 `def run_stream_worker` 这种嵌套函数边界写回给另一个嵌套函数看见。用单键 dict 当"可变盒子"是常见的绕过写法——读写的是 dict 内容，不是变量绑定本身。

---

## 1. 函数签名与依赖注入

```python
@router.post("/stream")
def chat_stream(
    request: ChatTurnRequest,        # 请求体，Pydantic 自动解析校验
    current_user: User = Depends(get_current_user),  # 从 Bearer Token 解出当前用户
    db: Session = Depends(get_session),              # 请求级 DB 会话，随请求结束关闭
) -> StreamingResponse:
```

注意 `db` 是**请求级**的：FastAPI 在 handler 返回后就会关掉它。
但我们的 worker 线程和 relay 生成器都活得比 handler 长，
所以它们**都不能用这个 `db`**，必须各自 `with Session(engine)` 新建。这是本函数一个反复出现的关键约束。

**`ChatTurnRequest` 长这样**（`app/session/session_schema.py:213`）：

```python
{
  "tenant_id": "tenant_demo",
  "session_id": "session_a1b2c3d4",      # 可空：为空表示新建会话
  "agent_id": "agent_5f6e7d8c",
  "client_turn_id": "turn_1730governed",  # 前端生成，用于幂等/取消
  "message": "帮我查一下上个月的销售数据",
  "attachments": [],
  "channel": "web",
  "interaction_mode": "normal",           # normal | scheduled_task | team_task | team_tl
  "client_timezone": "Asia/Shanghai",
  "debug": false
}
```

---

## 2. 前置校验段（同步执行，失败直接 4xx）

```python
    _ensure_request_tenant(request.tenant_id, current_user)
```
**租户越权拦截。** 实现只有两行（`chat.py:3371`）：请求体里的 `tenant_id` 必须等于
token 里解出来的 `current_user.tenant_id`，否则 403 `"Tenant mismatch"`。
这是第一道闸门——防止 A 租户用户伪造 `tenant_id` 去读 B 租户的会话。

```python
    request = request.model_copy(
        update={
            "user_id": current_user.id,
            "context_injection": None,
            "message_visibility": "visible",
        }
    )
```
**强制覆写三个"客户端不可信"字段。**
- `user_id`：无论前端传什么，一律以 token 身份为准。
- `context_injection`：这是**服务端专用**的 prompt 前缀（schema 里标了 `exclude=True`）。
  这里显式清空，防止客户端通过请求体注入任意系统提示词——**这是一道 prompt injection 防线**。
- `message_visibility`：强制 `"visible"`。内部重试轮次才会用 `"internal"`，
  客户端不允许自己声明"这条消息对用户不可见"。

`model_copy` 而非原地修改，是因为 Pydantic 模型默认不可变语义更安全。

```python
    request = _validate_chat_turn_attachments(request)
```
**附件校验。** 内部调用 `validate_chat_turn_attachments`，约束两条：
最多 `MAX_CHAT_ATTACHMENTS = 8` 个、单个不超过 `MAX_CHAT_ATTACHMENT_BYTES = 12MB`。
校验失败把 `ValueError` 转成 400。返回的是**新的 request**（附件被规范化过）。

```python
    ensure_tenant(db, request.tenant_id)
```
**确保租户行存在**（不存在则创建）。注意后面 worker 线程里还会再调一次——
因为那是另一个 DB Session，需要在自己的事务上下文里确认。

```python
    team_tl_team_id: str | None = None
```
**预声明团队 TL 标记。** 只存 id 不存 ORM 对象——因为 `Team` 实例绑定在
请求级 `db` 上，handler 返回后该 session 关闭，对象就变成 detached 了，
worker 线程再访问它的属性会抛 `DetachedInstanceError`。**存 id、用时重查**是跨线程传 ORM 数据的正确姿势。

### 2.1 分支：已有会话 vs 新会话

```python
    if request.session_id:
        chat_session = _ensure_chat_session_available(db, request.tenant_id, current_user.id, request.session_id)
```
**会话可达性校验**（`chat.py:2658`）。三重判断：
1. 会话存在，且 `tenant_id` 匹配 → 否则 404
2. `row.user_id != user_id` **且** `not row.team_id` → 404

   即：普通会话只有创建者能访问；**团队会话（`team_id` 非空）对同租户成员开放**。

注意这里故意返回 404 而不是 403——不泄露"这个 session id 是否存在"。

```python
        _ensure_team_session_human_writable(chat_session)
```
**团队会话写入闸门**（`chat.py:2670`）。逻辑：
```python
if chat_session.team_id and "TL 对话" not in (chat_session.title or ""):
    raise HTTPException(403, "Team execution sessions are read-only")
```
团队内部的任务执行/竞标/验收会话由唤醒机制自主驱动，
人类直接插话会污染任务历史并绕过 Agent 权限校验，所以**只读**。
只有标题含「TL 对话」的团队会话才允许人发言。

```python
        request = _bind_request_to_session_agent(db, request, chat_session, current_user)
```
**会话-员工绑定**（`chat.py:2639`）。两种情况：
- 会话**已绑定** agent：若请求带了不同的 `agent_id` → 409 `"Session is already bound to another agent"`；
  否则以会话上的为准覆写 request。
- 会话**未绑定**：校验请求里的 agent 可用（active、非 overall、对当前用户可见），
  然后**写库落定绑定**并 commit。

> 换句话说：一个会话的员工身份是**一次性锁定**的，不能中途换人。

```python
        team_tl_team = _team_tl_session_team(db, chat_session)
        team_tl_team_id = team_tl_team.id if team_tl_team is not None else None
```
**判定是否为"人对 TL 聊天"会话**（`chat.py:2680`）。四个条件全满足才返回 Team：
1. `chat_session.team_id` 和 `agent_id` 都非空
2. 标题含「TL 对话」（与上面的可写判据一致）
3. Team 存在、同租户、`status == "active"`
4. `get_team_leader(db, team.id).agent_id == chat_session.agent_id`（当前绑定的就是现任 TL）

同样只留下 `id`，理由见上面的 detached 说明。

```python
    else:
        _ensure_chat_agent_available(db, request.tenant_id, request.agent_id, current_user)
```
**新会话路径**：没有 session 可校验，只校验 agent（`chat.py:2622`）。
拒绝条件：`agent_id` 为空 → 400；不存在 / 跨租户 / 非 active / `is_overall` → 404；
对当前用户不可见（非管理员、非所有者、且未发布到 gallery）→ 403。

> **注意这里不创建会话。** 会话是由 worker 线程里的 `get_or_create_harness_session` 创建的，
> 所以 relay 一开始并不知道 session_id —— 这正是 `relay_ready` 这个 Event 存在的原因。

```python
    if not request.message.strip() and not request.attachments:
        raise HTTPException(status_code=400, detail="Message cannot be empty")
```
**空消息拦截。** 注意是 `and`：**纯附件无文字**是合法的（用户拖个文件进来说"看看这个"也可以不打字）。

```python
    original_message = request.message
```
**快照用户原文。** 下一步可能会把 `message` 换成注入了团队上下文的版本，
但后处理 `process_tl_reply` 需要拿到**用户真正说的那句话**，所以先存一份。

### 2.2 团队 TL 上下文注入

```python
    if team_tl_team_id is not None:
        request = request.model_copy(
            update={
                "context_injection": build_tl_chat_context(
                    db, db.get(Team, team_tl_team_id), original_message
                ),
                "interaction_mode": "team_tl",
            }
        )
```
只对 TL 会话生效。`build_tl_chat_context` 会拼出一段包含
**团队花名册 / 未闭环任务 / 共享黑板 / 派任务格式说明**的上下文，塞进 `context_injection`。

注意这里用 `db.get(Team, team_tl_team_id)` 重新查了一次——
因为上面只留了 id。这次查询发生在**请求线程**、`db` 还活着，所以安全。

**`context_injection` 的关键性质**：schema 里标了 `exclude=True`，意味着
它**不会**被序列化进用户可见的消息、也不会进后台任务的 payload。
它只在本次运行时被 prompt 组装消费。这就是为什么第 2 节要先把它清空——
清空是为了防注入，这里重新填是服务端自己的可信写入。

---

## 3. 跨线程共享状态的初始化

```python
    relay_ready = threading.Event()
    worker_done = threading.Event()
    source_session_id = {"value": request.session_id or ""}
    worker_terminal = {"seen": False}
```
四个共享变量，语义见第 0 节的表格。

```python
    initial_cursor = _latest_event_cursor(db, request.tenant_id, request.session_id) if request.session_id else None
```
**这一行至关重要——它决定了"从哪里开始播"。**

`_latest_event_cursor` 取该会话**当前最新一条**事件的 `(created_at, id)`：

```python
# 示例返回
(datetime(2026, 9, 9, 3, 14, 15, 926535), "evt_8f3a1c7d90e2b456")
```

relay 会**只播这个游标之后**的事件。如果不做这一步，
新开一个流会把该会话**历史上所有事件**重播一遍，前端就会看到旧对话重新"打字"一遍。

新会话时为 `None`，表示"从头播"（此时表里本来也没有该 session 的事件）。

```python
    def set_source_session(session_id: str) -> None:
        if not session_id:
            return
        source_session_id["value"] = session_id
        relay_ready.set()
```
**回填 + 放行。** worker 一旦知道了真实 session_id（新建会话时），
就调这个函数：写进共享盒子，并 `set()` 让阻塞中的 relay 醒过来。
空串直接忽略——避免把 relay 放行到一个无效 session 上空转。

```python
    if source_session_id["value"]:
        relay_ready.set()
```
**已有会话的快捷路径。** session_id 请求里就带了，不用等 worker，直接放行。

---

## 4. Worker 线程：`run_stream_worker`

### 4.1 Span sink 挂载

```python
    def run_stream_worker() -> None:
        span_sink_token = None
        try:
            with Session(engine) as worker_db:
```
**worker 自己的 DB Session。** 前面说过，绝不能用外层的 `db`。
`with` 保证线程结束时连接归还池子。

```python
                span_turn_id = {"value": ""}

                def persist_span(event_type: str, payload: dict[str, object]) -> None:
                    session_id = source_session_id["value"] or request.session_id or ""
                    if not session_id:
                        return
```
**LLM 调用追踪回调。** 又一个可变盒子 `span_turn_id`——
span 事件在 turn_id 确定**之前**就可能触发（比如规划阶段的 LLM 调用），
所以要等 `user_message_received` 到达后回填。session_id 还没定就直接丢弃该 span。

```python
                    turn_id = span_turn_id["value"]
                    event_payload = dict(payload)
                    if turn_id:
                        event_payload.setdefault("turn_id", turn_id)
                        event_payload.setdefault("user_message_id", turn_id)
                    if request.client_turn_id:
                        event_payload.setdefault("client_turn_id", request.client_turn_id)
```
**给 span 贴上归属标签。** 用 `setdefault` 而非直接赋值——
如果 payload 里本来就有 turn_id（更精确的来源），不覆盖它。

```python
                    _persist_relay_only_event(
                        worker_db, request.tenant_id, session_id, event_type, event_payload,
                    )

                span_sink_token = set_span_sink(persist_span)
```
`set_span_sink` 基于 **ContextVar**（`app/observability/spans.py:50`），
所以只对**当前线程/上下文**生效，不会污染其他并发请求。返回的 token 用于 `finally` 里精确复位。

> **⚠️ 一个容易误解的点：span 事件写进了库，但前端收不到。**
> `_events_after_cursor` 里有 `AgentEvent.event_type.notin_(SPAN_EVENT_TYPES)`，
> 而 `SPAN_EVENT_TYPES` 正好包含 `llm_call_started` / `llm_call_finished` /
> `llm_call_failed` / `knowledge_span_*`。
> 也就是说 **span 是纯落库的可观测性数据，被 relay 显式过滤掉了**，
> 只供事后追溯和 Debug 页面查询，不参与 SSE 推送。

```python
                ensure_tenant(worker_db, request.tenant_id)
```
在 worker 自己的事务里再确认一次租户存在。

### 4.2 定时任务快速路径

```python
                if request.session_id:
                    chat_session = _ensure_chat_session_available(
                        worker_db, request.tenant_id, request.user_id, request.session_id,
                    )
```
在 worker 的 session 里**重新加载** ChatSession（外层那个已经属于别的 Session 了）。
顺便二次校验——防御 TOCTOU（校验后、执行前会话被删）。

```python
                    if request.interaction_mode == "scheduled_task":
                        _persist_relay_only_event(
                            worker_db, request.tenant_id, chat_session.id, "stream_status",
                            {"phase": "scheduled_task_intent", "text": "识别定时任务需求"},
                        )
                        _persist_relay_only_event(
                            worker_db, request.tenant_id, chat_session.id, "stream_status",
                            {"phase": "scheduled_task_parse", "text": "解析执行计划"},
                        )
```
**先给前端两条"正在做什么"的状态提示。**
这两条会立刻被 relay 捞到并推给前端，用户马上看到进度，不用干等。

前端收到的 SSE（`stream_status` 经 `STREAM_RELAY_EVENT_ALIASES` 别名为 `status`）：
```
id: evt_3c9f2a...
event: status
data: {"kind":"status","sessionId":"session_a1b2","timestamp":"2026-09-09T03:14:15.926535",
       "provider":"skill","phase":"scheduled_task_intent","text":"识别定时任务需求"}
```

```python
                    scheduled_response = _maybe_handle_scheduled_task_request(worker_db, request, chat_session)
                    if scheduled_response:
                        response, draft = scheduled_response
                        set_source_session(response.session_id)
```
**尝试走定时任务捷径。** `_maybe_handle_scheduled_task_request`（`chat.py:614`）
在满足以下条件时返回结果，否则返回 `None` 落到通用路径：
- `interaction_mode == "scheduled_task"` 且有 `agent_id`
- **本轮未被取消**（取消优先级高于捷径，让正常 Harness 路径去终结这一轮）
- `detect_scheduled_task_draft` 识别出了 `should_create` 的草案

命中后它内部已经完成了：claim turn → 落用户消息 → 落 assistant 消息 → 写事件 → complete turn。
**完全不经过 LLM Agent Loop**，所以极快。

```python
                        message_id, client_turn_id = _resolve_turn_ids_from_events(
                            worker_db, request.tenant_id, response.session_id, request.client_turn_id or "",
                        )
                        turn_payload = {
                            "turn_id": message_id,
                            "user_message_id": message_id,
                            "client_turn_id": client_turn_id or None,
                        }
```
**把前端的 `client_turn_id` 翻译成服务端的 `message_id`。**
`_resolve_turn_ids_from_events`（`chat.py:1777`）倒序扫 `user_message_received` 事件，
找到匹配的那条，返回 `(message_id, client_turn_id)` 二元组。找不到就原样返回。

示例：
```python
# 输入 requested_turn_id = "turn_1730abc"
# 匹配到的事件 payload:
#   {"message_id": "msg_7e2f9d1a", "client_turn_id": "turn_1730abc", ...}
# 返回:
("msg_7e2f9d1a", "turn_1730abc")

# turn_payload 结果:
{"turn_id": "msg_7e2f9d1a", "user_message_id": "msg_7e2f9d1a", "client_turn_id": "turn_1730abc"}
```

**为什么每个事件都要带这三个 id？** 前端是按 turn 分组渲染的——
一条用户消息下面挂着它的状态、增量、最终回复。没有 turn_id 就没法归组，
多轮并发时消息会串位。

```python
                        _persist_relay_only_event(
                            worker_db, request.tenant_id, response.session_id, "stream_status",
                            {
                                "phase": "scheduled_task_draft",
                                "text": "生成定时任务草案",
                                **draft.model_dump(mode="json"),
                                **turn_payload,
                            },
                        )
                        _persist_relay_only_event(
                            worker_db, request.tenant_id, response.session_id,
                            "scheduled_task_draft",
                            {**draft.model_dump(mode="json"), **turn_payload},
                        )
```
**同一份草案数据发两遍，但用途不同：**
- `stream_status`：给"执行轨迹"面板显示一行进度
- `scheduled_task_draft`：给聊天区渲染**可确认的任务卡片**

`draft.model_dump(mode="json")` 示例：
```json
{
  "should_create": true,
  "title": "每日销售日报",
  "prompt": "汇总昨日销售数据并生成简报",
  "schedule_type": "daily",
  "schedule": {"time": "09:00"}
}
```

```python
                        for chunk in _reply_chunks(response.reply):
                            _persist_relay_only_event(
                                worker_db, request.tenant_id, response.session_id,
                                "stream_delta", {"content": chunk, **turn_payload},
                            )
```
**把完整回复切片成"打字机效果"。** `_reply_chunks`（`chat.py:925`）按
`STREAM_REPLY_CHUNK_SIZE = 96` 字符硬切：

```python
reply = "我已按你选择的定时项目整理成自动任务草案。\n任务：每日销售日报\n计划：每天 09:00\n..."
# → 第 1 片: reply[0:96]
# → 第 2 片: reply[96:192]
# → ...
```

> 注意这是**伪流式**：内容早就全部生成好了，只是切开分批推送，制造逐字输出的观感。

```python
                        _persist_relay_only_event(..., "stream_end", turn_payload)
                        _persist_relay_only_event(..., "complete",
                            {**response.model_dump(mode="json"), **turn_payload})
                        worker_terminal["seen"] = True
```
`stream_end` 告诉前端"增量推完了"，`complete` 携带完整的 `ChatTurnResponse`
（含 `reply` 全文、`session_state`）作为权威结果，前端可用它校正拼接结果。

`worker_terminal["seen"] = True` 是给 `finally` 看的：**已产生终端事件，不需要兜底补 interrupted**。

```python
                        _schedule_session_title_summary(
                            request.tenant_id, request.user_id, response.session_id, request.agent_id,
                        )
                        return
```
**异步生成会话标题**（又开一个 daemon 线程），然后 `return` 结束 worker——
定时任务路径到此完结，不走下面的 Agent Loop。

`_schedule_session_title_summary`（`chat.py:243`）内部有个进程级去重集合
`_session_title_summary_jobs` + 锁，保证同一会话只有一个标题生成任务在跑。

### 4.3 通用路径：驱动 AgentLoop

```python
                for item in AgentLoop(worker_db).handle_turn_stream(request):
                    event_name = str(item["event"])
                    data = item["data"] if isinstance(item.get("data"), dict) else {}
```
**核心循环。** `handle_turn_stream` 是个生成器，yield 出的每个 item 形如：
```python
{
  "event": "stream_delta",
  "data": {
    "kind": "stream_delta",
    "sessionId": "session_a1b2c3d4",
    "timestamp": "2026-09-09T03:14:16.102938",
    "provider": "skill",
    "content": "根据数据库查询结果，上个月总销售额为 ",
    "turn_id": "msg_7e2f9d1a",
    "user_message_id": "msg_7e2f9d1a",
    "execution_engine": "harness_v2"
  }
}
```

`data` 做了 `isinstance` 防御——万一不是 dict 就退化成空字典，避免下面 `.get()` 崩掉。

```python
                    item_session_id = str(data.get("sessionId") or request.session_id or source_session_id["value"] or "")
                    if item_session_id:
                        set_source_session(item_session_id)
```
**三级兜底取 session_id**，取到就回填并放行 relay。
新建会话场景下，这里就是 relay 被解除阻塞的那一刻。

```python
                    if event_name == "session_created" and item_session_id:
                        _persist_relay_only_event(worker_db, request.tenant_id, item_session_id, event_name, data)
                    elif event_name == "complete" and item_session_id:
                        _persist_relay_only_event(worker_db, request.tenant_id, item_session_id, event_name, data)
                        worker_terminal["seen"] = True
                    elif event_name in {"stream_cancelled", "stream_interrupted", "error", "error_occurred"}:
                        worker_terminal["seen"] = True
```
**只有 `session_created` 和 `complete` 由这里补写库。**

其余事件（`stream_delta` / `stream_end` / `status` / `tool_result` …）
**在 `AgentLoop._stream_event` 内部就已经写库了**（`agent_loop.py:485`，
通过 `persisted_stream_events` 白名单 + `self.events.record`）。
这里再写会重复，所以不写。

取消/中断/错误类事件同理——由引擎层落库，这里只需**标记终端状态**。

```python
                    if item["event"] == "user_message_received":
                        event_source_session_id = str(item["data"].get("sessionId") or request.session_id or "")
                        set_source_session(event_source_session_id)
                        span_turn_id["value"] = str(
                            data.get("turn_id") or data.get("user_message_id") or data.get("message_id") or ""
                        )
```
**回填 span 的 turn_id。** 从这一刻起，后续所有 LLM 调用的 span
都会自动带上正确的 turn 归属（还记得 4.1 里那个空着的 `span_turn_id` 盒子吗）。

```python
                        _schedule_session_title_summary(
                            request.tenant_id, request.user_id, event_source_session_id, request.agent_id,
                        )
                        continue
```
**用户消息一落库就启动标题生成**，不等回复完成——这样用户切走再回来，
侧边栏已经有标题了。`continue` 跳过后面的 complete 处理。

### 4.4 `complete` 事件的后处理

```python
                    if item["event"] == "complete":
                        event_source_session_id = str(item["data"].get("sessionId") or request.session_id or "")
                        _schedule_session_title_summary(...)
```
再次触发标题生成——**这是兜底**。第一次触发时可能助手还没回复，
`_summarize_session_title_once` 内部会重试 8 次（每次 sleep 0.25s）等消息落库；
如果那时仍未就绪，这次 complete 后的调用能补上。去重集合保证不会重复跑。

```python
                        if team_tl_team_id is not None:
                            try:
                                tl_team = worker_db.get(Team, team_tl_team_id)
                                tl_session = worker_db.get(ChatSession, event_source_session_id or request.session_id or "")
                                tl_user = worker_db.get(User, request.user_id) if request.user_id else None
                                if tl_team is not None and tl_session is not None and tl_user is not None:
                                    process_tl_reply(
                                        worker_db,
                                        team=tl_team, session=tl_session, user=tl_user,
                                        user_message=original_message,   # ← 第 2 节存的原文
                                        reply=str(data.get("reply") or ""),
                                        client_turn_id=request.client_turn_id,
                                    )
                            except Exception:
                                logger.exception("team TL reply post-processing failed")
```
**TL 派任务后处理。** 三个 ORM 对象都在 **worker_db 里重新加载**（不是复用外层的）。

`process_tl_reply` 会解析 TL 回复里的派任务块，为团队成员创建任务。

**注意这个 `try/except Exception` + 只记日志**：这是刻意的降级设计——
派任务失败不应该让用户看不到 TL 的回复。用户体验优先，失败留日志人工排查。

```python
                        if event_source_session_id:
                            summary_payload = _session_title_summary_payload(worker_db, request.tenant_id, event_source_session_id)
                            if summary_payload:
                                _persist_relay_only_event(
                                    worker_db, request.tenant_id, event_source_session_id,
                                    SESSION_TITLE_SUMMARY_EVENT, summary_payload,
                                )
```
**如果标题已经生成好了，顺手推给前端**，让侧边栏立即更新，不用等下次刷新。

`summary_payload` 示例：`{"sessionId": "session_a1b2c3d4", "title": "上月销售数据查询"}`

它是从 `AgentEvent` 表里查 `session_title_summarized` 事件读出来的——
也就是说，异步线程写、这里读，如果异步还没写完就查不到，那就跳过（下次刷新自然会有）。

```python
                        if request.interaction_mode != "scheduled_task" or not request.agent_id:
                            continue
                        draft = detect_scheduled_task_draft(
                            worker_db, request.tenant_id, request.agent_id, request.user_id,
                            request.message, event_source_session_id or None, request.client_timezone,
                        )
                        if draft and draft.should_create:
                            _persist_scheduled_task_draft(worker_db, request.tenant_id, event_source_session_id, draft)
                            _persist_relay_only_event(
                                worker_db, request.tenant_id, event_source_session_id,
                                "scheduled_task_draft", draft.model_dump(mode="json"),
                            )
```
**定时任务的第二次机会。** 4.2 的快速路径没命中（比如需要 Agent 先澄清需求），
但走完完整对话后可能就能识别出草案了，这里再检测一次并推送卡片。

### 4.5 异常处理三连

```python
        except Exception as exc:
            logger.exception("chat stream worker failed")
            session_id = source_session_id["value"] or request.session_id or ""
            if session_id:
                with Session(engine) as error_db:
```
**又开一个全新 Session。** 因为异常很可能就是 `worker_db` 的事务炸了
（比如约束冲突导致事务进入 aborted 状态），此时用它写任何东西都会继续失败。
**错误处理路径必须用干净的连接。**

```python
                    chat_session = error_db.get(ChatSession, session_id)
                    if chat_session:
                        _persist_chat_turn_interrupted(
                            error_db, request.tenant_id, chat_session,
                            request.client_turn_id or "",
                            str(exc) or "stream worker failed",
                            error_details={
                                "error_type": exc.__class__.__name__,
                                "error_traceback": traceback.format_exc()[-STREAM_INTERRUPTED_TRACEBACK_CHAR_LIMIT:],
                            },
                        )
```
`_persist_chat_turn_interrupted`（`chat.py:1726`）做三件事：
1. 先查 `_turn_has_terminal_event` —— **已有终端事件就直接返回 False，不重复写**
2. 写一条 `stream_interrupted` 事件
3. 补一条内容为 `"本次响应中断，请重试发送。"` 的 assistant 消息（同样带去重检查）

`traceback` 用 `[-6000:]` 截尾——保留**最内层**的调用栈（最接近报错点的那部分），
同时防止超大 traceback 撑爆 JSON 字段。

```python
        except BaseException as exc:
            ...  # 结构与上面几乎相同
            if isinstance(exc, (KeyboardInterrupt, SystemExit)):
                raise
```
**为什么要单独接 `BaseException`？**
`KeyboardInterrupt`、`SystemExit`、`GeneratorExit` 都不继承 `Exception`。
服务重启/优雅关闭时会抛这些，我们希望**先把中断状态落库**（否则这一轮就永远悬着），
然后对真正的退出信号 `raise` 让它继续传播，不阻碍进程退出。

### 4.6 `finally` 兜底

```python
        finally:
            if span_sink_token is not None:
                reset_span_sink(span_sink_token)
```
**复位 ContextVar**，防止 sink 泄漏到线程池复用的下一个任务。

```python
            session_id = source_session_id["value"] or request.session_id or ""
            if session_id and not worker_terminal["seen"]:
                with Session(engine) as final_db:
                    chat_session = final_db.get(ChatSession, session_id)
                    if chat_session:
                        changed = _persist_chat_turn_interrupted(
                            final_db, request.tenant_id, chat_session,
                            request.client_turn_id or "",
                            "stream worker ended before terminal event",
                        )
                        if changed:
                            final_db.commit()
                            set_source_session(session_id)
```
**最后一道保险。** 覆盖那些"既没正常完成、也没抛异常"的诡异路径
（比如生成器提前 return、某个分支忘了标记终端）。

这保证了一条硬不变量：**每一轮对话最终必然有一个终端事件**。
否则 relay 会一直等到 660 秒超时，用户界面就卡在"生成中"转圈。

```python
            worker_done.set()
```
**通知 relay 可以收尾了。** 放在 `finally` 的最后一行——
无论前面走哪条路径，这个信号一定会发出。少了它，relay 会空转到超时。

```python
    threading.Thread(target=run_stream_worker, daemon=True).start()
```
**启动 worker。** `daemon=True` 意味着主进程退出时不等它——
配合上面的 `BaseException` 处理，中断状态已经落库了，可以安全丢弃线程。

---

## 5. Relay 生成器：`stream_events`

```python
    def stream_events() -> Iterator[str]:
        nonlocal initial_cursor
```
`nonlocal` 是必需的：下面要**赋值**给 `initial_cursor`（推进游标），
不声明的话 Python 会把它当成新的局部变量，游标永远不前进 → 无限重播同一批事件。

```python
        relay_ready.wait(15)
```
**最多等 15 秒**拿 session_id。超时也继续往下走（不抛异常）——
此时 `source_session_id["value"]` 是空串，循环里 `if session_id:` 不成立，
就只发心跳，直到 worker 设置了 session_id 或 `worker_done`。

```python
        deadline = time.monotonic() + STREAM_RELAY_IDLE_TIMEOUT_SECONDS   # 660 秒
        last_heartbeat_at = time.monotonic()
        terminal_sent = False
        internal_relay_turn_ids: set[str] = set()
```
用 `time.monotonic()` 而非 `time.time()`——**单调时钟不受系统校时影响**，
NTP 回拨不会让超时逻辑错乱。

`internal_relay_turn_ids` 收集所有"内部轮次"的 id，用于过滤。

### 5.1 主轮询循环

```python
        while True:
            session_id = source_session_id["value"]
            emitted = False
            if session_id:
                with Session(engine) as relay_db:
                    rows = _events_after_cursor(relay_db, request.tenant_id, session_id, initial_cursor)
```
**每轮都开关一次 Session。** 看起来浪费，但连接池让开销很小，
而换来的是：**每次都读到最新提交的数据**，不会被长事务的快照隔离挡住 worker 的新写入。

`with` 块在 `for` 循环外结束——**先把行全部取出来再处理**，
避免在 yield（可能阻塞很久，取决于客户端消费速度）期间占着数据库连接。

`_events_after_cursor`（`chat.py:1939`）的 SQL 大致是：
```sql
SELECT * FROM agent_events
WHERE tenant_id = ? AND session_id = ?
  AND event_type NOT IN ('llm_call_started','llm_call_finished','llm_call_failed',
                         'knowledge_span_started','knowledge_span_finished','knowledge_span_failed')
  AND (created_at > ? OR (created_at = ? AND id > ?))
ORDER BY created_at, id
LIMIT 200
```

```python
                for row in rows:
                    payload = row.payload_json or {}
                    row_turn_ids = {
                        str(payload.get(key) or "").strip()
                        for key in ("turn_id", "user_message_id", "message_id", "client_turn_id")
                        if str(payload.get(key) or "").strip()
                    }
```
**收集这一行涉及的所有 turn 标识。** 四个 key 都查是因为不同事件类型
用的字段名不统一（历史原因），这里做归一化。集合推导顺便去重、去空。

```python
                    if payload.get("message_visibility") == "internal":
                        internal_relay_turn_ids.update(row_turn_ids)
```
**发现内部轮次就登记。** 内部轮次是系统自动重试产生的，
可审计但不该出现在用户界面上。

```python
                    event_name, data = _relay_event_payload(row)
                    initial_cursor = (row.created_at, row.id)
                    emitted = True
```
**注意这两行的位置：游标推进和 `emitted` 标记在过滤判断之前。**
这是对的——即便这行要被跳过，游标也必须前进，否则下轮又会查到它，
陷入死循环。`emitted = True` 同理，表示"确实有新数据"，应该重置超时。

`_relay_event_payload`（`chat.py:1926`）做两件事：
1. 应用别名：`stream_status` → `status`，`router_decision_created` → `router_decision`
2. 包装信封：注入 `kind` / `sessionId` / `timestamp` / `provider`，再展开原 payload

```python
                    if row_turn_ids & internal_relay_turn_ids:
                        continue
```
**集合求交做过滤。** 只要这行事件涉及任何一个已知的内部 turn，整行丢弃。

> **⚠️ 这个过滤有个固有的时序限制：**它是"发现即生效"的。
> 如果某个内部轮次的第一条事件不带 `message_visibility: internal`（比如 status 先到、
> 带标记的 assistant 消息后到），那么在标记到达之前发出的事件已经推给前端了，收不回来。
> 实践中引擎会尽早打标，但这不是一个强保证。

```python
                    yield _sse(event_name, data, row.id)
                    if event_name in STREAM_RELAY_TERMINAL_EVENTS:
                        terminal_sent = True
```
**吐出 SSE 文本。** `_sse`（`chat.py:1973`）生成：
```
id: evt_8f3a1c7d90e2b456
event: stream_delta
data: {"kind":"stream_delta","sessionId":"session_a1b2c3d4","timestamp":"2026-09-09T03:14:16.102938","provider":"skill","content":"根据数据库查询结果，上个月总销售额为 ","turn_id":"msg_7e2f9d1a"}

```
（末尾是两个 `\n`，SSE 协议用空行分隔事件块）

`json.dumps(..., ensure_ascii=False)` 保留中文原文，不转成 `\uXXXX`，
省流量也便于调试时肉眼阅读。

`id:` 字段用 AgentEvent 主键——标准 SSE 的 `Last-Event-ID` 断线重连机制可以用上它
（当前后端还没实现该 header 的处理，但字段已经预留了）。

```python
                if emitted:
                    deadline = time.monotonic() + STREAM_RELAY_IDLE_TIMEOUT_SECONDS
                    last_heartbeat_at = time.monotonic()
```
**有数据就续命。** 660 秒是**空闲**超时，不是总时长上限——
只要事件持续产生，流可以跑任意久（长任务 Agent 可能跑几十分钟）。

同时重置心跳计时——刚发过真实数据，没必要紧接着再发心跳。

### 5.2 退出条件

```python
            if terminal_sent and worker_done.is_set() and not emitted:
                return
            if worker_done.is_set() and not emitted:
                return
```
**两个条件其实等价**（第二个完全覆盖第一个）——
第一个是显式表达"正常收尾"的意图，第二个是兜底。留着无害，但确实冗余。

关键是 `not emitted` 这个条件：**worker 完成了也不能立刻退，
必须再空跑一轮确认表里没有残留事件**。
否则 worker 最后写的那几条（比如 `complete`）可能还没被 relay 读到就断流了。

```python
            if time.monotonic() > deadline:
                if session_id:
                    with Session(engine) as timeout_db:
                        chat_session = timeout_db.get(ChatSession, session_id)
                        if chat_session:
                            _persist_chat_turn_interrupted(
                                timeout_db, request.tenant_id, chat_session,
                                request.client_turn_id or "",
                                "stream relay timed out waiting for terminal event",
                            )
                            timeout_db.commit()
                    continue
                return
```
**空闲超时处理。** 写一条 `stream_interrupted` 落库，然后 `continue`——
下一轮就能把这条事件读出来推给前端，让界面从"生成中"变成"已中断"，
接着 `terminal_sent` 置位、正常收尾。

> **⚠️ 这里有个值得留意的边界：`continue` 跳过了末尾的 `time.sleep`。**
> 如果 `_persist_chat_turn_interrupted` 返回 False（该轮已有终端事件，不重复写），
> 就不会产生新事件 → 下一轮 `emitted` 仍为 False → `deadline` 不会被重置 →
> 再次进入这个分支 → 又 `continue`。
> 只要 worker 线程还没结束（`worker_done` 未置位，前面的 return 不生效），
> 这会变成一个**无 sleep 的忙循环**，空转烧 CPU。
>
> 触发条件比较苛刻（需要 660 秒无事件 + 该轮已终结 + worker 仍在跑），
> 实践中罕见，但从代码结构上看这个路径是存在的。修法很简单：
> 把 `continue` 改成 `time.sleep(STREAM_RELAY_POLL_SECONDS); continue`，
> 或在写入后无论成败都重置一次 `deadline`。

`session_id` 为空的超时（等了 15 秒 + 660 秒 worker 还没给出 session）直接 `return`。

### 5.3 心跳与节流

```python
            now = time.monotonic()
            if now - last_heartbeat_at >= STREAM_RELAY_HEARTBEAT_SECONDS:   # 5 秒
                last_heartbeat_at = now
                yield _sse(
                    "heartbeat",
                    {"phase": "relay", "sessionId": session_id or request.session_id or ""},
                )
```
**每 5 秒一次心跳。** 目的有三个：
1. 防止 Nginx / 云负载均衡的空闲连接超时（常见默认值 60s）把连接掐掉
2. 让客户端能感知服务端还活着
3. 让 `yield` 有机会抛出 `GeneratorExit` —— **这是检测客户端断开的唯一途径**。
   如果一直不 yield，服务端不会知道对方已经关了页面。

心跳事件**不带 `id:`**（`_sse` 的第三个参数没传），因为它不对应任何 AgentEvent 行，
不应该干扰 `Last-Event-ID` 的语义。

```python
            time.sleep(STREAM_RELAY_POLL_SECONDS)   # 0.08 秒
```
**节流。** 80ms 是延迟和数据库压力的折中：
- 太短（如 10ms）→ 每秒 100 次查询 × 并发会话数，DB 扛不住
- 太长（如 500ms）→ 打字机效果变成一顿一顿的，观感差

80ms 大约每秒 12 次查询，人眼感知上已经足够连续。

```python
    return StreamingResponse(stream_events(), media_type="text/event-stream")
```
**返回流式响应。** 注意此刻 `stream_events()` 只是**创建了生成器对象，还没执行任何一行**——
生成器是惰性的。真正的执行发生在 Starlette 开始迭代它的时候，那时 HTTP 响应头已经发出去了。

这带来一个重要后果：**一旦返回了 `StreamingResponse`，就无法再返回 HTTP 错误码了。**
所以所有可能失败的校验都必须放在第 2 节那段同步代码里——
那时候还能干净地抛 `HTTPException`。这就是为什么前置校验写得那么长、那么靠前。

---

## 6. 完整时序示例

以"用户在已有会话里发一条消息"为例：

```
t=0ms      HTTP 线程：校验通过，initial_cursor = (T0, evt_old)
t=1ms      HTTP 线程：relay_ready.set()（session_id 已知）
t=2ms      HTTP 线程：启动 worker 线程，返回 StreamingResponse
t=3ms      relay：wait(15) 立即返回，进入轮询
t=3ms      relay：查询 → 无新事件 → sleep(80ms)

t=10ms     worker：AgentLoop 开始，yield user_message_received
           └─ 引擎内部已落库 evt_001
t=12ms     worker：yield status "正在规划本轮任务"
           └─ 引擎内部落库 evt_002

t=83ms     relay：查到 evt_001, evt_002
           ├─ yield  event: user_message_received
           ├─ yield  event: status
           ├─ initial_cursor 推进到 (T2, evt_002)
           └─ deadline 重置为 now + 660s

t=90ms     worker：调用 LLM（span 落库 evt_003，但 relay 会过滤掉）
t=2100ms   worker：LLM 返回，切片推送
           └─ 落库 evt_004..evt_012（stream_delta × 9）

t=2160ms   relay：查到 evt_004..evt_012 → 逐条 yield stream_delta
                （evt_003 因在 SPAN_EVENT_TYPES 中被 SQL 过滤，不会出现）

t=2200ms   worker：yield stream_end（落库）→ yield complete（worker 显式落库）
           └─ worker_terminal["seen"] = True
t=2205ms   worker：_schedule_session_title_summary()（另起线程）
t=2210ms   worker：finally → worker_terminal 为 True，不补 interrupted
           └─ worker_done.set()

t=2240ms   relay：查到 stream_end + complete → yield → terminal_sent = True
t=2320ms   relay：查询无新事件（emitted=False）+ worker_done 已置位 → return
           └─ SSE 连接正常关闭
```

---

## 7. 这段代码里反复出现的几个设计约定

把上面散落的点收拢一下，这些是读懂/修改这段代码必须记住的：

1. **三个执行单元、三套 DB Session。**
   请求线程用 `db`（依赖注入，随请求结束）；worker 用 `worker_db`；
   relay 每轮新建。错误处理路径**再额外新建**，因为原事务可能已经 aborted。

2. **跨线程只传 id，不传 ORM 对象。**
   `team_tl_team_id` 而非 `team_tl_team`，用的时候在目标 Session 里重查。

3. **可变 dict 当共享盒子。**
   `source_session_id` / `worker_terminal` / `span_turn_id` 都是这个模式，
   因为闭包无法跨兄弟嵌套函数写回变量绑定。

4. **游标推进必须在过滤之前。**
   否则被跳过的行会被反复查出来，死循环。

5. **终端事件是硬不变量。**
   `worker_terminal` 标记 + `finally` 兜底 + relay 超时补写，三层保证
   每一轮必然收敛，界面不会永久卡在"生成中"。

6. **span 写库但不推送。**
   `SPAN_EVENT_TYPES` 在 SQL 层被 `notin_` 过滤掉，属于纯可观测性数据。

7. **所有可能失败的校验都在返回 StreamingResponse 之前。**
   一旦开始流式输出，就只能通过事件体传递错误，没法再给 HTTP 状态码了。

8. **后处理失败只记日志。**
   TL 派任务、标题生成这类增强功能失败，绝不能影响用户看到回复。
