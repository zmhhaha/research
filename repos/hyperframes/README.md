# HyperFrames

- **调研日期**：2026-10-12
- **上游地址**：https://github.com/heygen-com/hyperframes
- **调研版本 / commit**：`main` HEAD = `2905162d925fc87fb8242e24ed0bbf025d307ea1`（2026-10-12）；最新 tag `v0.8.149`（2026-10-11）。本文 `文件:行号` 均以该 HEAD 为基准。
- **本地副本**：`D:\github\research\.cache\hyperframes`（9262 个文件；git pack 494.42 MiB；工作树 322.3 MB）
- **许可证**：Apache-2.0（`LICENSE`）。**但第三方素材并非全是宽松许可**，见「坑与风险」第 4 条。
- **本地副本的一个已知偏差**：该仓库用 **Git LFS** 存回归基线媒体（README 称约 240 MB 视频）。本环境的沙箱禁止 git-lfs 过滤器启动（它经 `sh` 拉起，命名管道被拒），所以我是在**关闭 LFS 过滤器**的情况下检出源码的——因此工作树里有 **159 个媒体文件是 LFS 指针文本**（87 个 `.mp4`、67 个 `.png`、2 个 `.mp3`、2 个 `.webm`、1 个 `.mov`），不是真实媒体。这不影响读代码，但下述任何"体积/媒体内容"结论都不含这些真实媒体。

## 一句话结论

HeyGen 开源的 **HTML → 确定性 MP4 渲染框架**：一份 HTML（`data-*` 时序属性 + 可寻址动画）在 headless Chrome 里**逐帧 seek**、再用 FFmpeg 编码成视频。它最关键的三个立场是：**无构建步骤**（`index.html` 直接能播能预览）、**动画运行时靠适配器**（GSAP 为主，另支持 CSS/Lottie/Three.js/Anime.js/WAAPI/自定义）、以及 **Apache-2.0 且没有按次渲染费或商用门槛**——README 把最后这条直接当作与 Remotion 的核心差异列出来（`README.md:286,299`）。

配套的是**面向 agent 的一整套**：21 个 skills（10 个 core + 10 个创作工作流 + `figma`，实读计数）、非交互式 CLI、以及 registry 里 **454 条可安装项**（210 blocks + 236 components + 8 examples）。它同时也是 HeyGen 自家视频产品的渲染内核（`ADOPTERS.md:21`），所以这不是一个玩具项目。

代价是：**渲染是重活**（headless Chrome + FFmpeg，视频本身就重），**版本仍停在 0.8.x、却已经打了 508 个版本 tag**（外加 19 个里程碑 tag），并且仓库规模不小（9262 文件 / 322 MB 工作树 / 494 MiB pack）。

## 项目概况

| 项 | 值 | 来源 |
| --- | --- | --- |
| 仓库地址 | https://github.com/heygen-com/hyperframes | 已验证（git remote） |
| 许可证 | Apache-2.0（根 `LICENSE`）；`packages/cli` 标注 Apache-2.0，其余多数包未在 package.json 声明 license 字段 | 已验证 |
| 定位 | 开源渲染框架 + CLI + agent skills + 可安装区块目录 + 浏览器 Studio；HeyGen 生产在用 | 已验证（`README.md:33,330`、`ADOPTERS.md:21`） |
| 技术栈 | TypeScript monorepo（`packages/*`，15 个包）、bun 工作区、Headless Chrome（Puppeteer）+ FFmpeg、Hono、esbuild 等 | 已验证（根 `package.json`、`packages/cli/package.json`） |
| 提交数 / 贡献者 | 5483 commits / 107 人；Top1 `Miguel Ángel` 2418 次（**44%**）、Top2 `Vance Ingalls` 1089、Top3 `James Russo` 538 | 已验证（`git shortlog -sn HEAD`） |
| 最新版本 | `v0.8.149`（2026-10-11）。**tag 总数 527，但要拆开看**：其中 508 个是 `v*` 版本 tag，另 19 个是 `backup/*`、`pre-split/*`、`reference/*`、`model-assets-v1` 这类里程碑/备份 tag。`v0.8.*` 一共 151 个；`v0.8.100`→`v0.8.149`（50 个版本）只用了 **10.6 天** | 已验证 |
| 首次提交 | 2026-03-10，"chore: initialize repository" | 已验证 |
| 仓库体积 | git pack 494.42 MiB；工作树 322.3 MB（`docs/` 120.5 MB、`registry/` 101.9 MB、`packages/` 79.2 MB、`skills/` 15.8 MB）；另有 LFS 托管的约 240 MB 回归视频 | 已验证（`README.md:340`） |
| 代码规模 | 追踪文件 9262：`.ts` 3226、`.json` 1179、`.html` 1179、`.md` 847、`.mdx` 604、`.tsx` 562、`.mjs` 430；**测试 1814 个文件 / 427,900 行**（studio 695、cli 362、core 203、producer 125、engine 95）；`docs/` 下 604 个 `.mdx` | 已验证 |
| CI 规模 | **16 个 workflow 文件**；其中 `ci.yml` 一个文件就有 **30+ 个 job**（顶层条目实测 31 个：build / lint / format / typecheck / fallow 死代码 / test-reachability / test-studio / test-runtime / test-support / producer-source-tests / test-skills / skills-manifest / studio-load-smoke / studio-edit-accuracy / cli-smoke-required / filesize / semantic-pr-title …），另有 `publish.yml`、`regression.yml`、`preview-regression.yml`、`fast-video-validation.yml`、`windows-render.yml`、`player-perf.yml`、`codeql.yml` 等专项流水线 | 已验证（`.github/workflows/`） |
| 发布工具链 | **没有用 changesets**；是自研脚本：`scripts/release-prepare.ts`、`set-version.ts`（注释写明 fixed versioning）、`draft-changelog.ts`、`validate-release-channel.mjs`；`publish.yml` 强制"稳定版只能来自已合并的 `release/v*` PR"；`releases/` 下 309 篇发布说明 | 已验证 |
| npm 发布 | 仓库内的 `@hyperframes/cli` **以不带 scope 的 `hyperframes` 名发布到 npm**（`npx hyperframes …` 就是它）；14 个包版本 lockstep 在 0.8.149（`@hyperframes/sdk-playground` 滞后在 0.6.106） | 已验证（`.github/workflows/publish.yml:165`、各 package.json） |
| 生态采用者 | `ADOPTERS.md` 列 10 家：HeyGen（生产）、tldraw、TanStack、OptinMonster、reap.video、Typeframe、THU-MAIC、Mini Course Generator、PandaStudio、Skyflo | 已验证（`ADOPTERS.md:19-30`） |
| 热度（star） | 未能核实 | **未核实**（本环境无法访问 GitHub 页面） |

