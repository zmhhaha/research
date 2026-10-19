# PPT Master

- **调研日期**：2026-10-20（本机 +08:00）。
- **上游地址**：[hugohe3/ppt-master](https://github.com/hugohe3/ppt-master)。
- **源码基线**：`c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7`，提交时间 2026-10-19 03:30:10 UTC；Skill 元数据版本 **6.7.2**。
- **发布基线**：获取到的最高版本 tag 为 `v6.7.2`，指向 `b113d23720260c411ee47b2b05a73615f954deeb`（2026-10-17）；HEAD 比它多 1 个赞助文档提交。未独立核验 Release 下载包。
- **许可证**：主项目 MIT；依赖、图标、网络素材另有许可。
- **验证口径**：「源码已验证」表示静态实现存在，不等于运行验收；「官方声明」仅确认官方如此描述。本次未安装依赖、未执行测试/生成、未读取 Office 二进制、未调用收费模型。

## 一句话结论

**PPT Master 是给通用 coding agent 使用的演示文稿 Skill 与 Python 工具链：AI 先规划内容与设计，再写受约束的 SVG，确定性工具将其编译成原生 DrawingML PPTX。** 它确有原生图表/表格、Master/Layout、保留型编辑等实现，不只是把整页图片塞进 PowerPoint。

值得用于“需要后续修改的 PPT 草稿”，但不是任意 SVG 转换器、任意 PPTX 无损往返器或一键完美成片产品。主要代价是 **模型能力与人工收尾、原生对象与视觉保真的取舍、PowerPoint 优先的兼容边界，以及依赖/素材许可和 provider 数据流**。

## 项目概况

| 项 | 结果 | 证据 / 口径 |
| --- | --- | --- |
| 产品形态 | Skill + Python 工具；不是 SaaS、桌面应用或 PowerPoint 替代品 | [产品定位](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/docs/zh/project-positioning.md#L13)，官方声明 |
| 技术路线 | 受约束 SVG → DrawingML；既有 deck 另有 OOXML 保留型编辑 | [技术设计](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/docs/zh/technical-design.md#L13) 及源码已验证 |
| 主许可证 | MIT | [LICENSE](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/LICENSE)，已验证 |
| Skill 版本 | 6.7.2 | [元数据](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/SKILL.md#L11)，已验证 |
| 历史 | HEAD 可达提交 2,174；最早可达提交 committer 时间为 2025-12-11 +08:00 | 完整获取历史后本地 Git 统计；不等同于仓库创建日期 |
| 作者统计 | shortlog 为 30 条身份；hugohe3 2,076、HugoHe 28、Kevin 17 | 不是 30 个独立维护者；别名未归并 |
| tags | 43 | 本次 Git fetch 获取；不等同于 GitHub Release 数 |
| Git 对象体积 | pack 134.81 MiB | 本机 `git count-objects -vH`；不是 Skill ZIP、安装大小或内存 |
| 测试文件 | 测试目录下 57 个 `test*.py` | 路径枚举，非用例数/通过数/覆盖率；未执行 |
| CI | 当前检出未发现 GitHub workflow；维护政策明确不引入 CI | [贡献政策](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/CONTRIBUTING.md#L64)，不据此排除外部私有检查 |
| star / 下载量 | 未核实 | GitHub 网页抓取受网络解析限制，未把 badge 当实时数据 |

## 它解决什么问题

很多 AI 制稿工具交付的是图片，或只有文字框可改；后续更新图表数据、调整图形、复用模板仍需重做。PPT Master 将“内容推理”和“PowerPoint 构造”拆开：模型决定叙事、风格、页面内容；脚本负责转换、合同检查和打包。

[官方入口](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/AGENTS.md) 区分三条顶层路线：

| 路线 | 目标 | 边界 |
| --- | --- | --- |
| Generate PPTX | 主题、文档、网页、数据等 → 新 deck | 重新组织并创作页面，质量受模型影响 |
| Create Template | 资料/品牌/既有演示 → Brand、Style、Layout、Deck 工作区 | 可复用结构需事先建立，不能在导出时猜测 |
| Edit Native PPTX | 修改既有 PPTX 或追加讲稿、动画、旁白 | 保留型 roundtrip，与重新设计分开 |

默认 Generate 包含策略规划、用户确认、持久规范与锁；**显式 Quick 跳过独立规划/确认，但仍保留最终质量门**。Quick 不生成默认流程的预览终稿，官方说明上下文丢失后无法恢复其规划状态。

“美化 PPTX”属于 1:1 重生成 profile，不等于原生对象保留型编辑。“图片还原 PPTX”则目前仅声明支持 Codex，始终走 Quick；不能把“任意 agent 可运行”扩展成所有 profile 都受支持。见 [路线边界](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/docs/zh/technical-design.md#L41)。

## 核心机制

### 1. 受约束 SVG 是中间语言

```text
材料转换 / 必要事实研究 → 来源归档
        ↓
默认：沟通确认 → 设计方案 → design_spec / spec_lock
Quick：当前 agent 在上下文内直接决定
        ↓
素材与数据准备 → 创作 svg_output
        ↓
静态质量门 → 图表校准 / 可选视觉审查
        ↓
DrawingML 编译与 OOXML 打包 → PPTX
```

源码 [converter.py:1864](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/drawingml/converter.py#L1864) 按元素分派转换，失败抛异常；未知可视元素直接拒绝，不是自动截图兜底。**浏览器能显示的任意 SVG，不一定是合法创作输入。**

SVG 包含可见页面内容，但不独自描述整个 package；讲稿、动画、旁白、关系和模板行为另有阶段。预览与最终 PowerPoint 播放效果也不能画等号。

### 2. “可编辑”需要分层理解

| 对象 / 路线 | 实际含义 |
| --- | --- |
| 默认图表/表格 | DrawingML 形状，逐形状可编辑；没有原生图表工作簿/表格对象模型 |
| `--native-charts-and-tables` | 显式 marker + JSON 驱动，受支持对象输出真实 Chart/Table；chart 可带嵌入工作簿 |
| structured 模板 | 按显式 Master/Layout/Placeholder 合同生成继承结构 |
| flat 页面 | 内容在 slide 本地；干净 Master + Blank Layout，不等于复用源模板结构 |
| 公式 | 官方声明生成可编辑 OMML；不能推断任意 Office Math 往返或跨客户端一致 |
| 图片 | 仍是图片对象；可编辑 PPT 不等于照片内全部内容变成矢量 |

源码已验证：[原生图表/工作簿](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/native_objects/__init__.py#L342)、[原生表格](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/native_objects/table.py#L2060)、[structured 构造](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/pptx_package/builder.py#L4082)、[flat 构造](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/pptx_package/builder.py#L5024)。公式的官方边界见 [说明](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/docs/zh/why-ppt-master.md#L33)。

原生图表不是任意 SVG 图的自动识别，需要配套 metadata。实现支持常见图表、部分 ChartEx、combo/stock；[明确拒绝 3D，暂缓 bullet/gantt/heatmap/map](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/native_objects/chart_data.py#L739)。复杂轴、标签、样式及表格合并等组合仍有限制。SmartArt 的源对象保留与用普通形状重画语义要分开。

[fallback 指纹机制](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/native_objects/fallback_hash.py#L111) 在 SVG 权威模式用 SHA-256 绑定视觉 fallback，避免改了 SVG 后被陈旧 JSON 静默覆盖；JSON 权威模式另有豁免。

### 3. Roundtrip 不是通用无损保证

实现让[未改页走 passthrough](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/pptx_package/builder.py#L7302)，在修改页基于源 XML 做 [overlay](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/pptx_package/builder.py#L1587)，区分修改、删除与未改对象。编辑若会丢失音视频或不透明原生 payload 关系，[导出会阻断](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/pptx_package/cli.py#L2993)。

源码支持官方“未改页面逐字节保留”的实现方向，但不能推广为整份 ZIP 字节恒等、任意对象/编辑都无损。本次没有实测保真率。

### 4. 质量门部分强制，视觉审查默认不执行

[导出入口](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/svg_to_pptx/pptx_package/cli.py#L3110) 对默认/output 或 Quick 且非 compatibility 导出，要求当前 SVG 指纹匹配且通过的 final 报告；否则失败。不能据此声称所有兼容/其他 source 分支都走相同门禁。

warning 非阻塞；事实、审美和真实客户端效果不能由静态通过证明。[逐页视觉审查](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/workflows/stages/visual-review.md#L7) **仅用户明确请求时触发**，用 Chromium/Playwright 渲染 SVG，不是 PowerPoint 桌面验收。

## 实测与标称数据

| 官方声明 | 出处 | 本次判断 |
| --- | --- | --- |
| Skill ZIP 约 56 MB | [安装说明](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/README_CN.md#L245) | 未下载 Release 包核对 |
| 示例为一次生成、未经精修 | [示例说明](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/README_CN.md#L81) | 作者描述，非独立可复现质量基准 |
| 20 页视觉审查约增加 100–150K 输入 token，K=5 | [审查说明](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/workflows/stages/visual-review.md#L13) | 官方估计，依上下文/模型/重试而变，非实测 |
| 免费工具，用户支付 AI/可选 provider 费用 | [产品承诺](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/docs/zh/project-positioning.md#L61) | 没有额外点数层，不代表零成本 |

本次仅静态核查与 Git 统计；未运行 57 个测试文件，未测生成耗时、token、成功率、内容正确率或 PowerPoint 渲染。未取得可复核第三方基准。

[贡献政策](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/CONTRIBUTING.md#L64) 明确 CI 等不在项目范围；单测可选，要求真实项目本地验证。存在测试不代表每次提交自动执行，本次不能声称 CI 全绿。

## 硬件 / 软件门槛

- **Python 3.10+ 与 agent 宿主**：需读写文件、执行命令、多轮工作的能力；不是普通网页聊天粘贴提示词即可。官方提供 Windows/macOS/Linux 安装路径，本次未验证。
- **模型**：主要内容和 SVG 由 agent 模型生成，工具不附赠模型。没有必须购买 GPU 的依据；若运行本地模型另算硬件。最低内存、页数上限未测。
- **转换依赖**：python-pptx、XlsxWriter、skia-pathops、uharfbuzz、Pillow 等；原生扩展的平台 wheel 需核实。
- **扩展能力**：PDF 用 PyMuPDF、DOCX 用 mammoth、XLSX 用 openpyxl；还包含网页、Google 生图、Edge TTS、Flask 等。
- **外部程序**：少见文档格式可能需要 Pandoc；可选视觉审查需 Playwright + Chromium，不全由一条 pip 命令安装。
- **验收**：Microsoft PowerPoint 是主目标。生成不必依赖 PowerPoint 自动化，但图表、模板、公式、动画应在目标版本验证；软件许可另算。

依据：[环境说明](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/README_CN.md#L162)、[依赖列表](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/requirements.txt)。虽然清单称部分依赖可选，根 requirements 递归包含整份；按推荐命令全量安装就包含它们。

## 上手方式

以下为后续试用建议，本次未执行：

1. 用独立 Python 虚拟环境，固定 v6.7.2 或本次 commit，避免污染现有环境。
2. 安装完整官方 Skill 包，让 agent 读取入口；不能只复制一段提示词而遗漏工具/模板/引用文件。
3. 用公开短 Markdown 生成 3–5 页，记录模型版本、时长、token/费用及人工修复量。
4. 比较默认形状版和 `--native-charts-and-tables` 数据驱动版，确认需要的可编辑层次。
5. 显式请求视觉审查；最终用目标 PowerPoint 打开检查中文字体、换行、公式、图表数据、母版、动画。
6. 已有模板另测 Edit Native，核对未改页、修改对象、媒体关系，不用新建成功代替往返验证。

[官方安装说明](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/README_CN.md#L224) 提供 clone、ZIP、marketplace。当前 DSH 提供 agent 工具能力，但本次未安装注册、未跑生成验收，不标为已兼容。

## 坑与风险

### 1. PowerPoint 优先，跨软件一致不是保证

部分宣传强调形状路线的跨端稳定，但[贡献政策](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/CONTRIBUTING.md#L71) 只以 PowerPoint 为目标，其他渲染器独有差异不作为修复对象。WPS/Keynote/LibreOffice/Google Slides 能打开不代表渲染与编辑一致，尤其图表、结构、公式和动画。

### 2. 本地生成不是端到端离线

[完整定位](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/docs/zh/project-positioning.md#L62) 已承认模型、搜索、生图、语音 provider 可接收输入。源码确认：

- [生图后端](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/image_backends/backend_openai.py#L443) 发送 prompt，编辑模式上传参考图。
- [Openverse](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/image_sources/provider_openverse.py#L124) 发送搜索 query。
- [MiniMax 语音](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/tts_backends/backend_minimax.py#L155)、[Edge TTS](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/tts_backends/backend_edge.py#L110) 涉及旁白文本发送。

配置允许第三方中转，中转是额外接收方。赞助推荐不构成性能或保密性独立证据；商密材料应按启用的数据流评估，不能只看文件留本机。

### 3. MIT 不覆盖所有依赖与素材

[依赖说明](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/requirements.txt#L37) 明确 **PyMuPDF 为 AGPL-3.0，另有商业许可**。打包再分发需单独评估，不统一标成 MIT；这不等于断言生成 PPT 自动全部变成 AGPL。

[图标声明](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/templates/icons/THIRD_PARTY_NOTICES.md) 区分 Tabler/Phosphor MIT、CHUNK CC BY 4.0、Simple Icons 项目 CC0 及品牌商标限制。网络候选还包含 CC BY/CC BY-SA，署名和改编义务需核对。

[直接 URL 图片](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/image_search.py#L1831) 标为 `unverified — direct URL`，其中 `attribution_required=False` 不等于已获免署名授权。提供商许可元数据不是法律权属核验。

### 4. 归属完整性检查影响裁剪/重打包

[attribution_guard](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/attribution_guard.py#L28) 检查固定版权/许可/仓库 metadata、赞助文件存在、LICENSE 哈希以及入口 guard 调用；失败退出码 78。它不是作者签名或完整代码安全认证。

不能随意删文件后期待原工具照常运行。**MIT 文本的法律条件与当前实现的运行检查应分开理解**，本次不作法律裁决。Skill 也明确赞助材料只在用户要求相关推荐时读取，不等于每次生成必须展示广告。

### 5. 更新可能改变当前 Python 环境

[更新脚本](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/skills/ppt-master/scripts/update_repo.py#L174) 保护已跟踪本地修改、执行 `git pull --ff-only`；依赖树摘要变化会调用当前 Python 的 `pip install -r`，除非 `--skip-pip`。更新可能联网安装，不只是拉代码；pip 失败时代码可能已更新。

依赖主要使用 `>=` 下限，没有精确版本/哈希锁定，安装可复现性有限。

### 6. 维护集中与安全范围

hugohe3 身份占 2,076/2,174（约 95.5%）可达提交，显示维护集中，不独自证明真实 bus factor。无通用 CI 的维护策略也需计入集成成本。

[安全政策](https://github.com/hugohe3/ppt-master/blob/c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7/SECURITY.md#L29) 记载 live preview SVG sanitizer、图标属性注入等修复，说明不可信内容与预览有实际安全面；只为最新版本提供安全更新。本次未复现漏洞，不声称无漏洞或存在未修复漏洞。

## 结论

**适合**：已有 agent 习惯，希望从可上传的文档/数据生成可继续修改的草稿，重视原生对象与模板结构，愿意在 PowerPoint 收尾的个人或小团队。

**不适合直接承诺**：无人审阅的一次完美交付、零配置浏览器产品、任意 SVG/PPTX 无损转换、调用云服务却要求严格离线、所有客户端视觉/行为一致。

对 DSH 可借鉴/接入的是“宿主负责 agent，Skill 负责专业流程，Python 负责确定性构造”的分工；真正引入前，应分别测试短材料生成、带源数据图表、已有模板编辑，记录质量、费用和保留结果。

## 未核实 / 存疑

1. 实时 star/下载、Release 内容、独立用户体验与榜单成绩。
2. 不同模型/宿主，尤其 DSH 下全流程兼容性与质量。
3. 指定 PowerPoint 版本的渲染、公式、数据编辑、模板及 roundtrip 保真率。
4. 耗时、费用、重试、人工精修量、最低内存及页数上限。
5. 原生 Python 依赖在本机安装兼容性。
6. 完整安全审计、素材权属与 provider 留存政策。

## 调研方式与复现

- Git clone 后补全历史与 tags；只读源码、文档、许可、目录统计。
- 两个独立只读核查分别覆盖原生转换/保留机制，以及质量门/数据流/供应链。
- 网页获取报非公网 IP 解析限制；Git 传输可用，没有绕过网页抓取保护，外部口碑留待核实。
- 未执行安装、更新、guard、转换、预览、模型或 Office 命令；目标是调研，不执行 Skill 制作流程。
- 缓存源码不提交；证据链接固定到调研 commit。

```bash
git clone https://github.com/hugohe3/ppt-master.git
git -C ppt-master checkout c350bc31e6d43b7bcf5234183d650ba8f9f4c9d7
git -C ppt-master rev-list --count HEAD
git -C ppt-master shortlog -sn HEAD
git -C ppt-master tag
git -C ppt-master rev-list --count v6.7.2..HEAD
git -C ppt-master count-objects -vH
```
