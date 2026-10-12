# REA（Reverse Engineer Anything）

- **调研日期**：2026-10-12
- **上游地址**：https://github.com/morluto/rea
- **调研版本 / commit**：`main` HEAD = `83a4445f440fab978ad9376b4260acae3f5fb189`（2026-10-12）；npm 包 `rea-agents@6.4.0`（2026-10-11）。本文 `文件:行号` 均以该 HEAD 为基准。
- **本地副本**：`D:\github\research\.cache\rea`（2936 个追踪文件；git pack 50.45 MiB）
- **许可证**：MIT（`Copyright (c) 2026 morluto`）。注意仓库含 **5 个 git submodule**（`jadx-headless-mcp`、`binwalk`、`unblob`、`ghidra-nativeaot`、`wakaru`），它们各有自己的许可。
- **一点历史**：首次提交是 2026-04-14 的 `Initial commit: betterBinaryMCP - Enhanced MCP server for Hopper Disassembler`——也就是说它**起源于一个 Hopper 反汇编器的 MCP 服务器**，7 月 12 日 `Remove legacy Python MCP implementation` 之后重写成了现在的 TypeScript 项目。

## 一句话结论

REA 是**把 AI agent 接到逆向工程工具链上的一套本地工具**：一个 MCP server + CLI + agent skill，能分析原生二进制（驱动 Hopper / Ghidra / IDA）、JS/Electron 应用、.NET 程序集、Android APK/设备、固件、网站、抓包、进程行为等十几种目标。核心卖点不是"agent 会读反汇编"，而是**把分析做成带证据与边界的工具调用**：每条结论要能追溯到来源，同时明确标出"未知"；并且有相当严格的权限模型（IDA 只读、Ghidra 用临时数据库不改文件、安装任何东西都要用户先批准）。

它是 2026 年 10 月的现象级项目：**仓库自己的提交记录显示 star 从 2 万涨到 11 万只用了 4 天**（10-08 → 10-11）。这个数字我只能引用它自己的记录，**无法独立核实**。

## 项目概况

| 项 | 值 | 来源 |
| --- | --- | --- |
| 仓库地址 | https://github.com/morluto/rea | 已验证（git remote） |
| 许可证 | MIT；另有 5 个 submodule 与 `third_party/`、`licenses/` 目录 | 已验证（`LICENSE`、`.gitmodules`） |
| 定位 | 本地逆向工程工具：MCP server（给 agent 用）+ CLI（给人用）+ agent skill；分析在本地进行 | 已验证（`README.md:52`、仓库 `AGENTS.md`） |
| 技术栈 | TypeScript / Node（`npm@11.16.0`、turbo、vitest、oxlint/oxfmt）；`src` 分 domain / provider adapter / application / CLI-MCP 适配器四层 | 已验证（`AGENTS.md`、`package.json`） |
| 提交数 / 贡献者 | 2462 commits / 123 人；Top1 `morluto` 1277 次（**52%**）、Top2 `Kaoru0822` 351、Top3 `N0zoM1z0` 270、Top4 `Ysopwnable` 169 | 已验证（`git shortlog -sn HEAD`） |
| 最新版本 | npm `rea-agents@6.4.0`（tag 指向 release 提交 `a56f7bb`，非本文走读的 HEAD）；tag 共 35 个，命名形如 `rea-agents-6.4.0`（release-please 管理，`always-bump-minor`、tag 不带 `v`） | 已验证 |
| 首次提交 | 2026-04-14（betterBinaryMCP）；**5–6 月无提交**，7-12 起以 Node 重写 | 已验证（`git log --reverse`） |
| 仓库体积 | git pack 50.45 MiB；工作树主要分布：`src` 10.3 MB、`website` 10 MB、`tests` 5.1 MB、`docs` 2.4 MB | 已验证 |
| 代码规模 | `src` **1565 个 .ts / 321,374 行**（含 357 个 `*.test.ts`）；`tests` 目录 812 个文件 / 154,747 行（其中 `.ts` 634 个 / 144,696 行）；`docs` 57 个文件 / 31,778 行；`scripts` 288 个文件 / 55,661 行；`bridge` 30 文件 / 8,998 行（含 Ghidra Java 侧 9 文件 / 4,541 行）；`native` 5 文件 / 1,482 行（Windows N-API C++ addon）；`website` 84 文件 / 265,788 行（其中 18 个 `.po` 翻译占 23.7 万行） | 已验证（.NET `ReadAllLines` 统计） |
| npm 依赖 | 36 个直接依赖 + 19 个开发依赖 | 已验证（`package.json`） |
| 支持的 agent 客户端 | 源码里有 **19 个**：Claude Code、Claude Desktop、Codex、Cursor、Gemini CLI、Windsurf、Devin、opencode、Antigravity、Copilot CLI、CommandCode、Qwen Code、VS Code、Grok Build、omp、pi、Hermes、**DeepSeek Harness（dsh）** 等（`src/application/SupportedClients.ts:354-520`；`grok_bot` 标记为 unsupported） | 已验证 |
| CI | 19 个 workflow / 合计 **40 个 job**，其中一大批是**真实工具集成测试**：`real-hopper.yml`、`real-hopper-linux.yml`、`real-ghidra-windows.yml`、`real-browser.yml`、`real-web-runtime.yml`、`real-javascript-recovery.yml`、`real-binary-layout.yml`、`real-evm-interface.yml`、`real-go-binary.yml`、`real-recorded-crash.yml`、`real-termux.yml`、`real-web-network-captures.yml`、`real-web-source-map.yml`、`nativeaot-fixtures.yml` | 已验证（`.github/workflows/`） |
| 治理 | 外部 PR **确实接受**（CHANGELOG 里有 25 个不同的贡献者 handle，另有 `docs/releases/6.4.0.contributions.json` 记录贡献）；但 `SECURITY.md` **没有响应时限承诺**（只承诺确认与协调披露），且**没有 CODEOWNERS / FUNDING / dependabot** 配置 | 已验证 |
| 文档与站点 | `docs/` 49 篇（安装、CLI/证据、MCP contracts、工具目录、测试、路线图等）；`website/` 是官网与 showcase 来源 | 已验证 |
| 热度（star） | README 自称 **110,000**；仓库提交里留下了里程碑痕迹：10-08 记 2 万、10-09 记 3 万与 4 万、10-10 记 5 万与 6 万、10-11 记 8 万与 9 万与 11 万 | **未独立核实**（见下） |

