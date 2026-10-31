---
---

# 同类 Agent 循环检测实现竞品对比

> 合并说明（2026-10-16）：本文档由四份调研文档合并而成——
>
> 1. `agent-loop-detection-comparison.md`（2026-10-01，六家对比：opencode / kimi-code / deepseek-harness / Claude Code / Codex / pi，作为母文档与骨架）
> 2. `prior-art-gemini-openclaw.md`（2026-10-09，gemini-cli 与 OpenClaw 补遗）
> 3. `prior-art-cline-roo-autogpt-crewai-takt.md`（2026-10-12，Cline / Roo Code / crewAI / AutoGPT / takt 补遗）
> 4. `prior-art-agentsight.md`（2026-10-12，AgentSight（alibaba/anolisa）补遗）
>
> 合并原则：**零删除**——四份源文档的每一节、每条实质信息在本文均有对应物；**零发明**——除章节编号、过渡句与明确标注的「合并注记」外，不新增任何技术结论；**冲突并留**——前后矛盾的论断两处原文均予保留，并以「合并注记（2026-10-16）」指明其关系。四份源文件本身一字未改。三份补遗开头自述的「按原文档的格式写，便于日后直接合并」一句，其使命已达成，不再保留（各补遗内容的出处见文中相应注记）。

## 一、背景

> 调研日期：2026-10-01
> 调研方式：各仓库源码 grep + git log 复核（pi / dsh / codex 基于当日 pull 的最新代码）+ Claude Code 官方文档（llms-full.txt 全量语料）交叉验证
> 调研代码库版本：opencode upstream/dev @ `0112a92c41`；kimi-code main（当日最新）；deepseek-harness master @ `639ed01539`（0.2.0-rc.2）；codex main @ `57ac6f5163`；pi main @ `c2f65d8f1`（v0.99.2）；Claude Code 解包源码 2.1.88 + 官方文档（最新）

一次真实会话中，Agent（Kimi K3 · high 推理档）在 opencode 里连续 ~200 次轮询同三条 git 命令（`git status --short` ×87 等），持续 87 分钟、烧掉 56.8M cache-read tokens，期间用户两次打断均无效。由此引出调研：各家 Agent harness 如何防「重复工具调用死循环」。

## 二、循环的时序模式：从混沌收敛到节拍器

把循环区 200 次调用编码为符号（S=`git status --short` ×85、D=`git diff <file>` ×53、T=`git diff --stat` ×49、V=写法变体 ×13）后，时序呈现清晰的三个相位：

```
[0..49]    VSDVTDVDTVVSVVTVDSVDSDVVDSTVDSVDSTDSTDSTDSTDSTDSTD
[50..99]   STDSTDSTDSTDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDST
[100..149] SDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSD
[150..199] STSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDSTSDST
```

| 相位 | 范围 | 结构 | 特征 |
|---|---|---|---|
| 混沌期 | call#0–30（约 10 分钟） | 无周期 | **全部 13 个变体都集中在这 31 次调用里**；模型在「换花样自证」 |
| 周期 3 | call#31–63 | `D→S→T` ×11 | 第一次收敛出稳定循环，33 次调用零变体 |
| 周期 4 | call#64–199（**占循环 68%**） | `S→D→S→T` ×**34** | 136 次调用的节拍器，持续一个多小时；用户两次打断都落在这一段，周期连相位偏移都没有 |

三个关键事实：

1. **循环的熵在下降**——变体全部集中在开头 15% 区段，之后归零。模型不是越转越乱，而是越转越「整齐」，最终收敛到极限环（limit cycle）。对检测的含义：周期 4 相位里根本不存在变体逃逸，streak 类检测抓不到它纯粹是因为**轮换**。
2. **周期从 3 变 4 的方式是多插了一个 S**——转移矩阵 S→D:36 / S→T:46 双高，模型在已稳定的循环上又「加固」了一层无效验证。
3. **用户打断发生在周期 4 相位内部**，打断前后节奏无任何扰动——行为层面证明此时模型的行动已经和上下文输入脱钩。

## 三、补遗说明：对母文档两条全称判断的修订

> 本节内容来自补遗 `prior-art-gemini-openclaw.md` §0（2026-10-09）。

> 调研方式：本地源码实读（`D:\Proj\github\gemini-cli` HEAD `56da16d` 2026-10-08；`D:\Proj\github\openclaw` HEAD `dd0ecdd4` 2026-10-08）+ `gh api` 拉提交史
> 术语沿用已定词：信号 / 清零 / 警告 / 拦截 / 熔断 / 行级信息新颖度

原六家对比得出两条全称判断：

1. 「六家没有一家做工具输出 result 的指纹比对」
2. 「我们的方案若落地，是七家中第一个同时覆盖轮换 + 变体 + 输出语义的」

补进这两家后，**两条都不成立**：OpenClaw 把输出哈希做成了每一个计数器的门槛（证伪第 1 条），gemini-cli 用 k 周期匹配覆盖了轮换（削弱第 2 条）。但排查同时捞到一条更硬、且实测能站住的差异化，见 §十一.2（补遗 §6.1）。

样本还漏了 Cline、Roo Code、crewAI、AutoGPT classic、**nrslib/takt**（建在 opencode 之上，已用 `(tool, outcome, inputHash, resultHash)` 四元组做守卫）。takt 是同生态里最直接的同类工作，PR 描述必须引。

> 合并注记（2026-10-16）：本节证伪的两条全称判断，其**原文仍保留在本文 §四.1 的「关键空白」段与 §六的「对 kimi 评价的注脚」段**，两处均未改写；其修订依据见本节与 §十一.2。第三批补遗（`prior-art-cline-roo-autogpt-crewai-takt.md` §0）进一步指出：takt 与 OpenClaw 已有「输出哈希进检测口径」的先例，「输出逐字节相同才累计」必须改写为组合主张，见 §五.14 与 §十一.3。

## 四、总览矩阵

矩阵按三批调研分三张表呈现（列太宽，拆表但信息不丢）：表 1 为母文档六家；表 2 为 gemini-cli 与 OpenClaw 新增两列（出自补遗 `prior-art-gemini-openclaw.md` §1）；表 3 为第三批五家与本方案（出自补遗 `prior-art-cline-roo-autogpt-crewai-takt.md` §1）。AgentSight 依其补遗 §0 的自述「不需要进对比矩阵」，未列入（它是进程外可观测组件，见 §五.15）。

### 4.1 表 1：第一批六家（母文档）

| 维度 | opencode | kimi-code | deepseek-harness | Claude Code | Codex | pi |
|---|---|---|---|---|---|---|
| 专门检测器 | ✅ doom loop | ✅ Repeat Breaker | ✅ repeat-tool-reminder | ❌ | ❌（goal 模式有停滞熔断） | ❌ |
| 判定口径 | 同工具+输入 JSON 完全相同 | 工具名+规范化参数 sha256 | 工具名+key 排序参数 | —（无指纹） | —（按轮次有无产出） | — |
| **比对输出 result** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| 检测范围 | 同一 assistant 消息内 | **跨 step 连续 streak** | 链式连续（跨调用） | — | goal 续跑轮次 | — |
| 阈值 | 3 | 3 / 5 / 8 / 12 分级 | 3 / 5 / 8 分级 | — | 空转/失败各连续 3 轮 | — |
| 触发动作 | permission.ask 问用户 | 分级警告→强制停 turn | 注入提醒（绝不拦截） | — | goal 置 blocked | — |
| 硬熔断 | ❌（"always" 可永久绕过） | ✅ streak≥12 强制停 | ❌ | 仅 maxTurns 被显式设置时 | ✅（仅限 goal 模式） | ❌ |
| 用户新消息清零 | — | —（未提及） | ✅ 清零链 | — | — | — |
| 被拒调用也计数 | — | —（未提及） | ✅ | — | —（Guardian 统计拒绝本身） | — |
| 步数/轮数上限 | agent.steps（默认 ∞，达限只注入提示） | maxStepsPerTurn（默认 ∞，可配置） | ❌ 无（goal/ralph 有 round 预算） | maxTurns 家族（默认 ∞） | ❌ 无（靠 token 压缩） | ❌ 无 |
| 可整体关闭 | `continue_loop_on_deny` 实验开关 | env `KIMI_CODE_REPEAT_BREAKER=false` | 插件可禁用 | — | — | — |

**关键空白**：六家没有一家做「工具输出 result 的指纹比对」——所有实现都只比对调用（工具名+参数）。这是「相同工具+相同参数+**相同输出**连续 N 次」判定方式的增量空间。

> 合并注记（2026-10-16）：此「关键空白」段及「比对输出 result」行是 2026-10-01 第一批调研（六家）的结论；2026-10-09 补入 gemini-cli 与 OpenClaw 后，该全称判断被证伪（见 §三，原文两处均保留）。表 2 新增两列即当时的修订增量。

### 4.2 表 2：第二批新增两列（gemini-cli / OpenClaw）

| 维度 | gemini-cli | OpenClaw |
|---|---|---|
| 专门检测器 | ✅ `LoopDetectionService`（三层） | ✅ 6 个检测器（`tool-loop-*.ts` 家族） |
| 判定口径 | `sha256(工具名 + ":" + JSON.stringify(参数))`，**无 key 排序** | 工具名 + 参数哈希 + **结果哈希 resultHash** + 失败等价哈希 failureIdentityHash |
| **比对输出 result** | ❌ 一二层不比；第三层由 LLM 语义判断（能看到工具返回内容） | ✅ `sha256(stableStringify(结果))`，且是**每个 streak 的累加门槛** |
| 检测范围 | 跨调用，最近 25 条（`5 × 5`）；文本层最近 5000 字符 | 跨调用，最近 30 条，按 `runId` 分域 |
| 阈值 | 周期检测：k=1..5 各需连续重复 5 遍；文本块 10；LLM 层满 30 轮后每 5~15 轮自适应，置信度 ≥0.9 | 警告 10 / 拦截 20 / 全局熔断 30 / 未知工具 10 |
| 触发动作 | 第一次命中：注入系统反馈、开新 turn 继续（**不打扰用户**）；第二次命中：抛 `LoopDetected` 掐断流并问用户 | 警告：追加到工具结果；严重：挡下整批工具调用；同一次运行内第二次严重：结束本次运行 |
| 硬熔断 | ✅ 第二次命中掐流 | ✅ 拦截档 20 + 全局熔断 30 |
| 用户新消息清零 | ✅ **硬清零**（`client.ts:954` → `reset()` 清 toolCallHistory / 文本 / LLM 计数 / detectedCount） | 按 `runId` 分域，跨运行不继承 |
| 被拦下的调用继续计数 | —（掐流后无后续调用） | ✅ `outcomeKind: "tool-loop-veto"` 记入历史，`pendingLoopVetoes` 累加进 streak |
| 参数校验失败（没真跑）的调用 | — | ✅ 单独归类 `argument-validation`，**既不延长也不终止**失败尾部 |
| 步数/轮数上限 | `boundedTurns`（每次恢复递归 −1） | 另有轮次预算 |
| 可整体关闭 | ✅ `getDisableLoopDetection()` 配置项 + `disableForSession()` 会话级 | ✅ 但 #111382（2026-07-19）已把数值旋钮从公开配置里砍掉 |

