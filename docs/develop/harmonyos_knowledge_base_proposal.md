# 鸿蒙知识库建设方案

## 1. 方案概述

本方案建设一个面向团队内部的鸿蒙研发知识库，用于支撑 HarmonyOS、OpenHarmony、ArkTS、ArkUI、DevEco、HMS Kit 等日常开发问题的检索和问答。

整体方案采用 RAGFlow 作为知识库的 RAG 数据库和检索引擎，使用内网已有的 Qwen3-Embedding-8B 作为第一阶段 Embedding 模型，通过 MCP 服务提供给 Nanobot、Claude Code、opencode 等编码工具调用。为满足后续优化要求，在 MCP 前增加审计网关，统一记录团队成员的提问、检索命中、来源文档、回答结果和反馈。

第一阶段重点解决三个问题：

1. 让编码工具能稳定查到鸿蒙 API、官方文档、样例代码和团队内部规范。
2. 让团队能看到大家高频问什么、哪些问题没有命中、哪些资料需要补充。
3. 建立可持续更新的数据同步机制，避免资料一次性导入后过期。

## 2. 推荐架构

```text
数据源
  - 团队内部文档
  - 团队项目代码 / 组件库
  - DevEco SDK .d.ts
  - HarmonyOS Next 文档快照
  - OpenHarmony docs
  - HarmonyOS / OpenHarmony Samples

        |
        v

数据同步脚本
  - 数据包校验
  - 文件 hash 比对
  - 新增 / 更新 / 删除识别
  - 生成同步报告

        |
        v

RAGFlow
  - Dataset 管理
  - 文档解析
  - Chunk 切分
  - Qwen3-Embedding-8B 向量化
  - 关键词 + 向量混合检索

        |
        v

MCP 审计网关
  - 统一团队入口
  - 只暴露检索能力
  - 记录提问、检索命中、来源、耗时、错误
  - 可扩展记录最终回答和用户反馈

        |
        v

团队工具
  - Nanobot
  - Claude Code
  - opencode
  - 其他支持 MCP 的工具
```

RAGFlow 不直接暴露给普通成员作为上传和解析入口。普通成员通过 MCP 查询；知识库管理员通过 RAGFlow 管理数据集和解析任务。

## 3. RAGFlow 定位和公开访问结论

RAGFlow 在本方案中定位为内网 RAG 后端，不定位为匿名公开文档站。

当前 RAGFlow 的数据集权限主要是 `me` 和 `team`，API 和 MCP 都依赖登录态或 API Key。现有能力不能按“免登录、匿名、只读访问某个文档页面”来设计。因此回答中不强依赖 RAGFlow 页面链接。

推荐链接策略如下：

1. 优先返回内部文档站、Git Web、静态 Markdown 站、对象存储只读链接。
2. 对没有 URL 的资料，返回源文件路径和数据源名称。
3. 对已登录 RAGFlow 的团队成员，可附加 RAGFlow 内部 document 或 chunk 链接。

这样可以避免把 RAGFlow 暴露为公共页面，同时仍然保证回答有来源可追溯。

## 4. 数据源规划

数据源按来源拆分为多个 dataset，便于设置优先级、更新周期、检索范围和后续问题分析。

| Dataset | 内容 | 作用 | 更新周期 |
| --- | --- | --- | --- |
| `hm-internal-guides` | 团队规范、FAQ、踩坑记录、封装约定 | 回答团队约定类问题，优先级最高 | 每周 |
| `hm-project-code` | 团队项目代码、组件库、页面模板 | 支撑“项目里怎么写” | 每周或版本发布后 |
| `hm-deveco-sdk-api` | DevEco SDK `.d.ts`，含 OpenHarmony / HMS API | 校验 API 签名、参数、枚举、返回值 | SDK 升级后 |
| `hm-harmonyos-docs` | HarmonyOS Next 官方文档快照 | 解释概念、生命周期、官方用法 | 每月或版本发布后 |
| `hm-openharmony-docs` | OpenHarmony 开源文档 | 补充系统、底层、开源能力说明 | 每月或按需 |
| `hm-harmonyos-samples` | 官方样例、OpenHarmony Samples、Codelabs | 提供真实工程写法和代码组合方式 | 季度或重要版本后 |

