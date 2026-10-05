# Strata

- **调研日期**：2026-10-04（第一版）；2026-10-04 补全（精读 `docs/DETAILS.md`、论文 PDF、`setup.py`）
- **上游地址**：https://github.com/Niko1221/Strata
- **调研版本 / commit**：`v0.1.39` = `6f32ec070f23ced9f50e704d854d775da52591ab`（2026-10-04）
- **本地副本**：`D:\github\Strata`（完整历史，845 commits）
- **许可证**：MIT（但见「坑与风险」第 7 条，许可证并不单一）

## 一句话结论

把 125B 的 MoE 大模型（Qwen3.8-Flash-Next）拆到"显卡 + 内存 + SSD"上协同跑，12 GB 显存 + 32 GB 内存的普通游戏 PC 就能用，MIT 开源、**有完整论文**。

值得关注的点是它**不是** llama.cpp 的又一次封装：它有一个真实的技术增量（MTP 自推测解码），并且工程完成度很高（一键安装、OpenAI/Anthropic/Responses 三协议、MCP server、多语言 README）。

需要修正两个常见误解：

1. **内存门槛不是「32 GB 起、64 GB 才全」这么粗。** 真实分档要以显卡大小和上下文长度一起看：32 GB 内存 + 24 GB 卡可跑 Q2_0 / IQ2_XS（resident 模式），32 GB + 12-16 GB 卡只剩 Coder；64 GB 能跑全部档位，但 IQ3_XXS / IQ3_S 需把上下文压到 ≤128K（`docs/DETAILS.md:251`、`docs/DETAILS.md:132-134`）。
2. **官方速度不是虚标，但也不是随便一台机器都能跑到。** 论文实测 RTX 5070 + Q2_0 + 4K 上下文 = **94.6 tok/s**（与仓库标称的 94 tok/s 一致），262K 上下文掉到 56.3 tok/s。第三方报的 30 tok/s 量级来自更弱的卡或更长的上下文，**二者不矛盾**。

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
| 核心代码量 | `src/` 162 文件 68,796 行；`include/` 110 文件 10,116 行；`serve/` 35 文件 11,076 行；`tools/` 75 文件 13,307 行 | 已验证（逐目录统计） |
| 论文 | `docs/paper/Strata-Paper.pdf`，9 页，341 KB，署名 "September 2026" | 已验证（已全文解析） |
| 热度（star） | 媒体称"开源 5 天 1078 stars" | **未核实**（无法访问 GitHub 页面） |

版本节奏：9-24 首提交 → 10-04 已是 v0.1.39，**10 天内 39 个版本**。开发极快，也意味着接口和默认配置仍在频繁变动。

## 它解决什么问题

**痛点**：Qwen3.8-Flash-Next 这类 125B MoE 模型通常需要几百 GB 显存的服务端。消费级显卡只有 12-24 GB。

**思路**：不追求"塞进显存"，而是把整个 PC 当成一个异构计算单元。论文把这个思路说得最清楚（`docs/paper/Strata-Paper.pdf` 第 1-3 节）：

- 一个 token 只激活 **6B / 125B** 参数，每层 512 个专家里 router 只挑 **10 个**，所以一个 token 只碰 **2%** 的专家权重。
- 48 层里 **36 层是 Gated DeltaNet**（循环 mixer，状态大小固定而不是随上下文增长），另 **12 层是 Qwen Sparse Attention**（只有 2 个 KV head，无论上下文多长都只看约 2,048 个选中位置）。这两条是该模型能"小卡跑长文本"的架构前提。
- 一个 **51B 参数的 n-gram 嵌入表**由最后三个 token id 寻址，所以它的行**在需要之前就能知道**——这正是把它放 SSD 也够用的原因。

**同类对比**（论文第 9 节给出了明确的谱系，不再是猜测）：

| 方案 | 关系 | 出处 |
| --- | --- | --- |
| llama.cpp / ggml | **有从属关系**：`third_party/ggml` 就是 llama.cpp 的 ggml（MIT），并复用了它的 i-quant 内核 | 论文第 3.6 节、参考文献 9 与 24 |
| KTransformers | 同一族思路（MoE 专家卸载到 CPU/内存），论文引为相关工作 | 论文第 9 节，参考文献 14 |
| Fiddler / Eliseev & Mazur | CPU-GPU 编排的 MoE 推理 | 参考文献 13、22 |
| MoE-Infinity / Pre-gated MoE | 专家缓存与**下一层专家预测** | 参考文献 15、16 |
| PowerInfer | 消费级 GPU 上的热/冷激活 | 参考文献 23 |
| **Splash / ninfer / HyperQwen** | **"一个引擎只服务一个模型"这个想法来自它们** | 论文第 9 节，参考文献 5-8 |
| Marlin / vLLM PagedAttention / FlashInfer / T-MAC | 量化 GPU 内核 / 分页注意力 / 注意力引擎 / 查表式 CPU 内核 | 参考文献 17-20 |

Strata 的差异化主要在**工程完成度**而非算法：一键安装、三协议 API、MCP server、多语言文档、社区 benchmark 流程。论文自己也承认这一点——机制部分引用的大量组件都是别人的。

## 核心机制

以下来自 `docs/HOW_IT_WORKS.md`、`README.md` 与**论文全文**（三者一致的部分视为已验证）。

### 1. 三层异构分工

模型是 **24,576 个专家**（48 层 × 512），而每个 token **只需要其中 10 个**。

论文表 1 给出了一份 2-bit 文件的**字节普查**（`docs/paper/Strata-Paper.pdf` 表 1），这张表比仓库文档更精确：

| 部分 | 文件里占 | 每 token 读 | Strata 放哪 |
| --- | ---: | ---: | --- |
| 路由专家（48 层 × 512） | 34.0 GB | 0.66 GB（2%） | 内存（全部）+ 显存缓存（最常用的约 4,500 个） |
| mixer / attention / 共享专家 / router | 1.8 GB | 1.8 GB | 显存 |
| gated residual（hyper-connections） | 1.3 GB | 1.3 GB | 显存 |
| 输出头 | 0.44 GB | 0.44 GB | 显存 |
| n-gram 嵌入表 | 28.8 GB | ≤ 23 KB | SSD + OS page cache |
| MTP 层 | 0.8 GB（来自 BF16 checkpoint） | 每轮草稿 | 显存 |

论文的总结是：**显存常驻部分约 3.5 GB；专家流量 0.66 GB/token 才是整个设计围绕的量**。

论文图 1 还给了测试机上实测的带宽（这是"为什么这样分层"的定量依据）：显存 12 GB @ 672 GB/s；PCIe 4.0 x16 ≈ 26 GB/s；DDR5 ≈ 41-52 GB/s；NVMe 每 token 16 次行读取。

### 2. MTP 自推测解码（"先猜后验"）

这是 Strata 相对同类方案的**实质技术增量**：

- 模型**自带的 MTP 层**草拟最多 3 个 token，加上最后一个已接受的 token，一起作为**一个 verify window** 过全部 48 层做批量校验。
- 论文实测**每轮产出 2.4-3.6 个 token**（表 4，按档位和上下文不同），仓库文档写 2.4-3.2，量级一致。
- 草稿只有在 MTP 层**置信度 ≥ 50%** 时才进入窗口。
- 论文称同样质量下快 **1.6-1.8x**，并给了对照：纯 greedy 在 4K 是 47-57 tok/s，开 verify window 后 82-92 tok/s（论文第 6 节第 2 条）。
- **输出逐 token 与纯 greedy 完全一致**，并用"故意给错草稿 + 回滚"测试过（论文第 3.3 节）。与 llama.cpp 的 next-token 分布 KL 散度为 0.022，**低于 llama.cpp 自己 CPU 与 GPU 构建之间的 0.058**。
- 另有 **prompt lookup**：当回复在复述上下文时（改代码、引用文本），直接从早先的副本草拟最多 5 个 token。但仅在实测收益为正时才启用——代码编辑快 6-11%，其他文本无变化。

### 3. 一个层里三个 worker 同时干活

论文第 3.2 节描述得比仓库文档细：