**提交分布很不均匀**（按作者日期）：2026-04 仅 1 个、7 月 697、8 月 21、9 月 100、**10 月（12 天内）1643 个**。它不是"慢慢长大的开源项目"，而是**7 月重写、10 月爆发**的两段式曲线。作者/提交者时间差最大只有 1.2 天，说明历史没有明显被改写。

## 它解决什么问题

**痛点**：逆向工程本身是"证据密集"的活——反汇编、字符串、符号、调用关系、IPC、抓包散落在不同工具里；而直接让通用 agent 去啃二进制，它既没有得力的工具，也容易**把猜测说得像结论**。

**REA 的做法**（`README.md:50-56,135-175`）：把分析能力做成 **MCP 工具**交给 agent，工具返回的每份结果都带**证据与限制**；agent 用这些证据继续追问、解释行为、写实现。CLI 走同一套 workflow，所以人和 agent 得到的是同样的东西。分析在本地进行，不上传目标（FAQ 专门强调："REA analyzes targets locally. Your agent receives the tool results, and its model provider has its own data policy."——**注意这句话的后半段：数据最终仍会进入你所用的模型供应商**）。

**它明确区分的两类行为**（`README.md:173-175`）：静态检查（JS/.NET 只读文件，不运行程序）vs **运行时捕获**（会以你的用户权限运行或交互目标）。后者在每个 guide 里单独描述其副作用。

**仓库自己的工程信条**（`AGENTS.md`，值得单独引用）：区分 observations / derivations / inferences / unknowns；配置改动必须**只增不减、幂等、带备份**；安装器**不得安装或升级无关软件**（包括 Homebrew、Node、npm、Java、Ghidra）；`rea setup` 必须先展示将要做的改动并取得批准；凭据必须脱敏，但**不得凭字段名或文本模式推断敏感性**。

## 核心机制

### 能分析什么（以及每种目标的真实依赖）

| 目标 | 返回什么 | 依赖 / 平台限制 |
| --- | --- | --- |
| 原生二进制 | 伪代码、汇编、字符串、符号、调用与引用 | **Hopper / Ghidra / IDA 之一** |
| 离线 ELF 布局 | 段、节、原始符号/重定位、静态缓解候选 | 需调用方自带 pwntools，**Linux x64** |
| Go 二进制 | 编译器版本、模块、replace、构建设置 | 内置静态检查，**不需要装 Go** |
| WASM 产物 | 校验、节、精确 import/export WAT 形式与解码 | 需调用方自带 WABT 1.0.42 |
| EVM 字节码 | 分发选择器、字节偏移、推断参数与可变性 | 本地 raw/hex，**Linux x64** |
| 已录制的 Linux 崩溃 | 各线程寄存器/信号、可选映射候选 | 需 pwntools，可选 GDB/pwndbg，Linux x64 |
| JavaScript / Electron | 模块、import、source map、路由、IPC、原生扩展关系 | 只要 Node + npm（**无需原生分析引擎**） |
| 网站 | 页面结构、脚本、网络观察、截屏 | Chrome 系浏览器 |
| 已保存的网络抓包 | 请求/响应、暴露的载荷与来源位置 | HAR；原生 mitmproxy 捕获需 Linux 上的 mitmdump |
| .NET 程序集 | 元数据、CIL 指令、声明的原生依赖、构建比较 | 无额外依赖（**纯静态**） |
| Android APK | Manifest 声明、类、反编译方法、引用 | headless JADX + 完整 JDK（Linux/macOS/Windows x64） |
| Android 设备 | 设备、包、进程、日志、dump、截屏、文件传输 | 调用方自带 adb |
| JEB 项目 | 引擎支撑的项目单元、产物摘要、类型/方法伪代码 | 需调用方先启动 JEB 并暴露 MCP（`REA_JEB_MCP_URL`） |
| Android 资源 | 解码后的 manifest、字符串表、语言与版本事实 | 调用方自带 Apktool |
| 固件 | 区域、提取结果、交给原生分析的交接 | Binwalk / Unblob，**Linux** |
| 包与资源 | 文件清单、摘要、plist、Apple bundle 解剖、抽取资源 | — |
| 进程行为 | 终端输出、交互、退出与文件系统观察、运行比较 | **Linux/macOS**，需要原生 PTY |