**发布节奏异常激进，但"527 个 tag"不等于 527 次发布**：按月统计 `v*` tag 数——3 月 16、4 月 90、5 月 92、6 月 86、7 月 68、8 月 47、9 月 81、10 月 47；提交量 3 月 218 → 7 月峰值 1461 → 10 月 476（近 30 天 1175、近 90 天 2786）。项目的实际做法是"**每次合并后打一个可安装快照**"（日均 10 个以上），再用自研脚本在版本号上做 fixed versioning——所以"版本停在 0.8.x 却有 500 多个 tag"不是自相矛盾，而是 pre-1.0 阶段的策略选择：**它明确没有承诺 API 稳定**。

## 它解决什么问题

**痛点**：做视频一直是"不可编程、不可 diff、不可回归测试"的活。Remotion 证明了"用 headless 浏览器 + FFmpeg 把网页原语变成确定性视频"这条路可行，但它的赌注是 **React 组件 + 打包器**——对人友好，对"让 agent 直接写文件"则多了一层负担。

**HyperFrames 的赌注**（`README.md:281-286,288-301`）：源码就是**纯 HTML 文件**，没有构建步骤，`index.html` 打开就能预览；动画通过**适配器**接进来（GSAP 默认，另有 CSS keyframes、Lottie、Three.js、Anime.js、WAAPI、TypeGPU、自定义）；渲染器的职责是"seek 到第 N 帧"，所以同一份输入必须产出同一段视频——这使视频可以进 CI、可以做回归测试。

**与 Remotion 的关系**（README 有一张明确对比表，`README.md:292-299`；`CREDITS.md:8-18` 承认借鉴）：

| | HyperFrames | Remotion |
| --- | --- | --- |
| 编写方式 | HTML + CSS + 可寻址动画 | React 组件 |
| 构建步骤 | 无 | 需要打包器 |
| 许可 | Apache-2.0 | 源码可得（Remotion License） |
| 分布式渲染 | 本地 + AWS Lambda | Remotion Lambda（更成熟） |

值得注意的是它们**对借鉴这件事的态度**：`CREDITS.md:8-14` 明确写出"并行帧捕获的有序异步屏障""dev server 的多主机端口探测"等具体架构想法来自研究 Remotion，同时声明代码是独立实现、与 Remotion 无关联。这种"承认思路来源"的写法在同类项目里不多见。

## 核心机制

### 仓库骨架

```
packages/
  cli/                  → hyperframes CLI（init/add/lint/check/snapshot/preview/render/publish/doctor/lambda）
  core/                 → 类型、解析、生成、linter、运行时、frame adapters
  engine/               → 可寻址的"页面→视频"捕获引擎（Puppeteer + FFmpeg）
  producer/             → 完整渲染管线（捕获 + 编码 + 音频混音）
  player/               → 可嵌入的 <hyperframes-player> web component
  shader-transitions/   → WebGL 着色器转场
  studio/ studio-server/→ 浏览器里的合成编辑器
  aws-lambda/ gcp-cloud-run/ → 分布式渲染部署面
  lint/ parsers/ sdk/   → 静态检查、解析、SDK
registry/               → blocks（50+ 可安装场景）/ components / examples
skills/                 → 21 个 agent skill
docs/                   → Mintlify 文档站（604 个 mdx）
releases/               → 309 份按版本的发布说明
```

（结构依据：仓库根 `AGENTS.md` 与 `CLAUDE.md` 的 "Project Structure" 段；各目录文件数与体积为本机实测。）

### 一份 composition 长什么样

`README.md:218-248` 给的完整示例就是契约本身：

```html
<div id="stage" data-composition-id="launch" data-start="0" data-width="1920" data-height="1080">
  <video class="clip" data-start="0" data-duration="6" data-track-index="0" src="intro.mp4" muted playsinline></video>
  <h1 id="title" class="clip" data-start="1" data-duration="4" data-track-index="1">Launch day</h1>
  <audio data-start="0" data-duration="6" data-track-index="2" data-volume="0.5" src="music.wav"></audio>
  <script src="https://cdn.jsdelivr.net/npm/gsap@3/dist/gsap.min.js"></script>
  <script>
    const tl = gsap.timeline({ paused: true });          // 必须是暂停的根时间轴
    tl.from("#title", { opacity: 0, y: 40, duration: 0.8 }, 1);
    window.__timelines = window.__timelines || {};
    window.__timelines.launch = tl;                      // 注册到约定好的全局表
  </script>
</div>
```

要点：**时间不靠播放，靠声明**（`data-start` / `data-duration` / `data-track-index`）；需要被捕获的元素要标 `class="clip"`；每个 composition 要在 `window.__timelines` 注册**一个 paused 的根时间轴**，渲染器 seek 它来取任意帧（根 `AGENTS.md` 里额外提醒：手动挂到根上的场景时间轴**不能是 paused**，否则 seek 根时它不会前进）。

契约的权威文档在 skill 里而不是文档站：`skills/hyperframes-core/references/data-attributes.md:9-17,29-39`（root 与 clip 属性表）、总纲 `packages/core/docs/core.md:180-198`。几个容易踩的点：

- root 必需 `data-composition-id` / `data-width` / `data-height`，`data-duration` 是**条件必需**——它在编译期只读一次，脚本里改它无效（`data-attributes.md:13`）。
- `data-start` 除了数字秒数，还支持 `<元素 id> + N` 的相对引用语法（`tracks-and-clips.md:87-119`）；另有 `data-media-start`、`data-volume`、`data-has-audio`、`data-fade-in/out`、`data-hidden`、`data-automation`。
- **`data-track-index` 的语义在仓库里是自相矛盾的**：skill 明确写它只是 "Studio 展示用的泳道，render 从不读它"，前后层级用 CSS `z-index` 决定（`tracks-and-clips.md:19-25`、`data-attributes.md:34`）；但 `packages/core/docs/core.md:187` 仍写着"track 序号大的渲染在前 / 同轨不能重叠"。本次**未从源码判定哪种为准**（lint 只在两个 audio 同轨重叠时报 `duplicate_audio_track`）。旧名 `data-layer` / `data-end` 已废弃（`data-attributes.md:73-80`）。
- `class="clip"` **只是约定、运行时并不读**：脚手架用 `.clip{position:absolute;inset:0}` 给出满屏盒子，Studio 把它当编辑提示；缺失时 lint 报 `timed_element_missing_clip_class`（`packages/lint/src/rules/composition.ts:897`）。注意别加在 `<video>` / `<audio>` 上。
- 媒体由**框架托管**：运行时按 `data-start` / `data-duration` 自己播放、暂停、seek、mount/unmount，**禁止脚本去调 `play()` / `pause()` / `currentTime`**（`core.md:26-38`）；每个 `<audio>` 必须有 `id`，否则会被静音并报 `media_missing_id`（`variables-and-media.md:103`）。
- 变量用 `<html data-composition-variables='[{id,type,label,default}]'>` 声明（类型 string/number/color/boolean/enum），标量会自动注入成 `--{id}` CSS 变量，实例级可用 `data-variable-values` 或渲染期 `--variables` 覆盖（`variables-and-media.md:7-56`）。
- 子合成用 `<template>` 包住子文档，宿主元素写 `div[data-composition-id][data-composition-src][data-start][data-track-index]`；**standalone root 不能包 `<template>`**，否则 lint 报 `standalone_composition_wrapped_in_template`（`hyperframes-core/SKILL.md:42`、`composition.ts:923`）。
- 还有一处**文档版本冲突**：`core.md:63,198` 说 composition 没有 `data-duration`（长度取自 timeline），而 `hyperframes-core/SKILL.md:56` 与 `minimal-composition.md:63` 说渲染长度就是 root 的 `data-duration`、timeline 超长会被截断。写长视频前值得实测确认。

