# Archify

- **调研日期**：2026-10-10
- **上游地址**：https://github.com/tt-a1i/archify
- **调研版本 / commit**：稳定版 `v3.0.1`（2026-09-28）；本次走读的是 `main` HEAD = `58ab0b98d2d0def8e49dced705c7b566936723ec`（2026-10-10）。注意 DSH 插件捆绑的是 `7158026e852f3aa6578c741e673b46d7878c92c1`（晚于 v3.0.1 tag，含 #704 的 Skill 安装修复），两者不是同一份字节。
- **本地副本**：`D:\github\research\.cache\archify`（798 个文件；git pack 672.87 MiB）
- **许可证**：MIT，但为**双版权**（`Copyright (c) 2026 tt-a1i (Archify)` + `Copyright (c) 2025 Cocoon AI`）——v1.0 的原始设计归 Cocoon AI，Skill 元数据写着 `based_on: Cocoon-AI/architecture-diagram-generator (MIT, v1.0)`（`archify/SKILL.md:8`）

> **本次未安装、未运行任何东西**：仓库 `AGENTS.md` 明确"只有用户明确要求才能安装/更新/删除 Archify 安装"。本文所有结论来自读源码、文档与 git 历史。

## 一句话结论

Archify 是**给 coding agent 用的画图 skill**：agent 按 schema 写一份 typed JSON，仓库里那套**零运行时依赖的 Node CLI**（Node ≥18）把它渲染成一张**自包含的交互式 HTML**（实测每张约 745 KB），交付前还要过 schema / 布局 / HTML / 真实浏览器四道闸，失败时给出机器可读的修复回执。

它真正值得借鉴的，是把"图丑不丑"这个主观问题**拆成可机器校验的工程问题**：typed IR + 确定性布局 + 原子交付 + 首过可用率基准。反过来，它**自己不生成任何内容**——图的语义质量完全取决于你的 agent 和模型，而厂商自己测出来的"第一次就能用"的比例只有 **53%–67%**（2026-07，15 次运行）。

## 项目概况

| 项 | 值 | 来源 |
| --- | --- | --- |
| 仓库地址 | https://github.com/tt-a1i/archify | 已验证（git remote） |
| 许可证 | MIT，但 `LICENSE:3-4` 是**双版权**：`Copyright (c) 2026 tt-a1i (Archify)` + `Copyright (c) 2025 Cocoon AI`——v1.0 的原始设计（暗色主题、调色板、网格背景、摘要卡布局、JetBrains Mono 排版）归 Cocoon AI；该声明被 `scripts/package-smoke.mjs:73-74` 与测试强制保留 | 已验证 |
| 第三方素材 | `THIRD_PARTY_NOTICES.md` 逐条列出 Simple Icons 16.28.0 的**单个图标许可**，其中混有 **CC-BY-NC-SA-4.0（Vue）、CC-BY-SA-3.0（Jenkins）、CC-BY-4.0（Angular）、CC-BY-SA-4.0（Rust）** 等非纯宽松条款；JetBrains Mono 为 SIL OFL-1.1 | 已验证 |
| 定位 | Agent skill（非库、非编辑器）：给 Claude Code / Codex / Cursor / opencode / Copilot / Claude.ai / Hermes / DeepSeek Harness 用 | 已验证（`README.md:391-401`、`archify/SKILL.md`） |
| 技术栈 | Node ESM（`.mjs`，296 个文件）+ 自研渲染器/布局校验器 + 构建期 Astro 官网；**运行时零 npm 依赖** | 已验证（`archify/package.json:31-38` 只有 devDependencies） |
| 提交数 / 贡献者 | HEAD 口径 926 commits / 83 人（全 refs 口径为 1,934 / 101）；Top1 `tt-a1i` 600 次（**65%**），Top5 约 79% | 已验证（`git shortlog -sn HEAD`） |
| 最新版本 | `v3.0.1`（2026-09-28）；共 22 个 tag（含 2 个 DSH adapter tag） | 已验证（`git tag`） |
| 首次提交 | 2026-04-15，"Initial commit: Archify 2.0" | 已验证 |
| 仓库体积 | git pack 672.87 MiB；工作树仅 61.3 MB（其中 `docs/` 30.8 MB、`archify/` 10.1 MB、`examples/` 9.4 MB） | 已验证 |
| 代码规模 | 追踪文件 798：`.mjs` 296、`.md` 131、`.json` 121、`.html` 58；CLI 单文件 `archify/bin/archify.mjs` 6,936 行；测试 `archify/test/*.test.mjs` **165 个文件 / 62,919 行**（另有 fixtures 与 helpers，目录共 206 个文件） | 已验证（行数用 .NET `ReadAllLines` 复核） |
| CI 规模 | `.github/workflows/ci.yml` 有 12 个 job：Node 18/20/22/24 矩阵测试、真实 Chrome 浏览器门、zip 字节一致性、已发布 manifest 校验、三平台打包冒烟、官网构建、Windows 路径可移植性等 | 已验证 |
| 稳定版分发物 | `archify.zip` 2,157,447 字节 / 106 条（清洗过的 Skill 包） | 已验证 |
| 热度（star） | 仓库自维护的 star-history 图（`star-history` 分支，2026-10-05 更新）显示 5 月起步、8 月前数千、9 月起陡增，终点越过 60K 网格线，按坐标外推约 **7–8 万** | **估算**（方法见「调研方式」，非 GitHub 官方页面） |
| 官方自述的认可 | "GitHub Trending 周榜全语言第 1"（2026-09-01，仅 creator 的 X 截图外链）、"skills.sh 130K+ installs"（徽章外链）、QbitAI 报道两篇 | **未核实**（本环境无法打开网页） |

