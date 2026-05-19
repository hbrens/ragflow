# 鸿蒙知识库批量上传和更新技术文档

## 1. 目标

本文档说明如何把鸿蒙知识库资料批量上传到 RAGFlow，并在后续资料变化时做增量更新。

本方案采用低维护成本方式：

- 原始资料直接入库，不做复杂预处理。
- 脚本负责批量上传、去重、更新识别和生成报告。
- 文档解析优先在 RAGFlow 界面一键操作。
- 后续如需自动化，可再调用 RAGFlow 解析 API。

## 2. 整体流程

```text
准备数据源目录
  -> 配置 dataset 映射
  -> 执行同步脚本
  -> 上传新增文件
  -> 删除并重传变化文件
  -> 输出同步报告
  -> 在 RAGFlow 界面一键解析
  -> 抽样检索验证
```

## 3. 数据目录约定

建议把不同来源资料拆成独立目录，目录名和 RAGFlow dataset 对应。

```text
data/raw/
  hm-internal-guides/
  hm-project-code/
  hm-deveco-sdk-api/
  hm-harmonyos-docs/
  hm-openharmony-docs/
  hm-harmonyos-samples/
```

每个目录对应一个 RAGFlow dataset：

| 本地目录 | RAGFlow dataset |
| --- | --- |
| `data/raw/hm-internal-guides` | `hm-internal-guides` |
| `data/raw/hm-project-code` | `hm-project-code` |
| `data/raw/hm-deveco-sdk-api` | `hm-deveco-sdk-api` |
| `data/raw/hm-harmonyos-docs` | `hm-harmonyos-docs` |
| `data/raw/hm-openharmony-docs` | `hm-openharmony-docs` |
| `data/raw/hm-harmonyos-samples` | `hm-harmonyos-samples` |

第一阶段不要求调整原始文件内容。文件可以是 Markdown、TXT、PDF、DOCX、代码文件、`.d.ts`、配置文件等。具体可上传格式以 RAGFlow 当前版本支持为准。

## 4. 配置文件

建议维护一个同步配置，例如 `harmony_kb_ingest.yaml`：

```yaml
ragflow:
  base_url: "http://ragflow.internal:9380"
  api_key_env: "RAGFLOW_API_KEY"

sources:
  - name: hm-internal-guides
    path: data/raw/hm-internal-guides
    dataset_id: "填写 RAGFlow dataset id"
    include:
      - "**/*.md"
      - "**/*.txt"
      - "**/*.docx"
      - "**/*.pdf"
    exclude:
      - "**/.git/**"
      - "**/~$*"

  - name: hm-deveco-sdk-api
    path: data/raw/hm-deveco-sdk-api
    dataset_id: "填写 RAGFlow dataset id"
    include:
      - "**/*.d.ts"
      - "**/*.md"
      - "**/*.txt"
    exclude:
      - "**/.git/**"

  - name: hm-harmonyos-docs
    path: data/raw/hm-harmonyos-docs
    dataset_id: "填写 RAGFlow dataset id"
    include:
      - "**/*.md"
      - "**/*.mdx"
      - "**/*.txt"
    exclude:
      - "**/progress.json"
      - "**/*_failed.json"
      - "**/.git/**"
```

`dataset_id` 可以在 RAGFlow 页面中查看，也可以通过 API 查询 dataset 列表后填入。

API Key 不写入配置文件，统一通过环境变量提供：

```bash
export RAGFLOW_API_KEY="ragflow-xxxx"
```

## 5. Manifest 机制

批量更新的核心是 manifest。脚本每次同步后记录本地文件和 RAGFlow document 的关系。

建议文件位置：

```text
data/.ragflow-manifest/
  hm-internal-guides.json
  hm-deveco-sdk-api.json
  hm-harmonyos-docs.json
```

manifest 示例：

```json
{
  "source": "hm-deveco-sdk-api",
  "dataset_id": "dataset_xxx",
  "updated_at": "2026-05-20T10:00:00+08:00",
  "files": {
    "openharmony/ets/api/@ohos.router.d.ts": {
      "hash": "sha256:abc...",
      "size": 12345,
      "mtime": "2026-05-20T09:30:00+08:00",
      "document_id": "document_xxx",
      "document_name": "@ohos.router.d.ts",
      "status": "uploaded"
    }
  }
}
```

manifest 用来判断：

