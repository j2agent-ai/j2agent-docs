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
| 远程知识库仓库配置 | `.../service/rag/knowledge/repository/KnowledgeRepositoryService.java` |

## 远程知识库名称

知识库列表支持为 **远程知识库** 设置系统展示名称；本地文件型知识库为只读目录，不提供展示名称配置。

| 项 | 规格 |
|----|------|
| 配置范围 | 仅 `KnowledgeRepositoryConstants.TYPE_REMOTE` 的远程仓库配置 |
| 前端入口 | 知识库列表页新增/编辑远程 Git 配置时填写“知识库名称” |
| 落库位置 | 远程仓库记录的 `protocol_config` JSON |
| JSON 结构 | `{"display_name":"维基"}`；可选精确覆盖为 `{"collectionAliases":{"rc_wiki":"维基"}}` |
| DTO 字段 | 对外暴露为 `KnowledgeRepositoryDtos.Item.displayName` / `UpsertRequest.displayName`，并兼容 `collectionAliases` |
| 兼容规则 | 读取时兼容旧 `protocol_config.alias`，保存时统一写回 `display_name` |
| 空值规则 | 名称为空时不保存；保存时会清理空名称 |
| 展示规则 | 优先使用 collection 精确名称，其次使用远程知识库名称；显示为 `知识库名称 (collectionId)`，无名称显示 collectionId |

该名称只影响前端展示，不改变 Milvus collection 名称，也不改变聊天请求中的 `knowledgeCollections` 真实 id。

## 与检索模块的关系

查询侧混合检索、超长 query 融合、维护期间检索降级见 [检索模块](../检索/README.md)。