（依据：`README.md:150-175` 的表格。表中的"依赖"是文档声称，本次未逐项运行验证。）

### 分层与依赖方向

仓库 `AGENTS.md` 明确规定依赖只能**向内**流动：**domain 语义与契约 → provider 适配器 → 共享 application workflow → CLI/MCP 适配器**；引擎协议与 provider 专有代码不得渗进 domain/application 层。这条规则在 `src/` 的体量分布上看得出来（`src` 1565 个 `.ts` / 321,374 行，其中生产 1208 个 / 235,847 行、测试 357 个 / 85,527 行）：

| 层 | 规模 |
| --- | --- |
| `domain`（纯语义：evidence、comparisons、application graphs） | 353 文件 / 76,531 行 |
| `application`（CLI + MCP 共享工作流） | 222 文件 / 41,609 行 |
| `contracts`（工具契约） | 63 文件 / 8,910 行 |
| 适配层 | browser 87/17,637、artifacts 45/11,404、native 43/10,235、ghidra 40/9,289、dotnet 22/7,242、hopper 21/4,926、ida 12/2,289、jeb 3/712 |
| 端点层 | server(MCP) 75/7,018、cli 40/5,431、main 5/315 |
| 支撑 | process 36/8,254、composition 24/554、config 10/471 |

**domain 层是最大的一层**（7.6 万行，接近四分之一）——这与"把语义和契约放在最里面、不让 GUI/引擎细节渗进来"的设计是一致的：它把大量代码花在**建模和校验**上，而不是花在调用工具上。

### MCP 工具面

工具不是手写清单，而是**契约驱动**的：`src/contracts/toolContracts.ts:28` 聚合 **25 组契约**，Zod 契约经 `src/server/toolRegistrationOptions.ts:8` 直接投影成 SDK 注册（没有"先生成代码再注册"这一步），转换后再用 Ajv2020 反校验 schema；绑定按**命名契约**而不是 catalog 顺序（`src/server/contractToolHandler.ts:46`），CLI 与 MCP 共用同一个 `createDirectAnalysis` 与 `analysis.execute(contract.name, …)`（`src/cli.ts:56,75`）。

**工具数 = 139**（按源码逐文件统计，并与测试期望 `tests/boundary/filesystem/generatedSkill.test.ts:60` 互证）。注意一个**文档滞后**：`docs/installation.md:197` 写的是 "172 tools"，那对应的是更早的 commit `04089d5be`，与当前 HEAD 不符。生成的 catalog（`docs/public/product-catalog.json`）**不签入仓库**——`docs/mcp-contracts.md:5-11` 说明它由 `npm run docs:generate` 产出，运行中 server 的 `binary_session` 才是工具可用性的权威来源。也就是说**工具集随环境变化**（取决于装了哪个 provider、目标是什么），别指望一张静态工具表。

### 权限与安全模型（这是它最像"工程"的部分）

`AGENTS.md` 把"权威"（authority）写得非常具体，而且**源码里确实照着实现了**：

