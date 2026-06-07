# 文件管理与对象存储

本文档说明 J2Agent 文件管理功能、对象存储供应商配置，以及对象存储和数据库台账之间的人工同步流程。

## 1. 功能范围

- 仅 `ADMIN` 角色可访问首页入口、系统菜单和 `/files` 页面。
- 首版只管理 `j2agent.storage.bucket` 配置的默认 Bucket，不支持页面切换 Bucket。
- 支持虚拟目录、面包屑导航、名称搜索、状态筛选和分页。
- 支持上传、短期签名 URL 预览/下载、单个删除和批量删除。
- 支持 MinIO、阿里云 OSS、七牛云 Kodo 和 Cloudflare R2。
- 同步扫描和差异处置均由管理员手动触发，不包含定时任务和云事件通知。
- 数据库仅保存对象元数据，不保存文件正文。

对象键中的 `/` 用于构建虚拟目录。例如 `manual/images/logo.png` 会显示为 `manual / images / logo.png`，对象存储中不会额外创建目录对象。

## 2. 启用配置

本地配置位于：

```text
j2agent/j2agent-starter/src/main/resources/application.yaml
```

生产环境配置位于：

```text
j2agent/j2agent-starter/src/main/resources/application-product.yaml
```

通用配置：

```yaml
j2agent:
  storage:
    enabled: true
    # minio、oss、qiniu、r2
    type: minio
    bucket: j2agent-files
    sync:
      retention-days: 7
      cleanup-cron: "0 30 2 * * *"
```

`enabled=false` 时不会创建对象存储服务和文件管理 REST Controller，前端调用文件接口会返回 404。

本地开发配置默认启用 MinIO。MinIO 适配器首次访问默认 Bucket 时会检查 Bucket 是否存在，不存在则自动创建。阿里云 OSS、七牛云 Kodo 和 Cloudflare R2 的 Bucket 仍需提前创建。

### 2.1 MinIO

```yaml
j2agent:
  storage:
    enabled: true
    type: minio
    bucket: j2agent-files
    minio:
      endpoint: http://127.0.0.1:19000
      access-key: minioadmin
      secret-key: change-me
```

### 2.2 阿里云 OSS

```yaml
j2agent:
  storage:
    enabled: true
    type: oss
    bucket: your-bucket
    oss:
      endpoint: https://oss-cn-hangzhou.aliyuncs.com
      access-key-id: ${OSS_ACCESS_KEY_ID}
      access-key-secret: ${OSS_ACCESS_KEY_SECRET}
```

`endpoint` 应填写 Bucket 所在地域的 OSS Endpoint。运行环境需要具备列举对象、读取元数据、上传、下载签名和删除对象权限。

### 2.3 七牛云 Kodo

```yaml
j2agent:
  storage:
    enabled: true
    type: qiniu
    bucket: your-bucket
    qiniu:
      access-key: ${QINIU_ACCESS_KEY}
      secret-key: ${QINIU_SECRET_KEY}
      domain: files.example.com
      use-https: true
```

`domain` 是默认 Bucket 绑定的下载域名，可以带或不带协议。当前实现会移除协议和末尾 `/`，再按 `use-https` 生成私有下载 URL。

### 2.4 Cloudflare R2

```yaml
j2agent:
  storage:
    enabled: true
    type: r2
    bucket: your-bucket
    r2:
      endpoint: https://<account-id>.r2.cloudflarestorage.com
      access-key-id: ${R2_ACCESS_KEY_ID}
      secret-access-key: ${R2_SECRET_ACCESS_KEY}
```

R2 使用 S3 兼容接口，访问密钥需要允许目标 Bucket 的对象读写、列举和删除。

### 2.5 Docker 环境变量

`j2agent/docker/.env` 可配置：

