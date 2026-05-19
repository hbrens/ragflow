# 鸿蒙知识库建设方案讨论稿

## 0. 会议讨论摘要

本次讨论建议重点确认以下决策：

1. RAGFlow 是否只作为内网 RAG 检索和管理系统，不承担匿名公开文档站能力。
2. 回答中的引用链接是否优先使用内部文档站、Git Web、静态文件链接或源文件路径，而不是强依赖 RAGFlow 页面。
3. 团队工具是否统一经过 MCP 审计网关接入，避免 Claude Code、opencode、Nanobot 各自直连造成日志不完整。
4. 是否要求记录最终回答。如果要求，需要统一问答网关或客户端回传，因为单纯 MCP 检索层通常只能记录问题和检索结果。
5. 数据源是否按来源拆分 dataset，并用 manifest 脚本做周期更新，避免重复新增旧文档。
6. 第一阶段是否只支持文本、代码和 Markdown，图片先以链接和人工文字说明处理。
7. Embedding 第一阶段是否固定使用内网已有的 Qwen3-Embedding-8B，先通过评测集验证效果，再考虑 rerank 或换模型。

## 1. 背景和目标

团队希望建设一个面向 HarmonyOS / OpenHarmony / ArkTS / DevEco / HMS 开发的内部知识库，解决日常编码中 API 签名不准、版本差异不清、官方文档分散、团队经验难沉淀的问题。

本方案以 RAGFlow 作为知识库的 RAG 数据库和检索引擎，通过 MCP 服务提供给 Nanobot、Claude Code、opencode 等工具调用，并通过审计链路记录团队成员的提问、检索命中、回答结果和反馈，用于后续优化文档和检索策略。

核心目标：

1. 建立可持续更新的鸿蒙知识库数据源。
2. 使用 RAGFlow 完成文档录入、解析、切分、Embedding、检索。
3. 通过 MCP 给团队编码工具统一调用。
4. 记录提问、检索结果、回答和用户反馈，形成优化闭环。
5. 支持内网离线部署和周期性数据更新。
6. 明确 RAGFlow 网页是否可公开只读访问，以及替代方案。

## 2. 总体结论

建议采用以下架构：

```text
外部/内部数据源
  -> 数据同步脚本
  -> RAGFlow 数据集
  -> 文档解析 / Chunk / Qwen3-Embedding-8B
  -> MCP 审计网关
  -> Nanobot / Claude Code / opencode
  -> 提问、检索、回答、反馈日志
  -> 文档补充和检索参数优化
```

RAGFlow 负责知识库管理、解析、索引和检索；MCP 审计网关负责团队统一入口、权限控制、审计记录和结果包装；Skill 或工具提示词负责约束编码工具在鸿蒙相关问题上主动调用知识库。

## 3. 当前 RAGFlow 能力判断

### 3.1 MCP 能力

当前 RAGFlow 已提供 MCP Server，支持两种模式：

- `self-host`：MCP Server 持有一个 RAGFlow API Key，客户端不用分别配置 RAGFlow API Key。适合团队共用只读检索入口。
- `host`：每个 MCP Client 请求时携带自己的 RAGFlow API Key。适合按个人权限隔离数据集。

MCP Server 当前主要提供 `ragflow_retrieval` 工具，用于调用 RAGFlow retrieval 接口检索 dataset 和 document 中的 chunk。

本项目建议优先使用 `self-host`，由服务端保存一个只用于检索的 RAGFlow API Key，再在网关层限制 MCP 只暴露检索工具。

### 3.2 RAGFlow 权限和公开访问

当前 RAGFlow 数据集权限主要是：

- `me`：仅创建者可访问。
- `team`：团队成员可访问。

从当前代码和文档看，不能假设 RAGFlow 原生支持“免登录、匿名、只读访问某个文档或 chunk 页面”的公开模式。RAGFlow 的 API 和 MCP 都依赖登录态或 API Key。

因此，“回答里附 RAG 网站链接”需要分两种情况处理：

