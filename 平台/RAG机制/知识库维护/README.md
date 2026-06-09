# 知识库维护

本模块描述 `knowledge-repo` 的 **入库与维护**：目录约定、增量同步、启动初始化、完全重建、Redis 分布式锁与维度一致性。

## 文档

| 文档 | 说明 |
|------|------|
| [知识库维护](知识库维护.md) | 目录/`info.json`、分片、同步、协调器、exclusive 门禁、维度一致性 |
| [静态文件展示机制](../静态文件展示机制.md) | 图片等静态资源 URL 改写与 HTTP 直链（不入向量库） |

## 代码入口

| 模块 | 路径 |
|------|------|
| 维护协调器 | `.../service/rag/knowledge/repo/KnowledgeRepoMaintenanceCoordinator.java` |
| Redis 维护锁 | `.../service/rag/knowledge/repo/KnowledgeRepoMaintenanceLockService.java` |
| 同步逻辑 | `.../service/rag/knowledge/repo/KnowledgeRepoSyncService.java` |
| 向量库初始化 | `.../config/VectorDatabaseInit.java` |
| Embedding 变更委托 | `.../service/embedding/EmbeddingChangeOrchestrator.java` |
| Milvus 写入 | `.../service/rag/knowledge/MilvusKnowledgeWriteService.java` |

## 与检索模块的关系

查询侧混合检索、超长 query 融合、维护期间检索降级见 [检索模块](../检索/README.md)。
