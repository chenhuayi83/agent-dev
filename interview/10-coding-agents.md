---
last_updated: 2026-08-19
type: interview
topic: 编码代理实战
questions: 6
---

# 面试题:10 编码代理与 vibe coding 实战组

> 每题讲透五件套:本质 / 机制 / 实例 / 边界 / 检验(追问预判)。答案分 30 秒版 + 深入版。
> 这组考"用 AI 编码工具的实战功力"——技能篇([skills/claude-code/01](../skills/claude-code/01-core-workflow.md))讲"怎么用",本篇讲"面试怎么答透机制与权衡"。
> 全文功能/形态断言以 **as-of 2026-08-19** 为口径(来源见各题关联),赛道两周一变,面试中应主动声明口径。

## Q1:Claude Code 这类编码 agent 的 harness 由哪些部分组成?为什么说"同一模型,harness 决定表现"?

**考察点**:能否把 harness 拆成可枚举的部件,而不是笼统说"就是个壳";能否给出 harness 影响表现的**因果通道**,并对基准分差做克制归因。

**30 秒版**:harness = 模型之外的一切工程,可拆成七件:(1) 系统提示与身份;(2) 工具集及其 schema;(3) 启动上下文(每会话必载的指令与记忆);(4) 上下文管理(compaction / 按需加载 / 子上下文);(5) 权限与沙箱;(6) 循环控制与终止条件(验证闭环、hooks、goal);(7) 编排与扩展(subagents、skills、plugins、MCP)。说"harness 决定表现"是因为模型只做一件事——**在当前上下文下决定下一步**;而每一步**看到什么、能做什么、错了怎么被告知、什么时候算完**全由 harness 决定。同代模型下这是主要可操作变量(Artificial Analysis Coding Agent Index,as-of 2026-05-18:Claude Code+Opus 4.7 = 66 > Codex+GPT-5.5 = 65 > Cursor Composer 2.5 = 62)。

**深入版**:

harness 影响表现有**四条因果通道**,把它们讲出来就超过背部件清单的答案了:

1. **信息通道(模型看到什么)**——本质是上下文预算的分配。会话还没开始,系统提示 + 工具定义 + CLAUDE.md + auto memory 索引(`MEMORY.md` 首 200 行或 25KB)就已经占位。harness 高明与否体现在**默认加载 vs 按需加载**的划线:skills、path-scoped rules、subagent 隔离都是把"偶尔要用的知识"从常驻上下文里挪走的手段。
2. **行动通道(能做什么、什么粒度)**——工具集设计。给结构化的 `Edit`(带精确字符串匹配、失败即报错)还是只给 `Bash` 让它自己 `sed`,编辑成功率差一个量级。工具颗粒度、输出截断策略、能否并行执行,都在这条通道上。
3. **反馈通道(错了怎么被告知)**——错误作为 `tool_result` 回传、hook 的 stderr、评估器的判定。这是 agent 自愈能力的燃料;harness 若静默吞掉错误,模型就失去了纠错依据。
4. **终止通道(什么时候算完)**——最被低估的一条。默认终止条件是"模型自判完成";装上 Stop hook / `/goal` 之后,终止条件变成**外部谓词**。这条通道直接决定"你得盯着的会话"与"你可以走开的会话"的分界(见 Q3)。

还要讲**协同演化**:模型被后训练得越擅长某类工具协议,harness 越倾向暴露那类工具;而主流 harness 的默认形态又反过来进入下一代训练分布。所以 harness **不可无损移植**——把一套工具集接到另一家模型上,表现不会原样搬过去。

**实例**(Claude Code 的部件 → 载体 → 你的可调点,as-of 2026-08-19):

| harness 部件 | 载体 | 你能调什么 |
|---|---|---|
| 系统提示 | 内置 + `--append-system-prompt` | 脚本化注入硬规则(须每次调用都传) |
| 工具集 | Read/Edit/Bash/Grep/Glob/WebFetch + MCP | `--allowedTools`、`claude mcp add` |
| 启动上下文 | `CLAUDE.md`、`.claude/rules/`、auto memory | 精简、path-scoped、`claudeMdExcludes` |
| 上下文管理 | auto-compact、`/compact <指令>`、`/rewind` 摘要、`/btw` | 在 CLAUDE.md 里指定 compaction 保留项 |
| 权限/沙箱 | auto mode 分类器、`/permissions`、`/sandbox` | 白名单 + OS 级隔离 |
| 终止条件 | 模型自判 / `/goal` / Stop hook(连续 8 次阻止后强制放行) | 四档验证强度 |
| 编排 | `.claude/agents/`、fork、agent teams、worktrees | per-subagent 模型与工具白名单 |
| 扩展/记忆 | `.claude/skills/`、plugins、auto memory | 按需加载而非常驻 |

