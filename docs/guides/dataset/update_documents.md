---
sidebar_position: 11
slug: /update_documents
sidebar_custom_props: {
  categoryIcon: LucideRefreshCw
}
---
# 文档内容更新后的处理方式

当一篇文档的内容发生变化时，RAGFlow 目前不会自动对新旧文件做内容 diff，也不会只更新发生变化的 chunk。常规上传场景下，应把更新后的文件视为一次“文档级重新入库”。

## 结论

- 源文件内容变了：上传新版文档，解析成功后删除旧文档。
- 原文件没变，只想重建索引：对已有文档重新解析，并清空旧 chunk。
- 只改少量文字、关键词、问题、标签或可用状态：可以手工编辑对应 chunk。
- 普通本地上传流程没有自动的内容级增量更新能力。

## 推荐流程

本地文件更新后，推荐二选一：

- 先删除旧文档，再上传新版文档并解析。
- 先上传新版文档并解析，确认检索正常后再删除旧文档。这个方式可以避免知识库短时间没有可用内容。

需要注意：在普通本地上传流程中，上传同名文件不会覆盖已有文档。RAGFlow 会创建一个新的文档记录，并在需要时生成一个不冲突的文件名。

## 重新解析已有文档

如果 RAGFlow 中保存的原始文件仍然是正确版本，只是需要重新生成 chunks 和索引，可以直接重新解析已有文档。

在界面上，点击文档的 **Parse / 解析**。如果该文档已经有 chunks，RAGFlow 会提示是否清空已有 chunks。要重建该文档索引时，保持这个选项勾选。

REST API 示例：

```bash
curl --request POST \
     --url http://{address}/api/v1/datasets/{dataset_id}/documents/parse \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer <YOUR_API_KEY>' \
     --data '{
       "document_ids": ["<document_id>"]
     }'
```

对已经解析完成的文档重新解析时，RAGFlow 会删除该文档在文档存储中的旧 chunks，重置 chunk 和 token 计数，删除旧解析任务，并重新创建解析任务。

## 替换已经变化的源文件

如果源文件本身已经发生变化，建议把新版文件作为新文档上传并解析：

```bash
curl --request POST \
     --url http://{address}/api/v1/datasets/{dataset_id}/documents \
     --header 'Authorization: Bearer <YOUR_API_KEY>' \
     --form 'file=@./updated.pdf'
```

然后解析新上传的文档：

```bash
curl --request POST \
     --url http://{address}/api/v1/datasets/{dataset_id}/documents/parse \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer <YOUR_API_KEY>' \
     --data '{
       "document_ids": ["<new_document_id>"]
     }'
```

确认新版文档解析完成、检索效果正常后，再删除旧文档：

```bash
curl --request DELETE \
     --url http://{address}/api/v1/datasets/{dataset_id}/documents \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer <YOUR_API_KEY>' \
     --data '{
       "ids": ["<old_document_id>"]
     }'
```

## 手工 chunk 级更新

如果只是小范围修正，可以不重新上传和解析整篇文档，而是直接修改对应 chunks。适合的场景包括：

- 修改某个 chunk 的文本内容。
- 增删关键词、问题或标签。
- 删除不需要的 chunks。
- 启用或禁用某些 chunks。

这种方式属于手工 chunk 级维护，不会重新从原始文件中抽取结构，也不会同步修改 RAGFlow 中保存的源文件。

## 关于“增量更新”

RAGFlow 内部存在任务级复用机制。例如某些解析任务在页码范围和解析配置 digest 与历史完成任务匹配时，可以复用之前的 chunks。

但这只是内部优化，不是通用的内容 diff 增量更新。任务 digest 会包含文档属性，例如内容 hash；当文档内容发生变化时，历史解析任务通常无法复用。

因此，对普通文档上传和解析而言，不应把它理解为支持“只更新变化段落”的增量解析。