回答时默认来源优先级：

```text
团队内部文档
  > 团队项目代码
  > DevEco SDK .d.ts
  > HarmonyOS Next 官方文档
  > OpenHarmony docs
  > Samples
```

API 签名类问题优先以 `.d.ts` 为准；用法类问题优先以官方文档和 Samples 为准；团队封装和项目规范类问题优先以内部文档为准。

## 5. 数据录入和解析方案

### 5.1 录入方式

采用“原始资料直接入库 + 脚本批量录入”的方式，不把数据预处理作为第一阶段前置条件。团队只需要把资料按数据源目录放好，同步脚本负责上传、更新和触发解析。

同步脚本负责：

- 扫描数据源目录。
- 读取上次同步 manifest。
- 计算文件 hash。
- 判断新增、更新、删除、未变化。
- 调用 RAGFlow API 上传原始文件。
- 触发解析任务。
- 轮询解析状态。
- 输出同步报告。

同步脚本第一阶段不做 AST 解析、不做 `.d.ts` 结构化拆分、不做复杂清洗，只做必要的文件过滤和元信息记录。这样可以把维护成本控制在较低水平，先让知识库跑起来。

### 5.2 Chunk 策略

第一阶段使用 RAGFlow 原生解析和 chunk 能力，资料直接上传到对应 dataset。不同类型资料的处理口径如下：

| 类型 | 第一阶段处理方式 |
| --- | --- |
| `.d.ts` | 原样上传到 `hm-deveco-sdk-api`，依赖文件名、路径、关键词和 RAGFlow 原生切分 |
| Markdown 文档 | 原样上传到对应文档 dataset |
| ArkTS / TS / ETS 代码 | 原样上传到项目代码或样例 dataset |
| 内部 FAQ | 原样上传。若本来就是 Markdown，效果会更好 |
| 配置文件 | 原样上传，保留路径 |
| 图片 / PDF | 可以上传，但第一阶段不依赖图片内容作为核心召回依据 |

这种方式的优点是落地成本低、维护简单、数据更新快。缺点是 `.d.ts` 和代码文件的 chunk 不一定能天然切到函数或接口级别，API 精确召回可能不如结构化预处理方案。该风险通过后续评测和日志分析来处理，而不是在第一阶段投入复杂预处理。

后续如果发现某类高频问题召回不稳定，再针对少量高价值数据做增强，例如只对 DevEco SDK `.d.ts` 增加结构化解析，或把高频问题沉淀为内部 FAQ。

### 5.3 Metadata 标准

第一阶段 metadata 以“文档级元信息”为主，不强求 chunk 级 metadata。同步脚本在 manifest 和 RAGFlow document metadata 中尽量记录：

| 字段 | 含义 |
| --- | --- |
| `source` | 数据源分桶，如 `deveco-api-openharmony` |
| `source_type` | `internal_doc`、`project_code`、`sdk_dts`、`official_doc`、`sample_code` |
| `dataset` | 所属 RAGFlow dataset |
| `path` | 源文件路径 |
| `url` | 可访问的只读来源链接 |
| `version` | SDK 或文档版本 |
| `updated_at` | 源文件更新时间 |
| `owner` | 内部资料维护人 |

metadata 用于检索过滤、回答引用、日志分析和后续问题归因。`module`、`symbol` 这类精细字段不作为第一阶段要求，避免引入额外解析成本。

## 6. Embedding 和检索策略

第一阶段固定使用内网已有的 Qwen3-Embedding-8B。

采用该模型的原因：

- 已满足内网离线部署约束。
- 适合中文技术文档和代码语义场景。
- 初期无需引入多模型部署和切换复杂度。
- RAGFlow 中 dataset 产生 chunk 后不适合频繁更换 embedding；更换需要重新解析和向量化。

