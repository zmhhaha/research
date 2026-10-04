# Strata

- **调研日期**：2026-10-04
- **上游地址**：https://github.com/Niko1221/Strata
- **调研版本 / commit**：`v0.1.39` = `6f32ec070f23ced9f50e704d854d775da52591ab`（2026-10-04）
- **本地副本**：`D:\github\Strata`（完整历史，845 commits）
- **许可证**：MIT

## 一句话结论

把 125B 的 MoE 大模型（Qwen3.8-Flash-Next）拆到"显卡 + 内存 + SSD"上协同跑，12 GB 显存 + 32 GB 内存的普通游戏 PC 就能用，MIT 开源、有完整论文。

值得关注的点是它**不是** llama.cpp 的又一次封装：它有一个真实的技术增量（MTP 自推测解码），并且工程完成度很高（一键安装、OpenAI/Anthropic 双协议、MCP server、7 种语言 README）。需要警惕的点是**内存门槛被营销话术掩盖**——仓库简介写"8GB+"，正文写死 12 GB 起，而真正的门槛是 32 GB 内存起、64 GB 才能跑全部档位。

## 项目概况

| 项 | 值 | 来源 |
| --- | --- | --- |
| 仓库地址 | https://github.com/Niko1221/Strata | 已验证（git remote） |
| 许可证 | MIT（Copyright 2026 Niko1221 and the Strata contributors） | 已验证（LICENSE） |
| 语言 / 技术栈 | C++/CUDA/HIP 引擎 + Python 服务端 + Web 应用 | 已验证 |
| 提交数 | 845 | 已验证（git log） |
| 贡献者 | 88 人；Niko1221 518 次提交、maxfridbe 69、Jakub Luwierski 28、sergiywith 24、Jeremiah Ritchey 20 | 已验证（git shortlog） |
| 最新版本 | v0.1.39，共 40 个标签 | 已验证（git tag） |
| 首次提交 | 2026-09-24，"Strata: Qwen3.8-Flash-Next on a 12-24 GB NVIDIA GPU + 64 GB RAM, one-click install" | 已验证 |
| 仓库体积 | 44.1 MB（pack 17.87 MiB） | 已验证（git count-objects） |
| 代码构成 | 235 个 .cpp、167 个 .hpp、140 个 .json、103 个 .py、56 个 .cu、51 个 .md | 已验证（git ls-files） |
| 核心代码量 | `src/` 160 文件 68,796 行；`include/` 110 文件 10,116 行；`serve/` 22 文件 11,076 行 | 已验证 |
| 热度（star） | 媒体称"开源 5 天 1078 stars" | **未核实**（无法访问 GitHub 页面） |

版本节奏：9-24 首提交 → 10-04 已是 v0.1.39，**10 天内 39 个版本**。开发极快，也意味着接口和默认配置仍在频繁变动。

## 它解决什么问题

**痛点**：Qwen3.8-Flash-Next 这类 125B MoE 模型通常需要几百 GB 显存的服务端。消费级显卡只有 12-24 GB。

**思路**：不追求"塞进显存"，而是把整个 PC 当成一个异构计算单元。

**同类对比**（部分为未核实推断）：

| 方案 | 关系 |
| --- | --- |
| llama.cpp / ggml | **有从属关系**：Strata 的 `third_party/ggml` 就是 llama.cpp 的 ggml（MIT），并复用了它的 i-quant 内核 |
| ktransformers | 同一族思路（MoE 专家卸载到 CPU/内存），**未核实**具体差异 |
| Splash / ninfer / HyperQwen | 文档明确致谢"借鉴了其思路"，具体借鉴点**未核实** |

Strata 的差异化主要在**工程完成度**而非算法：一键安装、双协议 API、MCP server、多语言文档、社区 benchmark 流程。

## 核心机制

以下均来自 `docs/HOW_IT_WORKS.md` 与 `README.md`（已验证）。

### 1. 三层异构分工

模型是 **24,576 个专家**，而每个 token **只需要其中 10 个**。

| 层 | 装什么 | 干什么 |
| --- | --- | --- |
| **显存** | attention 与 DeltaNet mixer、gated-residual 权重、router、共享专家、输出头、MTP 草稿层、KV cache（64K 起只留最常读的部分，其余从内存流式读）、**专家缓存** | 每 GB 显存约多放 ~700 个专家；显存越大越快，因为每个留在 GPU 上的专家就不必由 CPU 算 |
| **内存** | 全部 24,576 个专家（pinned） | CPU 用 AVX-512/AVX2 内核对"不在 GPU 上"的专家**就地计算**，与 GPU 并行，互不等待 |
| **SSD** | 28.8 GB 的 n-gram 查找表 | 每个 token 只读其中几行，走 OS cache |