- **Ghidra**：headless 启动参数直接写死 **`-readOnly` + `-deleteProject`**（`src/ghidra/GhidraLauncher.ts:551-552`）——临时库"关闭即删"、**永不写原始字节**；写权限只存在于 ephemeral 会话库内（`GhidraProviderCapabilities.ts:30`）。临时库目录要求不含点号前缀（Ghidra 的命名规则会拒绝）；**Windows 上走认证过的 IPv4 回环**，因为 Windows 的 Node 没有 Java 所需的 AF_UNIX（`:37`）；Windows Ghidra P0 只读（`docs/windows-ghidra-p0.md:9`）。
- **IDA**：接上游 `ida-pro-mcp`，有 attached（旧式 1.4 工具）与 headless（数据库 supervisor）两种模式（`src/ida/IdaConfiguration.ts:8`）。只读是硬约束：**不在 `IDA_OPERATIONS` 白名单里的操作一律返回 `capability_unavailable`**（`IdaSessionClient.ts:148`）；关闭库时显式传 **`save:false`**（`:426`），并且必须回执 owned + worker + 未保存（`:436`）。attached 模式下 GUI 导航仍有副作用（`IdaProviderCapabilities.ts:45`）——所以 `AGENTS.md` 才写"绝不保存或关闭已连接的 GUI 数据库"。
- **Hopper**：由 REA **拉起 GUI 进程**并用 Unix socket 通信（启动脚本写入 `REA_SOCKET` / `REA_TOKEN` / `REA_RUN_ID`，`src/hopper/BridgeLauncher.ts:379`；协议在 `src/hopper/protocol.ts:27`）——也就是说**用 Hopper 是有副作用的**，不像 Ghidra 那样完全后台。
- **JEB**：**不启动引擎**，只连调用方已经启动的 JEB MCP，且**只允许字面 loopback 地址**（`src/jeb/JebMcpEndpoint.ts:35`）。
- **安装**：`rea setup` 必须先展示计划改动并**取得批准**才写文件或安装 Hopper；安装器不得升级无关软件；Ghidra 与 IDA 一律自带。
- **进程所有权是有 token 的**：`OwnedProcessGroup{runId, leaderPid, processGroupId, expectedCommand, expectedParentPid}`（`src/process/ProcessOwnership.ts:16`），spawn 时注入 **`REA_PROCESS_RUN_ID`**（`src/process/ProviderProcess.ts:170`）再读回校验；身份不符即拒绝（`ProcessOwnershipIdentity.ts:8`）；**读不到 token 记为 `unverified` 而不是当成外来进程**（`ProcessOwnership.ts:134`）。POSIX 用进程组、Windows 用 Job Object（`ProviderProcess.ts:172,201`）。清理失败的资源**保留所有权**而不是丢弃。
- **数据**：保留调用方指定的输入与本地证据；脱敏传输凭据与被显式标记的敏感值；不得因为子进程继承了环境就把环境持久化。产物落在 `mkdtemp` + `chmod 0700` 的私有运行时根里，`close` 时幂等删除（`src/process/PrivateRuntimeRoot.ts:63,93`）。
- **本地性可验证，但"local-only"是声明而不是机制**：出站网络**只有 loopback**（CDP、V8 Inspector、JEB、Windows Ghidra 回环），**唯一非 loopback 的是安装器下载 Hopper 安装包**（`src/application/HopperDownload.ts:10`）；dotnet、artifacts、ghidra、hopper 的 provider 元数据里都显式标注 `mayAccessNetwork:false`。把所有源码翻过一遍后，实际会出站的只有三条路径：`rea update` 走 npm/bun 注册表（`src/application/UpdateRuntime.ts:198,207,223`）、经批准的 Hopper 安装走 `www.hopperapp.com`、以及 `install.sh` 查一次 `api.github.com` 的 latest release。**没有遥测**：`package.json` 里没有任何遥测类依赖（我在 `package-lock.json` 里也查不到 posthog / @sentry / analytics / amplitude 这类包），`src` 里搜到的 "sentry" 全是 `AdbProcessEntry`、`carriesEntry` 这类含 `Entry` 的标识符被子串匹配出来的误命中；同样没有启动期版本检查或自动更新。

### 证据模型

`AGENTS.md` 要求跨 CLI 与 MCP 保留 Evidence provenance 与 session 语义；源码里的落地是：

- **置信度只有三档**：`confidence: observed | derived | inferred`（`src/domain/evidence.ts:121`），另有一组 authority 取值（`:95`）——注意这是**三档而不是四类**，`AGENTS.md` 里的 "unknowns" 在代码中是**独立模型**：`ResidualUnknown` 带状态机（open / investigating / blocked / contradicted / resolved）与**修订链**（`src/domain/residualUnknown.ts:44,47,57`）。把"未知"当成一等对象而不是缺失值，是这套设计里最关键的一点。
- **provenance 的承载字段**：`subject`（sha256 + 格式 + 架构 + 本地路径，`:41`）、`provider`（`:36`）、`analysis_profile`（`:115`）、`locations`（地址 / 产物路径 / 文件偏移，`:79`）、`evidence_links`（`:127`）、以及带语义摘要校验的 `evidence_id`（`:158`）。
- **会话层**：`EvidenceLedger` 幂等且**拒绝冲突**（`src/application/investigation/EvidenceLedger.ts:48`），证据 bundle 可导出/导入（`src/server/registerSessionTools.ts:86,91`）。
- **错误处理**：`Result`（`src/domain/result.ts:2`）+ **28 个稳定错误标签**（`src/domain/analysisErrorBase.ts:2`），错误以 `{error:…}` JSON 文本 + `isError` 返回（`docs/mcp-contracts.md:175`）。