检索采用向量检索和关键词检索结合。鸿蒙问题中大量包含 API 名、类名、装饰器、枚举和错误码，纯向量检索容易漏掉精确符号，因此需要保留关键词召回。

初始参数建议：

| 参数 | 建议值 |
| --- | --- |
| `top_k` | 20 到 50 |
| `page_size` | 5 到 10 |
| `similarity_threshold` | 0.2 起步 |
| `vector_similarity_weight` | 0.3 到 0.5 |
| `keyword` | API 名、错误码、枚举类问题启用 |

上线前准备 30 到 50 个评测问题，用于验证召回质量。评测集覆盖：

- `UIAbility` 生命周期。
- `router.pushUrl` / `replaceUrl` 参数。
- ArkUI 状态管理。
- `Navigation`、`Tabs`、`List` 等组件用法。
- HMS Kit 导入方式。
- DevEco 工程配置。
- 团队内部封装规范。

第一阶段不强依赖 rerank。若评测发现召回到了正确 chunk 但排序不稳定，再引入离线 rerank 模型。

## 7. 图片和多模态策略

第一阶段不建设图片 embedding 或多模态检索能力。

原因：

- Claude Code、opencode 主要面向文本和代码。
- 鸿蒙编码问题的主要证据来自 `.d.ts`、Markdown 文档、代码样例和内部规范。
- 图片 OCR 和图片 embedding 会增加部署复杂度，但对第一阶段收益有限。

处理方式：

- 文档图片保留原始链接或文件路径。
- 关键架构图、流程图、截图由维护人补充 Markdown 文字说明。
- 后续若发现高频问题依赖图片，再单独评估 OCR 和多模态解析。

## 8. MCP 服务和 Skill 方案

### 8.1 MCP 服务

使用 RAGFlow MCP Server 的 `self-host` 模式作为基础，服务端保存 RAGFlow API Key，团队工具连接 MCP 审计网关。

推荐链路：

```text
Claude Code / opencode / Nanobot
  -> MCP 审计网关
  -> RAGFlow MCP Server 或 RAGFlow Retrieval API
  -> RAGFlow Dataset
```

MCP 审计网关只暴露检索工具，不暴露上传、解析、删除等管理能力。管理操作只通过管理员账号和同步脚本完成。

### 8.2 Skill

MCP 只能提供工具，不能保证模型主动调用。需要配套 `harmony-kb` Skill 或等价提示词。

Skill 规则：

- 遇到 HarmonyOS、OpenHarmony、ArkTS、ArkUI、DevEco、HMS 相关问题时，先查知识库。
- 涉及 API 签名、参数、枚举、返回值时，优先查 `hm-deveco-sdk-api`。
- 涉及用法、生命周期、配置说明时，优先查 `hm-harmonyos-docs`。
- 涉及项目内写法时，优先查 `hm-internal-guides` 和 `hm-project-code`。
- 回答必须附来源。
- 证据不足时明确说明，不编造 API。

### 8.3 客户端接入

| 工具 | 推荐接入方式 |
| --- | --- |
| Nanobot | 通过统一问答网关接入，最适合完整记录最终回答 |
| Claude Code | 配置 MCP 审计网关，并安装 `harmony-kb` Skill |
| opencode | 配置 MCP 审计网关，并安装 `harmony-kb` Skill |

需要完整记录最终回答时，优先让团队通过 Nanobot 或统一问答网关使用知识库。Claude Code / opencode 直连 MCP 时，网关只能稳定记录问题和检索结果，无法天然拿到模型最终回答。

## 9. 审计网关方案

MCP 前增加审计网关是本方案的关键。它承担三个职责：

1. 统一入口：团队工具都连同一个 MCP 地址。
2. 权限控制：普通成员只能检索，不能上传和解析。
3. 数据闭环：记录问题、命中、回答和反馈，用于优化资料。

### 9.1 审计内容

最低可行版本记录：