版本节奏：`v2.0.0` 起（2026-04-15，首日连发 6 个版本）→ 7–8 月约每 1–2 周一版 → `v3.0.0`/`v3.0.1` 同日（2026-09-28）→ DSH adapter `v1.0.0`（2026-10-05）。提交强度在 2026-09 出现异常高峰：单月 639 个提交（其次 8 月 104 个），对应其"登上 trending"的窗口。

## 它解决什么问题

**痛点**：让 agent 画架构图这件事，现状有三个老大难——Mermaid 之类只能换皮、改不了信息结构；draw.io/Figma 是给人用的，agent 进不去；直接让模型吐 SVG/HTML 则完全不可控，图能不能看全靠运气。

**它的做法**（`README.md:309-324`、`PRODUCT.md:29-34`）：
1. 把图变成**有 schema 的数据**（typed JSON IR），而不是让模型直接写图形代码；
2. 由**确定性布局 + 校验器**负责"能不能看"（重叠、标签避让、路由交叉、HTML/SVG 合法性）；
3. 交付前**原子替换**：候选产物必须先过所有闸，才允许替换上一次的成品（失败保留旧的）；
4. 输出**自包含 HTML**，发出去就能看，接收方不需要装任何东西。

**它明确不做**（`README.md:410`、`PRODUCT.md:21-25`）：不做通用绘图编辑器、不是 Mermaid 主题、不自动解析 Mermaid 成图（只取语义重写，`SKILL.md:64-70`）、无通用自动布局引擎、无托管分享、无 WYSIWYG、不做数值图表/仪表盘。反参照写得很直白：反对"换主题的 Mermaid 美化器""编辑 chrome 变成产品的 WYSIWYG""暗示不存在关系的动效 demo"。

**工程纪律**是这个项目最有意思的副产品：ROADMAP 里每一个已交付切片的结尾几乎都在重复同一句话——"没有新增 schema、依赖、托管服务、遥测或移动端功能"（例如 `ROADMAP.md:16,20,22,26,30`）。这种"以不做什么来定义边界"的写法，在个人主导的开源项目里不常见。

## 核心机制

### 质量工程（这是它最突出的部分）

在一个"个人主导 + agent 辅助"的项目里，它的质量门密度不常见：

- **12 个 CI job**（`.github/workflows/ci.yml`）：Node 18/20/22/24 四版本矩阵测试（`:42`）、**真实 Chrome 浏览器门**（`scripts/run-browser-tests.mjs:8-38` 列了 29 个浏览器测试文件，`scripts/run-browser-tests.mjs:42-45` 在缺 Chrome 时直接 `exit 1`，**不允许跳过**）、zip 新鲜度（`:141-148`，强制"提交进仓库的 zip 必须与重新构建的字节一致"）、已发布 manifest 校验（`:150-217`，与 Release 资产字节比对）、三平台打包冒烟（`:219-226`）、官网构建、Windows 路径可移植性。
- **浏览器检查是量出来的，不是"看起来没问题"**：例如 `archify/test/architecture-reading-size-browser.test.mjs:22` 真跑一次 render，`:36` 用 CDP 打开产物，`:48` 量主字号，`:57` 断言没有横向溢出。
- **golden 文件回归**：`test/golden.mjs` 要求新渲染与仓库内已提交的示例 HTML 一致——这也是它敢声称"渲染确定性"的依据。
- **有多语言 README 一致性测试**（`ci.yml:32-34` → `archify/test/readme-showcase.test.mjs:170` 断言每个 README 头部都链接到其他语言）。顺带说，这类测试的存在也解释了为什么它是多语言 README 而不只是英文。
- **没有 lint job**（workflows 里搜不到 lint/eslint/prettier）——这不是疏忽就是风格选择，本次未核实原因。

### 一个 skill + 一个零依赖 CLI