### 质量工程与治理（HeyGen 内部主力项目的做法）

- **提交前就拦**：`lefthook.yml` 的 pre-commit 跑 oxlint / oxfmt / skills-manifest 新鲜度 / typecheck / fallow 死代码审计 / 大文件检查 / 被跟踪产物检查，并限制 studio 单文件不超过 600 行；commit-msg 走 commitlint。`knip.config.ts` 另做未用导出检测。
- **CI 里真的渲染视频**：`ci.yml` 的 `cli-smoke-required` 用 `ffmpeg testsrc2` 造一段输入 → 跑 `hyperframes render`（libx264）→ 断言产出的 mp4 非空，还带 `page.goto` ≤5s 的预算；`test-support` 通过 `browser-actions/setup-chrome` 注入浏览器路径；仓库自备 `install-ffmpeg-linux` / `install-ffmpeg-windows` action。**这套东西是"确定性渲染"能成立的实际证据**——不是口号，而是每次 PR 都在跑。
- **贡献是开放的**：`CONTRIBUTING.md`（269 行）写明 fork → PR 流程、`good first issue` / `help wanted` / `difficulty/*` 标签体系（配合 82 行的 `TRIAGE.md`）、每个 commit 必须签名、PR 需至少一位有写权限的 maintainer 批准；还专门有一段 **AI 贡献政策**（不需要披露 AI 参与，但作者对正确性负责、AI 写的测试必须真实、看不懂的代码会被拒）。
- **但治理是 BDFL**：`CONTRIBUTING.md:259-263` 明说"HeyGen 的核心 maintainer 有最终决定权"。`.github/CODEOWNERS` 只有 3 行、且只覆盖 `pr-captures` 相关文件，**不是全仓所有权**；仓库里没有 `GOVERNANCE.md` / `MAINTAINERS.md` / `FUNDING.yml` / dependabot 配置。
- **贡献者高度集中、且以公司内部为主**：107 位作者共 5483 次提交，前三名约占 **72%**；按提交邮箱统计，`heygen.com` 域名占 **90.2%**，外部贡献约 9.8%——**"开源"的成分真实存在，但项目实质由 HeyGen 团队驱动**。
- **AI 协作是明牌**：`869 / 5483`（15.9%）的提交带 `Co-Authored-By` trailer，且以 Claude 系列为绝对主力（Opus 5 出现 622 次、Opus 4.8 451 次、Opus 4.6 409 次…）；仓库根同时提供 92 行的 `CLAUDE.md` 与 84 行的 `AGENTS.md`。这既解释了它为什么能在 7 个月里堆出 42 万行测试代码，也意味着**代码审计与接手的方式需要按"人机混合产出"来安排**。

### 渲染管线：它自己承认的现实问题

链路本身不神秘（`README.md:250`）：**渲染器在 headless Chrome 里逐帧 seek，再把每帧交给 FFmpeg 编码**。有信息量的是 `packages/engine/src/config.ts` 这份配置——它把"视频渲染到底难在哪"写得相当直白：

- **管线是六个阶段**（`packages/producer/src/services/renderOrchestrator.ts:11-21` 声明）：compile → probe → extractVideos → audio → capture → encode → assemble。
- **seek 的真正入口在页面内**：引擎调 `window.__hf.seek(t, opts)`（`packages/engine/src/services/frameCapture.ts:2827-2842`），帧时刻统一量化到 `floor(t*fps + 1e-9)/fps` 网格（`packages/core/src/inline-scripts/parityContract.ts:48-52`）；GSAP 适配器先 `pause()` 再 `totalTime(t, suppressEvents)`，必要时重放同一时刻的步骤（`core/src/runtime/adapters/gsap.ts:11-49,140-163`）。
- **默认参数**（`packages/engine/src/config.ts:282-335`）：`fps: 30`（可选 24/60）、`quality: "standard"`、`format: jpeg`、`jpegQuality: 80`、`concurrency: "auto"`、`browserGpuMode: software`、`chunkSizeFrames: 360`、`streamingEncode: true`、`staticFrameDedup: true`（静态帧去重，`frameCapture.ts:4280-4309`）。CLI 侧的质量默认值叫 `looks`，对应 **CRF 16**（`packages/cli/src/commands/render.ts:169`）。
- **质量档位与编码参数是成对的**：`chunkEncoder.ts:50-54` 的表是 **draft = ultrafast/CRF 28、standard = medium/18、high = slow/15**；WebM 走 `vp9 + yuva420p`（透明），MOV 走 `prores4444`（`:143-158`）；`crf` 与 `bitrate` 互斥（`renderOrchestrator.ts:4363-4371`）。有 GPU 时会探测并使用硬件编码器（nvenc / videotoolbox / vaapi / qsv / amf，`engine/src/utils/gpuEncoder.ts:17-31,140-177`）。
- **并行度是按内存算出来的**：`engine/src/services/parallelCoordinator.ts:138-166` 规定每个 worker 约 **1536 MB 内存 + 640 MB 堆**、每个 worker 至少 30 帧、绝对上限 24 个 worker；`auto` 的上限是 `max(6, min(16, cpu/8))`，并按争用缩容（`:397-404`）。
- **内存是按 GB 算的**：帧的 data-URI 缓存默认上限 256 条，配置注释直接给出量级——**1080p 每帧 JPEG 约 6 MB，256 条就是约 1.5 GB；4K 每帧 PNG 约 25 MB、base64 后约 33 MB**，所以 4K 下会先触发字节预算淘汰（`config.ts:205-219`）。
- **流式编码（streaming encode）与它的超时哲学**：`ffmpegStreamingTimeout` 默认 600 s，但它是**"没有任何帧到达"的非活动超时**，每成功 `writeFrame` 一次就重置——所以它管的是"Chrome 卡死了"，不是"总渲染时间"（`config.ts:189-195`）；另有 `ffmpegEncodeTimeout` 600 s 与 `ffmpegProcessTimeout` 300 s（`config.ts:185-188`）。
- **Windows 是有特殊处理的**：存在一个 `shouldAutoDisableStreamingEncodeOnWin32Compound` 启发式，会在检测到"软件 GPU 组合"时**自动关掉流式编码**，并把这次自动决策暴露给日志/遥测以便和用户手动关闭区分（`config.ts:160-167`）。这类补丁通常意味着真实环境里踩过坑。
- **媒体抽帧有内容寻址缓存**：缓存键是 `(路径, mtime, size, mediaStart, duration, fps, 格式)`，发布是原子的（先写唯一临时目录、写 `.hf-complete` 哨兵、再改名），并发渲染最多重复劳动、不会互相破坏；还有 LRU 磁盘清理，并且**明确指出 NFS/SMB 的 mtime 精度可能只到秒、可能造成错误命中**（`config.ts:241-269`）。
- 其他：支持 **HDR** 输出（`hlg`/`pq` 传输函数 + 自动探测，`config.ts:197-201`）；入口页导航超时默认 60 s，注释提醒"视频多、字体大、几百个资源请求的重合成在冷缓存下会超过 60 s"（`config.ts:224-233`）；运行时会做 **SHA-256 校验**（`config.ts:236-237`）。