> 合并注记（2026-10-16）：本表出自补遗 `prior-art-gemini-openclaw.md` §1「总览矩阵（新增两列）」，原即设计为并入母文档矩阵，此处如原样并入。「被拦下的调用继续计数」与表 1 的「被拒调用也计数」为同一维度，合并时并为一行理解；第一批六家在该维度的取值仍见表 1 末几行。

### 4.3 表 3：第三批五家与本方案（Cline / Roo Code / crewAI / AutoGPT / takt）

> 调研方式：本地源码实读 + HEAD hash 核实——Cline `95e54e4`（2026-10-08）；Roo Code `b867ec9`（2026-05-15）；crewAI `42ae4bf`（2026-10-08）；AutoGPT `0edeafc`（2026-10-12，浅克隆）；takt `11c18844`（2026-10-12，浅克隆）
> 术语沿用已定词：信号 / 清零 / 警告 / 拦截 / 熔断 / 行级信息新颖度

| 维度 | Cline | Roo Code | crewAI | AutoGPT (platform) | takt | 我们方案 |
|---|---|---|---|---|---|---|
| 专门检测器 | ✅ `LoopDetectionTracker`（新 SDK 架构） | ✅ `ToolRepetitionDetector` | ⚠️ 半个：`_check_tool_repeated_usage`（阈值 1 的相邻查重） | ⚠️ 失败熔断器，不算通用循环检测 | ✅ 10 个 guard 的套件，其中 3 个管循环 | ✅ 三信号 |
| 判定口径 | `工具名 + JSON.stringify(sortKeys(参数))`（**有 key 排序**） | `safe-stable-stringify({name, params, nativeArgs})`（**有 key 排序**） | `sanitize(工具名) + 参数` 浅比较 | `(工具名, json.dumps(args, sort_keys=True))` | **`(tool, outcome, inputHash, resultHash)` 四元组**，域分隔 sha256 + 递归 key 排序 | A 工具+归一化参数；B 输出行；C 工具+失败等价哈希 |
| **比对输出 result** | ❌ | ❌ | ❌ | ❌（只比「失败」这一 outcome） | ✅ resultHash 是四元组一员，**输出不变 streak 才续** | ✅ A 要求逐字节相同才累计 |
| 检测范围 | **仅相邻连续**，不同调用即清零 | **仅相邻连续** | **仅相邻 1 次**（跟上一次比） | **仅相邻连续**（按签名，成功即全清） | **仅相邻连续**（单 lastKey） | 按签名独立计数，穿插不清零，无窗口 |
| 阈值 | 警 3 / 拦 5 | 3（复用 mistake 上限） | 1（相邻相同即触发） | 失败 3；空调用 5 | 精确重复 12；不可用工具 2；无效参数 4；edit 冲突 3；连续错误 10 | A 警3/拦6；B 3/5/8/12；C 警4/拦6 |
| 触发动作 | 警：注入 user 角色消息自救；拦：强制计满 mistake 并停 run，等用户发消息恢复 | 拦下本次调用，**阻塞式问用户**，压入 tool 错误结果 | 不执行，把「别再重复」文案当工具结果喂回 | 拦下执行，返回 STOP 文案当工具结果；空调用熔断直接掐流 | 掐断 SSE 流 → 纠偏 prompt（每指纹 1 次）→ 全新 session 重试 → fail | 权限询问，always 按签名粒度，熔断停 run |
| 用户新消息清零 | ❌ 不清（只在压缩/restore 时清，见 §5.9） | ❌（计数器被拦后自清；用户引导后模型换调用自然清零） | —（只记上一次） | ❌ | —（无用户交互，按 attempt 生命周期清零） | 不清（反对硬清零） |
| 可配置 | ✅ 阈值 + 整体关闭 | ✅ `consecutiveMistakeLimit` | `max_iter` 等 | 环境变量 | ✅ profile（standard/minimal）+ 环境变量 | 3/5/8/12 拟可调 |
| 兜底 | `maxConsecutiveMistakes` 默认 6 | 同左（默认 3，阻塞问用户） | `max_iter` 25 → 强制收尾 | 每 turn 100 轮 + $10 预算 + 空闲 30min 超时 | 字节量/事件数/超时资源 guard | — |

> 合并注记（2026-10-16）：本表出自补遗 `prior-art-cline-roo-autogpt-crewai-takt.md` §1，原表即含「我们方案」列，照原样保留。原表中「只在压缩/restore 时清，见 §2.3」的交叉引用在本文中对应 §五.10（Cline 细节 · 清零语义）。「—」均沿用各源文档原文。

## 五、逐家细节

> 合并注记（2026-10-16）：本节先列母文档原有六家（§5.1–5.6，编号与内容保持原样），再接第二批 gemini-cli / OpenClaw（§5.7–5.8，出自 `prior-art-gemini-openclaw.md` §2–§3，保留其原小节号 2.x / 3.x），第三批 Cline / Roo Code / crewAI / AutoGPT / takt（§5.9 为总述、§5.10–5.14 为五家，出自 `prior-art-cline-roo-autogpt-crewai-takt.md` §2–§6，保留其原小节号），最后为 AgentSight（§5.15，出自 `prior-art-agentsight.md` §1–§4）。

### 1. opencode —— doom loop（有缺陷的检测）

- `packages/opencode/src/session/processor.ts:29`：`DOOM_LOOP_THRESHOLD = 3`；`:353-380`：同一 assistant 消息内最近 3 个 parts 同为 tool、同名、输入 JSON 完全相同 → `permission.ask({ permission: "doom_loop" })` 问用户是否继续。
- UI 文案：`src/cli/cmd/run/permission.shared.ts:111-117`「⟳ Continue after repeated failures」。
- **缺陷一**：检测范围限同一 assistant 消息内——跨消息每轮一个调用的循环永远凑不齐 3 连。
- **缺陷二**：参数要求逐字节相同——加 `echo "---"` 等装饰变体即逃逸。
- **缺陷三**：触发后走 permission.ask，用户对某工具点一次 "always" 即永久放行（`src/permission/index.ts:143-163` 批准缓存），自动化场景等于无兜底。
- 步数上限：`src/session/prompt.ts:1178` `maxSteps = agent.steps ?? Infinity`（默认无上限），达限仅注入 `MAX_STEPS_PROMPT`（`packages/core/src/session/runner/max-steps.ts`）让模型自己收尾，非硬退出。
- 实验开关 `experimental.continue_loop_on_deny`（processor.ts:647）：true 时被拒也继续跑。

### 2. kimi-code —— Repeat Breaker（六家中最完善）

- 实现：`packages/agent-core-v2/src/agent/toolDedupe/toolDedupeService.ts`（阈值常量 :63-66，分级逻辑 :536-552）。
- 判定：工具名 + 规范化参数（`canonicalTelemetryArgs`）sha256 签名，**跨 step** 统计连续相同调用 streak。
- 分级干预（全部以 system-reminder 注入工具结果）：
  - streak ≥ 3：警告「同一调用已重复多次」，要求先写一句「期望获得什么新信息」再行动；
  - streak ≥ 5：要求三选一——(1) 证伪检查 (2) 向用户索要缺失输入 (3) 基于现有证据收束；
  - streak ≥ 8：要求立即停止调用工具、输出纯文本总结（阻塞点 + 已尝试方案 + 需要用户提供什么）；
  - **streak ≥ 12：强制停止当前 turn**（stopReason=`repeat_breaker`），随后进入纯文本 handoff 步，该步内任何工具调用一律 veto（`HANDOFF_VETO_RESULT`），强迫模型只输出文本收尾。
- 同 step 内重复调用：直接 veto 并复用首个调用的结果（去重，`checkToolCall()` :450-455）。
- 开关：env `KIMI_CODE_REPEAT_BREAKER=false`（默认开启）。
- 其他层：maxStepsPerTurn（默认 ∞，`human/agent/machine.ts:750`，可经 `loop_control.max_steps_per_turn` 或 env 配置）；每步 LLM 重试上限 10（`human/llm/requester/retry.ts`）；print 模式 steer 轮次上限。

### 3. deepseek-harness —— repeat-tool-reminder（只提醒不拦截）

- 实现：`packages/guard/repeat-tool-reminder/src/index.ts`（阈值 :53，检测 :201-204，canonicalize :96-112）。
- 判定：(tool, 规范化参数) 链键，参数深 key 排序后 stringify；同键连续重复计数，不同调用清零。
- 阈值 [3,5,8]：首档温和提醒，后续详细提醒（列工具名、连续次数、参数预览截断 500 字符）。
- 设计哲学明确「observe-and-enrich, **never veto**」（:216-220），显式 DELEGATE 让下游仍可 block。
- 两条值得借鉴的细节：**被 deny 的调用也计数**（:193-196 注释「model hammering a denied call is exactly the loop worth breaking」）；**用户新消息清空链**（:236-239）。
- 存储：每 agent 独立链 `WeakMap<Agent, Chain>`，子 agent 不与父共享。
- 配套：timeout-policy（单次工具调用超时返回结构化错误）；goal/ralph 自治场景的 round 预算（maxGoalRounds / maxRounds 256，部署配 64）；主循环 `agent.ts:254` `while (await this.turn()) {}` 无步数硬上限。
- 最新 master 复核：无行为变化。

### 4. Claude Code —— 无通用检测，间接多层兜底（文档交叉验证）

