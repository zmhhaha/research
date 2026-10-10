# Hindsight

- **调研日期**：2026-10-10
- **上游地址**：https://github.com/vectorize-io/hindsight
- **调研版本 / commit**：`v0.10.3` = `eb6df499d35300e5b2f3f029b2e6adda04ed90f8`（2026-10-08）；实际走读的是 clone 的 `main` HEAD = `a9eeed77f9bce99d8f97d39a30d3281747300824`（2026-10-10，领先 v0.10.3 共 13 个提交）。本文所有 `文件:行号` 都以 HEAD `a9eeed77` 为基准。
- **本地副本**：`D:\github\research\.cache\hindsight`（完整历史，`.cache/` 被 gitignore，不进本仓库；git pack 783 MiB）
- **许可证**：MIT（Copyright 2025 Vectorize AI, Inc.）——13 个集成目录的 LICENSE 也都是 MIT

## 一句话结论

Vectorize.io 开源的 agent 长期记忆服务：`retain` 时用 LLM 把内容抽成事实/实体/因果链，`recall` 时四路检索（向量 + BM25 + 图 + 时间）+ RRF 融合 + cross-encoder 重排，后台再把事实沉淀成带证据的 observation 和自动刷新的 mental model。MIT、Postgres/pgvector、Docker 一条命令能起，集成面铺得极广（60+，含主流 coding agent）。

工程上最值得借鉴的是**"写侧重、读侧轻"**这个取舍：所有 LLM 重活都压在写入和后台，读路径只做检索与重排。最大的代价也在这里——**每写一条记忆都要花外部 LLM 的 token**，且这是默认行为，不是可选项。此外：单仓 5041 个文件 / 31 万行 Python、版本仍是 0.x、明确不接受外部 PR、厂商标称的 SOTA 分数在仓库里**找不到可复现的数据**。

## 项目概况

| 项 | 值 | 来源 |
| --- | --- | --- |
| 仓库地址 | https://github.com/vectorize-io/hindsight | 已验证（git remote） |
| 许可证 | MIT，Copyright 2025 Vectorize AI, Inc. | 已验证（`LICENSE:1-3`） |
| 语言 / 技术栈 | Python 3.11 + FastAPI（引擎/服务端）、PostgreSQL + pgvector（可换 Oracle 23ai）、Next.js 控制台、Rust CLI、Go/TS/Python 客户端 | 已验证 |
| 提交数 / 贡献者 | 3573 commits / 297 人；Top1 Nicolò Boschi 1927 次（54%），Top5 合计约 77% | 已验证（`git shortlog -sn HEAD`） |
| 最新版本 | `v0.10.3`（2026-10-08）；`v*` 主版本 tag 90 个，全仓 tag 285 个（含各集成独立版本号） | 已验证（`git tag`） |
| 首次提交 | 2025-10-30，"initial commit" | 已验证 |
| 仓库体积 | git pack 783 MiB；工作树约 1 GB | 已验证（`git count-objects -vH`） |
| 代码构成 | 追踪文件 5041 个：`.py` 1982、`.md` 832、`.ts` 584、`.go` 245、`.rs` 36 | 已验证（`git ls-files`） |
| 核心代码量 | `hindsight-api-slim/hindsight_api` 351 个 `.py` / 146,612 行；其中 `engine/` 173 个文件 / 102,831 行；最大单文件 `engine/memory_engine.py` 20,953 行 | 已验证（逐文件统计） |
| 论文 | arXiv:2512.12818（README 引用）；**仓库内只有链接，没有 PDF/LaTeX 正文** | 已验证（全仓唯一 `.pdf` 是 retain 示例附件） |
| 热度（star） | 厂商博客称 2026-09-28 达到 4 万 star（更早一篇称 2026-08-14 达 2 万） | **未核实**（本环境无法访问 GitHub 页面，只有搜索摘要） |
| 官方自称规模 | "25+ LLM providers"、"60+ integrations" | 未核实（README 自述，未逐项清点） |

版本节奏：从 v0.0.1（2025-11-11）到 v0.10.3（2026-10-08）约 11 个月 90 个主版本；近期稳定在 1–2 周一个版本。提交强度极高：近 30 天 618 个提交，近 90 天 1511 个。

## 它解决什么问题

