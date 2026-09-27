# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库定位

Claude Code / Codex 双宿主的**插件 marketplace 仓库**（`cospowers-v3`），实现 PEV（Plan-Execute-Verify）研发流程。没有应用代码，主体是 Skill / Agent 提示词 + Node 脚本 + 合同测试。

- 权威流程标准在 `docs/standards/*-stage-flow.md`，仅覆盖 5 个阶段：user-requirements、system-requirements、overall-design、feature-design、module-design（draft-solution、micro-design、decompose、tdd-loop 无 stage-flow 文档）。
- `/CLAUDE.md` 本身被 `.gitignore` 忽略，仅本地生效。
- `docs/` 是白名单式 gitignore：除 `README.md`、`standards/`、两个点名的 decompose/dev 文档外，**新增文件默认不入库**。
- `plugins/cospowers-master/tests/verify-ipd-token.mjs` 是本地诊断脚本（非测试，测试运行器不会捡起它），用于验证 IPD token 获取链路。

## 测试

```bash
# 只跑目标插件（根目录 npm test = node --test 递归全量，非常慢）
node --test plugins/cospowers-feature-design/tests/*.test.mjs
node --test --test-name-pattern "版本五方一致" plugins/cospowers-feature-design/tests/contracts.test.mjs
py plugins/cospowers-module-design/tests/test_contracts.py   # Python 测试用 py（python 不在 PATH），直接执行
cd plugins/cospowers-tdd-loop && npm run validate            # 14 个 node --check + pev doctor + stage-lint + 测试
```

`node --test` 传目录在本机不可靠，传具体文件或用 glob 展开；`sync-template` 等集成测试会拉起本地 git/http，较慢。

## 插件的 harness 组成

每个 `plugins/<name>/`（11 个插件，双 marketplace `.claude-plugin/` 与 `.codex-plugin/marketplace.json` 登记，两文件当前逐字节相同）：

| 目录/文件 | harness 作用 |
|---|---|
| `.claude-plugin/plugin.json` + `.codex-plugin/plugin.json` | 双宿主 manifest，版本/描述必须一致 |
| `skills/<skill>/SKILL.md` | Skill 定义；`references/` 放细则（渐进披露），`scripts/` 放配套脚本；tdd-loop 的 skill 目录还可嵌 `agents/`（prompt 文件、`openai.yaml`）与 `agent-profile.yaml`（仅 plan/execute/verify-tdd-loop 三个段执行器有） |
| `agents/<role>.md` | 子代理定义（8 个插件有），经 Agent 工具以 `插件名:角色名` 调用 |
| `hooks/hooks.json` | 仅 cospowers-master：SessionStart 跑 `sync-template.mjs` |
| `.mcp.json` | 仅 cospowers-qianliu-ipd：cosworkflow MCP（`npx -y @costrict/cosworkflow@0.2.2`，依赖内网 registry，外网拉不起） |
| `scripts/` | Node `.mjs`/`.js`：路径解析、机械门、状态 CLI |

## Skill 体系

全仓库 81 个 SKILL.md。按约定对用户暴露的**公共入口 Skill 只有 15 个**：