这套约束的目的是：**让 agent 无法把"我猜的"混进"工具告诉我"**。

## 实测与标称数据

**本次零运行时实测**：没有安装、没有运行任何分析（REA 需要 Node 22.19+ 与真实目标），所以下面只有我实测的仓库规模数据，与仓库自述。

我实测的规模数据：

| 项 | 数值 |
| --- | --- |
| 追踪文件 / git pack | 2936 个 / 50.45 MiB |
| `src` | 1565 个 `.ts` / **321,374 行** |
| `tests` | 634 个 `.ts` / **144,696 行**（测试与源码比约 0.45:1） |
| `docs` | 49 个 `.md` / 11,322 行 |
| `bridge` / `native` | 8,998 行 / 1,482 行 |
| `CHANGELOG.md` | 2467 行（release-please 生成） |

**star 时间线（仓库内自证部分）**——这些数字来自它自己的提交、CHANGELOG 与 README，**不是我从 GitHub 读到的**：

| 日期 | 里程碑 | 证据 |
| --- | --- | --- |
| 2026-10-08 | 20,000 | `docs: celebrate 20,000 GitHub stars 🎉 (#1079)`（`CHANGELOG.md:657`） |
| 2026-10-09 | 30,000 / 40,000 | `CHANGELOG.md:433-434,195-196` |
| 2026-10-10 | 50,000 / 60,000 | `docs/releases/6.4.0.contributions.json:290`、提交 `2ec93ba16` |
| 2026-10-11 | 80,000 / 90,000 / **110,000** | 提交 `6ff4d4a05`、`587677901`、`0d78c99c1`；`README.md:320` |

也就是说：**从 2 万到 11 万，只用了 4 天**。同期 10 月的前 12 天产生了 1643 次提交。这个增速本身是这条调研里最值得记的事实——但**它同时意味着"成熟度"和"热度"是两回事**。（一个精确性说明：`rea-agents-6.4.0` 这个 tag 指向的是 release 提交 `a56f7bb`，不是本文走读的 HEAD。）

**showcase 的可自证程度**：仓库内的 `website/evidence/` 有 **8 份案例证据文件**，其中 DX-Ball 那份记录了目标 SHA-256、涉及的 63 字节函数地址范围（`0x406400-0x40643e`）与所用工具版本（REA 4.1.0 / Ghidra 12.1.4）——这部分是**可在仓库内核对**的。但宣传语的"通过 3,205 个原始 x86 用例"的外链结论、以及重建仓库本身（`github.com/N0zoM1z0/*`）都在仓库之外，`src`/`tests` 里没有对应 fixture，所以**结论部分仍属声称**。

CI 侧的可信度信号：19 个 workflow 里有 13 个是 `real-*`（真实 Hopper / Ghidra-on-Windows / 浏览器 / EVM / Go 二进制 / 录制崩溃 / Termux 等），说明它**尽量用真实工具与真实产物做验证**——这与仓库 `AGENTS.md` 里"simulated providers do not establish real-provider behavior"（模拟 provider 不能证明真实 provider 行为）的要求一致。

## 硬件 / 软件门槛

| 项 | 门槛 | 来源 |
| --- | --- | --- |
| Node.js | `^22.19.0 \|\| ^24.11.0 \|\| >=26.0.0`，外加 npm | `package.json` engines |
| 深原生分析 | **需要 Hopper / Ghidra / IDA 之一**（Hopper 与 IDA 是商业软件；Ghidra 免费） | `README.md:236-242` |
| 无原生引擎也能用 | 静态 JavaScript/Electron 与 .NET 检查不需要 | `README.md:54` |
| 各目标的额外工具 | pwntools（ELF / 崩溃）、WABT 1.0.42（WASM）、JADX + JDK（APK）、Apktool（资源）、adb（设备）、Binwalk/Unblob（固件）、mitmdump（原生抓包）、Chrome（网站）、JEB（JEB 项目） | `README.md:153-171` |
| 平台 | 多项能力**限 Linux x64**（ELF 布局、EVM、录制崩溃）；**Windows Ghidra 是实验性的 P0**（无修改权限）；进程捕获需 Linux/macOS 原生 PTY | `README.md:156-171`、`docs/windows-ghidra-p0.md` |
| LLM | 不自带 key——它把结果交给**宿主 agent 的模型**；模型供应商的数据政策由用户自己承担 | `README.md:273-279` |
| 磁盘/算力 | 仓库 50 MiB；分析成本主要在目标本身（大二进制要调高 provider 启动超时，如 `REA_GHIDRA_STARTUP_TIMEOUT_MS`） | `README.md:184-185` |

## 上手方式

