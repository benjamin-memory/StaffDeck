# LLM 协议驱动（Protocol Drivers）报文转换走读

> 范围：`backend/app/llm/` 下从「统一内部请求」到「四家供应商线协议报文」的完整转换链，
> 以及响应/SSE 事件如何被归一化回统一形态。
>
> 相关文件：
> - `backend/app/llm/client.py`：统一门面 `LLMClient`，负责构造内部请求、重试、观测
> - `backend/app/llm/protocol_drivers.py`：四个 Driver，负责真正的线协议报文转换与 HTTP 调用
> - `backend/app/llm/model_protocols.py`：协议枚举与 base_url 归一化
> - `backend/app/llm/model_config_resolver.py`：运行时模型配置快照与信任校验
>
> 设计背景见仓库根目录 `design-model-api-protocols.md`。本文只描述**当前代码实际行为**。

---

## 1. 一句话模型

业务层永远只产生一种**内部规范请求（canonical request）**——OpenAI Chat Completions 风格的
`{model, messages, temperature, max_tokens, response_format?, extra_body?, _cancellation?}`。
四个 Driver 各自把它翻译成对应供应商的线协议报文，并把响应/SSE 事件翻译回一个
**统一 completion/chunk 形态**（`choices[0].message.content` / `choices[0].delta.content`）。

```
LLMClient 组装 canonical request
        │
        ├─ ChatCompletionsDriver       ──► OpenAI SDK  chat.completions.create
        ├─ OpenAIResponsesDriver       ──► OpenAI SDK  responses.create
        ├─ AnthropicMessagesDriver     ──► Anthropic SDK messages.create
        └─ GeminiGenerateContentDriver ──► httpx POST  .../v1beta/models/{model}:generateContent
```

Driver 是 `@dataclass(frozen=True)`，无状态、可在多次调用间复用（`LLMClient._protocol_driver()`
会缓存 `self.driver`，Gemini 还能惰性重建）。

---

## 2. 进入 Driver 之前：canonical request 的形成

`LLMClient.generate_text()`（`client.py:174`）/ `generate_text_stream()`（`:303`）在调用
`driver.complete/stream` 之前，先把入参规整成统一消息序列。

### 2.1 消息三段式

`_request_messages()`（`client.py:964`）固定产出：

1. `{"role": "system", "content": system_prompt}`
2. `context_messages`：从 `user_payload["conversation_context"]` 投影出的多轮历史
   （`_prepare_user_input` / `_project_messages_from_context`）
3. 当前轮 user 消息：
   - 阶段协议（`_agent_stage`）走 `render_stage_user_message()` 渲染成结构化中文文本；
   - 普通 dict payload 用 `json.dumps(..., ensure_ascii=False)` 序列化；
   - 若同时存在历史，会包一层「本轮输入（仅用于当前调用，不写入对话历史）：」前缀。

content 既可能是**纯字符串**，也可能是**多模态分段数组**：

```json
{"role": "user", "content": [
  {"type": "text", "text": "..."},
  {"type": "image_url", "image_url": {"url": "data:image/png;base64,...."}}
]}
```

> 内部统一用 OpenAI 的 `image_url` + `data:` URL 表达图片；各 Driver 再转成自家图片形态。

### 2.2 输入预算与截断

`_fit_request_messages()`（`client.py:1095`）用粗估 token（UTF-8 字节数 / 4 + 6，`_request_tokens`）
把请求压进 `DEFAULT_INPUT_TOKEN_BUDGET = 32_000`：

- 优先删除中间（非首 system、非末 user）的普通历史；
- 保留「历史总结」消息和阶段消息，删除其余消息；
- 再保阶段消息删其它；
- 仍超预算则裁剪/丢弃最旧的阶段交换，最后才截断末条 user 文本。

### 2.3 组装给 Driver 的 request dict

```python
request = {
    "model": self.model,
    "messages": request_messages,
    "temperature": self.temperature,
    "max_tokens": max_output_tokens,          # 唯一以模型配置为准（output_policy）
    "response_format": {"type": "json_object"},  # 仅 JSON 模式
    "extra_body": {...},                      # 见 2.4
    "_cancellation": CancellationToken,       # 下划线开头，不出现在线报文
}
```