- **Skill 是给 agent 的流程契约**（`archify/SKILL.md`，109 行）：选五种类型之一 → 读 authoring defaults 与对应示例 → **直接写完整 candidate**（明令禁止先跑 doctor、先画临时图、先列目录，`SKILL.md:30-31`）→ 用**一条命令**收口：`node bin/archify.mjs finalize <type> <candidate.json> <output.html> --quality showcase --json`（`SKILL.md:32-38`）。这条命令内部串起 validate / deliver / 严格 check / 真实浏览器 browser-check 四道闸（`SKILL.md:40`）。
- **落盘约定**：每个请求一个独立目录 `.archify/<type>-<slug>-<YYYYMMDD-HHMMSS>/`，内含 `candidate.json` 与 `<slug>.html`，修复时复用同目录（`SKILL.md:15`）。
- **修正被硬性限流**：同一问题最多 2 次聚焦修正 + 1 次有证据的重试（`archify/references/delivery-contract.md:9`）；感知复核的 `correction_rounds` 只允许 0/1/2（同文件 `:558-560`）；Architecture 交叉问题只给 1 轮"仅移动坐标"的重排，产物落到 `review-2`/`review-3`（`SKILL.md:78`）。
- **诚实性要求写进契约**：非零退出永不等于成功（`SKILL.md:42`）；必须把"自动检查 / 浏览器证据 / 截图 / 人工感知复核"当成四种不同结果分别汇报，不得声称看过没看过的图（`SKILL.md:88,109`）。
- **CLI 有 17 个入口**（派发在 `archify/bin/archify.mjs:6865-6928`，用法全文 `:2176-2199`）：`render`、`validate`、`deliver`、`finalize`、`preview`、`compare`、`migrate`（仅 workflow→schema v2）、`inspect`（= `validate architecture --layout-json`）、`check`（静态 HTML 检查）、`visual-check`、`browser-check`（Chrome 取证）、`guide`、`brands` / `brands capture`、`examples`、`doctor`、`demo`。
- **退出码是有约定的**：用法/参数错误走 `fail()` 默认 **2**（`archify.mjs:2201-2204`）；校验或交付失败 **1**（`:6781-6792`）；`browser-check` 的"跳过"在文档里定义为 **2**（`references/delivery-contract.md:405-407`）。配合 SKILL.md 那句"非零退出永不等于成功"，agent 不容易把失败读成成功。
- **零依赖是真的**：`archify/bin/archify.mjs:3-8` 只 `import` Node 内建模块；`archify/package.json` 没有 `dependencies`，只有 6 个 devDependencies（ajv、parse5、saxes、simple-icons、skills、yaml），且 ajv/simple-icons 只在前述两个生成器里用到、而生成器**被排除在打包之外**；schema 校验用的是 AJV standalone 预编译产物（`renderers/shared/generated-validators.mjs`，内联了 `ucs2length`）；`viewer/*.js` 无任何 import。`engines.node >= 18`（`archify/package.json:11-13,31-38`）。

### 渲染：IR → 自包含 HTML

- 五种图类型各有 schema + renderer + 示例：architecture / workflow / sequence / dataflow / lifecycle（`SKILL.md:52-58`）。类型→渲染器是固定映射（`archify.mjs:2217-2227`）：architecture 配 `grid.mjs`（栅格定位）+ `routing.mjs`（路由）+ `labels.mjs`（标签避让）；workflow 的编译主体在 `workflow-compiler.mjs`；lifecycle 复用 architecture 的路由与自己的 `grid-routing.mjs`。
- **viewer 源码在仓库根的 `viewer/`**（约 16 个模块：`focus.js` 79 KB、`guided-views.js` 80 KB、`route-probe.js` 47 KB、`semantic-radar.js` 29 KB、`export.js` 67 KB、`viewer.css` 168 KB、`template.source.html` 163 KB…），由 `scripts/generate-viewer.mjs:14-29` 编译进 **`archify/assets/template.html`（710.9 KB）**——即最终交付 HTML 里内联的完整交互层。
- **"自包含"经得起检查**：`template.html` 里 51 处 `http` 全部是注释或 `data:` URI，没有外链资源；JetBrains Mono 以 base64 WOFF2 内联（`template.html:35-168`，`@font-face` 在 `:145-166`）；渲染阶段只做占位符替换（`renderers/shared/utils.mjs:144-195`、`renderers/shared/cli.mjs:262-273`）。
- 产物量级（仓库内已生成的样例实测）：`archify/examples/*.html` 在 **741–750 KB**；仓库根 `examples/*.html` 762–816 KB；架构差异图因为要并排两份 SVG 且双主题，单张可到 **2.12 MB**（`examples/checkout-platform-delta.html`）。其中内联字体子集约占 96 KB（`CHANGELOG.md:65`）。
- **确定性是被测试固化的**：`archify/test/golden.mjs:2-3` 明确"渲染器是确定性的，新渲染必须与已提交的示例 HTML 一致"，因此仓库里存着 golden HTML 供比对。架构定位刻意**不是**自动布局：`renderers/architecture/grid.mjs:1` 写着 "Not auto-layout — fixed cell math only"。渲染路径没有 `Math.random`（只有品牌抓取的超时与临时文件名用到时间戳/随机字节）；workflow 编译器靠排序与代价比较决策（`workflow-compiler.mjs:1274,4197,4222`）。

### 校验、修复回执与原子交付

