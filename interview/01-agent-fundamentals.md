---
last_updated: 2026-08-19
type: interview
topic: Agent 基础
questions: 6
---

# 面试题:01 Agent 基础

> 每题讲透五件套:本质 / 机制 / 实例 / 边界 / 检验(追问预判)。答案分 30 秒版 + 深入版。

## Q1:什么是 AI agent?它和 workflow、chatbot 的本质区别是什么?

**考察点**:概念边界是否清晰;能否用一个可操作的判据而不是背定义。

**30 秒版**:agent 是"LLM 在循环中自主使用工具":模型根据目标决定调用什么工具,读取环境反馈,再决定下一步,直到判断任务完成。判据是**控制流由谁决定**——workflow 的每一步由代码预先编排,LLM 只是步骤里的函数;agent 的下一步由模型在运行时根据反馈动态决定。chatbot 则只产出文本,不作用于环境。

**深入版**:

三者的分界线在两个维度:**谁决定控制流** × **是否作用于环境**。
- chatbot:人决定控制流(一问一答),不作用于环境;
- workflow(pipeline/chain):代码决定控制流(写死的 DAG),LLM 是其中的变换函数——可预测、可测试、成本可控,但遇到未编排的分支就僵住;
- agent:模型决定控制流(loop + 分支 + 终止判断),通过工具作用于环境——灵活、能处理开放任务,但引入不确定性:延迟、成本、失败模式都变成分布而非定值。

"自主性"是谱系不是开关:从"每步需人批准"到"全自动无人值守",生产系统通常落在中间(高风险动作留审批,其余自动)。Anthropic 的经典建议是**能用 workflow 解决就不要用 agent**——自主性是为"无法预先枚举步骤"的任务保留的开销。

**实例**:"每天定时拉取日志→LLM 总结→发邮件"是 workflow(步骤固定);"帮我查清这个 bug 并修好"是 agent 任务(读哪些文件、跑什么命令、修几轮,都由模型在运行时决定)。同一产品里两者常混合:agent 的某个工具内部就是一条 workflow。

**边界与误区**:
- 误区一:把"调了工具"当 agent。单次 function calling 后由代码接管,仍是 workflow。
- 误区二:把 agent 当高级货。可预测任务上 workflow 更便宜、更稳、更好测。
- 灰区:带条件分支的 workflow(router)接近 agent,但分支集合仍是预先枚举的——判据还是"运行时谁决定下一步"。

**追问预判**:
- 追问:"什么任务你会坚持用 workflow?" → 答:步骤可枚举、失败成本高、需要审计回放的(如账单处理);举例说明 agent 化只会增加不确定性。
- 追问:"自主性怎么分级落地?" → 答:按动作风险分级(只读自动/可逆自动/不可逆审批),引用 human-in-the-loop 设计。

**关联**:[kb/00-fundamentals/agent-definitions.md](../kb/00-fundamentals/agent-definitions.md)

## Q2:描述 agent loop 的完整一轮:从用户输入到任务完成,内部发生了什么?

**考察点**:是否真的理解 harness 与模型的分工;能否讲清数据流。

**30 秒版**:harness 把系统提示+工具定义+历史+用户消息组装成请求发给模型;模型输出要么是文本(回合结束),要么是工具调用;harness 执行工具、把结果作为新消息追加,再次请求模型——如此循环,直到模型不再要求调工具。模型只"看请求、出下一步",循环、执行、状态都在 harness。

**深入版**:

一轮的完整数据流(以 Anthropic Messages API 形态为例):
1. **组装**:system prompt(角色/规则)+ tools(每个工具的 name/description/JSON Schema)+ messages(全部历史)→ 一次无状态 API 调用。模型没有记忆,"记忆"就是 messages 数组本身。
2. **决策**:模型输出 assistant 消息,内容块可能是 text、thinking、或一个/多个 `tool_use` 块(含工具名与结构化参数)。`stop_reason: tool_use` 表示模型在等工具结果。
3. **执行**:harness(不是模型!)解析 tool_use,做权限检查,真正执行(跑命令/读文件/调 API),把结果包成 `tool_result` 追加进 messages。错误也作为 tool_result 回传——模型看到报错自己调整,这是 agent 自愈能力的来源。
4. **迭代**:带着新增的 tool_result 再次调用模型。循环的终止条件:模型输出纯文本(自判完成)、达到轮次/预算上限、或用户打断。

