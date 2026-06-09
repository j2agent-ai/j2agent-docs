# Markdown 解析器（聊天气泡渲染）

本文说明前端如何将 Agent 回复中的 Markdown 转为聊天气泡 HTML，并异步渲染其中的图表块。

> 后端知识库分片用的 `MarkdownQaParser` 属于 RAG 管线，与本文无关。

## 文档索引

| 文档 | 说明 |
|------|------|
| [架构与流程.md](架构与流程.md) | `markdown-it`、围栏类型、异步渲染与流式推迟 |
| [图表渲染.md](图表渲染.md) | Mermaid / Vega-Lite / PlantUML、xychart 标签重叠、错误与预览 |
| [样式约定.md](样式约定.md) | `markdown.scss`、图表高度、行内 code 与围栏 code |

## 职责边界

| 层级 | 职责 |
|------|------|
| 后端 `ChatService` | 流式推送 `MessageDto.content`（Markdown 原文） |
| `markdownRenderer.ts` | `markdown-it` 同步 HTML + 图表块懒加载渲染 |
| `ChatView.vue` | `v-html` 挂载、`renderMarkdownBlocks` 触发、全屏预览 SVG |
| `markdown.scss` | 表格、图表容器、行内代码等视觉 |

与 WebSocket 状态机、气泡 busy 等约定见后端文档 [Agent-UI 交互机制](../../平台/agent-ui交互机制/README.md)。

## 支持的图表能力

- **Mermaid**（` ```mermaid `）：flowchart、pie、xychart-beta、sequenceDiagram 等
- **Vega-Lite**（` ```vega-lite ` / ` ```vegalite `）
- **PlantUML**（` ```plantuml ` / ` ```puml `，依赖 `/vendor/plantuml-js/plantuml.js`）
- **HTML 预览**（` ```html `，sandbox iframe）

## 关键代码

| 主题 | 路径 |
|------|------|
| 解析与图表 | `src/utils/markdownRenderer.ts` |
| 样式 | `src/styles/markdown.scss` |
| 挂载点 | `src/pages/chat/components/ChatView.vue` |

## 智能报表 Agent 约定

插件 Skill `data-chart` 要求 TopN 柱状图使用 Mermaid `xychart-beta`。前端在类目数 ≥ 8 且未写 `horizontal` 时会自动补上横向布局，详见 [图表渲染.md](图表渲染.md)。