`_wire_request()`（`protocol_drivers.py:818`）在 Chat 协议出口剥掉所有 `_` 前缀的内部控制字段。

### 2.4 thinking / extra_body 的合并

- 模型配置里的 `extra_body_json`（legacy vendor 字段）与 `protocol_options`（按协议分桶的规范选项）
  通过 `_merge_extra_body()` 深合并（`client.py:924`）。
- thinking 模式来源：`extra_body.thinking.type` 或 settings 的 `model_thinking_mode`
  （还受 `model_thinking_models` 白名单约束）。
- `_thinking_request_kwargs()`（`client.py:952`）把最终 thinking 写回 `extra_body.thinking.type`，
  以 `{"extra_body": body}` 并入 request。

> ⚠️ **协议覆盖差异**：`extra_body` / `thinking` 目前**只对 Chat Completions 协议真正生效**
> （`_wire_request` 原样透传给 SDK 的 `extra_body`）。
> `_anthropic_request`、`_responses_request`、`_gemini_request` 都**不读取** `extra_body`，
> 也不透传 thinking。Anthropic 的 Extended Thinking 因此在当前线报文中不会下发。

### 2.5 JSON 模式的协议分支

`generate_json()`（`client.py:542`）在文本生成之上做最多 3 次 JSON 修复（`JSON_REPAIR_ATTEMPTS`），
具体候选由 `_generate_json_candidate()`（`:653`）决定：

- **Anthropic**：根本不传 `response_format`，改为在 system prompt 末尾追加
  「只返回一个合法 JSON object；不要输出 Markdown……」的指令（`:665-672`）。
- **其它协议**：传 `response_format={"type":"json_object"}`；供应商报不支持时
  （`_response_format_unsupported`）降级为不带 response_format 重试。

此外 `generate_text()` 还会在 user 消息末尾注入一句英文 JSON 指令
（`_with_json_mode_instruction`，`:995`），属于「提示词 + 结构化参数」双保险。

---

## 3. ChatCompletionsDriver（OpenAI Chat Completions，默认协议）

**文件位置**：`protocol_drivers.py:73-107`。这是最薄的一层——内部 canonical request 本就是 Chat 形态。

### 3.1 请求

- 非流式：`client.chat.completions.create(**_wire_request(request))`
- 流式：同上加 `stream=True`；`observable_request` 也会补 `"stream": true` 供审计。

`_wire_request` 仅剥除 `_cancellation` 这类下划线控制键，其余字段（含 `extra_body`、
`response_format`、图片分段数组）原样交给 OpenAI SDK。

### 3.2 响应

- 非流式直接返回 SDK 对象，`LLMClient` 用 `choices[0].message.content` 取文本
  （`_completion_message_content`，`client.py:692`）。
- 流式把 SDK chunk 生成器包一层，每个 chunk 前后检查取消信号，并在 `finally` 里 `stream.close()`。
  文本取自 `choices[0].delta.content`；推理摘要从 `delta.reasoning_content / reasoning / thinking`
  取（`_reasoning_text`，`client.py:1447`），只用于观测，不进入可见回复。

**转换要点**：基本恒等映射；这也是其余三家要回归到的「目标形态」。

---

## 4. OpenAIResponsesDriver（OpenAI Responses API）

**文件位置**：`protocol_drivers.py:110-190`（事件处理）、`:346-458`（报文/归一化）。

### 4.1 请求转换 `_responses_request()`（`:346`）

| canonical | Responses 线报文 |
|---|---|
| `messages: [...]` | `input: [{role, content}, ...]`（逐条转换 content） |
| 文本段 `{"type":"text"}` | `{"type":"input_text","text":...}` |
| 图片段 `{"type":"image_url"}`（仅 user） | `{"type":"input_image","image_url":...}` |
| `max_tokens` | `max_output_tokens` |
| `temperature` | `temperature` |
| —（固定） | `"store": false` |
| `response_format=json_object` | `"text": {"format": {"type": "json_object"}}` |
| system / developer / user / assistant 之外的角色 | 丢弃 |
| 空 content | 整条丢弃 |

