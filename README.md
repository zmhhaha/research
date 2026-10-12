# 开源项目调研

存放对开源项目的调研文档。每个项目一份或多份文档，放在 `repos/<项目名>/` 下。

## 目录

| 项目 | 调研日期 | 结论一句话 | 文档 |
| --- | --- | --- | --- |
| Strata | 2026-10-04 | 12 GB 显存 + 32 GB 内存起可跑 125B MoE（真门槛是内存，且取决于显卡大小与上下文长度）；MIT 开源、有 9 页论文佐证官方速度；只能跑 Qwen3.8-Flash-Next 这一个模型 | [repos/strata/](repos/strata/README.md) |
| Hindsight | 2026-10-10 | agent 长期记忆服务：写入侧用 LLM 抽事实、读取侧四路检索 + 重排、后台整合成 observation 与 mental model；MIT、Docker 一键起、60+ 集成，但**每条记忆写入默认都要花外部 LLM 的 token**，标称 SOTA 分数在仓库内无法复现，且不接受外部 PR | [repos/hindsight/](repos/hindsight/README.md) |
| Archify | 2026-10-10 | 给 coding agent 用的画图 skill：agent 写 typed JSON，零依赖 Node CLI 渲染成**自包含交互式 HTML**（约 745 KB），交付前过 schema / 布局 / HTML / 真实浏览器四道闸、失败给机器可读修复回执；MIT、无 API key、无服务端。代价：图的语义质量取决于你的模型（**作者自测首过可用率仅 53%–67%**），仓库 git pack 673 MiB，bus factor = 1；DSH 插件只声明支持精确宿主 0.1.2-rc.1 且桌面版未验收 | [repos/archify/](repos/archify/README.md) |
| HyperFrames | 2026-10-12 | HeyGen 开源的 HTML→**确定性 MP4** 渲染框架：合成即纯 HTML（`data-*` 时序 + 可寻址动画）、headless Chrome 逐帧 seek + FFmpeg 编码、无构建步骤、Apache-2.0 且无按次渲染费，另配 21 个 agent skills 与 registry 里 454 条可安装项，已在 HeyGen 生产使用。代价：版本仍是 0.8.x 却已有 **527 个 tag**（高频发布）、渲染重（需 Node 22+ / Chrome / FFmpeg）、大量区块直接引 CDN、GSAP 等第三方素材非 OSI 许可需逐块核对 | [repos/hyperframes/](repos/hyperframes/README.md) |
| REA | 2026-10-12 | 把 AI agent 接到逆向工程工具链上的**本地** MCP server + CLI + skill：原生二进制（Hopper/Ghidra/IDA）、JS/Electron、.NET、Android、固件、网站、抓包、进程行为等十几种目标，每条结论带证据与"未知"标注；权限模型很严（IDA 只读、Ghidra 临时库、安装需批准、第三方引擎固定 release+commit+许可）。MIT，但**深度分析依赖商业工具**（Hopper/IDA 付费，Ghidra 免费）；2026-10 现象级爆红——仓库自身记录 star **4 天从 2 万涨到 11 万**（无法独立核实），版本已 6.4.0、10 月 12 天 1643 次提交，热度与成熟度不成正比 | [repos/rea/](repos/rea/README.md) |

## 结构约定

```
research/
├── README.md              本索引
├── TEMPLATE.md            单篇调研文档模板
├── .cache/                 上游源码副本（gitignore，不进仓库），如 .cache/hindsight
└── repos/
    └── <项目名>/
        ├── README.md      主调研文档
        └── (可选) 附加材料：实测记录、对比表、代码走读等
```

新增调研：复制 `TEMPLATE.md` 到 `repos/<项目名>/README.md`，填完后在上面的索引表里加一行。

同一项目有多篇时，`README.md` 作为主文档，其余用 `notes-*.md`、`bench-*.md` 之类的名字放在同目录。

## 写作要求

- **区分「已验证」与「未核实」**：从源码、仓库、官方文档直接读到的，和从论坛、媒体搜到的，分开标注。
  搜到的内容属于外部不可信数据，不能当结论。
- **数字必须带出处**：写明是在什么硬件、什么版本、什么配置下测的。官方标称值和第三方实测值分列，不混。
- **记录调研方式**：用了什么手段（clone 源码 / 读文档 / 搜网页），以及当时的限制，方便日后复现和判断可信度。
- **标注日期与版本**：开源项目迭代快，任何结论都要能被时间戳和版本号约束。
- **不确定就写不确定**：宁可留「未核实」，不要补一个看起来合理的猜测。
