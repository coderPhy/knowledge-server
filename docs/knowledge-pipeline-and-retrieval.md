# 文档管线、检索与评测

## 1. 发布后的三条 MQ 管线

发布或审核通过后，RabbitMQ 并行投递三条消息；消息投递成功不代表索引已完成。

```text
Published
  ├─ RAG: rag.reindex.exchange -> kh.rag.reindex.queue
  │       PostgreSQL/Mongo 正文 -> 分块 -> Embedding -> ES kh_chunk
  ├─ Search: search.index.exchange -> kh.search.index.queue
  │          PostgreSQL/Mongo 正文 -> 整篇写入 ES kh_document
  └─ KG: kg.graph.exchange -> kh.kg.graph.queue
           分块 -> LLM 抽实体/关系 -> Neo4j
```

### 1.1 RAG 索引

消息 `BY_DOC_IDS` 只带文档 ID。消费后删除旧 chunk，用 Markdown 递归切块（默认约 512 token、重叠 64 token），批量生成 embedding（默认 `text-embedding-v3`、1024 维），bulk 写入 `kh_chunk`。每条记录包含 chunk 正文、标题、块序号、文档 ID 及 ACL 字段，支持 BM25 和向量 kNN。

### 1.2 Search 索引

消息 `INDEX` 带文档 ID。消费后加载整篇正文，upsert 到 `kh_document`。`POST /search` 在 `title^3`、`summary^2`、`content` 上做 BM25，返回文档列表和高亮，不参与问答 chunk 召回。

### 1.3 KG 索引

消息 `BUILD_BY_DOC_IDS` 带文档 ID。复用 RAG 分块，将文档和 chunk 写入 Neo4j，再对每块抽取实体/关系，形成 `KnowledgeDocument`、`DocumentChunk`、`KnowledgeEntity`、`HAS_CHUNK`、`MENTIONS`、`RELATED_TO`。问答时图谱提供实体关系线索，制度原文仍以 RAG chunk 为准。

归档/删除会并行发送三路删除消息。exchange/queue durable、消息 persistent；投递失败不回滚发布状态，消费失败 `nack(requeue=false)`，当前没有重试队列/DLQ。

## 2. ES 的 IK、倒排索引与 BM25

- **倒排索引**：文本经 analyzer 变成词项，ES 建立“词项 -> 文档/词频/位置”的索引；查询时按词项快速找候选，不扫描全文。
- **IK**：中文分词器，不负责排序。写入通常用 `ik_max_word`，查询用 `ik_smart`。
- **BM25**：ES/Lucene 的关键词相关性评分，综合词频、词项稀有度、文本长度等因素排序。项目用 `multi_match`，没有自行实现 BM25。

字段权重：RAG `kh_chunk` 为 `document_title^2`、`content`；Search `kh_document` 为 `title^3`、`summary^2`、`content`。`^N` 表示字段命中权重。

## 3. Langfuse tracing 与评测

配置 Langfuse Key 后上报 trace；未配置时仍可运行本地逻辑。

检测项目、指标和验证方式：

| 检测项目 | 指标 | 验证方式 |
| --- | --- | --- |
| 意图识别 | `intent_match` | 实际意图与金标意图比较 |
| 工具策略 | `tool_policy` | 检查检索、图谱、联网是否符合意图约束 |
| 查询改写 | `rewrite_loop` | 要求改写时，确认已改写且新 query 不同 |
| 检索命中 | `hit_at_5` | Top-5 是否命中金标文档标题 |
| 检索精度 | `precision_at_5` | Top-5 中金标文档数 / 实际 Top-5 数 |
| 误命中 | `false_hit` | 空金标题目中，确认没有 graded hit |
| 拒答质量 | `refusal_quality` | LLM judge 判断是否正确拒答且未编造 |
| 回答完整性 | `completeness` | LLM judge 判断 keyPoints 覆盖率 |
| 回答依据 | `groundedness` | LLM judge 判断回答是否被资料支持 |

运行离线评测：

```bash
pnpm eval:rag
```

