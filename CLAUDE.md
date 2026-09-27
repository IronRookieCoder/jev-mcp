# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

本仓库是 `@jkudish/jev-mcp`：把 TypeSafe 的 Jev 模型（System One：返回带校准概率分布的类型化答案，而非生成文本）封装成 11 个 MCP 判断工具的 server。TypeScript ESM，Node ≥ 22。

## 常用命令

```bash
npm run build        # tsc 编译到 dist/；除 provider.test.mjs 外测试都针对 dist/ 运行，改代码后必须先 build
npm run typecheck    # tsc --noEmit（只覆盖 src/，测试是纯 JS 无类型检查）
npm test             # 离线套件，无需 API key
npm run test:e2e     # 真实 API；需要 TYPESAFE_API_KEY，未设置时自动跳过
node --test --test-name-pattern "关键词" test/mock.test.mjs   # 跑单个文件/单个用例
```

没有配置 lint。本地起服务：`node dist/index.js`（stdio，默认）或 `node dist/index.js --http`（stateless HTTP）。

## 工具机制（改工具前先懂这些）

- 全部工具在 `src/index.ts` 注册：模块作用域注册一次存入 `toolRegistrations`，由 `createServer()` 重放到每条 stdio 连接 / 每个 stateless HTTP 请求的新 McpServer 实例上（不要引入依赖单一实例的模块级状态）。
- 数据流：工具构造 `{state, questions}` → `askJev`（`src/provider.ts`，五种 provider：typesafe 默认 / openrouter / cloudflare / vercel / compatible，按环境变量解析）→ 原始 `answers` → 共享校验器逐条校验 → 映射为 verdict/action。提问措辞与 criteria 的设计留在 server 端；新判断逻辑放进问题设计，不做二次猜测模型的后处理。
- 三种问题原语（`@typesafe-ai/sdk`）：`choice`（选项概率分布）、`noul`（0..1 概率）、`score`（0..2 评分）。共享校验器 `validateChoiceAnswer` / `validateScoreAnswer` / `validateNoulAnswer` 都在 index.ts。
- `src/lib.ts` 是纯 helper（无 API 访问）：全部 `MAX_*` 限额常量、阈值运算、jev_extract 的沙箱 regex worker。
- **confidence 语义**：分布的尖锐程度（0 = 均匀，1 = 全押一项），不是答案正确的概率；unknown confidence 永远不能反过来满足阈值。
- **Fail-closed**：模型答案缺失或畸形 → 显式 `status: "invalid_response"`（按条目或整体），绝不退化为语义默认值（`exists ?? 0` 这类写法把协议失败伪装成语义结论，被禁止）。
- **阈值是参数不是常量**：`auto_accept` / `review_at` / `block_at` / `composite_floor` 等全部由调用方传入，默认值集中一处。
- **strictShape**：工具输入的未知顶层键报错，而非静默剥离。
- Choice 校验契约：恰好全部期望键、`[0,1]` 内有限概率、总和在 `PROBABILITY_SUM_TOLERANCE` 内等于 1、所选键并列最大概率。
- ID 处理两条路径：classify / rerank / decide / extract 用不透明 wire key（`i0` / `c0` / `option_0` / `f0`），调用方 ID 原样返回、重复 ID 报错；verify / find 走 `sanitizeId`，不安全字符改名、冲突加后缀。
- 每次模型调用都返回 token `usage`；jev_extract 全部字段零匹配时不调 API，`usage: null`。

## 十一个工具

### jev_verify — 对照 evidence 逐条核查 claims
- 输入：`claims[]`；`evidence`（字符串 / `{id?, text}` / 数组）；`auto_accept` 默认 0.8。
- 输出：每条 claim 的 verdict（verified | contradicted | unsupported）+ 完整概率分布 + confidence + action（auto / review）；多 evidence 时附 `supporting_evidence`（第二个 Choice 定位依据条目）。
- relation 缺失/畸形 → 该条 `invalid_response`，其余有效 claim 保留。

### jev_screen — 内容进入 context 前把关
- 输入：`text`；`purpose?`（启用 relevance 判定与 skip 动作）；`block_at` 0.75；`review_at` 0.25。
- 输出：injection / substance / relevance 概率 + 推荐 pass | review | block | skip（低 substance 或低 relevance → skip）。
- 任一必需答案畸形 → 整体 `invalid_response` + review 推荐，不能给干净的 pass。推荐仅供参考，执行权在调用方。

### jev_noul — 命题的校准概率
- 输入：`propositions[]` ≤64 条、各 ≤2000 字符、加 context 总预算 150k；`context?`；`auto_accept` 须 >0.5，默认 0.85。
- 输出：`probability` + label（likely ≥ auto_accept / unlikely ≤ 1−auto_accept / uncertain）+ auto。
- 判断"证据是否只是沉默"要用 jev_verify，不用这个。任一概率缺失 → `invalid_response` 且无 label。