关键认知:**agent = 模型 × harness 的乘积**。上下文管理(compaction)、并行工具执行、权限、重试,全是 harness 的工程;模型决定"下一步做什么",harness 决定"这一步怎么发生"。同一模型换 harness,表现可以差一个档次(Artificial Analysis Coding Agent Index 用同模型不同 harness 的分差实证了这点,as-of 2026-05)。

**实例**:Claude Code 里问"修复这个测试":模型先 `tool_use: Bash("npm test")` → harness 执行并回传失败输出 → 模型 `tool_use: Read(测试文件)` → `tool_use: Edit(源文件)` → 再 `Bash("npm test")` → 通过,输出文本总结。四次工具调用就是四轮 loop。

**边界与误区**:
- 误区:"模型自己执行了命令"。模型永远只输出"我想调 X,参数 Y"的结构化意图,执行权全在 harness——这也是权限/沙箱设计的落点。
- 误区:把多轮 loop 与多轮对话混淆。一个用户回合内可以有几十轮 model↔tool 循环。

**追问预判**:
- 追问:"loop 怎么终止才安全?" → 答:模型自判 + 硬护栏(最大轮次/token 预算/超时)+ 验证闭环(测试通过才算完成,而非模型说完成)。
- 追问:"并行工具调用怎么处理?" → 答:模型一次输出多个 tool_use 块,harness 可并发执行,结果按 id 对应回填(见 Q3)。

**关联**:[kb/00-fundamentals/tool-use.md](../kb/00-fundamentals/tool-use.md) · [skills/claude-code/01](../skills/claude-code/01-core-workflow.md)

## Q3:Function calling / tool use 的底层机制是什么?模型是怎么"学会调工具"的?

**考察点**:对"结构化输出+约定协议"的理解深度;schema 设计的工程意识。

**30 秒版**:工具调用的本质是**约定格式的结构化输出**。工具以 JSON Schema 形式进入上下文;模型被训练成在需要时输出符合 schema 的 `tool_use` 块;运行时(harness/API 层)解析、执行、把结果回填。所谓"调用"从头到尾只是文本进出,模型从未接触真实函数——因此 schema 质量与 description 写法直接决定调用成功率。

**深入版**:

三个层次拆开:
1. **训练层**:模型在后训练中大量学习"给定工具清单+任务 → 输出合法调用序列"的数据,习得(a)何时该用工具而非直接回答;(b)如何生成合法 JSON 参数;(c)如何消费 tool_result 继续推理。这是能力的来源——不是运行时魔法。
2. **协议层**:请求里每个工具 = `{name, description, input_schema(JSON Schema)}`。工具定义占 token(几十个工具可吃掉数千 token),本身就是上下文成本。API 可用 `tool_choice` 强制/禁止/指定工具。部分平台在解码时做约束(constrained decoding/strict mode),从机制上保证参数合法——把"模型大概率对"变成"结构上必对"(as-of 2026-08 各家旗舰 API 均已支持 strict 工具参数)。
3. **工程层**(高分回答的区分点):
   - **description 是给模型的 API 文档**:写清楚"何时用/何时不用/参数含义/返回什么",错误调用率显著下降;
   - **并行调用**:一次输出多个 tool_use 块,harness 并发执行——独立读操作应并行(延迟砍半),有依赖的必须串行;
   - **错误即数据**:执行失败把错误文本作为 tool_result 回传,模型会重试/换路——比 harness 静默重试更能利用模型智能;
   - **工具数量**:工具过多会稀释选择准确率,大型系统用"工具检索/分组按需注入"(如 MCP 场景下 ToolSearch/延迟加载,as-of 2026-08 已是常见 harness 手法)。