- **`validate` 是三段串联**（`archify.mjs:6699-6816`）：(a) JSON Schema（预编译 AJV，`renderers/shared/validator.mjs:38-85`）；(b) 渲染器自身的布局/路由/标签规则（`geometry.mjs`、`route-quality.mjs`、`workflow-compiler.mjs`）；(c) `scripts/check-render-output.mjs` 对产出的 HTML/SVG 做结构与构图检查（`single_svg`、`finite_svg`、`composition/*`）。仓库里能数出 100 多个规则码，真实例子：`schema/additionalProperties`、`composition/label-route-clearance`（标签与路由净距）、`clean-flow/edge-through-node`（连线穿过节点）、`composition/excessive-route-detour`（绕行过长）、`workflow/solver-budget-exhausted`。
- `validate --json` / `deliver --json` 失败时输出**单个结构化 JSON**：稳定的 `diagnostics[]`，结构为 `{code, severity, message, subject, evidence, supportedFixes}`（`archify.mjs:2333-2342`）——即"哪个规则、多严重、卡在哪个对象、实测证据是多少、只有哪些修复手段是被支持的"，不把 Node 堆栈倒给使用者（`ROADMAP.md:20`、`CHANGELOG.md:62`）。
- 交付是**原子**的：先写同目录候选 → 过产物与组合检查 → 生成字节/SHA-256 回执 → 用一次 rename 提交；任一步失败都保留旧产物（`ROADMAP.md:36`）。
- `preview` 是**只绑 127.0.0.1 随机端口**的桌面模式：只监听一个 JSON 源，只有通过全部校验的修订才会替换预览，失败保留上一次可用产物（`SKILL.md:105`、`ROADMAP.md:22`）。
- 产物旁边会留 machine-readable sidecar（`<stem>.finalize-summary.json`、`.delivery.json`，`references/delivery-contract.md:28-29,267`）。

### 源码证据（repository evidence）与 Architecture Delta

- 当图必须反映真实代码时，可启用 evidence：`meta.repository` + 每个节点的 `sources`，渲染前做 Git 校验（顶层目录、origin 匹配、commit、blob、路径包含、行号边界），节点上标 `SRC n` 并链接到**固定 commit** 的文件与行范围（`README.md:319`、`CHANGELOG.md:35`）。2026-09 起扩展到全部五种图类型（此前仅 architecture）。
- `compare`（Architecture Delta）**只支持 architecture**（`archify.mjs:3584`），对比 Before/Delta/After 三份快照，产出带机器回执的差异图（`delta/architecture-delta.mjs`）。它要求节点有**稳定 id**（缺 id 直接失败，`:264-268`），回执里带 base/head 的 `rawSha256` 与 `semanticSha256`、`summary`（added/changed/removed/moved/rerouted）、以及 `proofLevel`——只有双方 revision 都是 40 位十六进制且证据已验证时才升级为 `revision-pinned`（`:285-290`），并显式声明**不推断影响、风险或合并安全性**（`README.md:268`）。

### 分发：清洗过的 zip 与窄接口集成

- `archify.zip` 由 `scripts/stage-clean-skill.mjs` + `write-deterministic-zip.mjs` 生成：只取 Git 索引里 `archify/` 的追踪文件（拒符号链接），**排除 208 个 test 文件、package-lock 与 2 个生成器**，并清掉 `package.json` 的 `scripts`/`devDependencies`（打包后 349 B vs 源码 1,730 B）；抽检 `bin/archify.mjs`、`SKILL.md`、`schemas/README.md` 与 HEAD **逐字节一致**（只做剔除，不重写）。
- 换句话说，仓库里的 `archify/`（315 文件 / 10.4 MB）并不等于用户装到的东西：真正随包分发的是 `bin/`、`renderers/`（1.5 MB，含 518 KB 预编译校验器）、`schemas/`、`references/`、`delta/`、`recipes/`、`migrations/`、`assets/template.html` 和 `examples/`（其中 4 MB 是已渲染的示例 HTML）；`test/`（3.2 MB）与两个生成器属于开发资料。
- **DSH 集成**（对本机环境最相关，`integrations/deepseek-harness/README.md`）：npm 包 `@tt-a1i/archify-dsh@1.0.0`，**只声明支持精确宿主 `@deepseek-ai/dsh@0.1.2-rc.1`**，Node `^22.19.0 || >=24.0.0`，无 dependencies/peerDependencies、无 install/postinstall 钩子、无遥测、无自带网络客户端（`README.md:5,11,150-155`）。它注册一个名为 `archify-plugin` 的文件系统 Skill provider；其 `cordis.patch.yml` 用 `!!js` 表达式在 **DSH 宿主进程**里解析已安装包路径——README 自己点明"这是宿主加载的代码，要当代码审"（`README.md:15-17`）。已知限制：shell 产物不会出现在 Web 的 Produced Files 区，需要 agent 回传精确工作区路径（`README.md:138-140`）。

### 治理与贡献

- **外部 PR 是接受的**（与不少"开源但闭门"的项目不同）：有 PR 模板、3 个 issue 模板，社区包走单文件 `community/packages/<name>.json` 提交（`community/README.md:24-31,94`）。
- 分支模型是 **dev → main 双分支**，且 `CONTRIBUTING.md:145-149` 明确"CI 通过不等于试用验收通过"；**没有 CLA/DCO**，只声明同意 MIT（`:161-163`）；CodeRabbit 是**咨询性**评审（`.coderabbit.yaml:2` 自称 advisory pilot），`REVIEWING.md:30-35` 定义了四级结论（blocking / required evidence / suggestion / ready）。
- `ROADMAP.md:214` 有一份 **12 条 "Not planned"** 清单（自动布局、Mermaid 解析器等），把"不做"也写进路线图。
- `SECURITY.md:9-14` 只承诺维护 latest stable 与 main、不承诺回溯修复；漏洞走 GitHub 私密报告（`:18`）。
- **无遥测**：`community/README.md:94` 明示，且 DSH 插件的测试用断言禁止源码里出现 telemetry/OTLP（`integrations/deepseek-harness/test/adapter-security.test.mjs:18`）。
- 设计规范（`DESIGN.md`）、产品定义（`PRODUCT.md`）也都是版本库内的一等公民——`PRODUCT.md` 甚至写了"反参照"和可访问性要求（键盘访问、非颜色状态提示、`prefers-reduced-motion`）。