1. GPU 跑 mixer 和 router，把选中的 **10 个专家 id 写进一块 pinned 主机内存**（论文叫 "doorbell"）。
2. CPU **自旋在这个 doorbell 上**（而不是等驱动），立刻拆分专家：已在显存缓存里的给 GPU；一部分**由 PCIe 的 DMA 拷贝引擎**在后台搬（i-quant 档位搬 55% 的未命中，Q2_0 只搬 20%——因为 Q2_0 的 CPU 自己就要吃满内存带宽）；剩下的由 CPU 核心**在内存旁边就地算**。
3. CPU 把结果写回 mapped memory，GPU 汇总，下一层开始。

整个 48 层 pass 是**按窗口大小捕获的一张 CUDA graph**，所以主机端从不等同步调用。一层约 0.6 ms @ 4K。

### 4. 专家缓存会跟着对话走

论文第 3.4 节：启动时用**别的 prompt 上记录的 profile** 填满显存缓存；之后**每 4 轮**把当前对话反复问到的专家换进最不常用的槽位，**换的时候 GPU 正在起草**（不阻塞）。论文图 3 的实测：profile 填的静态缓存在 4,500 个槽位上命中 **50%**，加上自适应换入后升到 **约 72%**。

### 5. 长文本与多轮

- prefill 分块：论文写 2,048 token/块，**仓库文档与代码已改为 `--prefill auto`（最大 8,192，可 opt-in 32,768）**——见「文档内部矛盾」。
- 论文实测 prompt 速度**几乎不随上下文衰减**：4K 539 tok/s → 262K 496 tok/s（Q2_0），原因是稀疏注意力只读有界的位置集合。
- 超 8K 后 KV cache 存 8-bit；MTP 层只看**最后 32,768 个位置**，所以在 262K 起草依然便宜。
- **论文当时的最大缺陷已被后续版本补上**：论文第 7 节把"保持对话状态"列为"对 agent 最大的单项改进"，因为当时每个请求都要重读整个 prompt；现在仓库已有 prompt cache / conversation cache（`docs/DETAILS.md:630-657`），Codex CLI 场景下后续每轮复用约 96% 的 prompt，只读新增部分花 1-2 秒。**引用论文时要注明这是论文写作时的状态。**

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

仓库明确标注了测量口径："NVIDIA 一行 Q2_0 用引擎 0.1.36，其余行用 0.1.26（4K 回答、32K prompt）"。**这种把版本和口径写清楚的做法是加分项。**

### 论文实测（**新增：此前未读，现已全文解析**）

论文的测量口径与仓库表**不同**，引用时不可混用：RTX 5070 12 GB + Ryzen 5 7600 + 64 GB DDR5-5200（EXPO 关）+ NVMe + Windows 10；每个长度一条真实代码审查 prompt，生成 256 token，**greedy**，MTP 开（窗口最多 4 token，草稿置信度 ≥0.5），专家缓存按空闲显存自动定尺，**32K 起用 8-bit KV**，每次都是冷启动的完整运行。

**输出速度（tok/s），论文表 3：**

| 档位 | 1K | 4K | 32K | 64K | 128K | 262K |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Q2_0 | 88.7 | **94.6** | 87.5 | 76.4 | 65.1 | 56.3 |
| IQ2_XS | 82.0 | 78.0 | 65.3 | 63.7 | 52.0 | 48.0 |
| IQ3_XXS | 64.6 | 65.6 | 57.3 | 54.4 | 44.8 | — |

**读 prompt 速度（tok/s），论文表 2：**

| 档位 | 1K | 4K | 32K | 64K | 128K | 262K |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Q2_0 | 389 | 539 | 571 | 561 | 543 | 496 |
| IQ2_XS | 332 | 463 | 495 | 486 | 472 | 437 |
| IQ3_XXS | 285 | 410 | 435 | 427 | 414 | — |

> 论文表 2 的注脚：IQ3_XXS 在 262K **没测**——43 GB 专家 + 260K 上下文 + 操作系统会把 64 GB 机器顶到内存极限；**64 GB 上应把 128K 视为 IQ3_XXS 的实际上限**。
>
> 这里的 prompt 数字（539 @4K）明显低于仓库 0.1.26 表格的 1,299 @4K，因为论文测的是**真实代码审查的完整运行**（含 48 层与 n-gram 查找），而仓库表格是单条 prompt 的吞吐测量。**两者不是一个口径。**

**时间与窗口，论文表 4：**

| 指标 | 4K | 128K | 262K |
| --- | ---: | ---: | ---: |
| Q2_0 首 token 时间 | 6.7 s | 236.7 s | 523.8 s |
| Q2_0 每 verify window 产出 | 3.23 token | 2.65 | 2.46 |
| Q2_0 显存里的专家数 | 4,464 | 3,079 | 1,588 |

**一轮 verify window 去哪了（论文表 5，4K，Q2_0）**：总 34.3 ms = GPU 14.6 + CPU 专家 13.4 + 起草 2.0 + 其他 4.3；产出 3.23 token；显存命中率 0.72；CPU 流式读专家 **41 GB/s**（单独可达 52 GB/s，与 GPU 并发时 41 GB/s）。

**论文第 6 节的 13 条发现**（这是论文最有价值的部分，仓库文档没有）：

| # | 发现 |
| --- | --- |
| 1 | **引擎是平衡的**：CPU 等 GPU 约 14 ms，GPU 等 CPU 约 13 ms。只升级一侧最多快约三分之一。 |
| 2 | MTP 推测解码值 **1.6-1.8x**（4K：47-57 → 82-92 tok/s）。比"整模型全在显存里"的引擎报的 2.5-3x 低，因为窗口里每多一个 token 就要多路由一批 CPU 专家。 |
| 3 | **"精确"是能做到的，而且值得坚持**：逐 token 与 greedy 一致，故意给错草稿也一致；与 llama.cpp 的 KL 只有 0.022。 |
| 4 | **自适应专家缓存远胜静态缓存**：profile 静态只服务 50%，换入对话热点后约 72%。 |
| 5 | **层内把一个窗口拆成两组重叠，不划算**：精确但慢约 7%（4K：85.8 vs 91.8 tok/s），因为 dense 权重被读了两遍。 |
| 6 | **CPU 侧最大的收益来自指令组合**：为 512-bit AVX-512 重写 Q2_0 专家内核，把 CPU 推到约 42 GB/s，接近内存能给的量。 |
| 7 | **i-quant 受 CPU 算术限制，不受内存限制**：解码一个码本权重要好几条指令，六个核只有约 25 GB/s。**这就是 IQ3_XXS 只大 25% 却慢得多的原因。** |
| 8 | **缓存要按字节定尺，别按槽位数**：IQ3_XXS 的专家从 1.3 到 2.3 MB 不等；按层定尺而非按最大专家定尺，能装 3,556 个而不是 2,673 个，1K 解码快 13%。 |
| 9 | **真正划算的重叠是拷贝引擎**：让 GPU 直接从内存经 PCIe 读一部分未命中专家。用 copy kernel 做会占 GPU 关键路径，未命中超 20% 就变慢；改由 CPU 线程在规划该层时立刻发起 **DMA**，GPU 的拷贝引擎就能与 CPU 专家和 GPU 自身工作并行。i-quant 档位 55% 的未命中走 PCIe：**IQ2_XS 4K 从 70 → 78 tok/s，IQ3_XXS 从 56 → 65**。Q2_0 只留 20%，因为它的 CPU 已经在吃内存带宽。 |
| 10 | **换缓存不要让任何人等**：旧专家立刻驱逐、新的等拷贝落地才启用，4K 从 91.7 → 94.4 tok/s。 |
| 11 | **读 prompt 的速度不随上下文衰减**（539 @4K → 496 @262K），瓶颈是逐专家 kernel 启动和半精度反量化，只用掉 26 GB/s PCIe 里的约 7 GB/s。 |
| 12 | **长上下文贵在显存而非算力**：262K 时 8-bit KV 让显存只能装约 1,600 个专家（而非 4,500），加上 MTP 草稿接受率下降，Q2_0 从约 95 掉到约 56 tok/s。 |
| 13 | **Windows 上 pinned memory 要小心**：把 34-43 GB 作为一个区间 page-lock 会失败；要按层分区间 + 对剩余部分用 working-set lock，且一次 PCIe 读不能跨两个区间。 |