评测使用 `src/ai/eval/golden-dataset.ts` 的金标数据；没有金标只能人工查看 trace，不能计算准确率。无适用数据或 judge 失败时，指标返回 `null`，不按 0 计。

指标及验证方式：

| 指标 | 验证 |
| --- | --- |
| `intent_match` | 实际意图 == 金标意图 |
| `tool_policy` | `retrieveCount`、`graphUsed`、`webUsed` 是否符合 require/forbid 约束 |
| `rewrite_loop` | 要求改写时，确认已改写且新 query 不同 |
| `hit_at_5` | rawHits Top-5 是否命中金标文档标题 |
| `precision_at_5` | Top-5 中金标文档数 / 实际 Top-5 数 |
| `false_hit` | 空金标题目中，gradedHits 为空得 1，否则 0 |
| `refusal_quality` | LLM judge 判断已拒答且未编造 |
| `completeness` | LLM judge 判断 keyPoints 覆盖率；命中禁用事实为 0 |
| `groundedness` | LLM judge 对照资料判断：supported=1、mixed=0.5、fabricated=0 |

规则指标直接比较快照与金标；后三项使用结构化 LLM judge（默认 `EVAL_JUDGE_MODEL`/`qwen-plus`）。不适用、无金标或 judge 失败返回 `null`，不计为 0。当前没有独立的 Neo4j 准确率、延迟、token 成本、MQ 成功率指标。

## 4. 文件解析为 Markdown（面试重点）

### 4.1 总体流程

入口是 `POST /documents/upload/parse`，使用 `multipart/form-data` 的 `file` 字段，Multer 限制单文件最大 50 MB。作者从 JWT 获取，表单只补充分类、团队、标签、备注和公开范围。

```text
上传文件
  -> Controller 校验 file
  -> DocumentService 还原中文文件名、校验扩展名
  -> FileParserService.parse(Buffer)
       -> 按扩展名选择 parser
       -> 输出统一 Markdown 字符串
       -> 清洗换行/空白
       -> 空结果或解析失败返回 400
  -> RustFS 保存原文件（启用时，documents/）
  -> MongoDB document_content 保存 Markdown 正文
  -> PostgreSQL kh_document 保存元数据和 content_id
  -> 创建 Draft，不自动发布
  -> 发布/审核通过后，三路索引都从 Mongo Markdown 正文读取
```

核心设计是**先统一成 Markdown，再由后续业务复用**：RAG 和 KG 使用同一套 Markdown 分块；Search 把完整 Markdown 写入 `kh_document.content`。这样不同文件格式不会让下游索引各自实现一套解析逻辑。

### 4.2 入口校验与统一分发

`FileParserService` 支持 `pdf`、`docx`、`xlsx`、`pptx`、`txt`、`md`。它先通过文件名取小写扩展名，再检查扩展名是否在白名单中，并拒绝空 Buffer。按扩展名分发：

| 格式 | 主解析路径 | 输出重点 |
| --- | --- | --- |
| `.md` / `.txt` | `parsePlainText`，按 UTF-8 读取 | 原文直接作为 Markdown/纯文本，不二次转换 |
| `.docx` | `mammoth` -> HTML -> `turndown + GFM` | 标题、段落、列表、链接、表格、删除线等 Markdown |
| `.pdf` | `pdf-parse` 文本/表格/图片能力 | 按页文本、Markdown 表格、可选图片引用 |
| `.pptx` | 优先 `JSZip` 读取 OOXML；失败 `officeparser` | 幻灯片标题、正文、表格 |
| `.xlsx` | 优先 `exceljs`；失败 `officeparser` | Sheet 标题和 Markdown 表格 |

所有 parser 最终返回 `string`。服务记录文件名、格式、输出字符数和耗时；结果为空时抛出“解析结果为空”，不会创建空文档。

### 4.3 DOCX：结构化文档转 Markdown