专家缓存**会在对话过程中自适应**——它持续学习哪些专家被问得最多。CUDA 与 HIP 共用同一套引擎，只是分别编译。

### 2. MTP 自推测解码（"先猜后验"）

这是 Strata 相对同类方案的**实质技术增量**：

- 模型**自带的 MTP 层**草拟最多 3 个 token，然后一次前向（48 层全过）**批量校验**，平均每次通过产出 **2.4–3.2 个 token**。
- 官方称同样质量下快 **1.6–1.8x**。因为是"小助手只负责猜、大模型逐个决定"，所以输出结果与不用 MTP 时**完全一致**。
- 另有 **prompt lookup**：当回复在复述上下文时（改代码、引用文本），直接从早先的副本草拟最多 5 个 token。但仅在实测收益为正时才启用——代码编辑快 6-11%，其他文本无变化。

### 3. 长文本与多轮

- prefill 按 **最大 8,192 token** 一块处理，所以长文档/代码库读取速度 **>1,000 token/s**。
- 下一层的专家在**当前层 attention 计算期间**就通过 PCIe 流式传到 GPU（预取）。
- 首条消息之后，Strata 保留对话状态、只读新增部分，所以追问"秒开"。

## 实测与标称数据

### 官方标称（`README.md`、`docs/MODELS.md`，已验证为仓库原文）

**NVIDIA RTX 5070（12 GB）+ Ryzen 5 7600 + 64 GB RAM：**

| 量化档 | 短对话出词 | 128K 上下文出词 | 读 prompt |
| --- | ---: | ---: | ---: |
| Q2_0 | 94 tok/s | 76 tok/s | 2,650 tok/s |
| IQ2_XS | 79 tok/s | 63 tok/s | 2,090 tok/s |
| IQ3_XXS | 62 tok/s | 49 tok/s | 1,750 tok/s |
| IQ3_S | 53 tok/s | 46 tok/s | 1,620 tok/s |
| Coder (IQ1_M) | 55 tok/s | 43 tok/s | 2,180 tok/s |

**AMD RX 9070 XT（16 GB）+ Ryzen 9 3900X + 47 GB RAM（Linux）：**

| 量化档 | 短对话出词 | 128K 上下文出词 | 读 prompt |
| --- | ---: | ---: | ---: |
| Q2_0 | 60 tok/s | 48 tok/s | 1,160 tok/s |
| IQ2_XS | 52 tok/s | 36 tok/s | 1,110 tok/s |
| Coder (IQ1_M) | 44 tok/s | 33 tok/s | 1,420 tok/s |

官方估算：RTX 3090（24 GB）约 **100-140 tok/s**。

仓库明确标注了测量口径："NVIDIA 一行 Q2_0 用引擎 0.1.36，其余行用 0.1.26（4K 回答、32K prompt）"。**这种把版本和口径写清楚的做法是加分项。**

### 第三方实测

| 硬件 | 结果 | 出处 | 可信度 |
| --- | --- | --- | --- |
| 未指明 | ~30 tok/s | 什么值得买文章 | 低（二手转述） |
| RTX 3060 12GB（Unraid） | ~30 tok/s（换 Swift 后） | lcz.me 论坛 | 中 |
| RX 7900 XTX 单卡 | PP 1300+ / TG 55+ | lcz.me 论坛 | 中 |
| W7800 48GB，100K 上下文 | 76.5 tok/s（no-thinking 均值） | lcz.me 论坛 | 中 |
| IQ2_XS 单卡，424K 上下文 | 155 tok/s（同帖列了"三个不利发现"） | lcz.me 论坛 | 中低 |

**交叉验证的结论**：官方 94/79 tok/s 是 **12 GB 卡 + 短对话**的上限值；第三方报出的 30 tok/s 量级来自**更弱的卡或更长的上下文**。二者不矛盾——**但仓库简介里的"8GB+"和媒体标题的"12G 卡跑 125B"容易被读成普遍性能**，实际要按上表逐档对照。仓库自己的社区 benchmark 模板（`docs/COMMUNITY_BENCHMARKS.md`）明确要求"把实测值和估算值分开标注"，说明维护者知道这里的口径风险。

