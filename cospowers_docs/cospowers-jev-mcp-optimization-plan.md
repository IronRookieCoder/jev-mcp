# CoPowers × Jev-MCP 执行效率优化方案

> 目标：在不降低交付质量（Verified Completion Rate、测试通过率、Milestone 完成率、产物质量）的前提下，引入 jev-mcp（D:\third\jev-mcp，TypeSafe Jev 判断模型的 MCP 封装，单次判断 150–500ms、成本远低于 1 美分、返回类型化概率分布 + 置信度），显著降低高能力 LLM 的调用次数、Token 消耗与端到端时长。
>
> 本方案基于对仓库 11 个插件执行链路的调用点级勘察（tdd-loop PEV 引擎、各阶段 evaluator/discovery/grilling、master 状态机）与 jev-mcp 11 个工具合同（README + skills/jev）的实测走读。
>
> 2026-09-27 核实修订：已对照两仓库源码逐条核实本方案的断言（关键断言行号级验证），并修正初稿的三处硬伤——直调技术路线（`ask()` 不通兼容网关，见 3.2 决策 1）、共享客户端落点（跨插件运行期调用无先例）、TOP 2 对 TR1/TR2 的「可复制」假设——以及 jev_verify 合同细节与 TOP 1/2/3 的改动面欠账。各节内标注「核实后修正/核实补充」处为本次修订内容。同日二次复核（4 路并行源码勘察）再修数处：paths.mjs 副本实为 9 份（非 7）、TR2 `contracts.test.mjs:78` 守卫对象勘误、B2 轮次拆分标注为估算、`auto_accept` 起步值统一 0.9（jev 默认 0.8）、fragment schema 结构与工具层封闭位置措辞精化、framework 注入仅限 L2。

---

## 1. 当前 CoPowers 执行链路中的主要效率瓶颈

### 1.1 成本结构总览

CoPowers 的状态机、计分、结构校验、PASS 凭证互锁已经高度脚本化（`pev verdict/aggregate`、`check-design.mjs`、`stage_lib.mjs` 等），**LLM 成本集中在三类环节：评审判断、上下文构建、修复循环重复**。

| # | 瓶颈 | 事实依据 | 量级 |
|---|---|---|---|
| B1 | **评审分片是最大 Token 消耗面，且修复轮全量重跑** | 分片数（核实修正）：feature 6 项语义检查、每批 ≤5（evaluate SKILL L15）；TR1 固定 5 片（SKILL L28）；TR2 为 1 global + N partition Reviewer（verify-protocol L15-16 原文只写「每个分片」；当前 review-partitions.json 为 3 分片、合计 4 个 Reviewer）。每片预载的 skill/agent 合同 15–115 行不等 + 各自读输入文档；evaluate-feature-design SKILL L16、requirement-evaluator SKILL L44（「每轮均执行全量评估…不做范围化复审」）、TR2 verify-protocol L72（「不维护增量复审依赖和复用来源」）明文完整复评；仅 overall-design 有 inputHash 增量复用先例（`mechanical-review.mjs` 的 hashFiles + `finalize-overall-design-evaluation.mjs` 的 reuse()，机制完整可抄） | 每失败轮成本 ≈ 首轮评审成本；每阶段最多 2–3 个失败轮 |
| B2 | **tdd-loop 单 task LLM 交互密度高** | 事实：单 task 分布在 controller/executor/verifier 3 个隔离 agent 会话（SKILL task 循环 L45-55），RETRY 由 max_rounds=3 限定；**轮次拆分（22–39 轮/task、controller ~6–7、executor ~10–22、verifier ~6–10、RETRY +15–35）为估算、仓库无出处**——唯一量化锚点是 cos-tdd-coding SKILL:43 记录的 round-1 曾烧掉 ~178 条转录 | 30 task 的中型迭代 ≈ 750+ 轮主 LLM 交互（由估算推导） |
| B3 | **判断型核对由 LLM spawn 承担，其中大半是「清单包含性」低复杂度判断** | l1-task-verifier 的核对项：outputs 已产生（存在性）、changed-files ⊆ writeScope（集合包含）、报告覆盖 acceptance（清单核对）——只有最后一项真正需要语义理解 | 每 task 每 round 一次完整 verifier spawn |
| B4 | **重复上下文加载** | executor/verifier 每次 spawn 都重读整个冻结 tasks.json（只为取 1 条 task，`l1-task-executor.md:13`）；每轮 Evidence 重读 gate.json + checker manifest；L2 每层重新注入 53–387 行 skill | 每 task 至少 2 次全量 tasks.json 读入 |
| B5 | **知识检索蒸馏完全由 LLM 做** | daedalus-knowledge SKILL L54-63（「### 3. 评估命中」）：每命中 1 次相关性/可采性判断（输入结构化且按类型而异：document 看标题/摘要、capsule 看 confidence 与内容、gene 看 summary/validation；输出二值+一句话），无任何脚本参与；由 knowledge-search 子代理整段承担——该 agent 位于 requirements/system-requirements/overall-design/micro-design 四插件（master 自身无 agents），运行期按项目 `.cospowers/cospowers.config.json` 的 `knowledgeArchive.skill`（默认 `cospowers-master:daedalus-knowledge`，由 sync-template 在项目缺失时补写；IPD 上游路径下该文件以模板仓库为准）经 Skill 工具加载本 skill | 每阶段 1 个子代理 spawn × 4 类（document/kb_network/capsule/gene）× top_k=20 命中 |
| B6 | **「伪机械检查」由 LLM 执行** | micro-design evaluator 的 9 条机械检查（锚点格式/孤儿需求/落点存在/用例回指等，SKILL L20-29 步骤 3）、TR2 PRE-01~11 实际执行者是 evaluator LLM，非脚本（定义在 master rules 的 mechanical-checks.md，明文「Evaluator 逐项执行」；feature 的 FD-PRE 是唯一真脚本 PRE；module-design 的 PRE-01~11 同为 LLM 判） | 每轮评估重复发生 |
| B7 | **近机械路由/查表占用 LLM 回合** | auto-test 的 framework→引擎 3 行查表（auto-test SKILL L36-39）、主编排器按 action 类型选 spawn 目标的静态映射、resultPath 存在性核对等 | 每 action / 每 testing 步 1 轮 |
| B8 | **RED 归因无独立核验（质量洞，同时是效率损耗源）** | executor 声明「RED 失败来自本 task 新写的测试」是 LLM 自证，脚本只看退出码；误归因进入 GREEN 后由 verifier/收口聚合兜底，触发不必要的 RETRY 轮 | 每个 RETRY 轮 +15–35 轮 |

