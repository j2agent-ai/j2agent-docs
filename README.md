# J2Agent 文档中心

本目录集中存放 **j2agent-ai** 工作区内 J2Agent 相关项目文档，按主题归类为四大板块。

## 平台 — 平台能力

| 文档 | 说明 |
|------|------|
| [RAG 机制](平台/RAG机制/README.md) | 知识库同步、融合检索、静态文件展示 |
| [安全与用户](平台/安全与用户/README.md) | 用户权限、邮箱注册 |
| [LLM 提供商配置](平台/LLM提供商配置/README.md) | `api_provider_config`、深度思考元数据 |
| [Agent 对话记录](平台/agent对话记录/README.md) | 多轮记忆、`conversationId`、Redis/JDBC |
| [Agent-UI 交互机制](平台/agent-ui交互机制/README.md) | 状态机、`AgentUiEventEnvelope`、WebSocket 事件 |
| [插件 Agent 接入与界面](平台/插件Agent接入与界面/README.md) | 插件 JAR → 注册 → REST/WebSocket 全链路 |

## 基础设施 — 部署与运维

| 文档 | 说明 |
|------|------|
| [Docker 部署](基础设施/docker部署/README.md) | Compose 全栈部署、构建启动、数据卷、离线镜像、前端热更新 |

## 前端

| 文档 | 说明 |
|------|------|
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
| [智能体平台架构图](智能体平台.drawio) | Draw.io 平台总体框架与技术选型 |
| [AI 编码规范](dev/rule.md) | 平台开发辅助规则 |