- 本地新增文件：上传。
- 本地文件 hash 未变：跳过。
- 本地文件 hash 变化：删除旧 document，再上传新文件。
- 本地文件消失：删除旧 document 或标记为过期。

第一阶段建议采用“删除旧 document，再上传新文件”的更新方式。这样最简单，也能避免新旧内容同时被检索到。

## 6. RAGFlow API

### 6.1 批量上传文档

接口：

```http
POST /api/v1/datasets/{dataset_id}/documents
Authorization: Bearer {api_key}
Content-Type: multipart/form-data
```

表单字段：

| 字段 | 说明 |
| --- | --- |
| `file` | 可传多个文件 |
| `parent_path` | 可选，用于保留目录层级 |

curl 示例：

```bash
curl -X POST "http://ragflow.internal:9380/api/v1/datasets/${DATASET_ID}/documents" \
  -H "Authorization: Bearer ${RAGFLOW_API_KEY}" \
  -F "file=@data/raw/hm-internal-guides/router.md" \
  -F "file=@data/raw/hm-internal-guides/state.md" \
  -F "parent_path=hm-internal-guides"
```

返回结果中会包含 document id。同步脚本需要把 document id 写入 manifest。

### 6.2 查询文档列表

接口：

```http
GET /api/v1/datasets/{dataset_id}/documents
Authorization: Bearer {api_key}
```

常用参数：

| 参数 | 说明 |
| --- | --- |
| `page` | 页码 |
| `page_size` | 每页数量 |
| `name` | 按文件名过滤 |
| `keywords` | 按关键词过滤 |

用途：

- 初始化 manifest。
- 排查某个文件是否已经上传。
- 同步脚本异常后做恢复。

### 6.3 删除文档

接口：

```http
DELETE /api/v1/datasets/{dataset_id}/documents
Authorization: Bearer {api_key}
Content-Type: application/json
```

请求体：

```json
{
  "ids": ["document_xxx"],
  "delete_all": false
}
```

更新文件时，脚本先删除 manifest 中记录的旧 `document_id`，再上传新文件，并写入新的 `document_id`。

### 6.4 解析文档

界面操作优先：

1. 进入 RAGFlow dataset。
2. 确认批量上传的文件已经出现在文档列表。
3. 全选本次新增或更新的文档。
4. 点击解析或一键解析。
5. 等待状态变为完成。

RAGFlow 也提供解析 API，可作为后续自动化补充：

```http
POST /api/v1/datasets/{dataset_id}/documents/parse
Authorization: Bearer {api_key}
Content-Type: application/json
```

请求体：

```json
{
  "document_ids": ["document_xxx", "document_yyy"]
}
```

第一阶段建议先使用界面解析，因为更直观，失败文件也容易人工查看。

## 7. 同步脚本逻辑

脚本主流程：

```text
读取配置
  -> 遍历每个 source
  -> 加载 source manifest
  -> 扫描本地文件
  -> 根据 include/exclude 过滤
  -> 计算 sha256
  -> 对比 manifest
  -> 生成新增、更新、删除、跳过列表
  -> 删除变化文件对应的旧 document
  -> 上传新增和变化文件
  -> 更新 manifest
  -> 输出同步报告
```

伪代码：

```python
for source in config.sources:
    manifest = load_manifest(source.name)
    local_files = scan_files(source.path, source.include, source.exclude)

    added = []
    changed = []
    skipped = []
    removed = []

    for file in local_files:
        rel_path = relative_path(file, source.path)
        file_hash = sha256(file)
        old = manifest.files.get(rel_path)

        if old is None:
            added.append(file)
        elif old["hash"] != file_hash:
            changed.append(file)
        else:
            skipped.append(file)

    for rel_path in manifest.files:
        if rel_path not in local_files:
            removed.append(rel_path)

    for file in changed:
        old_doc_id = manifest.files[rel_path]["document_id"]
        delete_document(source.dataset_id, old_doc_id)
        new_doc = upload_document(source.dataset_id, file)
        manifest.files[rel_path] = build_manifest_entry(file, new_doc)

    for file in added:
        new_doc = upload_document(source.dataset_id, file)
        manifest.files[rel_path] = build_manifest_entry(file, new_doc)

    for rel_path in removed:
        old_doc_id = manifest.files[rel_path]["document_id"]
        delete_document(source.dataset_id, old_doc_id)
        del manifest.files[rel_path]

    save_manifest(source.name, manifest)
```

## 8. 更新策略

### 8.1 新增

本地新增文件时：