**痛点**：对话式 agent 的"记忆"通常只有两种做法——把历史塞进上下文（昂贵、会溢出），或朴素 RAG（只能按相似度捞回文档片段，回答不了"Alice 上个月做的那件事后来影响到了谁"这类需要跨实体和跨时间推理的问题）。

**它的方案**：把记忆当作结构化的、会自己演化的数据库，而不是一段可检索的文本。核心概念（`README.md:277-369`）：

- **世界事实 / 经验**：区分"世界是怎样的"和"agent 自己经历过什么"。
- **Observations（观察）**：把零散事实去重、合并成"有证据支撑的信念"，每条带原始引文和证据计数；新证据是**修订**而不是覆盖旧信念。
- **Mental models / Knowledge pages**：对某个问题（"这个用户的偏好是什么"）的**常驻答案**，由后台自动重写；读取时是纯数据库读，不触发检索也不调 LLM——这是它"让 agent 开局就带着结论"的关键设计。
- **Banks**：记忆隔离单元，每个 bank 配 disposition（怀疑/字面/共情三项 1-5）只影响 reflect，不影响 recall。

**与 RAG 的差异**（`hindsight-docs/docs/developer/rag-vs-hindsight.md:11-44`，文档说法）：检索策略从"仅语义"变成四路；支持图上的多跳（Alice → Project Atlas → Kubernetes → 故障）和日期范围查询；有跨会话的持久状态与知识整合。README 更激进地宣称"消除了 RAG 和知识图谱的缺陷"（`README.md:28`）——这是营销表述，不是可验证结论。

**生态**：官方自称 60+ 集成，覆盖 Claude Code / Codex / Cursor / Copilot CLI / opencode / Cline 等 coding agent，以及 LangGraph、LlamaIndex、CrewAI、Pydantic AI、OpenAI Agents SDK、n8n、Dify 等（`README.md:239-259`）；每个服务端自带 MCP endpoint（`http://localhost:8888/mcp/{bank_id}/`，`README.md:263-269`）。本次未逐项核实这些集成的完成度。

## 核心机制

以下除标注外均为源码实读。

### retain（写入）

- 分块是纯 CPU 操作，无 LLM；默认每块 3000 字符（`engine/retain/fact_extraction.py:767,835`；默认值 `config.py:1643`）。
- **常规路径下唯一的 LLM 调用是"事实抽取"，每个 chunk 一次**（`fact_extraction.py:2013`，scope `retain_extract_facts`），按 chunk 并发（`engine/retain/orchestrator.py:2808`）。即：N 个 chunk ≈ N 次 LLM 调用（另有结构化输出失败时的重试，`fact_extraction.py:2096`）。
- 实体和因果关系由**同一次调用**一起返回，不额外抽取（`fact_extraction.py:281-286`）；因果抽取默认开启（`config.py:1644`）。
- 抽取后的归一化**不调 LLM**：`EntityResolver` 走 trigram / 模糊匹配 + 共现打分（`engine/memories/pg/entity_resolver.py:453,804,941`）。
- 落库：`documents` / `chunks` / `memory_units` / `unit_entities` / `entities` / `memory_links`（写点分别在 `engine/memories/pg/retain.py:264,502`、`writes.py:419-421,470`、`pg/links.py:210`）。
- 例外：`retain_extraction_mode="chunks"` 时完全跳过 LLM（`fact_extraction.py:3508-3509`）。
- 成本侧参考值：文档写"每 chunk 大约产出 17 条事实"、单批 100 chunk ≈ 1700 条事实、consolidation 每 8 条事实一次 LLM 调用（`configuration.mdx:1725,2277`）。

### recall（读取）