换句话说：**"确定性"不是免费的**，它靠的是逐帧捕获、缓存、超时与并发控制这一整套工程，以及"遇到 Windows 软件 GPU 就退回非流式"这类妥协。

### 确定性是怎么被"管住"的

"同输入同输出"不是口号，它有明确的约束和检查：

- **约定**（根 `AGENTS.md` "Key Conventions"）：不得使用 `Date.now()`、不得用未播种的 `Math.random()`、渲染期不得发起网络请求。
- **真正的强制点是运行时的"虚拟时间垫片"**：`packages/producer/src/services/fileServer.ts:234-385` 的 `buildVirtualTimeShim` 会**冻结 `Date.now` / `performance.now`、接管 `requestAnimationFrame` 队列，并暴露 `__HF_VIRTUAL_TIME__.seekToTime(ms)`**；注入点在渲染管线与探测阶段（`renderOrchestrator.ts:3414`、`render/stages/probeStage.ts:366`）。也就是说：即使你的代码里写了时钟调用，**它拿到的也是虚拟时间**——这比"靠约定"强得多。（分布式模式下还会用虚拟时间给 Mulberry32 播种，默认关闭，`fileServer.ts:241-284,393`。）
- **静态检查**：`packages/lint` 里 `non_deterministic_code` 规则（`severity=error`，`rules/core.ts:887-964`）覆盖 `Math.random` / `Date.now` / `new Date()` / `performance.now` / `crypto.getRandomValues` / `gsap.utils.random` 等模式；另有 `requestanimationframe_in_composition`（`rules/composition.ts:1051`）、`css_transition_used`（`core.ts:296`）、`missing_timeline_registry`（`core.ts:618`）、`root_missing_dimensions`（`core.ts:510`）等。
- **但注意默认行为**：lint 报错**并不默认阻断渲染**——只有 `--strict` / `--strict-all` 才会拦（`packages/producer/src/services/render/execute.ts:52-62` 配合 `packages/lint/src/shouldBlockRender.ts:5-12`），否则是警告后继续。真正的硬拒绝只有一种：`data-requires-webgpu` 在无 GPU 环境下（`config.ts:55-58`）。所以"想靠 lint 兜住确定性"，得自己加 `--strict`。
- **规则码是具体的、可读的**（规则实现分散在 `packages/lint/src/rules/`）：例如 `gsap_css_transform_conflict`、`media_crossorigin_breaks_preview`、`video_nested_in_timed_element`、`media_missing_id`、`gsap_animates_clip_element`、`font_family_without_font_face`、`gsap_infinite_repeat`、`clip_ends_past_root_duration`、`gsap_timeline_registered_before_async_build`、`duplicate_audio_track`、`timed_element_missing_clip_class`、`standalone_composition_wrapped_in_template`（清单见 `skills/hyperframes-core/SKILL.md:60-70` 与 `references/determinism-rules.md:20-33`）。这份清单本身就是"哪些写法会在渲染时出问题"的最好教材。
- **动画适配器共 13 个**（`packages/core/src/runtime/init.ts:4041-4065`）：frameSource、WAAPI、CSS、Anime.js、Lottie、Three.js、Mapbox、Leaflet、Google Maps、MapLibre、D3、TypeGPU、GSAP。它们的复现机制统一是"**只写时间、不播放**"（例如 CSS 适配器写 `animation.currentTime` 后立刻 `pause()`，`adapters/css.ts:99-113`），GPU 类适配器共享一个 `hf-seek` CustomEvent + 完成屏障来同步（`adapters/seek-dispatch.ts:32-128`）。
- **运行时只有诊断、没有硬拒绝**：比如根时间轴没绑定注册表时只打控制台警告 `root_timeline_unbound_registry_present`（`init.ts:4110-4128`，此时渲染会冻结在 t=0），地图类只警告 `live_map_detected`（`frameCapture.ts:2148-2159`）。
- **远端资源会被本地化**：编译期有 `localizeRemoteImageSources` 预下载远端图片（`packages/producer/src/services/htmlCompiler.ts:1506`，调用点 `:2092`）——这解释了为什么"渲染期不联网"在多数情况下成立。

### 面向 agent 的 21 个 skills

`README.md:66-127` 把 skill 分成三层，这个划分本身就是它的产品设计。实读 `skills/` 目录的计数是：**10 个 core（默认安装）+ 10 个创作工作流（按需安装）+ 1 个 `figma`**，合计 21（划分逻辑在 `packages/cli/src/utils/skillsManifest.ts:117-161`，判据是 `hyperframes` 或 `hyperframes-*` 或 `media-use` 归入 core）。