### 1.2 必须保留给主模型的工作（不可外包面）

- **生成型写作**：writer 正文、TR2 integrator 跨章统一、executor 测试/实现编码、repair 改码。
- **跨文档语义综合**：方案选型、FMEA 失效场景理解与 S/O/D 定级、grilling 交互生成（用户在环，每轮仅 1 主模型回合，外包收益低且伤交互质量）。
- **高风险裁决**：finding 的 severity/dimension/route 归类（直接决定 -40/-20/-8 扣分与 passed）、user_decision vs execute_repair 分流、最终 passed 判定与用户确认。

---

## 2. Jev-MCP 最适合介入的场景

筛选原则：**高频 × 高 Token × 判断型 × 结构化输入输出 × 可自动验证**。Jev 的 fail-closed 合同（`invalid_response` 永不静默通过）与 CoPowers「注入失败不算静默通过」哲学天然一致。

| 场景 | 所在环节（频率） | 当前方式 | Jev 用法（工具/判断对象） | 是否需要主模型兜底 | 预期收益 | 风险 |
|---|---|---|---|---|---|---|
| S1 评审条目预筛 | feature/TR1/TR2 evaluator 分片派发前（分片数见 B1 × 2–3 轮） | 全部条目交 LLM reviewer 逐条判 pass/fail | `jev_verify`：把「证据可定位」子条目断言化（如「REQ-03 在 ch02 接口清单有落点」），evidence=文档片段——结构化 `{path, section}` 切片仅 feature-design 的 Plan inputs 具备（TR1/TR2 无 inputRefs，见 TOP 2）；按检查域分组批量调用（jev_verify 的 claims 无数量上限，64 是 jev_noul/jev_classify 的上限）；auto-verified-pass 条目从分片剔除 | 是：contradicted/review 条目 + 语义综合类条目照旧给 LLM；severity 归类永远 LLM | 分片 token ↓40–60%，部分整片消失 | 断言化模板质量决定覆盖率；需抽样审计 |
| S2 修复轮增量复评 | 同上，修复轮（B1） | 全量复评（明文规定） | 无需 Jev：inputRefs 文件 sha256 未变 → 复用上轮结论（移植 overall 的 inputHash 机制）；Jev 仅复核「改动是否波及未变文件引用的条目」 | 是：波及判定 review 时回 LLM | 复评轮成本 ↓60–80% | 漏检跨文件传递影响 → 用 Jev 波及复核 + 机械门兜底 |
| S3 L1 task Verify 快门 | cos-tdd-coding controller spawn verifier 前（每 task 每 round） | spawn l1-task-verifier（LLM，6–10 轮） | 先脚本判机械项（outputs 存在、changed-files⊆writeScope——连 Jev 都不用）；再 `jev_verify`：claims=executor 报告关键声明（RED 归因/GREEN 通过/产出齐备），evidence=测试命令输出+changed-files 清单+验收标准；全 auto → 脚本代写 fragment，不 spawn LLM verifier | 是：任一 review/contradicted → 原 LLM verifier 全量重做；RETRY 轮强制 LLM verifier | 顺利 task 省 1 spawn/task（~8 轮+预载）；RED 归因从自证变独立核验（质量增量） | Jev 误判 pass → L1 收口全量聚合 checker 兜底 |
| S4 知识命中蒸馏 | daedalus-knowledge「评估命中」步骤（每阶段 4 类 × top_k=20） | knowledge-search 子代理 LLM 逐条判断+蒸馏（每类 ≤5 条、文件 ≤80 行——上限在四插件的 agent 合同里，非 SKILL 本身） | `jev_rerank`（query=类别意图，candidates=命中标题+摘要，取 top8）→ `jev_classify`（directly_usable / needs_adaptation / irrelevant）→ 脚本生成 team-knowledge-context.md 草稿 | 是：needs_adaptation/review 条目（通常 ≤3）留 LLM 定夺 | knowledge-search spawn 从必然变为条件触发 | 相关性误杀 → 保留原文链接可回查；fallback 链路不变 |
| S5 Discovery 文档预筛 | 各 discover-*-context（每阶段 1–2 批） | discovery LLM + Explore 批次决定读什么，读多读少全凭 LLM | `jev_rerank`：候选文档清单（docs index/文件树摘要）按阶段意图排序，只把 top 相关交给 Explore；`jev_screen` 过滤无实质内容文件 | 是：top 边界条目仍进 Explore | Explore 读取轮次 ↓30–50% | 漏读关键文档 → discovery 自带 partial/blocked 状态与补查机制兜底 |
| S6 finding 三分类辅助 | 各 evaluator 汇总（每轮全量 findings） | reviewer 候选 severity、evaluator 归类（映射表已冻结但归类是 LLM） | `jev_classify` 产 severity 候选分布作参考，不直接采信 | **必须**：归类终审永远 LLM（错分代价 -40 分级） | 降低 evaluator 汇总轮次内的犹豫成本 | 低（仅参考） |
| S7 auto-test 引擎路由 | auto-test 步骤 2（每 testing 步） | LLM 查 3 行映射表 | **不用 Jev**：改脚本判定（framework 值 init 时已冻结） | 否 | 每步省 1 LLM 回合 | 零 |
| S8 checklist checker 判定 | tdd-loop L2 Evidence（每 round 每 checker） | verify-tdd-loop LLM 逐条主观判 pass | `jev_verify`：checklist 条目断言化 vs 产物/日志证据，产出 fragment；LLM 只处理 review 项 | 是：rubric 布尔判定保留 LLM（需代码理解） | L2 Evidence 瘦身 | 中，放二期 |
| S9 KB 五维覆盖判定 | 各 discovery（每阶段） | discovery LLM 判 complete/partial/unavailable | `jev_classify` 五维三档批量判 | 是：partial 档回 LLM 复核 | 减少 Explore 补查批次的误触发 | 中，放二期 |