- 四路检索：semantic 与 BM25 合并在**同一条 UNION SQL** 里（`engine/memories/pg/recall.py:28,166,185`）；graph 走 `LinkExpansionRetriever`（`engine/memories/pg/link_expansion.py:114`）；temporal 走 `retrieve_temporal_combined_sql`（`recall.py:374`）；统一入口 `PostgresMemories.recall_unified`（`engine/memories/postgres.py:124`），由 `engine/search/retrieval.py:191` 调用。
- 融合用 RRF，**k=60**，四路等权（`engine/search/fusion.py:29`，签名即 `k: int = 60`）；去重场景换用 `interleave_fusion`（`fusion.py:112`）。
- 重排默认开（`config.py:1949`），用本地 cross-encoder + 组合打分（`engine/search/reranking.py:307,174`），候选上限 300，默认模型 `cross-encoder/ms-marco-MiniLM-L-6-v2`（`config.py:1315,1286`）。
- 默认参数：`budget=mid`、`max_tokens=4096`（`api/http.py:527-528`）→ 内部 thinking budget 三档固定为 100/300/1000，`mid`=300（`config.py:1957-1959`）。
- 文档/博客声称 **recall 路径 0 次 LLM 调用**（`hindsight-docs/blog/2026-05-08-how-hindsight-scales.md:55,59`）。但引擎里存在 `engine/query_analyzer.py`（查询意图分析），本次**没有逐行确认它是否调 LLM**，见「未核实」。

### 存储

- PostgreSQL 14+ + 向量扩展（默认 pgvector；可换 pgvectorscale / vchord / scann），默认用**嵌入式 pg0**（`config.py:1100` `DEFAULT_DATABASE_URL = "pg0"`）——文档自己写明"方便但不推荐生产"（`hindsight-docs/docs/developer/installation.md:37`）。
- Alembic 迁移 **111 个**，每一个都必须经 `run_for_dialect` 做 PG/Oracle 双方言分派，有测试强制（`hindsight-api-slim/tests/test_migration_shape.py:32`）。
- **Oracle 23ai 是真实实现**，不是文档承诺：`oracledb` extra（`pyproject.toml:157-158`）、后端 `engine/db/oracle.py`、Oracle 基线迁移用 `VECTOR(384,FLOAT32)`、CI 有独立 job（`.github/workflows/test.yml:2206`）。
- observations 不是独立表，而是 `fact_type='observation'` 的 `memory_units`（`engine/memories/pg/consolidation.py:292-317`）；knowledge pages 有独立表（`alembic/versions/a9b8c7d6e5f4_add_knowledge_pages.py:46`）。

### 后台：consolidation / mental models

- 以**事件驱动为主**：retain 完成后提交 post-insert maintenance（`engine/memory_engine.py:6826-6854`），前提是 `enable_observations` 与 `enable_auto_consolidation` 都开——**两个默认都是 True**（`config.py:1725-1726`）。
- consolidation 本身是 worker 异步任务（`engine/consolidation/consolidator.py:1469`，跑在 `worker/poller.py`），并且**要额外花 LLM**：批量整理在 `consolidator.py:3282`，近重复裁决 scope 为 `consolidation_dedup`（`:348`）。
- 另有定时任务 `MaintenanceLoop`（`engine/maintenance.py:216-231`）：对"有未整合事实但没在跑"的 bank 重新排期（`:418`），以及按 `trigger.refresh_cron` 刷新 mental model（`:224-226`）。

### 扩展点

- 6 个插槽，每槽最多一个实现：TENANT / MEMORIES / OPERATION_VALIDATOR / MEMORY_DEFENSE / HTTP / MCP（`hindsight_api/extensions/loader.py:173-178`）。
- **开箱即用的只有两个**：单共享 key 的 `ApiKeyTenantExtension` 和正则版 memory defense。真正的多租户 + 鉴权要自己写——`hindsight-extensions/` 里只有 supabase-tenant 和 static-keys-tenant 两个样例，且**不发 PyPI，要自建镜像**（`hindsight-extensions/README.md:12-13,209-230`）。
- Memory Defense：按 bank 开启，对新写入做敏感信息扫描，命中替换为 `[REDACTED:type]` 或整条丢弃（`hindsight-docs/docs/developer/memory-defense/index.md:7-25`）。**只影响未来写入，不回溯扫描已存数据**（同文件 `:15,41-43`）；开源版只实现 `sensitive_data` 一种规则，更强的检测器由 cloud 扩展分发（`hindsight_api/extensions/memory_defense.py:32-39`）。文档写"45 条正则"，源码正则表目测 46 条以上（本次未逐条核对，见「未核实」）。

## 实测与标称数据

**本次没有做任何运行时实测**（原因见「调研方式」），所以下表全部是仓库内的标称值，不是测量值。

**官方标称**：

