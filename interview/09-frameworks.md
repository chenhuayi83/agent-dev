---
last_updated: 2026-08-19
type: interview
topic: 框架与选型
questions: 5
---

# 面试题:09 框架与选型

> 每题讲透五件套:本质 / 机制 / 实例 / 边界 / 检验(追问预判)。答案分 30 秒版 + 深入版。
> 本组的判分线:**给判据,不站队**。说"某框架好"是零分,说"缺 X 时它赢、缺 Y 时它输"才是及格。

## Q1:Claude Agent SDK / OpenAI Agents SDK / LangGraph / Google ADK 各自的定位、抽象与适用场景?

**考察点**:能否落到"核心抽象是什么"而不是背特性清单;能否说出每个框架的失分场景。

**30 秒版**:四者不在同一层,不是同题竞品。**Claude Agent SDK** = 把 Claude Code 的 harness 开放出来,抽象单位是**会话 + 工具环境**(内置工具、subagent、hooks、权限);**OpenAI Agents SDK** = 轻量 runtime,抽象单位是 **Agent / handoff / guardrail**(控制权在 agent 之间转交);**LangGraph** = 显式状态机,抽象单位是 **state / node / edge + checkpointer**(编排与持久化);**Google ADK** = 企业 agent 交付链路,抽象单位是 **agent 组合(LlmAgent + Sequential/Parallel/Loop workflow agent)+ 部署/评估/治理**。选型只问一句:你缺的是 harness、是转交语义、是可持久化状态机,还是企业交付链路?

**深入版**:

四者的横向对照(as-of 2026-08-19,能力细节以各家官方文档为准):

| | Claude Agent SDK | OpenAI Agents SDK | LangGraph | Google ADK |
|---|---|---|---|---|
| **核心抽象** | session + tools + subagent + hooks + permission | Agent / handoff / guardrail / session / tracing / runner | StateGraph:state + node + edge;checkpointer + thread | LlmAgent、workflow agent(Sequential/Parallel/Loop)、graph agent(ADK 2.0);session/state/memory/artifacts |
| **控制流谁决定** | 模型(loop 内自主),harness 给护栏 | 模型(agent 自主选择 handoff),runner 跑循环 | 开发者(图是显式声明的),节点内可放模型 | 混合:workflow agent 显式,LlmAgent 自主 |
| **开箱自带** | 编码向工具集、上下文压缩、子代理隔离、权限/hook 审计点 | tracing、guardrail、sessions、HITL 暂停恢复 | persistence、HITL 中断/恢复、time travel | 工具生态、evaluation、A2A、Vertex Agent Engine / Cloud Run / GKE 部署 |
| **模型绑定** | Claude(强绑) | OpenAI 优先,可经 LiteLLM 等适配器接他家 | 模型无关(纯编排层) | model-agnostic(Gemini/Claude/OpenAI/Ollama/vLLM,经 LiteLLM) |
| **赢在** | 文件系统/代码类长任务;想直接拿到一个已被打磨过的 harness | 多 agent 分工清晰、要开箱观测、已在 OpenAI 栈内 | 需要断点续跑、审批插入、回放、复杂静态拓扑 | Google 云上企业交付,多 agent + 评估 + 治理一体 |
| **输在** | 跨模型、非编码域、需要显式图拓扑 | 复杂状态机与 durable 需求、跨厂商可移植性 | 简单任务上抽象过重;调试要先读懂图 | 离开 Google 栈价值快速衰减;抽象层数多 |

**机制推导:为什么抽象会长成这四种形状**——因为四家赌的"最小单位"不同:

1. 以**环境**为单位(Agent SDK):赌注是"agent 的上限由 harness 决定"。所以它卖的不是编排,是被 Claude Code 生产验证过的工具集、上下文管理策略、子代理隔离与权限模型。你买的是默认值的质量。
2. 以 **agent 与转交**为单位(Agents SDK):赌注是"分工靠 handoff"。控制流仍在模型手里,框架只提供交接协议 + 输入输出校验 + trace。抽象最薄,学习曲线最短。
3. 以**状态与转移**为单位(LangGraph):赌注是"生产系统需要可断点、可审批、可回放"。它把控制流从模型手里**拿回来一部分**,换取状态可持久化——这是它一切能力的根。
4. 以**组合与交付**为单位(ADK):赌注是"企业买的不是 loop,是从开发到部署到评估到治理的完整链路"。所以它同时给你 workflow agent 模板、eval、A2A 与三种部署目标。