```dotenv
J2AGENT_STORAGE_ENABLED=true
J2AGENT_STORAGE_TYPE=minio
J2AGENT_STORAGE_BUCKET=j2agent-files
J2AGENT_MINIO_ENDPOINT=http://minio:9000

OSS_ENDPOINT=
OSS_ACCESS_KEY_ID=
OSS_ACCESS_KEY_SECRET=

QINIU_ACCESS_KEY=
QINIU_SECRET_KEY=
QINIU_DOMAIN=
QINIU_USE_HTTPS=true

R2_ENDPOINT=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
```

只需要填写当前 `J2AGENT_STORAGE_TYPE` 对应的一组供应商参数。Bucket 需提前创建。

## 3. 数据库迁移

升级已有环境时，Flyway 会执行：

```text
V0_2__object_storage_file_management.sql
```

新增三张表：

| 表 | 作用 |
|---|---|
| `object_file` | 文件台账，以 `bucket_name + object_key_hash` 唯一 |
| `object_storage_sync_task` | 持久化扫描任务、进度、统计和失败原因 |
| `object_storage_sync_diff` | 保存扫描时两侧元数据快照和处置结果 |

`object_key_hash` 是对象键的 SHA-256，仅用于建立稳定且长度可控的唯一索引；原始对象键仍完整保存在 `object_key`。

文件操作状态：

| 状态 | 说明 |
|---|---|
| `UPLOADING` | 已写入数据库占位记录，正在上传对象 |
| `READY` | 对象和数据库台账均已就绪 |
| `DELETING` | 已标记删除中，正在删除对象 |
| `ERROR` | 上传或删除失败，可查看 `last_error` 后重试或同步处理 |

## 4. 文件操作

### 上传

1. 根据当前虚拟目录和原始文件名生成对象键。
2. 同时检查数据库台账和对象存储；任一侧已存在同名对象即返回 `409 Conflict`，不会覆盖。
3. 数据库先写入 `UPLOADING` 状态。
4. 上传完成后读取对象存储元数据，并更新为 `READY`。
5. 上传失败时保留台账并标记 `ERROR`，记录错误原因。

### 删除

1. 数据库先标记为 `DELETING`。
2. 删除对象存储中的对象。
3. 删除数据库台账。
4. 任一步失败时将台账标记为 `ERROR`，批量删除会返回失败的对象键，其余对象继续处理。

### 预览与下载

后端生成有效期 15 分钟的签名 URL。图片、PDF、文本、音视频等由浏览器直接预览，其他类型由浏览器或对象存储响应决定是否下载。

## 5. OSS 与数据库同步

管理员在 `/files` 页面的“同步检查”区域手动创建扫描任务。任务状态为：

- `PENDING`
- `RUNNING`
- `SUCCESS`
- `FAILED`

同一个 Bucket 同时只允许一个 `PENDING` 或 `RUNNING` 任务，重复提交返回 `409 Conflict`。扫描使用分页列举，默认每页读取 500 个对象，并持续更新任务进度。

同步任务和差异明细默认保留 7 天。系统每天凌晨 2:30 清理超过保留期的 `SUCCESS`、`FAILED` 任务，先删除差异明细再删除任务；`PENDING`、`RUNNING` 任务不会被清理。可通过 `j2agent.storage.sync.retention-days` 和 `j2agent.storage.sync.cleanup-cron` 调整。

### 5.1 差异判定

| 类型 | 判定 |
|---|---|
| `IN_SYNC` | 两侧存在，ETag、大小和对象修改时间一致 |
| `OSS_ONLY` | 仅对象存储存在 |
| `DB_ONLY` | 仅数据库台账存在 |
| `METADATA_MISMATCH` | 对象键一致，但元数据不同 |

ETag 比较前会去除首尾引号。最后修改时间按秒归一化后比较，以兼容对象列表接口和元数据接口不同的时间精度。同步不会下载文件，也不会重新计算文件 SHA-256。

### 5.2 人工处置