1. **RAGFlow 内部链接**：只给已经登录 RAGFlow 的团队成员使用，可用于跳转到 dataset、document 或 chunk 页面。
2. **只读资料源链接**：更推荐。回答中附内部文档站、Git Web、静态 Markdown 站、对象存储只读链接或源文件路径。这类链接不依赖 RAGFlow 登录，更适合做引用来源。

建议不要直接暴露 RAGFlow 给匿名用户。若需要外部只读页面，应单独建设“知识来源只读展示站”，RAGFlow 只作为检索后端。

## 4. 数据源设计

参考 `hm-local-kb` 的数据源清单，建议按来源拆分数据集，而不是全部混入一个数据集。这样便于控制优先级、更新周期、检索范围和问题定位。

### 4.1 建议数据集

| 数据集 | 内容 | 优先级 | 更新方式 |
| --- | --- | --- | --- |
| `hm-internal-guides` | 团队编码规范、FAQ、踩坑记录、封装约定、项目约定 | 最高 | 每周或 Git 合并后 |
| `hm-project-code` | 团队真实项目代码、组件库、页面模板 | 高 | 每周或版本发布后 |
| `hm-deveco-sdk-api` | DevEco SDK 中 OpenHarmony / HMS 的 `.d.ts` API 签名 | 高 | DevEco / SDK 升级后 |
| `hm-harmonyos-docs` | HarmonyOS Next 官方文档 Markdown 快照 | 中高 | 每月或重要版本后 |
| `hm-openharmony-docs` | OpenHarmony 开源文档 | 中 | 每月或按需 |
| `hm-harmonyos-samples` | OpenHarmony / HarmonyOS 样例代码、Codelabs | 中 | 季度或重要版本后 |

### 4.2 数据源优先级

回答时建议按以下顺序使用证据：

1. 团队内部文档和项目约定。
2. DevEco SDK `.d.ts` API 签名。
3. HarmonyOS Next 官方文档。
4. OpenHarmony docs。
5. 官方或开源 Samples。

原因：

- 团队内部规范决定“本团队怎么写”。
- `.d.ts` 决定 API、参数、枚举、返回值是否真实存在。
- 官方文档解释概念、生命周期、使用场景和注意事项。
- Samples 证明代码组合方式和实际工程结构。

## 5. 数据录入和解析策略

### 5.1 文件类型和处理方式

| 类型 | 示例 | 建议处理 |
| --- | --- | --- |
| `.d.ts` | `@ohos.*.d.ts`, `@kit.*.d.ts` | 按模块、接口、函数、类、枚举切分，保留 API 名和路径 metadata |
| Markdown | 官方文档、内部规范 | 按标题层级切分，保留标题路径、来源链接、版本 |
| ArkTS / TS / ETS | 示例代码、项目代码 | 按文件和代码块切分，保留路径、模块、组件名 |
| JSON / 配置文件 | 示例配置、工程配置 | 保留文件路径和上下文，避免过碎 |
| PDF / 图片 | 架构图、截图类资料 | 第一阶段不作为重点，必要时人工补 Markdown 说明 |

### 5.2 RAGFlow Chunk 方法建议

RAGFlow 内置多种 chunk 方法。第一阶段建议保守使用：

- Markdown / 文本文档：使用 `naive` 或适合文档的通用切分策略。
- `.d.ts` / 代码文件：先按文本方式导入，但同步脚本应尽量预处理为更适合检索的 Markdown 片段。
- 表格 FAQ：可以整理为 Markdown 或 CSV，再评估是否用 Q&A / table 策略。
- 图片 / PDF：非核心资料先不强依赖 OCR 和图片解析。

对 `.d.ts` 不建议直接把大文件原样交给通用 chunk。更好的方式是同步脚本生成结构化 Markdown，例如：