一个判断技巧:**看框架把"上下文"藏得多深**。Agent SDK 帮你管上下文(优点也是黑箱),LangGraph 把 state 完全交给你(自由也是负担)——这条差异比任何 feature list 都更能预测你未来的调试体验。

**实例**(同一需求:"查库存 → 若不足则下采购单(不可逆,需人工批准)"的骨架对比,伪代码):

```python
# Claude Agent SDK 风格:环境即抽象——把审批做成 hook/permission,循环交给 SDK
client = ClaudeSDKClient(options={
    "tools": [check_stock, create_po],
    "hooks": {"PreToolUse": require_approval_if(tool="create_po")},   # 审批是横切关注点
})
await client.query("库存不足就补货")            # 何时调哪个工具:模型自己决定

# OpenAI Agents SDK 风格:agent + handoff——分工即抽象
inventory = Agent(name="inventory", tools=[check_stock], handoffs=[purchaser])
purchaser = Agent(name="purchaser", tools=[create_po],
                  input_guardrails=[amount_limit])                   # 护栏挂在 agent 上
Runner.run(inventory, "库存不足就补货")

# LangGraph 风格:状态机——转移即抽象,审批是图上一个显式的中断点
g = StateGraph(State)
g.add_node("check", check_stock); g.add_node("approve", human_gate); g.add_node("po", create_po)
g.add_conditional_edges("check", lambda s: "approve" if s["low"] else END)
app = g.compile(checkpointer=PostgresSaver(...))                     # 中断后可跨进程恢复
app.invoke(inp, config={"configurable": {"thread_id": "t-1"}})

# Google ADK 风格:组合 + 交付——层级即抽象
root = SequentialAgent(sub_agents=[LlmAgent(name="check", tools=[check_stock]),
                                   LlmAgent(name="po", tools=[create_po])])
# 之后走 ADK 的 eval 与 Agent Engine / Cloud Run 部署链路
```
四段代码解决同一问题,差别不在语法,在于**审批这件事被放在了哪儿**:hook(横切)/ guardrail(挂 agent)/ 图节点(显式状态)/ 层级子 agent。你的系统里这类横切关注点越多、越需要显式落盘,越应该往右选。

**边界与误区**:
- 误区一:把四者当同层竞品做 A/B。常见的生产组合是**外层 LangGraph 编排 + 节点内跑厂商 SDK 的 loop**——分层用,不是二选一。
- 误区二:按 star 数/热度选。判据永远是"你缺的那一块",不是社区规模。
- 误区三:忽略版本时效。LangGraph 1.0 / LangChain 1.0 于 **2025-10-22 GA**(核心图原语零 breaking,semver 承诺到 2.0,`langgraph.prebuilt` 弃用、能力并入 `langchain.agents`);ADK 有 2.0 引入 graph-based agent(as-of 2026-08-19)。面试里给版本要带 as-of,不确定就只讲机制。

**追问预判**:
- 追问:"Claude Agent SDK 和直接用 Messages API 手写 loop 差在哪?" → 答:差在**已调优的默认值**(工具集、compaction 策略、子代理隔离、hook/权限埋点)与厂商持续跟进升级;裸写换来的是完全可控与可移植。判据是你对默认值的满意度——满意就用,总在绕过它就该裸写。
- 追问:"handoff 和 edge 的本质区别?" → 答:handoff 是**运行时由模型选择**的控制权转移(动态,集合开放);edge 是**开发期声明**的转移关系(静态,条件边的分支集合仍是预先枚举的)。这正是 01 组 Q1 里"谁决定控制流"那条判据在框架层的投影。