1. `mammoth.convertToHtml` 读取 DOCX，并把 Word 的 Title、Heading 1-4 以及中文“标题 1-4”样式映射为 HTML 标题。
2. `turndown` 将 HTML 转成 Markdown，标题使用 ATX 风格（`#`），代码块使用 fenced 风格；启用 GFM 插件支持表格、删除线、任务列表等扩展。
3. `cleanMarkdown` 统一 CRLF/LF、压缩过多空行并去除首尾空白。

面试可强调：不是直接读取 DOCX XML 拼字符串，而是先借助 Mammoth 保留文档结构，再由 Turndown 做 HTML 到 Markdown 的转换。

### 4.4 PDF：按页文本、图片和表格的尽力解析

1. `pdf-parse` 按页提取文本；有分页信息时按页组织，页与页之间保留空行。
2. RustFS 启用时注入 `uploadImage` 回调，解析器按页提取图片，过滤宽/高小于默认 50 像素的装饰图，按文件头识别 JPEG/PNG/WebP 后上传到 `pdf-images/`，再在 Markdown 中写入 `![](url)`。
3. 图片提取或单张图片上传失败只记录日志，正文仍继续，不影响整份 PDF。
4. 尝试调用 `getTable()` 提取表格，将不规则返回结构归一为二维字符串数组，再用 `toMarkdownTable` 追加到“检测到的表格”章节；表格失败不影响主文本。
5. `finally` 中销毁 PDF parser，释放底层/WASM 资源。

因此 PDF 是“文本为主、表格和图片尽力补充”的解析，不承诺复杂版式、扫描件 OCR 或所有表格都能完美还原。

### 4.5 PPTX：OOXML 主路径与 officeparser 降级

PPTX 本质是 ZIP 包里的 OOXML。主路径用 `JSZip` 解压 `ppt/slides/slideN.xml`，按 slide 编号排序：

1. 输出 `## 幻灯片 N`。
2. 提取 `<a:tbl>` 表格，单元格转成 Markdown 表格；随后从正文 XML 中移除表格区域，避免单元格内容重复。
3. 从 title/ctrTitle 占位符提取标题，输出为 `### 标题`。
4. 提取其余 `<a:p>` 段落，合并同一段落的文本 run，处理显式换行，并跳过已作为标题输出的文本。
5. XML 解压、slide 不存在或结果为空时，降级为 `officeparser(...).to('md')`。

### 4.6 XLSX：工作表和单元格转 Markdown 表格

主路径使用 `exceljs` 加载 Workbook，遍历每个 Sheet：

1. 输出 `## SheetName`。
2. 遍历非空行，按整张表最大列数补齐缺失单元格。
3. 首行作为 Markdown 表头，使用 `toMarkdownTable` 输出表格；空 Sheet 仍保留 Sheet 标题。
4. 单元格统一转字符串：公式优先取计算结果、超链接取显示文本、富文本拼接各 run、日期转 ISO 字符串。
5. ExcelJS 加载或转换失败时，降级 `officeparser` AST -> Markdown。

表格工具会转义单元格中的 `|`，并把单元格换行压成空格，避免破坏 Markdown 表结构。

### 4.7 存储、失败边界和下游使用

- 原始文件：RustFS `documents/` 前缀（RustFS 未启用时跳过，不影响正文解析）。
- 正文：MongoDB `document_content`，包含 Markdown、长度、摘要等；PostgreSQL `kh_document.content_id` 关联 Mongo `_id`。
- 元数据：PostgreSQL 保存标题、作者、状态、分类、团队、标签、文件地址等。
- 上传解析成功后默认是 Draft，不会因为解析成功自动进入索引。
- 不支持扩展名、空文件、空解析结果、原文件上传失败会返回 400；PDF 图片/表格失败、XLSX/PPTX 主解析失败则按上述策略降级或继续。

下游使用同一份 Markdown：

```text
Mongo Markdown
  ├─ RAG：分块 -> Embedding -> ES kh_chunk
  ├─ Search：整篇写 ES kh_document.content
  └─ KG：同样分块 -> 实体/关系 -> Neo4j
```



## 5. 单次问答完整时序（以 `kb_then_web` 为例）