```bash
# 1) 一键接入 agent（会展示改动并请求批准，并备份既有配置）
npx rea-agents setup

# 2) 纯 CLI 用法：不装原生引擎也能跑的 JS/Electron 分析
npx -y rea-agents@latest analyze-javascript-application /absolute/path/to/app --json

# 3) 安装全局命令
npm install --global rea-agents
rea --help

# 4) 更新（它迭代很快，官方建议保持最新）
rea update
```

给 agent 的典型提问（`README.md:79-84`）：

> Understand how search works in the Notes app, show me the evidence, and build a similar feature for my project.

**安装与 setup 的实际行为**（值得先知道，因为它会改你的 agent 配置）：

- `install.sh` **只装全局 npm 包、不碰任何 agent 配置**；仅支持 Darwin/Linux，会检查 `curl` 与 Node/npm，查一次 GitHub latest release，Linux 上加 `--prefix $HOME/.local`，支持 `--dry-run` 只打印计划，并且只有在有 TTY 时才会**询问后**启动 `rea setup`（`install.sh:71-135`）。
- `npx rea-agents setup` 先让你多选客户端并**展示将要做的改动，必须批准**才会写文件或安装 Hopper（`docs/installation.md:377-388,410-411`，实现在 `src/application/Setup.ts:250-271`）；写配置时**保留无关条目与注释**、原子替换、写后校验，并创建 `.rea.backup`（且不覆盖已有快照，`:424-431`）。`--no-prompt` / 非交互模式**不授予安装外部依赖的权限**（`:81`）。
- 支持的客户端源码里有 **19 个**（`src/application/SupportedClients.ts:354-520`）：Claude Code、Claude Desktop、Codex、Cursor、Gemini CLI、Windsurf、Devin、opencode、Antigravity、Copilot CLI、CommandCode、Qwen Code、VS Code、Grok Build、omp、pi、Hermes、**DeepSeek Harness（dsh）** 等；`grok_bot` 被标为 unsupported。**DSH 集成不是顺手加的**：它有专门的实现与卸载处理（`src/application/DshCordisPatch.ts`、`src/application/Uninstall.ts:312,375` 会识别 dsh dialect 与 patch 是否仍在），安装方式是 **Cordis YAML patch**（`docs/installation.md:308-317`）——也就是说配置是**宿主进程加载的内容**，装之前值得看一眼 patch。
- 手动注册 MCP 的形状（`docs/installation.md:1018-1027`）：`{"mcpServers":{"rea":{"command":"npx","args":["-y","rea-agents@6.4.0","mcp"]}}}`。
- **三个名字别搞混**：npm 包名是 `rea-agents`（`package.json:2`），可执行文件同时提供 `rea` 与 `rea-agents`（都指向 `scripts/rea.mjs`），`io.github.morluto/rea` 是 **MCP Registry 的标识**而不是 npm scope。我检索时看到的 `@morluto/rea` 在仓库里**查无此名**，作为 npm 包属**未核实**。

官方 showcase 给了三个案例：DX-Ball 音效声像算法重建（`README.md:194-201`，称重建通过 3205 个原始 x86 用例、并复现全部 63 字节函数）、Notion 的 Electron 剪贴板桥、TH04 的 DOS 弹幕角度计算。

## 坑与风险