## 实测与标称数据

### 官方自测：ordinary-model-floor（这是本文里最有价值的一组数字）

`benchmarks/ordinary-model-floor/` 是一个**"第一次就能用"的交付闸门**，不是模型排行榜：三个条件全过才算数——语义要求齐全且连接正确、真实 CLI `validate --quality showcase` 通过、**有人真的看过最终产物并记录无缺陷**（`benchmarks/ordinary-model-floor/README.md:5-11`）。它还专门声明"仓库里的参考 fixture 只证明测试装置能跑，**不是基准证据**"（同文件 `:17`）。

仓库里带三份结果（均记录于 2026-07-26，agent 为 `pi`，三个"普通"模型 × 5 个用例 = 15 次，全部 attempt 1、candidate 冻结后不得事后编辑）：

| 运行 | 首过可用率 | minimax-m3 | qwen3.7-plus | deepseek-v4-flash | 失败聚类 |
| --- | --- | --- | --- | --- | --- |
| `2026-07-26-pi-three-models.json` | **10/15 = 66.7%** | 4/5 (80%) | 3/5 (60%) | 3/5 (60%) | validation 5、visualReview 5 |
| `…-postfix.json` | **8/15 = 53.3%** | 4/5 (80%) | 2/5 (40%) | 2/5 (40%) | validation 6、visualReview 7、semantic 2 |
| `…-quality-first.json` | **8/15 = 53.3%** | 3/5 (60%) | 3/5 (60%) | 2/5 (40%) | validation 5、visualReview 7、semantic 2 |

要点：失败几乎都出在**校验与人工视觉复核**，语义错误反而少；三份结果的 repository commit 各不相同，说明这是 v2.x 时代的结果，**不能直接外推到 v3.0.1**。

另有一个基准 `benchmarks/repair-rounds/`：用确定性缺陷注入（`defects.mjs` 里 8 类缺陷）测"要修几轮才收敛、每次修花多少 token"，指标表在它的 README 里。这两个基准合起来，回答的是"这个工具在普通模型手里靠不靠得住"——而不是"它能画出多漂亮的图"。

**能被外部自证的工程证据**：`docs/gallery/manifest.json` 记录了 11 个 checked-in 场景（`docs/gallery/artifacts/` 与 `docs/gallery/sources/` 各 11 份）的 `entryCount=11`、`checkCount=99` 与每个产物的 SHA-256。README 里说的"Proof Lab 11 个场景与 validation receipts"在仓库内就是这个 manifest（**没有独立的 receipt 文件**，全仓唯一的 `*.receipt.json` 是 `examples/checkout-platform-delta.receipt.json`）。

### 实测的体积数据（本次亲自量的）

| 项 | 数值 | 来源 |
| --- | --- | --- |
| 单张交付 HTML | 741–750 KB（5 个样例） | `archify/examples/*.html` 实measure |
| 内联 viewer 模板 | 710.9 KB | `archify/assets/template.html` |
| Skill 包（zip） | 2.06 MB / 106 条 | `archify.zip` |
| viewer 源码总量 | 约 700 KB JS + 168 KB CSS | `viewer/` 逐文件统计 |

### 第三方实测

**没有找到任何独立第三方对 Archify 的实测或评测。** 所有质量数字都来自仓库自带的基准。

## 硬件 / 软件门槛

| 项 | 门槛 | 来源 |
| --- | --- | --- |
| Node.js | Skill 本身 `>=18`；DSH 插件要求 `^22.19.0 \|\| >=24.0.0` | `archify/package.json:11-13`、`integrations/deepseek-harness/package.json` |
| Agent 能力 | **这是真正的门槛**：要有一个能读长契约、会跑 shell、能按 diagnostics 迭代的 coding agent；官方首过可用率 53%–67% 说明"随手一问就出好图"不成立 | `benchmarks/…/results/*.json` |
| Shell 访问 | 必需（agent 要执行 `node bin/archify.mjs …`）。Claude.ai 上传路径明确写"取决于沙箱是否有 Node.js 访问" | `README.md:398` |
| 网络 | 渲染本身不联网。仅两处可选联网：**更新检查**（`deliver` 时起一个 1 秒硬超时的子进程，GET 固定清单 `https://tt-a1i.github.io/archify/skill-updates/archify/stable.json`；`finalize` 自己起一个 4 秒期限的检查并**显式关掉内层 deliver 的重复检查**；`ARCHIFY_UPDATE_CHECK_DISABLED=1` 可整体关闭）与 **`brands capture <url>`**（只在明确要求某个品牌标记时才发起请求） | `bin/delivery-update.mjs:6,48-51`、`bin/finalize.mjs:699,750`、`scripts/update-contract.mjs:3`、`README.md:222` |
| API key / GPU / 数据库 | 全都不需要 | 已验证（无任何相关依赖与配置） |
| 磁盘 | Skill 包约 2 MB，解包后 `archify/` 约 10 MB；每张图约 0.75 MB | 已验证 |
| 支持平台 | Linux / macOS / Windows（DSH 插件的三平台发布门也只覆盖 CLI 与 tarball） | `README.md:391-401`、`integrations/deepseek-harness/README.md:53` |

