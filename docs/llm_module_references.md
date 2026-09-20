# `backend/app/llm` 模块引用关系分析

> 分析日期：2026-09-20
> 范围：`backend/app/llm` 目录的构成，以及它在后端代码中被引用的具体位置与方式。

## 1. 目录定位

`backend/app/llm` 是整个后端的 **LLM 基础设施层**，对上提供统一的模型调用客户端、阶段协议、配置解析与提示词模板，对下封装多家厂商的 HTTP 协议差异。业务模块（core / skills / knowledge / memory 等）只依赖本包，不直接接触各家 SDK。

## 2. 文件构成与职责

| 文件 | 职责 |
|------|------|
| `__init__.py` | 包入口，仅导出 `LLMClient`、`LLMError` |
| `client.py` | 核心客户端 `LLMClient`，对外方法 `generate_text` / `generate_text_stream` / `generate_json` / `generate_json_sequence`；含重试、空响应修复、token 裁剪、图片处理、观测埋点 |
| `protocol_drivers.py` | 四家协议的实际 HTTP 驱动：`ChatCompletionsDriver`、`OpenAIResponsesDriver`、`AnthropicMessagesDriver`、`GeminiGenerateContentDriver` |
| `model_protocols.py` | 协议枚举 `ModelApiProtocol`、协议选项归一化、base_url 校验、配置指纹 |
| `model_config_resolver.py` | 从 DB 的 `ModelConfig` 解析出运行时配置：`resolve_model_config_for_runtime` / `resolve_model_config_for_verification` / `snapshot_model_config` |
| `stage_protocol.py` | 统一 Agent 阶段协议：`unified_system_prompt`、`stage_payload`、`render_stage_user_message`、各阶段输出 schema、`TURN_STAGE_MESSAGES_KEY` 等常量 |
| `output_policy.py` | 按操作类型决定 token 上限与空响应重试次数 |
| `schemas.py` | 模型配置管理 API 的 Pydantic schema（`ModelConfigCreateRequest` 等） |
| `prompts/*.md` | 27 个提示词模板（各 Agent 阶段、技能、知识、记忆等） |

## 3. 三种被引用方式

1. **作为包入口导入客户端**
   ```python
   from app.llm import LLMClient, LLMError
   ```
2. **直接导入子模块符号**
   ```python
   from app.llm.model_config_resolver import resolve_model_config_for_runtime
   from app.llm.stage_protocol import stage_payload, unified_system_prompt
   ```
3. **运行时按文件路径读取 `prompts/*.md`**
   ```python
   PROMPT_PATH = paths.resource_dir() / "app" / "llm" / "prompts" / "router_prompt.md"
   PROMPT_PATH.read_text(encoding="utf-8")
   ```

## 4. 具体引用代码清单

### 4.1 `LLMClient` / `LLMError`（经 `__init__.py`）

**core（Agent 执行核心）**
- `app/core/agent_loop.py:59`
- `app/core/harness_agent.py:23`
- `app/core/step_agent.py:13`
- `app/core/turn_planner.py:17`
- `app/core/router.py:12`
- `app/core/reflection_agent.py:13`
- `app/core/response_generator.py:15`（仅 `LLMClient`）

**api**
- `app/api/chat.py:53`
- `app/api/model_configs.py:14`
- `app/api/skills.py:53`（仅 `LLMError`）

**业务服务**
- `app/memory/service.py:11`（仅 `LLMClient`）
- `app/knowledge/service.py:49`
- `app/evolution/service.py:33`（仅 `LLMClient`）
- `app/feedback/service.py:10`
- `app/channels/service_autoroute.py:20`（仅 `LLMClient`）
- `app/scheduled_tasks/service.py:35`

**技能 / 通用技能**
- `app/skills/skill_editor.py:10`
- `app/skills/skill_reflection.py:8`
- `app/skills/skill_distiller.py:12`
- `app/general_skills/runner.py:34`

### 4.2 `model_config_resolver`（运行时模型配置解析）

- `resolve_model_config_for_runtime`
  - `app/core/agent_loop.py:60`
  - `app/knowledge/service.py:34`
  - `app/evolution/service.py:34`
  - `app/feedback/service.py:11`
  - `app/api/general_skills.py:64`
  - `app/api/knowledge.py:41`
  - `app/api/skills.py:52`
- `snapshot_model_config`
  - `app/skills/llm_limits.py:4`
  - `app/general_skills/runner.py:35`
- 多符号导入
  - `app/agents/branching.py:29`
  - `app/api/model_configs.py:15`

### 4.3 `stage_protocol`（统一 Agent 阶段协议）