1. 上传到对应 dataset。
2. 写入 manifest。
3. 报告中标记为 `added`。
4. 后续在界面解析。

### 8.2 修改

本地文件 hash 变化时：

1. 根据 manifest 找到旧 document id。
2. 删除旧 document。
3. 上传新文件。
4. 更新 manifest。
5. 报告中标记为 `changed`。
6. 后续在界面解析新 document。

### 8.3 删除

本地文件删除时：

第一阶段推荐同步删除 RAGFlow document，避免旧内容继续被检索。

如果担心误删，可以先改成“报告 removed，但不自动删除”，人工确认后再执行删除。

建议脚本支持参数：

```bash
--delete-removed true
--delete-removed false
```

默认可以设为 `false`，正式跑通后再改为 `true`。

## 9. 同步报告

每次同步输出一份报告：

```text
source: hm-deveco-sdk-api
dataset_id: dataset_xxx

added: 12
changed: 3
removed: 1
skipped: 1068
uploaded: 15
deleted: 4
failed: 0

new_or_changed_document_ids:
  - document_xxx
  - document_yyy

next_step:
  Open RAGFlow dataset and parse the new_or_changed documents.
```

建议同时保存 JSON 报告：

```text
data/.ragflow-reports/2026-05-20-hm-deveco-sdk-api.json
```

用途：

- 知道本次需要解析哪些文档。
- 出问题时能重试。
- 后续自动调用解析 API 时可直接读取 document ids。

## 10. 操作步骤

### 10.1 首次入库

1. 在 RAGFlow 创建 datasets：
   - `hm-internal-guides`
   - `hm-project-code`
   - `hm-deveco-sdk-api`
   - `hm-harmonyos-docs`
   - `hm-openharmony-docs`
   - `hm-harmonyos-samples`
2. 在配置文件中填入 dataset id。
3. 准备本地数据目录。
4. 设置 API Key：

```bash
export RAGFLOW_API_KEY="ragflow-xxxx"
```

5. 执行同步脚本：

```bash
python scripts/harmony_kb_sync.py --config harmony_kb_ingest.yaml
```

6. 查看同步报告。
7. 打开 RAGFlow 界面，进入对应 dataset。
8. 选择本次新增的文档，点击一键解析。
9. 等待解析完成。
10. 用几个典型问题做检索验证。

### 10.2 后续更新

1. 更新 `data/raw/<source>` 下的资料。
2. 执行同步脚本。
3. 查看报告中的 `added`、`changed`、`removed`。
4. 打开 RAGFlow 界面。
5. 解析新增和变化的文档。
6. 抽样验证高频问题。

## 11. 脚本参数建议

```bash
python scripts/harmony_kb_sync.py \
  --config harmony_kb_ingest.yaml \
  --source hm-deveco-sdk-api \
  --dry-run
```

参数：

| 参数 | 说明 |
| --- | --- |
| `--config` | 配置文件路径 |
| `--source` | 只同步某个 source，不传则同步全部 |
| `--dry-run` | 只计算差异，不上传、不删除 |
| `--delete-removed` | 本地删除后是否同步删除 RAGFlow document |
| `--max-files` | 限制本次最多上传文件数，便于试跑 |
| `--report-dir` | 报告输出目录 |

首次执行建议使用：

```bash
python scripts/harmony_kb_sync.py --config harmony_kb_ingest.yaml --dry-run
```

确认差异无误后再正式上传。

## 12. 注意事项

1. 不要每次全量重复上传，否则旧内容和新内容会同时被检索到。
2. 更新文件时使用“删除旧 document + 上传新 document”的方式。
3. API Key 只放环境变量，不提交到 Git。
4. 首次入库建议按 source 分批上传，避免一次失败难排查。
5. RAGFlow 界面解析完成前，文档不会进入稳定可检索状态。
6. 删除本地文件是否同步删除 RAGFlow document，首次建议人工确认。
7. 如果大量文件解析失败，先检查文件类型、大小、编码和 RAGFlow 支持范围。

## 13. 后续自动化扩展

第一阶段解析由界面一键完成。稳定后可以继续扩展：

- 同步脚本自动调用 `/documents/parse`。
- 解析后轮询文档状态。
- 自动生成失败文件清单。
- 自动跑一组检索测试问题。
- 将同步报告发到群里。
- 每周定时同步自有文档。

这些都不是第一阶段必需项。第一阶段先把批量上传、更新识别、manifest 和界面解析流程跑通。