## 硬件 / 软件门槛

**这段是全文最该看的部分**——`README.md` 的表格（已验证）：

| | 要求 |
| --- | --- |
| **显卡** | NVIDIA RTX 20/30/40/50 系，或 AMD RX 7900 XT/XTX、7800 XT、7700 XT、9060 XT、9070/9070 XT、Radeon AI PRO R9700、RX 6800/6900 系。**需要 12 GB 显存或更多** |
| **内存** | **32 GB 起**。内存决定能跑哪个档位，**64 GB 才能跑全部档位** |
| **磁盘** | **约 80 GB 可用**，建议 SSD（首次启动快很多） |
| **系统** | Windows 10/11 或 Linux，较新的显卡驱动 |

补充（`docs/MODELS.md`，已验证）：

- 模型下载本体 **66-76 GB**（三个较小档位）；首次启动还要拉 MTP 草稿层约 6 GB（带图像再 +1 GB）
- 启动时会**往内存里压 35-55 GB 并锁定一部分给显卡**，此时"PC 可能无响应 1-3 分钟"（首次最久）——文档明说这是正常现象
- **是否装得下**：专家的体积是 Q2_0 34 GB / IQ2_XS 35.5 GB / IQ3_XXS 43 GB / IQ3_S 50 GB / Coder 23 GB，**内存需 ≥ 专家体积 + 约 10 GB**（留给 Windows 和其他程序）
- **低内存模式**：内存装不下全部专家时，改为从模型文件按需映射，内存只留显卡装不下的部分。代价是专家从 SSD 读，**慢很多**

> **营销话术与正文的落差**：仓库简介（GitHub 上显示的描述）写 "8GB+ NVIDIA GPU"，但 `README.md` 第 58 行写死"需要 12 GB 显存或更多"，正文示例全是 12 GB 卡起步。**以正文为准。**

## 上手方式

```bash
git clone https://github.com/Niko1221/Strata.git
```

**Windows**：双击 `START-HERE.bat`
**Linux**：`./setup.sh`

安装器会问三个问题（模型与档位、上下文长度、是否读图），全部回车用推荐值即可。然后下载模型（约 70 GB）并启动，浏览器打开 `http://127.0.0.1:8080`。下载中断后重跑会**断点续传**。

**接口**：

| 用途 | 地址 |
| --- | --- |
| Web 应用（Chat / Monitor / About） | `http://127.0.0.1:8080` |
| OpenAI 兼容 | `http://127.0.0.1:8080/v1`（任意 API key、任意模型名都行） |
| Anthropic 兼容 | `http://127.0.0.1:8080/v1/messages`（Claude Code: `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`） |
| OpenAI Responses API（Codex CLI 等） | `/v1/responses` |

**其他命令**：

```bash
START-HERE.bat --setup --family coder            # 装 Coder 版（32 GB 内存能跑）
START-HERE.bat --setup --family swift --model IQ2_XS   # 装 Swift 1.5（想得快、答得早）
START-HERE.bat --calibrate                       # 在本机实测几种引擎配置，取最快（5-10 分钟，目前仅 NVIDIA）
START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>   # 给手机/其他电脑用，务必设 key
START-HERE.bat --setup --low-ram on|off          # 强制开关低内存模式
UPDATE.bat                                       # 只更新不启动（Linux: ./update.sh）
```

**给 AI 代装**：仓库提供了一个面向 AI 助手的安装入口——把一句话贴给 Claude Code / Cursor / Codex 等：