1. **双用途与法律风险是真实的，而且合规表述只有一句**：`README.md:332-334`（19 个语言版本都有）写道"REA 提供用于**合法**逆向研究、分析与重建的工具；**获取授权与合规的责任在你**"，该声明由 PR #888 引入（`CHANGELOG.md:1024`）。但全仓检索 `DRM|circumvent|crack|piracy|EULA|terms of service` **只命中一张 PNG**——也就是说它对"绕 DRM/破解/违反厂商 EULA/授权范围"**没有任何政策表述**。它做对的部分是 BYO 合规：文档明确说明 **REA 不读取、不安装、不自动化任何 license 凭据**（`docs/installation.md:696-698`），也不安装/激活/授权 IDA（`docs/ida-provider.md:3`）。**边界判断完全在使用者。**
2. **它不是沙箱——这一点仓库自己说得很清楚**：`SECURITY.md` 明确写 "This is not a sandbox… does not protect against malicious processes"，且与被分析对象**同属一个 OS 用户**；进程捕获以你的用户权限运行、非沙箱、观测路径不限制读写（`docs/process-capture.md:9-13`）；Web 运行时观测会改变 timing/JIT（`docs/web-runtime.md:14`）；首次 ADB 查询可能启动本机 adb server（`docs/adb-device-analysis.md:10-11`）。更值得注意的量化事实：`src/contracts/toolEffects.ts` 逐工具标注副作用，其中 **5 个工具标记 `executesForeignCode: true`**。它的安全模型是**审批 + 能力边界**，不是隔离——**分析不可信目标前请自己上沙箱/虚拟机**。
2. **商业工具依赖是硬成本**：深度原生分析的三个 provider 里有两个是付费商业软件（Hopper、IDA），JEB 也是商业的。MIT 只覆盖 REA 自己的代码——**工具链的许可与合规要你自己负责**（Ghidra 免费是唯一零成本路径）。
3. **submodule 与第三方引擎的许可——它做得比多数项目严谨，但仍需你自己审**：`third_party/README.md` 逐个引擎记录了来源 URL、**固定 release、固定 commit 与许可**（Binwalk v3.1.0 MIT、Unblob 26.6.4 MIT、jadx-headless-mcp v0.7.1 Apache-2.0），并声明**上游源码保持未修改、各自保留自己的许可**；`licenses/` 下另有 `nativeaot-parser-MIT.txt`。三条关键承诺值得记下来：**REA 在分析过程中从不下载引擎**（`third_party/README.md:22-24,55-56`）、**不打包提取器**、**调用方自带的可执行文件保留其真实摘要与"未知源修订"**（除非字节与已审计版本一致，`:26-28,53-55`）。依赖树也干净：`package-lock.json` 里没有 GPL/AGPL（唯一的 GPL-2.0 是 Unicorn，属调用方自带的外部依赖）。**但两点仍需注意**：① 引擎要你自己提供，**它们的许可与合规由你承担**，且 `binwalk`/`unblob` 的用途是**解包不可信文件**，本身就是攻击面；② npm 包里带了一个**预编译的 Windows 原生模块** `native/windows/build/rea-windows-x64.node`（git 未跟踪、只在发布包里），属于供应链需要留意的部分。
4. **"本地"只覆盖分析环节，不覆盖结果**：分析确实在本地做，但工具结果会交给你的 agent ——**最终按模型供应商的数据政策上传的是分析内容**。README 的 FAQ 自己承认了这后半句。要完全离线，就得配本地模型。
5. **运行时捕获会真的动目标**：进程捕获（PTY）、adb 操作、mitmproxy 抓包、网站截屏都会以你的用户权限运行或交互目标。这既是能力也是风险面——在不受控环境里跑之前先看对应 guide 的副作用说明。
6. **热度与成熟度不匹配**：4 天涨 9 万 star、10 月 12 天 1643 次提交、版本已经 6.4.0、CHANGELOG 2467 行。这种速度意味着**接口与行为在被频繁修订**（也是官方要求你 `rea update` 的原因）。生产使用要钉版本。
7. **平台覆盖不均**：相当多能力明确只支持 Linux x64；Windows 上只有 Ghidra 的实验性 P0。macOS 上因为 Hopper 反而最完整。
8. **贡献高度集中 + AI 参与**：`morluto` 一人 52% 提交；157 个提交带 `Co-Authored-By`，其中 Claude Opus 5.5 出现 281 次（多提交可含多 trailer）。项目把 `AGENTS.md` 写成了一份很严的工程规范，这既解释了质量取向，也意味着**接手/审计要按"人机混合产出"来安排**。
9. **结论强度依赖工具输出**：REA 的设计是"给证据 + 标未知"，但最终解释权在 agent。**不要把它输出的伪代码当编译器级等价**——showcase 里的"通过 3205 个用例"是作者的重建验证，不是工具自带的保证。
10. **蹭名风险**：README 专门声明"我们从未发行或背书任何加密货币或代币，使用 REA 之名的代币与本项目无关"——说明市面上已经出现借名项目。
11. **接入它会改写你的 agent 配置**：`setup` 会往各客户端的 MCP 配置与 skill 目录写文件（有备份、原子替换、写后校验与批准流程，设计上不激进），DSH 那条路径更是**宿主进程加载的 Cordis patch**。企业环境里这属于"一次配置写入"，建议先在隔离 profile/机器上试。

## 结论

**适合谁用**：
- 做**合法**逆向/互操作/兼容性研究的人：要把反汇编、符号、IPC、source map、抓包这些证据串起来，并且希望 agent 帮你在证据上推理而不是编答案。
- 已经在用 Claude Code / Codex / Cursor 等，希望把它们接到**本地真实工具**（Ghidra 免费即可起步）上的。
- 需要"证据可追溯"的分析流程，用于写报告、做兼容实现、或做安全评估的（它把 conclusions 与 unknowns 分开的模型正是为此）。
- 想学习"怎么把危险能力做成有权限边界的 agent 工具"的工程师：约束写进 `AGENTS.md`、只读 provider、临时数据库、安装需批准、清理保留所有权——这套设计值得抄。

**不适合谁用**：
- 没装过 Ghidra/Hopper/IDA、也不想学逆向工具链的人——REA 是**编排层**，不是替代品。
- 需要稳定接口与长期支持的生产系统（6.x + 高频发布）。
- 只用 Windows 且目标涉及 ELF/EVM/进程捕获的（多数能力限 Linux x64，Windows Ghidra 还是 P0）。
- 想找"一键破解/绕过授权"的人：它明确站在"合法研究"一侧，且技术上也没有这类功能。
- 不能接受分析内容进入第三方模型的人（除非配本地模型）。