论文第 8 节的其它显卡数字（RTX 5060 Ti 16GB、RTX 3090 24GB）与仓库 `docs/DETAILS.md` 的估算表**数值一致**，口径也一致（**未实测，±20%**），不再重复列出。

### 第三方实测

| 硬件 | 结果 | 出处 | 可信度 |
| --- | --- | --- | --- |
| 未指明 | ~30 tok/s | 什么值得买文章 | 低（二手转述） |
| RTX 3060 12GB（Unraid） | ~30 tok/s（换 Swift 后） | lcz.me 论坛 | 中 |
| RX 7900 XTX 单卡 | PP 1300+ / TG 55+ | lcz.me 论坛 | 中 |
| W7800 48GB，100K 上下文 | 76.5 tok/s（no-thinking 均值） | lcz.me 论坛 | 中 |
| IQ2_XS 单卡，424K 上下文 | 155 tok/s（同帖列了"三个不利发现"） | lcz.me 论坛 | 中低 |

**交叉验证的结论**：官方 94/79 tok/s 是 **12 GB 卡 + 短对话**的上限值，论文实测 94.6 @4K 与之一致，可信；第三方报出的 30 tok/s 量级来自**更弱的卡或更长的上下文**。二者不矛盾——**但仓库简介里的"8GB+"和媒体标题的"12G 卡跑 125B"容易被读成普遍性能**，实际要按上表逐档对照。仓库自己的社区 benchmark 模板（`docs/COMMUNITY_BENCHMARKS.md`）明确要求"把实测值和估算值分开标注"，说明维护者知道这里的口径风险。

## 硬件 / 软件门槛

**这段是全文最该看的部分。** 仓库正文（`README.md:56-61`，已验证）写的是：

| | 要求 |
| --- | --- |
| **显卡** | NVIDIA RTX 20/30/40/50 系，或 AMD RX 7900 XT/XTX、7800 XT、7700 XT、9060 XT、9070/9070 XT、Radeon AI PRO R9700、RX 6800/6900 系。**需要 12 GB 显存或更多** |
| **内存** | **32 GB 起**。内存决定能跑哪个档位，**64 GB 能跑全部档位** |
| **磁盘** | **约 80 GB 可用**，建议 SSD（首次启动快很多） |
| **系统** | Windows 10/11 或 Linux，较新的显卡驱动 |

### 但 `setup.py` 里的判定逻辑和这张表并不一致（**新增：走读 `setup.py` 4,422 行的结果**）

| 项 | 文档说法 | 代码事实 | 出处 |
| --- | --- | --- | --- |
| 显存门槛 | "需要 12 GB 显存或更多" | **没有可执行的显存门槛**。唯一的显存判断是一句 `warn`，且阈值写的是 **11**、文案才说 12："less than 12 GB of VRAM: Strata will run, but most experts stay on the CPU and it will be slow" | `setup.py:3797-3798` |
| 真正的硬件底线 | 未提 | **compute capability ≥ 7.5**（或 opt-in 的 sm_60-70）+ 驱动版本（CUDA 13 需 580+；CUDA 12 需 Win 528 / Linux 525） | `setup.py:111,631`、`setup.py:106,116` |
| NVIDIA 型号白名单 | 列了一串 | **没有型号表**，只按 `nvidia-smi` 报的 compute capability 判断；`nvidia-smi` 行解析失败的卡会被**静默丢弃** | `setup.py:531-540`、`setup.py:631` |
| AMD 型号白名单 | 列了一串 | **有硬编码白名单**，只有 6 个 gfx：`gfx1100/1101/1200/1201/1030/1031` | `setup.py:1359,1413-1415` |
| 内存硬停线 | 32 GB 起 | `need = min(ram_gb) = 32`（来自 Coder 的 `IQ1_M`），`ram < need - 4` 即 **< 28 GB** 触发 `confirm_risk`（默认停，可用 `--model --yes` 强行继续） | `setup.py:3801-3813` |
| 档位切换点 | 32→Coder / 48→IQ2_XS / 64→IQ3_XXS | **代码里不存在这张表**。默认 family 是 qwen、默认档位是列表第一项 **Q2_0**；**唯一的切换点是 `ram >= 60` → IQ3_XXS**。"64"在 `setup.py` 里没有任何阈值比较，只出现在注释和文案里 | `setup.py:3865,3891` |
| 磁盘 | 约 80 GB | **不是固定值**，按档位和已有文件算：`待下载 + 8 + (Q2_0 且有 AVX-512 时 +40) + (读图时 +1) + (低内存模式时 arena+1)` | `setup.py:4080-4086` |
| 内存够不够 | — | `fits` 判定为 `ram >= 档位ram_gb`；`ram_gb - 8` 以内算 `tight` | `setup.py:3841` |

**代码里真实存在的三个内存分界**：28 GB（硬停线）、32 GB（`IQ1_M` 声明需求）、**60 GB**（推荐档位切换点）。`64` 不是代码里的分界。

### 内存档位对照（`docs/MODELS.md:12-17`，官方正文口径，已验证）

| 你的内存 | 官方推荐 | 说明 |
| --- | --- | --- |
| **32 GB** | **Coder** | 它装得下 32 GB。**但配 24 GB 卡时 Q2_0 / IQ2_XS 也能跑**（低内存 resident 模式），且通用场景更该选它们 |
| **48 GB** | **IQ2_XS**（或 Q2_0，最快） | 更大的档位装不下 |
| **64 GB** | **IQ2_XS**（推荐），或 IQ3_XXS / IQ3_S | 每个档位都装得下（IQ3_S 要少开别的程序） |
| **96 GB 以上** | **IQ3_S**，或 Unsloth UD-IQ4_XS（约 4-bit） | 给最大档位留出余量 |

> **对交接文档的修正**：此前记录的"32 GB 内存起步、64 GB 才能跑全部档位"过于粗糙。准确说法是——32 GB 内存下**通用档位能否跑取决于显卡**：24 GB 卡可跑 Q2_0 / IQ2_XS / Coder（resident 模式，内存里约 16-18 GB 专家、显存拿另外约 18 GB），12-16 GB 卡则只剩 Coder（`docs/DETAILS.md:132-134`）。64 GB 能跑全部档位，但 **IQ3_XXS / IQ3_S 需要把上下文压到 ≤128K**（`docs/DETAILS.md:63-65`、`docs/MODELS.md:16`）。

### 档位体积与需求（三个互不相同的口径，务必别混用）

同一批档位在仓库里有三套数字，含义不同：

| 档位 | ① 下载体积 | ② 专家 arena（内存里） | ③ "RAM+VRAM Requirements" | 每档低内存触发线（arena+10） |
| --- | ---: | ---: | ---: | ---: |
| Q2_0 | 66.4 GB | 34.0 GB | 37.6 GB | 44.0 GB |
| IQ2_XS | 68.0 GB | 35.5 GB | 39.2 GB | 45.5 GB |
| IQ3_XXS | 75.8 GB | 42.9 GB | 47.0 GB | 52.9 GB |
| IQ3_S | 83.6 GB | 50.3 GB | 54.8 GB | 60.3 GB |
| Coder (IQ1_M) | 58.4 GB | 23.4 GB | — | 33.4 GB |
| Unsloth UD-IQ4_XS | 93.7 GB | 59.5 GB | — | — |
| Unsloth UD-Q4_K_XL | 111.3 GB | 77.0 GB | — | — |

- ① 来自 `setup.py:122-152` 的 `MODELS` 表（`download_gb`）
- ② 来自同一张表的 `arena_gb`，以及 `docs/DETAILS.md:247-249`（写 "~34 / ~36 / ~43 GB experts"）
- ③ 来自 `docs/MODELS.md:60-65`，含义是"内存+显存合计需求"，**不能和 ① ② 互换引用**

### 磁盘：别只记住"约 80 GB"

基线（`docs/DETAILS.md:307`、`docs/INSTALL.md:19`）：模型本体约 70-80 GB，MTP 层约 6 GB（带图像再 +1 GB）。但下面几项常被漏掉：

