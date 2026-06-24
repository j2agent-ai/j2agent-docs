# LLM 提供商配置

本文说明 **管理端「设置 → LLM / Embedding 接口」** 与库表 `api_provider_config` 的对应关系，以及各 `provider_type` 的 **baseUrl 填写示例**。

相关代码：

| 模块 | 路径 |
|------|------|
| 配置 CRUD / 热更新 | `config/provider/ApiProviderConfigService.java`、`ActiveProviderHolder.java` |
| ChatModel 装配 | `config/llm/LlmBackedChatModelFactory.java`、`ReloadableRoutingChatModel.java` |
| **深度思考 metadata 适配** | `service/llm/reasoning/SpringAiReasoningMetadataAdapter.java`（见 §1.3） |
| 管理端表单 | `src/pages/settings/components/ProviderConfigForm.vue`、`src/pages/settings/components/ModelConfigListPanel.vue` |
| 对话记忆 Advisor | `service/llm/advisor/ReactCompatibleMessageChatMemoryAdvisor.java`（见 [Agent 对话记录机制](../agent对话记录/README.md)） |

## 1. 数据模型

- 表：`api_provider_config`（Flyway `V0_2__migrate_provider_config.sql` 自旧 `ai_properties` 迁移）。
- 每种 `api_type`（`llm` / `embedding`）可有多条配置，**仅一条** `enabled=1` 且 `is_current=1` 为当前生效项。
- 连接参数在 `config_json`（JSON）；`api_key` 存库，接口返回时脱敏。
- 修改当前生效项后发布 `ProviderConfigChangedEvent`，由 `AiRuntimeReloadService` 热更新 ChatModel / Embedding 客户端。

### 1.1 LLM `config_json` 常用字段

| 字段 | 说明 |
|------|------|
| `modelName` | 提供商侧模型 ID |
| `baseUrl` | 服务根地址（**不含** chat/embeddings 路径） |
| `apiKey` | 认证密钥 |
| `completionsPath` | 仅 **OpenAI 兼容 / vLLM / LM Studio** 使用 |
| `embeddingsPath` | 仅 **Embedding + OpenAI 兼容** 使用 |
| `maxTokens` | 仅 **Anthropic兼容**；可选正整数；对应 Messages API `max_tokens`（单次回复最大输出 token）；未填时运行时默认 **16384** |
| `contextLength` | 仅 **Ollama**（对应 `num_ctx`）；可省略或留空，省略后使用模型 Modelfile 或 Ollama 默认值；**open-ai / vllm / anthropic 无此 API 参数**，保存时会剔除遗留键 |
| `keepAliveSeconds` | 仅 **Ollama** |
| `temperature` | 采样温度（LLM 运行时） |
| `thinkingMode` | 仅 **Anthropic兼容 / open-ai / LM Studio / Ollama**；`provider_default`（默认，不传参）/ `on` / `off`（可被单轮聊天请求或 Agent 默认覆盖）；历史值 `auto` 读时等价 `provider_default` |
| `thinkingBudgetTokens` | **Anthropic兼容** 且 `thinkingMode=on` 时对应 `thinking.budget_tokens`；**LM Studio** 且 `thinkingMode=on` 时映射为 `reasoning_tokens`；**open-ai 不使用**；未填默认 **4096** |

以下字段 **已从产品与配置中移除**（保存时会被剔除，旧 JSON 中的键会被忽略）：`useRag`、`useTools`、`useMcpTools`、`chatMemoryDualRead`。RAG / 工具 / MCP 由各 Agent 实现与 MCP 运行时配置决定，不再挂在 LLM 提供商条目上。

保存时：Ollama 的 `contextLength` 为 `null`、空或 `≤0` 时不写入 JSON，运行时亦仅在正整数时设置 `num_ctx`；Anthropic兼容 的 `maxTokens` 缺失或无效时不写入 JSON，运行时默认 **16384**；非 Ollama 配置会剔除 `contextLength`，非 Anthropic兼容 会剔除 `maxTokens`。