**什么情况下值得引入**：如果你**已经**有一个逆向任务，并且**已经**有 Ghidra（免费）或 Hopper/IDA，那值得花一次 `npx rea-agents setup` 试一下：让它分析一个你自己写的或明确授权的二进制，重点看两件事——① 它给出的证据链能不能被你自己复核；② 它标"未知"的地方是否诚实。这两点决定它对你是不是可信的分析伙伴。**法律与授权的判断始终在你自己这边**，项目不会替你把关。

## 未核实 / 存疑

- **star 数与"4 天 9 万"**：本环境无法访问 GitHub（DNS 被解析到 198.18.x.x 的 fake-IP）。所有 star 数字都来自**仓库自身的提交/CHANGELOG/README 记录**，属于自报；我查到了 2 万→9 万这条自洽的里程碑链，但**没有第三方或 GitHub 页面佐证**。
- **showcase 案例的可信度**：DX-Ball"通过 3205 个原始用例、复现 63 字节函数"与 TH04 的对比结论都来自作者的重建仓库与文章，本次无法访问其仓库核对。
- **各目标类型的实际成功率**：仓库内没有分析质量的量化基准；不同 provider（Hopper/Ghidra/IDA）对同一二进制的一致性也未核实。
- **"本地-only"的边界**：除宿主 agent 的模型调用外是否还有出站（例如版本检查、遥测）未逐条核实；`rea update` 的下载源也未核实。
- **各 submodule 与 `third_party/`、`licenses/` 的完整许可清单**：本次只确认了 submodule 的存在与来源 URL，未逐项核对许可条款。
- **MCP 工具的精确清单**：仓库不签入生成的 catalog（`docs/mcp-contracts.md:5-11`）。源码逐文件统计是 **139 个工具**（并与测试期望互证），但 `docs/installation.md:197` 里的 "172 tools" 对应更早的 commit——**仓库内文档与代码存在这类滞后**，实际可用工具数还取决于 provider 与目标类型。
- **治理细节**：`SECURITY.md` **没有响应时限承诺**（只说会确认并协调披露）；没有 CODEOWNERS / FUNDING / dependabot 配置。
- **IDA 的 HTTP supervisor 与 Windows Ghidra P0 的实机行为**：只读了源码与文档，未验证。
- **本次零运行时验证**：没安装、没跑任何分析；所有能力描述都来自源码/文档阅读。

## 调研方式

- **手段**：
  1. `git clone` 全量历史到 `.cache/rea`（2936 文件 / 2462 commits），基于 main HEAD `83a4445f` 走读。
  2. 自己读：`README.md`（350 行）、仓库 `AGENTS.md`、`CHANGELOG.md`、`.gitmodules`、`package.json`、`.release-please-manifest.json`，并实测了文件数/体积/目录构成/提交分布/tag/贡献者/star 里程碑时间线/CA 协作 trailer。
  3. 三个子代理分工走读：① 架构与核心实现（分层与代码量、MCP 契约面、provider 驱动机制、进程所有权、证据模型、本地性与 `REA_*` 环境变量）；② 能力面与安装门槛（17 类目标的真实依赖与平台限制、install.sh 与 setup 行为、19 个客户端、skills、docs/website 结构）；③ 安全/法律/许可与治理（免责声明边界、权限模型落地、出站网络实测、第三方许可、发布与治理、热度自证程度）。
  4. **两处我自己复核并纠正的结论**：① MCP 工具数——我最初按 `toolContract(` 出现次数估成"约 100 个"，子代理逐文件统计并与测试期望互证后确认是 **139 个**（文档里的 172 是旧 commit 的数）；② `@morluto/rea` 这个 npm 包名在仓库里查无此据。
  5. **行数一律用 .NET `ReadAllLines` 统计**——本机是 Windows PowerShell 5.1，`Get-Content` / `Measure-Object -Line` 会少算（这个坑在前一批调研里已踩过并修正）。
- **当时的限制**：
  - `web_fetch` 与 pdf MCP 的 URL 读取被 DNS fake-IP 挡住（SSRF 保护），**无法打开任何网页**（rea.tools、GitHub、npm 页面都不可达），外部信息只有搜索摘要，一律按"未核实"处理。
  - 按仓库 `AGENTS.md` 的要求与本次授权范围，**未安装、未配置、未运行** REA 或任何逆向工具。
  - 本地副本的 5 个 submodule 目录在 clone 后为空（未 `--recursive`），因此 `third_party/` 下的代码内容未读。
- **复现方式**：
  ```powershell
  git clone https://github.com/morluto/rea.git          # 需要 submodule 时加 --recursive
  cd rea && git checkout 83a4445f440fab978ad9376b4260acae3f5fb189
  ```
  文档内行号会随版本漂移，请以 commit hash 为准。