| 额外占用 | 量 | 条件 | 出处 |
| --- | ---: | --- | --- |
| Q2_0 的 AVX-512 重打包 | **+40 GB** | CPU 有 AVX-512 且 family=qwen（一次性） | `setup.py:4083`、`docs/DETAILS.md:307` |
| 低内存模式的 `experts.bin` | **+23-50 GB** | 开 `--mmap-experts` | `docs/DETAILS.md:122-123` |
| ROCm | **+约 10 GB** | Linux + AMD，安装器装进 `.venv` | `docs/INSTALL.md:19,68` |
| 固定余量 | +8 GB | 始终 | `setup.py:4082` |

按公式代入（推论）：IQ3_S + 关图像 + 低内存关 ≈ **91.6 GB**；同配置开低内存 ≈ **142.9 GB**；Q2_0 + AVX-512 ≈ **114.4 GB**。数据目录整体量级是 **70-120 GB**（`docs/DETAILS.md:311`）。

> **对交接文档的修正**：此前记录的"磁盘约 80 GB"只在"默认 qwen 档位 + NVIDIA + 非 AVX-512"下成立。应理解为**80 GB 起，按配置可到 120-140 GB**。另注：0.1.31 起若无 `experts.bin` 会直接映射 GGUF，**省掉那 23-50 GB 的拷贝**（`docs/DETAILS.md:160-166`）。

### 其它门槛与行为

- 模型下载本体 **66-84 GB**（按档位）；首次启动还要拉 MTP 草稿层约 6 GB（带图像再 +1 GB）
- 启动时会**往内存里压 34-55 GB 并锁定一部分给显卡**，此时"PC 可能无响应 1-3 分钟"（首次最久）——文档明说这是正常现象。注意仓库四处对首启内存量的写法不一致（34-43 / 34-55 / 35-55 / 32-62 GB，见「文档内部矛盾」）
- **低内存模式**：内存装不下全部专家时，改为从模型文件按需映射，内存只留显卡装不下的部分。代价是专家从 SSD 读，**慢很多**。触发线是**逐档位不同**的"专家体积 + 10 GB"（`setup.py:2343,2350`），不是固定阈值

> **营销话术与正文的落差依然存在**：仓库简介（GitHub 上显示的描述）写 "8GB+ NVIDIA GPU"，但 `README.md:58` 写"需要 12 GB 显存或更多"，而代码只对 <11 GB 发警告。**以正文为准；从代码看，12 GB 是"体验门槛"而不是"能否启动的门槛"。**

## 上手方式

```bash
git clone https://github.com/Niko1221/Strata.git
```

**Windows**：双击 `START-HERE.bat`
**Linux**：`./setup.sh`

安装器会问三个问题（模型与档位、上下文长度、是否读图），全部回车用推荐值即可。然后下载模型（约 70 GB）并启动，浏览器打开 `http://127.0.0.1:8080`。下载中断后重跑会**断点续传**。

**模型从哪来**（`setup.py:65-92`，**新增**）：

| family | HuggingFace 仓库 | 版本 |
| --- | --- | --- |
| `qwen` | `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` | 钉死 40 位 revision |
| `swift` | `ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF` | 钉死 40 位 revision |
| `coder` | `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-Coder-GGUF` | 钉死 40 位 revision |
| `unsloth` | `unsloth/Qwen3.8-Flash-Next-GGUF` | 钉死 40 位 revision |

- 下载**不需要 token**：请求头只有 `User-Agent: strata-setup`，全文件无 `Authorization` / `HF_TOKEN`（`setup.py:1045,1068`）
- 可用 `HF_ENDPOINT` 指向镜像（如 `hf-mirror.com`）
- 引擎来自 GitHub Releases，llama.cpp 源码 zip 也钉了固定 commit
- **校验**：只有 Unsloth 两个档位带官方 sha256（`setup.py:155-173`）；其余档位改为读 GGUF 张量目录核对文件长度是否覆盖全部张量（`setup.py:1233-1251`）

**接口**（`docs/DETAILS.md:473-482`，比第一版更全）：

| 用途 | 地址 |
| --- | --- |
| Web 应用（Chat / Monitor / About） | `http://127.0.0.1:8080` |
| OpenAI Chat Completions（流式/非流式、工具） | `POST /v1/chat/completions` |
| Anthropic Messages（流式/非流式、工具） | `POST /v1/messages`（Claude Code: `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`） |
| OpenAI Responses（流式/非流式、工具、**无状态**） | `POST /v1/responses` |
| 模型列表 / 健康 | `GET /v1/models`、`GET /models`、`GET /health` |
| 模型属性 | `GET /props`（也接受 `?model=<id>`） |
| 当前在做什么 | `GET /status`、`GET /slots` |
| Monitor 页的全部数据 | `GET /metrics` |
| MCP server 状态与工具 | `GET /mcp` |
| 手工装卸模型 | `POST /v1/load`、`POST /v1/unload`（忙时 409、未知模型 404） |
| 动态让出显存 | `POST /v1/vram` with `{"reserve_mib": 8000\|0\|null}` |

**请求字段要点**（`docs/DETAILS.md:488-737`）：

- 任意 API key、任意模型名都行；**别名**可用 `aliases` 配置
- 思考控制：OpenAI 侧 `reasoning_effort`（不设则**默认 high**）、`reasoning:{"effort":...}`、`chat_template_kwargs:{"enable_thinking":false}`；Anthropic 侧 `output_config:{"effort":...}`、`thinking:{"type":"disabled"}`
- `reasoning_budget_tokens` 是**硬预算**（请求级优先，`0` = 无）；但 Anthropic 的 `thinking.budget_tokens` **只是选档**，不是硬预算
- 采样：`temperature` / `top_p` / `top_k` / `min_p` / `seed` 逐请求生效；**`top_k` 上限 64**（`0` 或 >64 都等于用满 64）；惩罚项 `presence_penalty` / `frequency_penalty` / `repetition_penalty` 配 `penalty_last_n`（默认 64）。配置里没有 `sampling` 块时，不带采样字段的请求**走 greedy**
- **上下文超出会被拒绝（400），绝不静默截断**；agent 常要满额输出时可用 `"fit_max_tokens": true` 改成截到剩余空间
- **自定义字段**：`strata_mcp`（布尔，让该请求用 MCP 工具）、`experimental_speed_projection`（逐请求开关）。注意 **`parallel` 和 `low_ram` 都不是请求体字段**——前者是配置文件键，后者是安装器/Docker 开关
- Responses API 是**无状态**的（不存任何东西），`previous_response_id` / `conversation` / `background` 以及 hosted tools 都被拒绝或剔除；`tool_choice` 的 `"none"` 能隐藏工具，但**无法强制**指定工具

**其他命令**：

```bash
START-HERE.bat --setup --family coder                # 装 Coder 版（32 GB 内存能跑）
START-HERE.bat --setup --family swift --model IQ2_XS # 装 Swift 1.5（想得快、答得早）
START-HERE.bat --calibrate                           # 在本机实测几种引擎配置，取最快（5-10 分钟，仅 NVIDIA；只保留快 >3% 的设置）
START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>   # 给手机/其他电脑用，务必设 key
START-HERE.bat --setup --low-ram on|off              # 强制开关低内存模式（resident|mmap 可强制变体）
START-HERE.bat --check                               # 只检查这台 PC，不安装
UPDATE.bat                                           # 只更新不启动（Linux: ./update.sh）
```

主要默认值（`setup.py`，**新增**）：上下文按 `min(GPU 规则, RAM 规则)` 推荐（卡 <14 GB → 32768，<20 GB → 65536，否则 131072）；`--kv` 在 ctx ≤8192 用 fp16、否则 int8；`--vision` 默认 **no**；`--draft-vocab` 默认含 CJK；`--low-ram` 默认 **auto**；`--vram-reserve-mib` 默认 700；端口 8080；`--host` 默认 `127.0.0.1`。

**给 AI 代装**：仓库提供了一个面向 AI 助手的安装入口——把一句话贴给 Claude Code / Cursor / Codex 等：