用 `/context` 可以直接看到这些在启动时吃掉了多少窗口——这是把"harness 是抽象概念"落成可量化数字的最快一招。

**边界与误区**:
- 误区:"harness 就是提示词工程"。提示词只覆盖信息通道的一部分;终止通道、权限通道跟提示词无关,而它们决定了能否无人值守。
- 误区:"模型变强 harness 差距会消失"。恰恰相反——模型越强,能被 harness 释放的**自主时长**越长,终止/验证/上下文设计的差距被放大。但也要承认反向事实:有些 harness 技巧是补模型短板的补丁(如手写 ReAct 提示),确实已被内化淘汰(见 [01-Q4](01-agent-fundamentals.md))。
- 边界:66/65/62 的分差里**同时包含模型与 harness**,不能拆开归因。严谨说法是"同代模型下 harness 是主要可操作变量",而不是"harness 值 4 分"。

**追问预判**:
- 追问:"从零做一个 coding harness,先做哪三件?" → 答:(1) 可靠的结构化编辑工具(diff 能精确应用、失败必报错);(2) 验证入口(测试/构建的退出码与输出回传进对话);(3) 上下文预算控制(工具输出限幅 + 子上下文隔离)。系统提示排在这三件之后。
- 追问:"harness 能跨模型移植吗?" → 答:接口能移,表现不能。工具 schema 风格、思考预算、指令遵循强度都是模型特性,移植后必须重调工具粒度与提示并重跑基准。

**关联**:[kb/09-coding-agents/harness-design.md](../kb/09-coding-agents/harness-design.md) · [kb/09-coding-agents/claude-code.md](../kb/09-coding-agents/claude-code.md) · [01-Q2 agent loop](01-agent-fundamentals.md)

## Q2:CLAUDE.md / AGENTS.md 的作用机制是什么?为什么写太长反而失效?该写什么不该写什么?

**考察点**:是否知道这些文件**以什么形态**进入模型(是上下文,不是配置);能否从机制推出"该写什么",而不是背清单。

**30 秒版**:它们是**每会话必载的持久指令文件**。关键机制:Claude Code 把 CLAUDE.md 作为**系统提示之后的一条 user message** 注入——所以它是**软约束**,不保证遵守;要硬约束得用 hook 或 managed settings。AGENTS.md 则是跨工具的开放约定(Codex 等 20+ 工具支持),语义是**就近的文件胜出**。写太长失效有两条叠加原因:(1) 它每会话都在占用最稀缺的上下文,挤压任务空间;(2) 指令密度被稀释,关键规则淹没在噪声里。官方给的量化线是**每个 CLAUDE.md 目标 200 行以内**(as-of 2026-08)。判据一句话:逐行问"删了它会出错吗?",不会就删;只留"每个会话都需要" ∧ "读代码推不出来"的内容。

**深入版**:

**机制一:注入形态决定了它是建议不是命令。** CLAUDE.md 内容作为 user message 跟在系统提示后面,不是系统提示的一部分,更不是策略引擎。因此在里面写 "YOU MUST 不许改 migrations 目录"只是提高概率;真要拦住,得用 PreToolUse hook(确定性阻断)或 managed settings 的 `permissions.deny`。**把软约束当硬约束用**是这题最高频的失分点。

**机制二:加载与合并规则(两家语义不同,跨工具复用时会踩)。**
- Claude Code(as-of 2026-08-19):按作用域从宽到窄**拼接**(不是覆盖)——managed policy → `~/.claude/CLAUDE.md` → `./CLAUDE.md` 或 `./.claude/CLAUDE.md` → `./CLAUDE.local.md`。工作目录**以上**的祖先目录文件在启动时全量载入;**子目录**里的 CLAUDE.md 是按需载入(Claude 读到该目录的文件时才载)。`@path` 导入最多 4 跳,且**导入不省上下文**——被导入的内容启动时照样展开进窗口,只是文件组织更好看。
- AGENTS.md:**最近的文件胜出**,显式的用户 prompt 覆盖一切。
- 同一仓库要同时服务两边:官方推荐在 CLAUDE.md 里写 `@AGENTS.md` 导入(可在下面追加 Claude 专属规则),或做符号链接 `ln -s AGENTS.md CLAUDE.md`。Claude Code 也提供 `/import` 把其他 agent 的配置一次性并进来(需 v2.1.213+)。

**机制三:失效是两条独立通道叠加。** 上下文预算(每行都在跟任务抢窗口)+ 注意力稀释(规则越多,单条被遵守的概率越低)。好消息是症状可诊断:
- **你反复强调的规则总被忽略** → 文件太长,规则丢在噪声里;
- **Claude 反复问文件里已经写了的问题** → 措辞歧义,不是长度问题。