| 场景 | 数值 | 出处 |
| --- | --- | --- |
| recall 延迟 | 100–600 ms（瓶颈是 CPU 上的 reranker） | `hindsight-docs/docs/developer/performance.md:30` |
| reflect 延迟 | 800–3000 ms（瓶颈是 LLM 生成） | `performance.md:31` |
| retain 延迟 | 500–2000 ms / 批（瓶颈是 LLM 事实抽取） | `performance.md:32` |
| 向量检索 | 10–50 ms（10 万条事实规模） | `performance.md:217` |
| LongMemEval-S | **94.6**（图上第二名为 74.0） | `hindsight-docs/src/components/HomeBenchmarks.tsx:32` |
| LoComo / PersonaMem | 92.0（次优 80.3） / 86.6（次优 84.4） | `HomeBenchmarks.tsx:33-34` |
| 对比口径 | 94.6 vs SuperMemory 81.6 / Zep 71.2 / Mem0 67.6 | `hindsight-docs/blog/2026-05-21-agent-memory-consolidation.md:124` |
| 同一篇博客的自我澄清 | 文中承认 **Chronos 95.6 / Mastra 92.8 / Honcho 90.4 不低于自己**（与 README "most accurate ever tested" 的措辞并不一致） | 同上 |
| 另一篇的对比数字 | 同一家厂商 2026-04 的文章里 Zep 记 63.8、Mem0 记 49.0，与上表 71.2 / 67.6 不一致 | `blog/2026-04-21-hindsight-10k-stars.md:136` |
| 历史口径 | 2026-04 时自称 91.4（同一榜单的更早版本） | `blog/2026-04-21-hindsight-10k-stars.md:50,134,152` |
| 版本性能改进 | 0.10.0 声称 `/health/live` 2476 → 7917 rps、p99 108 → 17 ms | `blog/2026-09-14-version-0-10-0.md:52` |
| 存储侧 | 引用第三方数据：pgvectorscale 相对 pgvector p95 快 28x、吞吐 16x、成本降 60–75% | `docs/developer/configuration.mdx:140-141` |

**第三方实测**：

| 来源 | 说法 | 可信度 |
| --- | --- | --- |
| README 声明 | LongMemEval 数据被 Virginia Tech Sanghani Center 与 The Washington Post **独立复现** | **未核实**：全仓检索这三个名字只命中 `README.md:50` 这一句，没有任何数据、报告或复现脚本 |
| ACL 2026 demo 论文 | 检索到 `aclanthology.org/2026.acl-demo.27` 标题与 arXiv 论文一致（"Hindsight: Structured Agent Memory that Retains, Recalls, and Reflects"） | **未核实**：本环境无法打开页面，只有搜索摘要 |
| 第三方复现数据 | 未找到任何独立第三方的实测数字（延迟/成本/准确率） | — |

**可复现性**：仓库内**没有** LongMemEval / LoComo 的结果文件（`.gitignore` 仍在忽略 `hindsight-dev/benchmarks/locomo|longmemeval/results/`）。数据集、judge 和打分都在**另一个仓库** `vectorize-io/agent-memory-benchmark`（`hindsight-system-evals/README.md:344-388`）；在线榜单由第三个仓库 `hindsight-continuous-performance-monitor` 发布，且 CI 只推 LoComo、**明确不推 LongMemEval**（`.github/workflows/perf-test.yml:242-244`）。能离线跑的是 perf-test（mock LLM + pg0）和 token 计数微基准，不是质量评测。

## 硬件 / 软件门槛