**关联**:[kb/02-frameworks/claude-agent-sdk.md](../kb/02-frameworks/claude-agent-sdk.md) · [kb/02-frameworks/openai-agents-sdk.md](../kb/02-frameworks/openai-agents-sdk.md) · [kb/02-frameworks/langgraph.md](../kb/02-frameworks/langgraph.md) · [kb/02-frameworks/google-adk.md](../kb/02-frameworks/google-adk.md)

## Q2:什么时候**不该**用框架(直接裸写 agent loop)?框架的隐性成本是什么?

**考察点**:有没有"抽象税"意识;能否把隐性成本讲成可测量的东西,而不是抱怨。

**30 秒版**:判据一句话——**你的编排复杂度是否超过一个 while 循环**。单 agent + 一组工具 + 跑到完成,裸写通常一两百行,可读、可调、零绑定;只有当出现跨进程续跑、人工审批中断、复杂静态拓扑、团队要统一观测口径时,框架才开始赚回它的成本。隐性成本有五项:**调试面积、上下文所有权、版本税、抽象错配、运行时锁定**。

**深入版**:

先看裸写到底要写什么(这就是全部):

```python
messages = [user_msg]
while True:
    resp = client.messages.create(system=SYS, tools=TOOLS, messages=messages)
    messages.append(resp)
    if resp.stop_reason != "tool_use":
        break                                        # 模型自判完成
    results = run_tools_parallel(resp.tool_use_blocks)   # 执行 + 权限检查 + 错误回填
    messages.append(results)
    if turns > MAX_TURNS or tokens > BUDGET:
        break                                        # 硬护栏
```
框架替你做的是"其余的一切"。问题在于:"其余的一切"里,你真正需要的可能只有 10%,而 100% 的成本你都要付。

**五项隐性成本(逐条给可测的代理指标)**:

1. **调试面积翻倍**:bug 出在你的 prompt 还是框架注入的默认 prompt?框架会追加自己的系统提示、改写工具描述、包装错误格式——**你看到的 trace 不等于模型看到的 context**。可测指标:能否在 5 分钟内 dump 出实际发出的完整请求体。做不到的框架,后期每个玄学 bug 都要付这笔税。
2. **上下文所有权丢失**:上下文工程是 agent 质量的第一杠杆(见 [01 组 Q5](01-agent-fundamentals.md));框架把上下文管理藏起来,等于把你最大的调优杠杆锁进黑箱。当你需要定制"压缩时保留什么"时,才发现改不动。
3. **版本税**:即便有 semver 承诺,子包仍可能在 patch 内引入 breaking(LangGraph 1.0.2 等 patch 的实例:[issue #6363](https://github.com/langchain-ai/langgraph/issues/6363))。可测指标:近 6 个月 changelog 里 breaking 变更频次。任何框架都要按"每季度一个维护窗口"预算。
4. **抽象与领域错配**:框架抽象是**别人问题域的固化**。一旦错位,你会开始写"绕过框架的代码"——此时净收益为负。可测信号:代码里 escape hatch(直接调原生 API)的比例超过三成,就该重新选型或裸写。
5. **运行时与生态锁定**:不只是 API 锁定,更是 trace 只在某平台好看、部署只在某云顺畅、checkpoint 格式只有它认识——这是最贵的一层(见 Q5 的四层绑定定价)。

**反向清单(什么时候框架确实赢)**:a) 需要 durable execution / 崩溃续跑;b) 需要 HITL 中断-审批-恢复;c) 团队 >1 人且需统一 trace 与 eval 口径;d) 复杂静态拓扑(十几个节点、条件路由、并行汇聚);e) 合规要求可回放审计。命中两条以上,基本可以上框架。

**2026 的新变量**:很多过去"只有框架有"的能力,现在 API 层与 harness 层自带(见 Q3),裸写的性价比在上升;另一头,托管运行时(managed agents)又在把裸写的运维成本吃掉。所以裸写的合理区间是"复杂度低"和"需要极致掌控"两端,中间地带正在被两侧同时挤压。