**深度思考**：**open-ai** 使用 `reasoning_effort`（on=high，off=low）；**vLLM** 本期不支持 `thinkingMode`；Anthropic兼容 使用 Messages API `thinking`；**LM Studio** 使用 OpenAI 兼容 `reasoning_effort` + `reasoning_tokens`；Ollama 使用 `think` 字段（Spring AI 标准封装）。

### 1.2 深度思考运行时优先级

- LLM 提供商配置中的 `thinkingMode` / `thinkingBudgetTokens` 是**全局默认值**（管理端「设置 → LLM 接口」）。
- 单轮 WebSocket 请求可在 `ChatRequestDto.thinkingMode` 指定 `provider_default` / `on` / `off`（供聊天界面开关，后续前端接入）；仍接受历史值 `auto`。
- Agent 可在代码中声明默认策略（`AiAgent#getThinkingOverride()`），仅在请求未传 `thinkingMode` 时生效。
- **生效优先级**：`ChatRequestDto.thinkingMode`（有值）> `Agent 默认` > `提供商默认`。
- 实现上由 `ChatService` 按 `conversationId` 写入 `ThinkingOverrideRegistry`，`ReloadableRoutingChatModel` 在每次 LLM 调用时从 Prompt metadata 读取并应用（跨 Reactor 线程安全）。
- 覆盖仍受 provider 能力约束：**vLLM** 即使覆盖也不会下发 thinking 参数。
- Anthropic兼容 / LM Studio 下 `mode=on` 时，budget 取自提供商配置，未配置时默认 `4096`。
- **open-ai** 下 `mode=on` 时下发 `reasoning_effort=high`；`mode=off` 时下发 `reasoning_effort=low`（请配合推理模型使用）。
- LM Studio 下 `mode=on` 时下发 `reasoning_effort=high` 与 `reasoning_tokens`；`mode=off` 时下发 `reasoning_effort=low`。

### 1.3 Spring AI 深度思考 metadata 坑（必读）

Spring AI **统一了** `ChatModel.call/stream()` 与 `AssistantMessage` 类型，但 **没有** 统一的 reasoning API（例如 `getReasoningContent()`）。各 `*ChatModel` 把 Provider 原生 thinking 写进 **不同的 metadata 键与 content 布局**：

| Spring AI 消息形态 | 典型 ChatModel | 若只读 `metadata.reasoningContent` 的后果 |
|--------------------|----------------|---------------------------------------------|
| `metadata["reasoningContent"]` | OpenAI 兼容 | 正常 |
| `metadata["thinking"] == true` + `getText()` | Anthropic 流式 | thinking 被当成最终回答，UI 错乱 |
| `metadata["signature"]` + `getText()` | Anthropic 非流式 | 同上 |
| `Generation.metadata["thinking"]`（累积字符串） | Ollama 流式 | 思考丢失或被流过滤丢弃 |
| `metadata["type"] == "thinking"` + `getText()` | Ollama 部分路径 | 同上 |

**本项目约定**：业务层与 DTO 只认统一字段 **`reasoningContent`**（`SpringAiReasoningMetadataAdapter.UNIFIED_REASONING_KEY`）。  
**禁止**在 `ChatService` / `Translator` 等处直接解析 Provider 专有 metadata；一律经 **`SpringAiReasoningMetadataAdapter`** 适配后再写入 `MessageDto.reasoningContent` 或持久化 `meta_json`。

适配类路径：`j2agent-server/.../service/llm/reasoning/SpringAiReasoningMetadataAdapter.java`  
流式双通道与 UI 协议见 [Agent UI 交互机制 §4](../agent-ui交互机制/README.md)。

## 2. baseUrl 填写示例

管理端在「模型连接参数 → Base URL」下按提供商展示示例；下表与线上一致。

### 2.1 LLM