```text
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

它还会检查显卡/内存/磁盘、自动选档位、装完启动并告诉你怎么接应用。另有 **MCP server**（`docs/MCP_SERVER.md`），可以把安装、启停做成工具调用。

## 坑与风险

1. **内存才是真门槛，不是显存。** 简介写 8 GB，正文写 12 GB，但实际是 32 GB 内存起步。按「显卡够了就能跑」去准备会踩坑。
2. **启动时机器会假死 1-3 分钟。** 官方承认。不知情的人会以为装坏了，去强杀进程。
3. **Coder 版中文会坏。** 仓库自己写了 issue #438：Coder 只保留每层 512 个专家中的 256 个（按代码数据挑的），**中文等 CJK 文本会出现答案错误或循环**。要中文就用 Q2_0 / IQ2_XS / IQ3_S（保留全部专家）。
4. **默认一次只答一个请求**，其余排队；开并发要设 `"parallel": 2`，而 12 GB 卡上这会拖慢每一个回答。
5. **超长 prompt 首条很慢**：约每 30,000 token 要 1 分钟。
6. **10 天 39 个版本**，接口和默认配置都在动。社区需要自己打补丁才能覆盖某些硬件组合（如 `ExTV/strata-5090-4070` 为 0.1.33 打的分层/预取补丁）。要跟版本就得跟得勤。
7. **许可证不是单一 MIT**：主仓库是 MIT，但 `third_party/ggml`（MIT）、Web 应用字体（SIL OFL 1.1）、`data/experimental-speed-projection` 里的向量（**Qwen Community License 1.0**）各有各的许可证。**模型文件不在仓库里**，各模型自己的许可证适用——商用前要逐个确认。
8. **AMD 读图在 Windows 上还不行**（Linux 上走处理器）。Intel Arc 需要自己在 Linux 上从源码编译。

## 结论

**适合**：

- 有 12 GB+ N 卡（或列表内的 A 卡）+ **32-64 GB 内存** + 80 GB SSD 空间，想在本机跑一个真正的大模型
- 想本地接 Claude Code / Cursor / Codex 等编码代理，且不希望代码出本机
- 需要 OpenAI 和 Anthropic 双协议本地端点
- 关注 MoE 卸载 + 推测解码工程实现的人（有论文、有 845 次提交的完整代码可读）

**不适合**：

- 内存只有 16 GB 或更少——直接出局
- 只想要一个通用本地推理框架跑各种模型——**Strata 只服务 Qwen3.8-Flash-Next 这一个模型**（及其 Coder/Swift/Unsloth 变体），不是 llama.cpp 那样的通用引擎
- 需要高并发服务——默认单请求串行
- 主要用中文且想用 Coder 版——会踩 #438
- 追求长期接口稳定——版本迭代过猛

**是否值得引入**：作为**个人本地大模型方案**值得试，前提是硬件对口（先确认内存）。作为**生产依赖**要谨慎：单模型绑定 + 高速迭代 + 许可证需逐个核对。

## 未核实 / 存疑

| 项 | 状态 | 原因 |
| --- | --- | --- |
| GitHub star / fork / issue 数 | 未核实 | 本环境 `web_fetch` 被沙箱拦截（见下），无法打开 GitHub 页面 |
| 论坛实测数据的原始上下文 | 未核实 | 只有搜索结果标题和摘要，未读到原帖全文 |
| 与 ktransformers 的具体差异 | 未核实 | 未读对方实现，不做对比结论 |
| Splash / ninfer / HyperQwen 具体借鉴了哪部分 | 未核实 | 仓库只写"ideas from"，未指明细节 |
| 论文（`docs/paper/Strata-Paper.pdf`，341 KB）的具体论证与实验 | **未读** | 未解析 PDF；本文所有机制描述均来自 Markdown 文档，未与论文交叉验证 |
| `docs/DETAILS.md`（91 KB）的全部实测数字 | **未读完** | 仅读了 README、MODELS、HOW_IT_WORKS、COMMUNITY_BENCHMARKS |
| 引擎内部实现是否与文档一致 | 未核实 | 未做源码走读（`src/` 68,796 行） |

## 调研方式

- **手段**：`git clone` 拿到完整仓库（845 commits，v0.1.39）后**直接读源文件**——`README.md`、`docs/HOW_IT_WORKS.md`、`docs/MODELS.md`、`docs/COMMUNITY_BENCHMARKS.md`、`LICENSE`，外加 `git log` / `git shortlog` / `git ls-files` 统计。
- **当时的限制**：本机 DNS 被网络层劫持为 fake-ip（`198.18.0.0/15`），DSH 的 `web_fetch` 因 SSRF 防护拒绝非公网 IP；同时沙箱只放行 git 出网，`curl` / `.NET TLS` 全部失败（连 `www.baidu.com` 都不通）。**所以网页调研这条路走不通，全部结论基于 git 通道取得的真实源码**，这也让可信度高于纯搜索。
- **复现方式**：
  ```bash
  git clone https://github.com/Niko1221/Strata.git
  cd Strata && git checkout v0.1.39
  ```
  本地已存在副本：`D:\github\Strata`
- **后续可做**：精读 `docs/DETAILS.md`、解析论文 PDF、走读 `src/` 与 `setup.py` 的选档逻辑、按本机实际硬件做适配性判断。