| 差异 | 可执行动作 | 结果 |
|---|---|---|
| `OSS_ONLY` | `REGISTER_DB` | 读取 OSS 当前元数据并写入数据库 |
| `OSS_ONLY` | `DELETE_OSS` | 删除 OSS 对象 |
| `DB_ONLY` | `DELETE_DB` | 删除数据库孤儿记录 |
| `METADATA_MISMATCH` | `UPDATE_DB` | 使用 OSS 当前元数据覆盖数据库 |
| `METADATA_MISMATCH` | `DELETE_BOTH` | 删除 OSS 对象和数据库记录 |

`DB_ONLY` 无法恢复对象正文，因此不提供“恢复到 OSS”操作。

执行处置前，后端会重新读取 OSS 和数据库当前状态，并与扫描时快照比较。扫描后任一侧发生变化时，拒绝执行并将差异标记为 `STALE`，管理员需要重新扫描。

处置结果状态：

- `PENDING`：等待人工处理。
- `RESOLVED`：处置成功。
- `STALE`：扫描结果已过期。
- `FAILED`：处置失败，可查看错误后重试。

## 6. REST API

所有接口均位于 `/v1/rest/j2agent/files`，并要求管理员权限。

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/files` | 分页查询文件和虚拟目录 |
| `POST` | `/files` | 上传文件 |
| `DELETE` | `/files?object-key=...` | 删除单个文件 |
| `POST` | `/files/delete-batch` | 批量删除文件 |
| `GET` | `/files/preview?object-key=...` | 获取短期签名 URL |
| `POST` | `/files/sync/tasks` | 创建异步扫描任务 |
| `GET` | `/files/sync/tasks/{task-id}` | 查询任务进度 |
| `GET` | `/files/sync/tasks/{task-id}/diffs` | 分页查询差异 |
| `POST` | `/files/sync/resolve` | 批量执行同一种处置动作 |

详细请求和响应模型以 `j2agent-model/src/main/resources/openapi-interface.yaml` 和 `openapi-model.yaml` 为准。

## 7. 页面与权限

- 路由：`/#/files`
- 角色：`ROLE_ADMIN`
- 入口：首页文件管理卡片、系统菜单文件管理项。
- 页面区域：文件列表、同步检查。
- 删除 OSS 或两侧数据前会弹出二次确认。

前端项目未全局注册 Element Plus 组件。新增或拆分文件管理页面时，需要在 Vue 组件中显式导入模板所用的 `ElButton`、`ElTable`、`ElMenuItem`、`ElPagination` 等组件，否则页面可能无法正确渲染。

## 8. 验收与排查

部署验收建议按以下顺序执行：

1. 确认数据库迁移完成，三张表存在。
2. 确认 Bucket 已创建，应用凭证具备所需权限。
3. 启用 `j2agent.storage.enabled` 并重启后端。
4. 使用管理员账号打开 `/#/files`。
5. 上传测试文件，确认数据库状态最终为 `READY`。
6. 在云控制台新增一个对象，扫描后确认出现 `OSS_ONLY`。
7. 删除一个云端对象，扫描后确认出现 `DB_ONLY`。
8. 修改云端对象，扫描后确认出现 `METADATA_MISMATCH`。
9. 分别验证登记、删除和元数据覆盖动作。

常见问题：

- 页面接口返回 404：检查 `j2agent.storage.enabled` 是否为 `true`。
- 页面入口不可见或路由被拦截：确认当前用户角色为管理员。
- 页面纯白且控制台提示 `Failed to resolve component`：检查页面是否显式导入了所用 Element Plus 组件。
- 签名 URL 无法访问：检查 Endpoint、下载域名、HTTPS 配置和 Bucket 权限。
- 扫描一直失败：查看 `object_storage_sync_task.error_message` 和后端日志。
- 差异处置变为 `STALE`：对象或台账在扫描后发生变化，重新扫描后再处理。

自动化测试使用模拟对象存储，不依赖真实云凭证。MinIO、阿里云 OSS、七牛云 Kodo 和 Cloudflare R2 的真实凭证联调属于部署验收项。