| provider_type | baseUrl 示例 | 说明 |
|---------------|--------------|------|
| `open-ai` | `https://dashscope.aliyuncs.com` | 通义等 **OpenAI 兼容**；需配 `completionsPath` 如 `/compatible-mode/v1/chat/completions`；深度思考需推理模型（如 qwen-plus） |
| `open-ai` | `https://api.openai.com` | 官方 OpenAI；深度思考需 o 系列推理模型（如 o3/o4-mini） |
| `vllm` | `http://127.0.0.1:8000` | 本机 vLLM；`completionsPath` 常为 `/v1/chat/completions` |
| `anthropic` | `https://dashscope.aliyuncs.com/apps/anthropic` | 百炼 **Anthropic兼容 Messages 兼容**（通义 `qwen*` 等）；**不要**填 OpenAI 的 `completionsPath` |
| `anthropic` | 留空或 `https://api.anthropic.com` | Claude 官方 API |
| `ollama` | `http://127.0.0.1:11434` | 本机 Ollama |
| `lm-studio` | `http://127.0.0.1:1234` | 本机 LM Studio；`completionsPath` 默认 `/v1/chat/completions`；本地推理模型推荐此类型（勿与 `anthropic` 混用） |

### 2.2 Embedding

| provider_type | baseUrl 示例 |
|---------------|--------------|
| `open-ai` | `https://dashscope.aliyuncs.com` |
| `ollama` | `http://127.0.0.1:11434` |

百炼 OpenAI 兼容 Embedding 单条 input 长度一般为 **`[1, 8192]`**（字符/token 以控制台为准）。检索侧超长 query 的切分、混合检索与融合规则见 [融合检索](../RAG机制/融合检索.md)（配置前缀 `com.nms.ai.retrieve`）。

### 2.3 LM Studio 注意点

- 使用 **OpenAI 兼容** `/v1/chat/completions`，非 Anthropic `/v1/messages`。
- 深度思考响应经 `reasoning_content` 返回，由 `SpringAiReasoningMetadataAdapter` 适配为 `reasoningContent`。
- 部分 GGUF 模型需在 LM Studio UI 开启 Reasoning Parsing；REST `chat_template_kwargs.enable_thinking` 在部分版本/模型上可能被忽略，以 `reasoning_effort` 为主。
- 官方文档：[LM Studio OpenAI Chat Completions](https://lmstudio.ai/docs/developer/openai-compat/chat-completions)。

### 2.4 百炼 Anthropic兼容 注意点

- 官方文档：[Anthropic 兼容 Messages](https://help.aliyun.com/zh/model-studio/anthropic-api-messages)。
- 推荐 baseUrl：`https://dashscope.aliyuncs.com/apps/anthropic`（**不要**在末尾自行拼接 `/v1`；Spring AI 会相对 baseUrl 请求 `/v1/messages`）。
- 若误填为 `https://dashscope.aliyuncs.com/v1` 等 OpenAI 风格根地址，请求会落到错误路径，表现为 404 或鉴权失败。
- 模型名示例：`qwen3.6-plus`（以控制台为准）。
- 偶发 `Connection reset by peer`：多为网关关闭空闲连接后客户端复用失效 TCP 连接所致。服务端已通过 `LlmReactiveHttpClientFactory` 限制连接池空闲时间，并在首 token 前对 RST 自动重试（最多 2 次）。若持续失败，请检查 API Key、配额及百炼控制台状态。

保存/启用 LLM 配置时**不再**根据模型名与 baseUrl 做组合校验；请自行按网关文档选择 `provider_type` 与 baseUrl（误配时可能在对话阶段出现 404、鉴权失败或空流式响应）。

## 3. 相关文档

| 文档 | 内容 |
|------|------|
| [Agent 对话记录机制](../agent对话记录/README.md) | `conversationId`、Advisor、Redis/JDBC 记忆 |
| [Agent UI 交互机制](../agent-ui交互机制/README.md) | `providerError` 与 FAILED 事件；`reasoningContent` 双通道 |
| [Docker 部署](../../基础设施/docker部署/README.md) | compose、数据卷、网络 |
| [RAG 机制](../RAG机制/README.md) | 知识库同步、融合检索、静态资源 |
