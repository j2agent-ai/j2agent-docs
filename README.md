# AI Center 文档中心

本目录集中存放 **inc-ai** 工作区内 AI Center 相关项目文档，按主题归类为四大板块。

## 平台 — 平台能力

| 文档 | 说明 |
|------|------|
| [RAG 机制](平台/RAG机制/README.md) | 知识库同步、融合检索、静态文件展示 |
| [安全与用户](平台/安全与用户/README.md) | 用户权限、邮箱注册 |
| [LLM 提供商配置](平台/LLM提供商配置/README.md) | `api_provider_config`、深度思考元数据 |
| [Agent 记忆机制](平台/agent记忆机制/README.md) | 多轮记忆、`conversationId`、滑动窗口、Redis/JDBC、流式进行中状态 |
| [Agent-UI 交互机制](平台/agent-ui交互机制/README.md) | 状态机、`AgentUiEventEnvelope`、WebSocket 事件 |
| [插件 Agent 接入与界面](平台/插件Agent接入与界面/README.md) | 插件 JAR → 注册 → REST/WebSocket 全链路 |
| [平台通用助手](平台/通用助手/README.md) | 内置 `universal_assistant`、被动意图召回与调用子智能体、双轨记忆 |
| [文件管理与对象存储](平台/文件管理与对象存储/README.md) | 虚拟目录、直传、差异检查、MinIO/OSS/七牛/R2 |
| [聊天图片附件](平台/聊天图片附件/README.md) | 对话图片上传、访问模式、引用保护与 LLM 投递 |

## 基础设施 — 部署与运维

| 文档 | 说明 |
|------|------|
| [Docker 部署](基础设施/docker部署/README.md) | Compose 全栈部署、构建启动、数据卷、离线镜像、前端热更新 |

## 前端

前端代码路径均**相对于前端工程根目录**，写作 `src/...`、`lib/...`，不写具体项目/仓库名（如 `j2agent-web/`）。Chat 模块逻辑集中在 `src/pages/chat/ts/`。

| 文档 | 说明 |
|------|------|
| [智能体多任务机制](前端/智能体多任务机制/README.md) | 并行流式任务登记、会话跳转、离开守卫与全局任务浮窗 |
| [Markdown 解析器](前端/md解析器/README.md) | 聊天气泡 Markdown 渲染与图表懒加载 |

## agent开发 — Agent 插件开发

| 文档 | 说明 |
|------|------|
| [快速入门](agent开发/文档/README.md) | 前置条件、工程骨架、五步入门 |
| [Agent 开发](agent开发/文档/Agent开发.md) | `AiAgent` 契约、生命周期、热重载 |
| [工具](agent开发/文档/工具.md) | `@Tool` 编写与 `buildTools()` |
| [Skill](agent开发/文档/Skill.md) | 技能目录与 `ExternalSkills` |
| [MCP](agent开发/文档/MCP.md) | MCP 配置与 `McpFeature` |
| [可选能力](agent开发/文档/可选能力.md) | RAG、热门问题、深度思考 |
| [示例 Agent 模板](agent开发/agents/0_example-agent/README.md) | 复制 `0_example-agent` 开始开发 |

## 其他

| 文档 | 说明 |
|------|------|
| [原始需求规格](原始需求规格/README.md) | 平台原始需求追溯：主表（高层）+ 附录（详细），反向归纳自专题文档与代码 |
| [智能体平台架构图](智能体平台.drawio) | Draw.io 平台总体框架与技术选型 |
| [AI 编码规范](dev/rule.md) | 平台开发辅助规则 |