图片与体积护栏：≤ 6 张、单张 ≤ 5MB、合计 ≤ 18MB、整报文 ≤ 25MB
（`_MAX_IMAGE_COUNT` / `_MAX_IMAGE_BYTES` / `_MAX_TOTAL_IMAGE_BYTES` / `_MAX_REQUEST_BYTES`），
超限抛 `MODEL_TOO_MANY_IMAGES` / `_IMAGE_TOO_LARGE` / `MODEL_REQUEST_TOO_LARGE`。

### 4.2 非流式 `complete()`（`:126`）

`client.responses.create(**payload)` 后经 `_responses_completion()`（`:417`）归一化：

- 遍历 `response.output[]`，只取 `type=="message"` 的 content 中 `type=="output_text"` 的 `text` 拼接；
- 包成统一对象：`SimpleNamespace(id, provider_response, usage, choices=[{message:{content:text}, finish_reason}])`；
- `finish_reason`（`_responses_finish_reason`，`:439`）：`completed→"stop"`，
  `incomplete` 且原因为 `max_output_tokens → "length"`，否则透传状态；
- usage（`_responses_usage`，`:450`）：`input_tokens→prompt_tokens`、
  `output_tokens→completion_tokens`，并带 `input_tokens_details`。

### 4.3 流式事件 → 统一 chunk（`stream()`，`:136`）

消费 Responses 的 SSE 事件流，只处理三类，其余忽略：

| 事件 | 产出 chunk |
|---|---|
| `response.created` | 记录 `response_id`，发一个空 usage chunk（携带 provider_event） |
| `response.output_text.delta` | `_stream_chunk(response_id, text=event.delta)` |
| `response.completed` / `response.incomplete` | 带 `finish_reason` + 最终 `usage` 的终止 chunk |
| `response.failed` / `error` | 转 `ProtocolCallError("MODEL_UPSTREAM_ERROR", retryable=True, ...)` |

`_stream_chunk()`（`:886`）产出的就是 OpenAI Chat 风格的 `{id, usage, choices:[{delta:{content:text}, finish_reason}]}`，
于是 `LLMClient.generate_text_stream` 里那段面向 Chat chunk 的消费逻辑（delta.content / finish_reason / usage）
对 Responses 完全复用。

---

## 5. AnthropicMessagesDriver（Anthropic Messages）

**文件位置**：`protocol_drivers.py:193-280`（驱动）、`:632-683`（请求）、`:822-883`（多模态/usage）。

### 5.1 base_url 预处理（在 `LLMClient.__init__`，`client.py:128-139`）

Anthropic SDK 总会在 base_url 后追加 `/v1/messages`，因此构造 SDK 前先剥后缀，保证最终 URL
恰好是「运维配置的 API 根 + /messages」：

- 配置以 `/v1/messages` 结尾 → 去掉它；
- 以 `/v1` 结尾 → 去掉 `/v1`；
- 同时固定 `max_retries=0`（重试由本层的空响应/错误策略掌控，不交给 SDK）。

### 5.2 请求转换 `_anthropic_request()`（`:632`）

canonical 的「system 是 messages 里一条」要改成 Anthropic 的「system 是顶层字段」，
且角色只保留 user/assistant 并需**交替连续**：

1. `role=="system"` → 收集进 `system_parts`，不进 messages；
2. 非 user/assistant 角色丢弃；
3. content 经 `_anthropic_content()`（`:822`）转 block：
   - 文本 → `{"type":"text","text":...}`；
   - 图片（仅 user，必须是 data URL）→
     `{"type":"image","source":{"type":"base64","media_type":...,"data":...}}`；
4. **相邻同角色合并**：`converted[-1].content.extend(blocks)`；
5. 若首条是 assistant（通常是 system 被抽走后错位）→ 删除首条，保证 user 开头；
6. system 文本用 `"\n\n".join` 拼到顶层 `payload["system"]`。

最终 payload：