- **Router（先读）**：`/hyperframes` —— 能力地图 + 意图层（先确认 brief）+ 路由器，`SKILL.md` 154 行（`skills/hyperframes/SKILL.md:100-113` 是路由表）。
- **10 个创作工作流**：`/product-launch-video`（给个网站 URL）、`/faceless-explainer`（纯文本、无网站捕获）、`/pr-to-video`（GitHub PR → 变更讲解，走 `gh` CLI）、`/embedded-captions`（给已有口播视频加字幕）、`/talking-head-recut`（口播视频加图形包装）、`/motion-graphics`（<10s 无旁白动效）、`/music-to-video`（音乐驱动卡点）、`/slideshow`（可导航演示稿，**不渲染成视频**）、`/general-video`（兜底 + companion mode）、`/remotion-to-hyperframes`（把既有 Remotion 合成**单向迁移**成 HyperFrames HTML）。
- **9 个领域技能**（按需加载）：`hyperframes-core`（合成契约）、`-animation`、`-keyframes`、`-creative`、`media-use`（媒体"操作系统"：找/生成 BGM、SFX、图、配音，冻结成本地文件并记账）、`-cli`、`-audio`（混音、效果链、submix 总线）、`-registry`、`figma`。
- **skill 的体积差异很大**：`music-to-video` 6.08 MB（169 文件）、`embedded-captions` 3.41 MB（144）、`media-use` 1.78 MB（110）、`hyperframes-animation` 1.21 MB（122）、`hyperframes-creative` 1.18 MB（79），而 `slideshow` 只有 2 个文件。也就是说"按需加载"不只是路由设计，**也是实打实的上下文预算控制**。
- **一处 README 与仓库的出入**：README 的领域技能表里没有 `hyperframes-studio`，但 `skills/hyperframes-studio` 确实存在且属于默认安装的 core set。
- **发布清单**：根 `skills-manifest.json` 记录每个 skill 的内容哈希与文件数，CLI 用它做完整性与新鲜度比对（`skillsManifest.ts:112-115`）。
- **6 个仓库内部 skill**：`.claude/skills` 与 `.agents/skills` 下各有 `changelog-video`、`motion-doctrine`、`cut-the-curve`、`captions-overlay`、`seam-craft`、`oversized-cursor` —— 只用于开发这个仓库本身、不随市场分发；两者是**字节级镜像**（CI 有 `scripts/check-skill-mirror.mjs` 把守），靠 `metadata.internal: true` 从默认发现与 `--all` 中排除（具名 `--skill <name>` 仍可装）。

安装策略也写得很细（`README.md:56-72`）：默认只装 **core set**，创作工作流**按需安装**（router 会在进入某个工作流前自己跑一次 `skills update <workflow>`），`skills add --all` 会装全部 21 个；`npx hyperframes skills update` 从当前 `main` 装（比走 skills.sh 注册表更新，后者可能滞后数小时）。每个 skill 的结构统一为 `SKILL.md`（frontmatter 只允许 `name`/`description` 必填，`scripts/lint-skills.ts:99-104`）+ `references/` / `scripts/` / `examples/` 等子目录，按需加载靠 SKILL.md 里的路由表。

### registry、脚手架与 frame.md

- **registry 的实际规模比 README 大得多**：README 说 "50+ blocks"，而 `registry/registry.json` 里实际有 **454 条 item —— 210 个 blocks、236 个 components、8 个 examples**（目录计数与清单完全吻合）。用 `npx hyperframes add <name|tag>` 安装（`README.md:269-277`）。
- **block 的格式**是"一整份 HTML 文档 + 元数据"：例如 `registry/blocks/aurora/` = `aurora.html`（完整的 `<!doctype html>`，自带 `data-composition-id/start/duration/width/height`）+ `registry-item.json` + `lib/` 资源；元数据 schema 在 `packages/core/schemas/registry-item.json:7`（必填 `name/type/title/description/files`，block/example 必须带 `dimensions`+`duration`，component 必须不带）。
- **安装是防覆盖的**：`add` 会把文件落到项目的 `compositions/`，并写 `hyperframes.lock.json` 记录哈希，用于发现用户改动后不静默覆盖（`packages/cli/src/commands/add.ts:442-483`、`registry/installer.ts:1-90`）。`hyperframes catalog` 支持本地语义检索，配一份 **384 维向量索引**产物（`registry/catalog-artifact/README.md:3-13`）。注意 **registry 基址是 GitHub raw 上的 `main/registry`，装区块需要联网**（`registry/remote.ts:31-32`）。
- **脚手架产出**（`packages/cli/src/commands/init.ts:561-625`）：模板目录整树拷贝 + `meta.json` + `hyperframes.json`（记录 registry 与路径，并标记 `authoringSkill`）+ `package.json` + **`CLAUDE.md` 与 `AGENTS.md` 两个模板文件逐字节相同**（约定在 `CLAUDE.md:55`，实测两者 SHA-256 一致）。最小可渲染项目就是 `index.html` + 上述几个 json/md 文件，内置模板只有 `blank`，其余从远端 registry 拉。
- **`frame.md` 是"给视频用的设计系统"**：README 称之为 web 语境 `design.md` 的翻译层、一个 `DESIGN.md` 超集（`README.md:151-157`）。规范在 `skills/hyperframes-creative/references/design-spec.md:5-27`：YAML frontmatter 写 colors/typography/spacing/components，正文写语境；解析优先级是 `frame.md → design.md → DESIGN.md`（永远小写）。内置 **13 个 preset**，各自带 `FRAME.md` + showcase，由 `build-frame.mjs --preset <name>` 重映射成项目里的 `frame.md`。
- **一个容易混淆的点**：仓库根的 `DESIGN.md` **不是**视频设计系统，而是 HyperFrames 文档站（Mintlify）的品牌样式指南（`DESIGN.md:1-3`）。

### Studio 与三条渲染路径

- `packages/studio` 是浏览器里的合成编辑器（`README.md:262` 标注 "Available, evolving"），配套 `packages/studio-server`；`packages/player` 提供可嵌入的 `<hyperframes-player>` web component。
- 渲染路径有三条，且它们的开源程度不同（这是选择时的关键分界）：
  - **本地 CLI**：完全开源、免登录、免费用。
  - **AWS Lambda / GCP Cloud Run 分布式**：也是**开源自托管**。`packages/aws-lambda/README.md:6-14` 说明它把 OSS 的 `plan / renderChunk / assemble` 包成 handler，用 Step Functions（Plan → Map(N) → Assemble）+ S3 + `@sparticuz/chromium`，并提供 CDK L2 构造 `HyperframesRenderStack`；CLI 侧只是薄胶水（`packages/cli/src/commands/lambda.ts:1-11`）。分布式原语在 `packages/producer/src/services/distributed/`。**只需要你自己的云凭据，不需要 HeyGen 账号。**
  - **HeyGen 托管 `cloud render`**：是**纯远端客户端，没有本地回退**——`packages/cli/src/commands/cloud/render.ts:2-29` 记录的顺序是 `POST /v3/assets/direct-uploads` → 预签名 PUT → `POST /v3/assets/{id}/complete` → `POST /v3/hyperframes/renders` → 轮询 → 下载 `video_url`，凭据走 `packages/cli/src/cloud/auth.ts:27-31`，Base URL 由 `HEYGEN_API_URL` 决定（默认 `https://api.heygen.com`）；素材上传上限 **200 MB**（`:79`）。**必须联网 + HeyGen 账号。**

## 实测与标称数据

**本次零运行时实测**：按仓库 `AGENTS.md` 的构建要求（`bun install` + `bun run build`）需要联网拉依赖，且渲染还需要 FFmpeg 与 Chrome，本次都没有执行。所以下面只有**我实测的体积/规模数据**与**仓库自述**，没有渲染速度数据。

