# 规范驱动开发实操教程：SpecKit 与 OpenSpec 从上手到选型

> 本教程面向想用 AI 编码智能体落地「规范驱动开发（SDD）」的开发者与团队，基于内部整理的 SpecKit、OpenSpec 两篇文档，并核对了两个开源仓库 2026 年 9 月的最新 README。覆盖：两个工具从安装到跑通第一个项目的完整步骤、产物结构、治理机制、横向对比与选型建议。

## SDD 是什么：把「写清楚要什么」变成工程产物

AI 编码智能体很强，但当需求只存在于聊天记录里时，产出不可预测——同一个提示词换个会话就可能跑偏。规范驱动开发（Spec-Driven Development, SDD）的解法是：把「规范」而非「代码」作为核心产物，人先与 AI 就规范达成共识，代码由规范驱动生成 [[1]](https://v11enp9ok1h.feishu.cn/docx/BMocdaAI4o2KMtxuCBScVx5Unae)[[2]](https://v11enp9ok1h.feishu.cn/docx/TS1TdYeyUo9SJmxoPiAcPz6bn2c)。

两个代表性工具的口号恰好概括了分工：GitHub SpecKit 说「Define what to build before building it」（先定义要建什么，再动手建），强调规范即代码、工程化治理；Fission-AI OpenSpec 说「Agree before you build」（先对齐，再实现），强调轻量、可审计、AI 原生协作。

截至 2026 年 9 月的进展：SpecKit 已发布 1.0.0（首个 commit 一年后）[[3]](https://github.com/github/spec-kit)；OpenSpec 推出了新的 artifact 引导工作流（/opsx:\* 系列命令）[[4]](https://github.com/Fission-AI/OpenSpec)。本教程实操部分以两个仓库当前 README 的命令为准。

## 实操一：用 SpecKit 跑通第一个 SDD 项目

### 环境准备

四项前置条件缺一不可 [[3]](https://github.com/github/spec-kit)：

- **uv**（推荐）或 pipx，用于安装 CLI；**Python 3.11+；Git**；**任一受支持的 AI 编码智能体**。SpecKit 兼容 Claude Code、Gemini CLI、GitHub Copilot、Cursor、Qwen Code、opencode、Windsurf、Codex CLI、Kilo Code、Auggie CLI、Roo Code、CodeBuddy、Amazon Q Developer CLI 等 13 种智能体 [[1]](https://v11enp9ok1h.feishu.cn/docx/BMocdaAI4o2KMtxuCBScVx5Unae)，仓库 README 现已标称支持 30+，可运行 `specify integration list` 查看当前版本支持的全部集成。

### 安装与初始化

```bash
# 方式一：从 GitHub 安装（把 vX.Y.Z 替换为最新 release tag，注意保留前导 v）
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z

# 方式二：直接装 PyPI 版
uv tool install specify-cli

# 初始化项目并指定智能体集成
specify init my-project --integration copilot
cd my-project

# CI / 无键盘环境（避免卡在交互式选择器）
specify init my-project --non-interactive --ignore-agent-tools

# 已有目录里初始化
specify init --here --force --non-interactive --integration claude

# 版本自检与原地升级
specify self check
specify self upgrade
```

初始化只执行一次：CLI 校验工具链、下载模板、设置权限并完成 Git 初始化，随后在项目里生成 `.specify/` 目录和所选智能体的命令文件（如 Claude Code 对应 `.claude/commands/`，Copilot 对应 `.github/prompts/`，Cursor 对应 `.cursor/rules/`）[[1]](https://v11enp9ok1h.feishu.cn/docx/BMocdaAI4o2KMtxuCBScVx5Unae)。

### 核心工作流：七个 slash 命令

在项目目录里启动智能体，按顺序执行（命令名以 `/speckit.*` 形式为例；Codex CLI 等用 `$speckit-*`）：

| 步骤 | 命令 | 做什么 |
|-|-|-|
| 0（一次性） | `/speckit.constitution` | 建立项目治理原则与开发守则 |
| 1 | `/speckit.specify` | 描述要建什么（what 与 why，不谈技术栈） |
| 2 | `/speckit.clarify` | 澄清模糊需求，回写 spec.md |
| 3 | `/speckit.plan` | 给出技术栈与架构方案 |
| 4 | `/speckit.analyze` | 跨产物一致性校验 |
| 5 | `/speckit.tasks` | 把方案分解为可执行任务 |
| 6 | `/speckit.implement` | 按任务清单执行实现 |

各命令的功能、触发脚本与产物对照来自文档整理 [[1]](https://v11enp9ok1h.feishu.cn/docx/BMocdaAI4o2KMtxuCBScVx5Unae)；v1.0 工作流在此之上新增了 `/speckit.converge`：对照 spec、plan、tasks 收敛检查实现，未收敛就把剩余工作追加为新任务——**重复 implement 与 converge，直到 converge 报告 Converged 为止**[[3]](https://github.com/github/spec-kit)。另有 `/speckit.taskstoissues`（任务转 GitHub Issue）和 `/speckit.checklist`（自定义质量检查清单）两个可选命令。

写第一条规范时，聚焦「要什么、为什么」，而不是「用什么技术」：

```text
/speckit.specify Build an application that can help me organize my photos
in separate photo albums. Albums are grouped by date and can be re-organized
by dragging and dropping on the main page. Albums are never in other nested
albums. Within each album, photos are previewed in a tile-like interface.
```

技术决策放到 `/speckit.plan` 里给：

```text
/speckit.plan The application uses Vite with minimal number of libraries.
Use vanilla HTML, CSS, and JavaScript as much as possible. Images are not
uploaded anywhere and metadata is stored in a local SQLite database.
```

### 产物结构与九条宪章

一次完整流程会沉淀这些产物：`constitution.md`（治理原则）、`specs/NNN-feature/spec.md`（功能规范）、`plan.md` 及配套的 `research.md`、`data-model.md`、`contracts/`、`quickstart.md`，以及 `tasks.md`（任务清单）。`.specify/memory/constitution.md` 里的九条宪章是治理核心 [[1]](https://v11enp9ok1h.feishu.cn/docx/BMocdaAI4o2KMtxuCBScVx5Unae)：

| 条款 | 名称 | 类型 | 作用 |
|-|-|-|-|
| I | Library-First 原则 | 塑造性 | 指导 plan.md 架构 |
| II | CLI 接口强制 | 塑造性 | 要求库必须有命令行接口 |
| III | 测试优先开发 | 塑造性 | 强制 TDD，约束 tasks.md |
| IV | 文档优先 | 塑造性 | 实现前需完善文档 |
| V | 功能隔离 | 塑造性 | 强制关注点分离 |
| VI | 版本控制规范 | 塑造性 | 强制 Git 工作流 |
| VII | 简约门控 | 前置门控 | 超过 3 个项目或未来预留即阻断 |
| VIII | 反抽象门控 | 前置门控 | 禁止间接框架、多模型 |
| IX | 集成优先门控 | 前置门控 | 缺少契约或测试即阻断 |

`/speckit.analyze` 会校验所有产物，发现 CRITICAL 级宪章违规会**阻断 `/speckit.implement` 阶段**——这就是「治理层」的含义：不是文档摆设，而是真的卡在流程里。

### 按需扩展

v1.0 之后的 SpecKit 有三层定制机制：extension 加新能力（`specify extension add bug` 提供修 bug 的 assess→fix→test 工作流）、preset 改模板与术语、bundle 一键装整套角色化配置。用 `specify extension search` / `specify preset search` / `specify bundle search` 浏览 [[3]](https://github.com/github/spec-kit)。

## 实操二：用 OpenSpec 跑通第一个项目

### 安装与初始化

前置条件只有一项：**Node.js 20.19.0 或更高版本**[[4]](https://github.com/Fission-AI/OpenSpec)。零外部 API Key，可离线使用 [[2]](https://v11enp9ok1h.feishu.cn/docx/TS1TdYeyUo9SJmxoPiAcPz6bn2c)。

```bash
# 全局安装
npm install -g @fission-ai/openspec@latest

# 在项目根目录初始化（交互式选择你用的 AI 工具）
cd your-project
openspec init

# 升级后刷新各工具的提示 stub（不覆盖用户自定义区）
openspec update
```

### 核心工作流：explore → propose → apply → archive

OpenSpec 当前的工作流是一组 `/opsx:*` 命令（不同工具拼写略有差异：Cursor 和 Copilot 写 `/opsx-propose`，Amazon Q 写 `@opsx-propose`，Codex 写 `$openspec-propose`；`openspec init` 会打印你所选工具的正确形式）[[4]](https://github.com/Fission-AI/OpenSpec)：

```text
你：/opsx:explore
AI：想探索什么？
你：我想加暗色模式，但不确定怎么做得干净。
AI：看了你的样式结构……最干净的路径是 CSS 变量 + 小的主题
    context + 系统偏好探测，不加新依赖。要立项吗？
你：好。

你：/opsx:propose add-dark-mode
AI：已创建 openspec/changes/add-dark-mode/
    ✓ proposal.md — 为什么改、改什么
    ✓ specs/       — 需求与场景
    ✓ design.md    — 技术方案
    ✓ tasks.md     — 实现清单

你：/opsx:apply
AI：按 tasks 实现中……全部完成。

你：/opsx:archive
AI：归档到 openspec/changes/archive/，specs/ 已更新。
```

四步各自的定位：**explore** 没想清楚时先做无风险的探索；**propose** 生成变更提案（人审这一步，代码还一行没写）；**apply** 按 tasks.md 实现；**archive** 把 delta 应用回 `specs/` 并归档。需要更完整的命令集（`/opsx:new`、`/opsx:continue`、`/opsx:ff`、`/opsx:verify`、`/opsx:bulk-archive`、`/opsx:onboard`）时，用 `openspec config profile` 切到 expanded profile 再 `openspec update`。

### 目录结构与 delta 格式

初始化后的目录骨架 [[2]](https://v11enp9ok1h.feishu.cn/docx/TS1TdYeyUo9SJmxoPiAcPz6bn2c)：

```text
openspec/
├── AGENTS.md            # 与各 AI 工具约定的根级说明
├── project.md           # 项目上下文
├── specs/               # 当前事实（source of truth）
│   └── capability-name/
│       ├── spec.md
│       └── design.md
└── changes/             # 变更提案（每个提案一个子目录）
    └── change-name/
        ├── proposal.md
        ├── tasks.md
        └── specs/       # delta：只写变动部分
```

delta 用明确的节头表达变更类型：`## ADDED Requirements`、`## MODIFIED Requirements`、`## REMOVED Requirements`、`## RENAMED Requirements`。需求用带场景的纯 Markdown 写，无需学新语法 [[4]](https://github.com/Fission-AI/OpenSpec)：

```markdown
## ADDED Requirements

### Requirement: Theme selection
The app SHALL let users switch between light and dark themes,
defaulting to the system preference.

#### Scenario: User toggles dark mode
- **WHEN** the user clicks the theme toggle
- **THEN** the app switches to dark mode and persists the choice
```

`openspec/` 下常用 CLI 命令：`list` / `show` 查看提案与规范、`validate` 做格式与规则校验（可选严格模式）、`archive` 完成归档 [[2]](https://v11enp9ok1h.feishu.cn/docx/TS1TdYeyUo9SJmxoPiAcPz6bn2c)。团队落地第一步建议：仓库根跑一次 `openspec init` 并放好标准 `AGENTS.md`，让不同 AI 工具遵循同一套约定。

## 横向对比与选型

两个工具的定位差异（整理自 OpenSpec 文档的对比表 [[2]](https://v11enp9ok1h.feishu.cn/docx/TS1TdYeyUo9SJmxoPiAcPz6bn2c)）：

| 方面 | OpenSpec | SpecKit |
|-|-|-|
| 设计取向 | 本地化、可审计、AI 原生；受控/离线环境先对齐规范 | 规范即代码、工程化治理；规范作为可执行产物驱动实现与自动化 |
| 工作流与产物 | Draft→Review→Implement→Archive，specs 与 changes 分离，审计友好 | CLI（specify）+ 模板 + slash 命令生成 spec/plan/tasks，宪章治理门控 |
| AI 集成 | 本地 slash/提示 stub，无需外部 Key | 多智能体适配（整理时 13 种，现已 30+），脚本在 CI 与开发流中更新上下文 |
| 验证与治理 | delta 变更记录构成可审计链路，验证器检查格式与语义 | 九条宪章前置门控，/analyze 分级校验并可阻断实现 |
| 优势 | 轻量、可审计、易于私有/受限环境部署 | 工程化成熟、与 GitHub/CI 深度集成，模板与脚本支持流水线 |
| 典型场景 | 需要强审计、合规、离线运行的小到中型团队 | 希望规范直接驱动 CI/CD 与代码/测试生成的大型工程化团队（GitHub 优先） |

选型给一个直接的判断：**只有 Node.js、要审计链路、环境受限或离线——选 OpenSpec**；**已有 GitHub 工程体系、想让规范接 CI/CD 和代码生成、团队规模大——选 SpecKit**。OpenSpec 官方对 SpecKit 的评价也是「thorough but heavyweight」，把轻量迭代留给自己 [[4]](https://github.com/Fission-AI/OpenSpec)。

两者并不互斥。组合打法来自文档建议 [[2]](https://v11enp9ok1h.feishu.cn/docx/TS1TdYeyUo9SJmxoPiAcPz6bn2c)：用 OpenSpec 承担「审计与对齐」环节，作为人机对齐的事实源；审查通过后把 delta 同步到 SpecKit 风格的 `specs/` 目录（或由其消费），驱动实现与 CI——形成「审计对齐 → 工程化实现」的闭环，兼顾合规性与自动化效率。

## 常见坑与注意事项

- **SpecKit 安装 tag 必须带前导 v**：`v1.0.0` 而非 `1.0.0`，否则 git 引用解析失败 [[3]](https://github.com/github/spec-kit)。
- **别跳过 converge 循环**：implement 之后跑 `/speckit.converge`，未 Converged 就回到 implement 继续，直到报告 Converged [[3]](https://github.com/github/spec-kit)。
- **OpenSpec 重上下文卫生**：README 明确建议开始实现前清空上下文窗口，全程保持干净的 context [[4]](https://github.com/Fission-AI/OpenSpec)。
- **OpenSpec 遥测默认开启**：只收集命令名与版本，但受限环境可用 `openspec config set telemetry.enabled false` 或环境变量 `OPENSPEC_TELEMETRY=0` 关闭 [[4]](https://github.com/Fission-AI/OpenSpec)。
- **文档数据有时效**：本教程引用的两篇内部文档整理于 SpecKit 支持 13 种智能体、OpenSpec 尚为三阶段工作流的时点；两个仓库迭代都很快（SpecKit 一年内到 1.0.0，OpenSpec 已切换 /opsx 工作流），动手前以 `specify integration list` 和官方文档为准。

## 小结

SpecKit 与 OpenSpec 代表了 SDD 的两条路线：前者把规范做成工程化产物，用宪章和门控保证纪律，适合重流程、重集成的团队；后者把规范做成轻量协议，用 delta 链路保证可审计，适合快迭代、受限环境。对个人开发者，半天即可分别跑通两者的 quickstart；对团队，建议先在小项目上试运行一个迭代周期，再决定选型或组合方式。