**关于本机环境的一个具体提醒**：这台机器的 `DSH_PROFILE=desktop`，而插件文档给的安装命令是 `dsh plugin --profile web add @tt-a1i/archify-dsh@1.0.0`；插件的验收矩阵里 **Electron 桌面版被明确列为"pending native acceptance"，并写明"no Windows/Desktop success claim"**（`integrations/deepseek-harness/README.md:109`）。也就是说：桌面 profile 装它属于**未验收组合**，真要装应先确认宿主机版本是否就是它声明支持的 `0.1.2-rc.1`。

## 上手方式

```bash
# 1) 作为 skill 安装（Claude Code / Codex / Cursor / opencode 等）
npx skills add tt-a1i/archify -g

# 2) 或按平台显式安装（示例：Cursor，全局）
npx -y skills add tt-a1i/archify --skill archify --agent cursor --global --copy --yes

# 3) 不安装先试
npx skills use tt-a1i/archify@archify --agent codex

# 4) DSH 插件（社区集成，注意支持范围见上）
dsh plugin --profile web add @tt-a1i/archify-dsh@1.0.0
```

给 agent 的话术（`README.md:68-71,242-246`）：

```text
Use Archify to draw: Browser -> API -> Redis cache -> PostgreSQL fallback.

# 要源码证据时：
Analyze this repository, then use archify to create a high-level runtime architecture diagram.
Show 8–12 core components, one primary path, external dependencies, and trust boundaries.
Put supporting detail in cards instead of adding more edges.
```

手动/CI 用法（`README.md:341-349`）：

```bash
cd archify
node bin/archify.mjs doctor                                  # 环境自检
node bin/archify.mjs guide "Show CI/CD checks, approval, deploy, and rollback"
node bin/archify.mjs validate workflow examples/agent-tool-call.workflow.json --quality showcase --json
node bin/archify.mjs deliver  workflow examples/agent-tool-call.workflow.json out.html --quality showcase --open --json
node bin/archify.mjs compare architecture base.json head.json delta.html --json   # 架构差异
```

## 坑与风险

1. **图的质量上限 = 你的模型能力上限**。它把渲染做成了确定性的，但**语义对不对仍由模型决定**；官方首过可用率 53%–67% 是清醒的自我披露，也意味着实际使用中"问一次就满意"的概率大约一半。修正又被硬性限流（2 次聚焦 + 1 次重试），差的模型会直接卡在门槛上。
2. **单点主导（bus factor = 1）**。`tt-a1i` 一人占 65% 提交，`.github/CODEOWNERS` 只有 4 行且全是 `@tt-a1i`；安全响应与 CI 治理也集中在同一人。83 个贡献者里 Top5 约 79%。另外 2026-09 那个 639 提交的高峰（占全部提交的 69%）里有 **266 个 merge commit**，更像是"长期开发历史批量合入 main + 单人高频提交"，而不是 83 人同时活跃——所以别把那一个月的数字读成社区规模。
   项目**大量使用 AI 辅助开发**：49 个提交带 `Co-Authored-By` trailer，其中 Devin 33、Cursor 10、Claude Fable 5 7 等，另有 172 个远程分支（大量 `codex/*`）。这不直接等于质量差（它的 CI 门反而比多数同类项目严），但意味着**代码审计与长期接手的成本结构和传统项目不同**。
3. **文档与实现会漂移**（实测到两处）：`README_ZH.md:218` 说 agent switcher 只支持 cursor/codex/claude-code/opencode，但 `website/src/scripts/start.js:7` 已含 `github-copilot`，而 README 的安装能力表里**根本没有 Copilot 这一行**；`docs/index.html` 构建产物与 `website/src` 也不一致。
4. **仓库体积异常，而且原因很具体**：工作树只有 61 MB，git pack 却有 **672.87 MiB**。主因是 **`archify.zip` 在历史里有 734 个版本、原始合计约 1.93 GiB，被 362 个提交反复改动**——因为 CI 强制要求"仓库里的 zip 必须与重建结果逐字节一致"，每次改 skill 都会重新提交一份 2 MB 的压缩包。次要来源包括一个 5.9 MB 的 mp4、若干赞助商 PNG 与 28 MiB 生成 HTML。`.gitattributes` 只把生成 HTML 标了 `linguist-generated`，**没有**把 `archify.zip` 标成 binary。clone/fetch 成本因此明显高于同类项目。
   （另外 `journal/` 是 62 篇研究笔记、`experiments/` 是 3 个一次性验证，都被 ROADMAP 引用但没有定义生命周期，实质是版本库里的开发日志。）