```markdown
# @ohos.router - pushUrl

source: deveco-api-openharmony
file: openharmony/ets/api/@ohos.router.d.ts
kind: function
symbol: pushUrl

```ts
function pushUrl(options: RouterOptions, callback: AsyncCallback<void>): void;
```

说明：
- 所属模块：@ohos.router
- 用途：页面路由跳转
```
```

这样可以显著提升 API 签名类问题的召回稳定性。

### 5.3 Metadata 建议

每个导入文档或 chunk 应尽量带 metadata：

- `source`：数据源分桶，如 `deveco-api-openharmony`。
- `source_type`：`sdk_dts`、`official_doc`、`sample_code`、`internal_doc`。
- `version`：SDK 或文档版本。
- `path`：源文件路径。
- `url`：源文档只读链接，没有则为空。
- `title`：文档标题或符号名。
- `module`：ArkTS 模块名，如 `@kit.ArkUI`。
- `symbol`：API、类、函数、组件名。
- `owner`：内部资料维护人。
- `updated_at`：源文件更新时间。

这些 metadata 后续可用于筛选来源、生成引用、做审计分析和定位低质量文档。

## 6. Embedding 和检索策略

### 6.1 模型选择

当前内网离线环境已有 `Qwen3-Embedding-8B`，第一阶段建议直接作为主 embedding 模型，不在方案初期引入复杂多模型对比。

原因：

- 内网离线部署下模型选择受限。
- 鸿蒙知识库主要是中文技术文档、ArkTS API 和代码片段，Qwen 系列 embedding 具备较好的中文语义基础。
- RAGFlow 中 dataset 一旦产生 chunk，Embedding 模型不应随意切换；切换需要清空 chunk 后重新解析。

### 6.2 检索参数初始建议

初始参数建议：

- `top_k`: 20 到 50，用于控制候选召回。
- `page_size`: 5 到 10，用于返回给 LLM 的最终片段。
- `similarity_threshold`: 从 0.2 开始。
- `vector_similarity_weight`: 从 0.3 到 0.5 试验。
- `keyword`: 对 API 名、枚举、错误码、类名问题建议开启或提高关键词权重。

API 签名类问题不能只靠向量相似度。像 `UIAbility`、`router.pushUrl`、`PersistentStorage` 这类精确符号，关键词检索和 metadata 过滤很重要。

### 6.3 评测集

上线前建议准备 30 到 50 个评测问题，覆盖：

- API 签名：`router.pushUrl` 参数是什么。
- 生命周期：`UIAbility` 各生命周期函数区别。
- ArkUI 组件：`Navigation`、`Tabs`、`List` 的典型用法。
- 状态管理：`@State`、`@Prop`、`@Link`、`Observed` 区别。
- HMS Kit：账号、推送、地图等 Kit 的导入和调用方式。
- DevEco 工程问题：ohpm、签名、构建、module 配置。
- 团队规范：项目里页面跳转、网络请求、组件封装怎么写。

每个问题应记录期望命中来源和正确答案要点，用于后续比较导入策略和检索参数。

### 6.4 Rerank

第一阶段可以不强依赖 rerank。如果召回片段多但排序不稳定，再考虑增加离线 rerank 模型。

优先优化顺序：

1. 数据清洗和 chunk 结构。
2. metadata 和 source 优先级。
3. 检索参数。
4. rerank。

## 7. 图片和多模态处理

第一阶段不建议把图片作为核心能力。

原因：

- Claude Code、opencode 这类编码工具主要消费文本和代码。
- 团队主要问题是 API、代码、工程配置和规范，文本资料优先级更高。
- 图片 embedding / OCR 会增加部署复杂度和质量不确定性。

建议处理方式：

1. 文档中的图片保留原始链接或文件路径，作为引用。
2. 对关键流程图、架构图、配置截图，人工补 Markdown 文字说明。
3. 若后续发现大量问题必须依赖截图，再单独评估 OCR 或多模态解析。

## 8. MCP 接入方案

### 8.1 为什么需要 MCP

团队会通过 Nanobot、Claude Code、opencode 等工具使用知识库。MCP 可以把 RAGFlow 检索能力统一暴露成工具，避免每个客户端单独实现 RAGFlow API 调用。

### 8.2 为什么还需要 Skill

MCP 只解决“工具可调用”，不解决“什么时候调用”和“如何使用结果”。

因此需要配套 Skill 或工具提示词，例如 `harmony-kb`：

- 当问题涉及 HarmonyOS、OpenHarmony、ArkTS、ArkUI、DevEco、HMS 时，必须先调用知识库。
- 涉及 API 签名、参数、枚举时，优先查 DevEco SDK `.d.ts`。
- 涉及用法、生命周期、配置时，优先查官方文档和 Samples。
- 涉及团队项目约定时，优先查内部文档。
- 回答必须附来源，证据不足时明确说明不确定。

### 8.3 客户端接入

| 客户端 | 接入方式 | 注意事项 |
| --- | --- | --- |
| Nanobot | 统一接入 MCP 审计网关 | 最适合记录最终回答 |
| Claude Code | 配置 MCP Server + Skill | 能记录检索，最终回答需额外回传或统一代理 |
| opencode | 配置 MCP Server + Skill | 同上 |

如果希望稳定记录“最终回答”，推荐团队优先通过 Nanobot 或统一问答网关使用知识库。Claude Code / opencode 直连 MCP 时，MCP 服务只能稳定记录检索请求和返回内容，不能保证拿到模型最终回答。

## 9. MCP 审计网关设计

### 9.1 结论

建议在 RAGFlow MCP Server 前面套一层 MCP 审计网关，或者基于 RAGFlow MCP Server 做轻量 fork/包装。

原因：

- 需要统一团队入口。
- 需要限制只暴露检索工具，避免误开放上传、解析、删除能力。
- 需要记录问题、检索命中、返回内容、耗时和错误。
- 如果走统一问答网关，还可以记录最终回答。

### 9.2 最低可行审计

MCP 网关至少记录：

- `request_id`
- `user_id` 或 `client_id`
- `client_type`：Nanobot / Claude Code / opencode
- `question`
- `dataset_ids`
- `document_ids`
- `retrieval_params`
- `matched_chunks`
- `source_documents`
- `latency_ms`
- `status`
- `error_message`
- `created_at`

这一级能回答：

- 团队都在问什么。
- 哪些问题没有命中文档。
- 哪些数据源经常被命中。
- 哪些 chunk 被频繁使用。
- 哪些问题检索慢或失败。

### 9.3 完整问答审计

如果走统一问答网关，可以额外记录：

- `final_answer`
- `answer_model`
- `prompt_version`
- `citations`
- `user_feedback`
- `is_resolved`
- `need_doc_update`
- `review_status`

这一级能真正用于优化文档：

- 高频问题是否已有规范答案。
- 回答是否引用了正确来源。
- 哪些问题需要补内部 FAQ。
- 哪些回答有幻觉或引用不足。

### 9.4 审计表建议

可以先使用关系型数据库表：

```sql
CREATE TABLE harmony_kb_query_logs (
  id              VARCHAR(64) PRIMARY KEY,
  user_id         VARCHAR(128),
  client_type     VARCHAR(64),
  question        TEXT NOT NULL,
  dataset_ids     JSON,
  document_ids    JSON,
  retrieval_params JSON,
  matched_chunks  JSON,
  source_documents JSON,
  final_answer    TEXT,
  citations       JSON,
  latency_ms      INTEGER,
  status          VARCHAR(32),
  error_message   TEXT,
  feedback        VARCHAR(32),
  need_doc_update BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMP NOT NULL
);
```

后续可再拆成 query、retrieval、answer、feedback 多张表。

## 10. 回答引用和链接策略

由于 RAGFlow 不建议作为匿名只读文档站，回答引用建议按以下顺序生成：

1. 内部文档站 URL。
2. Git Web 文件链接。
3. 对象存储或静态站只读链接。
4. RAGFlow 内部 document / chunk 链接。
5. 源文件路径。

最终回答中建议包含：

```text
参考来源：
- internal-guides/router.md
- DevEco SDK: openharmony/ets/api/@ohos.router.d.ts
- HarmonyOS Docs: guides/arkts-ui-development.md
```

若同组同学都能登录 RAGFlow，可以同时附 RAGFlow 内部链接；否则以资料源链接为主。

## 11. 数据更新方案

### 11.1 为什么是“更新”而不是“新增”

如果每次同步都直接上传一份新文件，会导致：

- 同一份文档有多个版本同时被检索到。
- 旧 API 和新 API 混在一起，回答容易冲突。
- chunk 数量膨胀，检索变慢。
- 无法判断某个问题命中的是旧文档还是新文档。

因此同步脚本应维护 manifest，把源文件和 RAGFlow document 绑定起来。

### 11.2 Manifest 设计

示例：

```json
{
  "source": "hm-deveco-sdk-api",
  "files": [
    {
      "path": "openharmony/ets/api/@ohos.router.d.ts",
      "hash": "sha256:...",
      "mtime": "2026-05-20T10:00:00+08:00",
      "dataset_id": "xxx",
      "document_id": "yyy",
      "version": "DevEco-5.x",
      "status": "parsed"
    }
  ]
}
```

### 11.3 同步语义

同步脚本按文件 hash 判断：

- 新文件：上传到 RAGFlow，触发解析，记录 document_id。
- 文件未变化：跳过。
- 文件变化：旧 document 删除或禁用，再上传新 document 并解析。
- 文件消失：按策略删除、禁用或标记过期。
- 解析失败：记录失败清单，下一次重试。

建议第一阶段采用“删除旧文档并上传新文档”的方式，逻辑简单。若后续需要保留历史版本，再增加 `version` 和 `enabled=false` 的策略。

### 11.4 更新周期建议

| 数据源 | 更新周期 | 触发方式 |
| --- | --- | --- |
| 自有文档 | 每周一次 | 定时脚本或 Git 合并后手动触发 |
| 项目代码 | 每周或版本发布后 | 手动触发，避免频繁索引噪音 |
| DevEco SDK `.d.ts` | SDK 升级后 | 手动触发 |
| HarmonyOS 文档快照 | 每月或重要版本后 | 外网机器拉取，内网导入 |
| OpenHarmony docs | 每月或按需 | 外网机器拉取，内网导入 |
| Samples | 季度或重要版本后 | 手动触发 |

### 11.5 内网离线流程

建议拆成两段：

```text
外网机器：
  拉取 GitHub / Gitee / Huawei 文档
  -> 生成数据包和 manifest
  -> 计算 hash
  -> 拷贝到内网

