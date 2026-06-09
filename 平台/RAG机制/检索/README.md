# 检索

本模块描述 RAG **查询侧**：向量化、混合检索、超长 query 融合，以及知识库维护期间的检索降级。

## 文档

| 文档 | 说明 |
|------|------|
| [融合检索](融合检索.md) | Milvus 稠密+稀疏混合检索、QueryChunker 多段融合、配置与验证 |

## 代码入口

| 模块 | 路径 |
|------|------|
| 检索引擎 | `j2agent/j2agent-server/.../service/rag/retrieval/Retriever.java` |
| 查询切分 | `.../service/rag/retrieval/QueryChunker.java` |
| Milvus 混合检索 | `.../service/rag/vdb/milvus/MilvusService.java` |
| 对话 RAG | `.../rag/AbstractCollectionKbRetriever.java` |

## 与知识库维护的关系

- 入库、增量同步、完全重建见 [知识库维护](../知识库维护/README.md)。
- `KnowledgeRepoMaintenanceCoordinator.isExclusiveSyncActive()` 为 true 时，检索返回降级/空结果，避免用新维度 query vector 查询旧 collection schema。
- Milvus 检索前会校验 query 向量维度与 `lastDimension`、collection schema 一致。