5. **联网行为虽少但存在，且设计得比较克制**：更新检查只 GET 固定的 GitHub Pages 清单、1 秒硬超时、`finalize` 内层显式关闭以免重复请求，README 也明确披露服务端只看到常规 HTTP 元数据（IP/时间），可用 `ARCHIFY_UPDATE_CHECK_DISABLED=1` 完全关闭；品牌抓取只在你明确要求某个品牌标记时才发生，且代码里有私网/端口/凭据校验与 **DNS 钉扎**（防 DNS 重绑定），另有 `ARCHIFY_BRAND_ALLOW_PRIVATE`、`ARCHIFY_BRAND_CAPTURE_TIMEOUT_MS` 两个开关。这些属于"知情可选"，不是遥测。
6. **变现与同名混淆**：README 里有 **7 家赞助商**（Kimi Work、Supercode、OpenLux、ZTURBO、PackyCode、Infistar、EverMind），其中多条的注册链接带 `aff`/`utm` 返佣参数；README 专门写了"第三方在线服务与付费计划由其各自运营方负责，不代表 Archify"、"相似的名字或域名不构成官方关系"（`README.md:11,431-435`）。搜 "Archify" 时容易撞上第三方付费服务——免责声明覆盖了法律表述，但不构成技术隔离。
7. **能力边界窄且是刻意的**：不是绘图编辑器、不做 Mermaid 自动解析、不做通用自动布局、无托管分享、无 WYSIWYG。想找这些的会直接失望；反过来，这也意味着它不会长成一个你不想要的庞然大物。
8. **DSH 集成面窄**：只声明支持精确的 `dsh@0.1.2-rc.1`，Desktop/Windows 未验收（见上）；且其 `cordis.patch.yml` 的 `!!js` 是在**宿主进程**里求值的代码——README 主动要求"升级时把它当代码审"，这个提醒值得照做。
9. **产物的安全属性**：交付物是一段**自包含的交互式 HTML/JS**（没有服务端、没有外链脚本，字体也已内联）。这带来一个使用习惯问题——**"打开同事发来的图"等于执行一个本地 HTML 文件里的脚本**。项目本身没有声称这是安全边界（没有相关文档），这条是本次调研的分析性提示，不是项目结论。
10. **第三方素材里混着非宽松许可**（商业使用者要留意）：项目本体是 MIT，但 `THIRD_PARTY_NOTICES.md` 逐条列出的 Simple Icons 图标集里，个别图标是 **CC-BY-NC-SA-4.0（Vue）、CC-BY-SA-3.0（Jenkins）、CC-BY-SA-4.0（Rust）** 这类带"非商业/相同方式共享"条款的许可。项目做对了两件事——把许可逐条列清、并在打包时强制保留声明（缺失即 fail closed）——但**如果你的产物用到了这些品牌图标，随之而来的署名/非商业条款是你要自己评估的**，MIT 不会替它们兜底。这也是它把 `brands` 做成显式命令、而不是自动猜品牌的原因之一。

## 结论

**适合谁用**：
- 日常要给 agent 产物做架构图 / 流程图 / 时序图 / 数据流 / 状态机，并且**想要一份能直接发出去的成品**（单文件 HTML，收件人不用装东西）。
- 团队里已经在用 Claude Code / Codex / Cursor / opencode 等，想让"解释这个仓库"变成可交付物；尤其是需要**源码证据**（`SRC n` 精确到 commit + 行号）做评审材料的场景。
- 想要 PR 前后架构对比（`compare`）而又不想引一堆基础设施的。
- 想学习"怎么把主观质量问题工程化"的人：typed IR、确定性校验、原子交付、首过可用率基准、真实 Chrome 门、以及那份反复声明"没有新增依赖/遥测/托管服务"的 ROADMAP。
- **想参与贡献的人**：外部 PR 是被接受的（无 CLA/DCO，只有 MIT 声明），issue/PR 模板、评审分级、社区包目录都齐备——这一点在当前"开源但只收 issue"的风气里算加分项。

**不适合谁用**：
- 想要 WYSIWYG 拖拽编辑、Mermaid 一键美化、或在线协作画布的。
- 只想"一句话出图"且不接受迭代的——官方数据说明这不现实。
- 使用**非 coding agent 环境且没有 Node/shell** 的（Claude.ai 上传路径依赖沙箱有 Node）。
- 需要长期稳定 API/契约承诺的：仍在 2.x→3.x 的快速演进期（3.0.0 就移除过 guided/story 视图与普通 share card 导出，`CHANGELOG.md:16`）。
- 主力环境是 **DSH 桌面版**的：插件目前把 Desktop 列为未验收（社区集成 + 精确宿主版本 + `!!js` 宿主补丁），最好等官方宿主版本进入其验收矩阵再上。

**什么情况下值得引入**：如果你已经在用 coding agent 并且**架构沟通是重复性工作**，先用 `npx skills add tt-a1i/archify`（**不带 `-g`**，装到项目级）试一周：让 agent 对一个真实仓库出图，重点看"第一次能不能用"以及 diagnostics 是否可行动。它零依赖、无 API key、无服务端、产物可离线打开，试错成本几乎只有模型 token 和你的审图时间；反过来，如果试出来你的模型首过经常失败，这套东西的价值会迅速下降到"你替它改 JSON"。