内网机器：
  校验数据包
  -> 执行同步脚本
  -> 上传/更新 RAGFlow
  -> 触发解析
  -> 输出同步报告
```

同步报告应包含：

- 新增文件数。
- 更新文件数。
- 删除或禁用文件数。
- 跳过文件数。
- 解析成功数。
- 解析失败清单。
- 总 chunk 数变化。

## 12. 权限和安全

### 12.1 推荐权限模型

| 角色 | 权限 |
| --- | --- |
| 知识库管理员 | 数据源配置、上传、解析、删除、模型配置 |
| 文档维护人 | 更新自有文档，查看审计和反馈 |
| 团队普通成员 | 通过 MCP 查询 |
| 外部或匿名用户 | 默认不开放 |

### 12.2 MCP 访问控制

建议：

- MCP 网关部署在内网。
- RAGFlow API Key 只保存在服务端。
- 客户端只拿 MCP 地址和团队鉴权 token。
- MCP 网关只暴露检索工具。
- 管理类 API 不通过 MCP 对普通成员开放。

### 12.3 数据安全

审计日志中可能包含内部代码、业务信息和用户问题，应注意：

- 日志只在内网保存。
- 对敏感字段做访问控制。
- 需要定期清理或归档。
- 对外分享方案或样例时脱敏。

## 13. 分阶段落地计划

### 阶段 1：最小可用知识库

目标：让团队工具能查到鸿蒙资料。

范围：

- 部署 RAGFlow。
- 配置 Qwen3-Embedding-8B。
- 导入 DevEco SDK `.d.ts`、HarmonyOS 文档、少量内部 FAQ。
- 启动 RAGFlow MCP Server。
- 配置 Nanobot / Claude Code / opencode 调用 MCP。
- 编写基础 `harmony-kb` Skill。

验收：

- 能回答 `UIAbility`、`router.pushUrl`、ArkUI 状态管理等问题。
- 回答能包含来源路径。
- 团队成员可通过至少一种客户端调用。

### 阶段 2：审计和更新脚本

目标：可以持续维护并知道团队在问什么。

范围：

- 增加 MCP 审计网关。
- 记录问题、检索参数、命中 chunk、耗时、错误。
- 编写数据同步脚本和 manifest。
- 支持新增、更新、删除/禁用。
- 输出同步报告。

验收：

- 能看到一周内高频问题。
- 能定位无命中问题。
- 数据源重复同步不会产生重复文档。
- 解析失败有清单。

### 阶段 3：问答闭环

目标：把日志用于优化文档和回答质量。

范围：

- Nanobot 或统一问答网关记录最终回答。
- 增加用户反馈。
- 建立高频问题 -> FAQ 补充流程。
- 建立评测集和定期回归。
- 优化 chunk、metadata、检索参数。

验收：

- 每周产出待补文档清单。
- 高频问题有稳定回答。
- 评测集命中率和回答准确率可跟踪。

### 阶段 4：增强能力

目标：进一步提高复杂问题质量。

可选范围：

- 增加 rerank。
- 增加更细粒度代码解析。
- 增加只读文档展示站。
- 增加 OCR / 图片说明处理。
- 增加版本化知识库。

## 14. 需要同组讨论确认的问题

1. 是否接受 RAGFlow 不做匿名公开文档站，只作为登录后的内部管理和检索系统？
2. 回答中的引用链接优先使用内部文档站/Git Web，还是必须附 RAGFlow 页面链接？
3. 团队是否统一通过 Nanobot 走一层问答网关，还是允许 Claude Code / opencode 直连 MCP？
4. 是否必须记录最终回答？如果必须，是否接受统一代理或客户端回传机制？
5. 第一阶段数据集是按来源拆分，还是先建一个总 dataset 快速验证？
6. 自有文档的负责人和更新节奏如何定？
7. DevEco SDK 和 HarmonyOS 文档的数据包由谁从外网拉取并搬入内网？
8. Qwen3-Embedding-8B 的部署资源是否稳定，是否需要准备降级方案？
9. 是否需要保留历史版本文档，还是只保留最新版本？
10. 审计日志保留多久，哪些人可查看？

## 15. 风险和应对

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| RAGFlow 没有匿名只读页面 | 回答链接无法给未登录用户打开 | 建内部只读文档站，RAGFlow 只做检索 |
| 只记录 MCP 检索，拿不到最终回答 | 难以完整优化回答质量 | 统一 Nanobot/问答网关，或客户端回传答案 |
| 数据重复导入 | 旧文档和新文档冲突 | 使用 manifest + hash 做更新 |
| `.d.ts` 原样切分效果差 | API 签名类问题召回差 | 预处理成结构化 Markdown |
| 内网离线更新麻烦 | 文档过期 | 外网打包、内网导入、固定更新周期 |
| 图片资料无法被编码工具有效使用 | 图片内容缺失 | 关键图片人工补 Markdown 说明 |
| 单 embedding 模型效果不稳定 | 部分问题召回差 | 建评测集，先优化 chunk 和检索参数，再考虑 rerank |

## 16. 推荐最终方案

第一阶段不要追求“公开网页、图片、多模型、全自动更新”一次到位。建议先落地：

1. RAGFlow 内网部署。
2. 使用 Qwen3-Embedding-8B。
3. 按来源拆分 dataset。
4. 导入 `.d.ts`、官方 Markdown、内部 FAQ。
5. MCP 只暴露检索能力。
6. 配套 `harmony-kb` Skill。
7. 回答引用源文件路径或内部文档链接。
8. 用 manifest 脚本做周期更新。
9. MCP 网关记录检索日志。
10. 后续通过 Nanobot 或统一问答网关记录最终回答。

这样可以先解决团队真实使用问题，同时为后续优化留出清晰的扩展路径。