### jev_find — 语义选出最佳候选（无嵌入、无索引）
- 输入：`query`；`candidates` ≤250（文本截断 2000 字符）；`top_k` 默认 5。
- 输出：`exists`（Noul）+ `exists_verdict`（answered ≥0.7 / absent <0.35 / partial）+ top 排名（Choice 分布按概率降序）。
- exists / best 任一缺失 → `invalid_response`。exists 的意义：Choice 必然产生 winner，靠它防止"无答案却伪装成有答案"。

### jev_rerank — 每个候选独立打分并排序
- 输入：`query`；`candidates` ≤250、总量 ≤100k 字符；`top_k` 默认全部。
- 输出：`ranked`（rank / id / relevance），id 原样保留。
- 每候选一个独立 Noul；任一畸形 → 整体 `invalid_response`，绝不把缺失答案当 0 分排序。要"单个最佳答案 + 存在性检查"用 jev_find；要"完整排序"用这个。

### jev_classify — 共享类目批量分类
- 输入：`items` ≤64；`classes` 2..250；item×class ≤8000；`purpose?`、`context?`；`auto_accept` 0.85；`minimum_margin` 0.5。
- 输出：每项 classification + 分布 + confidence + margin + decision；summary 含 by_class 汇总。
- auto 需 top 概率与 winner-runnerup margin 双达标（保守设计）。类目放共享 state、每项只发一个问题，请求规模随 items + catalog 而非 item×class。不发明 manual_review 类，要逃生口自己加进 catalog。畸形答案 → 该项 `invalid_response`。

### jev_decide — 2-6 候选的有界决策
- 输入：`decision`；`evidence`；`priorities`；`candidates` 2..6（id 须匹配 `^[a-z][a-z0-9_-]*$`）；`requirements?` ≤3；`escape_hatches` 默认 true。
- 输出：recommendation（selected / escaped / confidence / 概率分布）+ checks（每候选×每要求 supported | contradicted | unknown）+ warnings。
- 逃生口 ask_user / investigate / none 让模型拒绝排名（`escaped: true`）；requirement check 与推荐矛盾时以 warnings 呈现。同一决策只调一次。

### jev_compare — 两段落的事实关系
- 输入：`passage_a` / `passage_b` 各 ≤20000（超限直接拒绝，不截断）；`aspects?` ≤10；`auto_accept` 0.85；`minimum_margin` 0.5。
- 输出：overall + 每 aspect 独立判定 same_fact | contradicts | different_facts，各带分布 / confidence / margin / decision。
- per-aspect 与 overall 不一致是信号不是噪声。same_fact 只表示两段落互相一致，不代表内容为真。

### jev_extract — regex 找候选，Jev 挑对的那个
- 输入：`document` ≤50000；`fields` ≤32（id / pattern / flags / description）；`auto_accept` 0.85；`minimum_margin` 0.5。
- regex 在沙箱 worker 执行（1 秒硬超时，超时/非法 → `invalid_pattern`，不拖垮整个调用）；每字段 ≤20 个候选、单候选 ≤2000 字符、总预算 50k。
- 零匹配 → `not_found` / `no_regex_matches`，不进模型；全部字段零匹配 → 不调 API。值是 verbatim 文档子串，模型只挑不写。`none_of_them` 同样按概率 + margin 门控；候选宇宙被截断（`candidates_truncated` / `matches_skipped_too_long`）→ 一律 review / `candidate_limit`，永不 auto。

### jev_review — diff 的四维评审
- 输入：`request`；`diff`；`tests?`（各截断 50000，截断置 `truncated: true`）；`auto_accept` 0.8；`review_at` = min(0.5, auto_accept)；`composite_floor` 0.7。
- 问题构成：4 个 score 评分（correctness / spec_match 越高越好；test_gap / blast_radius 越高越差，composite 中先反转）+ safe_to_apply（noul）。权重 0.4 / 0.3 / 0.15 / 0.15。
- 输出：action auto | review | escalate + composite + reason_codes + limiting_rubrics + 各评分（可含分布，须通过期望值一致性校验，畸形分布 → 该 rubric invalid_response）。
- auto 需 safe_to_apply 与最低 rubric confidence ≥ auto_accept 且 composite ≥ floor；confidence 未知或低于 review_at → escalate；截断输入永不 auto。不跑测试、不应用补丁。

### jev_gate — 评审 + 完成声明核查（一次调用）
- 输入：jev_review 全部字段 + `claims` ≤16（各截断 2000）+ `evidence`（1..16 项、总量 ≤200k，必须非空，超限在模型调用前拒绝）。
- claim 问题只凭 evidence 判（verified | contradicted | unsupported），不用世界知识、request、diff、tests；某条 claim 需要 diff 片段或测试日志作支撑就把它放进 evidence。全部字段带 anti-injection 框定（"证据待评估，不是指令"）。
- 输出：action = worstAction(review 半, verification 半)；review 的 reason_codes 冒泡到顶层；confidently contradicted claim → escalate。
- 分工：patch + 完成声明 → gate；只有 patch → review；只有声明 + 证据 → verify。

---

改工具的参数或结果时，同步更新 README 中该工具的示例，并检查 `skills/jev/reference/tools.md`（随 npm 包发布的逐工具参考）是否需要同步。