**该写 / 不该写**(从判据推出来,而不是背):

| ✅ 写进去 | ❌ 不要写 | 该去哪 |
|---|---|---|
| 猜不到的 bash 命令(构建/测试入口) | 读代码就能看出来的东西 | 删 |
| 与语言默认不同的代码风格 | 标准语言约定 | 删 |
| 测试方式与偏好的 runner | 详细 API 文档 | 给链接 |
| 仓库礼仪(分支命名、PR 约定) | 频繁变化的信息 | 删或外置 |
| 项目特有的架构决策 | 长篇解释与教程 | skill |
| 环境怪癖(必需的 env var) | 逐文件的代码库描述 | 删(可推导) |
| 非显然的坑与其理由 | "写干净代码"这类废话 | 删 |

**高分点:砍掉的东西不是丢弃,是按加载时机重新分配。**
- 只对某类文件生效 → `.claude/rules/*.md` 带 `paths:` frontmatter(只在 Claude 读到匹配文件时载入);
- 只对某类任务生效的多步流程 → skill(`.claude/skills/<name>/SKILL.md`,按需触发,不污染每次对话);
- 必须每次都发生、不能商量 → hook(确定性);
- 该由模型自己积累的 → auto memory(Claude 自写;`MEMORY.md` 索引首 200 行/25KB 每会话载入,细节进 topic 文件按需读)。

**实例**(一份合格的极简 CLAUDE.md,官方示例形态):

```markdown
# Code style
- Use ES modules (import/export) syntax, not CommonJS (require)
- Destructure imports when possible (eg. import { foo } from 'bar')

# Workflow
- Be sure to typecheck when you're done making a series of code changes
- Prefer running single tests, and not the whole test suite, for performance
```

配套的可验证操作:`/init` 生成初版(已存在则给改进建议);`/context` 确认到底载入了哪些 memory 文件;`/doctor` 会提出裁剪建议——砍掉能从代码推出的目录结构/依赖清单/架构概览,保留坑、理由、与工具默认不同的约定(需 v2.1.206+);`InstructionsLoaded` hook 可日志化"何时载入了什么",调试 path-scoped rules 很有用。

还有一条实战细节:**compaction 之后,项目根 CLAUDE.md 会被从磁盘重新注入,而子目录 CLAUDE.md 与 path-scoped rules 不会**(它们等下次读到匹配文件才回来)。所以"只在对话里口头说过的约束"压缩后就蒸发了——重要约束必须落盘。

**边界与误区**:
- 误区:把 CLAUDE.md 当强制配置。它是上下文,不是策略引擎。
- 误区:用 `@import` 给上下文减负。导入内容启动时照样展开,减的是文件长度不是 token。
- 误区:monorepo 用一个巨型根文件统治。正解是分层 + path-scoped rules,并用 `claudeMdExcludes` 排掉别的团队的文件。
- 边界:AGENTS.md 就近生效 vs CLAUDE.md 全链拼接——同一份文本在两个工具下行为不同,跨工具复用时必须意识到。

**追问预判**:
- 追问:"CLAUDE.md 和 skill 怎么划界?" → 答:按**加载时机**分,不是按内容类型。每会话都需要的事实 → CLAUDE.md;偶尔才需要的领域知识或多步流程 → skill,按需载入。判据是"这条知识在 90% 的会话里是噪声吗?"。
- 追问:"团队的 CLAUDE.md 怎么治理?" → 答:当代码管——进 git、走 code review、定期 prune;组织级硬规则走 managed policy(如 macOS 的 `/Library/Application Support/ClaudeCode/CLAUDE.md`,或 managed settings 的 `claudeMd` 键,个人设置无法排除);个人偏好走 `CLAUDE.local.md` 并 gitignore。

**关联**:[kb/01-context-engineering/overview.md](../kb/01-context-engineering/overview.md) · [kb/05-memory/memory-architectures.md](../kb/05-memory/memory-architectures.md) · [skills/claude-code/01](../skills/claude-code/01-core-workflow.md)

## Q3:怎么设计验证闭环,让编码 agent 能无人值守地自我收敛?

**考察点**:能否说清"没有检查就没有自主"的因果,并把验证强度**分档**——每档在拿什么换什么、各自的失败模式是什么。

**30 秒版**:agent 在"工作看起来完成"时停下,所以要让它自己收敛,必须给一个**它能跑、且结果会回到对话里**的判定。官方四档,按无人值守程度递进(as-of 2026-08):(1) **提示词内嵌**——同一条消息里要求"实现后跑测试并修复失败",零配置、任何任务当场可用;(2) **`/goal` 条件**——独立评估器每轮之后复查目标,未达成就继续;(3) **Stop hook**——脚本作为确定性闸门,不通过就阻止回合结束(**连续 8 次阻止后 Claude Code 强制放行**);(4) **对抗评审 subagent**——fresh context 只看 diff 与标准来挑错,干活的不给自己打分。贯穿四档的心法:**要证据不要断言**——让它贴测试输出、跑了什么命令返回了什么、截图,而不是说"已完成"。

