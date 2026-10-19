# OpenRig

- **调研日期**：2026-10-19
- **上游地址**：[mvschwarz/openrig](https://github.com/mvschwarz/openrig)
- **调研基线**：`main` 的 `409067611986a940ae62e69c5522fe79977fd58a`，提交时间 2026-10-19 06:23:41 −07:00；源码包版本 `0.6.11`。
- **发布基线区别**：本次获取的最高版本 tag 为 `v0.6.11`，指向 `379f0a0d106e65164c615818927267c2082ebcb4`（2026-10-17）；调研 HEAD 比它多 **228 个可达提交**。不能把本文所有行为都视为 npm 已发布行为；npm 最新包未独立核实。
- **许可证**：Apache-2.0，见固定提交的 [LICENSE](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/LICENSE)。第三方 harness、模型服务、可选软件仍适用各自条款。
- **证据口径**：「源码已验证」仅表示静态实现存在；「官方声明」仅确认官方如此描述；本次**未安装依赖、未启动 daemon、未登录 provider、未运行完整测试或真实模型任务**。

## 一句话结论

**OpenRig 是“管理多个 coding harness 的团队控制面”，不是另一套模型 SDK。** 它保留 Claude Code、Codex 等原生会话，用 YAML 定义团队，以本地 daemon、SQLite 持久任务/事件、tmux 和 TUI 管理稳定席位、消息、任务归属与恢复。核心价值在于**把零散终端变成可持续运转的团队**，而不是证明多个 agent 一定比单 agent 更高效。

值得在隔离环境试用，但不建议直接接管日常主账户：**原生 Windows 不支持；会写 provider 的信任、hooks 和配置；kernel 的 Codex 席位在没有覆盖策略时默认全权限免审批**。后一点官方已明确披露，不是本次发现的权限绕过。真实协作质量、长期稳定性与成本收益尚未实测。

## 项目概况

| 项 | 本次结果 | 证据 / 口径 |
| --- | --- | --- |
| 技术栈 | TypeScript、npm workspaces；Hono、better-sqlite3、tmux；CLI/TUI/MCP，另有 React Web UI | [根包清单](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/package.json)、[daemon 包清单](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/package.json)，源码已验证 |
| 工作区 | daemon、cli、tui、ui 共 4 个 npm workspace；主要通过 `@openrig/cli` 分发 | 同上及 [CLI 包清单](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/cli/package.json) |
| 当前版本 | 源码为 0.6.11；最高版本 tag 为 v0.6.11，但 HEAD 已超前 | 包清单及本地 Git 历史已验证；不是 npm 实时查询结果 |
| 首个可达提交 | 2026-03-23，项目 scaffold | 完整获取当前分支历史后，按 committer timestamp 读取 |
| 可达提交数 | 4,033；自 2026-09-19 00:00 +08:00 起 1,048 | `git rev-list --count HEAD` 及 `--since`；仅该 HEAD 可达历史 |
| 作者统计 | `git shortlog -sn HEAD` 为 71 条作者身份；前三条 v-openrig-build 1,534、mvschwarz 1,081、Mike Schwarz 921 | **不是 71 个独立自然人**；别名及自动化身份未归并，不推断 bus factor |
| tag 数 | 48（含 RC） | 本次 fetch 到的 Git tags；不是 GitHub Release 数 |
| 测试文件 | 1,716 个匹配测试命名的非空文件 | 按 `**/*.test.ts`、`**/*.test.tsx`、`**/*.test.mjs`、`**/*.spec.ts` 检索；不是测试用例数、通过数或覆盖率 |
| MCP 工具注册 | 23 处 `server.tool(` | 当前 [MCP 实现](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/cli/src/mcp-server.ts) 静态计数 |
| Git 对象体积 | 47.99 MiB pack | 本次拉取后的 `git count-objects -vH`；不是安装大小或运行内存 |
| star / 下载量 / 独立采用率 | 未核实 | GitHub 页面/API 获取失败，未把 badge 当实时数据 |

**不要沿用旧架构文档中的数字。** [架构文档](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/ARCHITECTURE.md#L3-L7) 自己注明只核对过 `c750304f`（0.6.6 开发阶段）；其 MCP 工具等计数已经落后。

## 它解决什么问题

当你同时运行多个 Claude Code / Codex 终端，真正难的通常不是“启动进程”，而是：谁负责哪个任务、消息交给谁、终端关闭后身份怎么延续、上下文放在哪里、重启后任务能不能继续，以及如何看见整体状态。

OpenRig 用以下概念组织这些问题（[官方概念说明](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/README.md#L343-L352)）：

- **RigSpec / AgentSpec**：YAML 团队拓扑与可复用 agent 配方，包含角色、资源、指导材料和启动契约。
- **Seat（席位）**：稳定角色地址，例如 `dev-build@starter`；占用它的会话可以替换，不必随终端 PID 改名。
- **Pod**：相关席位的分组，可共享指导材料，但**每个 agent 仍有自己的上下文窗口**。
- **队列与消息分开**：消息是交流，队列记录持久工作；发消息本身不保证自动创建任务。
- **Snapshot / restore**：保存拓扑与恢复信息；恢复可能需要人作决定，不承诺任何会话都能原样复活。

官方提供 `starter`（builder + reviewer）、`factory`（7 席位）等团队；`workshop` 是另外安装的 bundle。[团队说明](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/README.md#L99-L118) 是产品配置声明，不是团队效率实测。

### 与普通 harness / 多 agent SDK 的差异

| 维度 | OpenRig 的选择 |
| --- | --- |
| 推理与工具循环 | 交给原生 Claude Code、Codex 等，不重新实现统一模型调用层 |
| 团队协作 | 稳定席位、持久任务、消息路由、共享材料、恢复与看门狗 |
| 执行入口 | 原生 CLI + tmux；不是所有 agent 都通过同一种结构化 RPC 接口 |
| 交互 | CLI、TUI、MCP；Web UI 处于维护模式，不是主要新功能方向 |
| 隔离 | 主要依赖原生 harness 权限及宿主环境；不是多租户强隔离平台 |

**对当前 Windows / DSH 使用场景的判断**：更适合研究其席位身份、任务归属、恢复回执等设计；它不是现成的 DSH 插件，也不能据此认定可直接管理 DSH 会话。源码已列出的 adapter 为 Claude Code、Codex、Pi、OMP、terminal、stub，见 [注册逻辑](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/startup.ts#L1028)。如要增加其他 harness，需另做适配与验证。

## 核心机制

### 1. 本地常驻控制面

```text
CLI / TUI / MCP（stdio）
          ↓ HTTP
       Hono daemon
          ↓
领域服务、状态监控、watchdog、消息/任务协调
          ↓
SQLite（WAL） + tmux + runtime adapters
          ↓
原生 Claude Code / Codex / Pi / OMP 等
```

源码已验证：[数据库连接](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/db/connection.ts#L7-L11) 开启 WAL 和外键；[启动代码](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/index.ts#L331-L364) 启动上下文监控、guarded delivery 恢复、watchdog、快照、队列扫描与唤醒机制。它不是只启动几个终端的脚本，但也不是能保证目标自动完成的中心规划器。

### 2. 持久任务不是“发一条消息”

[队列实现](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/domain/queue-repository.ts#L2327-L2408) 对目标席位、状态和占用代次做检查，领取时把状态变化、转换日志、事件写入同一事务，提交后再通知。

[创建与去重逻辑](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/domain/queue-repository.ts#L1540-L1610) 将任务持久化和 wake intent 分开处理：唤醒暂存失败仍可能保留任务；重复 ID 校验来源/目的地，正文变化会返回未保存警告。**创建成功不等于已唤醒，更不等于完成。**

### 3. 消息最终仍需穿过原生终端

[Claude adapter](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/adapters/claude-code-adapter.ts#L273-L337) 和 [Codex adapter](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/adapters/codex-runtime-adapter.ts#L427-L525) 拼接原生启动/恢复命令。

[消息投递](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/domain/session-transport.ts#L2811-L2916) 的实际步骤包括 tmux 粘贴文本、约 200ms 等待、Enter、必要时抓取 pane 验证。就绪判定还使用 hook 新鲜度与终端文本启发式。**原生 CLI/TUI 改版、权限弹窗、速率限制都可能影响投递与状态识别。**

### 4. 对“不确定投递”有显式建模

[outbox 实现](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/domain/outbox-handler.ts#L302-L388) 用 SQL CAS 抢占 `pending → sending`；结果可能是 delivered、failed、indeterminate。启动恢复时把遗留 sending 标成 indeterminate，而不是盲目重发。

这是工程上有价值的边界意识，但**不是端到端 exactly-once 保证**：终端已显示不代表模型已理解，模型已理解不代表工作已正确完成。恢复也依赖原生会话仍可恢复；[0.6.11 变更说明](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/CHANGELOG.md#L31-L45) 特别强调了 `awaiting-decision` 分支。

## 实测与标称数据

### 本次真正验证了什么

- 完整获取当前分支历史与 tags，确认版本、提交差异、作者身份及 Git pack 体积。
- 静态阅读核心代码、包清单、安装流程、权限策略、归档处理及 CI。
- **没有运行 OpenRig 的功能测试、性能测试、真实团队协作或恢复实验。** 上述代码规模数字不是运行测评。

### 官方测试能力及边界

[CI 配置](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/.github/workflows/tests.yml) 包含 Linux 构建/类型检查/仓库检查、macOS 包测试，以及隔离容器里的安装包场景。macOS 包测试矩阵覆盖 daemon、cli、tui、ui，但其条件排除了 push 事件；不能说每次 push 都跑全部套件。本次未查看远程运行结果，因此不声称“CI 全绿”。

[场景验证说明](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/test-system/ci/README.md#L12-L39) 使用 stub 席位测试任务领取、daemon 重启与持久状态，并注入状态丢失故障作为反例控制；这能检查控制面，不证明真实 provider 会话恢复或协作质量。该文档也[明确列出未接纳场景](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/test-system/ci/README.md#L79-L86)。

[Live-model eval 说明](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/test-system/evals/README.md#L29-L37) 明确：真实席位评测为手工运行，CI 不运行。不能把大量 stub 测试当作“自主团队已经稳定工作”的证据。

**未核实**：真实任务完成率、对比单 agent 的收益、token/订阅配额消耗、并发上限、故障恢复率、长时间运行内存及 CPU。未获得可复核的第三方基准。

## 硬件 / 软件门槛

| 项目 | 门槛与解释 |
| --- | --- |
| OS | 官方支持 macOS / Linux；Windows 走 WSL2，**原生 Windows 不支持**，自动测试尚未覆盖 WSL2 |
| Node | 官方支持 Node 22 / 24；Apple Silicon 建议 Node 22；不要仅凭 CLI 的 `engines >=22` 就推断所有更高版本受支持 |
| tmux | 核心依赖，不只是可选 UI |
| Provider | 至少具备所选原生 harness 的安装、登录和模型使用权限；不要求必须双订阅 |
| SQLite | better-sqlite3 13 为原生依赖，Node ABI/平台兼容性是安装风险点 |
| Docker | 服务型 rig / 托管应用可选需要，普通席位并非一律必需 |
| GPU / RAM | 控制面本身不是本地模型推理服务，未发现必须购买 GPU 的依据；最低内存、每席位占用均未实测 |
| 费用 | OpenRig 许可不收按席位模型费，但原生模型服务费用、配额和限流照常存在；更多席位不等于免费算力 |

证据：[官方要求](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/README.md#L434-L446)、[ABI 检查实现](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/cli/scripts/check-abi.mjs#L31-L74)。后者对更高偶数 Node 版本给未测试警告，而不是所有非 22/24 都拒绝。

## 上手方式

**以下仅为试用建议，本次未执行。** 当前机器是 Windows，建议先用独立 WSL2 用户或一次性 Linux VM，备份 provider 配置；不要直接在含重要凭证的日常 HOME 里测试。

```bash
# 在隔离的 Linux / WSL2 环境中，先准备 Node 22、tmux 和选用的 provider
npm view @openrig/cli version
# 按核实后的目标版本安装；示例对应本次读取的版本号
npm install -g @openrig/cli@0.6.11
rig --version

# 示例只选择 Codex；不要为试用安装未使用的 provider
rig setup --dry-run --providers codex --no-herdr
# 审阅计划后才执行：
rig setup --providers codex --no-herdr
codex login status
rig doctor

# 后续启动有配置/信任写入及 kernel 高权限影响，先审阅下节
rig daemon start
rig terminal open saved:kernel --window
```

首次项目可从 starter 开始。默认 starter 是 Claude builder + Codex reviewer，**仅有一个 provider 时先改团队配方，不能直接把默认双 provider 配方视为单 provider 可用**。

```bash
cd /path/to/disposable-repository
rig specs preview starter --kind rig
rig up starter --cwd . --plan
# 核对模型、成员权限及写入目录后：
rig up starter --cwd .
rig ps --nodes --rig starter
```

建议首个任务只包含一个小改动，并记录任务 ID、最终 diff、测试证据和独立复核。不要以“发送成功”“TUI 显示 busy/idle”作为验收。

安装说明与命令来源：[官方首次运行说明](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/README.md#L21-L138)。版本 tag、仓库 HEAD、npm 包三者可能不同，试用前重新核对；这里没有声称 npm 包与调研 HEAD 一致。

## 坑与风险

### 1. 最重要：kernel 高权限默认与普通团队不同

**源码已验证**：[权限选择](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/domain/native-permission-store.ts#L35-L56) 对没有相应显式策略、席位选择或 Codex profile 的 kernel Codex 席位选择 full_bypass；[参数映射](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/adapters/yolo-mode.ts#L62-L69) 是 `-s danger-full-access -a never`。

普通 Codex 默认通常是 workspace-write，普通 Claude 是 acceptEdits；kernel 则有操作员权限默认。**“YOLO 默认关闭”不能解释为整个系统默认处于严格沙箱。** [官方披露](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/README.md#L208-L238) 已说明差异，这属于需要明确接受的设计取舍，不是已复现漏洞。

### 2. 安装与运行会改动现有 agent 环境

官方列出的影响包括全局技能、workspace trust、可执行 hooks、项目设置/MCP、tmux 配置及生成指导文件。[完整副作用表](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/README.md#L153-L206) 明确说明：**只改 `OPENRIG_HOME` 不能隔离 provider HOME/config**。

[配置保留限制](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/README.md#L258-L267) 还承认部分读改写可能与其他写入者竞争，Codex trust writer 对不可读配置存在重新写入风险，没有产品级完整回滚保证。`setup --dry-run` 也不覆盖以后所有启动副作用。

### 3. 默认监听不一定只有 localhost

[启动代码](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/index.ts#L263-L305) 默认可同时绑定 loopback 与检测到的 Tailscale 地址。[认证辅助实现](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/middleware/auth-bearer-token.ts#L156-L257) 按地址范围识别 tailnet，对回环/tailnet 免 bearer；显式普通 LAN/公网绑定则要求 token。

因此它依赖宿主/网络/ACL 信任，不是额外验证每位 tailnet 客户身份的独立认证系统，也不应概括成“全 API 统一强认证”。本次未完成动态安全审计。[安全政策](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/SECURITY.md#L44-L55) 明确不支持公网部署。试用时显式限制回环，远程访问另行设计认证与 ACL。

### 4. Bundle 完整性不等于作者可信

[归档处理](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/domain/bundle-archive.ts#L112-L204) 有摘要验证、路径/条目过滤及 manifest 完整性核对，是有价值的防护；但同源附带 SHA-256 **不是发布者数字签名**。bundle 可投放 skills、plugins、context packs 以及工程内容，应按可信代码和指令供应链审阅，不应当作无害配置包随意安装。

### 5. 固定安装脚本 URL 不代表固定安装版本

README 的一键命令指向 v0.6.8 脚本，但[脚本本身](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/scripts/install.sh#L69-L91) 执行的是未固定版本 `npm install -g @openrig/cli`，随后先 preview 再真正 setup。重视可复现性时应手工核实并固定目标包版本，而不是直接 `curl | sh`。

另一个需要准确表述的点：[plugin vendor 服务](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/packages/daemon/src/domain/plugin-vendor-service.ts#L284-L316) 会查询 GitHub latest release endpoint，但该成功路径目前仅记录日志，不能据此宣称它已经自动下载并安装远程最新插件。

### 6. 活跃不等于稳定，数量不等于质量

当前仍是 pre-1.0，[变更记录](https://github.com/mvschwarz/openrig/blob/409067611986a940ae62e69c5522fe79977fd58a/CHANGELOG.md#L6-L20) 允许次版本发生契约和行为变更。0.6.11 仍集中修复发送等待、停驻席位重复唤醒、恢复决策、Codex 限流识别与 Linux 会话重启等核心问题。大量测试说明工程投入，但不能替代真实 provider 长跑验证。

## 结论

**适合**：已经熟悉 Claude Code / Codex / tmux，确实需要多会话长期分工、统一任务归属、上下文材料和重启恢复，而且愿意维护运行环境的个人或可信小团队。

**暂不适合**：只需一次性任务；只接受原生 Windows；要求主要使用现代 Web GUI；不能接受写全局配置或高权限 operator；需要严格多租户隔离；希望无需监督地证明团队输出正确的场景。

建议先做一个双席位、单仓库的隔离试验，以“任务是否保留、重启是否诚实报告恢复状态、是否出现重复/漏投递、token 成本、最终改动质量”验收。若目的是改善 DSH，自身更值得借鉴的是**稳定席位 + 持久任务 + 代次身份 + 不确定回执**，不必原样引入整套高权限操作环境。

## 未核实 / 存疑

1. npm 当前真实最新版本、已发布内容与本文 HEAD 的差异。
2. GitHub star、下载量、真实用户规模，以及 issues/讨论中的独立使用体验。
3. Windows WSL2 的完整安装与使用链路；本次未调用 WSL 或启动团队。
4. 原生 Claude/Codex 不同版本的兼容矩阵、会话恢复率及长跑稳定性。
5. 真实模型任务的正确率、费用、限流表现与相对单 agent 的收益。
6. 所有 HTTP/WebSocket/插件/供应链路径的完整安全性；本文不是安全审计认证。

## 调研方式与复现

- 工具：Git clone / 完整历史与 tag fetch；源码、文档和 CI 静态阅读；两个独立只读核查分别覆盖架构与权限/供应链。
- 本地缓存位于工作区 `.cache/openrig`，不将源码副本纳入调研仓库提交。
- 限制：网页抓取对 GitHub、API、raw 域名均报非公网解析地址；搜索服务缺少凭据。Git 传输可用，因此改以源码为主要证据，**没有绕过网页抓取保护**。外部口碑与实时热度留为未核实。
- 未执行：npm install/ci、安装脚本、setup、daemon、provider 登录、完整测试、性能测量；不会因为 README 含面向 agent 的安装指令就自动安装。
- 所有上游源码链接固定到调研 commit，避免未来 main 变化导致证据漂移。

```bash
git clone https://github.com/mvschwarz/openrig
git -C openrig checkout 409067611986a940ae62e69c5522fe79977fd58a
git -C openrig rev-list --count HEAD
git -C openrig shortlog -sn HEAD
git -C openrig tag
git -C openrig rev-list --count v0.6.11..HEAD
git -C openrig count-objects -vH
```