本机实测的规模数据：

| 项 | 数值 | 说明 |
| --- | --- | --- |
| 追踪文件 | 9262 | `git ls-tree -r` |
| 测试文件 | 1814 | 匹配 `*.test.*` / `*.spec.*` |
| 工作树 | 322.3 MB | 不含 LFS 真实媒体 |
| git pack | 494.42 MiB | `git count-objects -vH` |
| 顶层目录体积 | docs 120.5 MB / registry 101.9 MB / packages 79.2 MB / skills 15.8 MB | 逐目录统计 |
| GitHub Actions workflows | 16 个 | 见「调研方式」 |

仓库自述但本次未验证的性能/行为：确定性渲染（同输入同输出）、`preview` 即时预览、"没有按次渲染费或商用门槛"（`README.md:283,286`）。

**仓库内能找到的性能数字**（都属"文档声称"，本次未复现）：

| 数值 | 出处 |
| --- | --- |
| 1920×1080 / 300 帧，三次取中位：**25.0 s（改进后） vs 9.8 s** | `docs/guides/performance.mdx:16-18` |
| 页面侧渲染约 **6×** 提升；`drawElement` 路径约 **2×** | `docs/packages/cli.mdx:960-961` |
| 某次改动带来 **1.16×–1.24×** 提升，反向回归 **1.8×** | `docs/weekly-updates.mdx:768` |
| **真实的性能门限**：`packages/producer/tests/perf/baseline.json` 里 `parityFixtureMaxMs = 90000`、`allowedRegressionRatio = 0.15`，由 `perf-gate.ts:4-30` 判定 | 已验证（这是 CI 里会失败的阈值，不是宣传） |

## 硬件 / 软件门槛

| 项 | 门槛 | 来源 |
| --- | --- | --- |
| Node.js | **22+**（CLI 包 `engines.node >= 22`）；README 徽章同样写 `>=22` | `packages/cli/package.json`、`README.md:13,138` |
| FFmpeg | **必需，且要 ffprobe**；CLI 有 preflight 检查，可用 `HYPERFRAMES_FFMPEG_PATH` / `FFPROBE_PATH` 指定路径（`packages/cli/src/browser/preflight.ts`、`ffmpeg.ts:6`）；AWS Lambda 包还自带 `ffmpeg-static` | 已验证 |
| 浏览器 | Headless Chrome（`puppeteer` / `puppeteer-core` + `@puppeteer/browsers` 管理）；可用 `HYPERFRAMES_BROWSER_PATH` 指定 | `packages/cli/package.json`、CI 里的 `browser-actions/setup-chrome` |
| 包管理器 | 开发用 **bun**（仓库明确"不要用 pnpm/npm 做工作区操作"，也不要生成 pnpm-lock.yaml） | 根 `AGENTS.md` |
| Git LFS | 仅开发/跑回归测试需要：约 240 MB 的基线视频；只读源码可 `GIT_LFS_SKIP_SMUDGE=1` | `README.md:338-360` |
| 运行时依赖 | CLI 有真实依赖（puppeteer-core、sharp、esbuild、postcss、prettier、fontkit、hono、adm-zip 等约 19 个直接依赖）——不是零依赖工具 | `packages/cli/package.json` |
| 算力 | 视频渲染是 CPU/内存密集型（逐帧截图 + 编码）；仓库提供 `windows-render.yml`、`regression.yml`、`player-perf.yml` 等流水线说明它自己也把渲染放进 CI | `.github/workflows/` |
| 账号 | 本地渲染不需要账号；`cloud render`（HeyGen 托管）与 `lambda deploy`（AWS）需要各自的云账号 | 见「坑与风险」第 6 条 |

## 上手方式

```bash
# 1) CLI 三步（本地）
npx hyperframes init my-video
cd my-video
npx hyperframes preview      # 浏览器实时预览
npx hyperframes render       # 渲染 MP4

# 2) 装区块/组件
npx hyperframes add flash-through-white   # 着色器转场
npx hyperframes add data-chart            # 动画图表

# 3) 装 agent skills（默认 core set，工作流按需）
npx hyperframes skills update
npx skills add heygen-com/hyperframes            # 交互式选择（仅终端）
claude plugin marketplace add heygen-com/hyperframes && claude plugin install hyperframes@hyperframes
```

质量门（写合成后必须过，`AGENTS.md`）：

```bash
npx hyperframes lint     # 静态 HTML 结构检查
npx hyperframes check    # headless Chrome 真渲染门：运行时错误、布局、动效、WCAG 对比度
```

两个门的分工（源码实读）：`lint` 走静态规则（`non_deterministic_code`、`missing_timeline_registry`、`css_transition_used` 等）；`check` 是**真的开 headless Chrome 跑五段**——lint / runtime / layout / motion / contrast（`packages/cli/src/utils/checkTypes.ts:277-289`），默认取样 9 帧、容差 2、超时 3000 ms，对比度检查默认开启（`checkPipeline.ts:84-95`），能报出 `console_error` / `page_error` / `request_failed` / `http_error` / `media_proxy_failed` 以及 22 个布局问题码和 `contrast_aa_failure`（`checkBrowser.ts:367,405,499,513,1391-1414`、`checkPipeline.ts:1352`）。**默认渲染不会因为 lint 报错而中止，要拦就加 `--strict`**（`packages/producer/src/services/render/execute.ts:52-62`）。

给 agent 的一句提示词示例（`README.md:60-64`）：

> Using `/hyperframes`, create a 10-second product intro with a fade-in title, a background video, and subtle background music.

## 坑与风险

