---
sidebar_position: 11
slug: /update_documents
sidebar_custom_props: {
  categoryIcon: LucideRefreshCw
}
---
# Update parsed documents

When the content of a document changes, RAGFlow does not automatically perform a content-level diff and update only the changed chunks. Treat the updated file as a document-level re-ingestion unless you are making small manual edits to existing chunks.

## Recommended workflow

For local files, use one of the following workflows:

- Delete the old document, upload the updated file, then parse the new document.
- Upload the updated file first, parse it successfully, then delete the old document to avoid a retrieval gap.

Uploading a file with the same display name does not overwrite the existing document in normal local-upload flows. RAGFlow creates a new document entry and generates a non-conflicting name if needed.

## Reparse an existing document

If the original stored file is still the correct file and you only need to rebuild its chunks and indexes, reparse the document.

In the UI, click **Parse** for the document. If the document already has chunks, RAGFlow asks whether to clear the existing chunks. Keep this option selected when you want to rebuild the document index from scratch.

Using the REST API:

```bash
curl --request POST \
     --url http://{address}/api/v1/datasets/{dataset_id}/documents/parse \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer <YOUR_API_KEY>' \
     --data '{
       "document_ids": ["<document_id>"]
     }'
```

When reparsing a completed document, RAGFlow clears the document's previous chunks from the document store, resets chunk and token counters, deletes existing parse tasks, and queues new parse tasks.

## Replace a changed file

If the source file itself changed, upload the new file and parse the new document:

```bash
curl --request POST \
     --url http://{address}/api/v1/datasets/{dataset_id}/documents \
     --header 'Authorization: Bearer <YOUR_API_KEY>' \
     --form 'file=@./updated.pdf'
```

Then parse the uploaded document:

```bash
curl --request POST \
     --url http://{address}/api/v1/datasets/{dataset_id}/documents/parse \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer <YOUR_API_KEY>' \
     --data '{
       "document_ids": ["<new_document_id>"]
     }'
```

After the new document is parsed and validated, delete the old document:

```bash
curl --request DELETE \
     --url http://{address}/api/v1/datasets/{dataset_id}/documents \
     --header 'Content-Type: application/json' \
     --header 'Authorization: Bearer <YOUR_API_KEY>' \
     --data '{
       "ids": ["<old_document_id>"]
     }'
```

## Manual chunk-level edits

For small corrections, you can update chunks directly instead of uploading and parsing the whole file again. This is useful for fixing text, keywords, questions, tags, or availability on known chunks.

Use the chunk APIs to add, update, delete, or disable chunks under a document. This is a manual chunk-level operation; it does not re-extract structure from the original file and does not keep the stored source file in sync with your edits.

## About incremental reuse

RAGFlow has task-level reuse for some parsing tasks. For example, parse tasks can be reused when their page range and parsing configuration digest match previous completed tasks.

This is an internal optimization, not a general content-diff incremental update feature. The task digest includes document properties such as the content hash, so when the document content changes, previous task chunks normally cannot be reused.

## Summary

- Changed source file: upload the updated file as a new document, parse it, then delete the old document.
- Same source file, rebuild needed: reparse the existing document and clear previous chunks.
- Small text or metadata correction: edit the affected chunks manually.
- Automatic content-level incremental update is not currently available for normal document uploads.