**深入版**:

**为什么必须把终止条件外部化。** agent loop 的默认终止是模型自判(输出纯文本即回合结束)。自判的失败模式不是随机的,而是**系统性偏乐观**:模型看到的是自己刚写的代码和自己的推理链,天然缺少反例。把终止条件换成外部谓词(测试退出码、评估器判定、脚本闸门),loop 就从开环变闭环。

**四档在换什么**(这张表是本题的核心):

| 档位 | 判定者 | 触发时机 | 成本 | 覆盖什么 | 典型失败 |
|---|---|---|---|---|---|
| 提示词内嵌 | Claude 自己 | 它自己决定何时跑 | 0 | 它记得跑的那些检查 | 上下文一长就忘了跑;失败后偷偷放宽断言 |
| `/goal` 条件 | 独立评估器模型 | 每轮之后 | 每轮一次额外评估 | "是不是在做你要的事"这类语义判断 | 目标写得含糊 → 判不准;卡住时会带着未达成的 goal 停下 |
| Stop hook | 你的脚本 | 回合结束时 | 一次性写脚本 | 脚本能测的确定性事项 | 脚本慢 = 每回合都慢;**8 次阻止后强制放行**,不是绝对的墙 |
| 对抗评审 subagent | fresh context 的模型 | 你指定的检查点 | 一次子会话 | 测试测不到的:越界改动、需求漏项、缺的边界测试 | 必然报出东西 → 过度工程(见 Q6) |

**四档不是四选一,而是叠加**——它们覆盖的失败类型正交:测试覆盖"行为对不对",评估器覆盖"是不是在做要求的事",hook 覆盖"必须发生的事有没有发生"(lint/typecheck/禁改某目录),对抗评审覆盖"有没有绕过要求或改了不该改的"。长时间无人值守的典型配置是 **hook(硬闸门)+ goal(语义目标)+ 收尾一次对抗评审**。

**三条工程原则**:
1. **判定必须 agent 可读**:退出码 + stdout 回到对话里。要人点一下 GUI 才知道结果的,不算判定(除非用截图对比之类的方式工具化)。
2. **判定必须先于实现存在**:先写能复现失败的测试再修(官方提示词模式:"write a failing test that reproduces the issue, then fix it"),否则"通过"可能只是断言被改松了。
3. **判定必须防作弊**:agent 有让检查变绿的动机。反制在三层——权限层(`--allowedTools` 不给测试目录写权限 / PreToolUse hook 拦截)、diff 层(评审 prompt 点名"检查是否修改了测试或放宽了断言")、流程层(失败测试先 commit,再实现)。

**实例**(四档各一条真实写法):

```text
# 第 1 档:提示词内嵌
implement the OAuth flow from your plan. write tests for the
callback handler, run the test suite and fix any failures.
```

```bash
# 第 3 档配 auto mode 的无人值守跑法(分类器审查动作)
claude --permission-mode auto -p "fix all lint errors"
```

```text
# 第 4 档:对抗评审(范围/标准/检查项/判定线四要素齐全)
Use a subagent to review the rate limiter diff against PLAN.md. Check that
every requirement is implemented, the listed edge cases have tests, and
nothing outside the task's scope changed. Report gaps, not style preferences.
```

内置的 `/code-review` skill 是第 4 档的开箱形态:在 fresh subagent 里审当前 diff 找 bug,发现直接回传到当前会话。

**边界与误区**:
- 误区:"有 CI 就够了"。CI 在 PR 之后才跑,反馈慢且**不在 agent 的上下文里**——闭环要求判定在会话内可读。
- 误区:"Stop hook 是绝对闸门"。连续 8 次阻止后会被强制放行,它是强约束不是不可越过的墙;长跑要靠多档叠加。
- 误区:验证越多越好。每档都收税(评估器每轮一次调用、hook 每回合跑一次脚本、评审一次子会话),小任务开满是纯浪费。
- 边界:没有可自动判定的任务(纯设计/文案/开放探索),四档退化为"要证据"——让它把做了什么、依据是什么摆出来给人审。

**追问预判**:
- 追问:"怎么防止它为了让测试变绿而改测试?" → 答:权限层隔离 + diff 层点名检查 + 流程层先 commit 失败测试。三层里权限层最硬,因为它不依赖模型的配合。
- 追问:"`/goal` 和 Stop hook 选哪个?" → 答:判定需要理解意图、是语义的 → `/goal`(评估器是模型);判定能写成脚本、必须确定性 → Stop hook。两者可叠,分别兜语义与确定性两类失败。