| 项 | 门槛 | 来源 / 性质 |
| --- | --- | --- |
| 内存（full 镜像） | 最低 1.5 GB / 推荐 2 GB（含本地 embedder ~130 MB + reranker ~90 MB 及其运行时） | `installation.md:57`，文档声称 |
| 内存（slim 镜像） | 最低 512 MB / 推荐 1 GB（无本地模型，须配外部 embedding/reranker） | `installation.md:58`，文档声称 |
| 控制台 / PostgreSQL | 128 MB / 512 MB+ | `installation.md:59-61`，文档声称 |
| 磁盘 | full 镜像 ~9 GB（AMD64）、~3.7 GB（ARM64）；slim ~500 MB；自建 CUDA 版约 11 GB | `installation.md:117-120,136`，文档声称 |
| LLM | **事实上必需**：事实抽取、reflect、consolidation 都要 LLM。默认 provider `openai`，默认模型 `gpt-4o-mini`（`config.py:1102,1106`；**注意**文档 `configuration.mdx:280` 写的是 `gpt-5-mini`，与源码冲突） | 已验证（源码） |
| 无外部 LLM 的选项 | ① `provider=none`：确定性写入，放弃 reflect/consolidation；② 内置 llama.cpp：首次自动下载 Gemma 4 E2B GGUF（~3.5 GB），需 `local-llm` extra，Docker 镜像不含它 | `config.py:1127`、`engine/providers/none_llm.py`、`installation.md:124`、`models.mdx:63-66` |
| embedding / reranker | 默认本地（`BAAI/bge-small-en-v1.5`、`cross-encoder/ms-marco-MiniLM-L-6-v2`），首次运行从 HuggingFace 下载 | `config.py:1222-1223,1284-1286`，已验证 |
| 数据库 | 默认嵌入式 pg0；生产需外部 PostgreSQL 14+ 且启用 pgvector（或 pgvectorscale/vchord/scann） | `installation.md:28-45` |
| CPU / GPU | 2 vCPU 纯 CPU 可跑开发；生产瓶颈是本地 cross-encoder，官方建议上 GPU 或换外部 reranker | `installation.md:67` |
| 平台 | Linux / macOS（含 Apple Silicon）/ Windows 都支持；**Intel Mac 只能用 slim**（full 包的本地模型没有 x86_64 wheel，`pip install` 会静默回退到几个月前的旧版本） | `installation.md:13-18` |
| 国内网络 | 文档专门给了中国网络小节：`HF_ENDPOINT=https://hf-mirror.com` + DeepSeek + 本地 embedding + flashrank reranker | `installation.md:363-387` |

## 上手方式

四种部署，全都不需要你先读代码：

```bash
# 1. Docker 单容器（含嵌入式 PG），API :8888、控制台 :9999
docker run -it --pull always --name hindsight -p 8888:8888 -p 9999:9999 \
  -e HINDSIGHT_API_LLM_API_KEY=$OPENAI_API_KEY \
  -v hindsight-data:/home/hindsight/.pg0 ghcr.io/vectorize-io/hindsight:latest

# 2. pip（裸机）
pip install hindsight-api && hindsight-api

# 3. Helm
helm install hindsight oci://ghcr.io/vectorize-io/charts/hindsight --set postgresql.enabled=true ...

# 4. 嵌入 Python 进程
pip install hindsight-all        # HindsightServer（同进程）或 HindsightEmbedded（本地 daemon）
```

关键配置：`HINDSIGHT_API_LLM_PROVIDER` / `_API_KEY` / `_MODEL`（provider 默认模型见 `config.py:1105-1136`）、`HINDSIGHT_API_DATABASE_URL`、`HINDSIGHT_API_LLM_MAX_CONCURRENT`（**默认 32，指向本地 LLM 时务必降到 2 左右**，否则会占满 llama.cpp/vLLM 的槽位把别的客户端饿死，`performance.md:95-101`）、`HINDSIGHT_API_LLM_TIMEOUT`（默认 120 s，本地大模型建议 300）、`HINDSIGHT_API_RERANKER_MAX_CANDIDATES`（默认 300，纯 CPU 建议降到 100）。

客户端三行上手（`README.md:130-145`）：

```python
from hindsight_client import Hindsight
client = Hindsight(base_url="http://localhost:8888")
client.retain(bank_id="my-bank", content="Alice works at Google as a software engineer")
client.recall(bank_id="my-bank", query="What does Alice do?")
client.reflect(bank_id="my-bank", query="Tell me about Alice")
```

coding agent 一条命令接入：`npx @vectorize-io/hindsight-coding-agents install all`（`README.md:255-259`）。

发布物分 4 类，容易买错：`hindsight-api` = `hindsight-api-slim[all]`（含本地模型）；`hindsight-api-slim` = 无本地模型，必须配外部 embedding/reranker；`hindsight-all` = slim[all] + client + embed；`hindsight-all-slim` = 只有 slim。PyPI 上共 6 个包（`.github/workflows/release.yml:86-117`）。

## 坑与风险