**优先级排序**（收益 = 调用频率 × 主模型成本 × Jev 可替代率 × 决策可靠性）：

| 排序 | 场景 | 频率 | 主模型成本 | 可替代率 | 可靠性 | 综合 |
|---|---|---|---|---|---|---|
| 1 | S3 L1 Verify 快门 | ★★★ 每 task 每 round | ★★★ 整 spawn | ★★ 60–70%（顺利 task） | ★★★（jev_verify + 机械项先行 + 聚合兜底） | 最高 |
| 2 | S1+S2 评审预筛+增量复评 | ★★★ 每阶段每轮 | ★★★★ 最大 token 面 | ★★ 40–60% 条目 | ★★★（fail-open + 审计） | 高 |
| 3 | S4 知识蒸馏 | ★★ 每阶段 | ★★ 1 spawn | ★★★ 接近全量 | ★★★（结构化最彻底） | 高（且投入最小） |
| 4 | S5 Discovery 预筛 | ★★ | ★★★ | ★★ | ★★ | 中 |
| 5 | S7 路由脚本化 | ★★ | ★ | ★★★ | ★★★ | 顺手做 |
| 6 | S6/S8/S9 | — | — | — | — | 二期 |

---

## 3. 推荐的新执行架构

### 3.1 分层判断模型（Layered Judgment）

核心原则：**规则能判的不给 Jev，Jev 判不了的给主模型，终审凭证永远不外包**。Jev 不替换任何现有脚本与 PEV 内核，只插在「机械门之后、LLM 之前」和「LLM 结论之后、终审之前」两个位置。

```
┌─────────────────────────────────────────────────────────────────────┐
│ CoPowers 编排层（SKILL.md 提示词 + Node 脚本，不变）                  │
│                                                                     │
│  判断请求（评审条目 / task 声明 / 检索命中 / 候选文档）                │
│      │                                                              │
│      ▼                                                              │
│  层0 机械规则（已有脚本 + 补齐 B6/B7 欠账）                           │
│      │ 命中 → 直接终结（0 成本）：存在性 / 集合包含 / sha256 比对 /   │
│      │            查表路由 / 结构校验                                │
│      ▼ 未命中（需要语义判断）                                        │
│  层1 Jev 快判（Node 脚本直调 jev，批量）                             │
│      │  jev_verify / jev_classify / jev_rerank / jev_screen          │
│      │  ├─ verdict=auto：verify 类仅看 confidence≥auto_accept        │
│      │  │   （verify 无 margin 字段）；classify 类另需 margin≥        │
│      │  │   minimum_margin 双门槛                                     │
│      │  │    → 采纳：落结构化结论（fragment / 条目剔除 / 草稿）       │
│      │  └─ review / invalid_response / escaped / 超时 / JEV_DISABLED │
│      │       → fail-open 升级层2（效率回退，质量不降）                │
│      ▼                                                              │
│  层2 主模型 LLM（原 skill / Agent(插件:角色) 链路，只收升级条目）      │
│      │  判断 + 写作 + severity 归类（永远在此层）                     │
│      ▼                                                              │
│  层3 终审不变量（不可外包、不可跳过）                                 │
│      │  pev verdict PASS proof 互锁 / 机械门脚本重算分数与 passed /   │
│      │  用户（或 autorun 代理）确认                                  │
│                                                                     │
│  横切 A：审计通道 —— 抽样 x%（初期 10%）jev-auto 结论回主模型复核，   │
│          偏差率驱动 auto_accept 阈值校准；偏差率超限 → 自动收紧阈值   │
│          或停用对应场景的层1（降级为纯层2）                           │
│  横切 B：度量通道 —— 每次 jev 调用自带 usage，统一落                  │
│          .cospowers/jev-usage.jsonl；LLM 侧由编排脚本统计 spawn 数    │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 具体调用链（以 prompt 要求的模式展开）

```
CoPowers 编排（skill/Node 脚本）
   → 层0 脚本判定（check-design / pev / 新增机械核对）
   → 层1 Jev-MCP（jev_client.mjs 直调，非 LLM 自主调用）
        auto ──→ 结构化结论落盘（fragment/条目剔除/蒸馏草稿）→ 进入层3 终审
        review/invalid/escaped/超时 ──→ 层2 主模型原链路（范围不变，全量重做该判断单元）
   → 层2 主模型结论
   → 层3 终审不变量（机械门重算 + proof 互锁 + 用户确认；Jev 不参与）→ 状态推进