- 用户或客户端标识。
- 客户端类型：Nanobot、Claude Code、opencode。
- 原始问题。
- dataset_ids。
- 检索参数。
- 命中的 chunk。
- 来源文档和路径。
- 耗时。
- 错误信息。
- 请求时间。

完整版本增加：

- 最终回答。
- 引用来源。
- 使用的模型。
- prompt / skill 版本。
- 用户反馈。
- 是否需要补充文档。

### 9.2 日志表

第一阶段可先用单表：

```sql
CREATE TABLE harmony_kb_query_logs (
  id                VARCHAR(64) PRIMARY KEY,
  user_id           VARCHAR(128),
  client_type       VARCHAR(64),
  question          TEXT NOT NULL,
  dataset_ids       JSON,
  document_ids      JSON,
  retrieval_params  JSON,
  matched_chunks    JSON,
  source_documents  JSON,
  final_answer      TEXT,
  citations         JSON,
  latency_ms        INTEGER,
  status            VARCHAR(32),
  error_message     TEXT,
  feedback          VARCHAR(32),
  need_doc_update   BOOLEAN DEFAULT FALSE,
  created_at        TIMESTAMP NOT NULL
);
```

后续数据量变大后，再拆成 query、retrieval、answer、feedback 多张表。

### 9.3 优化闭环

每周基于审计日志生成优化清单：

- 高频问题 Top 20。
- 无命中问题。
- 低质量回答问题。
- 高频命中文档。
- 需要补 FAQ 的问题。
- 需要调整 chunk 或 metadata 的资料。

文档维护人根据清单补充内部 FAQ、修正文档、调整数据源，再由同步脚本更新到 RAGFlow。

## 10. 数据更新方案

更新采用 manifest + hash 机制，避免每次同步都重复新增。

### 10.1 Manifest

每个数据源维护一份 manifest：

```json
{
  "source": "hm-deveco-sdk-api",
  "files": [
    {
      "path": "openharmony/ets/api/@ohos.router.d.ts",
      "hash": "sha256:...",
      "mtime": "2026-05-20T10:00:00+08:00",
      "dataset_id": "ragflow_dataset_id",
      "document_id": "ragflow_document_id",
      "version": "DevEco-5.x",
      "status": "parsed"
    }
  ]
}
```

### 10.2 更新规则

| 文件状态 | 处理方式 |
| --- | --- |
| 新文件 | 上传 RAGFlow，触发解析，写入 manifest |
| hash 未变化 | 跳过 |
| hash 变化 | 删除或禁用旧 document，上传新 document，重新解析 |
| 源文件消失 | 删除、禁用或标记过期 |
| 解析失败 | 记录失败清单，下次重试 |

第一阶段采用“删除旧 document，再上传新 document”的方式，保证检索只命中新版本资料。后续如需要版本追溯，再改为多版本保留。

### 10.3 更新周期

| 数据源 | 周期 |
| --- | --- |
| 自有文档 | 每周一次 |
| 团队项目代码 | 每周一次或版本发布后 |
| DevEco SDK `.d.ts` | DevEco / SDK 升级后 |
| HarmonyOS 文档快照 | 每月一次或重要版本后 |
| OpenHarmony docs | 每月一次或按需 |
| Samples | 每季度一次或重要版本后 |

### 10.4 内网离线更新

内网部署下，外部数据获取和内网导入分开。

外网机器负责：

- 拉取 GitHub / Gitee / Huawei 文档。
- 整理数据源目录。
- 生成数据包。
- 生成 hash 清单。
- 拷贝到内网。

内网机器负责：

- 校验数据包。
- 执行同步脚本。
- 上传或更新 RAGFlow document。
- 触发解析。
- 输出同步报告。

同步报告包含：

- 新增文件数。
- 更新文件数。
- 删除或禁用文件数。
- 跳过文件数。
- 解析成功数。
- 解析失败清单。
- chunk 数变化。

## 11. 权限和安全

权限分层：