## 未核实 / 存疑

- **star 数**：本环境无法访问 GitHub 页面（DNS 被解析到 198.18.x.x 的 fake-IP）。给出的"约 7–8 万"是我按仓库 `star-history` 分支自绘 SVG 的坐标轴（20K/40K/60K 三条网格线的 y 坐标）线性外推出的**估算值**，曲线终点在 60K 网格线之上；不是官方数字，也不是 GitHub 页面读数。
- **"GitHub Trending 周榜全语言第 1"、"skills.sh 130K+ installs"、QbitAI 两篇报道、Trendshift 徽章、star-history 站点**：全部只有外链或徽章，**无法在仓库内自证**。其中 "130K installs" 只是 README 里一个 shields.io **静态徽章**（不是 CI 产物，也没有对应数据文件）。间接吻合的只有时间线：2026-09 的 639 个提交与其声称的 trending 周（2026-09-01）对得上，但这只证明活跃度。
- **npm / GitHub Releases 的线上发布状态**：本次禁网，`@tt-a1i/archify-dsh@1.0.0`、`archify.zip` 的线上字节与仓库内文件是否一致未核实。
- **各宿主实际解析的 skills 目录**：README 里那张"安装位置"表属**声称**；仓库内唯一被测试断言的落盘是项目级 `.agents/skills/archify`（`archify/test/skill-installation.test.mjs:89,113`）。落盘位置最终由外部 `skills` CLI 决定，逻辑不在本仓库。
- **渲染耗时 / 大图性能**：仓库里没有可用的时间数据，本次也未运行 CLI。
- **v3.0.1 的真实首过可用率**：现有三份基准结果都是 2026-07 的 v2.x 快照，repository commit 各不相同，不能外推到当前版本。
- **本机 DSH 是否落在插件的支持范围**：`DSH_PROFILE=desktop`，且沙箱内读不到宿主版本号（profile 里没有 `@deepseek-ai/*` 包，应用本体在 `app.asar` 内）。**未确认**本机宿主版本是否等于它要求的 `0.1.2-rc.1`。
- **第三方独立评测/实测**：没有找到任何一份。

## 调研方式

- **手段**：
  1. `git clone` 全量历史到 `.cache/archify`（798 文件 / 926 commits），走读基于 main HEAD `58ab0b98`；所有 `文件:行号` 出自该副本。
  2. 自己读：`README.md`（473 行）、`archify/SKILL.md`、`CHANGELOG.md`、`PRODUCT.md`、`ROADMAP.md`、`integrations/deepseek-harness/README.md`、`archify/package.json`、`skill-release.json`、`benchmarks/ordinary-model-floor/README.md`，并实测了文件体积、tag 日期、提交/贡献者统计、测试数量、CLI 子命令清单、打包内容与 co-author trailer。
  3. 三个子代理分工走读：① 渲染/CLI 内部（renderers、校验规则、确定性、网络行为、compare）；② skill 契约与各平台集成（SKILL.md、schemas、打包链、DSH 集成）；③ 质量工程与治理（CI、测试、基准、许可与第三方、维护风险、商业化信号）。
  4. **star 数估算方法**：从 `origin/star-history` 分支的 `assets/star-history-light.svg` 读取网格线文本的 y 坐标（20K→314.707、40K→205.58、60K→96.455，线性间距 109.125 px/20K），再按同一条折线路径的终点 y 坐标换算。属自绘图表反推，误差未知。
- **当时的限制**：
  - `web_fetch` 与 pdf MCP 的 URL 读取均被 DNS fake-IP 挡住（SSRF 保护），**无法打开任何网页原文**，star 徽章、trending 截图、QbitAI 报道、skills.sh 计数全部只能按"未核实"处理。
  - 按仓库 `AGENTS.md` 的约束，**本次未安装、未执行 Archify 任何组件**（没有跑 `npx skills add`，也没有装 DSH 插件），因此**没有任何运行时实测**：渲染耗时、真实出图效果、browser-check 在本机的表现都未验证。基准数字全部来自仓库自带结果文件。
  - 子代理提供的行号与体积我做了抽查复核（SKILL.md 实际 109 行、迁移/打包条目数、tag 日期、测试文件数）；凡与本机测量冲突的，以本机测量为准。
  - **代码行数在 2026-10-12 做过一次口径修正**：本机是 Windows PowerShell 5.1，`Get-Content` / `Measure-Object -Line` 在含非 ASCII 内容的 UTF-8 文件上会吞换行、且默认只算非空行，导致行数偏小。上表的行数已改为 .NET `ReadAllLines` 统计的**真实总行数**（CLI 单文件 6,936 行、测试 62,919 行；此前记的 6,732 / 58,484 偏小）。
- **复现方式**：
  ```powershell
  git clone https://github.com/tt-a1i/archify.git
  cd archify && git checkout 58ab0b98d2d0def8e49dced705c7b566936723ec   # 本次走读基准（main，2026-10-10）
  # 或 git checkout v3.0.1                                             # 本次锁定的稳定版（2026-09-28）
  git checkout origin/star-history                                      # star 曲线数据（用于上文估算）
  ```
  文档内的行号会随版本漂移，请以 commit hash 为准。