```python
{"model": ..., "messages": converted, "max_tokens": ...,
 "temperature": ...,        # 见下面条件
 "system": ...}             # 有 system 才加
```

**温度特例**（`:657-662`）：模型名匹配 `claude-(sonnet|opus)-5...` 时**不下发 temperature**
（注释说明 LLM Center 的 Claude 5 部署拒绝旧采样字段），其它部署保留。

图片护栏同样 6 张 / 5MB / 18MB / 25MB（Anthropic 用解码后 base64 字节数统计）。
**不透传** `extra_body`、`response_format`、thinking（JSON 走 2.5 的 prompt 注入）。

### 5.3 非流式 `complete()`（`:209`）

`client.messages.create(**payload, stream=False)`，然后归一化：

- 文本 = `response.content` 中所有 `type=="text"` block 的 `text` 拼接（tool_use / thinking block 被忽略）；
- usage 经 `_anthropic_usage()`（`:872`）：`input_tokens→prompt_tokens`、
  `output_tokens→completion_tokens`、两者相加→ `total_tokens`；
- 包成统一 `choices=[{message:{content:text}, finish_reason: response.stop_reason}]`。

### 5.4 流式事件 → 统一 chunk（`stream()`，`:236`）

| Anthropic SSE 事件 | 归一化 chunk |
|---|---|
| `message_start` | 记录 `message.id`；携带初始 `usage`（input_tokens） |
| `content_block_delta` 且 `delta.type=="text_delta"` | `_stream_chunk(id, text=delta.text)` |
| `message_delta` | 携带 `delta.stop_reason`（finish_reason）+ 增量 `usage`（output_tokens） |

`text_delta` 之外的 delta（如 thinking delta）不下发文本。`finally` 关闭事件流。

---

## 6. GeminiGenerateContentDriver（Gemini Generate Content）

**文件位置**：`protocol_drivers.py:283-343`（驱动/HTTP）、`:467-629`（URL、报文、响应）。
**唯一不使用官方 SDK 的协议**，直接用同步 `httpx.Client`。

### 6.1 鉴权头与端点

- 头（`_gemini_headers`，`:467`）：同时带 `authorization: Bearer <key>` 与
  `x-goog-api-key: <key>`，`accept: text/event-stream, application/json`。
- URL（`_gemini_endpoint`，`:476`）：在 base_url 路径后补 `/v1beta`，再拼
  `/models/{quote(model)}:{method}`；流式时追加 query `alt=sse`。
  - 非流式 method：`generateContent`
  - 流式 method：`streamGenerateContent`

### 6.2 请求转换 `_gemini_request()`（`:490`）

Gemini 用 `contents: [{role, parts}]` + 顶层 `systemInstruction`，角色只有 `user`/`model`：

1. system 消息的 parts 收集进 `system_parts` → `systemInstruction: {parts}`；
2. assistant 角色映射为 `model`，其余为 `user`；
3. parts 经 `_gemini_content_parts()`（`:539`）：
   - 文本 → `{"text": ...}`；
   - 图片（仅 user、必须 data URL）→
     `{"inlineData":{"mimeType":...,"data":<原始 base64>}}`；
4. **相邻同角色合并 parts**；`contents` 首条若为 `model` 则删除（保证 user 开头）；
5. 生成参数放进 `generationConfig`：

```json
{"contents": [...],
 "systemInstruction": {"parts": [{"text": "..."}]},
 "generationConfig": {
   "temperature": ...,
   "maxOutputTokens": ...,
   "responseMimeType": "application/json"   // 仅 JSON 模式
 }}
```

护栏：≤ 6 个 inlineData、单图 ≤ 5MB、合计 ≤ 18MB、整报文 ≤ 25MB。

### 6.3 非流式 `complete()`（`:299`）

`httpx.Client.post(endpoint, headers, json=payload)`：

- 网络层 `httpx.HTTPError` → `_protocol_call_error()`；
- `status >= 400` 由 `_raise_for_gemini_response()`（`:579`）转 `ProtocolCallError`
  （尝试解析 JSON 错误体，取 `x-request-id`/`request-id`）；