- `app/core/agent_loop.py:63` — `stage_payload, unified_system_prompt`
- `app/core/step_agent.py:14`
- `app/core/turn_planner.py:18`
- `app/core/router.py:13` — `ROUTER_OUTPUT_SCHEMA, stage_payload, unified_system_prompt`
- `app/core/reflection_agent.py:14`
- `app/core/response_generator.py:16`
- `app/general_skills/runner.py:36`
- `app/core/context_projection.py:7` — 仅用常量 `TURN_STAGE_MESSAGES_KEY`

### 4.4 `model_protocols` / `schemas`

- `app/api/model_configs.py:20`（`model_protocols`）
- `app/api/model_configs.py:30`（`schemas`）
- 包内部依赖：
  - `app/llm/client.py:19`（`model_protocols.ModelApiProtocol`）
  - `app/llm/model_config_resolver.py:12`（`model_protocols`）

> `output_policy.py` 与 `protocol_drivers.py` 仅被包内 `client.py` 引用（`client.py:20`、`client.py:24`），不对外暴露。

### 4.5 `prompts/*.md` 的文件路径加载

| 加载位置 | 提示词文件 |
|----------|-----------|
| `app/llm/stage_protocol.py:11` | `unified_agent_prompt.md`（统一主提示词） |
| `app/core/router.py:23` | `router_prompt.md` |
| `app/core/step_agent.py:23` | `step_agent_prompt.md` |
| `app/core/step_agent.py:25-46` | `step_agent_repair_rules.md`、`step_agent_tool_continuation_rules.md`、`step_agent_knowledge_rules.md`、`step_agent_awaiting_input_rules.md`、`step_agent_general_skill_rules.md`、`step_agent_tool_rules.md` |
| `app/core/turn_planner.py:36` | `turn_planner_prompt.md` |
| `app/core/harness_agent.py:27` | `harness_agent_prompt.md` |
| `app/core/reflection_agent.py:24` | `reflection_prompt.md` |
| `app/core/response_generator.py:22` | `response_generator_prompt.md` |
| `app/skills/skill_editor.py:25` | `skill_editor_prompt.md` |
| `app/skills/skill_reflection.py:12` | `skill_reflection_prompt.md` |
| `app/skills/skill_distiller.py:25` | `skill_distiller_prompt.md` |
| `app/general_skills/runner.py:39-45` | `general_skill_selector_prompt.md`、`general_skill_runner_prompt.md`、`general_skill_repair_prompt.md`、`general_skill_review_prompt.md`、`general_skill_reply_prompt.md`、`general_skill_read_prompt.md` |
| `app/knowledge/service.py:54-58` | `knowledge_bucket_prompt.md`、`knowledge_discovery_prompt.md`、`knowledge_search_prompt.md`、`knowledge_document_route_prompt.md` |
| `app/memory/service.py:17` | `memory_extractor_prompt.md` |

> 注意：`prompts/` 目录下未被上述代码显式拼路径加载的 `.md` 文件不会进入运行时。

## 5. 典型调用链（以路由 Router 为例）

`app/core/router.py:36-52` 展示了业务模块使用本包的标准模式——先用 `stage_protocol` 组装阶段 payload，再读取 `.md` 提示词，最后通过 `LLMClient` 发起结构化调用：

```python
from app.llm import LLMClient, LLMError
from app.llm.stage_protocol import ROUTER_OUTPUT_SCHEMA, stage_payload, unified_system_prompt

PROMPT_PATH = paths.resource_dir() / "app" / "llm" / "prompts" / "router_prompt.md"

payload = stage_payload(
    phase="Router",
    user_message=message,
    instructions=PROMPT_PATH.read_text(encoding="utf-8"),
    output_contract=ROUTER_OUTPUT_SCHEMA,
    ...
)
raw = LLMClient(model_config).generate_json(unified_system_prompt(), payload)
```

整体依赖方向：

```
业务模块 (core / skills / knowledge / memory / api ...)
        │  from app.llm import LLMClient
        ▼
LLMClient (client.py)
        ├── model_config_resolver  ──► model_protocols
        ├── stage_protocol         （读取 prompts/unified_agent_prompt.md）
        ├── output_policy
        └── protocol_drivers       ──► OpenAI / OpenAI Responses / Anthropic / Gemini
```

## 6. 测试引用

`backend/tests/` 下约 17 个测试文件直接引用本包，包括：
`test_llm_client.py`、`test_anthropic_driver.py`、`test_openai_responses_driver.py`、`test_gemini_driver.py`、`test_model_protocols.py`、`test_model_configs_api.py`、`test_router.py`、`test_response_generator.py`、`test_reflection_agent.py`、`test_step_agent_model_driven.py`、`test_unified_agent_prompt.py`、`test_single_port_app.py`、`test_memory_service.py`、`test_knowledge_base.py`、`test_general_skills.py`、`test_feedback_analysis.py` 等。