- 解包源码 2.1.88 结论：无 (tool, input, result) 指纹比对；靠提示词约束（`prompts.ts:189`「用户拒绝后不要重试同一调用」、`:233`「不要盲目重试相同动作」）+ maxTurns（`query.ts:1704`，默认不设）+ token 收益递减早停（`query/tokenBudget.ts:59-62`：续写 ≥3 次且相邻两次输出增量 <500 tokens → 停）+ 防压缩震荡守卫（`query.ts:1292`）+ Edit/Write 单次调用级陈旧拦截。
- 官方文档（最新）交叉验证：**仍无通用重复调用检测**；官方 issue 反向印证——[anthropics/claude-code#67812「Claude Code does not bound repeated identical tool calls」](https://github.com/anthropics/claude-code/issues/67812)（同一 call id 重复上千次无人拦截）至今 open。
- 2.1.88 之后新增的「特定路径熔断器」（均非调用内容重复检测）：

  | 机制 | 版本 | 内容 |
  |---|---|---|
  | Stop hook 8 次帽 | v2.1.143 | Stop/SubagentStop hook 连续阻止结束 8 次 → 强制结束 turn；`CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` 可调（0 关闭） |
  | Autocompact thrash 熔断 | v2.1.89 | 压缩后立刻又填满连续 3 次 → 停止（`rapid_refill_breaker`） |
  | 扇出限额 | v2.1.212 | WebSearch 每会话 200 次上限（"stop runaway search loops"）；subagent 派生深度限 3（`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`） |
  | maxTurns 家族 | — | CLI `--max-turns`（仅 print 模式）、`CLAUDE_CODE_MAX_TURNS`、SDK `max_turns`/`max_budget_usd`（超限 `error_max_turns`）、subagent frontmatter `maxTurns`（partial 标记 v2.1.246+） |
  | 重试耗尽型终止 | — | SDK terminal_reason：`malformed_tool_use_exhausted`、`structured_output_retry_exhausted`、`budget_exhausted` 等 |

### 5. Codex —— 通用层零检测，goal 模式有停滞熔断（最新 main 复核）

- 通用层：`core/src/session/turn.rs:612` 注释原文「as long as compaction works well ... we shouldn't worry about being in an infinite loop」——靠 token 压缩兜底，不设循环上限。
- Guardian 熔断器（`ext/guardian-reviewer/src/circuit_breaker.rs`）：每 turn 连续审核拒绝 3 次（Cyber 模型 1 次）、或近 50 次窗口内拒绝 10 次 → InterruptTurn 中断 turn。统计的是**审核拒绝**这一元信号，不看调用内容。
- 新增（2026-08/09，goal 自治模式专属）：`ext/goal/src/accounting.rs:143-230`——连续 3 轮空响应（无工具调用/文本/推理摘要）或连续 3 轮执行失败 → goal 置 blocked，停止自动续跑。按「轮次有无产出」计数，仍非调用内容比对。
- 其他：goal token/时间预算（BudgetLimited）；Stop hook 扩展点；prompt 层软提示「avoid unnecessary repetition」。

### 6. pi —— 零检测，框架层留扩展点（最新 main 复核）

- 主循环 `packages/agent/src/agent-loop.ts:162+`：只按「无工具调用即停」「工具结果全部 terminate 即停」、宿主 `finishTurn` 返回 end 结束。README 明确承认宿主回调使用不当会死循环（`README.md:154`）。
- 局部守卫：LLM 重试上限 3（指数退避封顶 60s）；上下文溢出恢复仅 1 次（`agent-session.ts:355` `_overflowRecoveryAttempted`）；驱动状态机「无进展」不变量（`drive.ts:101-104`，防内部自旋非模型行为）。
- 扩展点：`finishTurn`、`beforeToolCall` 钩子可供宿主自建熔断，仓库内无实现。

### 7. gemini-cli（补遗第二批）

> 出自 `prior-art-gemini-openclaw.md` §2。主文件 `packages/core/src/services/loopDetectionService.ts`，781 行。

#### 2.1 第一层：工具调用周期检测（`:313-349`）

```
key = sha256(工具名 + ":" + JSON.stringify(参数))          // :174-178，无 key 排序
历史 = 最近 25 条 key（5 × TOOL_CALL_LOOP_THRESHOLD）      // :317-320
R = 5                                                      // :29
for k in 1..5:                                             // :326
    需要 k × 5 条历史
    取最后 k 条当作周期核
    校验最后 k×5 条是否严格等于周期核重复 5 遍              // :332-340
    命中即返回 true
```

要点：

- **是精确的连续匹配**，一个变体插进来就把整段匹配打断。
- **不看输出**，只看工具名+参数。
- 参数用裸 `JSON.stringify`，属性顺序变了就算不同签名 —— 这是它的弱点，本方案的归一化能补。
- 命中后 `loopDetected` 置位并**锁存**（`:192-198`），后续事件直接返回上次结果，直到 `reset()` 或 `clearDetection()`。

#### 2.2 第二层：文本复读（`:362-533`）

流式文本按 `CONTENT_CHUNK_SIZE = 50` 切块哈希，同一块在短距离内反复出现即判循环（阈值 `CONTENT_LOOP_THRESHOLD = 10`，历史上限 `MAX_HISTORY_LENGTH = 5000`）。两条防误报措施值得记：

- **代码块内跳过并清零**（`:359-360` 注释、#5144）—— 重复的代码结构不是循环。
- **遇到新的 Markdown 元素类型就清零**（#5820）—— 跨元素边界的重复不算。
- 哈希命中后还要做一次**原文逐字比对**（`isActualContentMatch`，`:534-543`，注释「prevents false positives from hash collisions」）。

#### 2.3 第三层：LLM 判官（`:563-697`）

- 满 30 轮后启用（`LLM_CHECK_AFTER_TURNS`，`:42`），每 5~15 轮自适应复查（`:48-60`，`updateCheckInterval` 按置信度调间隔：越像循环查得越勤）。
- 喂进去的是最近 20 轮完整对话（`LLM_LOOP_CHECK_HISTORY_COUNT`，`:37`）**含工具返回内容** —— `trimRecentHistory`（`:545-561`）只掐掉尾部悬空的 function call 和头部悬空的 function response，中间的 functionResponse 全保留。
- 同时喂**原始用户请求**（`:577-588`），提示词明说：如果用户要的就是批量操作，那么「参数在变的重复调用」应当强烈倾向于不判循环（`:99-101`）。
- 置信度 ≥0.9 才算命中（`LLM_CONFIDENCE_THRESHOLD`，`:65`），命中后还要**第二个模型复核**（`DOUBLE_CHECK_MODEL_ALIAS`，`:66`）。
- 输出是结构化 JSON：`unproductive_state_analysis` + `unproductive_state_confidence`（`:103-118`）。

#### 2.4 两级升级（`client.ts:843-886`、`:1300-1328`）

| 命中次数 | 动作 |
|---|---|
| 第 1 次（count 1） | `clearDetection()` 清标志但**保留 detectedCount**（`:1308`，注释「the count remains 1」），注入系统反馈、`boundedTurns - 1` 递归开新 turn。**不叫人** |
| 第 2 次（count > 1） | 抛 `GeminiEventType.LoopDetected`，**直接掐断流**（`:844-847`） |

注入的反馈原文（`:1310`）：

> `System: Potential loop detected. Details: {detail}. Please take a step back and confirm you're making forward progress. If not, take a step back, analyze your previous actions and rethink how you're approaching the problem. Avoid repeating the same tool calls or responses without new results.`

#### 2.5 用户逃生口（`packages/cli/src/ui/components/LoopDetectionConfirmation.tsx:39-52`）

只有两个选项：

- `Keep loop detection enabled (esc)` —— **默认就是 esc，即保持防护**
- `Disable loop detection for this session` —— 作用域只到本会话（`disableForSession()`，`:166-172`）

三点全都和 opencode 的缺陷三反着做：升级前先自救 / 逃生口默认关闭 / 作用域限会话，语义是「关掉这个检测器」而不是「永久允许这个工具」。

#### 2.6 系统提示词里的「什么不算循环」分类（`:81-101`）

这是本次调研最值钱的意外收获，直接可当误伤语料规格，已单独整理成 [误伤语料.md](../design/误伤语料.md)。

### 8. OpenClaw（补遗第二批）

> 出自 `prior-art-gemini-openclaw.md` §3。`src/agents/tool-loop-*.ts`，8 个实现文件 + 4 个测试文件。

#### 3.1 输出指纹是怎么用的

| 文件 | 内容 |
|---|---|
| `tool-loop-outcome-hash.ts:4-23` | `digestToolOutcome(value)` = `sha256(stableStringify(结果))`，即 **resultHash**。序列化前把外部不可信内容的信封 ID 规范化成全零（`:8-20`），避免同一结果因 ID 不同被算成两个 |
| `tool-loop-no-progress.ts:79-81` | 每次 streak 累加都要求 `record.resultHash === latestOutcome.resultHash`，**输出变了立即 break** |
| `tool-loop-no-progress.ts:39-47` | 不同工具名 `continue`（不打断）、不同参数哈希 `continue`（不打断）、只有输出变了才 `break` —— **穿插其它调用不影响计数，这就是轮换免疫** |
| `tool-loop-no-progress.ts:9-14, 55-78` | `exec` 工具额外走「终端失败尾部」通道，用 `failureIdentityHash` 判「换了参数但失败方式相同」，即**参数漂移的失败循环也算同一个循环** |
| `tool-loop-no-progress.ts:35-38` | `argument-validation`（参数被拒、根本没跑）**既不延长也不终止**失败尾部 |
| `tool-loop-no-progress.ts:48-51, 61, 82` | `tool-loop-veto`（被检测器挡下的调用）记为 `pendingLoopVetoes`，**累加进 streak** —— 与本方案 `repeatBlocked` 的设计独立撞到同一个答案 |

`getNoProgressStreak` 这个函数与本方案信号 A 同构：按签名独立计数 + 输出一致才算重复 + 允许穿插。

#### 3.2 窗口与阈值 —— 一个结构性 bug，值得抄教训

`tool-loop-detection.ts:47-50`：

```
TOOL_CALL_HISTORY_SIZE           = 30   // 检测窗口只保留最近 30 次调用
UNKNOWN_TOOL_THRESHOLD           = 10
CRITICAL_THRESHOLD               = 20
GLOBAL_CIRCUIT_BREAKER_THRESHOLD = 30
```

窗口截断在 `:620-622` 和 `:677-679`（`splice` 只留最后 30 条）。

`exec` 不属于 poll 工具（`tool-loop-call-kind.ts:3-12`，只有 `command_status`、`wait`、`process` 的 poll/log 动作算），所以走 `:568` 的 `!knownPollTool && noProgressStreak >= 20` 拦截分支。但窗口只有 30 条，轮换会稀释每个签名的计数：

| 相位 | 30 条窗口内单签名出现次数 | 结果 |
|---|---|---|
| 周期 3（D→S→T） | 约 10 | 够警告档（10），**不够拦截档（20）** |
| 周期 4（S→D→S→T） | 约 15 | 够警告档，**不够拦截档** |
| 全局熔断（30） | 单签名在 30 条窗口内最多 30，轮换下 ≤15 | **结构上不可达** |

**结论：OpenClaw 对本次事故会反复弹警告，但拦不下来，而它的全局熔断在轮换场景下是死代码。** 这不是推测，是常量除出来的。

规律：**窗口大小必须 ≥ 阈值 × 轮换周期，否则拦截档永远够不着。**

#### 3.3 六个检测器

| detector | 管什么 | 阈值 |
|---|---|---|
| `no_progress` | 同参同输出重复 | 警告 10 / 拦截 20 |
| `ping_pong`（`:422-446, 471, 532-558`） | **k=2 交替**（A B A B），要求两侧都有稳定输出 | 警告 10 / 拦截 20 |
| `argument_churn`（`tool-loop-argument-churn.ts`） | 同工具多个参数变体轮换，但**所有变体必须共用同一个 resultHash**，且每个变体至少稳定出现 3 次（`MIN_STABLE_CALLS_PER_VARIANT`） | 警告 10 |
| `unknown_tool`（`:475-485`） | 反复调用不存在的工具 | 10 |
| 全局熔断（`:486-500`） | 同签名无产出累计 | 30 |
| compaction 后循环守卫 | 压缩后立刻重复（`embedded-agent-runner/post-compaction-loop-guard.ts`，默认开启） | — |

`argument_churn` 的限制条件（`tool-loop-argument-churn.ts:23-25, 40-44`）值得注意：**所有变体必须共用同一个 resultHash**，一旦某个变体输出不同就 break；`:33-36` 的注释还明说「新参数是逃离循环的可能出口，必须清零活跃度证据而不是继承它」。本次事故里 `git status --short` / `git diff <file>` / `git diff --stat` 三者输出各不相同，所以这条对它无效。同理 `ping_pong` 只管 k=2，管不了周期 3 和周期 4。

**所以 OpenClaw 唯一能咬到本次事故的是 `no_progress`，而它被 30 条窗口稀释到只够警告。**

#### 3.4 阈值哲学的反转

`tool-loop-thresholds.ts:1-7` 只有一个数 10，注释：

> `Numeric loop tuning was retired in #111382. Keep every admission path on the same built-in threshold so policy rewrites cannot drift from detection.`

查证：#111382 = 「refactor(config): config-surface reduction tranche 2 — purge tuning knobs behind built-in defaults」，**2026-07-19 合并**，正文写明这一批砍掉 209 个生成的配置路径，理由是「未设置时的内置默认值本来就稳定，暴露成公开配置只是白白加宽 schema / 文档 / 生成元数据 / 测试 / 长期兼容面，却没给运维者有意义的产品选择」。

也就是：他们 2026-02-16 先做了可配置数值（`076df941 feat: add configurable tool loop detection`），五个月后主动砍掉、收敛到硬编码 10。本方案 §7 把「3/5/8/12 可调」当优点，方向与业界最成熟的实现相反。

### 9. 第三批总述：一句话结论

> 出自 `prior-art-cline-roo-autogpt-crewai-takt.md` §0（2026-10-12）。

五家里四家只有「相邻 N 次完全相同」的计数器（Cline / Roo / crewAI / AutoGPT 各一种），全部不比对工具输出、全部被轮换天然免疫；**takt 是唯一把 `(tool, outcome, inputHash, resultHash)` 输出哈希做进检测口径的**，传闻属实，但它仍是「相邻连续」streak，没有按签名独立计数，也没有行级新颖度和失败等价哈希——我们的三条核心主张里，「输出逐字节相同才累计」在 takt（和 OpenClaw）已有先例，必须改写成组合主张；另外两条在五家中无撞车。

Roo Code 仓库状态须如实记录：经 `ls-remote` 核实其 `main` 自 2026-05-15 起五个月无提交，本文描述的是该快照的实现，不代表 Roo 当前线上能力。

### 10. Cline（`95e54e4`，2026-10-08）

> 出自 `prior-art-cline-roo-autogpt-crewai-takt.md` §2。仓库已重构为 `sdk/` + `apps/` 两层；检测器在新 SDK 内核里，旧版 apps/vscode 内联检测已不存在。

#### 2.1 `LoopDetectionTracker`（`sdk/packages/core/src/runtime/safety/loop-detection.ts`）

- 签名：`toolCallSignature()` = `JSON.stringify(sortKeys(input))`（`:40-59`），递归 key 排序——**gemini-cli 裸 stringify 的弱点在这里已修**。
- 口径：`toolName === lastToolName && signature === lastToolSignature` 则 `consecutiveIdenticalCount++`，否则重置为 1（`:72-81`）。**只记上一次调用，任何不同调用即清零**，轮换天然免疫。
- 阈值：软 3 / 硬 5（`DEFAULT_CONFIG`，`:113-116`），可经 `execution.loopDetection` 配置，传 `false` 整体关闭（orchestrator `:505`）。
- 不比对工具输出。

#### 2.2 干预（`session-runtime-orchestrator.ts:1455-1495`）

| 档位 | 动作 |
|---|---|
| soft（第 3 次） | 把 `Detected N consecutive identical calls ... consider trying a different approach.` 以 **user 角色消息**注入会话，让模型自救，不打断 |
| hard（第 5 次） | `enqueueMistakeRecord({ forceAtLimit: true })` 直接把 MistakeTracker 顶到上限，走 mistake-limit 通道停 run（`finishReason: "aborted"`） |

VS Code 侧（`apps/vscode/src/sdk/sdk-interaction-coordinator.ts:74-94`）：停 run 后显示错误行「Cline ran into N errors in a row and stopped the task…Send a message to give Cline guidance」，**不阻塞询问**；用户发消息即恢复。另有独立的 `MistakeTracker`（连续错误，默认上限 6，orchestrator `:471`）。

#### 2.3 清零语义

`loopTracker.reset()` 只在 `resetConversationBoundaryTrackers()` 里被调（`:604-609`）——即压缩（clearHistory）与 restore 两条路径。**新用户消息不清零**，与 gemini-cli 相反、与本方案相同。

### 11. Roo Code（`b867ec9`，2026-05-15，main 已五个月无提交）

> 出自 `prior-art-cline-roo-autogpt-crewai-takt.md` §3。

#### 3.1 `ToolRepetitionDetector`（`src/core/tools/ToolRepetitionDetector.ts`，89 行）

- 序列化：`safe-stable-stringify({name, params, nativeArgs?})`（`:76-88`），key 排序的规范化 JSON；不比输出。
- 只存 `previousToolCallJson`，相同则计数、不同则清零（`:40-45`）——**仅相邻连续**。
- 上限复用 `consecutiveMistakeLimit`，默认 3（`packages/types/src/provider-settings.ts:29`；`Task.ts:513`），可配置，0 表示不限制。
- 命中即**拦下本次执行**（`allowExecution: false`），并把计数器清零以便用户引导后恢复（`:52-54`）。

#### 3.2 命中后的流程（`src/core/assistant-message/presentAssistantMessage.ts:613-648`）

阻塞式 `ask("mistake_limit_reached", ...)` 问用户；用户给反馈则把反馈塞进 userContent；然后压入 tool 错误结果「Tool call repetition limit reached for {tool}. Please try a different approach.」并中断该 assistant 消息处理。第一次命中就叫人——正是 gemini-cli 两级升级所反对、也是我们批评 opencode 现状的那一点。

#### 3.3 其他相关机制

- 全局连续错误计数：`Task.ts:2483-2500`，≥3 阻塞问用户后清零。
- **按文件分的 diff 失败计数**：`consecutiveMistakeCountForApplyDiff` / `...ForEditFile`（`Task.ts:318-319`；`ApplyDiffTool.ts:84-119`、`EditFileTool.ts:174-365`）——同一文件连续失败则错误信息逐级加料，成功即删键。这是「按签名（文件）的失败计数」的雏形，与信号 C 同一思想家族，但仅限两个 diff 工具、仅相邻、目的是细化报错而非拦截。

### 12. crewAI（`42ae4bf`，2026-10-08）

> 出自 `prior-art-cline-roo-autogpt-crewai-takt.md` §4。

#### 4.1 `_check_tool_repeated_usage`（`lib/crewai/src/crewai/tools/tool_usage.py:779-789`）

```python
if last_tool_usage := self.tools_handler.last_used_tool:
    return sanitize(calling.tool_name) == sanitize(last_tool_usage.tool_name) \
       and calling.arguments == last_tool_usage.arguments
```

- **只跟上一次调用比**，阈值就是 1——相邻两次相同即触发，没有计数器、没有窗口、不比输出。
- 触发后**不执行工具**，把 i18n 文案当工具结果喂回（`tool_usage.py:254-265`，en.json:49：「I tried reusing the same input, I must stop using this action input. I'll try something else instead.」），并打 telemetry span（`telemetry.py:609`）。本质是「每次相邻复读都劝阻一次」，没有升级链、不拦 run。
- 两条调用路径（`:254` 与 `:510`）各有一份同样检查。

#### 4.2 兜底

- `max_iter` 默认 25（`base_agent.py:286-288`）：`has_reached_max_iterations` 命中后 `handle_max_iterations_exceeded` 追加一条强制收尾指令、再调一次 LLM 产出最终答案（`agent_utils.py:366-378`）——是「轮次预算 + 软着陆」，不是循环检测。
- 可选结果缓存 `cache=True`（`crew.py:232-241`，opt-in）：同参重复调用直接复用首个结果——消除重复执行的副作用，但不检测循环。
- 另有 per-tool `max_usage_count`、`tool_failure_policy`。

### 13. AutoGPT（`0edeafc`，2026-10-12；platform 的 copilot 为现役产品，classic 无任何重复检测）

> 出自 `prior-art-cline-roo-autogpt-crewai-takt.md` §5。

#### 5.1 失败熔断器（`autogpt_platform/backend/backend/copilot/sdk/tool_adapter.py:131-435`）

- 指纹：`_failure_key()` = `f"{tool}:{json.dumps(args, sort_keys=True)}"`（`:378-381`），有 key 排序。
- **只记失败**：`_record_tool_failure` 在执行出错时累加（`:413-419`）；同工具任意一次成功把该工具**所有参数变体全清**（`_clear_tool_failures`，`:422-435`，注释明说这是刻意给工具「fresh start」）。
- 阈值 `_MAX_CONSECUTIVE_TOOL_FAILURES = 3`（`:135`）。触发后 `_check_circuit_breaker` 在执行前拦下，返回硬文案作为工具结果（`:404-409`）：「STOP: Tool 'X' has failed N consecutive times with the same arguments. Do NOT retry this tool call.」——软拦截，不停 run。
- 存储是 `ContextVar`，按执行上下文隔离。
- 与本方案信号 C 的关系：**同参数失败计数**是撞车的，但它只认参数逐字节相同（无失败等价哈希），且「同工具任意成功全清」比我们的「穿插 read 不清零」激进得多（一个无关成功就解封）。

#### 5.2 空参数熔断（`copilot/sdk/service.py:272-285`）

连续 5 次空参数工具调用 → 直接掐断流并给用户报错。防的是上下文饱和/序列化故障，不是一般循环。

#### 5.3 预算兜底（`copilot/config.py:465-484` 等）

每 turn 工具轮次上限 100（`agent_max_turns`，`CHAT_AGENT_MAX_TURNS` 可调）、每查询 $10 预算、空闲 30 分钟 / 挂起工具 2 小时超时。无输出比对、无周期检测、无语义判定。classic（`classic/original_autogpt`）grep 无任何重复/循环检测，可写「无」。

### 14. takt（`11c18844`，2026-10-12）——四元组守卫核实，传闻属实

> 出自 `prior-art-cline-roo-autogpt-crewai-takt.md` §6。

#### 6.1 接入方式：wrapper，不是 fork，循环检测也不是 opencode 插件

takt 把 `opencode serve` 作为子进程拉起（`src/infra/opencode/server-process.ts:137`），经 v1/v2 transport 走 SDK HTTP + SSE 事件流驱动（`server-pool.ts` / `v1-transport.ts` / `v2-transport.ts`）。所有守卫都在 **takt 进程内、SSE 消费侧**观察事件，不改 opencode 本体。唯一注入 opencode 的插件是参数矫正器 `plugins/coerce-tool-args.ts`（经 `config.plugin` 传 dist 绝对路径，修上游 anomalyco/opencode#1328 等参数类型 bug），与循环检测无关。

#### 6.2 四元组（`src/infra/opencode/tool-call-tuple.ts`，103 行）

```
identityKey = tool + NUL + outcome + NUL + inputHash + NUL + resultHash
key        = normalize(tool) + NUL + outcome + NUL + inputHash + NUL + resultHash   // tool trim+lowercase
```

- `inputHash / resultHash` = `sha256(域分隔串 + canonicalize(value))`（`:58-74`）：递归 key 排序、拒绝非有限数/循环引用/class 实例（不可规范化返回 undefined，该调用不进守卫）。
- `outcome` 取 success 或 failure（被拒绝或 completed-with-error 算 failure，`guards/tool-outcome-ledger.ts:48-50`）。
- **`resultHash` 是对工具输出整体做哈希——「输出逐字节相同」正是 streak 延续的必要条件**，与我们信号 A 的口径同源。

#### 6.3 账本与守卫（`src/infra/opencode/guards/`）

- `ToolOutcomeLedger`（`tool-outcome-ledger.ts:29-69`）：按 callID 命名空间记录 input 哈希，终态事件到来时算 resultHash、产出四元组；同一 call 的矛盾终态更新记 anomaly。
- **`ExactRepeatStreakGuard`**（`integrity-guards.ts:133-168`）：单 `lastKey` 加 `streak`——**只有相邻两次四元组完全相同才累加，任何不同即归 1**。阈值 `OPENCODE_EXACT_TOOL_REPEAT_LIMIT = 12`（`policy.ts:21`，硬编码不可调）。注释明言主目标是**成功结果的复读**（错误复读由 ConsecutiveErrorsGuard 先接）。
- `ExactLoopGuard`（`integrity-guards.ts:34-131`）：三个错误侧子检测——不可用工具连续 2 次（`unavailable-tool-loop.ts:5`）、同工具无效参数连续 4 次（`unavailable-tool-loop.ts:70`，注释：弱模型曾 219 次连呼缺参 read）、**edit 冲突按 `sha256(filePath + oldString)` 签名累计 3 次**（`tool-guard.ts:21-23`）——后者是按签名的失败计数，与信号 C 同族但口径是输入而非失败等价性。任何成功或穿插文本即全清（`integrity-guards.ts:70-76`）。
- `ConsecutiveErrorsGuard`（`heuristic-guards.ts:13-88`）：连续错误大于等于 10，或最近 20 事件错误率大于等于 90% 触发 `tool_error_burst`。
- `CycleBudgetGuard`：连续 120 个 assistant 消息周期无成功工具调用触发 fail。
- 资源/完整性 guard：文本与 reasoning 字节量、事件数、ID 数、敏感值预算、不活动超时（`resource-guards.ts` / `time-guards.ts`）。
- **profile 分层**（`registry.ts:28-31` 加 `strategy.ts`）：standard 全量 10 个；**minimal 只留 mandatory，把 exact-repeat-streak、consecutive-errors、cycle-budget 三个循环相关守卫全部裁掉**。
- 生命周期：每次 attempt 开始 `start('attempt')` 重置各守卫计数（如 `integrity-guards.ts:45-51`）；无用户消息概念（takt 是自主编排器）。

#### 6.4 干预：不问用户，三级恢复阶梯（`tool-guard.ts:58-71`、`attempt-runner.ts:1471-1495`）

任一守卫 fail 则掐断 SSE 流，走 `resolveToolGuardRecoveryAction`：

1. **correction**：把针对性纠偏 prompt 续进同一 session（每指纹限 1 次，全局限 2 次；`buildToolGuardCorrectionPrompt` 按五种失败 kind 各写一段文案，`tool-guard.ts:111-148`）；
2. **fresh_session**：带「上一 session 已在同一 workspace 留下半成品，先重读文件」提示，全新 session 重试；
3. **fail**：放弃。

与 gemini-cli 两级升级同构（先注入自救、再升级），只是升级终点是新 session 而非问用户。

#### 6.5 与我们方案的对位

| 我们 | takt | 判定 |
|---|---|---|
| A：按签名计数 + 输出逐字节相同才累计 | resultHash 入四元组，输出不同 streak 即断 | **「输出一致才累计」撞车**；但 takt 是单 lastKey 相邻 streak，无按签名独立计数，轮换一击即断 |
| B：行级信息新颖度 | 无 | 不撞车 |
| C：失败等价哈希，穿插 read 不清零 | edit 冲突按输入签名计 3 次；成功即全清 | 部分撞车（按签名失败计数），无等价哈希、无穿插容忍 |
| 警告档 | 无（直接掐流加纠偏） | 不撞车 |
| 权限询问 / always 放行 | 无（自主场景不适用） | 不撞车 |

### 15. AgentSight（alibaba/anolisa）

> 出自 `prior-art-agentsight.md`（2026-10-12）。调研方式：本地源码实读（`D:\Proj\github\anolisa`，浅克隆，2026-10-11 推送的 HEAD）+ GitHub API。仓库事实：Apache-2.0，665 stars，创建于 2026-03-30，活跃（最近推送 2026-10-11）。

**一句话结论（其补遗 §0 原文）**：**AgentSight 是外挂式可观测组件（eBPF 守进程），不是 agent 运行时内检测器。**它确实做「重复工具调用 / 无推进」检测（`src/agentsight/src/interruption/loop_detector.rs`），但数据来自进程外抓取的 LLM 流量，干预手段是 **SIGTERM→SIGKILL 杀进程**，与我们「运行时内、给模型警告、权限询问放行」的路线不在同一层。**PR 同类工作一节值得一句话引用**（可观测层的代表做法），但不是直接对手，不需要进对比矩阵。

#### 1. 它是什么

ANOLISA（Agentic OS）的可观测组件，Rust 实现。eBPF 探针（sslsniff / proctrace / procmon）从内核抓 LLM API 明文流量与进程事件，解析成 GenAI 语义事件落 SQLite（`README.md` Architecture 一节；macOS 上退化为扫描本地 JSONL 会话文件）。agent 进程**零改动**——这是它和我们方案的根本分野：它站在进程外看，我们站在工具执行点拦。

#### 2. 循环检测实现（`src/agentsight/src/interruption/loop_detector.rs`，877 行）

每次新的 LLMCall 事件落库后，从 SQLite 取该会话**最近 10 条调用**（`window_size: 10`，`unified.rs:1752-1756`），跑三条规则（按优先级，`loop_detector.rs:127-165`）：

| 规则 | 判定口径 | 阈值 | 位置 |
|---|---|---|---|
| 1. 工具序列重复 | 最近 5 条**带工具调用的** LLM 调用，发出的工具序列（工具名 + 参数 JSON 哈希）完全相同 | 连续 5 次 | `:175-207` |
| 2. 输出相似循环 | 最近 3 条**带文本的** LLM 调用，输出文本两两 Jaccard ≥ 0.85（CJK 走 bigram） | 连续 3 次 | `:213-257` |
| 3. token 空烧 | 输入 token 严格递增 + 输出文本相似 | 连续 3 次 | `:264-311` |

关键事实：

- **比较的是 LLM 的产出（assistant 文本 + 发出的工具调用），不是工具返回结果**。`RecentCallSummary.output_text_snippet` 是 LLM 输出文本前 ~200 字符（`:80-81`），工具执行结果根本不进入检测输入。
- 规则 1 要求**连续**相同（`tail` 全等，`:192-198`）——A B A B 轮换直接绕过，与 kimi/dsh 同病。
- 参数指纹是 `serde_json::to_string` 的 DefaultHasher，**无 key 排序**（`:65-71`，注释自认顺序不同的等价参数哈希不同，「只会更保守」）。
- 检测去重：auto-kill 关闭时每会话最多记 1 条 DeadLoop 事件（`unified.rs:1746-1749`）。

#### 3. 触发后干什么（`interruption/escalation.rs`，181 行）

DeadLoop 事件严重度 Critical（`types.rs:139`）。升级阶梯：同一会话 DeadLoop 计数到 `kill_after_count`（**默认 3**，`config.rs:1267`）→ SIGTERM 优雅杀；超过 → SIGKILL（`escalation.rs:20-23`）。**没有给模型的警告通道**——agent 没有任何自我纠正的机会，要么被观测到，要么被杀。

#### 4. 与我们三信号的对照

| 维度 | AgentSight | 本方案 |
|---|---|---|
| 检测位置 | 进程外（eBPF 抓流量 / 扫会话文件） | 运行时内（工具 execute 层） |
| 判定输入 | LLM 输出文本 + 发出的工具调用 | 工具调用输入 + **工具返回结果** |
| 输出证据 | ❌ 不看工具结果（规则 1 只比调用本身） | ✅ 信号 A 的输出逐字节证据是防误伤核心 |
| 轮换免疫 | ❌ 连续相同才计数 | ✅ 按签名独立计数 |
| 变体免疫 | ❌ 参数逐字节哈希 | ✅ 信号 B 行级新颖度不看写法 |
| 相似度判定 | ✅ Jaccard 0.85 模糊匹配（我们没有） | 逐字节 / 行级精确匹配 |
| token 空烧信号 | ✅ 输入递增 + 输出相似（我们没有） | ❌ |
| 干预 | SIGTERM→SIGKILL（默认第 3 次检测后） | 警告注入输出 → 权限询问 → 熔断 |
| 模型自救机会 | ❌ 无 | ✅ 每档警告都给出路 |
| 文本/推理循环 | ✅ 规则 2/3 正好覆盖 | ➖ 我们划界出范围 |

## 六、各家方法能否规避本次循环？——全部不能

> 合并注记（2026-10-16）：本节标题与以下「四个反检测特征」「逐家判定」表为母文档（2026-10-01）原文。2026-10-09 补遗 gemini-cli / OpenClaw 后，该判断需修订——补遗原文「**结论：「六家全部拦不住」这张表必须改。** 但卖点不必推翻 —— 见 §6.1 实测口径」保留于本节下方；补遗的实跑结果（gemini-cli 能拦、OpenClaw 只警告不拦截）已并入下表。第三批补遗的 §0 另指出 Cline / Roo / crewAI / AutoGPT 四家「全部被轮换天然免疫」、takt 为相邻 streak「轮换一击即断」，其原文见 §五.9，此处不为其逐行追加判定。

本次 87 分钟循环有四个「反检测」结构特征：

1. **跨消息**：每条 assistant 消息只发 1 次工具调用，循环横跨 200 条消息；
2. **轮换**：`git status` / `git diff <file>` / `git diff --stat` 三条命令周期性轮换，而非单命令连发；
3. **变体装饰**：周期性插入 `&& echo "---"`、追加 `git log` 等写法变体（区段内 200 次调用共 9 种原始写法）；
4. **每轮都有"产出"**：每次调用都成功返回结果——不是空转，不是失败。

逐家判定（kimi 的 streak 语义经源码精读确认）：

| 代码库 | 能否拦截 | 败因 / 依据 |
|---|---|---|
| opencode（doom loop） | ❌（实证未拦住） | 特征 1：只看同一 assistant 消息内，跨消息每轮 1 次调用永远凑不齐 3 连；特征 3 进一步逃逸逐字节比对 |
| kimi-code（Repeat Breaker） | ❌ | 特征 2：streak 是全局「最近签名」连续计数（`toolDedupeService.ts:374-383, 523-533`，换签名即清零），A B C 轮换下恒为 1，连首档警告（≥3）都到不了；代码中虽有按签名独立的频率统计（`turnCallRecords` :174-177, 409-437），但仅用于遥测上报，不接任何干预动作 |
| deepseek-harness（repeat-tool-reminder） | ❌ | 特征 2：链式规则明文「不同调用清零为 1」，轮换即清零；且仅提醒不拦截 |
| Claude Code | ❌ | 特征 4：无调用级检测；token 收益递减早停看输出 token 增量，循环中模型每轮长篇 reasoning+新调用，不触发；maxTurns 默认不设 |
| Codex | ❌ | 特征 4：goal 停滞熔断要求连续 3 轮空响应/执行失败，循环每轮都有成功产出；通用层压缩兜底只能控制上下文大小，不能停循环 |
| pi | ❌ | 全特征：零检测，唯一终止条件是「模型不再发起工具调用」，而循环恰恰不会自己停 |
| gemini-cli | ✅ **能，在 call#45** | （补遗第二批）已实跑（§六.1）。k=3 精确周期匹配命中 |
| OpenClaw | ❌ **只警告不拦截** | （补遗第二批）已实跑（§六.1）。call#43 出警告，单签名最大 streak 实测 **15** < 拦截档 20，全局熔断 30 结构上不可达 |
| 本方案 B（行级信息新颖度 ×8） | ✅ call#12 | （补遗第二批）已实跑 |
| 本方案 A（按签名相同输出 ×6） | ✅ call#21 | （补遗第二批）已实跑 |

> 合并注记（2026-10-16）：表头「败因 / 依据」为合并处理——母文档原表列为「败因」，gemini-openclaw 补遗 §4 原表列为「依据」，两表已并为一；gemini-cli / OpenClaw / 本方案 A、B 四行出自该补遗 §4 原文。

**结论：「按签名独立计数 + 输出一致性」不是锦上添花，而是必要设计。** 三家有检测器的（opencode/kimi/dsh）全部把 streak 挂在「连续相同」（全局最近签名或同消息窗口）上，而真实死循环几乎从不单命令连发——模型会本能地换着花样取证，轮换是常态。必须把计数挂在每个签名自己头上，允许中间穿插其它调用；再叠加输出一致性判定，检测才能既激进（3 次警告）又不误伤合法重跑。

**对 kimi 评价的注脚**：Repeat Breaker 仍是六家最强（分级文本 + 熔断 + 同 step 去重），但其熔断止于「单命令连发型」循环；对「轮换型」循环与其它五家一样无效。我们的方案若落地，是七家中第一个同时覆盖轮换 + 变体 + 输出语义的。

> 合并注记（2026-10-16）：此注脚中「七家中第一个同时覆盖轮换 + 变体 + 输出语义的」为母文档原文（2026-10-01，当时共七家）；该全称判断已被 gemini-openclaw 补遗 §0 证伪（OpenClaw 把输出哈希做成每个计数器的门槛、gemini-cli 的 k 周期匹配覆盖轮换），两处原文均保留。修订后的差异化卖点见 §十一.2（补遗 §6.1）。另：第三批补遗 §7.3 进一步把「输出逐字节相同才累计」收紧为组合主张（「按签名独立计数、输出一致才累计、穿插容忍、无窗口」四者同时成立），见 §十一.3。

### 6.1 实跑结果（2026-10-09，补遗第二批）

> 出自 `prior-art-gemini-openclaw.md` §4.1。

已把两套算法照抄进 `tmp/pr-sim.ts`（纯追加，原有 8 套输出一字未变）：

- `simGemini()` 照抄 `loopDetectionService.ts:313-349`，**用原始命令文本不做归一化**（这正是它的弱点所在），另按 `client.ts:953-958` 的接线做了「用户消息处清零」的对照版。
- `simOpenClaw()` 照抄 `tool-loop-no-progress.ts:39-47,79-81` + `tool-loop-detection.ts:47-50`。

```
gemini-cli k周期(k=3) 不在用户消息处清零: call#45 (msg 218)
gemini-cli k周期(k=3) 用户消息处清零(实际接线): call#45 (msg 218)
openclaw 窗口30 警告10/拦截20: 警告 call#43 (msg 216) · 拦截 never · 单签名最大 streak 15 (git status --short)
```

四点：

1. 两个 gemini-cli 版本同为 #45，因为循环区内唯一的用户消息在 call idx 62，落在触发点之后，**清零救不了它**。
2. OpenClaw 的 15 与 §五.8（补遗 §3.2）的推算（窗口 30 ÷ 周期 4 × 2 ≈ 15）**精确吻合**，窗口稀释由此从推算变成实测。
3. gemini-cli 能命中的前提是本次循环**熵在下降**（13 个变体全集中在开头 31 次调用，周期相位零变体）；一个变体插进来就会打断 15 连精确匹配。这恰好划出信号 B 仍然有价值的位置：混沌期它比周期匹配早得多（#12 vs #45）。
4. **gemini-cli 的两级升级是可达的**（曾怀疑是死代码，已排除）：`client.ts:953-958` 的 `reset()` 被 `lastPromptId !== prompt_id` 挡住，而 `_recoverFromLoop`（`:1321-1327`）复用同一个 prompt_id，所以 `clearDetection()` 保留的 `detectedCount = 1` 不会被清掉，第二次命中确实会掐流。

**结论：「六家全部拦不住」这张表必须改。** 但卖点不必推翻 —— 见 §十一.2（补遗 §6.1）实测口径。

它能被抓有个前提：本次循环的熵在下降（原文档已观测到，13 个变体全集中在开头 31 次调用，之后归零）。**gemini-cli 的精确周期匹配只对收敛后的循环有效，一个变体就能打断。** 这恰好划出了本方案信号 B（行级信息新颖度）仍然有价值的地方：混沌期它比周期匹配早得多。

## 七、搁浅 PR 精读（diff 级，2026-10-03）

五个 PR 的本地引用已拉取（`pr-32089` / `pr-47797` / `pr-47845` / `pr-46272` / `pr-37109`，只读 fetch）。**在真实循环序列上的模拟触发点**（模拟器：[pr-sim.ts](../analysis/pr-sim.ts)）：

| 方案 | 机制 | 触发点 | 败因 |
|---|---|---|---|
| 旧 doom loop | 单消息尾 3 个 part 全等 | ✗ 永不（每消息仅 1 个 tool part） | 检测范围 |
| #32089（+9/−12） | 全会话历史累计同签名 ≥3 | call#7（最早） | **无窗口累计**：`npm test` 隔几小时跑 3 次也误报；每次调用全量读库 O(历史) |
| #47797（含 139 行单测） | 流式环形缓冲，周期 p=1..10 尾部 3 块相等 | call#39（drain 内抓到周期 3 相位） | 检测器活在**单次 drain** 里，用户插话即清零；命中后每轮重复弹窗 |
| #47845（同日竞争版） | 单消息窗口内块重复 | ✗ 永不 | 范围没解决 + O(n²) 重算 |
| #46272 | 同 user 回合尾 10 连全同 + 硬停止 | ✗ 永不（轮换无 10 连） | 不走权限系统无出口；阈值 10 先白烧 9 轮；死于 bot 清理 |
| #37109（+929 行，最完整） | 同名工具终态输出指纹 ×3（sha256 of 模型所见重放） | ✗ 永不（本例全是 bash，三条命令输出各异） | 改动面大 review 成本高；**死于 PR 模板不合规被 bot 两小时关掉，无技术 review** |
| 我们 A（签名+输出 ×6） | 按签名独立计数 | call#21 | — |
| 我们 B（新颖度 ×8） | 全局行级新颖度 | **call#12** | — |

**六条实现启示**：①正确 scope = 当前 user 回合内跨 assistant message（32089 全会话误报、旧逻辑单消息漏检都错）；②周期内核可借 #47797 环形缓冲但须挂会话级状态；③双信号互补（input 周期 + 输出指纹），指纹须基于模型所见重放形式；④坚持 permission.ask 软中断 + ask 去重防刷屏（46272 式硬停无出口不可取）；⑤稳定序列化别用裸 `JSON.stringify`，检测状态驻内存禁每调用读 DB；⑥**流程教训**：37109 死于模板、46272 死于陈旧、32089/47797/47845 死于无人 review——本方向无反对者，缺的是推进者；小步提交存活率显著更高。

> 合并注记（2026-10-16）：本节为母文档原文。gemini-openclaw 补遗 §6.1 曾对 ①⑤ 两条启示给出修订意见（无窗口累计须改为「每签名只存上次输出哈希 + streak」的 O(1) 结构，才能兼得无窗口与防误报），见 §十一.2；补遗对 #32089 的引用未否定其余启示。

## 八、社区需求图谱（anomalyco/opencode issues，调研于 2026-10-03）

Discussions 未开启，讨论全在 Issues/PRs。结论：**这是高频、持续、真金白银的痛点，且现有检测的缺陷早已被社区逐行指出、至今未修**。

### 呼声主体：死循环烧 token 的用户事故

- [#45442](https://github.com/anomalyco/opencode/issues/45442)（open）：subagent 364 次完全相同的 grep，50 分钟烧 **142M cache-read tokens**，父会话无从察觉
- [#49042](https://github.com/anomalyco/opencode/issues/49042)（open）：525 步 / 66 分钟零用户输入的 runaway auto-continue
- [#47253](https://github.com/anomalyco/opencode/issues/47253)：单任务 22M tokens；[#52092](https://github.com/anomalyco/opencode/issues/52092)：烧光整月 Go 额度
- 时间跨度 2025-11（#3743，27 评论 13👍）至 2026-10，几乎每周有新帖，覆盖 GLM/Kimi/DeepSeek/Qwen/GPT 等所有模型族；大量关闭是 stale-bot 关的，问题实际悬置

### 我们的循环正中的三个 open issue

| Issue | 内容 | 对应本次循环特征 |
|---|---|---|
| [#51965](https://github.com/anomalyco/opencode/issues/51965)（2026-09-29，事故前一天） | doom_loop 跨 step 永不触发（每条 assistant 消息只有 step-start+tool 两个 part，length=2 直接 early return） | 特征 1「跨消息」 |
| [#47759](https://github.com/anomalyco/opencode/issues/47759) | 周期 2–10 的 A,B,A,B 轮换完全漏检（附可复现测试） | 特征 2「轮换」 |
| [#52372](https://github.com/anomalyco/opencode/issues/52372) | 失败调用无限重试不计入（现检测只看成功重复） | 对应「deny 也计数」 |

更早的 [#25254](https://github.com/anomalyco/opencode/issues/25254)（2026-05，stale 关闭）已精准指出 processor.ts 的行号级缺陷；[#23531](https://github.com/anomalyco/opencode/issues/23531) 要求阈值可配；[#32187](https://github.com/anomalyco/opencode/issues/32187) 要求按工具名宽松匹配（参数微调即绕过）。

### 竞争格局：三个搁浅 PR（机会与差异化靶子）

| PR | 状态 | 内容 | 我们的差异 |
|---|---|---|---|
| [#32089](https://github.com/anomalyco/opencode/pull/32089) | open（2026-06 起） | 只修跨 message 检测 | 我们覆盖跨消息+轮换+输出证据 |
| [#47797](https://github.com/anomalyco/opencode/pull/47797)/#47845 | open（2026-09 起，同天重复） | 只修周期块检测 | 我们无需周期识别——按签名计数天然覆盖轮换 |
| [#46272](https://github.com/anomalyco/opencode/pull/46272) | closed 未合并 | 10 次同名同参硬停止 | 我们是分级警告→手术拦截→熔断，不是一刀切硬停 |

另有 #37109（终态结果指纹，未合并）与 #21112（thinking/text 重复块检测，open）触及相邻方向。社区 PR 大多烂在无人 review——PR 描述必须列出覆盖面。

### doom loop 检测的引入历史

2025-10-30 由 PR #3445 引入（作者 ElecTwix，co-author Aiden Cline），初衷即 GLM-4.6/Grok 的重复调用；此后演进（#4095 权限可配、#4412 重构迁移、#4433 修复、#6319 Permission rework）**全是权限层改动，检测算法本身一年未动**，#25254 指出的缺陷至今未修。

### PR 写作策略

描述列 `Closes #51965, closes #47759, refs #25254, refs #45442, refs #52372`，附与 #32089/#47797/#46272 的差异对比表。「一个 PR 关三个 open issue + 覆盖两个百万 token 事故」是最大卖点。

### 范围边界（勿混入）

自动 compaction 不动点循环（#15533 等）、rate-limit 无限重试（#45989）、模型服务端 500 循环（#33006）是「循环」但根因不同，不在本 PR 范围；文本/推理层循环（#12716/#46249/#46492，「Let me run…」意图循环）作为后续 scope 说明。

## 九、引入时间线（gemini-cli / OpenClaw / opencode 三家对照）

> 出自 `prior-art-gemini-openclaw.md` §5（2026-10-09）。opencode doom loop 的引入历史另见 §八「doom loop 检测的引入历史」（母文档原文），此处按补遗原文保留三家对照的全部表格。

### gemini-cli（`gh api repos/google-gemini/gemini-cli/commits?path=packages/core/src/services/loopDetectionService.ts`）

| 日期 | 提交 | 内容 |
|---|---|---|
| **2025-07-15** | `734da8b9` #3919 | **引入 loop detection service**，先只打断简单循环 |
| 2025-07-15 | `886faa29` #4193 | 两类循环分别打点 |
| 2025-07-17 | `ac8e9851` #4348 | 修「每隔一个事件就清零计数」的 bug |
| 2025-07-18 | `9dadf229` #4337 | **加入 LLM 判官层** |
| 2025-07-24 | `0ef9c0b7` #4765 | 命中时记 prompt id |
| **2025-07-25** | `d76cedb6` #4831 | **加入哈希式检测**（即现在的 k 周期匹配） |
| 2025-07-30 | `8985e489` #5144 | 代码块内跳过并清零 |
| 2025-08-08 | `e8815ba4` #5820 | 遇到新的 Markdown 元素类型就清零 |

### OpenClaw（同法）

| 日期 | 提交 | 内容 |
|---|---|---|
| **2026-02-16** | `e5eb5b3e` #17118 | **引入 stuck loop detection + 轮询指数退避** |
| 2026-02-16 | `076df941` | 加入可配置的工具循环检测 |
| 2026-02-16 | `83a8b78a` | 修配置整数解析 |
| 2026-04-13 | `74f2c4a5` #65922 | 止住反复调用不存在工具的循环 |
| **2026-04-27** | `e9bce3f8` | **稳定 exec 的输出哈希**（resultHash 这条线成型） |
| 2026-04-27 | `9b2f10dc` | 保留「不同的空 exec 失败」之间的区别 |
| **2026-07-19** | #111382 | **砍掉数值调参配置面**，收敛到硬编码阈值 |
| 2026-07-27 | `79c517df` #112620 | 加入 argument churn 检测 + 阈值文件 |
| 2026-07-27 | `f0371e8d` #114673 | 修被拦下的工具循环一直卡到超时 |
| 2026-08-02 | `e92a98c7` #118071 | 止住「等价的终端 exec 失败」反复重试 |
| 2026-08-30 | `a7a2dacc` #118402 | 止住**漂移的** exec 失败循环 |
| 2026-10-03/04 | `aff22764` #164199、`91e9b93b` #164566 | 限制每次运行保留的状态量、抽出输出哈希文件 |

### 三家对照

| | 引入 | 现状 |
|---|---|---|
| gemini-cli | 2025-07-15，10 天后就有哈希式周期检测 | 三层，持续演进 |
| **opencode doom loop** | **2025-10-30**（PR #3445） | 一年未动。`processor.ts:29` 阈值仍是 3、`:356` 仍限同一 assistant 消息内、`:365` 仍是裸 `JSON.stringify` 逐字节比对 |
| OpenClaw | 2026-02-16 | 8 个月 7 次针对性加固，6 个检测器 |

**opencode 的检测器出生时就比 gemini-cli 的哈希式周期检测晚三个月，此后没有任何算法演进。** 这条比「六家都没有」更适合当 PR 的开场论据，而且完全可核实。

## 十、横向规律

1. **阈值共识**：「连续 3 次」是各家警告档的共同起点（opencode 3、kimi 3、dsh 首档 3、codex goal 3 轮、Claude Code 收益递减窗口 3）。
2. **两派哲学**：硬熔断派（kimi streak≥12 停 turn、codex goal 置 blocked）vs 软提醒派（dsh never veto、opencode 问用户）。无人值守/自动化场景下软提醒会被绕过（opencode 的 always 放行是实证）。
3. **检测范围决定有效性**：opencode 的 doom loop 败在「同一 assistant 消息内」——真实的取证型死循环都是跨消息每轮一两个调用。kimi 的跨 step streak 是唯一经得起这个场景考验的设计。
4. **变体逃逸**：参数逐字节比对会被 `echo "---"` 这类装饰变体绕过（真实会话中已观测到），kimi/dsh 的参数规范化（key 排序）只解决属性序问题，不解决命令字符串级变体。
5. **业界趋势**：Claude Code v2.1.89→2.1.143→2.1.212 一路在补「无进展空转」的特定场景洞，codex 给 goal 模式加停滞熔断——说明这个方向被普遍需要，但**通用解（工具调用级 + 输出一致性）目前没人做**。

> 合并注记（2026-10-16）：本节为母文档原文（2026-10-01）。规律 5 的「通用解（工具调用级 + 输出一致性）目前没人做」与补遗发现存在张力：OpenClaw 的 `no_progress` / takt 的 `ExactRepeatStreakGuard` 已实现「调用级 + 输出一致性」（但受窗口 / 相邻连续限制），第三批补遗 §7.3 据此把独占主张收紧为组合主张，见 §十一.3；两处原文均保留。

## 十一、对本方案的启发与 PR 定位

> 合并注记（2026-10-16）：本节并列保留四份文档各自的「对本方案的启发 / 影响 / 价值」原文：§十一.1 为母文档（2026-10-01）原文；§十一.2 为补遗 `prior-art-gemini-openclaw.md` §6.1–6.8 原文（保留原小节号）；§十一.3 为补遗 `prior-art-cline-roo-autogpt-crewai-takt.md` §7 原文；§十一.4 为补遗 `prior-art-agentsight.md` §5 原文。三份补遗的部分建议与母文档或其他补遗存在出入，出入处均已加合并注记指明，原文未改动。

### 11.1 对我们 PR 设计的定位（母文档）

1. **输出一致性判定（result 指纹）是六家全没有的增量**——避免误伤的核心：编辑文件后再跑 `git status` 是合法的（输出不同不计数）；对着没变的仓库反复跑才是循环。
2. **分级警告文本参考 kimi**：每档给模型具体出路指令（「写下期望获得什么新信息」「三选一：证伪/问用户/收束」），适合无人值守场景。
3. **吸收 dsh 两条**：被 deny 的调用也计数；用户新消息清零计数器。
4. **跨 step 检测是硬性要求**（opencode doom loop 的失败教训）。
5. **bash 命令归一化**（去引号/剥 echo 分隔符/压空白）防变体逃逸——这是对 kimi/dsh 参数规范化的命令级补强。
6. PR 论据：`anthropics/claude-code#67812` 至今 open（连 Claude Code 都没有，社区在要）+ 真实会话数据（87 分钟、~200 次、56.8M cache-read tokens）。

> 合并注记（2026-10-16）：本节为母文档原文。其中第 1 条「输出一致性判定（result 指纹）是六家全没有的增量」、第 3 条「用户新消息清零计数器」与后续两份补遗的发现存在张力：前者被 OpenClaw / takt 的输出哈希先例削弱（见 §三、§十一.3）；后者被 gemini-openclaw 补遗 §6.6 明确评价为「本方案反对硬清零的判断优于 gemini-cli」（见 §十一.2），且 Cline「新用户消息不清零」（§五.10）提供了同盟——母文档此条建议与这两处原文均保留，未改动。

### 11.2 对本方案的启发（gemini-cli / OpenClaw 补遗，§6.1–6.8 原文）

#### 6.1 差异化卖点应该换成这一条（实测能站住）

原卖点「我们是第一个同时覆盖轮换+变体+输出语义的」不成立（§三 / 补遗 §0）。但排查捞到一条更硬的：

> **窗口大小必须 ≥ 阈值 × 轮换周期，否则拦截档是死代码。**

OpenClaw 的窗口 30 / 拦截档 20 在轮换下够不着（§五.8 / 补遗 §3.2）；gemini-cli 靠精确周期匹配绕开了这点，代价是一个变体就打断。于是本方案信号 A 真正不可替代的地方是**不设时间窗口**：周期 4 相位里 `git status --short` 能一路累到 85 次，任何有界窗口的实现都到不了。

**§六.1 的实跑把这条从推算变成了实测口径，可直接当 PR 头条：**

> 现有实现里最强的两个：gemini-cli 在 call#45 才触发，OpenClaw 在轮换下永远够不到拦截档（实测单签名最大 streak 15 < 阈值 20）。本方案 B 在 #12、A 在 #21。差别来自一处设计选择 —— **计数器不设时间窗口，且每签名只存「上次输出哈希 + streak」**，因此内存 O(签名数)、单次 O(1)，既不会被轮换稀释，也不需要 #32089 那样全量扫历史。

这条的好处是**不依赖任何全称判断**：不需要声称「没人做过」，只需要摆四个实测数字。

但要避开原文档自己批评 #32089 的两条（「无窗口累计导致 `npm test` 隔几小时跑 3 次也误报」、「每次调用全量读库 O(历史)」），实现必须改成：

```
每个签名只存两样：上一次输出的哈希 + 当前 streak
输出相同 → streak += 1
输出不同 → streak = 1
```

内存 O(签名数)、单次调用 O(1)、无时间窗口、轮换免疫、不读历史库。这一条同时优于 OpenClaw（窗口稀释）和 #32089（全量扫描），且与 gemini-cli 的周期匹配正交（后者管收敛后的精确周期，前者管有穿插的长程重复）。**这该是 PR 头条。**

#### 6.2 阈值：向 10 靠，别向 3 靠

- OpenClaw 做过可配置数值，2026-07-19 主动砍掉，收敛到 10（警告）/ 20（拦截）。
- gemini-cli 周期检测阈值是 5 遍完整周期（周期 3 就是 15 次调用），LLM 层满 30 轮才启用。
- 本方案 A 拦 6、B 拦 8、C 拦 4，是全场最激进的，而 §7 自认误伤语料「需构造」、至今没有。
- 「3 与业界共识一致」只对了一半：3 是 opencode/kimi/dsh 的**警告档**起点，**没有任何一家把拦截档设在 6 或 8**。

建议：警告档可留 3（与共识一致、成本低），拦截档往 10 以上走；拿到误伤语料实测结果之前不要把拦截档写死。

> 合并注记（2026-10-16）：此条「拦截档往 10 以上走」与第三批补遗 §7.3 第 5 点的发现直接冲突——后者指出「业界拦截档都在 10 以上」这一论断需要修正（Cline 拦 5、Roo 拦 3、crewAI 相邻 1 次即干预），并认为 Cline 的 3/5 低阈值更支持本方案。两条原文均保留（本处为 gemini-openclaw 补遗 2026-10-09 原文，§十一.3 为第三批补遗 2026-10-12 原文，晚三天、样本多五家），取舍由后续决策定。

#### 6.3 抄 gemini-cli 的两级升级，别在第一次命中就叫人

opencode 现在是第一次命中（阈值 3）就 `permission.ask` 弹给用户。gemini-cli 是：第一次注入反馈让模型自救 → 第二次才掐流并问用户（§五.7 / 补遗 §2.4）。本方案 §4 的分级干预（警 3 / 升 5 / 拦 8 / 熔 12）方向对，但**没写明哪几档是纯注入不打扰用户、哪一档才升级到人**，实现时容易又变成每档都弹窗（原文档批评 #47797「命中后每轮重复弹窗」正是这个坑）。

#### 6.4 逃生口按 gemini-cli 三点改

缺陷三（点一次 always 永久放行）的现成修法：默认选项是「保持防护」（esc）/ 关闭作用域限本会话 / 语义是「关掉检测器」不是「永久允许这个工具」（§五.7 / 补遗 §2.5）。

#### 6.5 信号 C 与 gemini-cli 的分类正面冲突，建议改成失败等价性

信号 C 的靶子是「同路径、参数微调、连续失败 ≥4」。gemini-cli 的系统提示词明写：

> `Retry with variation: Re-attempting a failed operation with modified arguments or a different approach.` —— 属于**必须放行**的形态（`:88`）

OpenClaw 走的是另一条路：不按「同路径失败次数」计数，而用 `failureIdentityHash` 判「失败方式是否等价」（`tool-loop-no-progress.ts:75-78`；2026-08-30 的 `a7a2dacc` 专门修「漂移的失败循环」）。**「失败方式等价」比「同路径失败次数」精确得多**：改了参数但报同一个错 = 等价 = 计数；改了参数报不同的错 = 在推进 = 不计数。

建议信号 C 改成失败等价性判定。顺带这也解了 §7.4 那个悬案（bash 类工具取不到路径 → 所有失败 bash 共用一个 key → 4 次无关失败即误拦），因为失败等价性不依赖路径提取。

另外两条可直接吸收的细节：被拦下的调用继续计数（`tool-loop-veto` → `pendingLoopVetoes`，与本方案 `repeatBlocked` 同解）；参数校验被拒、根本没跑的调用既不延长也不终止失败尾部（`argument-validation`，`:35-38`）—— 本方案没有这个区分。

#### 6.6 「用户新消息硬清零」这条，本方案是对的

gemini-cli 在每次新用户请求时 `reset()` 全清（`client.ts:954`），包括 detectedCount。这正是本方案 §6.2 依据 calebtt 的野生复现（83 步会话里用户两条消息都没打断模型）所反对的做法。**这一点本方案的判断优于 gemini-cli**，PR 描述里可以正面写，并引 calebtt 的复现作证。

OpenClaw 的做法是按 `runId` 分域，跨运行不继承 —— 介于两者之间，也值得参考：清零的粒度是「一次运行」而不是「一条用户消息」。

> 合并注记（2026-10-16）：此条与母文档 §十一.1 第 3 条「吸收 dsh 两条：……用户新消息清零计数器」口径相反（dsh 的做法是清零链，gemini-cli 是硬清零，本方案反对硬清零）。两条原文均保留。

#### 6.7 语义兜底层可以列为 future work

本方案 §0 明确「不读语义、不调 LLM」，这是合理的范围决定。但 gemini-cli 已经证明这条路可行且成本可控（便宜模型 + 满 30 轮才启用 + 5~15 轮自适应 + 置信度 0.9 门槛 + 第二个模型复核）。**PR 描述里应正面承认这是范围划定而非能力上限**，并引 gemini-cli 的做法，否则 reviewer 很容易问「为什么不做语义判定」。

#### 6.8 compaction 守卫要覆盖信号 B 的行库

OpenClaw 有默认开启的 compaction 后循环守卫。本方案 §6.1 的「压缩后首个相同调用不计、第二次才计」只写了信号 A 的计数，**没写信号 B 的行库**（`loop-report.ts:191,209` 的 `lineStore` 只加不删）。compaction 之后模型重建上下文时读的每一行都已在库里 → 零行级信息新颖度 → streak 爬升 → 在合法重建上触发。这正是 #52697 的场景，防护没落到 B 上。

### 11.3 对 PR「同类工作」一节的影响（第三批补遗，§7 原文）

#### 7.1 必须引的

1. **takt**——唯一建在 opencode 之上的同类工作，且 resultHash 入检测口径属实（见 §五.14）。引用要点：wrapper 接入（不是 fork）、四元组、阈值 12、相邻 streak、三级恢复、minimal profile 会裁掉循环守卫。
2. **Cline**——已迁入新 SDK 的两级检测（警 3 注入自救 / 拦 5 停 run），key 排序规范化，**用户消息不清零**（§五.10 / 第三批补遗 §2.3）——这一条为我们「反对硬清零」提供了第二个同盟（此前只有 OpenClaw 按 runId 分域）。
3. **AutoGPT 失败熔断器**——`(tool, sorted-args)` 连续失败 3 次拦截，是信号 C 的最近亲缘（§五.13 / 第三批补遗 §5.1），必须引并写清差异（无等价哈希、任意成功全清）。

#### 7.2 一句话带过的

- Roo Code：`ToolRepetitionDetector`（相邻相同 3 次拦下并阻塞问用户）加按文件 diff 失败计数；**注明其 main 已五个月无提交**。
- crewAI：相邻 1 次复读劝阻（不执行、喂文案）加 `max_iter` 25 软着陆。
- AutoGPT classic：无。

#### 7.3 推翻或收紧我们已有论断的发现

1. **「输出逐字节相同才累计」不再是独有主张。** takt 的 exact-repeat-streak（resultHash 是 streak 延续条件）与 OpenClaw 的 no_progress streak 都已实现。PR 表述必须从「第一个比对输出」改为组合主张：**「按签名独立计数、输出一致才累计、穿插容忍、无窗口」四者同时成立的只有我们**——takt 缺前两条中的按签名独立计数与穿插容忍，OpenClaw 缺最后一条（30 条窗口稀释，见 §五.8 / prior-art-gemini-openclaw.md §3.2）。
2. **行级信息新颖度（信号 B）**：五家全无，此前八家也全无，仍是我们独有。
3. **失败等价哈希（信号 C）**：五家全无（AutoGPT 只认参数逐字节相同、takt 只认输入签名）；唯一撞车的仍是 OpenClaw 的 failureIdentityHash。表述应写「两家里我们是第二家，且我们是唯一不要求相邻的」。
4. **key 排序归一化已是标配**：Cline（递归 sortKeys）、Roo（safe-stable-stringify）、AutoGPT（sort_keys=True）、takt（域分隔规范化）全部做了——gemini-cli 的裸 stringify 才是异类。我们方案里的「参数归一化」不能当卖点写。
5. **阈值分档**：五家里只有 Cline 有「先警告后拦截」两档（3/5），与我们 3/6 量级相近，这比 OpenClaw 的 10/20 更支持我们的低阈值——「业界拦截档都在 10 以上」（prior-art-gemini-openclaw.md §6.2）这一论断需要修正：Cline 拦 5、Roo 拦 3、crewAI 相邻 1 次即干预。
6. **「先注入自救、再升级」已成共识**：Cline（警注入 / 拦停 run）、takt（纠偏 prompt 到新 session）、gemini-cli（注入到掐流）三家同构，我们的分级干预方向与之一致，PR 可正面引。

> 合并注记（2026-10-16）：本小节第 5 点与 §十一.2 补遗 §6.2「阈值：向 10 靠，别向 3 靠」存在直接冲突，冲突两处的原文均保留，见 §十一.2 该处的合并注记。

### 11.4 对本方案的价值（AgentSight 补遗，§5 原文）

1. **PR 引用**：一句话即可——「可观测层也有同类问题意识，阿里 AgentSight 用 eBPF 在进程外做，检测后走 SIGTERM/SIGKILL 阶梯；本 PR 做的是运行时内、给模型留纠正余地的路线」。它的规则 2/3 恰好覆盖我们划界出去的文本/推理循环族（lane-C），可作为「那类问题在另一层有解」的佐证。
2. **规则 3（token 空烧）是我们没有的维度**，与轮次级护栏（裁定 6 的 follow-up）同源，写 future work 时可引。
3. **不构成对裁定的冲击**：它不比工具输出、轮换失明、干预是杀进程——三信号设计（尤其 A 的输出证据与分级干预）的差异化不受它影响。
4. 它的 `test_same_tool_different_args_not_a_loop`（`:507`，issue #2691 回归测试）与我们的误伤语料 §1.4 同旨——「同工具不同参数不算循环」是各家都踩过的误伤点，可作为误伤语料必要性的旁证。

---

> 合并注记（2026-10-16）：全文完。本文档内所有「合并注记（2026-10-16）」均为合并时新增，其余正文均出自四份源文档（个别仅为章节编号与交叉引用的适配，已就地注明）。被源文档引用但不在本目录下的文件：`pr-sim.ts`（母文档 §七称其为同目录模拟器；gemini-openclaw 补遗称在 `tmp/pr-sim.ts`）、`误伤语料.md`（gemini-cli §2.6 引用）——真实位置已核实为 `analysis/pr-sim.ts` 与 `design/误伤语料.md`，正文链接已修正指向。
