# example-agent

**最小可复制模板**。新业务 Agent 建议复制本目录，再按需扩展工具、Skill、RAG 等能力。

## 复制步骤

1. 复制整个 `example-agent/` 目录并重命名（如 `my-custom-agent/`）
2. 修改 `pom.xml` 中的 `artifactId`
3. 重命名 `ExampleAgent.java` 或改类名，并修改：
   - `getAgentId()` — **全局唯一**，与 WebSocket `agent-id` 一致
   - `getAgentName()` / `getAgentDescription()`
   - `getOrchestrationPrompt()`（可选，编排提示词：能力域 + 典型问法，不展示到界面）
4. 编辑 `src/main/resources/system-prompt.md`
5. 若放入本仓库 `agents/`，在 [`../pom.xml`](../pom.xml) 的 `<modules>` 中追加模块名

## 目录说明

```text
example-agent/
  pom.xml
  src/main/java/.../ExampleAgent.java
  src/main/resources/system-prompt.md
  src/main/assemblies/agent-package.xml   # 解压目录 + tar.gz
```

## 打包与部署

```bash
mvn clean package
```

`target/` 产出：

```text
target/
  <artifactId>-<version>.jar    # 瘦 JAR
  <artifactId>-<version>/         # 解压目录（与 tar.gz 内容一致）
    <artifactId>-<version>.jar
    resources/...
  <artifactId>-<version>.tar.gz
```

**本地部署：**

```bash
cp -a target/example-agent-1.0.0-SNAPSHOT/. "$PLUGIN_DIR/example-agent-1.0.0-SNAPSHOT/"
# POST /v1/rest/j2agent/agents/reload
```

**分发部署：**

```bash
mkdir -p "$PLUGIN_DIR/example-agent-1.0.0-SNAPSHOT"
tar -xzf target/example-agent-1.0.0-SNAPSHOT.tar.gz -C "$PLUGIN_DIR/example-agent-1.0.0-SNAPSHOT/"
```

扩展能力见 [Agent 开发文档](../../文档/Agent开发.md) 与 [快速入门](../../文档/README.md)。