1. **0.x 却已经 508 个版本 tag，升级频率是真实成本**：14 个包 lockstep 在 `0.8.149`，`v0.8.100`→`v0.8.149` 只用了 10.6 天。tag 在这里的含义是"每次合并后的可安装快照"，所以这既是"维护极其活跃"的证据，也意味着**上游没有承诺 API 稳定、破坏性变更不受语义化版本保护**。想拿它进生产，**必须自己钉死版本并锁定 `bun.lock`**，并且升级要走一遍 `lint` + `check` + 一次真实渲染。
2. **渲染是重活，不是"随手跑一下"**：headless Chrome 逐帧 + FFmpeg 编码意味着渲染耗时/内存与分辨率、帧率、时长直接相关，而且要装 Chrome 与 FFmpeg。仓库自己都需要专门的 CI 流水线（`windows-render.yml`、`regression.yml`、`fast-video-validation.yml`、`preview-regression.yml`）来覆盖渲染，可见其工程成本。
3. **"确定性"依赖本地资产，而仓库里的示例经常不满足"渲染期零网络"**：好消息是编译期确实会**预下载远端图片并本地化**（`packages/producer/src/services/htmlCompiler.ts:1506`）；但 README 的入门示例与**大量 registry block 直接引 CDN**（`registry/blocks/*/*.html` 里普遍是 `cdn.jsdelivr.net` 的 GSAP、`fonts.googleapis.com` 的字体），而 `packages/lint/src/rules/adapters.ts:23,54` 的修复建议**就是让你加一个 CDN `<script>`**。地图类组件更是只给警告不阻断（`live_map_detected`）。结论：**脚本/字体这类远端依赖是否被自动本地化，本次未核实**；要进 CI 或离线环境，请自己把依赖固化到本地（CLI 提供了字体本地化入口 `hyperframes-localize-fonts`）并加 `--strict` 渲染。
4. **第三方许可比"Apache-2.0"复杂得多，而且我已经查到具体问题**（`CREDITS.md:26-66`）：
   - **商业字体没带许可文件**：`TT_Norms_Pro_Normal/Medium/Bold.woff2`、`tt_norms_pro_mono_regular-webfont.woff2`、`ABCSolarDisplay-Bold.woff2` 同时存在于 `.agents/skills/changelog-video/assets/fonts/`、`.claude/skills/changelog-video/assets/fonts/` 和 `packages/producer/tests/heygen-promo-preview-assets/src/` 三处，**字体目录内没有任何许可文件**，同目录的 `bgm.mp3`（159 秒）与 12 MB `bg-pattern.mp4` 也没有来源说明；它们由 commit `e96ebd74d`（2026-07-16，PR #2552）引入。TT Norms Pro（TypeType）与 ABC Solar Display（ABC Dinamo）都是**商业零售字体**，Apache-2.0 不会自动授予再分发权。
   - **GSAP 明确"不是 OSI 开源许可"**（GSAP Standard License），却被 7 个 3D 目录区块 vendor 或从 CDN 加载；
   - **`ffmpeg-static`（AWS Lambda 包）分发的 FFmpeg 二进制通常是 GPL 构建**，仓库内未找到相关许可说明；
   - `mediabunny` 是 **MPL-2.0**；`hyperframes models install parakeet` 会下载 **NVIDIA Parakeet TDT 0.6B v3（CC-BY-4.0）**，跑在 sherpa-onnx 上；
   - 还有 **6 个素材的授权来源尚未确认**（`CREDITS.md:62-66` 原文 "Origin to be confirmed with the author"）；
   - 根目录**没有 `NOTICE` 文件**，且 14 个包里只有 `packages/cli` 声明了 `license: Apache-2.0`，其余 13 个 `package.json` 缺失该字段——对 Apache-2.0 项目来说这属于卫生问题。
   值得注意的是**同一仓库里其他地方的许可处理相当严谨**：catalog 会注明 "OpenAI Sans … is not licensed here. Inter is the redistributable OFL stand-in"，registry 里的字体都配了 `*-LICENSE.txt`。所以这更像是"测试资产/skill 资产"这条线上漏了，而不是整体态度问题。**结论：项目代码是 Apache-2.0，但"拿它的目录区块或 skill 资产做商业视频"必须自己逐个核对条款。**
5. **仓库体量大且依赖 Git LFS**：9262 文件、322 MB 工作树、494 MiB pack，另有约 240 MB LFS 视频。只读源码可以 `GIT_LFS_SKIP_SMUDGE=1`，但**跑回归测试必须装 LFS**（否则基线视频是指针文件，测试无意义）。
6. **商业边界要看清（已从源码确认）**：
   - **纯本地、免登录**：CLI 的 `init/lint/check/snapshot/preview/render/doctor`、engine / producer / player / studio / studio-server / parsers / lint / core / sdk，以及自建的 `packages/aws-lambda`、`packages/gcp-cloud-run` 渲染路径。Studio 是完整的本地编辑器。
   - **需要 HeyGen 凭据**：`cloud render` 走 `POST /v3/hyperframes/renders`（`packages/cli/src/commands/cloud/render.ts:6-28`），认证默认打 `https://api.heygen.com`（OAuth 设备码，或 `HEYGEN_API_KEY`；`packages/cli/src/auth/client.ts:7,23`）；`publish` 需要登录（`packages/cli/src/commands/publish.ts:148,191`）。
   - **免费/付费的界线写得很明确**：用 OAuth 登录可吃 Web 计划的免费额度（例如 HeyGen TTS 每月 10 分钟），换成 API key 就走正常计费；**没有凭据时，语音与 BGM 退回本地 Kokoro/MusicGen，音效用内置的 21 个文件（离线、确定性、免费）**（`skills/media-use/audio/references/requirements.md:15`、`tts.md:67-68`、`sfx.md:6`）。
   - **没有发现功能阉割**：仓库里找不到 enterprise / license key / paywall / 门控代码；README 也是"本地可用 + 托管渲染可选"的表述（`README.md:33,122`）。
   - 一句话：**按次渲染不收费是真的；但"云渲染机器 + 素材生成 + 发布"不是免费的。**
7. **CLI 有遥测，而且默认开启**：`packages/cli/src/telemetry/config.ts:540-541` 的默认配置是 `telemetryEnabled: true`；可用 `hyperframes telemetry disable`、或环境变量 `HYPERFRAMES_NO_TELEMETRY=1` / `DO_NOT_TRACK=1` 关闭，`hyperframes telemetry status` 可查当前生效状态（`packages/cli/src/commands/telemetry.ts:19-28,71,110`）。它的策略里还有两种"天然关闭"的情况：开发构建、以及没有内置 telemetry key 的构建（`telemetry.ts:24-27`）。**在意隐私的部署应当显式关掉或设置 `DO_NOT_TRACK=1`。**
8. **agent 相关的一处已知陷阱**（仓库自己用 Warning 标出，`CLAUDE.md`）：在 HyperFrames 检出目录里跑 `npx skills add ... --all` 会选中 OpenClaw 目标，而它的 `skills/` 目录**就是这个仓库自己的 `skills/`**，会先删除再软链每个目的地——未提交的改动无法恢复。要在空目录或自己的项目根里跑。
9. **两个平台的可移植性**：Windows 上有专门的 `windows-render.yml`，引擎里还有"检测到 Windows 软件 GPU 组合就自动关掉流式编码"的启发式（`packages/engine/src/config.ts:160-167`），说明跨平台渲染确实有坑；本次没有验证 Windows 下的实际渲染。

## 结论

**适合谁用**：
- 需要**把视频生产纳入工程流程**的团队：版本化、可 diff、可回归测试、能在 CI 里渲染（这正是"确定性"的价值）。
- 已经在用 coding agent 做内容/营销自动化，想要"agent 写 HTML → 出 MP4"这条最短路径的（21 个 skill 就是为这个场景写的）。
- 不想引 React/打包器、希望产物就是"一个 HTML 文件"的（设计、模板、可移植性都更好）。
- 计划用 AWS Lambda 做分布式渲染、且希望许可无按次费用的（Apache-2.0，对比 Remotion 的源码可得许可）。