```

关键设计决策：

1. **接入形态：Node 脚本直调为主，MCP 注册为辅**。tdd-loop 与各阶段的判断点都在脚本可触达位置（checkers、机械门、蒸馏步骤），脚本直调有三个好处：宿主无关（双宿主一次实现两边生效）、可用 `node --test` 合同测试 mock、失败处理确定（退出码/JSON，不依赖 LLM 是否记得调用工具）。MCP 形态留给交互式会话内的 ad-hoc 判断（二期）。
   - **直调路径（核实后修正）**：jev-mcp 的问题措辞与 per-claim 映射封闭在 MCP server 的 registerTool 注册闭包里，三个 `validate*Answer` 校验器是入口模块的顶层私有函数（未导出），且 import 入口模块（源码 src/index.ts，发布包 dist/index.js；文件末尾 Boot 块无 main-module 守卫）会**无条件启动 stdio server**——工具层不可整体复用。可行路径是依赖 `@jkudish/jev-mcp` 本体：import `dist/provider.js` 导出的 `askJev(state, questions, model, signal)`（自带五种 provider 解析——含 compatible 网关、60s 超时、有限重试）+ `dist/lib.js` 已导出的映射 helper（`RELATION_TO_VERDICT`、`verifyAction` 等），用 `@typesafe-ai/sdk` 的 choice/noul/score 原语构造 questions，并在客户端**复刻 per-answer fail-closed 校验**（对应 index.ts 未导出的 `validateChoiceAnswer`/`validateScoreAnswer`/`validateNoulAnswer`，函数体合计 56 行、含紧邻注释 65 行）。**`@jkudish/jev-agent-tools` 的 `ask()` 不能作为内部网关模式的主路径**——compatible 传输是 jev-mcp 本地实现的（README「Transport resilience」节明说三种 fetch 型传输 implemented locally），共享 wire 包不含它。运行环境要求 Node ≥22。
   - **客户端落点（核实后修正）**：全仓库没有「插件 A 运行期 `node` 调用插件 B 目录下脚本」的先例（运行期命令一律 `${CLAUDE_PLUGIN_ROOT}` 指向本插件；tdd-loop 的 CONTRACTS.md 亦有 ${CLAUDE_PLUGIN_ROOT} 定位约定——防同名 skill 互撞加载到别插件目录），把共享客户端放 `cospowers-master/scripts/` 供其他插件运行期调用与此隔离原则相悖。改为 **`paths.mjs` 式各插件自带副本**（全仓库 9 个插件各持一份 paths.mjs 是强先例；客户端代码量小，用合同测试钉住各副本行为一致），usage 统一落项目侧 `.cospowers/jev-usage.jsonl`（经项目侧中转是跨插件共享落盘物的既有通道）。客户端封装：超时、重试、`JEV_DISABLED=1` 全量降级（本方案自创的客户端开关，非 jev-mcp 语义）、延迟计时（jev 的 usage 只含 token 数，见 5.2）、usage 落盘。
2. **Jev 结果即 fragment**：tdd-loop 的不变量「状态转换只认结构化证据」保持——jev 的类型化输出（verdict/confidence/usage 的 JSON）由脚本写成 checker fragment，与命令型 checker 产物同构，`pev verdict` 照常求值。**Jev 永远不直接产 PASS proof**。
3. **降级语义 fail-open to LLM**：jev 任何异常（网络/invalid_response/escaped）都升级主模型，绝不静默通过，也绝不阻塞流程——与现有「网络失败不阻断」（sync-template）风格一致。
4. **质量红线（对应目标第 4 条）**：
   - severity/dimension/route 归类、user_decision 分流、最终 passed——永远层2+层3；
   - 每个引入层1 的场景必须有独立的后续验证点（L1 收口聚合 checker、机械门重算、用户确认）；
   - 审计抽样回炉（横切 A），校准数据先行：上线前对每个场景跑 50–100 条历史样本的「Jev vs 主模型」盲比，达到一致率门槛（建议 ≥95% auto 档一致率）才启用 auto 采纳。

### 3.3 部署前提（硬约束）

- **数据出境**：jev 默认把输入发往外部判断服务（TypeSafe/OpenRouter 等）。企业内部源码、需求与设计文档**必须**走 `JEV_PROVIDER=compatible` + 内部 System One 兼容网关（请求/响应合同见 jev-mcp README「Jev-compatible endpoints」节），或在私有化环境部署兼容端点。这是本方案落地的第一前提，需基础设施团队确认。
- **内网 registry**：`@jkudish/jev-mcp` 与 `@jkudish/jev-agent-tools` 均发布在公网 npm；若目标环境仅有内网 registry（仓库内 cosworkflow 即如此，其 `.mcp.json` 显式指向内网 npm 地址），需将依赖发布到内网或 vendored 进插件目录。
- **密钥管理**：`JEV_API_KEY`/`TYPESAFE_API_KEY` 走项目 `.cospowers/cospowers.config.json` 或环境变量注入，不进提示词、不入库。
- **运行时与传输语义（核实补充）**：jev-mcp 要求 Node ≥22；其 60s 请求超时与有限重试仅内建于 fetch 型传输（OpenRouter/Cloudflare/**compatible**）——本方案走 compatible 网关，超时/重试语义完整；TypeSafe 直连与 Vercel 走 SDK、无超时/重试一致性保证。

---

## 4. TOP 3 最值得优先落地的优化点

按 投入小 × 收益高 × 风险低 选出。每个点给出五元组：**在哪一步调用 / 判断什么 / 输入 / 返回 / 如何改变后续执行路径**。

### TOP 1：tdd-loop L1 task Verify 快门（对应 S3 + B3/B7/B8）

**位置**：`plugins/cospowers-tdd-loop/skills/cos-tdd-coding/SKILL.md` controller 的 task Verify 环节（现 L51 spawn l1-task-verifier 之前），新增脚本 `scripts/l1_fast_gate.js`。

**让 Jev 判断什么**：
1. （层0，纯脚本，不用 Jev）outputs 文件存在性、changed-files ⊆ writeScope 集合包含（前缀包含的确定性判定可抄 `validate_tasks.js` 的 writeScope 覆盖实现）、gate.required 与 checker manifest 一致性；对 `kind=scoped` 的单测门**亲自执行 `testScope.argv` 并以退出码写 fragment**（这正是现 verifier 的机械取证方式）——单测门的 pass 结论永远来自真实执行，不交给 Jev 判断；
2. （层1，`jev_verify`）executor 报告的关键声明 vs 证据的一致性：
   - claims（由脚本从 executor 产物模板化抽取，无需 LLM 组装）：
     - 「RED 阶段的失败输出全部来自位于 testScope.testPaths 内、本 task 新增的测试文件」
     - 「GREEN 后 testScope 内测试全部通过」
     - 「本 task 的 outputs 已在 writeScope 内产出且非空」
     - 「验证报告逐条覆盖 acceptance 与 testRequirements」
   - evidence：RED/GREEN 命令原始输出、changed-files 清单、task 验收标准文本。

**输入**（jev_verify 参数，脚本组装）：`{ claims: [上述 4–6 条], evidence: [{id:"red-log", text:...}, {id:"green-log",...}, {id:"changed-files",...}, {id:"task-acceptance",...}] }`，`auto_accept: 0.9`（收紧起步：高于 jev 默认 0.8；P2 校准后再评估放宽）。

**返回**：per-claim `{verdict: verified|contradicted|unsupported, confidence, action: auto|review}` + summary。

**如何改变执行路径**：
- 全部 claim `verified + auto` 且层0 机械项全过 → `l1_fast_gate.js` 写出该 task 该 round 的全部 checker fragments（结构为布尔校验器形态 `{checker, pass, findings[,runId,round]}`——verdict-fragment.schema.json 仅 checker 必填、另有可选 metrics/score；与聚合 gate/noRun 分支的「脚本写 fragment」既有先例同构；checker 名取自冻结 `testScopes.checker`（顶层单值、全 task 共享；init 默认写 `unit-tests`，`unit-tests-example` 是插件示例名），不得硬编码）→ controller 跑 `pev verdict` 照常求值 → **跳过 l1-task-verifier 的 LLM spawn**，直接 record-task；
- 任一 `contradicted` → 预填失败归因进 task envelope 的 previousFragments，直接进入 RETRY（省一次「verifier 发现问题→record→重派」往返）；
- 任一 `review` / jev 异常 / task 处于 RETRY 轮 → 照旧 spawn l1-task-verifier（原路径，全量重做）。

**必须连带的合同修订（核实补充）**：
1. `cos-tdd-coding SKILL.md:51` 与 `l1-task-verifier.md:17` 的契约红线——「声明式 `pass:true` 只允许来自冻结的 `noRun` 或冻结的 `kind=none`，否则属契约违规」——需扩展出第三种合法来源（快门：真实执行命令的退出码 + jev 声明核验）；机械项未实际核验就写 pass 仍属违规。
2. pev verdict 对同名 checker 的**冲突证据直接 NO_VERDICT**：快门与 LLM verifier 不能都写同一 `<checker>.json`——走快门就不 spawn verifier，反之亦然，快门路径必须独占该 round 的 fragment 写权。
3. task 循环是 SKILL prose 驱动的（无代码强制 spawn verifier），上述修订生效即改文档；record-task 只认 envelope 冻结的 resultPath + proof 校验，快门产出走原 `pev verdict → record-task` 链即不破互锁（核实确认：pev 对 round 目录内格式合法的 fragment 不校验写者，「脚本写 fragment」在聚合 gate 与 noRun 分支本就是既有先例）。

**兜底与质量增量**：L1 收口的全量聚合 checker（unit-tests.sh）仍然只跑一次且必须过；`pev verdict`/proof 互锁不动；审计抽样 10% 的 fast-gate-pass task 由 LLM verifier 复核。额外收获：RED 归因从 LLM 自证变为独立核验（修 B8 质量洞），预期减少假 GREEN 进入后续环节。

**可行性先例**：现架构已存在「非 LLM 产出 fragment」的合法通道——task envelope 带 `noRun:true` 时 verifier 本就被要求直接写声明式 `pass:true` fragment（cos-tdd-coding SKILL.md task 循环第 3 步）；且 verifier 对 `kind=scoped` checker 的取证方式就是「执行 argv、以退出码写 fragment」，本身可脚本化。快门只是把该通道扩展到「机械取证 + jev 声明核验」，不改变 fragment 消费方（pev verdict）的任何合同。

**投入**：1 个脚本 + 上述两处合同文本修订（SKILL:51 / verifier prompt:17）+ `node --test` 合同测试（mock jev 响应，覆盖 auto/review/contradicted/超时四分支）。注意 tdd-loop 现有测试**无 HTTP mock 先例**（唯一 mock 是 `t.mock.method(fs,...)`），需新建 mock 模式：node:test 的 `t.mock` 覆盖 jev 客户端的发送函数，或注入本地假端点。

### TOP 2：评审分片 Jev 预筛 + 增量复评（对应 S1+S2 + B1，先落 feature-design 再横向复制）

**位置**：feature-design evaluator 协调者在 spawn reviewer 分片之前（`evaluate-feature-design` SKILL L15；该位置已有「机械门未过不启动语义评审」的脚本先例——L14 的 check-design plan/output 即 FD-PRE），新增 `scripts/prescreen-checks.mjs`。TR1/TR2 是**按各自分片合同重新设计，不是同模式复制**（核实修正）：TR1 的分片派发在 requirement-evaluator SKILL L28（L22 是清单划分），派发为 prose + `evidenceAnchors`、无 inputRefs 结构，五分片的 assignedCheckIds 写死在 SKILL prose 里；TR2 的 Reviewer 派发在 verify-protocol「并行 Reviewer」节，Reviewer 自行按协议读文件（分片集合倒是机器可读：`review-partitions.json`）。两者的检查→输入映射都需先物化为机器可读清单才能预筛。

**让 Jev 判断什么**：
1. （层1，`jev_verify`）「证据可定位」类检查子条目是否被文档证据满足。预筛只处理可断言化的子条目（存在性/覆盖性/引用完整性/与冻结映射表的一致性），约占 checklist 的 40–60%；语义综合类（方案合理性、DFX 权衡）留给 LLM reviewer；
2. （层0，纯脚本）修复轮的 inputRefs sha256 比对：文件未变的检查项直接复用上轮结论（移植 overall-design 的 inputHash 机制到 feature/TR1/TR2）；
3. （层1 辅助，`jev_noul` 或 `jev_verify` 单 claim）「本次修复改动是否波及引用了未变文件的检查项」——防增量复评漏检传递影响。

**输入**：`{ claims: [子条目断言，如「ch02 §3 接口清单覆盖 ICD 里全部 REQ-03x 接口」「每条 Warning finding 的 file 字段都在产物白名单内」], evidence: {text: 文档片段} }`。**证据切片的数据源（核实修正）**：只有 feature-design 具备结构化切面——Plan inputs 是 `{path, section}` 且被 `check-design.mjs plan` 校验；但「检查 → inputRefs」映射目前是 review.md 的 Markdown 表，需先物化为结构化清单。按检查域分组批量调用（jev_verify 的 claims 无数量上限）。

**返回**：per-claim verdict/confidence/action + summary（verified/contradicted/unsupported 计数）。

**如何改变执行路径**：
- `auto + verified` 条目从 reviewer 分片工作清单剔除，evaluator 汇总时计入 checks pass（details 引用 jev evidence 切片，可追溯）；
- `contradicted` 条目转为**预填 finding 候选**（problem/evidence 已带），LLM reviewer 只确认 severity 与 route；
- 分片条目剔除后若整片清空 → 该 reviewer spawn 整体取消；
- 修复轮：sha256 未变条目复用 + jev 波及复核通过 → 分片只复查 delta；
- 任何 `review`/jev 异常 → 该分片全量回 LLM（fail-open）。

**兜底**：severity/dimension/route 归类与计分永远走 LLM + `check-design.mjs report` 重算（现状不变）；`contradicted` 不直接定罪，只作为 finding 候选交 LLM 确认；审计抽样 10% auto-pass 条目回炉。**报告覆盖与文档修订（核实补充）**：报告 schema 要求全量 checkId 覆盖（feature 侧 `check-design.mjs report` 的 CHECK_COVERAGE 校验；TR1/TR2 同样要求全集），被预筛剔除/复用的条目仍须在报告中产出 pass 记录——TR1 的「分片 `passedCheckIds` 直接写 `{id,passed:true}`、不得要求补充依据」与 overall 的「复制旧 review 改 round」是两个可抄的先例通道；「完整复评」文字需一并修订（feature SKILL L16 与 `cos-feature-design/SKILL.md:99`），TR2 侧「不维护增量复审」（L72）与绑定指纹（L74）文字**无测试守卫**（tests/ 目录 grep 零匹配）；其 `contracts.test.mjs:78` 守卫的是 verify-protocol L15 global Reviewer 的 PRE/SEM/红线覆盖结构，改动若波及该行才需连守卫测试一起改。

**投入**：feature-design 1 个 prescreen 脚本 + 检查→输入映射物化 + evaluator/execute SKILL 各几行 + inputHash 复用表（机制整体移植 overall-design：`mechanical-review.mjs` 的 hashFiles → `finalize reuse()` 比对复制 → `review-execution.json` 记 `strategy:"incremental"`/`reusedReviewIds`）；合同测试复用现有 contracts.test.mjs 模式。TR1/TR2 推广时的已知差异（核实修正）：TR2 测试已硬编码版本号（3.2.14）；**TR1（requirements）的测试只有相对一致性断言、无字面版本号**，如需硬编码风格要新增；TR2 有绑定指纹约束——候选文件任何修改即整 review 失效（`check-evaluation.mjs --bind` 写入 context/tr2 哈希 + verify-protocol:74），增量复用粒度只能到 review 级，且 finding severity 被脚本强制等于 quality-rules.json 冻结值；TR1 无任何 hash 基础设施，「五分片未全部完整不得归并」硬关卡（SKILL:40）需一并修订。

### TOP 3：knowledge-search 蒸馏去 LLM 化（对应 S4 + B5，投入最小的试点）

**位置**：`plugins/cospowers-master/skills/daedalus-knowledge/SKILL.md`「### 3. 评估命中」（L54-63，相关性判断；核实修正：原稿误写「步骤 4」）。蒸馏合同（**每类**最多 5 条、文件 ≤80 行）不在该 SKILL，而在 requirements/system-requirements/overall-design/micro-design 四插件的 `agents/knowledge-search.md`——这些 agent 不经 frontmatter 预载 skill，运行期按项目 `.cospowers/cospowers.config.json` 的 `knowledgeArchive.skill`（默认 `cospowers-master:daedalus-knowledge`，sync-template 缺失补齐时写入）经 Skill 工具加载。改动面 = SKILL 一段 + 4 份 agent 合同 + 新增 `distill-knowledge-hits.mjs`（落点遵循 3.2 决策 1 的「各插件副本」惯例，消费方是各阶段插件的编排侧）。

**让 Jev 判断什么**：检索命中与当前类别意图的相关性排序 + 可采性三分类。

**输入**（两步批量调用）：
1. `jev_rerank`：`{ query: "<类别意图>（如：历史方案中与〈需求关键词〉同类特性的设计决策）", candidates: [{id: 命中ID, text: 标题+摘要}, ...≤20] }` → 取 top 8；
2. `jev_classify`：`{ purpose: "知识命中可采性判定", items: top8, classes: [ {id:"directly_usable", description:"与类别意图直接相关，结论可直接引用（注明来源）"}, {id:"needs_adaptation", description:"相关但需结合本需求改写或部分采用"}, {id:"irrelevant", description:"关键词误命中，与本类别意图无关"} ] }`。

**返回**：排序 + per-item classification/margin/confidence/decision。

**如何改变执行路径**：
- `directly_usable + auto` 条目由脚本直接生成 team-knowledge-context.md 条目（保持 ≤5 条/≤80 行上限与来源格式）；
- `needs_adaptation` / `review` / jev 异常 → 才 spawn knowledge-search agent，且只处理这 ≤3 条（原 LLM 蒸馏链路保留为兜底）；
- `irrelevant` 丢弃。各阶段插件的 knowledge-search agent 从「每阶段必然 1 个 LLM spawn」变为「通常 0 spawn、偶尔小 spawn」（核实修正：master 自身无 agents，不在此链路上）。

**兜底**：distill 脚本输出带原文链接，主流程消费时可回查；jev 不可用 → 完整回退原 LLM 蒸馏路径。

**投入**：单脚本 + skill 一段改写 + 合同测试。**建议作为第一个落地项**：结构化最彻底、风险最低，同时验证内部 jev 网关链路与 usage 记账，为 TOP 1/2 的阈值校准积累「Jev vs 主模型」一致率数据。

### 顺手 Quick Wins（不占 TOP 位，随 TOP 1 一并做）

- **auto-test framework 路由查表脚本化**（S7）：映射表 3 行（auto-test SKILL L36-39）且 framework 值 init 时已冻结（cos-tdd-init 写入 `levelInputs.L2.execute.framework`、`level_manager.js` 每个 L2 execute 轮注入 action；L1 coding 不经 framework 路由）；`validate_execute_context.js` 现返回 `{valid, errors, capability}`、不含 framework——需扩展其返回值带上目标引擎名（mcp→auto-test-mcp / rf→auto-test-rf / claw→auto-test-claw），auto-test SKILL 删掉 LLM 查表步骤；
- **task envelope 切片**（B4）：`l1_task_manager.js` 的 taskEnvelope 生成时把本 task 定义（含其 testScope）内联进 envelope，executor/verifier 不再读全量 tasks.json。注意（核实补充）：tasksDigest 是**全文档 canonical-JSON 的 sha256**（`validate_tasks.js` 的 `digest()`），不是原始文件字节 hash——校验逻辑不变的前提下，任何自算 digest 必须复用 `validate_tasks.js` 导出的同一 canonical 函数；
- **micro/TR2 伪机械检查真脚本化**（B6）：锚点格式/孤儿需求/占位符/index 链接等先脚本化（参照 check-design.mjs output 子命令），剩余语义项才进层1/层2。

---

## 5. 量化收益（估算模型 + 度量方案）

### 5.1 估算（参照场景：中型特性全流程——8 阶段 + 30 个 L1 task + 1 个 L2，修复率 ~30%）

| 指标 | 优化前（基线估算） | 优化后（TOP1–3 落地） | 变化 | 依据 |
|---|---|---|---|---|
| 主 LLM 子代理 spawn 数 | 阶段评审 ~60–75（首轮 ~34 + 修复轮全量重跑）+ L1 verifier ~39 + knowledge-search 8 + 其余 | 评审 ~40–50（预筛剔条目/整片 + 增量复评）+ L1 verifier ~13–16 + knowledge-search ~0–2 | **↓ ~35–45%** | S1 条目剔除率 40–60%、S2 复用率 60–80%、S3 快门通过率按 60–70% 顺利 task 估 |
| 主 LLM Token 消耗 | 100%（评审面为最大单项，其次 executor 编码面） | 评审面 ↓40–60%、verifier 面 ↓60–70%、executor 面不变、tasks.json 重复读消除 | **总体 ↓25–40%** | executor（不可替代）占 L1 大头，拉低总体比例 |
| 端到端时长 | 100% | — | **↓20–35%** | 主要省评审分片串行等待、verifier spawn 往返、RETRY 误判轮 |
| Jev 调用次数 | 0 | 判断条目 ~150–250 条/流程；API 调用 ~60–100 次（verify/classify 均批量） | 新增 | 每次自带 usage 记账 |
| 总模型成本 | 100% | 主 LLM ↓25–40% + jev 增量（<1 美元/流程量级，占比 <5%） | **↓25–40%** | jev 单次远低于 1 美分 |
| 测试用例通过率 / Verified Completion Rate / 任务成功率 | 基线 | 持平（设计目标：不降） | **0 或微升** | fail-open + 终审不变量不动 + RED 归因独立核验预期减少假 PASS 假 GREEN |

> 以上为基于调用点勘察的估算模型，**不是实测值**。上线按「试点 → 抽样校准 → 全量」推进，每步用 5.2 的度量数据修正。

### 5.2 度量方案（让上表变成实测值）

1. **jev 侧**：`jev_client.mjs` 每次调用落 `.cospowers/jev-usage.jsonl`（时间/场景/工具/条目数/verdict 分布/input_tokens/output_tokens/延迟）——token 数由 jev 每个结果自带的 usage 直接采集（核实修正：usage 只有 token 两字段、无延迟），延迟由客户端计时补记；
2. **LLM 侧**：各编排脚本在 spawn/record 处累计计数（核实修正：现状没有 spawn 计数字段——tdd-loop 的 runRoot/state.json 只有 rounds/attempts，属新增仪表，落点沿用 state.json 扩展字段即可），对照 run 记录主 LLM spawn 数与轮次；
3. **质量侧**：沿用现有 final-report/evaluation.json，新增「jev-auto 结论 vs 终审结论」偏差率字段；审计抽样的结果单独落 `jev-audit.jsonl`；
4. **A/B**：同一需求集跑 开关前/后（`JEV_DISABLED=1` 即基线路径），比较 5.1 全部指标。

---

## 6. 落地路线与风险清单

| 阶段 | 内容 | 出口条件 |
|---|---|---|
| P0 前提 | 内部 jev 兼容网关确认/搭建；依赖入内网 registry；密钥方案 | 网关连通性测试通过；`jev_client.mjs` 合同测试绿 |
| P1 试点 | TOP 3（knowledge 蒸馏先行）+ Quick Wins；`JEV_DISABLED` 开关默认关 | 50–100 条历史样本 Jev vs 主模型一致率 ≥95%（auto 档）；试点阶段产物质量指标持平 |
| P2 校准 | 用审计数据调 auto_accept/margin 阈值；偏差率超限场景自动降级 | 两周稳定运行，偏差率 <2% |
| P3 横向 | TOP 2 复制到 TR1/TR2；S5 Discovery 预筛；S8/S9 二期评估 | 同 P1 出口条件 |

**风险与对策**：

| 风险 | 等级 | 对策 |
|---|---|---|
| Jev 误判 pass 且未被兜底发现 | 高影响低概率 | 三道兜底（聚合 checker/机械门重算/审计抽样）+ 阈值收紧起步（0.9，高于 jev 默认 0.8）+ 偏差率超限自动停用层1 |
| 断言化模板（S1/S3）覆盖偏差——预筛错杀本应 fail 的条目 | 中 | contradicted 不定罪只预填候选；auto-pass 条目可追溯（details 带 evidence 切片）；审计回炉 |
| 增量复评漏检传递影响（S2） | 中 | jev 波及复核 + 修复轮至少复查 delta + 终审机械门不变 |
| jev 服务不可用/延迟抖动 | 低 | fail-open 到层2，效率回退质量不降；60s 超时 + 有限重试（jev-mcp 内建，仅 fetch 型传输——compatible 在覆盖范围内，TypeSafe/Vercel 直连不保证） |
| 数据出境合规 | 高（前置） | P0 内部网关硬前提；jev_client 出网白名单只指向内部端点 |
| 双宿主行为漂移 | 低 | 脚本直调形态天然双宿主一致；合同测试进两套测试运行器（node --test / py） |

**明确不做**：不用 Jev 替代 severity 归类终审、writer/integrator/executor 生成、grilling 交互、pev proof 与用户确认；不让主 Agent 在 skill 提示词里「自主决定」何时调 jev MCP 工具（判断点全部脚本化锚定，保持机械消费结构化状态的仓库约定）。
