# /api/chat/stream 路由实现分析

## 1. 路由入口

**文件**: `backend/app/api/chat.py:1080`

```python
@router.post("/stream")
def chat_stream(
    request: ChatTurnRequest,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_session),
) -> StreamingResponse:
```

返回类型是 **SSE (Server-Sent Events)** 流式响应（`text/event-stream`）。

---

## 2. 整体架构：双线程中继模式

这是该路由最核心的设计——**Worker 线程 + Relay 生成器**通过数据库 `AgentEvent` 表解耦：

```
                    写入事件                    轮询事件
┌─────────────┐   ───────────► ┌──────────┐  ───────────►  ┌──────────────┐
│  Worker线程  │                │ AgentEvent│                │  SSE 输出流   │
│ handle_turn │                │   表       │                │ stream_events│
└─────────────┘                └──────────┘                └──────────────┘
```

- **Worker 线程**：异步执行 `AgentLoop.handle_turn_stream()`，把所有事件写入 `AgentEvent` 表
- **Relay 生成器**（主线程）：轮询 `AgentEvent` 表，把新事件以 SSE 格式推给客户端

### 为什么用数据库做中继？

- 解耦生产和消费，支持长连接中断重连（通过 cursor 续传）
- 事件持久化，可追溯
- Worker 是独立线程，即使 HTTP 连接断开，任务仍可继续执行

---

## 3. 执行流程详解

### 3.1 前置校验（`chat.py:1086-1117`）

- 校验租户身份、用户权限
- 校验附件（数量 ≤8、单文件 ≤12MB）
- 确保 session / agent 可用
- 团队 TL 会话注入团队上下文（`build_tl_chat_context`）

### 3.2 Worker 线程启动（`chat.py:1134-1397`）

`run_stream_worker()` 在独立线程中运行，逻辑如下：

1. **设置 span sink**：将 LLM 调用追踪也写入事件表
2. **定时任务快速路径**：如果是 `scheduled_task` 模式且命中草案，直接生成 `stream_delta` → `stream_end` → `complete` 事件，不走 AgentLoop
3. **主路径**：遍历 `AgentLoop.handle_turn_stream()` 的迭代器：
   - `session_created`：新会话创建
   - `user_message_received`：用户消息已接收
   - `complete`：完成（触发标题摘要、TL 派任务后处理、定时任务检测）
   - 其他事件（`stream_delta`、`status` 等）
4. **异常处理**：捕获 `Exception` 和 `BaseException`，写入 `stream_interrupted` 事件
5. **finally 兜底**：如果没有终端事件，自动补一个 `stream_interrupted`

### 3.3 Relay 生成器（`chat.py:1401-1463`）

`stream_events()` 是 SSE 的迭代器：

```python
def stream_events() -> Iterator[str]:
    relay_ready.wait(15)          # 等待 worker 确认 session_id
    deadline = time.monotonic() + STREAM_RELAY_IDLE_TIMEOUT_SECONDS  # 660s
    while True:
        rows = _events_after_cursor(...)  # 按 cursor 拉取新事件
        for row in rows:
            yield _sse(event_name, data, row.id)
            if event_name in TERMINAL_EVENTS:
                terminal_sent = True
        if terminal_sent and worker_done.is_set():
            return
        time.sleep(STREAM_RELAY_POLL_SECONDS)  # 0.08s 轮询间隔
```

#### 关键机制

- **Cursor 机制**：`(created_at, id)` 二元组做分页，确保不丢不重
- **心跳保活**：每 5 秒发一个 `heartbeat` 事件，防止代理超时
- **空闲超时**：660 秒无新事件则断开（并标记 `stream_interrupted`）
- **内部消息过滤**：`message_visibility == "internal"` 的消息及其 turn_id 全部跳过，不透给前端

---

## 4. 事件类型一览