**实例**(一次天气查询的报文骨架):
```
→ tools:[{name:"get_weather", input_schema:{city:string}}] + user:"北京多热?"
← assistant: tool_use{id:t1, name:"get_weather", input:{city:"北京"}}   (stop_reason: tool_use)
→ messages 追加 tool_result{tool_use_id:t1, content:"34°C"}
← assistant: text:"北京现在 34°C……"
```

**边界与误区**:
- 误区:"function calling 是模型在执行函数"——模型只生成意图,执行在外部(安全边界也在这)。
- 误区:把所有能力塞成工具。可用代码后处理的(格式转换、聚合)别浪费模型轮次。
- 边界:极长工具清单 + 极深嵌套 schema 是失败高发区——拆扁、拆组、写例子。

**追问预判**:
- 追问:"工具返回超长怎么办?" → 答:harness 侧截断/摘要/分页,或让工具返回句柄(文件路径)由模型按需再读——本质是上下文预算管理。
- 追问:"怎么评测工具调用质量?" → 答:构造带标准答案的调用轨迹集,测选择准确率/参数合法率/多轮修复率(引 tau-bench 类基准)。

**关联**:[kb/00-fundamentals/tool-use.md](../kb/00-fundamentals/tool-use.md) · [kb/01-context-engineering/structured-outputs.md](../kb/01-context-engineering/structured-outputs.md)

## Q4:ReAct 是什么?为什么 2026 年很少有人再手写 ReAct 提示了?

**考察点**:经典范式的历史意义 + 对"能力内化进模型/harness"这一演化主线的判断力。

**30 秒版**:ReAct(Reason+Act,Yao et al. 2022)是让模型交替输出 Thought(推理)/ Action(工具调用)/ Observation(结果)的提示范式,证明了"推理与行动交替"远胜纯生成。今天很少手写,是因为这个模式被**内化**了:模型原生具备 thinking/interleaved thinking 能力,tool use 进了 API 协议,循环进了 harness——ReAct 从"提示技巧"变成了"基础设施的默认形态"。

**深入版**:

ReAct 的历史贡献:在只有文本补全的时代,用 few-shot 提示强行搭出 agent loop——`Thought: 我需要查X → Action: search[X] → Observation: ...` 的文本轨迹,让"推理指导行动、观察修正推理"成为可能,大幅降低幻觉式行动。

它为什么被淘汰(三层内化,as-of 2026):
1. **推理内化**:旗舰模型原生支持 extended thinking(思考预算可调),并能在工具调用之间穿插思考(interleaved thinking)——不需要用 "Thought:" 前缀骗出推理;
2. **行动内化**:tool use 成为 API 一等公民(结构化 tool_use 块 + strict 参数),不再靠解析自由文本里的 `Action: search[X]`(脆弱、易注入);
3. **循环内化**:执行循环、重试、并行调用由 harness/SDK 承担(Claude Agent SDK、OpenAI Agents SDK 等),开发者不再手搓 while 循环解析文本。

留下的遗产:ReAct 的**结构思想**仍是每个 agent 系统的骨架——今天你在 Claude Code 里看到的"thinking → tool 调用 → 读结果 → 再 thinking"就是 ReAct 的工业化形态。理解 ReAct 依然值钱,因为它是理解现代 harness 在自动化什么的最短路径。

**实例**:2022 手写版 vs 2026 等价物——
- 2022:提示里塞 few-shot 的 Thought/Action/Observation 范例 + 停止词解析;
- 2026:`tools=[...]` + extended thinking 开关,模型自主决定何时思考何时调用,SDK 跑循环。行为同构,分工彻底不同。

**边界与误区**:
- 误区:"ReAct 过时了所以不用学"——范式没死,是下沉了;面试讲不清内化路径才是真扣分。
- 仍需显式结构的场景:弱模型/本地小模型(原生 tool use 不可靠时,ReAct 式提示仍是兜底)、需要强制可审计推理轨迹的合规场景。
- 相关范式定位:plan-and-execute(先全局计划再执行)与 ReAct(边想边做)是互补策略,现代系统常混用(Claude Code 的 plan mode ≈ 前者,执行期 ≈ 后者)。