**实例**(经验规则):**先裸写到"疼",疼点会告诉你该选哪个框架**。一个真实的演化路径——先裸写 150 行跑通;两周后发现"每次崩溃要从头跑 20 分钟"→ 疼点是 durability,指向带 checkpointer 的编排层;若疼点是"三个人的 trace 长得都不一样"→ 指向统一观测的框架;若疼点是"工具太多模型选错"→ 那根本不是框架问题,是上下文工程问题,换框架治不好。

**边界与误区**:
- 误区:"裸写 = 不用任何库"。裸写指**不用编排框架**;官方 SDK、tool runner 这类薄封装照用,它们不夺走上下文所有权。
- 误区:"框架 = 最佳实践"。框架的默认值是**它作者场景**的最佳实践,不是你的。
- 边界:团队工程能力参差时,框架的**约束本身是资产**——防止五个人写出五套不兼容的 loop。这时候选框架的理由是治理,不是技术。

**追问预判**:
- 追问:"裸写怎么避免重复造轮子?" → 答:把可复用部分抽成内部小库(上下文管理、重试与预算护栏、trace 埋点、工具注册)。这等于写了一个恰好匹配你领域的框架,而且**你拥有它**——出问题能改,升级由你排期。
- 追问:"怎么给隐性成本定量?" → 答:三个代理指标——P50 定位一个 agent 行为 bug 的时间、单次版本升级的工时、能否 5 分钟导出模型实际收到的完整 context。三个都差的框架,不管功能多全都要重新评估。

**关联**:[kb/02-frameworks/framework-selection.md](../kb/02-frameworks/framework-selection.md) · [kb/01-context-engineering/overview.md](../kb/01-context-engineering/overview.md) · [kb/00-fundamentals/tool-use.md](../kb/00-fundamentals/tool-use.md)

## Q3:"编排下沉"(multi-agent 编排进入 API 层与 harness 层)对框架层意味着什么?框架还剩什么不可替代的价值?

**考察点**:趋势判断力 + 能否说清"哪层做什么最合理"的机制原因,而不是复述新闻。

**30 秒版**:2026 的关键位移是 multi-agent 编排从"第三方框架的独家能力"变成 **API 层与 harness 层的原生能力**:OpenAI GPT-5.6(2026-07-09)在 Responses API 上提供原生 multi-agent(beta,模型可并行拉起 subagent 并汇总),同期还有 programmatic tool calling 与 persisted reasoning;Anthropic 把 subagent forking、跨会话消息做进 Claude Code / Agent SDK(2.1.232 起 subagent forking 默认开启,可继承会话与 cache)。结果是框架"我帮你跑多 agent"这个卖点被抽空。剩下的不可替代价值集中在四块:**durable execution / checkpointing、可观测与评估闭环、企业治理(权限/审计/多租户)、跨厂商可移植性**。

**深入版**:

**机制推导:为什么编排会往下沉?** 因为编排决策所依赖的信息,在下层最全:

1. **语义在模型端**。该并行还是串行、该拆几个子任务,依赖任务语义——模型有,框架只能看到你写死的图。把决策放在离信息最近的地方,是系统设计的一般规律。
2. **上下文在 API/harness 端**。subagent 的核心收益是**上下文隔离**,而上下文由 API/harness 持有。在框架层做隔离,要多走一遍序列化 + 摘要 + 重新组装,信息与 prompt cache 双重损耗;harness 内 fork 可以直接继承会话与缓存(Claude Code 的 subagent forking 正是这个设计)。
3. **成本结构**。一次请求内完成的分发,省掉 N 次网络往返与中间层组装,延迟与 token 都更低。

**框架层还剩什么(逐条给"为什么下层吃不掉")**:

| 剩余价值 | 为什么厂商 API/harness 不容易吃掉 | 什么系统真正需要 |
|---|---|---|
| durable execution / checkpointing | 需要外部持久化存储与事务语义,与模型无关;厂商不愿替你承担数据库与数据主权 | 跨天长任务、崩溃需续跑 |
| HITL 中断/恢复 | 中断点要落盘、要对接你的业务审批系统与身份体系 | 有不可逆动作的系统 |
| 观测/评估闭环 | 要跨模型、跨版本统一数据面并支持回放与回归 | 团队协作、需要 eval 驱动迭代 |
| 企业治理 | 多租户、审计、权限分层、合规是产品问题不是模型问题 | 受监管行业 |
| 跨厂商可移植 | 厂商原生编排天生只服务自己 | 多模型策略、规避单点 |

**但要诚实**:这几块也正在被上下夹击。托管运行时(managed agents)在把 durable execution 与观测做成平台能力——LangChain 2026-08-12 明确把"managed agents 是下一个大方向"当作押注,对应的是 Anthropic Managed Agents、Google Gemini Enterprise Agent Platform(2026-04-22 Next '26 发布)。所以独立框架层的终局更可能是:**要么向上变成托管平台(卖运行时),要么向下变成薄可移植层(卖跨厂商中立)**,卡在中间"只卖编排抽象"最危险。

**实例**(同一需求"并行调研 5 个来源再汇总"的三种落法):

| | 框架 supervisor 图 | harness subagent | API 原生 multi-agent |
|---|---|---|---|
| 谁决定拉几个 | 开发者写死或写路由逻辑 | 模型决定,harness 执行 | 模型决定,API 内完成 |
| 上下文隔离 | 要自己切分与回传 | 原生隔离,可继承会话/cache | 原生隔离 |
| 可插入审批点 | 容易(图上加节点) | 中等(hook/权限层) | 难(黑箱) |
| 失败重试策略 | 完全自定义 | 部分可定制 | 厂商定义 |
| 延迟/成本 | 最高(多次往返) | 中 | 最低 |
| 可移植性 | 高 | 绑 harness | 绑单一厂商 |

选择规则清楚了:**只读、可重跑的探索类并行 → 用原生**(便宜快);**含不可逆动作、需要审批与续跑 → 用框架/自建状态机**(可控);两者可共存,外层状态机 + 内层原生并行是 2026 的务实形态。

**边界与误区**:
- 误区:"编排下沉 = 框架要死"。下沉的是**编排**,不是持久化与治理;而且原生编排天然绑死单厂商,跨厂商需求会一直存在。
- 误区:"原生 subagent 一定更好"。原生编排的**可控性更低**:你插不进审批点、改不了重试策略、调试更黑箱。便宜是真便宜,失控也是真失控。
- 关键补充:Anthropic 2026-08-13 的多智能体研究提示,**基础设施变便宜不等于多 agent 变正确**——羊群效应、群体决策劣化(隐藏信息任务中群体准确率 17-36%,个体接近 100%)。编排变便宜会诱导过度使用,这是新的风险面。

**追问预判**:
- 追问:"那你新项目会用原生编排还是框架编排?" → 答:分层——探索/只读的并行调研走原生;有不可逆动作、需审批与续跑的走框架或自建状态机;并把两者的边界画在"这一步失败了能不能无副作用重跑"上。
- 追问:"编排下沉对你的技能储备意味着什么?" → 答:考点从"你会用哪个框架"移到"你怎么判断编排该放在哪一层",以及上下文工程、durability、评估这三项跨框架能力——它们不会随框架换代贬值。

**关联**:[kb/10-landscape/ecosystem-map.md](../kb/10-landscape/ecosystem-map.md) · [kb/04-multi-agent/orchestration-patterns.md](../kb/04-multi-agent/orchestration-patterns.md) · [kb/04-multi-agent/subagents.md](../kb/04-multi-agent/subagents.md) · [deep-dives/2026/2026-08-19-landscape-baseline.md](../deep-dives/2026/2026-08-19-landscape-baseline.md)

## Q4:Durable execution / checkpointing 类能力的价值边界在哪?什么系统必须要、什么系统是过度设计?

**考察点**:能否分辨"故障恢复"与"重跑就行";是否理解真正的门槛是**幂等与确定性**而不是存状态。

**30 秒版**:durable execution 的本质是**把执行状态外置成可恢复的检查点**,让进程崩溃/重启/人工等待之后能从断点继续,而不是从头重跑。价值边界由两个量决定:**重跑成本**(时间 × token × 已产生的副作用)与**中断概率**(任务时长 × 外部依赖数 × 是否有人工等待)。两者都低 → 直接重试整个任务,代码少一个数量级;任一高 → 需要 checkpoint。真正的门槛不是存状态,是**副作用的幂等性**——恢复靠重放,重放会重复执行工具。

**深入版**:

**机制**:checkpointer 在每个节点/步骤边界把状态快照写进存储,按会话(thread)组织;恢复时读最后一个 checkpoint 重建状态继续。LangGraph 的实现是 `InMemorySaver` / `SqliteSaver` / `PostgresSaver`,通过 `{"configurable": {"thread_id": ...}}` 定位线程(as-of 2026-08 官方文档;更细的 durability 模式与 interrupt/resume 参数名以官方文档为准)。三个能力其实是同一机制的推论:
- **HITL** = 主动在某点停下并落盘,人工批准后带着输入恢复;
- **time travel** = 回到任意历史 checkpoint、改状态、重跑分支;
- **崩溃续跑** = 恢复语义的默认用法。

**必须要的系统(四类)**:
1. **长时任务**(小时到天):中断概率随时长上升,重跑成本也随时长上升——**两者相乘是二次的**,这是最硬的理由。
2. **有人工审批环节**:审批天然是长等待(小时级),进程不可能一直挂着占资源。
3. **有昂贵或不可逆副作用**:重跑 = 重复付款/重复发信/重复建单,系统必须知道"哪一步已经做过"。
4. **合规要求可审计回放**:checkpoint 序列本身就是执行轨迹的审计证据。

**过度设计的信号**:
- 单次任务 < 几分钟且**纯只读** → 直接整任务重试,少一个数量级的代码与运维。
- 已有更简单的外置状态就够用:把计划写文件、进度写 state、阶段性 git commit——这就是"穷人版 checkpoint"。本仓库自身正是这个模式(PROTOCOL + state.json + queue + 每阶段 commit),不需要 durable 引擎。
- **幂等性还没解决就上 durable**:恢复会重放,重放会重复副作用——比不做更危险。

**真正的难点(这是区分性内容)**:
- **确定性要求**:恢复靠重放。节点里取随机数、读当前时间、直接调外部写接口,重放时都会产生不一致。工程做法是**把副作用收敛到明确的步骤边界并持久化其结果**,重放时读已存结果而不是重做——这是所有 durable execution 引擎(包括 Temporal 一类通用工作流引擎)的共性要求,不是某个框架的特殊规定。
- **幂等键**:每个外部动作带业务幂等键(订单 id、消息 id、请求 id),让重复调用无害。没有幂等键,durable 给你的是"更可靠地重复犯错"。
- **状态膨胀**:agent 状态里含完整 messages 历史,高频 checkpoint 会把存储写爆。做法是分层——历史外置存储、state 里只放指针与决策摘要。
- **恢复语义的诚实选择**:at-least-once(可能重复,靠幂等兜底)是默认;"恰好一次"在有外部副作用时不可能免费获得,需要幂等 + 事务边界配合。面试里能主动区分这两者,是加分点。

**实例**(两个方向的极端):
- **必须 durable**:"批量退款 500 笔"——不可逆副作用 + 长时 + 部分失败常态。设计:每笔一个步骤边界,退款结果与幂等键一起落盘,恢复时跳过已完成笔次;失败笔次进死信队列人工处理。
- **不该 durable**:"读代码库回答一个架构问题"——只读、几分钟、失败重跑成本 = 一次上下文。加 checkpointer 只是给自己增加一个存储依赖和一类新 bug。

**边界与误区**:
- 误区:"checkpoint = 数据备份"。它恢复的是**执行位置与状态**,不是你的业务数据。
- 误区:"上了 durable 就可靠了"。可靠性 = durable + 幂等 + 护栏(轮次/预算/超时)+ 验证闭环,缺一项都不成立。
- 权衡:checkpoint 频率是**延迟/存储 × 恢复粒度**的交换。每步落盘恢复最细也最慢;按阶段落盘更实用。
- 边界:如果你的中断主要来自**模型犯错**而不是进程崩溃,durable 帮不上忙——那是评估与护栏的问题。

**追问预判**:
- 追问:"不用框架怎么实现 durable?" → 答:把 agent 状态设计成可序列化(messages 引用 + 已完成步骤及结果 + 外部动作幂等键),每个步骤边界写存储;或者直接用通用工作流引擎(Temporal 一类),把 LLM 调用当作活动。框架只是把这套封装了,机制并不神秘。
- 追问:"checkpoint 里该存什么?" → 答:存"重建现场的最小必需集"——关键决策、已完成动作及其**结果句柄**、待办清单;可再生的中间产物(比如完整工具输出)存指针不存正文,否则存储成本随轮次爆炸。

**关联**:[kb/08-production/reliability.md](../kb/08-production/reliability.md) · [kb/05-memory/state-persistence.md](../kb/05-memory/state-persistence.md) · [kb/07-safety-security/permissions-hitl.md](../kb/07-safety-security/permissions-hitl.md)

## Q5:给定一个真实项目,你怎么做框架选型?迁移成本与厂商绑定怎么评估?

**考察点**:有没有一套可复述、可落地、可被别人复核的方法,而不是"看情况"。

**30 秒版**:四步:① 写**约束清单**(任务形态、自主性、时长、模型策略、合规、团队、已有栈);② 把约束翻译成**能力需求并排序**,只认前三条硬需求;③ 用**一周 spike** 让两个候选跑同一条**最难的真实链路**(不是 hello world),测四个可观测量;④ 按**可逆性**决策——先选最容易改主意的那个,并在代码里筑好隔离层。厂商绑定不看"用没用某家",看**换掉它要改多少行、丢多少能力、停多久**。

**深入版**:

**第一步:约束清单(八问)**
1. 控制流可枚举吗?(决定 workflow / agent / 显式图)
2. 单 agent 够吗?(默认起点永远是单 agent + subagent 隔离)
3. 有不可逆动作或人工审批吗?(决定要不要 durable + HITL)
4. 任务时长量级?(秒/分 vs 小时/天)
5. 模型策略:锁定单一厂商,还是必须可换?
6. 团队语言与人力(Python/TS;有没有人能读懂框架源码)
7. 合规与部署边界(是否必须私有部署、数据能否出网)
8. 已有可观测栈(要接现成 OTel/trace 平台吗)

**第二步:硬需求 → 候选方向(判据表,不是排行榜)**

| 排第一的硬需求 | 首选方向 | 理由 |
|---|---|---|
| 编码/文件系统类长任务,想直接得到强 harness | Claude Agent SDK | 内置工具 + 上下文管理 + 子代理是其护城河 |
| 多 agent 分工清晰、要开箱 tracing、已在 OpenAI 栈内 | OpenAI Agents SDK | handoff/guardrail/tracing 直给,抽象最薄 |
| 崩溃续跑、审批中断、复杂拓扑、要回放 | LangGraph | checkpointer + HITL 是其存在理由 |
| Google 云上企业交付,多 agent + 评估 + 治理一体 | Google ADK | 与 Vertex / Gemini Enterprise 链路打通,A2A 原生 |
| 需求简单、要完全掌控上下文 | 裸写 loop | 抽象税 > 收益(见 Q2) |

**第三步:一周 spike 的四个可观测量**
1. **最难用例通过率**:选你系统里最讨厌的那条链路,不是 demo。demo 通过率没有信息量。
2. **可观测性**:能否 5 分钟内导出模型实际收到的完整 context 与工具轨迹。
3. **逃生门**:框架不支持时,能否平滑下降到原生 API 调用(escape hatch 的存在性 = 未来的风险上限)。
4. **维护面**:近 6 个月 changelog 的 breaking 频次(patch 内的 breaking 也要计入,如 LangGraph 1.0.2 那类事件)。

**第四步:把"厂商绑定"拆成四层分别定价**

| 层 | 内容 | 迁移代价 | 压制手段 |
|---|---|---|---|
| 1 模型绑定 | prompt、工具定义、模型特性 | 低:大多可搬,成本在重新调优 + 评测重跑 | 保留一套模型无关的评测集 |
| 2 编排绑定 | 图结构 / handoff 拓扑 | 中:要重写,但结构是**你的设计**,文档化后可重建 | 编排逻辑保持薄,业务不写进节点 |
| 3 状态与持久化绑定 | checkpoint 格式、session 存储、历史数据 | 高:真正的沉没成本 | 用自有库表或明确可导出的格式 |
| 4 运行时/平台绑定 | 托管运行时、观测平台、部署形态 | 最高:换 = 换基础设施 | 观测走 OpenTelemetry 一类通用语义 |

由此得到设计原则:**把绑定尽量压在第 1、2 层**——业务逻辑(工具实现、领域提示、评测集、幂等键策略)必须住在框架之外的自有模块里。一句话估法:**迁移成本 ≈ 被框架抽象吞掉的代码量 × 抽象与新框架的语义差**。"框架越薄越好换"不是口号,是可算的。

**实例**(走一遍真实场景):

> 客服工单 agent:读工单 → 查三个内部系统 → 必要时退款(不可逆,需审批)→ 回复客户。平均 3 分钟,峰值排队;团队 3 人 Python;已有 Postgres 与 OTel。

- 硬需求排序:(1) 不可逆动作 + 审批 → 必须 HITL;(2) 峰值排队与审批等待 → 需要跨进程恢复;(3) 接现有 OTel。模型策略:暂用单一厂商,但要求"能换"。
- 结论:上一个带 checkpointer/HITL 的编排层(LangGraph 方向),**但**三个内部系统的调用、退款幂等键、领域提示与评测集全部放在框架外的 `domain/` 包;checkpoint 存自家 Postgres;trace 走 OTel 而非平台专有 SDK。绑定被压在第 2 层,将来若原生编排够用,重写的只是外层。
- 反例:同一团队若只做"读工单生成回复草稿"(只读、秒级、无审批),正确答案是裸写 loop + 一次重试。**同一个团队、同一个域,需求变一点,答案就翻转**——这正是选型题的考点。

**边界与误区**:
- 误区:选型是一次性决策。它是**有半衰期的决策**——2026 的答案(编排下沉、托管运行时兴起)明年可能变,所以**隔离层比选型本身更重要**。
- 误区:用 star 数 / 融资额做判据。更实在的三项:近 6 个月 breaking 频次、issue 响应质量、你的团队能不能读懂它的源码。
- 误区:为"未来可能的复杂度"提前上重框架。YAGNI 在这里成立:复杂度真来了,你会更清楚自己需要什么。
- 诚实边界:如果团队已在某云/某模型上重度绑定,"避免绑定"的边际收益很低,选与既有链路最顺的那个(如 Google 栈选 ADK)通常就是对的——**一致性也是一种工程价值**。

**追问预判**:
- 追问:"我们已经在用 X 了,该迁移吗?" → 答:先确认迁移动机是不是当下真实痛点(可观测?稳定性?成本?锁定?),再按四层绑定表估价。多数情况正确答案是**局部替换而非整体迁移**:新模块用新方案写,双轨跑一个季度,用数据而不是偏好定去留。
- 追问:"怎么向 leader 证明选型合理?" → 答:交付两份东西——spike 的四个量化结果,以及一张风险登记表(每个候选的最大风险 + 缓解措施 + 触发重估的条件)。把决策变成**可复核的记录**,而不是个人品味。

**关联**:[kb/02-frameworks/framework-selection.md](../kb/02-frameworks/framework-selection.md) · [kb/02-frameworks/other-frameworks.md](../kb/02-frameworks/other-frameworks.md) · [kb/10-landscape/ecosystem-map.md](../kb/10-landscape/ecosystem-map.md) · [kb/08-production/deployment-patterns.md](../kb/08-production/deployment-patterns.md)