| 插件 | 公共 Skill | 职责 |
|---|---|---|
| cospowers-master | `cos-master` | `/cos-master <需求> [-sop <id>]` 总调度：立项 + 每次推进一个阶段 |
| cospowers-master | `commit` | AI 原生 Git 提交（commit → push → MR；保护分支 main/master/develop/release/*/m-feature-*、原子提交禁 `git add .`、`[AI-COMMIT][TYPE]` 格式；push/MR 须用户逐项确认；提交前校验两 manifest 版本一致） |
| cospowers-draft-solution | `cos-draft-solution` | 方案初稿 |
| cospowers-requirements | `cos-user-requirement` | 用户需求（TR1） |
| cospowers-system-requirements | `cos-system-requirement` | 系统需求（TR2） |
| cospowers-overall-design | `cos-overall-design` | 总体设计 |
| cospowers-feature-design | `cos-feature-design` | 特性设计（总设+概设融合） |
| cospowers-module-design | `cos-module-design` | 概要设计 |
| cospowers-micro-design | `cos-micro-design` | 微型设计 |
| cospowers-decompose | `cos-task-decompose` | 计划：拆解为 implementation-plan.md、task-graph.md、冻结 tasks.json 三件套 |
| cospowers-tdd-loop | `cos-tdd-init` | 项目 TDD 初始化（plan → 确认 → apply 两阶段） |
| cospowers-tdd-loop | `cos-tdd-development` | 唯一完整 TDD 执行入口；缺 init receipt 时先走 cos-tdd-init；L1 单测 + 可选 L2 功能测试 |
| cospowers-tdd-loop | `cos-code-review` | 规范/SDL 检查与自动修复、覆盖率报告；默认绑定 finally-L1/finally-L2 |
| cospowers-tdd-loop | `cos-merge-branch` | 拉取分支并处理集成冲突 |
| cospowers-qianliu-ipd | `cos-qianliu-ipd` | 千流 IPD 工作项/交付物管理（经 cosworkflow MCP 14 个工具）；可联动 /cos-master（交接细则见其 `references/cos-master-handoff.md`） |

- `cos-qianliu-ipd` 是唯一显式 `user-invocable: true` 的 Skill；`cos-feature-design` 约定不写字段（默认可调用）。
- 其余 66 个应全部 `user-invocable: false`，**但 draft-solution 有 4 个漏标**（discover-draft-solution-context、draft-solution-evaluator、execute-draft-solution、prepare-draft-solution-brief）——守卫测试只有 feature-design 有（`contracts.test.mjs` 断言入口 skill 无字段、其余必须显式 false），draft-solution 无此断言。

### 内部 Skill 家族（按命名约定）

- 阶段插件通用五件套：`discover-*-context`（现状地图）→ `plan-*`（任务 DAG）→ `execute-*`（调度写作）→ `*-evaluator`（只读评审计分），外加可选 `repair-*` / `write-*` / `analyze-*-fmea`。
- 变体：`prepare-*-brief`（输入简报）、`integrate-system-requirements`（候选装配）、`coordinate-overall-design-execution|evaluation`（总设并行协调）、`cross-design-evaluator` + `design-module-database`（模设跨模块评审与数据库设计）。
- tdd-loop 内部家族：`code-lint-check|fix`、`code-sdl-check|fix`（仅由 cos-code-review 编排）；`auto-test`、`auto-test-claw|mcp|rf`（测试引擎，仅由 Execute dispatcher 调用，auto-test 按 execute-context 路由三种引擎）；`plan-tdd-loop`、`execute-tdd-loop`、`verify-tdd-loop`、`tdd-loop`、`pev-tdd-loop`、`level-manager`（PEV 段执行器与层级管理）；`cos-tdd-coding`（L1 逐 task 编码）；`ferret`（远程服务器操控：执行命令/同步文件/构建部署，供远程测试用）；`qianliu-aitest|ci|tp`（千流平台对接）。
- master 内部：`grilling`（决策树逐轮需求澄清，fixbug SOP 的 req-clarify 入口，用户不可手动调用）、`grill-with-docs`、`daedalus-knowledge`（团队知识检索，中文关键词 `LIKE %keyword%`，禁止翻译成英文）、`domain-modeling`。

### SKILL.md 编写约定

- **frontmatter**：`name`、`description`（写明触发条件与边界，是路由依据；仓库以中文为主，draft-solution 全部 5 个及 master 的 grilling/grill-with-docs/domain-modeling 是英文——新写建议中文）、`allowed-tools`（如 `Bash(node *)`）、`allowedCommands`、`allowedPaths`、`user-invocable`。
- **双宿主暴露控制**：Claude Code 侧用 `user-invocable: false` 隐藏内部 Skill；Codex 侧用 skill 目录内 `agents/openai.yaml` 的 `policy.allow_implicit_invocation: false`（公共 skill 则带 `interface.default_prompt`）——**该机制仅 tdd-loop 实现了**，其余插件在 Codex 侧无 per-skill 配置。
- **动态注入块**：```` ```! ```` 代码块在 Skill 加载时执行（10 个 SKILL.md 有），两种形态：输出结构化状态（如 `read-autorun-override.mjs` 输出 `<RUN_MODE>`/`<OVERRIDE>`/`<RUN_MODE_ROOT>`）供 Skill 机械消费；或用 heredoc 传参（cos-master 的 `pilot.mjs <<'COS_ARGS_END'` + `$ARGUMENTS`，承载 `/cos-master` 的命令行参数）。注入失败不算静默通过（如 `run-mode-unavailable` → `blocked`）。
- **`${CLAUDE_PLUGIN_ROOT}` 定位插件根**，脚本路径一律经它解析，禁止猜测或依赖 cwd。
- **autorun / interactive 双分支**：运行模式来自项目 `.cospowers/stage-config.json` 的 `autorun`（缺失/false → interactive，true → autorun + 八条代理规则，非法值 → blocked）；autorun 下由脚本输出与 planDigest 代替用户确认（协议规定的数据解析，不是猜测），机械门与质量门不放宽。

## Agent 体系

Agent 定义在插件顶层 `agents/<role>.md`（52 个）。frontmatter：`name`/`description`/`background` **恒存在**（`background` 恒为 `false`，即强制前台同步执行，不用后台轮询）；`skills`（预加载 Skill，44/52）与 `maxTurns`（26/52）**可选**——自包含合同型 agent 不写 `skills`；`tools` 白名单仅 requirements 插件 7 个 agent 使用（如 challenger 只读三件套 `Read/Grep/Glob`）。

调用规则：主流程经 Agent 工具调 `插件名:角色名`（如 `Agent(cospowers-feature-design:writer)`）；角色只执行预加载 Skill 的合同，不与用户交互，不把自身完成当作阶段完成；叶子失败如实上报，缺失角色时报错停止、不改调旧插件。**agent 可嵌套调用命名 agent**（requirements：discovery→challenger、plan→plan-shard、evaluator→evaluator-shard）。允许并发但同回合批量发出、`run_in_background: false` 收齐再处理（overall-design 同回合最多并发 8 个）。

各插件顶层角色清单（完整列举，不适用「通用四角色」假设）：

| 插件 | agents/ 角色 |
|---|---|
| feature-design | discovery、plan、execute、evaluator、writer、reviewer、fmea（7） |
| module-design | discovery、plan、execute、evaluator、writer、fmea、database、cross-evaluator（8） |
| overall-design | discovery、plan、execute、evaluator、repair、execute-coordinator、evaluation-coordinator、knowledge-search（8） |
| system-requirements | plan、writer、integrator、knowledge-search、evaluator（5；**无 discovery/execute**） |
| requirements | discovery、plan、execute、evaluator、challenger、plan-shard、evaluator-shard、knowledge-search、prepare-requirement-brief（9） |
| micro-design | discovery、plan、execute、evaluator、knowledge-search、prepare-brief（6） |
| draft-solution | discovery、evaluator、execute、prepare（4；**无 plan**，Plan 由主流程完成） |
| decompose | explore、plan、execute、evaluator、repair（5；explore 做设计文档勘察，无 discovery） |
| tdd-loop | **无顶层 agents/**。子代理用另一套机制：skill 级 `agents/*.md` prompt 文件（如 `cos-tdd-coding/agents/l1-task-executor.md`、`l1-task-verifier.md`、`cos-code-review/agents/coverage-runner.md`、`auto-test-rf/agents/polling-waiter.md`）+ 每个任务的 capability subagent（用 `Agent(general-purpose)` 全新 spawn 后加载指定 skill/prompt，每 task 一个独立 executor） |
| master、qianliu-ipd | 无 agents。master 在 autorun 模式下用 `Agent(general-purpose)` 子代理执行阶段（禁止用插件命名 agent），done 后不问用户直接续调 |

部分流程还依赖内置 `Agent(Explore)` 做只读勘察（system-requirements、overall-design 等）。

核心不变量：每文件唯一写者、并行任务文件边界不重叠、Verify 只读（不许"自己写自己判通过"）、修复后进入全新 Verify 轮次。

## 关键机制

- **master 状态机**：项目侧 `.cospowers/stage-state.json` 是唯一真相（`pending → running → done/failed`；`reset`/`resume` 需用户确认）。写入通道有两个：立项时 `pilot.mjs → stage_lib.createState` 一次性生成初始状态；运行期转移只能经 `scripts/stage_state.mjs`。不变量（done 连续前缀、最多一个 running、failed 之后只能 pending、start 要求前序全 done）违反即阻塞、绝不自动修复。
- **模板同步（种子 vs 权威）**：插件内 `plugins/cospowers-master/rules|templates/<N-阶段>/` 只是**种子/回退源**——仅从未同步过的项目用它补缺。SessionStart hook `sync-template.mjs` 的真正权威是 **IPD 模板仓库**：`git ls-remote` 比较 `.cospowers/.ipd-sync/manifest.json` 记录的 commit，上游优先覆盖本地受管文件（本地改动会被冲掉，无备份），用户自建文件不碰，网络失败不阻断。**改规则/模板要改 IPD 模板仓库**，不是改插件目录。会话中途 `/cos-master` 走 `--local-only` 不重拉 IPD（防止跑到一半被换配置）。
- **产物布局与路径解析**：`<stageRoot>/input/`（阶段级共享）+ `internal/`（阶段级共享）+ `output/<runId>/`（按轮次，只有 output 有 runId 子层）。`docsRoot` 默认 `.cospowers/docs`，由项目 `cospowers.config.json` 的 `paths.docsRoot` 覆盖；设计类插件统一经各自 `scripts/paths.mjs` 解析（tdd-loop 是 `scripts/tdd_paths.js`；requirements 无顶层 scripts/，qianliu-ipd 亦有一份 `scripts/paths.mjs`），不要硬编码。
- **质量门**：跨阶段一致的框架是——扣分 Critical -40 / Error -20 / Warning -8 / Info -0；等级 A≥95 / B≥80 / C≥65 / D<65、有 Critical 即 F；passed 需无 Critical/Error 且总分 ≥80；修复最多三轮。**五维维度与权重各阶段不同**：feature-design 与 module-design 是 feasibility 35% / testability 25% / requirements 20% / standards 15% / architecture 5%，micro-design 是另一套（改动明细 30% / 关联影响 25% / 测试用例 25% / 锚点守恒 15% / 文档质量 5%）。机械门脚本形态各异：feature-design 是 `scripts/check-design.mjs` 的 plan/output/report 子命令，system-requirements 是三个独立脚本（check-plan/check-tr2/check-evaluation），module/micro 在 evaluator skill 的 references 内。
- **PEV 引擎**：tdd-loop 自带 `framework/bin/pev` CLI（`doctor` / `state` / `context` / `verdict` / `aggregate` / `launch` / `handoff` / `stage-lint`），阶段合同放 `stages/`（gate.json 含 `max_rounds:3`、levels/ 下 L1 单测与 L2 功能测试两道层级门、prompts/、verifiers/）。**状态转换只认结构化证据、proof 和退出码，不认下游自然语言结论**（verdict 判 PASS 才产凭证，mark-complete 必须验凭证才推进，永不 AI 自评）。防双派发：`level_manager.js` 的 `issueAction()` 先原子写 action envelope 再返回路径（`next` 派发时执行），派发前另跑只读守卫 `scripts/dispatch_guard.js`（报 fresh/in_flight 并有界轮询 resultPath，不写状态），只信 envelope 登记的 resultPath；子代理失踪由 `write_technical_failed.js` 收口。L1 按依赖拓扑分层 + `maxParallelTasks` 并行批派发。init receipt（`init-result.json`）钉死 config/tasks/workflowRunId 来源，execute 全程比对。单阶段 PEV 参考实现在 `examples/example-cospowers-pev/`（开发新插件的模板，不入 marketplace）。
- **冻结合同**：tdd-loop 消费的 `tasks.json` 要求 `protocolVersion: "3.0"`、`immutable: true`、任务 id 严格匹配 `TASK-\d{3}`（恰好三位数字）、outputs/writeScope 为项目相对路径（glob/`..` 逃逸/反斜杠是 error，绝对路径仅 warning）、每个精确 output 路径唯一归属一个任务、依赖无环且无传递性冗余；内容摘要（planDigest/tasksDigest）变化即 ContractError，不被静默接受。

## 硬约束

1. **版本一致性（分级，非普适）**：有 `cospowers.config.json` 的 7 个插件（draft-solution、requirements、system-requirements、overall-design、feature-design、module-design、micro-design）是**五方一致**（两份 plugin.json + config `_version` + 两份 marketplace 条目）；其余 4 个（master、decompose、tdd-loop、qianliu-ipd）是**四方**（无 config）。测试守护参差：feature-design、system-requirements、decompose 的 tests **硬编码版本号**（漏改必挂）；module/overall/requirements/micro-design 只做相对一致性断言（不带版本号）；**draft-solution、master、tdd-loop、qianliu-ipd 无版本测试**——这 4 个漏改只有 commit skill 的提交前校验（两 manifest 版本不一致即停止提交）兜底。
2. **双 marketplace 条目等价**：两份 marketplace.json 的同名条目必须 deepEqual，description/author 与插件 manifest 一致；守卫是**条目级**测试（feature-design、micro-design 有），没有整文件 deepEqual 测试。
3. **LF 强制**（`.gitattributes`）：SKILL.md 动态块的 heredoc（含 `$ARGUMENTS` 传参）混入 `\r` 会被 shell 权限检查（shell-quote/bash tokenize）拒绝，不得引入 CRLF。
4. 内容以中文为主、文件一律 UTF-8；Windows 下读写中文文件显式按 UTF-8 处理（现存 8 个英文 description 属历史例外）。