| 事件类型 | 说明 | 来源 |
|---------|------|------|
| `session_created` | 新会话创建 | `agent_loop.py:316` |
| `user_message_received` | 用户消息已落库 | `agent_loop.py:326` |
| `status` / `stream_status` | 阶段状态（规划中、工具调用中等） | `agent_loop.py:338` |
| `stream_delta` | 文本增量分片 | `agent_loop.py:410` |
| `stream_end` | 流式输出结束 | `agent_loop.py:424` |
| `complete` | 本轮完整结束，含最终回复 | `agent_loop.py:431` |
| `stream_cancelled` | 用户取消了生成 | `agent_loop.py:380` |
| `error` / `error_occurred` | 运行时错误 | `agent_loop.py:396` |
| `stream_interrupted` | 中断（worker 异常/超时） | `chat.py:1758` |
| `heartbeat` | 中继心跳 | `chat.py:1456` |
| `scheduled_task_draft` | 定时任务草案 | `chat.py:1214` |

**终端事件集合**（`STREAM_RELAY_TERMINAL_EVENTS`）：`complete`、`error_occurred`、`stream_cancelled`、`stream_interrupted`

---

## 5. AgentLoop 层的流式实现

**文件**: `backend/app/core/agent_loop.py:298-497`

`handle_turn_stream()` → `_handle_turn_stream_v2()` 是流式生成的核心：

```
1. session_created (新会话时)
2. user_message_received
3. status: "正在规划本轮任务"
4. 调用 handle_turn() —— 同步执行完整一轮，拿到最终 response
5. 把 response.reply 按 chunk 切片，逐个 yield stream_delta
6. stream_end
7. complete (含完整 ChatTurnResponse)
```

> **注意**：当前的 v2 实现是**"先生成完再流式输出"**（先 `handle_turn()` 同步执行完，再分片推送）。真正的 token 级流式由更底层的 `HarnessV2Engine` 控制，这里只是把完整结果做了分片推送的效果。

分片大小由 `ResponseGenerator.chunk_text()` 控制。

---

## 6. 前端消费方式

**文件**: `frontend-enterprise/src/api/client.ts:146-247`

```typescript
export async function streamChatTurn(
  body: Record<string, unknown>,
  onEvent: (item: StreamEvent) => void,
  signal?: AbortSignal,
): Promise<void> {
  return streamPost('/api/chat/stream', body, onEvent, signal);
}
```

- 使用 `fetch` + `ReadableStream.getReader()` 读取
- 按 `\n\n` 切分 SSE block，解析 `event:` 和 `data:` 行
- 支持 `AbortSignal` 取消

---

## 7. 取消机制

调用 `/api/chat/sessions/{session_id}/cancel` 端点：

1. 持久化 `stream_cancelled` 事件
2. 调用 `cancel_chat_turn()` 做进程内快速取消
3. Worker 侧通过 `is_chat_turn_cancelled()` 检测取消状态
4. 生成 "已停止生成" 的 assistant message

---

## 8. 关键常量

| 常量 | 值 | 含义 |
|------|----|------|
| `STREAM_REPLY_CHUNK_SIZE` | 96 | 回复文本分片大小（字符） |
| `STREAM_RELAY_POLL_SECONDS` | 0.08 | 事件表轮询间隔（80ms） |
| `STREAM_RELAY_HEARTBEAT_SECONDS` | 5.0 | 心跳间隔 |
| `STREAM_RELAY_IDLE_TIMEOUT_SECONDS` | 660.0 | 空闲超时（11分钟） |
| `MAX_CHAT_ATTACHMENTS` | 8 | 单轮最多附件数 |
| `MAX_CHAT_ATTACHMENT_BYTES` | 12MB | 单附件最大体积 |

---

## 小结

`/api/chat/stream` 的核心设计是 **"数据库作中介的 SSE 中继"** 模式：

- Worker 线程专心跑业务逻辑，把状态变化写入 `AgentEvent` 表
- HTTP handler 轮询这张表，把新增事件推给前端
- 好处是天然支持断线续传、事件持久化、解耦生产消费
- 代价是有 80ms 级别的轮询延迟，以及数据库写入压力