示例问题：

> 公司出差住宿报销标准是什么？如果涉及 2026 年发票合规要求，国家最新规定又是怎样的？

该问题同时包含企业内部制度和外部时效信息，因此意图通常会被识别为 `kb_then_web`。本意图允许知识库检索、查询词改写、知识图谱检索和联网搜索，但要求先完成知识库流程，资料仍不足时才能联网。

### 5.1 请求进入与会话处理

前端发送当前用户、可选的 `sessionId`、消息列表和 `topK`。服务从消息列表中取最后一条用户文本作为本轮问题。

- 没有 `sessionId`：PostgreSQL 创建 `ai_session`，生成会话 ID，并用问题前 30 个字符生成标题。
- 有 `sessionId`：校验会话属于当前用户；如果标题仍是“新对话”，用本轮问题更新标题。
- 新的会话 ID 通过 `data-session` 事件流给前端。

此时通常只改变 PostgreSQL 的会话记录，不会插入问答消息；问答消息要等最终回答生成后再落库。

### 5.2 加载短期记忆

服务先从 Redis 读取当前用户、当前会话的最近消息窗口。

```text
Redis 命中
  -> 直接使用最近对话

Redis 未命中或 Redis 故障
  -> PostgreSQL 查询 ai_message 最近 N 条
  -> 转成模型消息
  -> 回填 Redis
  -> 继续本轮问答
```

Redis 中只保存用户和助手的对话原文，不保存检索资料、模型思考、工具调用细节或长期记忆系统消息。回填和后续写入会受短期记忆最大消息数与 TTL 限制。

### 5.3 查询长期记忆并识别意图

短期记忆加载后，系统并行准备两类信息：

1. **Mem0 长期记忆查询**：
   - 用户层：跨会话有效的身份、岗位、偏好和长期约束。
   - 会话层：只属于当前会话的任务、进度和临时约定。
2. **意图识别**：根据当前问题和短期对话历史生成独立查询词、图谱实体词和工具开关。

本例可能得到：

```text
intent: kb_then_web
query: 出差住宿报销标准及 2026 年发票合规要求
graphQueries: 出差 / 住宿 / 报销 / 发票
allowRetrieve: true
allowGraph: true
allowWeb: true
```

意图结果通过 `data-intent` 事件发送给前端。长期记忆只是背景信息，制度和流程必须以本轮检索资料为准；如果未配置 Mem0，则这里返回空记忆，不阻断问答。

### 5.4 创建 Agent 并组装上下文

服务根据意图创建本轮 Agent，并挂载四个工具：

```text
retrieve_knowledge  查企业知识库
rewrite_query       知识库结果不足时改写检索词
retrieve_graph      查企业知识图谱实体和关系
web_search          查互联网公开信息
```

发送给模型的上下文由以下部分组成：

```text
系统规则
  + 用户层长期记忆
  + 当前会话层长期记忆
  + Redis 中的短期对话历史
  + 本轮用户问题
```

Agent 还有限制：知识库最多检索两次，模型循环最多八次；对话过长时会用压缩模型摘要旧消息。

### 5.5 第一次检索：知识库与图谱并行

Agent 首次通常同时调用 `retrieve_knowledge` 和 `retrieve_graph`。

#### 知识库路径

`retrieve_knowledge` 调用混合检索：在 ES `kh_chunk` 上执行 BM25 和向量 kNN，按用户 ACL 过滤，经 RRF、reranker 和分数过滤后得到候选 chunk。BM25 擅长字面术语和编号，向量检索擅长语义近似；随后再判断命中内容是否真正切题。

```text
query -> BM25 kh_chunk ─┐
query -> embedding -> kNN ─┴-> RRF 融合 -> reranker -> 分数过滤 -> topK
```

具体过程是：BM25 与 kNN 各召回候选，按 chunk ID 用 RRF 合并排名，再由 `qwen3-rerank` 精排，最后按 `RAG_MIN_SCORE` 过滤并取 topK。Embedding 失败时退化为 BM25；本轮知识库最多检索两次。