- 2xx 但 body 非 JSON → `MODEL_INVALID_PROVIDER_RESPONSE`；
- 成功经 `_gemini_completion(data)`（`:600`）归一化：
  - 文本 = `candidates[0].content.parts[]` 中**非 `thought`** 的 `text` 拼接
    （`thought:true` 的思维链 part 被排除）；
  - usage：`usageMetadata.promptTokenCount / candidatesTokenCount / totalTokenCount`；
  - finish_reason = `candidates[0].finishReason`；
  - 同时塞 `message.content` 与 `delta.content`（流式/非流式复用同一归一化），
    无文本且无 finishReason 时 `choices` 为空（上层据此判空响应）。

### 6.4 流式 `stream()`（`:317`）

`httpx` 的 `client.stream("POST", ...streamGenerateContent?alt=sse)`，逐行：

- 去掉 `data:` 前缀，跳过空行与 `[DONE]`；
- 每个 JSON 片段都过 `_gemini_completion()` 转成统一 chunk（即每个 SSE 片段本身就是一个
  迷你 generateContent 响应），其 `delta.content` 被 `LLMClient` 当增量产出；
- 每个 chunk/响应前后检查取消信号；HTTP/解析错误同样归一化。

---

## 7. 响应归一化契约（四家收敛到同一形状）

不管哪家协议，Driver 出口对象都模仿 OpenAI Chat completion，`LLMClient` 因此与供应商解耦：

**非流式**

```
.choices[0].message.content   -> str   # 可见文本
.choices[0].finish_reason     -> str   # stop / length / 供应商状态
.usage.prompt_tokens / .completion_tokens / .total_tokens
.provider_response            -> 原始 SDK/JSON 响应（仅观测用）
.id                           -> 供应商请求/响应 ID
```

**流式 chunk**（由 `_stream_chunk()` 统一）

```
.choices[0].delta.content     -> str   # 增量文本（可能为空 chunk）
.choices[0].finish_reason     -> str | None
.usage                        -> 期间出现的 usage（首帧/末帧）
.provider_event               -> 原始事件（仅观测用）
```

字段映射总览：

| 维度 | Chat Completions | Responses | Anthropic | Gemini |
|---|---|---|---|---|
| 文本 | `message.content` | `output[].content[output_text]` | `content[text blocks]` | `candidates[0].parts[!thought].text` |
| 结束原因 | `finish_reason` | `status`（completed→stop, incomplete/max_output_tokens→length） | `stop_reason` | `finishReason` |
| 输入 token | `prompt_tokens` | `usage.input_tokens` | `usage.input_tokens` | `promptTokenCount` |
| 输出 token | `completion_tokens` | `usage.output_tokens` | `usage.output_tokens` | `candidatesTokenCount` |
| system 表达 | messages 内一条 | input 里 role=system/developer | 顶层 `system` | 顶层 `systemInstruction` |
| 角色集合 | system/user/assistant/... | system/developer/user/assistant | user/assistant | user/model |
| 图片形态 | `image_url` + data URL | `input_image` | `image` + base64 source | `inlineData` |
| 思维链 | `delta.reasoning_content`（仅观测） | （output_text 之外忽略） | 非 text block 忽略 | `thought:true` part 排除 |

---

## 8. 错误归一化与取消

- `CancellationToken`（`protocol_drivers.py:46`）基于 `threading.Event`。
  `_raise_if_cancelled()` 在**每次发请求前与每个流式 chunk 之间**检查，命中抛
  `MODEL_CANCELLED`。同步架构下这是协作式取消点。
- SDK/HTTP 异常统一经 `_protocol_call_error(exc)`（`:686`）抽取 status、body、
  provider code/message、request_id，再由 `_model_error_classification()`（`:711`）分类：