**关联**:[kb/06-evals/agent-evals.md](../kb/06-evals/agent-evals.md) · [kb/07-safety-security/permissions-hitl.md](../kb/07-safety-security/permissions-hitl.md) · [skills/claude-code/01](../skills/claude-code/01-core-workflow.md)(原理二同源)

## Q4:大规模改造(几百上千个文件)怎么用编码 agent 做?fan-out 与并行开发的工程实践与风险?

**考察点**:能否把"改一个文件"的经验**正确地**放大到规模;是否具备批处理工程意识(可分片性、抽样、幂等、隔离、可回滚)。

**30 秒版**:核心是**把一个巨型任务拆成 N 个独立小任务,每个跑在自己的干净上下文里**。官方 fan-out 三步:(1) 先让 Claude 把待改文件清单落盘(`list all 2,000 Python files that need migrating and save the list to files.txt`);(2) 写循环逐文件调无头模式,用 `--allowedTools` 收紧权限;(3) **先在 2-3 个文件上把 prompt 调对,再全量跑**。前提有两条:任务**可分片**(片间无耦合)且**单片可判定**。如果改动本质上是跨文件耦合的重构,fan-out 会产出一堆互相冲突的 diff——那时该走并行会话 + worktree 隔离,或者干脆让 agent 写一个确定性 codemod。

**深入版**:

**为什么 fan-out 有效**——回到上下文第一性原理。一个会话连改 500 个文件,上下文在第 30 个文件左右就开始退化;500 次独立的 `claude -p` 每次都是干净上下文,第 500 个与第 1 个质量一致。fan-out 的本质是**用进程隔离换上下文新鲜度**,代价是每片看不到全局——跨文件一致性只能靠 prompt 里的模式描述,或先手工建立一个参照实现让每片照抄。

**判断任务能不能 fan-out 的三个条件**:
1. **可分片**:改动之间无顺序依赖、无共享文件冲突(500 个进程同时改 `package.json` 必然打架);
2. **单片可判定**:每片自己能验证(该文件的测试/typecheck 通过),否则你要跑完全量才知道错了;
3. **失败可隔离**:一片失败不污染其他片——每片独立 commit,失败的记下来单独重跑。

**实例**(官方 fan-out 骨架):