1. **写入路径默认就要花钱**。每个 chunk（默认 3000 字符）一次 LLM 调用，之后 consolidation 每 8 条事实再一次。要用它就得接受"写入侧 LLM 成本 + 延迟"，或者退化成 `provider=none` 的纯存储模式（等于放弃它一半的能力）。异步 + Batch API 可省约 50%，仅 OpenAI/Groq 支持（`configuration.mdx:1720`）。
2. **文档与源码有实测冲突**：默认模型文档写 `gpt-5-mini`、源码是 `gpt-4o-mini`（`configuration.mdx:280` vs `config.py:1106`）；Memory Defense 文档写 45 条正则、源码正则表目测更多（本次未逐条核对）。这类不一致说明**文档不能当唯一依据**。
3. **标称 SOTA 无法在仓库内复现**。没有结果文件、数据集和 judge 都在外部仓库、在线榜单又是第三个仓库，且 LongMemEval 明确不进 CI 榜单。"被 Virginia Tech / 华盛顿邮报独立复现"在仓库里只有一句话。
4. **治理是公司主导**：`CONTRIBUTING.md:7-10` 明写"不接受团队外 PR，外部 PR 直接关闭、不审查"——只能提 issue。Cloud 是商业版，开放核边界已经出现（memory defense 的更强检测器在 cloud 侧）。
5. **0.x + 高频迭代 = 接口不稳定**：近 30 天 618 个提交；0.5.0 移除过 BFS/MPFP 与 Hermes 支持，0.10.3 移除了 worker 的 `--max-retries/--log-level`（`hindsight-docs/src/pages/changelog/index.md:12-16`），HEAD 上还有一个 `refactor(openclaw)!:` 的破坏性提交。升级要有回归预算。
6. **体量与复杂度**：核心引擎单文件 2 万行、111 个迁移、PG/Oracle 双方言、Next.js 控制台、Rust CLI、Go/TS 生成客户端。想只抽取"记忆层"嵌进自己系统，实际是接入一个独立服务 + 接受它的 schema，而不是引一个库。
7. **多租户/鉴权要自己写**。默认只有一个共享 key；扩展要自建镜像，官方样例不发 PyPI。
8. **AI 重度参与开发**（如实记录，不是贬义）：`CLAUDE.md` 26 KB 的 Claude Code 指引，`.claude/skills/` 里 `code-review`、`ship-it`（自述自动 review→fix→re-review 后 squash-merge），`git log` 中有 118 个提交带 `Co-Authored-By: Claude`（另有 2 个其他 Co-Authored-By，见「调研方式」）。这解释了它为什么能在 11 个月里堆出 31 万行，但也意味着代码的审计与追责路径和传统项目不同。
9. **隐私默认边界**：默认要把内容发给外部 LLM；Memory Defense 只对新写入生效、不回溯；开源版只有正则级检测。
10. **仓库体量对使用者不友好**：clone 一次 783 MiB（History 里含大量生成物与图片）。

## 结论

**适合谁用**：
- 想给产品加"跨会话、能学习"的记忆层，且**不介意跑一个独立服务 + Postgres + 外部 LLM** 的团队。
- 需要的不只是向量 RAG：要图检索、时间范围检索、事实去重与信念演化（observations）、以及"开局即结论"的 mental model。
- 想接入现成 coding agent 记忆的（Claude Code / Codex / Cursor 等一条命令）。
- 想借鉴架构的人：它的"写侧做重、读侧做轻 + 后台整合"是这份调研里最有价值的部分。

**不适合谁用**：
- 想找一个可内嵌的轻量记忆库（这是服务，不是库；嵌入式包也只是替你把服务拉起来）。
- 数据不能出内网、又不愿意自建本地 LLM 的。
- 需要稳定 API 契约的（0.x，且已经有多次移除型变更）。
- 想以社区身份参与改造的（外部 PR 不受理）。
- 把 94.6 这类分数当独立验证结论来选型的。

**什么情况下值得引入**：先按"写入侧成本"做一次估算——如果你的记忆写入量不大（每用户每天几十条），配合便宜快速模型（官方推荐 `gpt-oss-20b` via Groq）和异步 retain，成本可控，此时它的检索与整合能力明显强于朴素 RAG；如果写入量巨大，先算 Token 账单再决定。运维上，接受 Docker 单容器起步、生产换外部 PG + 多 worker 的路径；如果连运维都不想管，它也有商业 Cloud（本次未评估其定价与功能）。