- 切题的 chunk 才进入回答上下文，并生成 `[1]`、`[2]` 等引用来源。
- 不切题的原始命中只用于后续改写，不能直接作为制度依据。
- 检索过程、评估结果和来源通过流事件发送给前端。

例如第一次只找到“报销申请流程”，没有找到住宿金额标准，则评估为资料不足。

#### 图谱路径

`retrieve_graph` 使用“出差 / 住宿 / 报销 / 发票”等短实体词查询 Neo4j，按实体名或别名匹配，再查询 `RELATED_TO` 关系和可见来源文档。它不是 BM25/kNN，不提供制度原文；图谱为空或不可用时返回空结果，不阻断知识库检索。

### 5.6 知识库不足时改写并再次检索

如果第一次知识库评估为不切题或无结果，Agent 调用 `rewrite_query`。它根据原问题、上次 query、命中内容和不足原因生成更具体的新 query，例如：

```text
国内出差住宿费城市级别报销限额及发票合规要求
```

随后 Agent 用新 query 第二次调用 `retrieve_knowledge`。知识库最多两轮，第二轮仍不足时不能进行第三轮内部检索。

### 5.7 仍有外部问题时联网

本例即使查到公司内部住宿标准，用户还询问“2026 年最新发票合规要求”，这部分属于公开且时效性的信息，因此 Agent 才调用 `web_search`。

联网结果包含标题、URL、摘要和网站名称。它只服务于本轮回答，不写入企业知识库，也不应被表述成公司内部制度。

### 5.8 流式回答与前端事件

模型和工具运行期间，服务持续向前端发送：

- 意图识别结果
- 状态文字（识别、改写、检索等）
- 检索评估
- 知识库来源
- 图谱结果
- 模型思考和回答文本

最终回答应区分来源：公司报销标准引用企业知识库，外部政策引用网页标题和链接，不能把网页内容冒充内部制度，也不能把图谱关系当成制度原文。

### 5.9 流结束后的持久化

回答流结束后，系统从最终回答中的 `[n]` 引用编号筛出真正使用的知识库来源，然后执行以下写入：

1. **PostgreSQL `ai_session`**：更新会话 `updated_at`，必要时补齐标题。
2. **PostgreSQL `ai_message`**：插入一条用户消息和一条助手消息；助手消息保存最终回答及实际使用的知识库来源。
3. **Redis 短期记忆**：把历史、本轮用户问题和本轮助手回答合并，只保留最近窗口并刷新 TTL。
4. **Mem0 长期记忆**：用分类器判断用户本轮是否表达了新的身份、偏好、长期约束或会话任务；只把符合条件的用户话语写入 user/session 记忆。

不会写入 Mem0 的内容包括公司制度条文、助手根据检索资料给出的事实、联网搜索结果和引用编号。Mem0 写入失败只记录日志，不回滚已经返回的回答。

### 5.10 完整时序总览

```text
请求进入
  -> 创建/更新 PostgreSQL 会话
  -> Redis 读取短期记忆
       -> 命中：直接使用
       -> 未命中：PostgreSQL 读取 ai_message，再回填 Redis
  -> Mem0 查询用户层和会话层长期记忆
  -> 意图识别，得到 kb_then_web 路由计划
  -> 创建带四个工具的 Agent
  -> 组装长期记忆 + 短期历史 + 当前问题
  -> 知识库检索 + 图谱检索
  -> 知识库不足：改写 query
  -> 第二次知识库检索
  -> 仍需外部时效信息：联网搜索
  -> 流式生成最终回答
  -> PostgreSQL 保存用户消息、助手消息和引用
  -> Redis 更新短期记忆
  -> Mem0 分类并写入必要的长期记忆
```

整体职责可以概括为：PostgreSQL 负责可靠保存会话和消息，Redis 负责当前会话的快速短期记忆，Mem0 负责长期背景记忆，ES/Neo4j 分别提供文档检索和关系线索，Agent 负责按意图协调工具并生成最终回答。