```bash
# 1. 先让 Claude 生成清单(它比你更知道哪些文件需要改)
claude -p "list all 2,000 Python files that need migrating and save the list to files.txt"

# 2. 逐文件跑,权限收到最小集
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

**生产化要补的四件事**(这是区分点):
- **结构化输出**:`--output-format json` 拿 `result` 字段做程序判定,或 `--output-format stream-json --verbose` 流式落日志;成功/失败入表,便于只重跑失败集;
- **每片一个 commit**:`Bash(git commit *)` 进白名单的意义就在这里——每片是一个**可回滚单元**,而不是一个几千行的巨型 diff;
- **抽样先调 prompt**:官方明说先跑 2-3 个,按出错方式改 prompt 再全量。这一步省下的是几百次返工;
- **并发与限流**:可以用 `xargs -P` 提并发,但要考虑 API 速率与 **git 索引锁**(同一 worktree 里并发 commit 会冲突)——要么每个并发单元一个 worktree,要么 commit 串行化。

**fan-out ≠ 并行开发。** fan-out 是"同一个任务的切片",并行开发是"多个不同任务同时推进"。官方四种并行方式的协调成本递增(as-of 2026-08):
- **git worktrees**:各自独立 checkout,编辑不会碰撞——隔离最强、协调成本最低,适合互不相干的任务;
- **桌面端多会话**:每个会话有自己的 worktree,可视化管理;
- **Claude Code on the web**:云端跑,默认在 Anthropic 托管基础设施上;
- **agent teams**:自动协调多会话,带共享任务、消息与 team lead——协调成本最高,但能跑长链路。

**风险清单**(面试要主动列,这是"做过"的信号):
- **合并地狱**:并行分支越多冲突面积越大。缓解:按目录/模块切边界;**先 land 会被大家依赖的公共改动**(接口、类型定义);
- **一致性漂移**:每片独立决策 → 500 个文件出现 500 种写法。缓解:prompt 指定参照实现文件、约定命名、跑统一 formatter/linter 收敛;
- **静默失败**:模型返回 `OK` 但实际没改对。缓解:每片跑判定 + 全量跑一次集成测试/构建 + 按风险抽样人审;
- **权限放大**:无人值守时每个进程都握有 Edit/Bash 权限,一个坏 prompt 会被乘以 500。缓解:`--allowedTools` 最小集 + sandbox + 不该动的目录只读;
- **成本失控**:500 次调用 × 每次数万 token。缓解:抽样估单片成本再乘、分批跑、设预算闸。

**边界与误区**:
- 误区:"文件多就该 fan-out"。如果改动本质是一次跨文件重构(比如挪动一个接口),每片对全局的理解都是残缺的——这类更适合"单会话 + 精确 plan",或者更好:**能写成确定性脚本的,让 agent 写 codemod(ast-grep / jscodeshift)再自己跑**,别让模型当人肉 sed。规模化最省钱也最一致的一招就是"用模型生成确定性变换,而不是用模型做每一次变换"。
- 误区:并发越高越快。git 锁、API 限流、机器 IO 都会成瓶颈,而且失败诊断成本随并发上升。
- 边界:开批的前提是**沙箱/权限边界 + 可回滚的 commit 粒度**。这两样没有就别开无人值守批处理。

**追问预判**:
- 追问:"fan-out 跑完怎么验收 500 个 diff?" → 答:分三层——机器层(全量构建/测试/lint 必须绿)、抽样层(按风险分层随机抽 5-10% 人审)、对抗层(fresh context 的评审 agent 审聚合 diff 的模式一致性)。逐个人审等于把省下的时间还回去。
- 追问:"什么时候用 agent teams 而不是自己写循环?" → 答:片与片之间需要通信/共享状态/动态分配时(例如迁移中发现的公共问题要广播给其他片)用 teams;完全独立的切片用循环——协调机制是成本,没有需求就不要买。

**关联**:[kb/09-coding-agents/autonomous-swe.md](../kb/09-coding-agents/autonomous-swe.md) · [kb/04-multi-agent/orchestration-patterns.md](../kb/04-multi-agent/orchestration-patterns.md) · [kb/07-safety-security/sandboxing.md](../kb/07-safety-security/sandboxing.md)

## Q5:Claude Code、Codex、Cursor 的形态与 harness 取舍有什么差异?怎么选?

**考察点**:是否真的用过多家(能说出形态与**配置模型**的具体差异),以及能否克制地做公允比较——不吹不踩、数据带口径。

**30 秒版**:三家如今都是"多形态 + agent-first",差异在**重心**与**配置模型**(as-of 2026-08-19)。Claude Code:重心在终端与无头可编程,扩展面最厚(`CLAUDE.md` + `.claude/rules/` + skills + hooks + subagents + plugins + MCP),支持 per-subagent 模型控制。Codex:重心在 ChatGPT 生态的多面覆盖(桌面/网页/CLI/IDE 扩展/云),配置分层为 config 文件 + 环境变量 + **AGENTS.md** + subagents,带 profiles + sandboxing、内置代码审查与安全扫描。Cursor:重心在 IDE 的编辑与 diff 审查体验(Cursor 3 于 2026-04-02 转 agent-first),同时补齐了 CLI(含 Headless/CI)与 Cloud Agents,可选多家模型。选型判据不是"谁强",而是**你的工作形态**:要跑批/进 CI/脚本化 → 终端派;要逐 diff 精审、强 IDE 依赖 → IDE 派;组织已锁定某家生态(账号、合规、安全扫描)→ 生态一致性的价值通常大于单点能力差。

**深入版**:

**先立好比较的边界。** 三家的模型可换且同代,公开基准的差距已经很小:Artificial Analysis Coding Agent Index(as-of 2026-05-18)Claude Code+Opus 4.7 = 66、Codex+GPT-5.5 = 65、Cursor Composer 2.5 = 62。这个分差**同时包含模型与 harness**,拆不开。能站得住的结论只有一条:**同代模型下 harness 是主要可操作变量**——而 harness 的取舍是可以客观描述的,这才是面试该讲的部分。

**实例**:三家取舍轴对比(as-of 2026-08-19,官方文档口径):

| 维度 | Claude Code | Codex | Cursor |
|---|---|---|---|
| 形态 | CLI / 桌面 / Web / IDE 插件 / 移动端 | ChatGPT 桌面 / 网页 / Codex CLI / IDE 扩展 / Codex cloud | IDE(主)/ CLI(Shell Mode、Headless-CI)/ Cloud Agents |
| 持久指令 | `CLAUDE.md`(全链拼接)+ `.claude/rules/`(path-scoped) | `AGENTS.md`(就近生效)+ config 文件 + 环境变量 | Rules(与 plugins/skills/MCP 同在 Customize 面板) |
| 扩展机制 | skills / hooks / subagents / plugins / MCP | skills / plugins / subagents / MCP | plugins / skills / MCP / rules |
| 权限模型 | auto mode 分类器 + `/permissions` + `/sandbox` | profiles + sandboxing + agent approvals + internet access 控制 | Agent Review(合并前查 diff、跑检查) |
| 无头/批处理 | `claude -p` + `--output-format json\|stream-json` + `--allowedTools` | Codex CLI + Codex cloud | CLI Headless / CI |
| 审查能力 | `/code-review` skill、对抗评审 subagent | 内置代码审查 + 自定义审查规则 + 安全扫描(Deep scans、Security workbench) | Agent Review 面板;Plan / Debug / Design 模式 |
| 模型 | Anthropic 系,per-subagent 模型控制 | OpenAI 系 | 多家可选;文档列出的默认上下文 200k–300k、最高档到 1M |

**趋同的与分化的。** 趋同:形态(每家都补齐了 CLI + IDE + 云)、扩展机制(skills / plugins / MCP)、以及配置约定——`AGENTS.md` 已成跨 20+ 工具的开放格式,2026-08 公布的 **Agent Plugins 标准**(OpenAI 与 AWS/Cursor/GitHub/Microsoft/Vercel 共建,08-11 v1.0.0 草案)用 `plugin.json` 打包 Agent Skills + MCP 配置以跨端复用。分化:**默认工作面**与**治理形态**——终端派默认无头可编排、人在环外;IDE 派默认人在环内逐 diff 确认;生态派把审查与安全扫描做成平台能力。

**可操作的选型判据**:
- 验证靠脚本、要跑批或进 CI → 终端派(无头 + 权限白名单是刚需);
- 需要逐行读 diff、强依赖 IDE 的跳转与调试器 → IDE 派;
- 组织已统一在某家的账号/合规/安全扫描上 → 生态一致性往往压过单点能力差;
- 团队多工具并存(很常见)→ 把持久指令写进 `AGENTS.md` 作单一真相源,给 Claude Code 一个 `CLAUDE.md` 用 `@AGENTS.md` 导入或符号链接,避免两份漂移。

**边界与误区**:
- 误区:拿单一基准分排座次。口径、模型版本、任务分布任一变化都可能翻转结论,而且 66/65/62 在同一量级——个人工作流匹配度的影响通常大于这几分。
- 误区:"选一家就锁死"。三家如今都读得懂开放约定(AGENTS.md、MCP、Agent Plugins),真正的迁移成本在 hooks / 权限 / CI 这些 harness 侧胶水上,而不在提示词。
- 边界:本题所有形态与功能均为 **as-of 2026-08-19** 口径;面试里应主动声明"我给的是这个时点的口径,判据比结论更耐用"。

**追问预判**:
- 追问:"只能靠一个指标选,你看什么?" → 答:看**它能不能让我把验证闭环自动化**——有没有可脚本化的闸门(hook / 退出码回传)、有没有无头模式、权限能不能收到最小集。这决定了你能不能走开,而"能走开"才是生产力的跃迁点(见 Q3)。
- 追问:"多工具并存怎么治理?" → 答:指令层用开放约定做单一真相源(AGENTS.md + 各家导入机制);扩展层尽量走 MCP / Agent Plugins 这类跨端标准;工具特有的部分(hooks、权限、CI 集成)各自维护并写进 README,明确标注哪些不可移植。

**关联**:[kb/09-coding-agents/other-coding-agents.md](../kb/09-coding-agents/other-coding-agents.md) · [kb/09-coding-agents/claude-code.md](../kb/09-coding-agents/claude-code.md) · [kb/03-protocols/mcp.md](../kb/03-protocols/mcp.md) · [kb/10-landscape/ecosystem-map.md](../kb/10-landscape/ecosystem-map.md)

## Q6:用编码 agent 做代码审查为什么要"新开一个会话"?fresh context 的价值与对抗评审的陷阱是什么?

**考察点**:能否说清"自己审自己"的**具体失效机理**(而不是笼统的"不客观");以及能否识别对抗评审自带的反向失效——这是分辨"真用过"与"读过文档"的题。

**30 秒版**:因为在写代码的那个会话里,**产出这份 diff 的整条推理链也在上下文里**——模型看到的不是一份待判定的改动,而是"我为什么这么写"的完整辩护。fresh context 的 reviewer 只看 diff 和你给的标准,没有那条辩护链,所以它评的是结果本身。这就是官方 Writer/Reviewer 双会话模式的机制。陷阱在反面:**被要求挑错的 reviewer 一定会挑出东西**,哪怕代码本身是好的——追着每条发现改会导致过度工程(多余的抽象层、防御性代码、为不可能发生的情况写的测试)。解法是给它一条判定线:**只报影响正确性或既定需求的 gap,其余标为可选**。

**深入版**:

**"自己审自己"到底哪里失效**——三条具体机制,别停在"不客观":
1. **上下文污染**:写的过程中所有失败尝试、被否掉的方案、临时妥协都还在上下文里,它们构成"当时为什么这么选"的解释,让不合理的选择读起来合理。
2. **一致性压力**:模型的输出条件在自己之前的输出上。前面刚说过"这里不需要处理 null",后面审同一段时推翻自己是低概率路径。
3. **注意力已耗散**:实现结束时上下文往往已经很满,而审查恰恰需要高质量注意力——**最需要仔细看的时刻,正是模型状态最差的时刻**。

fresh context 一次修掉这三条:干净上下文 + 无自我一致性负担 + 注意力预算全给审查。**代价**是它不知道"为什么"——那些有意的取舍(为兼容旧接口故意保留的怪写法)会被当成缺陷报出来。所以 reviewer 的 prompt **必须补上意图**:给它 PLAN.md / SPEC.md / issue 作为判定标准。否则它只能拿通用最佳实践当标准,而这正是过度工程发现的主要来源。

**两种落地形态**(as-of 2026-08):
- **subagent 形态**:同一会话派一个 fresh-context subagent 审 diff,发现直接回到实现会话,可以"改了再审"而不用人搬运。内置 `/code-review` skill 就是这个形态。
- **双会话形态**:会话 A 写、会话 B 审,人把 B 的输出粘回 A。更重,但隔离更彻底(B 完全不受 A 的状态影响),也便于换模型或换人复核。同构的做法还有"A 写测试、B 写实现去过测试"。

**对抗评审的三个陷阱**(高分区):
1. **必然产出**(官方明确警告):被要求找 gap 的 reviewer 会找出 gap——**发现数量不等于代码质量**。反制:在 prompt 里定义什么算 finding(`report gaps, not style preferences`),并要求给严重度与依据。
2. **无锚标准**:不给需求文档,它就拿通用最佳实践当标准 → 报出的是"可以更好",不是"这里是错的"。反制:锚定 PLAN.md / SPEC / 验收标准,让它**逐条核对**而不是自由发挥。
3. **评审-修复死循环**:每轮修复产生新 diff,新 diff 又能被挑出新问题。反制:设收敛条件——固定轮次上限、只处理"影响正确性"那一档、或要求每条发现必须映射到一条既定需求。

还有一条容易漏的:**reviewer 不是自动判定的替代品**。它擅长测试测不到的东西——越界改动、需求漏项、缺失的边界测试、被悄悄放宽的断言;它不擅长替你跑一遍测试。正确组合是"机器判定(测试/构建/lint)兜正确性下限 + 对抗评审兜语义与范围",不是二选一(见 Q3 的四档叠加)。

**实例**(一条四要素齐全的评审 prompt):

```text
Use a subagent to review the rate limiter diff against PLAN.md. Check that
every requirement is implemented, the listed edge cases have tests, and
nothing outside the task's scope changed. Report gaps, not style preferences.
```

拆开看:**范围**(the rate limiter diff)、**标准**(PLAN.md)、**检查项**(需求覆盖 / 边界测试 / 越界改动)、**finding 判定线**(gaps not style)。缺任何一个,输出质量都会明显下滑。

**边界与误区**:
- 误区:"fresh context 一定更强"。它更中立,但**信息更少**;需求与有意取舍必须显式喂给它,否则中立换来的是噪声。
- 误区:把 reviewer 的发现当 todo 全改——官方点名这条导致过度工程。正确姿势是先分档,只有"影响正确性或既定需求"的必须改。
- 误区:在同一会话 `/clear` 之后审。`/clear` 能清对话,但你还得重新喂 diff 与标准——那本质就是双会话了,不如直接开 subagent,还能自动回传发现。
- 边界:审查者与被审者用**同一模型**仍有共同盲区(同源的错误偏好)。fresh context 解决的是"自我偏袒",不解决"同源盲区";高风险改动值得换模型或加人审。

**追问预判**:
- 追问:"reviewer 报了 12 条,你怎么处理?" → 答:按"影响正确性 / 影响既定需求 / 其余"三档分类;前两档改完复审,第三档进 backlog 不在本次动。更省事的做法是要求 reviewer 输出时自带分档。
- 追问:"能不能把 reviewer 自动化进 CI?" → 答:能(`claude -p` + `--output-format json` 解析发现),但要接受它的噪声率——CI 里应只让"影响正确性"这一档阻塞合并,其余作为评论呈现,否则会把 CI 变成过度工程发生器。

**关联**:[kb/04-multi-agent/subagents.md](../kb/04-multi-agent/subagents.md) · [kb/06-evals/agent-evals.md](../kb/06-evals/agent-evals.md) · [skills/claude-code/01](../skills/claude-code/01-core-workflow.md) · [01-Q5 上下文管理](01-agent-fundamentals.md)