## 未核实 / 存疑

- **star 数**：本环境无法访问 GitHub 页面（DNS 被解析到 198.18.x.x 的 fake-IP），「4 万 star」只来自厂商博客的搜索摘要。
- **LongMemEval 94.6、LoComo 92.0、PersonaMem 86.6 的真实性**：全部来自厂商自己的仓库文件、网站组件与博客；仓库内无结果数据，数据集与 judge 在外部仓库。**没有任何独立第三方数字被本次调研确认**。
- **Virginia Tech / Washington Post 独立复现**：仓库内只有 README 一句话，无数据、无报告。
- **ACL 2026 demo 收录**：只有搜索结果标题，未读到正文，也未确认 arXiv:2512.12818 与本仓库版本的对应关系。
- **recall 是否真的 0 次 LLM 调用**：博客这么说，但 `engine/query_analyzer.py` 的存在让这一条需要单独确认；本次没有逐行走完。
- **Memory Defense 正则条数**：文档写 45；用简单计数得到 46–47（口径不明），未逐条核对。
- **Cloud 与自托管的功能边界**：Cloud 端不开源，只能看到博客描述（SSO/MFA/审计日志/更强检测器属 Enterprise）；代码级强制程度未核实。
- **商标条款**：全仓只有一处 "Hindsight™"，没找到商标政策文件，是否存在额外限制未核实。
- **Oracle 路径的实际性能**：实现是真的，但没有任何基准数据。
- **retain 每千 token 需要几次 LLM 调用、每百万条记忆的存储占用**：仓库内未找到这类换算数据。
- **本次零运行时验证**：所有延迟、内存、镜像体积数字都是文档声称值，不是我们测出来的。

## 调研方式

- **手段**：
  1. `git clone` 全量历史到 `.cache/hindsight`（5041 文件 / 3573 commits），`git checkout` 保持 main HEAD `a9eeed77`；所有 `文件:行号` 均来自该副本的实读。
  2. 自己读：README、`docs/developer/installation.md`、`performance.md`、`models.mdx`、`rag-vs-hindsight.md`、`memory-defense/index.md`、`configuration.mdx` 关键默认值表、`CONTRIBUTING.md`、`LICENSE`。
  3. 三个子代理分工走读：① 引擎（retain/recall/存储/后台/扩展点/代码规模）；② 基准与可复现性（LongMemEval 数值、论文正文是否在仓库、eval 依赖）；③ 部署门槛、成本、许可证与商业边界、维护与治理。
  4. `git` 统计元数据：提交数/贡献者/提交强度/tag 与日期/代码行数/仓库体积；`git log --grep` 统计 AI 协作痕迹（118 条 `Co-Authored-By: Claude`、2 条其他 Co-Authored-By）。
  5. 网页信息仅通过搜索摘要获得（Hindsight 博客、ACL Anthology、Hermes/其他仓库的 issue），**全部按未核实处理**。
- **当时的限制**：
  - `web_fetch` 与 pdf MCP 的 URL 读取都被 DNS fake-IP（`github.com` → 198.18.0.54）挡在 SSRF 保护之外，本次**无法打开任何网页原文**，也无法读取 arXiv 论文全文；只有 `web_search` 的摘要可用。
  - 没有 LLM API key、没有 Docker/Postgres 环境，因此**没有把服务跑起来**：本文件里没有任何实测性能数据。要实测需要拉 ~9 GB 镜像（或 pip 安装含 torch 的 full 包）+ 下载 HuggingFace 模型 + 一个 LLM key。
  - 子代理报告的数字经过抽查：配置默认值、基准组件、贡献者统计均已本人复核；其中一位子代理给出的贡献者提交数与本机测量不符（其 Top5 之和超过总提交数），本文**采用本机测量值**并已弃用该组数字。
- **复现方式**：
  ```powershell
  git clone https://github.com/vectorize-io/hindsight.git
  cd hindsight && git checkout a9eeed77f9bce99d8f97d39a30d3281747300824   # 本次走读基准
  # 或 git checkout eb6df499d35300e5b2f3f029b2e6adda04ed90f8             # 本次锁定的发布版 v0.10.3
  ```
  文档内的行号在版本更新后会漂移，请以 commit hash 为准。