**追问预判**:
- 追问:"extended thinking 和 CoT 提示的区别?" → 答:CoT 是提示技巧(输出里写推理),extended thinking 是训练出的原生能力+独立思考预算,可与工具调用交错,且思考内容可不进最终输出。
- 追问:"什么时候显式 plan 仍有增益?" → 答:多文件大改动/不熟代码库/需要人审计划本身时(引 Claude Code plan mode 的官方适用判据:一句话说不清 diff 就先 plan)。

**关联**:[kb/00-fundamentals/planning-reasoning.md](../kb/00-fundamentals/planning-reasoning.md)

## Q5:长任务 agent 的上下文塞满了怎么办?讲讲你的分层方案。

**考察点**:上下文工程的系统观——这是 2026 年 agent 工程师的核心区分性技能。

**30 秒版**:分四层防线:**省**(不让无关内容进上下文:工具输出截断、subagent 隔离调查)、**压**(满了压缩:compaction 摘要旧轮次,保留决策与文件状态)、**存**(重要状态外置到文件/存储,上下文只留指针:计划、进度、结论落盘)、**续**(压缩/重启后能从外置状态恢复:checkpoint + 断点续跑设计)。原则是把上下文当缓存用,而不是当硬盘用。

**深入版**:

问题的物理根源:上下文有限且**性能先于容量耗尽**——远未到窗口上限,注意力质量就开始退化(遗忘早期指令、错误率上升)。所以方案不是"买更大窗口",而是分层管理:

1. **省(admission control)**:最便宜的一层。
   - 工具输出限幅:读文件只读需要的行段,命令输出截尾;
   - **subagent 隔离**:大规模探索(读几十个文件找答案)交给独立上下文的子代理,主上下文只收结论摘要——"过程消化在别处,结论带回来";
   - 按需加载:工具/知识延迟注入(用到才加载 schema 或文档),而非启动即全量。
2. **压(compaction)**:满了再救。自动或手动把旧对话压成摘要,**保什么**是关键:已改文件清单、关键决策、待办、测试命令——可在系统层定制(如 Claude Code 支持在 CLAUDE.md 里指定 compaction 保留项,以及 /compact 带指令、按 checkpoint 区间摘要,as-of 2026-08)。压缩必然丢信息,所以它是第二道而非第一道防线。
3. **存(externalize)**:根本解。把状态写出上下文——计划写 PLAN.md、进度写 state 文件、结论写文档,上下文只保留"去哪读"的指针。这也是跨会话记忆的基础:文件式记忆(memory 目录/CLAUDE.md/auto memory)、或向量/图式记忆系统。判据:**凡是"明天还需要的",都不该只活在上下文里**。
4. **续(resume)**:工程闭环。阶段性 checkpoint(git commit / state 快照)+ 启动自检(读外置状态、判断从哪续)→ 会话被杀/压缩失真都能恢复。长任务 agent 的可靠性来自"任意时刻可重建现场",而不是祈祷上下文不丢。

**实例**(本仓库自身,四层各落到什么文件):

| 层 | 本系统的实现 |
|---|---|
| 省 | 大批量写作交给 subagent 在独立上下文完成,主会话只收一行确认;简报只精读 ≤4 篇原文 |
| 压 | 长会话触发 compaction 时,协议要求保留:已改文件清单、当前阶段、待办 |
| 存 | 协议 `meta/PROTOCOL.md`、状态 `meta/state.json`(last_run/pending/counters)、队列 `meta/queue.md`、去重索引 `meta/seen.jsonl` 全部外置为文件 |
| 续 | 每阶段 `git commit` 作 checkpoint + `pending` 字段记录断点;新会话零上下文启动,靠读文件重建现场 |

一句话:**上下文只是工作台,git 仓库才是记忆**。

**边界与误区**:
- 误区:"上下文越大越好,塞满信息模型更聪明"——无关信息是噪声,会主动降低质量(kitchen sink 反模式);
- 误区:全靠 compaction——压缩丢失不可控,重要信息必须走"存"层;
- 权衡:隔离(subagent)有通信成本,单次小任务直接做更快——分层是菜单不是流水线,按任务规模取用。