| 角色 | 权限 |
| --- | --- |
| 知识库管理员 | 数据源配置、上传、解析、删除、模型配置 |
| 文档维护人 | 更新自有文档、查看审计、处理反馈 |
| 团队成员 | 通过 MCP 查询 |
| 匿名用户 | 不开放 |

安全策略：

- RAGFlow 部署在内网。
- RAGFlow API Key 只保存在服务端。
- 普通成员只访问 MCP 审计网关。
- MCP 网关只暴露检索工具。
- 审计日志仅内网保存。
- 对日志访问做权限控制。

## 12. 实施计划

### 阶段一：知识库可用

目标：团队工具能查鸿蒙知识库。

工作项：

- 部署 RAGFlow。
- 配置 Qwen3-Embedding-8B。
- 创建核心 datasets。
- 导入 DevEco SDK `.d.ts`。
- 导入 HarmonyOS 文档快照。
- 导入第一批内部 FAQ。
- 启动 MCP 服务。
- 配置 Nanobot / Claude Code / opencode。
- 编写 `harmony-kb` Skill。

验收标准：

- 能通过 MCP 查询知识库。
- 能回答 API 签名、生命周期、状态管理、团队规范等基础问题。
- 回答包含来源路径或链接。

### 阶段二：同步和审计可用

目标：数据可持续更新，问题可追踪。

工作项：

- 实现 manifest 同步脚本。
- 支持新增、更新、删除/禁用。
- 输出同步报告。
- 建设 MCP 审计网关。
- 记录问题、检索命中、耗时和错误。

验收标准：

- 重复执行同步不会产生重复文档。
- 文件变化后能更新 RAGFlow 文档。
- 能查看一周内问题日志和无命中问题。

### 阶段三：回答闭环

目标：用真实使用数据优化文档。

工作项：

- Nanobot 或统一问答网关记录最终回答。
- 增加用户反馈。
- 建立高频问题分析报表。
- 每周补充内部 FAQ。
- 建立评测集并做回归。

验收标准：

- 能看到高频问题和待补文档清单。
- 能追踪回答质量变化。
- 评测集命中率稳定提升。

### 阶段四：增强能力

目标：提高复杂问题处理能力。

可选工作项：

- 引入离线 rerank。
- 优化 `.d.ts` 结构化解析。
- 建设内部只读文档站。
- 对关键图片做 OCR 或人工说明。
- 支持多版本资料管理。

## 13. 风险和控制

| 风险 | 影响 | 控制措施 |
| --- | --- | --- |
| RAGFlow 不支持匿名只读页面 | 回答链接无法给未登录用户访问 | 使用内部文档站、Git Web、源文件路径作为引用 |
| 直连 MCP 拿不到最终回答 | 难以完整分析回答质量 | 通过 Nanobot 或统一问答网关记录最终回答 |
| 数据重复导入 | 新旧资料冲突，检索质量下降 | manifest + hash 更新机制 |
| 原始 `.d.ts` 和代码文件解析较粗 | API 签名问题命中不稳定 | 先接受原始入库，靠评测和日志定位高频问题，再对少量高价值资料做增强 |
| 内网离线导致资料过期 | 回答引用旧资料 | 固定更新周期，外网打包、内网导入 |
| 图片内容无法有效使用 | 图片里的信息缺失 | 关键图片补 Markdown 说明 |
| 单一 embedding 效果不足 | 部分问题召回差 | 先做评测集和 chunk 优化，再考虑 rerank |

## 14. 最终建议

本方案建议按“先可用、再可控、再优化”的路径推进。

第一阶段不投入匿名公开页面、图片 embedding、多模型评估、复杂数据预处理和全自动实时同步，先把原始资料直接入库、团队工具调用、审计记录和周期更新打通。RAGFlow 作为内网检索后端，MCP 审计网关作为团队统一入口，`harmony-kb` Skill 作为编码工具调用约束，manifest 同步脚本作为持续更新机制。

该路径能较快形成可用能力，同时保留后续增强空间。后续优化重点放在真实问题日志、无命中问题、低质量回答和高频内部规范补充上，让知识库随着团队使用逐步变准。