```text
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

它还会检查显卡/内存/磁盘、自动选档位、装完启动并告诉你怎么接应用。另有 **MCP server**（`docs/MCP_SERVER.md`），可以把安装、启停做成工具调用。

## 坑与风险

1. **内存才是真门槛，不是显存。** 简介写 8 GB，正文写 12 GB，但代码里对显存只发一句警告（阈值还是 11 GB），真正的硬停线来自内存：**<28 GB 直接劝停**。按「显卡够了就能跑」去准备会踩坑。反过来，**"32 GB 只能跑 Coder"也是错的**——配 24 GB 卡时 Q2_0 / IQ2_XS 能跑。
2. **启动时机器会假死 1-3 分钟。** 官方承认。不知情的人会以为装坏了，去强杀进程。另注：**放进 Windows 任务计划程序启动会慢 24 倍**（0.05 GiB/s ≈ 13-14 分钟，而非 1.4-1.5 GiB/s ≈ 35 秒），需改两个设置（`docs/DETAILS.md:375-383`）。
3. **Coder 版中文会坏。** 仓库自己写了 issue #438：Coder 只保留每层 512 个专家中的 256 个（按代码数据挑的），**中文等 CJK 文本会出现答案错误或循环**。要中文就用 Q2_0 / IQ2_XS / IQ3_S（保留全部专家）。
4. **默认一次只答一个请求**，其余排队。开并发要设 `"parallel": 2`（取值域 **2-8**，实际由显存能装多少专家决定），而 12 GB 卡上**每一个并发档位都是负收益**：单请求并发 2 槽 -11%、4 槽 -24%，总吞吐也降 11-24%。作者自己的结论是"槽位买到的是**等待时间**，不是速度"。另外**槽位下不跑 MTP 草稿、也不套用重复惩罚**。
5. **超长 prompt 首条很慢**：论文实测 Q2_0 首 token 时间 4K 6.7 s → 262K **523.8 s**（约 8.7 分钟）。仓库文档另写"约每 30,000 token 要 1 分钟"。
6. **10 天 39 个版本**，接口和默认配置都在动。社区需要自己打补丁才能覆盖某些硬件组合（如 `ExTV/strata-5090-4070` 为 0.1.33 打的分层/预取补丁）。要跟版本就得跟得勤。
7. **许可证不是单一 MIT**：主仓库是 MIT，但 `third_party/ggml`（MIT）、Web 应用字体（SIL OFL 1.1）、`data/experimental-speed-projection` 里的向量（**Qwen Community License 1.0**）各有各的许可证。**模型文件不在仓库里**，各模型自己的许可证适用——`setup.py` 里**只有 Swift 一个 family 带了许可证 URL**，qwen / coder / unsloth 的许可证在代码里**未见**（`setup.py:196`）。商用前要逐个确认。
8. **AMD 读图在 Windows 上还不行**（Linux 上走处理器）。Intel Arc 需要自己在 Linux 上从源码编译，且 `setup.py` 里**没有 Intel 分支**——`--backend sycl` 只是转交给 `sycl/setup_intel.py`。
9. **AMD 多卡在 Windows 上不可用**：`--gpus` 会直接 fail（"Linux-only for now"，`setup.py:3742-3743`）。
10. **Web 界面/端点有暴露面**：默认只听 `127.0.0.1`，改成 `--host 0.0.0.0` 时**必须设 API key**；另有 `allowed_hosts` / `trusted_origins` / `cors_origins`（CORS 默认关）。
11. **Responses API 是无状态的**：客户端每轮都要发完整对话（Codex 用 `store: false` 时就是这么做的），不要期待服务端记住上下文。

## 结论

**适合**：

- 有 12 GB+ N 卡（或列表内的 A 卡）+ **32-64 GB 内存** + 80-140 GB SSD 空间，想在本机跑一个真正的大模型
- 想本地接 Claude Code / Cursor / Codex 等编码代理，且不希望代码出本机
- 需要 OpenAI、Anthropic、Responses **三协议**本地端点
- 关注 MoE 卸载 + 推测解码工程实现的人——**现在有论文可读**（9 页，含 13 条实测发现），且有 845 次提交的完整代码

**不适合**：

- 内存只有 16 GB 或更少——直接出局（16-24 GB 区间基本也只能靠低内存模式硬撑）
- 只想要一个通用本地推理框架跑各种模型——**Strata 只服务 Qwen3.8-Flash-Next 这一个模型**（及其 Coder/Swift/Unsloth 变体），不是 llama.cpp 那样的通用引擎
- 需要高并发服务——默认单请求串行，而 12 GB 卡上开并发是纯亏
- 主要用中文且想用 Coder 版——会踩 #438
- 追求长期接口稳定——版本迭代过猛
- Windows + AMD 且想多卡或读图

**是否值得引入**：作为**个人本地大模型方案**值得试，前提是硬件对口（**先确认内存，再确认显卡能否让内存方案成立**）。作为**生产依赖**要谨慎：单模型绑定 + 高速迭代 + 许可证需逐个核对。

## 论文说了什么（新增小节）

论文（`docs/paper/Strata-Paper.pdf`，9 页）是这份调研里**可信度最高的一份材料**，因为它同时给了机器、口径、每个数字的测量条件，并且**明确区分了"实测"与"估算"**（第 4 节 vs 第 8 节）。它值得单独读的三点：

1. **第 5 节"时间去哪了"**：一轮 verify window 的毫秒级拆分（GPU / CPU 专家 / 起草 / 其他），以及"CPU GB/s"实测值。这是判断"我的机器该升级哪一块"的唯一定量依据——结论是**引擎两头平衡，单升级一侧最多快三分之一**。
2. **第 6 节 13 条发现**：每条都带对照实验。最有价值的是第 7 条（i-quant 受 CPU 算术限制，解释了为什么 IQ3_XXS 只大 25% 却慢很多）和第 3 条（推测解码的输出精确性可验证，KL 0.022 < llama.cpp 自身构建差异 0.058）。
3. **第 7 节"什么能让它更快"**：作者自己列出的未做事项——保持对话状态（**后来已实现**）、跨层重叠、i-quant 的 CPU 查表内核（T-MAC 风格）、prompt 路径的融合 8-bit kernel、温度采样需要拒绝采样。**读这一节可以判断这个项目的天花板在哪。**

论文的**已知局限**（作者自述 + 我们核实）：

- 只在**一张卡**（RTX 5070 12 GB）上真测过；其它卡全是估算（±20%）。
- 论文写作时的"每个请求都要重读整个 prompt"**已经不是当前状态**（仓库已有 conversation cache）。
- 论文写 prefill 分块 2,048、且"采样只有 greedy"——代码已经变成 `--prefill auto`（最大 8,192），greedy 之外的支持情况本次未核（见「未核实」）。
- 论文没有 benchmark 与真实用户场景的端到端对比，也没有与 ktransformers 等方案的同机对测。

## 文档内部矛盾与文档-代码差异（新增小节）

调研过程中发现的**上游自身不一致**，引用时需择一并注明：

| # | 冲突点 | 各方说法 | 出处 |
| --- | --- | --- | --- |
| 1 | **上下文范围** | "8K to 256K" / "(8K-262K)" / "384K 和 512K（实验）" / MCP 参数 `context` 8192…524288 | `docs/DETAILS.md:330`、`docs/DETAILS.md:576`、`docs/INSTALL.md:195-196`、`docs/MCP_SERVER.md:114` |
| 2 | **首启占用的内存** | "34-43 GB" / "34-55 GB" / "35-55 GB" / "32-62 GB" | `docs/DETAILS.md:345-346`、`docs/DETAILS.md:1002`、`docs/TROUBLESHOOTING.md:9`、`docs/INSTALL.md:112` |
| 3 | **prefill 分块** | 论文写 2,048；文档现写 `--prefill auto`（最大 8,192） | 论文第 3.5 节 vs `docs/DETAILS.md:205-206` |
| 4 | **0.1.36 的旧基线** | 同一台机器上 0.1.36 段落里的旧值（128K prompt 2,123 / 4K output 89 / 128K output 64.5）与 0.1.26 表格值（2,107 / 93.0 / 73.7）不一致，应是不同次运行 | `docs/DETAILS.md:22-29` vs `docs/DETAILS.md:31-55` |
| 5 | **首启尺寸菜单** | 3 个档位（DETAILS）vs 4 个（INSTALL） | `docs/DETAILS.md:329` vs `docs/INSTALL.md:193-194` |
| 6 | **Unsloth UD-Q4_K_XL 专家体积** | "72 GiB" vs "77 GB" | `docs/DETAILS.md:179` vs `docs/MODELS.md:136` |
| 7 | **推荐档位的代码逻辑 vs README 表** | README 写"32→Coder、48→IQ2_XS、64→IQ3_XXS"，代码里**没有这张表**：默认 family=qwen、默认档位=Q2_0，唯一分界是 `ram>=60→IQ3_XXS` | `README.md:116-121` vs `setup.py:3865,3891` |
| 8 | **显存门槛** | README 写"需要 12 GB 或更多"，代码只对 <11 GB 发警告 | `README.md:58` vs `setup.py:3797-3798` |
| 9 | **磁盘需求** | README 写"约 80 GB"，代码按档位+可选项算（80 / 120 / 140 GB 三档） | `README.md:60` vs `setup.py:4080-4086` |
| 10 | **模型下载体积** | AMD 文档写 "~60 GB"，README 写 "about 70 GB"，MODELS/DETAILS 写 "66-76 GB"（差 6-16 GB，未解释是否指净下载量） | `docs/AMD_HIP.md:79` vs `README.md:99` vs `docs/MODELS.md:67` |
| 11 | **内存门槛的三套体系** | README 的"32 GB 起"= Coder 能跑的下限；MODELS/DETAILS 的"64 GB 才舒服"；MCP server 用的 setup 内部门槛 **28 / 44 / 60 GB**。三套数字都对，但文档没说明彼此关系 | `README.md:59`、`docs/MODELS.md:16`、`docs/MCP_SERVER.md:128` |
| 12 | **MCP 工具数量** | `MCP_SERVER.md` 列 **8 个**工具，`AI_SETUP.md` 只列 **6 个**（缺 `strata_benchmark`、`strata_connect_info`） | `docs/MCP_SERVER.md:112-119` vs `docs/AI_SETUP.md:199` |
| 13 | **磁盘 80 GB 的自相矛盾** | README 的参考机是 Ryzen 5 7600（**有 AVX-512**），按代码逻辑它会额外写约 40 GB 的 Q2_0 重打包——即 README 自己推荐的参考配置实际需要约 120 GB，而不是它写的 80 GB | `README.md:60` vs `setup.py:4083` |

> 这些差异**不影响结论，但影响引用**：本调研对每个数字都保留了原始口径与出处，避免把估算当实测、把某一版本的值当通用值。

## 源码验证（新增小节）

对 `src/`、`include/`、`serve/` 做了**定点的常量级走读**（不是全文走读），目的是把「文档宣称」与「代码事实」分开。结论如下——**其中三项推翻了文档的表述方式**：

| 文档说法 | 代码事实 | 出处 | 判定 |
| --- | --- | --- | --- |
| 24,576 个专家 | **不是常量，是 `48 × 512` 的乘积**；`experts.bin` 恰好装 24,576 个 blob，索引为 `layer * 512 + expert`，无 padding | `include/strata/core/expert_source.hpp:402` | 代码证实（但应表述为乘积） |
| 每个 token 用 10 个专家 | `int64_t k = 10;`（注释 "experts per token"）；router 签名也叫 `router_top10` | `include/strata/core/session.hpp:48`、`include/strata/kernels/router_top10.hpp:17` | 代码证实 |
| 每层 512 个专家 | `int64_t n_expert = 512;`；CPU 内核 `inline constexpr int NE = 512;` | `include/strata/core/layout.hpp:52`、`include/strata/kernels/cpu/expert.hpp:36` | 代码证实 |
| 48 层，36 层 DeltaNet + 12 层稀疏注意力 | `int64_t n_layers = 48;` 且 `qsa_interval = 4`（**每第 4 层是全注意力**：3、7、…、47）——即 12 层 QSA、36 层 DeltaNet，与论文一致 | `include/strata/core/layout.hpp:29` | 代码证实 |
| **"每 GB 显存约多放 ~700 个专家"** | 代码里**没有**这个常量，它只在文档里。底层常量是每个专家 blob 的字节数 **`BLOB = 1,382,400`**：按 1 GiB 算是 **≈776**，按 1 GB 算是 **≈723**——"~700" 是取整说法 | `include/strata/kernels/cpu/expert.hpp:45` | **数字存在但含义不同** |
| **"MTP 草稿上限 3 个 token"** | 代码里**没有常量 3**。安装器写死的是 `--spec 4`（窗口 = 1 个已接受 token + 3 个草稿），草稿上限由 `set_max_drafts()` 在运行时下发 | `setup.py:4226`、`include/strata/core/mtp.hpp:49-50` | **数字存在但含义不同**（3 = `--spec 4` − 1） |
| **"prompt lookup 草稿上限 5 个 token"** | **代码里找不到 5**。窗口上限是 `kMaxT = 8`，suffix 草稿由 `--spec` 派生（要 5 个草稿需 `--spec 6`）。注意 `generate.cpp:526` 的 `suffix_draft = 3` 是**最小匹配长度**，不是草稿数 | `include/strata/spec/draft_policy.hpp:24`、`src/program/generate.cpp:1777-1779,8011` | **代码内未见** |
| prefill 分块上限 8,192 | `int64_t prefill_auto_max = 8192;`，可用 `--prefill auto:16384` / `auto:32768` 或 `STRATA_PREFILL_AUTO_MAX` 放宽（不超过上下文） | `src/program/generate.cpp:437` | 代码证实 |
| n-gram 表 28.8 GB | 精确到字节：`kTableFileSize = 28,800,138,432`（= 28.8 十进制 GB），注释说明它是 51.2e9 个 IQ4_NL 元素、且是引擎**唯一从 GGUF 而不是 canonical pack 读取**的张量 | `src/kernels/ple_oracle_vectors.inc:79-81`、`include/strata/kernels/ngram.hpp:8-9` | 代码证实 |
| `third_party/ggml` 来自 llama.cpp | 目录里只有 `ggml-common.h` + `LICENSE` + `VERSION.txt`（**不是完整 ggml 源码树**）。LICENSE 是 MIT、"Copyright (c) 2023-2026 The ggml authors"；VERSION.txt 钉在 commit `3cf03257f219afbe7334045ff7c6a06ac68c627d`（2026-09-20）。仓库自己的注释称其为 "llama.cpp's block layouts and codebook grids" | `third_party/ggml/LICENSE`、`third_party/ggml/VERSION.txt`、`CMakeLists.txt:362`、`include/strata/kernels/iq_kernels.hpp:5-6` | 代码证实（措辞上：是 llama.cpp 所用的 **ggml**） |

> 这张表的用处：**引用文档里的"3 个草稿 / 5 个草稿 / 每 GB 700 专家"时要知道它们是描述性说法，不是可依赖的常量**。真正的旋钮是 `--spec`（窗口）与 `BLOB`（专家体积）。
>
> 另注：`sycl/` 下存在大部分源文件的镜像副本，上述数值一致——印证了 Intel Arc 那条路径是同一引擎的并行移植。



`docs/` 下共 **28 个文件**（含 `benchmarks/`、`media/`、`paper/`）。下表按"本调研是否已覆盖"标注，方便后续接手时知道哪些还没读。

| 文档 | 大小 | 内容 | 本调研 |
| --- | ---: | --- | --- |
| `README.md`（仓库根） | 46.8 KB | 主 README：速度表、"需要什么"、按 RAM 选档 | 已覆盖 |
| `docs/DETAILS.md` | 91.8 KB | 全部细节：实测速度表、其它 GPU 估算、各档需求、API、图像、全部设置、故障表 | **已覆盖（全文）** |
| `docs/paper/Strata-Paper.pdf` | 341 KB | 论文，9 页 | **已覆盖（全文解析）** |
| `docs/paper/tiers.svg` | 3.1 KB | 论文的四层存储预算图（与表 1 同源） | 已覆盖 |
| `docs/MODELS.md` | 9.2 KB | 选档页：按 RAM 选档表、各档 "RAM+VRAM Requirements" | 已覆盖 |
| `docs/HOW_IT_WORKS.md` | 5.4 KB | 面向普通读者的三层分工原理页 | 已覆盖 |
| `docs/INSTALL.md` | 20.9 KB | 安装总页：平台、多卡、Docker、老 CPU、全部参数 | 已覆盖 |
| `docs/BATCHING.md` | 15.6 KB | 并发多请求（batch slots）的实测与推荐规则 | 已覆盖 |
| `docs/TROUBLESHOOTING.md` | 4.5 KB | 常见问题速查 | 已覆盖 |
| `docs/COMMUNITY_BENCHMARKS.md` | 8.8 KB | 社区 benchmark 投稿指南 | 已覆盖 |
| `docs/AI_SETUP.md` | 12.8 KB | 写给 AI 助手的分步安装指令 | **未覆盖** |
| `docs/MCP_SERVER.md` | 11.5 KB | `tools/strata_mcp.py` 的 8 个工具定义与安全边界 | **未覆盖（已从清点任务取得要点）** |
| `docs/INTEL.md` | 54.3 KB | Intel Arc 的 SYCL 移植长篇工程记录 | **未覆盖（已取得要点）** |
| `docs/INTEL_ARC.md` | 7.9 KB | Intel Arc 用户向短页（0.1.39 实验性） | **未覆盖（已取得要点）** |
| `docs/AMD_HIP.md` | 37.4 KB | AMD HIP 后端总页 | **未覆盖（已取得要点）** |
| `docs/AMD_HIP_PERFORMANCE.md` | 8.7 KB | gfx1100 后端的性能证据与复现配置 | **未覆盖（已取得要点）** |
| `docs/MULTI_GPU.md` | 11.9 KB | 2-3 张 N 卡 layer split（流水线并行） | **未覆盖（已取得要点）** |
| `docs/SECOND_GPU.md` | 9.5 KB | CUDA1-3 辅助专家缓存与 `--peer-device` | **未覆盖（已取得要点）** |
| `docs/UNSLOTH_Q4.md` | 20.9 KB | Unsloth UD-Q4_K_XL（实验）与 UD-IQ4_XS | **未覆盖（已取得要点）** |
| `docs/ORCA.md` | 5.0 KB | OrcaRouter Uncensored IQ3_XXS 手工兼容流程 | **未覆盖（已取得要点）** |
| `docs/ORCA_Q4_K_S.md` | 4.1 KB | OrcaRouter Uncensored Q4_K_S 手工档 | **未覆盖（已取得要点）** |
| `docs/OLDER_GPUS.md` | 7.7 KB | 老卡支持矩阵（Pascal/Volta、gfx1030/1031/1012/906） | **未覆盖（已取得要点）** |
| `docs/NVIDIA_V100.md` | 4.3 KB | Tesla V100 / Titan V (sm_70) 社区构建 | **未覆盖（已取得要点）** |
| `docs/benchmarks/2026-09-29-gfx1100.json` | 7.5 KB | 上面 AMD 性能文档的原始 JSON | **未覆盖（数据文件）** |
| `docs/media/*`（4 个） | 约 9.6 MB | 两张 SVG 插图 + 生成脚本 + 一张 2560×1440 截图 + 视频预览图 | **未覆盖（媒体）** |

### 关键补充（来自上述文档，前面几节未收录）

**平台缺口（对选型影响最大的部分）**：

| 平台 | 能用的功能 | 缺口 |
| --- | --- | --- |
| **AMD Windows** | 0.1.34 起免编译（内置 ROCm，约 550 MB） | **无图像**（CPU 编码器仅 Linux）、**一卡一模型**（`--gpus` 仅 Linux）、无 `--calibrate`；且现成 zip **尚未在独显上跑过模型**（维护者用核显 gfx1036 测的） |
| **AMD Linux** | 全功能 | 图像只能走 CPU 编码器（`--vision cpu`）；桌面同卡需 `--vram-reserve-mib 3072`（默认 700 会 OOM） |
| **Intel Arc** | 0.1.39 起实验性 SYCL 引擎，**必须 Linux 从源码构建**（release zip 里没有） | 无 Windows 原生路径、**无图像**、WSL2 下 setup 探测不到卡、B580 上见过 device loss；需要 `icpx` 2025.3+ 与 oneMKL（约 5 GB）；不 AOT 则首次 JIT 约 47 s |
| **老卡** | Pascal/Volta 走 CUDA 12 引擎 + opt-in | Volta（V100/Titan V）**CUDA 13 已砍掉**，必须 CUDA 12.x；gfx906（MI50/Radeon VII）**ROCm 已不再提供库**，需社区镜像；P100、sm_60 未测量 |
| **WSL2** | 能跑 | NVIDIA 驱动只 pin 约 1 GB RAM → **KV streaming 关闭**；AMD 探测不到卡 |

**多卡有两种互斥思路，别混用**：

| 模式 | 分工 | 代价 |
| --- | --- | --- |
| **layer split**（`MULTI_GPU.md`） | 层切成连续区间，一卡一段；**每卡只为自己那段保留专家缓存** → 2 卡≈2 倍专家。是**流水线并行不是张量并行**，一 token 每窗口过卡一次 | 不支持 CC<7.5 / Intel / N 卡+A 卡混用；每卡 prompt 缓冲 1.5 GB；WDDM 下 expert arena 只 pin 8 GiB；**每多一卡就多一轮 round**（2080 Ti 作第三卡反而把 5080+3090 拖慢到 68/90 tok/s） |
| **辅助专家缓存**（`SECOND_GPU.md`） | CUDA0 保留 dense/KV/MTP/主缓存，CUDA1-3 各挂一个独立专家缓存；**层内多卡并行，但每层要等全部卡** | 必须按顺序启用 tier；每卡至少留 512 MiB；`--remote-expert-opt` 改变浮点求和顺序，**不声称 bitwise 一致**；只测过双卡 CUDA |
| `--peer-device N` | 第三个选择：在设备 N 放第二个自适应专家缓存 | 与 `--layer-split`、`--expert-cache-device1..3` **互斥**；tier 大小需手调 |

**已知的非确定性（容易被误当成 bug）**：

- **AMD HIP：约每 10 次启动有 1 次**贪心输出会在某个 token 起与另一次不同（单卡双卡、0.1.29 都有），**文档未解释原因**（`docs/AMD_HIP.md:283-284`）。
- 多个 opt-in 开关会改变浮点求和顺序或专家归属，因而**输出可能与默认不同**（虽然稳定连贯）：`--remote-expert-opt`、`STRATA_STAGE_TRIM=1`、`STRATA_SPLIT_OWN=1`、`STRATA_PREFILL_HELP=1`（`docs/MULTI_GPU.md:75-103`、`docs/SECOND_GPU.md:117-123`）。
- Intel SYCL 路径上也记录了 chunk 边界导致的 near-tie 翻转（`docs/INTEL.md:240,421-426`）。

**三个"特殊档位"是什么**：

| 档位 | 实际构成 | 注意 |
| --- | --- | --- |
| Unsloth **UD-Q4_K_XL** | 不是全 Q4_K：专家 gate/up = Q4_K（第 2 层 Q5_K）、down = Q5_1；embedding/head/attention = Q8_0；PLE 表 = IQ4_NL。111.3 GB | **实验性、仅 NVIDIA、无图像**；需 ≥48 GB RAM；**不支持 layer split + RAM 预算并用** |
| Unsloth **UD-IQ4_XS** | gate/up = IQ3_S、down = IQ4_NL，59.5 GB 专家、93.7 GB | 0.1.39 起进正式菜单；**尚未在 NVIDIA 上测过**（只在 Strix Halo gfx1151 验证） |
| OrcaRouter Uncensored **IQ3_XXS** / **Q4_K_S** | 手工流程，需 `tools/iq_pack.py --compat-bf16` 或 `-DSTRATA_ORCA_Q4KS_MMQ=ON`；专家 arena 约需 **49.8 GiB** 可用内存 | 都不在安装器菜单；Orca 的 **IQ3_M 不受支持**；Q4_K_S 若含 Q5_0 down 层**引擎会拒绝启动并指名该层** |

> 三个文档都强调：`--compat-bf16` 只是把精度补到 BF16，**不能恢复原始 BF16 checkpoint 的质量**。

**MCP server 提供的 8 个工具**（`docs/MCP_SERVER.md:112-119`，方向是"让 AI 助手管理 Strata"，与"模型调用外部工具"相反）：

`strata_status`（状态/硬件/推荐）、`strata_models`（可选 family 与档位）、`strata_install`（无提问安装，需 `confirm: true`）、`strata_start`、`strata_stop`（有请求在跑需 `force: true`）、`strata_logs`、`strata_benchmark`、`strata_connect_info`（给出 Claude Code / Codex / Cursor 等接入配置）。

安全边界明确：只连 `127.0.0.1`，**不会**用 `--host 0.0.0.0` 或 API key 安装，不启用实验选项；`data_dir` 有白名单（拒绝 `..`、相对路径与 UNC）。


## 未核实 / 存疑

| 项 | 状态 | 原因 |
| --- | --- | --- |
| GitHub star / fork / issue 数 | 未核实 | 本环境 `web_fetch` 被沙箱拦截（见下），无法打开 GitHub 页面 |
| 仓库简介里 "8GB+ NVIDIA GPU" 那句原文 | 未核实 | 本调研第一版曾以此对比 README 的 "12 GB"；但**本地仓库里找不到这句话**（GitHub 页面描述不在 clone 内），也无法联网核实。已改以 `README.md:58` 与 `setup.py:3797` 为准 |
| 论坛实测数据的原始上下文 | 未核实 | 只有搜索结果标题和摘要，未读到原帖全文 |
| 与 ktransformers 的**同机对比** | 未核实 | 论文只把它列为相关工作（参考文献 14），**没有做过同机对测**；我方也未 clone 对方实现 |
| Splash / ninfer / HyperQwen 具体借鉴了哪部分 | **部分核实** | 论文第 9 节确认"一个引擎只服务一个模型"的想法来自这三者，但未指明代码级借鉴细节 |
| 论文的论证与实验 | **已核实（二次复查）** | 9 页经 `mcp__pdf__read` 读为干净 Markdown；表 2/3/4/5 逐格、13 条发现逐条、第 7/9 节逐项，共 **47 项引用全部对拍通过**，未发现错误 |
| `docs/DETAILS.md` 的全部实测数字 | **已核实** | 已全文精读 1,127 行，环境变量、API 字段、配置键、默认值均已提取 |
| `setup.py` 的判卡与选档逻辑 | **已核实** | 已走读 4,422 行（grep 定位 + 18 个区间精读），结论见「硬件 / 软件门槛」 |
| `docs/` 全部文档的覆盖度清单 | **已核实** | 28 个文件已清点并逐个给出内容摘要，见「`docs/` 全部文档清单」；其中 INTEL / AMD_HIP / MULTI_GPU / SECOND_GPU / UNSLOTH_Q4 / ORCA / MCP_SERVER / 老卡等已提取要点 |
| 引擎内部实现是否与文档一致 | **部分核实** | 已做**定点常量级走读**（专家数/层数/`BLOB`/`--spec`/prefill/n-gram 表/ggml 来源），见「源码验证」；其中 3 项与文档表述方式不符。但**未做**全文走读（`src/` 68,796 行）。论文与代码的**文字层面**一致性已由 47 项引用复查覆盖，机制层面仍未逐条核对 |
| 温度采样 / 拒绝采样现状 | 未核实 | 论文说 greedy-only 是当时的局限；代码是否已支持未核 |
| Intel Arc / ROCm 的**实际可用性** | 未核实 | 已读文档要点，但**未做实测**；且 Intel 路径仍标为实验性，AMD Windows 的现成引擎据其文档尚未在独显上跑过模型 |
| AMD HIP 的"每 10 次启动约 1 次输出不同" | 未核实 | 上游文档自己记录但**未解释原因**（`docs/AMD_HIP.md:283-284`）；这不是本调研的发现，而是转述 |

## 调研方式

- **手段**：`git clone` 拿到完整仓库（845 commits，v0.1.39）后**直接读源文件**——`README.md`、`docs/DETAILS.md`（全文 1,127 行）、`docs/MODELS.md`、`docs/HOW_IT_WORKS.md`、`docs/INSTALL.md`、`docs/BATCHING.md`、`docs/TROUBLESHOOTING.md`、`docs/COMMUNITY_BENCHMARKS.md`、`setup.py`（4,422 行，判卡/选档/下载/校验逻辑）、`LICENSE`，外加 `git log` / `git shortlog` / `git ls-files` 统计。
- **论文解析**：先用 DSH 新增的 **`pdf` MCP server（`@sylphx/anymd`，8.5.1）** 把 9 页全文读成 Markdown，逐表核对。该 server 由 `cordis.patch.yml` 的 `pdf-mcp` 条目接入，工具为 `mcp__pdf__read` / `search` / `outline` / `inspect`。
  - **此前它是纯手工解析的**，因为环境里没有任何 PDF 库。当时的自建提取器（`docs/_pdf_extract.py`）确认了三件事：该 PDF 的正文是 **Identity-H 子集字体**（每字形 2 字节）、每个字体带**真实 ToUnicode CMap**（必须用它；码值恰好等于 ASCII 只是巧合）、且**每个字形单独定位**（空格要从 `Td` 偏移里还原）。
  - 手工提取器的输出有**系统性失真**（字符错位、词内多余空格如 `Fl ash-Next`、表格塌成逐字）。用它核对数字，本质上是在核对"我自己的修正是否正确"。**换用 anymd 后，本轮又在未经加工的原文上把 47 项论文引用逐条复查了一遍，未发现错误**——所以这份"已核实"是后验确认过的，不是当初的自证。
  - 顺带记录：该 PDF 的**标题行本身带散落空格**（`Fl ash-Next`、`S trata`、`Wher e the time goes`），这是文件文字层自带的，不是提取器缺陷；正文段落干净。
- **当时的限制**：本机 DNS 被网络层劫持为 fake-ip（`198.18.0.0/15`），DSH 的 `web_fetch` 因 SSRF 防护拒绝非公网 IP；同时沙箱只放行 git 出网，`curl` / `.NET TLS` 全部失败（连 `www.baidu.com` 都不通）。**所以网页调研这条路走不通，全部结论基于 git 通道取得的真实源码与仓库内文档**，这也让可信度高于纯搜索。
- **本次环境故障**：工作区 `D:\github\research` 上缺少"取得所有权"权限，DSH 无法授予写入权限（`SetNamedSecurityInfoW failed (Win32 5)`），已按内置诊断流程修复并验证；回滚脚本在 `D:\github\_dsh-acl-recovery\`。
  - 后续发现 `docs\` 子目录仍有问题：文件**可创建、可读取、可修改，但删除被拒**（PowerShell 与 Python 都报 access denied）。目录与文件上其实都存在含 `Delete` 的权限项，但持久化的**不可继承**完全控制条目未能下传到子文件；诊断脚本对该文件判定"无需更改"后停止。
  - **收尾时该问题已被绕过**：`docs\` 下三个辅助脚本改用一次性提权删除成功，`docs\` 现为空目录（Git 不跟踪空目录，不影响仓库）。所以这条平台限制的最终状态是"**默认权限下删除被拒，需提权才能删**"。
  - 另注：沙箱内 **PowerShell 向 `docs\` 写入被拒**（Python 写入正常），故辅助脚本放在工作区根目录。
- **复现方式**：
  ```bash
  git clone https://github.com/Niko1221/Strata.git
  cd Strata && git checkout v0.1.39
  ```
  本地已存在副本：`D:\github\Strata`
- **后续可做**：
  1. **系统走读 `src/`**（68,796 行），核对论文与实现是否一致——这是剩下最大的空白（已做的只有常量级走读；论文的 47 项引用复查属文字层面，不是机制层面）。
  2. clone **ktransformers** 做同机/同口径对比（论文只把它列为相关工作）。
  3. 精读尚未展开的 **`docs/INTEL*.md`、`docs/AMD_HIP*.md`、`docs/MULTI_GPU.md`、`docs/UNSLOTH_Q4.md`、`docs/ORCA*.md`** 全文（本轮只提取了要点，清单见「`docs/` 全部文档清单」）。
  4. 核实温度采样/拒绝采样的现状（论文的局限之一）。
  5. 按本机实际硬件做适配性判断（本机 DNS 与沙箱限制不影响对已 clone 仓库的分析）。