**追问预判**:
- 追问:"prompt caching 和这套什么关系?" → 答:缓存降低重复前缀的成本与延迟(系统提示/工具定义命中缓存),但不解决窗口占用与注意力退化——省钱不省注意力,两者正交。
- 追问:"多 agent 是不是上下文问题的银弹?" → 答:subagent 隔离确实是"省"层利器,但引入协调成本与新失败模式(引 Anthropic 2026-08 多智能体研究:群体决策可能劣于个体),范围要克制。

**关联**:[kb/01-context-engineering/overview.md](../kb/01-context-engineering/overview.md) · [kb/05-memory/state-persistence.md](../kb/05-memory/state-persistence.md) · [skills/claude-code/01](../skills/claude-code/01-core-workflow.md)(原理一同源)

## Q6:除了 ReAct,还有哪些 agent 设计范式?Plan-and-Execute、Reflection/Reflexion、Tree-of-Thoughts 各解决什么问题、什么时候用?

**考察点**:能否把范式当作**带成本函数的搜索算法**来横向比较,而不是背四个名词;能否判断哪些今天还需要显式实现。

**30 秒版**:这几个范式其实是同一件事的不同形状:**用更多推理量/更多轮次换正确率**。ReAct 是线性搜索(边做边看),Plan-and-Execute 把"规划"从执行里拆出来单独做一次,Reflection/Reflexion 加一条"评判→重来"的回边,Tree-of-Thoughts 把线性轨迹展开成带回溯的树。因为它们都是搜索,收益就取决于同一件事:**你有没有一个比生成更便宜的验证/排序手段**。有廉价 verifier(测试、编译器、schema)时搜索很划算;没有 verifier 时,多采样只会得到更多**自信的错误答案**。所以选择的两个轴是**错误代价**与**可验证性**。

**深入版**:

**1. 统一视角:四个范式 = 四种搜索形状**

| 范式 | 控制流形状 | 多消耗什么 | 换来什么 | 成立的前提 |
|---|---|---|---|---|
| ReAct | 线性:think→act→observe 循环 | 每步一次调用 | 用真实观察纠正推理 | 环境能给出反馈 |
| Plan-and-Execute | 两段:planner 一次出全局计划 → executor 逐步执行 → replanner 修计划 | 一次昂贵的规划调用 | 全局一致性;executor 不必携带全部历史 | 步骤在**看到观察之前**就大致可枚举 |
| Reflection(self-refine) | 轮内回边:generate → critique → revise | 每次修订 ≥2× 调用 | 修掉"一遍过"的低级错误 | **critic 比 generator 有信息优势** |
| Reflexion | 跨试次回边:失败 → 把教训写成自然语言存入 memory → **整个任务重来** | N 次完整试次 | 跨试次不重蹈覆辙 | 存在**试次级别的成功/失败信号**,且任务可重来 |
| Tree-of-Thoughts | 树:每步生成 k 个候选 → 打分 → BFS/DFS + **回溯** | 分支 × 深度,常几十倍 token | 能走回头路 | 存在**对中间状态**打分的 evaluator |

看最后一列——这是本题的题眼。四行的前提各不相同,但都属于同一类东西:**一个外部的、比生成更便宜的判别信号**。

**2. 核心判据:generator-verifier gap**

搜索的收益 = 候选质量的方差 × **你挑对的能力**。生成更多候选很容易,**挑**才是瓶颈。所以决定"上不上范式"的是两个量:**错误代价**(一次错的后果多大:可回滚的 lint 错 vs 打款给错账户)和**可验证性**(验证一个候选比生成它便宜多少)。推出一个比背名词有用得多的 2×2:

| | 有廉价 verifier | 无廉价 verifier |
|---|---|---|
| **错误代价高** | **上搜索/反思循环**:verifier 会剪枝,多花的轮次值 | **别靠多采样**,上 human-in-the-loop / 计划审批 |
| **错误代价低** | 直接重试,别造范式 | 单次通过即可 |

右上角那格最能区分人:很多人下意识认为"重要任务就多让模型想几轮"。但**没有验证信号时,自我评判与自我生成共享同一套先验和同一份上下文,错误是相关的**——它评不出自己错在哪。这正是"你确定吗?"往往只换来一次谄媚式改口的机制原因。