**不适合谁用**：
- 只是想偶尔剪个视频的人——它是框架/CLI，不是剪辑软件；纯手工剪辑体验还不如桌面工具。
- 机器上没有 Chrome/FFmpeg 且不想装、也不打算用云渲染的。
- 需要**长期稳定 API**的：0.8.x + 每月几十个 tag，接口变动是常态。
- 对第三方许可敏感且不愿逐个区块核对素材来源的商业团队（尤其用了 3D 目录区块或商业字体资产时）。
- 离线/隔离环境又不做资产本地化的（见风险 3）。

**什么情况下值得引入**：如果你已经有"批量出视频"或"agent 自动产片"的真实需求，先做一次**最小验证**：`npx hyperframes init` + 用 README 那段 20 行示例渲染一个 6 秒 1080p 视频，记录**渲染耗时、内存峰值、以及离线能否成功**（把 CDN 依赖本地化后重试）。这三个数字决定它是否适合你的流水线——许可和确定性都是加分项，但渲染成本才是这类工具的实际门槛。如果只是"给 PR 生成讲解视频"这类少量场景，它自带的 `/pr-to-video` skill 反而是最省事的入口。

## 未核实 / 存疑

- **star 数与外部热度**：本环境无法访问 GitHub 页面（DNS 被解析到 198.18.x.x 的 fake-IP），任何 star/trending/npm 下载量数字都未核实。
- **渲染性能**：仓库内未找到公开的渲染耗时基准数据（本次也未运行）；"快速渲染""确定性"等说法均来自文档。引擎配置里的默认超时（如入口页导航 60 s、流式编码 10 分钟无帧才判死）可以当作量级参考，但不是实测。
- **`ADOPTERS.md` 的 10 家采用者**：属自报（文件本身邀请 PR 添加），仓库内无法验证 tldraw / TanStack 等实际用法，仅 HeyGen 自身有生产声明。
- **依赖的真实许可**：`node_modules` 未安装，只能从 `package.json` / `bun.lock` / `CREDITS.md` 推断；`ffmpeg-static` 的 GPL 影响、以及各 npm 依赖的许可证逐包核验，本次**未做**。
- **`cloud render` / Studio 托管 / playground 的实际能力与收费**：这些依赖 HeyGen 站点，本次无法访问；只能确认仓库里存在对应入口。
- **渲染期是否真的零网络请求**：约定如此，但 registry blocks 与 README 示例引用 CDN；**未验证** CLI 是否在渲染前自动本地化这些依赖（存在 `hyperframes-localize-fonts` 入口与 `capture/assetDownloader`，但我没有走完整链路）。
- **6 个待确认来源的素材**（`CREDITS.md:62-66`）：作者自己标注 "Origin to be confirmed"，实际授权状态不明。
- **Windows 下的真实渲染表现**：本仓库有 Windows 专用 CI，但本次未运行。
- **渲染期是否真的零网络请求**：编译期确认会本地化远端**图片**（`htmlCompiler.ts:1506`），但 registry blocks / README 示例里的 **CDN 脚本与 Google Fonts 是否也被本地化，本次未核实**；地图类组件只有警告。
- **`packages/sdk` / `sdk-playground` 是否调用远端服务**：未核实。
- **Three.js / TypeGPU 等 GPU 适配器如何处理浮点不确定性**：未核实（配置里只说明页面侧 WebGL 是 f32、与字节级比对不兼容，改用 PSNR）。
- **`data-track-index` 与文档 `data-duration` 的两处文档冲突**：见上文「核心机制」，源码未判定。
- **各 package 的 npm 发布状态与 license 字段缺失**：多数 `packages/*/package.json` 没有 `license` 字段（只有 cli 写了 Apache-2.0），是否为发布配置省略未核实。
- **LFS 媒体内容**：本地副本里 87 个媒体文件是指针文本，未下载，因此任何关于示例视频内容/质量的判断都不成立。

## 调研方式

- **手段**：
  1. `git clone` 全量历史到 `.cache/hyperframes`；因为沙箱禁用命名管道导致 git-lfs 过滤器无法启动，检出时用 `GIT_CONFIG_NOSYSTEM=1` 关闭系统级 LFS 过滤器（**只影响本地副本的媒体文件，不影响代码**）。
  2. 自己读：`README.md`（364 行）、根 `AGENTS.md` / `CLAUDE.md`、`CREDITS.md`、`ADOPTERS.md`、根 `package.json`、`packages/cli/package.json`，并实测了文件数/体积/目录构成/tag 与提交分布/测试文件数/npm 包清单。
  3. 三个子代理分工走读：① 渲染引擎（packages 职责与代码量、六阶段管线、seek 机制、虚拟时间垫片、默认参数与并行度、lint/check 规则码、三条云渲染路径、13 个帧适配器）；② composition 契约与 skills 体系（data-* 属性、21 个 skill 的实读计数、脚手架、registry 实际条目数、frame.md）；③ 发布节奏、CI 与质量工程、许可与商业边界、治理与协作痕迹。
  4. 交叉核对了三组"文档 vs 实际"：registry 规模（README 说 "50+ blocks"，实际 210）、`data-track-index` 语义（skill 与 core.md 相反）、CDN 依赖与"渲染期不联网"的张力。
  5. **子代理数字我自己复核过一遍**，并纠正了两处：workflow 文件是 **16 个**（不是 17）、`CONTRIBUTING.md` 是 **269 行**（我最初用 `Measure-Object -Line` 得到的 189 行是错的，该方法在长文件上不可靠），tag 拆分为 **508 个 `v*` + 19 个其他**；凡与本机测量冲突的以本机复测为准。
- **当时的限制**：
  - `web_fetch` 与 pdf MCP 的 URL 读取均被 DNS fake-IP 挡住（SSRF 保护），**无法打开任何网页原文**（hyperframes.dev、heygen.com、npm 页面都不可达），外部信息只能来自搜索摘要并按"未核实"处理。
  - 按仓库 `AGENTS.md`，构建需要 `bun install` + `bun run build`（联网），渲染还需 FFmpeg + Chrome；本次**未安装、未构建、未渲染**，因此**没有任何运行时实测**。
  - 本地副本的 LFS 媒体是占位指针。
- **复现方式**：
  ```powershell
  # 只读源码（推荐，跳过 240 MB LFS 媒体）
  $env:GIT_LFS_SKIP_SMUDGE='1'
  git clone https://github.com/heygen-com/hyperframes.git
  cd hyperframes && git checkout 2905162d925fc87fb8242e24ed0bbf025d307ea1
  ```
  要跑测试/回归则需先装 Git LFS 再 clone（`README.md:338-354`）。本文所有行号以该 commit 为准，版本更新后会漂移。