| 状态/异常 | code | retryable |
|---|---|---|
| 401 / authentication | `MODEL_AUTHENTICATION_FAILED` | 否 |
| 403 / permission | `MODEL_PERMISSION_DENIED` | 否 |
| 404 / not found | `MODEL_ENDPOINT_NOT_FOUND` | 否 |
| 429 / rate limit | `MODEL_RATE_LIMITED` | 是 |
| 408/504/timeout/connect | `MODEL_TIMEOUT` | 是 |
| 400/422 | `MODEL_INVALID_REQUEST` | 否 |
| 409 | `MODEL_UPSTREAM_CONFLICT` | 否 |
| 413 | `MODEL_REQUEST_TOO_LARGE` | 否 |
| 500/502/503 | `MODEL_UPSTREAM_UNAVAILABLE` | 是 |
| 其它 | `MODEL_UPSTREAM_ERROR` | 5xx 可重试 |

- 错误体先经 `_redact_sensitive_values()`/`_redact_sensitive_text()`（`:776`/`:793`）
  对 api_key、authorization、token、secret 及 `Bearer ...` 脱敏，并截断到 2000 字符。
- `LLMClient` 把 `ProtocolCallError` 包成对外的 `LLMError`（`_llm_error_from_protocol`），
  保留 `provider_code/message/upstream_body/request_id/retryable`。

### 图片不支持的自动降级

当供应商错误同时命中「图片相关词」与「不支持相关词」（`_image_parameter_unsupported()`，
`client.py:1050`，含中英文标记），且消息里确有图片时，`_without_image_parts()` 会移除图片段、
换成「图片参数不受当前模型支持；请改用任务中提供的沙箱文件路径。」文本并**重试一次**
（非流式 `:253`、流式 `:439` 各有一处）。

---

## 9. 空响应重试与观测（与 Driver 正交）

- 空响应（取完归一化文本后为空白）默认重试 `EMPTY_RESPONSE_RETRIES = 2` 次；
  控制面路由类操作在 `output_policy.OPERATION_EMPTY_RESPONSE_RETRIES` 里被显式设为 0。
- 每次 `complete/stream` 都包一个 `start_llm_call` span，记录：
  - `request_payload` = `driver.observable_request(request, stream=...)`（**线协议报文**，
    各 Driver 自己实现，便于审计真实下发内容）；
  - `response_payload` / `response_chunks` = `provider_response/provider_event` 的 JSON 化快照；
  - TTFT、chunk 数、reasoning 字符数、finish_reason、provider response id、usage token 等。
- 审计序列化（`_observable_model_value`）对 SDK 对象做 `model_dump/to_dict/vars` 兜底转换，
  深度上限 20，且对密钥字段打 `[REDACTED]`。

---

## 10. 关键结论与注意事项

1. **唯一内部表示是 OpenAI Chat 形态**，三家非 OpenAI 协议都靠 Driver 做双向（请求 + 响应/SSE）适配；
   新增供应商只需实现 `complete/stream/observable_request` 三个方法并在 `LLMClient.__init__` 注册。
2. **同步、无状态、可取消**：四个 Driver 全部基于同步 SDK/httpx；流式是同步生成器，
   取消依赖 chunk 边界上的 `CancellationToken` 轮询。
3. **能力不对等**（容易踩坑）：
   - `extra_body` / `thinking` 仅 Chat Completions 透传，Anthropic/Responses/Gemini 均忽略；
   - JSON 模式：Chat/Responses/Gemini 走结构化参数（`response_format` / `text.format` /
     `responseMimeType`），**Anthropic 走 system prompt 注入**；
   - temperature 对 `claude-(sonnet|opus)-5` 被刻意省略；
   - 图片仅允许 user 角色、必须是 `data:` URL，统一 5MB / 6 张 / 18MB / 25MB 护栏。
4. **多轮历史需重排**：Anthropic/Gemini 要求角色交替且 user 开头，Driver 通过
   「system 上提 + 相邻同角色合并 + 删除 leading assistant」保证；上层无需关心。
5. **思维链不进可见回复**：reasoning/thinking/thought 内容只用于观测，文本提取时一律排除。
6. **信任校验在更上层**：Driver 只拿解密后的 snapshot（`ResolvedModelConfig`），
   是否允许运行由 `resolve_model_config_for_runtime` 的 `trust_status` + 指纹决定，Driver 不负责。