**3. 逐个范式:机制、边界、2026 存活状态**(以下存活判断是**我的判断**,理由随附,非实测结论)

**(a) Plan-and-Execute**。机制:planner 一次性产出步骤列表 → executor 逐步执行(常用更便宜的模型,且只带当前步骤而非全部历史)→ 失败时 replanner 改计划。与 ReAct 的真实差别不是"想得更全",而是**规划所依据的信息更少**:计划在看到任何 observation 之前就定了。所以纯 plan-and-execute 对意外很脆,现实实现都必须带 replan 回边,于是退化成"粗粒度的 ReAct"。
存活状态:活得最好,但**理由换了**——今天它值钱不是因为提升推理质量,而是三个与模型聪明程度无关的工程属性:**审批闸门**(不可逆动作前给人看的必须是计划,不是执行到一半的结果)、**抗压缩的状态锚点**(写下来的计划/待办能在 compaction 后活下来,散在历史里的意图活不下来,见 [02 Q2](02-context-engineering.md))、**成本路由**(强模型规划、弱模型执行)。对应产物:plan mode、各家 agent 的 TODO/task list。

**(b) Reflection(自我批判)**。机制:同一轮内 generate → critique → revise。收益完全取决于 critic 相对 generator 有没有**信息优势**;同上下文、同模型、同提示的自我批判 ≈ 再采样一次,还额外引入"为批判而批判"的回归风险。
存活状态:**纯形式的自我反思价值最低**,但三种"制造了信息差"的变体今天广泛在用:critic 拿到**工具反馈**(测试输出、类型检查、schema 校验错误——这时它不是在反思,是在读事实);critic 在**干净上下文**里只看产物与验收标准(见 [02 Q6](02-context-engineering.md) 的对抗性复核);critic 换一套 **rubric**(安全/性能/需求覆盖)。所以正确说法不是"reflection 过时了",而是:**反思的价值来自信息差,不来自反思这个动作本身**。

**(c) Reflexion**(Shinn et al., 2023)。与 Reflection 的区别常被答错:Reflexion 是**跨试次**的——一次完整尝试失败后,拿到成功/失败信号,让模型把失败原因写成一段自然语言教训存进 episodic memory,然后**整个任务从头重来**并带上这段教训;作者把它描述为"用语言做梯度下降",反思文本就是那个梯度。前提很硬:任务可重来(幂等/沙箱)+ 有试次级信号。生产里很多任务两条都不满足——发出去的邮件不能重来。
存活状态:**机制活得很好,名字消失了**。它今天以两种形态存在:harness 的重试策略,以及记忆工程里的"负面知识"——[02 Q2](02-context-engineering.md) 压缩保留清单中那条"已排除的方案 + 原因",就是 Reflexion 的教训存储退化到单会话内的形态。能主动指出这个连接是加分项。

**(d) Tree-of-Thoughts**(Yao et al., 2023)。机制:把"一条推理链"换成"一棵状态树",每步用模型生成 k 个候选 thought,再用模型或程序给中间状态打分,按 BFS/DFS 展开并**回溯**。它真正的新东西是两件:显式的**状态评估器**,和**可以走回头路**。成本是分支 × 深度的乘法增长,外加每个节点一次评估调用——所有范式里最贵。
存活状态:**显式实现的必要性下降最多**。原因是"搜索发生在哪一层"变了:原生 reasoning 模型在 thinking 段内部就会提假设、自我否定、换路线,等于把小规模搜索与回溯搬进了单次调用,还不必付 k 倍的 prompt 重放成本。今天仍值得显式做树搜索的只剩一种情况:**评估器是真程序而不是模型**——每个节点跑一次测试/编译/形式验证/模拟器(代码合成、定理证明、路径规划)。反过来,如果打分还是同一个模型来做,你多半只是把 [02 Q1](02-context-engineering.md) 的注意力问题乘上了分支数。

**4. 与 Q4 的分工:把同一把尺子用到其他范式上**

Q4 讲 ReAct 被内化的三层(推理/行动/循环)。同样的尺子量其他范式,得到的不是整齐的"全被内化",而是**按层次分化**:

| 范式 | 已被内化的部分 | 仍需显式实现的部分 | 今天的落点 |
|---|---|---|---|
| Tree-of-Thoughts | 搜索与回溯(进了 thinking) | 接**程序化 evaluator** 的那种树搜索 | 少数强可验证域 |
| Reflection | "再检查一遍"的本能 | 制造**信息差**的复核 | reviewer subagent、测试闭环 |
| Reflexion | 单轮内的自我纠错 | 跨试次的**教训持久化** | memory / compaction 保留项 |
| Plan-and-Execute | "先想清楚再动手"的倾向 | 计划作为**可审批、可持久、可路由**的产物 | plan mode / TODO 列表 |

一句话收口:**被内化的是"认知动作",没被内化的是"工程属性"——凡是范式中要求外部信号、人的介入、跨会话持久化的那部分,模型再强也接管不了。**

**实例**(四种控制流的骨架对照,差异全在循环结构):

```python
# ReAct:线性,单一循环
while not done: obs = act(model.next_step(history)); history += obs

# Plan-and-Execute:规划与执行分离,失败才回到 planner
plan = planner(task)                        # 一次昂贵调用,此时还没有任何 observation
for step in plan:
    if not (r := executor(step)).ok:        # executor 可用更便宜的模型
        plan = replanner(task, plan, r); break   # ← 没有这条回边就是纯开环,必碎

# Reflection vs Reflexion:回边的位置不同(轮内 / 跨试次)
draft = gen(task);  draft = revise(draft, critique(draft))          # Reflection:同一轮内
for _ in range(N):                                                   # Reflexion:整任务重来
    if (t := run(task, memory)).success: break
    memory.append(reflect(t.trajectory, t.reward))   # ← 教训进 memory,这是关键

# Tree-of-Thoughts:分支 + 打分 + 回溯
frontier = [root]
while frontier:
    node = pick(frontier)
    kids = [s for s in model.propose(node, k=5) if evaluate(s) > τ]  # evaluate 是谁?决定成败
    frontier = backtrack_or_extend(frontier, node, kids)
```

**边界与误区**:
- 误区:"范式越多越好、可以叠 buff"。每加一层都乘上延迟、成本与新的失败面;先回答"我的 verifier 在哪",再决定加不加。
- 误区:把 Reflection 与 Reflexion 当同义词。轮内修订 vs 跨试次教训 + memory,是两个东西,面试常考。
- 误区:"plan-and-execute 比 ReAct 聪明"。它的计划是**信息更少**时做出的,它赢在工程属性而非推理质量。
- 边界:弱模型/本地小模型上,上面所有"已被内化"的判断都不成立——显式范式仍是主要提质手段(与 Q4 的兜底场景一致)。
- 诚实边界:内化程度的判断依据是机制推理(能力位置从提示迁移到权重与解码预算)加产品形态观察,**不是我做过的对照实验**;落到你自己的模型与任务,应当用评估集实测"加这层多提升多少、多花多少 token"。

**追问预判**:
- 追问:"self-consistency(多采样投票)算范式吗?k 怎么选?" → 答:算,而且是最朴素的一种。但它只在**答案空间离散可比**时成立(数值答案、分类标签),开放式生成上"多数"无定义;k 看边际收益拐点,通常几次后迅速衰减。关键判据:**如果你有 verifier,用 verifier 挑永远优于投票**——投票是没有 verifier 时的退而求其次。
- 追问:"你怎么判断一个任务该不该上显式 plan?" → 答:三个可操作信号——(1) 一句话说不清最终产物/diff;(2) 含不可逆动作,需要人在执行前看一眼;(3) 任务跨会话或会触发压缩,需要一个能落盘的锚点。三条都不满足时,plan 就是纯开销。

**关联**:[kb/00-fundamentals/planning-reasoning.md](../kb/00-fundamentals/planning-reasoning.md) · [kb/10-landscape/research-frontiers.md](../kb/10-landscape/research-frontiers.md) · [02 Q2](02-context-engineering.md)(教训持久化)· [02 Q6](02-context-engineering.md)(干净上下文复核)
