---
last_updated: 2026-08-19
type: interview
topic: 评估与可观测
questions: 5
---

# 面试题:06 评估与可观测

> 每题讲透五件套:本质 / 机制 / 实例 / 边界 / 检验(追问预判)。答案分 30 秒版 + 深入版。
> 本组的主线判断:**agent 的能力上限由模型决定,agent 的可交付性由评估闭环决定**。会搭 loop 的人很多,能证明 loop 没坏的人很少。

## Q1:为什么 agent 评估比单轮 LLM 评估难得多?轨迹级与结果级评估分别测什么?

**考察点**:能否说出难度的**结构性来源**(而不是"因为更复杂");是否理解两级评估的分工、各自盲区,以及为什么不能只用一级。

**30 秒版**:单轮评估是对一个**纯函数**做输入-输出比对;agent 评估要评的是一条**长度不定、路径不唯一、带副作用、且不可完全重放**的轨迹。四个结构性难点:①**路径非唯一**——同一目标有多条正确解,没法字符串比对;②**有状态**——评估要先建后毁一个世界,不是无状态调用;③**长程误差累积**——20 步每步 95% 正确,整体只剩约 36%,且失败点难定位;④**高方差**——同一 prompt 两次跑出不同轨迹,单次通过什么都证明不了。所以分两级:**结果级(outcome)**测"任务有没有真的完成"——靠可编程验证器(跑测试、断言数据库状态、比对最终答案);**轨迹级(trajectory)**测"过程对不对"——工具选择、参数正确性、步数与成本、是否走了危险捷径、是否违反流程约束。结果级做验收门禁,轨迹级做诊断和安全断言。

**深入版**:

先把难度差拆开,这是回答的骨架:

| 维度 | 单轮 LLM 评估 | Agent 评估 |
|---|---|---|
| 被测对象 | 一次生成 | 一条 (state, action, observation) 序列 |
| 正确性判据 | 输出匹配参考答案 | 世界的**最终状态**是否符合预期 |
| 环境 | 无 | 必须可建可毁、确定性、可回滚 |
| 失败定位 | 就在这一次输出里 | 需要找出 20 步里的**首个偏离点** |
| 方差来源 | 采样温度 | 采样 + 工具时序 + 环境状态 + 外部依赖 |

**结果级评估**的设计核心是一句话:**让环境判定成功,而不是让模型判定成功**。三种验证器形态,优先级从高到低:
1. **执行验证**:跑测试/编译/lint——最硬,能程序化就用这个(SWE-bench 用 FAIL_TO_PASS + PASS_TO_PASS 双组测试就是范式样板);
2. **状态断言**:任务结束后断言数据库行、文件树、外部 API 的最终状态——适合业务型 agent;
3. **答案比对**:精确/模糊匹配,实在不行才上 rubric + judge(见 Q2)。

**轨迹级评估**测的是结果级看不见的东西。可量化的维度:工具选择准确率、参数合法率与语义正确率、**步数冗余比**(实际步数 / oracle 轨迹步数)、成本($ 与 token)、**恢复能力**(工具报错后是否自行修复而非放弃或死循环)、**流程合规**(必须先 A 后 B 的不变式)、**安全违规**(是否碰了禁用工具/越权路径)。实现上分两类:**规则型**(在轨迹上写断言与不变式,便宜、确定、可进 CI)与 **judge 型**(对整条轨迹打分,贵但能覆盖开放判断)。

两级必须并存的原因,在于两种典型错配:
- **结果对但轨迹烂**:agent 直接改测试让它通过、绕过权限检查、或用 30 步和 10 倍成本蒙对——只测结果就是在**奖励作弊**(reward hacking),这是 agent 评估最贵的坑;
- **结果错但轨迹好**:外部依赖抖动、fixture 脏了——只看结果会把环境问题记到模型头上。

所以分工是:**结果级做 pass/fail 门禁,轨迹级做诊断切片 + 少量安全类硬断言**;不要拿轨迹相似度当主判据(理由见"边界")。最后是方差纪律:agent 评估必须重复运行并区分 **pass@k**(k 次至少一次成功——乐观,反映能力上限,适合有人复核或有验证器可挑选的场景)与 **pass^k**(k 次全部成功——悲观,反映可靠性,生产 SLA 该看这个)。只报单次成功率的评估报告,基本可以认为没做过评估。

**实例**(一个"退款处理 agent"的两级用例骨架):

```python
def verify_outcome(env):                                  # 结果级:环境说了算
    assert env.db.get_order("A-1001").status == "refunded"
    assert env.db.get_order("A-1001").refund_amount == Decimal("128.00")
    assert len(env.mail.sent_to("user@x.com")) == 1       # 不多发也不漏发
    assert env.db.get_order("A-1002").status == "paid"    # 没误伤旁边的订单(P2P 思路)

def verify_trajectory(traj):                              # 轨迹级:不变式 + 预算,不做精确匹配
    names = [s.tool for s in traj.tool_calls]
    assert names.index("verify_identity") < names.index("issue_refund")  # 顺序不变式
    assert "delete_order" not in names                                   # 安全硬断言
    assert len(traj.steps) <= 8 and traj.cost_usd <= 0.05                # 效率与成本预算
```

**边界与误区**:
- 误区:用最终回复文本的相似度(BLEU/embedding)评 agent。agent 的价值在**副作用**,不在措辞;文本相似度高而订单没退成,是纯粹的假阳性。
- 误区:轨迹级评估拿"与 golden trajectory 逐步一致"当满分——这会惩罚更短更优的新解法,把评估变成对旧实现的固化。轨迹级应该写**约束与不变式**,不写标准答案。
- 误区:把工具调用次数当质量指标(越少越好)。少调用也可能是过早终止;步数必须和成功率联合看。
- 边界:开放式任务(调研、写作、方案设计)没有可编程验证器,只能 rubric + judge + 人工抽检,此时必须老实报置信区间,别把 judge 分数说成成功率;并发/时序类 bug 也天然难复现,确定性 fixture(冻结时间、固定随机种子、mock 外部 API)是前提,否则测的是噪声。

**追问预判**:
- 追问:"20 步的轨迹失败了,你怎么定位根因?" → 答:切成 span 后找 **first divergence**(第一个偏离预期的**决策**)而不是最终报错点;用 counterfactual replay(从第 k 步注入正确观测再往下跑)区分"决策错"还是"工具/环境错";按失败分类法打标签(工具选错/参数错/观测误读/规划错/过早终止/死循环/上下文丢失),统计分布决定该改提示词、改工具描述还是改 harness。
- 追问:"pass@k 和 pass^k 怎么选?" → 答:能力研究和"有人兜底"的产品看 pass@k;无人值守的生产看 pass^k 或"平均成功率 + 方差";两者一起报才能说明"能不能做到"和"稳不稳定"是两件事。

**关联**:[kb/06-evals/agent-evals.md](../kb/06-evals/agent-evals.md) · [kb/00-fundamentals/agent-definitions.md](../kb/00-fundamentals/agent-definitions.md) · [kb/08-production/reliability.md](../kb/08-production/reliability.md)

## Q2:LLM-as-judge 的偏差来源有哪些?怎么校准?2026 年有什么新做法?

**考察点**:是否把 judge 当作**需要先被标定的测量仪器**,而不是免费的标注员;有没有量化校准的手感。

**30 秒版**:judge 本质是个**量具**,用之前必须先标定它。偏差分三类:①**格式/位置类**——成对比较里先出现的更容易赢(position bias)、更长更漂亮的更容易赢(verbosity bias);②**身份类**——偏爱自家或同族模型的输出(self-preference)、被"自信语气/权威引用"带偏;③**任务类**——rubric 歧义导致分数漂移、1-10 分制向中间聚集、长轨迹里对后段权重更高、judge 与被测模型同源导致**共享盲点**。校准四步:建**人工标注黄金集**(含 30-40% 边界难例)→ 量化 judge(与人类的一致性 κ、judge 自身重跑一致性)→ 工程消偏(双向换位、低基数标签、先给证据再判、拆成单维度 judge)→ 把黄金集纳入回归监控。2026 的新做法是**评估器可训练化**:LangSmith 于 2026-08-18 推出 **Tuned Evaluators**(首个针对 Perceived Error),用团队自己的标注微调 judge,把它从提示词工程变成**有训练集、有版本、有验收指标的模型资产**。

**深入版**:

**为什么 judge 天生会偏**:judge 和被测模型是同一套预训练目标训出来的,继承同一批先验——偏好流畅、结构化、篇幅充足、语气自信的文本。它手里**没有 ground truth**,做的是"这看起来像不像一个好答案"的似然判断。所以 judge 擅长的是"明显烂 vs 明显好"的粗分辨,不擅长"两个都还行,哪个更对"的精细判定——而后者恰恰是版本迭代最需要的分辨率。

偏差清单与对应机制(答题时逐条给"机制 → 消解手段"):

| 偏差 | 机制 | 消解手段 |
|---|---|---|
| position bias | 上下文顺序影响注意力分配 | 双向评测(A/B 与 B/A 各跑一次),不一致判平局 |
| verbosity/格式偏好 | 训练分布里长且结构化的文本更常被标为优质 | rubric 显式惩罚冗余;把"分数-长度相关性"当诊断指标看 |
| self-preference | 与自己生成分布更接近的文本似然更高 | judge 换异族模型;做 cross-judge 一致性检查 |
| 分数尺度漂移 | 细粒度分数缺乏锚点,跨批次不可比 | 用二元或 3 档标签;保留锚定样本做跨版本换算 |
| rubric 歧义 | 同一描述被不同样本触发不同解读 | 每档配 anchor examples(正例 + 反例) |
| 共享盲点 | judge 与被测同源,同样看不出的错误 | 关键维度必须有程序化验证器兜底,judge 只管主观维度 |
| 长轨迹注意力衰减 | 一次喂 50 轮,中段被稀释 | 分段评 + 汇总,而不是整条塞进去要一个总分 |

**校准的量化做法**(这是区分"听说过"和"做过"的地方):
- 黄金集 200-500 条,刻意超采边界样本(judge 在 happy path 上一致性很高,那不解决问题);
- 主指标:judge 与人类标注的一致性(分类用 Cohen's κ,序数用 Spearman);
- 辅指标:judge **自一致性**(同输入重跑 3 次的分歧率)——自己都不稳的 judge,和人一致也是巧合;
- 判据要现实:**人际一致性是天花板**。如果两个标注员之间 κ 只有 0.6,就别指望 judge 到 0.9;此时该做的是修 rubric,而不是换更大的 judge 模型。

**2026 的三条演化线**(答"新做法"时给方向,不要背工具名):
1. **可训练化**:评估器从"写提示词"走向"用团队标注微调"(as-of 2026-08-18,LangSmith Tuned Evaluators)。价值是把"我们团队认为什么算错"编码进权重;代价是过拟合历史标注、需要持续补标、并且它优化的是"符合我们的标准"而非"客观正确"。
2. **验证器优先**:最被低估的一条——**能程序化就绝不用 judge**。judge 应该只出现在"无法写断言"的维度上。很多团队 judge 用得多,只是因为没花时间把成功定义写成代码。
3. **judge 可审计化**:强制 judge 输出结构化判据(引用轨迹中的 span id / 原文片段)再给结论,而不是只吐一个分数。好处是双份的:人能复核,且"先证据后结论"本身就降低了随口打分的比例。

**成本视角**:大规模评估里 judge 的调用量可能超过被测 agent 本身。分层过滤是标准解:程序化规则先筛掉能判定的 → judge 只跑剩余争议样本 → 人工只看 judge 低置信/双向不一致的样本。

**实例**(judge 从 v0 到 v2 的演进,以及校准脚本骨架):

```
v0(反面教材):"给这个回答的质量打 1-10 分。"        # 无锚点、无维度、无证据
v1:拆维度 + 二元判定 + 先证据后结论
    "判断该回答是否包含未被检索结果支持的陈述。
     步骤:1) 逐句列出事实性陈述;2) 标注每句对应的证据片段 id,无证据则标 NONE;
     3) 输出 {unsupported: [...], verdict: 'supported'|'unsupported'}"
v2:v1 + 用 400 条人工标注微调评估器,κ 从 0.58 → 0.79(示意值,须自测)
```

```python
# 校准脚本骨架:先测量具,再用量具
rows = [(g.label, judge(g.input)) for g in golden_set]           # 与人对齐
kappa = cohen_kappa([h for h, _ in rows], [j for _, j in rows])
self_consistency = mean(len(set(judge(g.input) for _ in range(3))) == 1
                        for g in golden_set)                      # 自一致性
assert kappa >= HUMAN_HUMAN_KAPPA * 0.9, "judge 未达标,不得用于门禁"
```

**边界与误区**:
- 误区:把 judge 分数当绝对真值汇报("我们 agent 的质量是 4.3/5")。**没有一致性数据的 judge 分数不是测量,是意见**。
- 误区:用被测的同一个模型、同一个提示词做 judge——自证清白,尤其在"回答是否完整/是否有幻觉"这类共享盲点维度上。
- 误区:换了 judge 模型后直接对比新旧分数。换 judge = 换量具,必须在锚定集上重新标定并说明换算关系,否则"质量提升"可能只是量具变松了。
- 误区:rubric 越细越好。维度越多、档位越细,judge 自一致性掉得越快;宁可多个二元 judge,不要一个十档 judge。
- 边界:安全与合规判定不能只靠 judge。高风险维度要规则 + 人工双保险,judge 至多做召回前置筛选。

**追问预判**:
- 追问:"你怎么证明你的 judge 可信?" → 答:报三个数——与人类标注的 κ、judge 自一致性(重跑分歧率)、在已知失败样本上的召回率;并说明黄金集的抽样方式与难例配比。没有这三个数就直说"还没标定,所以只用于观察不用于门禁"。
- 追问:"tuned evaluator 会不会过拟合?" → 答:一定会,它拟合的是团队历史标注偏好。对策:留 holdout、定期补新标注、监控线上分布漂移;并明确它的定位是"把团队标准自动化",不是"逼近客观真理"——标准本身错了它会把错误放大。

**关联**:[kb/06-evals/agent-evals.md](../kb/06-evals/agent-evals.md) · [kb/10-landscape/research-frontiers.md](../kb/10-landscape/research-frontiers.md) · [kb/01-context-engineering/structured-outputs.md](../kb/01-context-engineering/structured-outputs.md)

## Q3:SWE-bench 这类 agent 基准怎么构成?饱和与污染问题怎么看?现在该看什么基准?

**考察点**:能否解剖一个基准的构造机制并据此判断它的效力边界;有没有"基准会过期"的时间感。

**30 秒版**:SWE-bench Verified 由 **500 个人工验证过的真实 GitHub issue** 构成:给仓库某个 commit 的快照 + issue 文本,agent 产出 patch,用 **FAIL_TO_PASS**(必须由失败转通过)+ **PASS_TO_PASS**(不许改坏别的)两组测试判定——是"环境即验证器"的范式样板。但它已经饱和:as-of 2026-08-17,头部五家挤在约 4 分内(GPT-5.6 Sol 96.2%、Opus 5 96.0%(vals.ai 复测 97.0%)、Mythos 5 95.5%、Fable 5 95.0%、Kimi K3 93.4%),**区分度已经小于复测口径之间的差异**;叠加污染(题目来自公开仓库,极可能进过训练语料)与标签噪声(测试过窄/过宽把正确解判错),剩下那几分更多反映 scaffold 和数据集缺陷,而非模型能力。现在该看:**SWE-bench Pro**(长程 + 抗污染设计)、**Terminal-Bench**(真实终端环境)、**HAL / Holistic Agent Leaderboard**(准确率-成本 Pareto 而非一维排名)、**tau2-bench**(工具-用户交互与策略合规)。但更重要的一句是:**公开基准只用于选型粗筛,发版决策必须靠你自己的私有 eval 集**。

**深入版**:

**基准的四个构件**(拿到任何新基准都用这套解剖,比记分数有用得多):
1. **任务源**——题目哪来的,有没有可能已经进过训练语料;
2. **环境**——能否可复现地重建(容器镜像、依赖锁定、外部依赖 mock);
3. **验证器**——怎么判成功,是执行验证还是模型判定;
4. **scaffold 契约**——agent 能用什么工具、最多多少步、什么模型档位。**第 4 条不固定,分数就不可比**——这是跨表比较分数时最常见的错误。

SWE-bench Verified 对应下来:任务源 = 真实 GitHub PR/issue;环境 = 仓库 commit 快照 + Docker;验证器 = F2P/P2P 双组测试;"Verified" 的含义就是雇人筛掉描述不全、测试不合理的题,剩下 500 道。它的历史贡献是把"编码能力"变成了**可自动判定**的东西。

**饱和:机制与识别信号**。饱和的定义不是"分数高",而是**测量分辨率低于噪声**。三个识别信号:
- 头部差距 < 同一模型重跑的方差;
- 不同复测口径互相打架(同一模型官方 96.0 / 第三方 97.0,排序随口径翻转);
- 排名随 scaffold 变化而变化——说明测的是 harness 不是模型。
一旦满足,继续报这个数就是营销而非评估。

**污染:三种形态与检测手段**:
- 形态一,**解法泄露**(题目与参考 patch 在训练语料里);形态二,**测试泄露**(判定用例本身被学过);形态三,**社区过拟合**——整个行业围绕同一个基准调 scaffold 和提示词,没有任何单点作弊,但集体把测试集当验证集用了。第三种最难察觉,也最致命。
- 检测手段:①**逐字复现测试**——不给上下文让模型直接补全参考解,看重合度;②**公开/私有孪生子集对照**——同分布两份,一份公开一份从未发布,看分数落差;③**时间切分**——用模型知识截止之后的 commit 建对照子集,看分数是否下滑。
- 结构性对策:选 copyleft/GPL 许可的仓库(法律屏障降低被商业语料收录的概率)、保留私有商业代码子集、以及持续滚动更新的 live benchmark。

**as-of 2026-08 的替代盘**(答题时讲"它补了什么维度",不要报流水账):
- **SWE-bench Pro**:约 1,865 题、41 个活跃仓库,覆盖 Python/Go/TS/JS;分三个子集——公开(GPL 仓库)、held-out、以及来自真实创业公司代码库的**商业私有集**;任务平均触及约 4.1 个文件,是"数小时到数天"量级的长程任务。它的抗污染设计不只是宣传:公开子集与私有子集之间存在可观的分数落差,这个落差本身就是污染的度量(数据来自 Scale 的公开材料与论文,as-of 2026-08,二手核对)。
- **Terminal-Bench 2.0**:Docker 隔离的真实命令行任务,覆盖软件工程、安全、科学计算、数据科学、调试等 16 类共 89 题、分难度档。它测的是"会不会用一台机器",而不是"会不会打 patch"——补上了 SWE-bench 完全不覆盖的环境操作维度。
- **HAL(Holistic Agent Leaderboard)**:用统一 harness 跨多基准跑,输出的是**准确率-成本 Pareto 前沿**而非一维榜单——直接暴露"贵很多但只好一点"的模型。其大规模 rollout 还给出反直觉结论(更高的 reasoning effort 在多数运行中反而降低准确率),这类结论只有统一 harness 才做得出来。**选型该看的就是这种形态的榜。**
- **tau2-bench**:工具-用户双向交互 + 策略合规,补上"跟人打交道、守不守规矩"这一整块 SWE-bench 不测的维度。

一个值得知道的行业信号(as-of 2026-08-19,**二手来源、置信度中,官方原文我未取到**):OpenAI 公开表态不再报告 SWE-bench Verified,理由集中在测试用例缺陷(过窄的测试强制实现细节、过宽的测试检查范围外功能)与训练污染,并建议改报 SWE-bench Pro;同期也有关于该推荐被回撤的报道。这条不要当事实背,但它的**教训是硬的**:基准的选择带有厂商利益,任何"新标准"公告都要先看谁在推、谁受益。

**收口的方法论**:公开基准回答"这个模型大概在什么档",私有 eval 回答"它在**我的**任务上行不行"。前者用于选型,后者用于发版。把公开基准分数当验收标准的团队,是在用别人的数据分布替代自己的。

**实例**(自建"内部 SWE-bench"的构造清单,复用 SWE-bench 的机制但天然抗污染):

```yaml
# 选题规则:从自己仓库的历史 PR 里挑
filters:
  - 有关联 issue 描述(输入才完整)
  - 该 PR 新增或修改了测试(F2P 才有来源)
  - diff 跨 >= 2 个文件(排除单行改动)
  - merged_at > 模型知识截止日期        # 天然抗污染的关键一条
# 单条 case
case:
  id: internal-swe-0042
  repo_snapshot: <该 PR 的父提交 sha>
  env_image: ci-base:2026-08              # 依赖锁定
  input: <issue 正文>
  verifier:
    fail_to_pass: [tests/test_billing.py::test_proration]
    pass_to_pass: [<既有测试全集>]
  budget: {max_steps: 40, max_usd: 1.50}
  labels: {type: bugfix, difficulty: medium, source: prod_incident}
```
50-100 条这样的题就足以做发版门禁,而且分数只对你自己有意义——这恰恰是它的价值。

**边界与误区**:
- 误区:跨来源比较分数而不看 scaffold。同模型换 harness 的分差可达数分(本仓库 landscape 基线里 Coding Agent Index 的同代对比已实证),跨表比较基本无效。
- 误区:把 SWE-bench 分数当"编程能力"。它只测"给定 issue 与测试的补丁生成",不测需求澄清、架构设计、代码评审、可维护性,更不测成本与安全。
- 误区:自建 eval 专挑最难的题。难题集只测上限;发版门禁需要覆盖**典型分布** + 已知回归,难题另建一个"能力探针集"。
- 边界:基准有保质期。任何写进文档或简历的分数都必须带 **as-of 日期 + 口径来源**,否则半年后就是错误信息。

**追问预判**:
- 追问:"怎么低成本判断自己的 eval 集被污染了?" → 答:①时间切分——用知识截止后的数据做对照子集,看分数落差;②不给上下文让模型直接复现参考解,看逐字重合;③公开/私有孪生子集对照。三者中任一出现显著落差就按污染处理。
- 追问:"基准分很高但线上不好用,怎么解释?" → 答:分布不匹配(语言/仓库规模/任务形态)、scaffold 不同、验证口径不同(基准把测试白送了,线上没人给)、以及基准根本不测的维度(多轮交互、成本、安全、可维护性)。然后把话题拉回自己的私有 eval 与线上指标的对应关系上。

**关联**:[kb/06-evals/benchmarks.md](../kb/06-evals/benchmarks.md) · [kb/10-landscape/model-capabilities.md](../kb/10-landscape/model-capabilities.md) · [deep-dives/2026/2026-08-19-landscape-baseline.md](../deep-dives/2026/2026-08-19-landscape-baseline.md) · [kb/09-coding-agents/autonomous-swe.md](../kb/09-coding-agents/autonomous-swe.md)

## Q4:生产环境 agent 的可观测性怎么设计?trace/span 模型与关键指标有哪些?

**考察点**:能否把 agent 的执行结构映射到成熟的分布式追踪模型;能否区分"能看见"和"能改进"。

**30 秒版**:把一次 agent 执行当成一棵 **trace**:根 span = 一次用户回合,子 span 按 agent loop 的结构分层——`invoke_agent` → `llm_call` / `execute_tool` / 嵌套的 `invoke_agent`(子代理) / `guardrail`。每个 span 记三类属性:**输入输出**(prompt、工具参数与结果,带脱敏与采样)、**成本**(input/output/thinking/cache token、$、延迟)、**决策元数据**(模型 id、stop_reason、工具名、重试次数、**版本三元组**)。指标分四层:**质量**(任务成功率、人工接管率)、**成本**(每任务 $ 与 token 分解、cache 命中率)、**效率**(端到端 P50/P95、步数分布)、**可靠性**(分工具的错误率、超时率、达轮次上限比例、护栏触发率、compaction 触发率)。落地上向 **OpenTelemetry GenAI 语义约定**对齐以避免平台锁定——但要知道 as-of 2026-08 这些 `gen_ai.*` 属性大多仍标 Development,属性名可能变(二手来源,置信度中)。

**深入版**:

**为什么必须是 trace 而不是 log**:agent 的失败是**路径失败**——第 7 步工具返回了一段脏数据,导致第 9 步做出错误决策,最终在第 12 步报错。孤立日志看不出这条因果链;而分布式追踪的 span 树天然同构于 agent 的调用树(父子关系 + 时序 + 属性)。这是"直接复用二十年可观测性积累"而不是另起炉灶的原因。

**span 模型分层**:

```
trace: user_turn            {session_id, user_id, agent_version, prompt_version}
└── span: invoke_agent      {agent=main, model=..., max_steps, budget_usd}
    ├── span: llm_call #1   {tokens_in/out/thinking, cache_read, stop_reason=tool_use, ttft, latency}
    ├── span: execute_tool  {name=Bash, args_hash, exit_code, bytes_out, truncated=true, error=null}
    ├── span: guardrail     {name=permission_check, decision=deny, rule=write_outside_workspace}
    ├── span: llm_call #2
    ├── span: invoke_agent  {agent=explorer}      # 子代理:独立上下文,同一条 trace
    │   └── ...
    └── span: compaction    {tokens_before, tokens_after, strategy}
```

四个关键设计选择:
1. **粒度**:"一次模型请求 = 一个 span"是底线。但 compaction、记忆读写、检索、护栏判定**都必须有自己的 span**——否则"上下文里到底发生了什么"永远不可见,而长任务的失败大半就藏在这里。
2. **关联 id 贯通**:session_id → trace_id → span_id → tool_use_id 四级贯通,保证 tool_result 能回指发起它的 tool_use;跨进程的子代理要传播 trace context,否则子代理的失败在主 trace 上是个黑洞。
3. **版本维度必须是一等属性**:`agent_version` / `prompt_version` / `model_id` / `tool_schema_version`。缺了这些,"上周还好好的,这周变差了"就永远归因不到具体变更。**这是实践中最常被漏掉的一条**——因为它在出事之前看起来毫无用处。
4. **payload 治理**:prompt 与工具输出是 PII 高危区,也是存储大头。分级策略:元数据 100% 记录、全文按采样率记录(如 5%)、**失败与护栏触发样本 100% 记录且不采样**。

**指标体系(四层)**,以及每层的定义陷阱:
- **质量**:任务成功率(必须绑定 Q1 里那种可判定的成功定义,而不是"用户没投诉")、pass^k、人工接管/回退率、用户显式负反馈率、重试后成功率。
- **成本**:每任务 $、token 分解(input / output / thinking / cache_read)、cache 命中率、子代理成本占比。
- **效率**:端到端延迟 **P50/P95/P99**、每任务步数分布、单步等待时间、并行工具调用占比。
- **可靠性**:分工具的错误率、超时率、达到轮次上限的比例、循环检测触发率、护栏拦截率、上下文溢出/compaction 触发率。

贯穿四层的一条纪律:**均值会撒谎**。agent 的步数与成本是重尾分布,少量长尾任务吃掉大部分预算;只看均值的成本报表,和只看单次成功率的评估报告,是同一种错误。

**从"能看"到"能改"**:trace 不是终点,而是**评估数据集的主要来源**。生产 trace → 分层采样(失败/低分/接管样本全量进候选池)→ 人工标注 → 沉淀为 eval case → 进回归集。这条回流线是可观测性投资回报最高的部分,也是 Q5 的入口。埋点做了却没有回流线的团队,买的是仪表盘,不是改进能力。

**标准与选型**:OTel GenAI 语义约定把 agent / 工具 / 模型建模为统一的 span 词汇(`gen_ai.operation.name` 覆盖 create_agent、invoke_agent、execute_tool、retrieval 等),MCP 相关约定也并入同一体系,意味着工具调用与发起它的 agent 共享同一条 trace 词汇(as-of 2026-08,二手来源,且 `gen_ai.*` 大多仍是 Development 稳定性)。工程建议:**自己的埋点层与厂商 SDK 解耦**——业务代码只发 OTel 语义的 span,后端平台可换。

**实例**(最小埋点骨架,重点是嵌套与成本归集):

```python
with tracer.start_as_current_span("invoke_agent") as agent_span:
    agent_span.set_attributes({
        "gen_ai.agent.name": "main", "gen_ai.request.model": model_id,
        "app.agent_version": AGENT_VER, "app.prompt_version": PROMPT_VER,  # 归因用
    })
    while not done and steps < MAX_STEPS:
        with tracer.start_as_current_span("llm_call") as s:
            resp = client.messages.create(...)
            s.set_attributes({
                "gen_ai.usage.input_tokens": resp.usage.input_tokens,
                "gen_ai.usage.output_tokens": resp.usage.output_tokens,
                "gen_ai.usage.cache_read_input_tokens": resp.usage.cache_read_input_tokens,
                "gen_ai.response.finish_reasons": [resp.stop_reason],
            })
        for call in tool_calls(resp):
            with tracer.start_as_current_span("execute_tool") as t:
                t.set_attributes({"gen_ai.tool.name": call.name,
                                  "app.tool_use_id": call.id})   # 与 tool_result 对齐
                try:    result = run(call)
                except Exception as e:
                    t.record_exception(e); t.set_status(ERROR)    # 错误也是数据
    agent_span.set_attribute("app.cost_usd", accumulated_cost)    # 子 span 成本上卷
```

**边界与误区**:
- 误区:只记 LLM 调用,不记工具执行。生产 agent 的失败大头在工具与环境(超时、权限、脏数据、返回过长被截断),漏掉这层等于没埋点。
- 误区:把 trace 当日志人肉翻。规模上来后必须有聚合视图:按失败类型 × 工具 × 版本切片,看的是分布不是个例。
- 误区:全量存 prompt 全文。合规风险 + 存储成本双杀;采样 + 脱敏 + 失败全存是标准配方。
- 误区:用平均延迟做 SLO。agent 延迟长尾极重,SLO 要挂 P95/P99,并对"步数超限"单独设指标。
- 边界:采样必然丢罕见失败。**安全类事件(护栏触发、权限拒绝、注入检测命中)必须 100% 记录,不进采样池**。
- 边界:trace 只解释"发生了什么",不解释"为什么这么决策"。要理解决策,需要把 thinking 内容与当时的完整上下文一起留存(注意这又是 PII 与成本的权衡)。

**追问预判**:
- 追问:"线上出现一次坏结果,你的定位流程?" → 答:按 trace_id 拉全轨迹 → 找 first divergence(第一个偏离预期的决策)而非最终报错点 → 检查该 span 的输入上下文(工具返回是否被截断、compaction 是否削掉了关键信息)→ 同版本重放确认可复现 → 归入失败分类法 → **沉淀成一条 eval case**。最后一步不做,同一个 bug 会再来一次。
- 追问:"怎么衡量可观测性本身够不够用?" → 答:两个元指标——MTTD/MTTR(从发生到发现、到定位至具体 span 的时间),以及**可解释率**(线上失败中能仅凭 trace 说清根因的比例)。可解释率低,说明埋点有洞,而不是模型不行。

**关联**:[kb/06-evals/observability-tracing.md](../kb/06-evals/observability-tracing.md) · [kb/08-production/reliability.md](../kb/08-production/reliability.md) · [kb/08-production/cost-latency.md](../kb/08-production/cost-latency.md) · [kb/07-safety-security/permissions-hitl.md](../kb/07-safety-security/permissions-hitl.md)

## Q5:给你一个刚上线的 agent,从零搭评估闭环你会怎么做?

**考察点**:能不能把评估当作**有交付顺序、有取舍、有防回归机制的工程系统**来讲,而不是罗列平台名。这题的分差主要来自"先做哪一件"和"怎么防回归"。

**30 秒版**:按"先能测,再测准"的顺序四步走。**第 0 步(第一周):把成功定义成断言**——和业务把"任务完成"翻译成可编程判定的证据,写不出断言的任务类型先标记为人工抽检,不进门禁。**第 1 步(第一周并行):埋点 + 冷启动数据集**——先上 trace(哪怕只是结构化 JSON 落盘),同时手写 30-50 条覆盖核心路径的种子用例,建立第一条基线。**第 2 步(2-4 周):分层评估 + 接 CI**——快集每 PR 跑(纯程序化验证器,< 10 分钟),慢集每日/发版跑(全量 + k 次重复 + judge),线上采样标注持续回流扩充。**第 3 步(持续):防回归**——锚定集不可变、阈值由实测方差推导、版本三元组归因、每次线上事故强制转成一条 eval case。一句话总纲:**评估数据集是产品资产,它的增长速度决定你的迭代速度**。

**深入版**:

### 阶段 0:把"成功"变成断言(最难,也最省后续力气)

对每个任务类型逼问业务方一句话:"这次算成功的**可观测证据**是什么?"——追问到能写成代码为止。输出物是一张表:任务分类法(3-7 类)× 每类的验证器类型(执行验证 / 状态断言 / 精确匹配 / rubric+judge)。写不出验证器的类型先降级为"人工抽检 + rubric",并明确标注**暂不进 CI 门禁**。

这一步做不完就往下走,是最常见的失败模式:后面所有数字都会失去意义,因为没人能说清它们在测什么。

### 阶段 1:数据集怎么来(四个来源,按获取成本排序)

1. **手写种子集**(第一天就有):照产品定义的核心路径写 30-50 条,覆盖 happy path + 已知边界。作用是**建立基线**,不追求覆盖率。
2. **生产 trace 采样**(最重要):分层采样——失败 / 低分 / 人工接管样本 100% 进候选池,正常样本按比例抽;标注后入库。**线上分布是唯一的真分布**,手写用例永远猜不全用户怎么把事情搞砸。
3. **事故回流**(价值密度最高):每个线上 bug 必须落一条 case,附最小复现 fixture——就是"bug 先写测试"的老规矩,只是搬到了 agent 上。
4. **合成扩增**(补长尾):基于已有 case 做措辞/参数/顺序扰动,生成边界与对抗样本;**必须人工过一遍**,合成数据是补充不是主体。

单条 case 的结构(注意第 3 项是 agent eval 与 LLM eval 的最大区别):

```yaml
- id: refund-013
  input: "帮我把订单 A-1001 退款,用户说收到的是破损商品"
  fixture:                      # 环境 fixture:必须能一键建/毁一个确定性世界
    db_seed: seeds/orders_v3.sql
    mocks: {payment_api: fixtures/payment_ok.json, clock: "2026-08-19T10:00:00Z"}
  verifier: verifiers/refund.py::verify_outcome
  invariants: verifiers/refund.py::verify_trajectory
  labels: {type: refund, difficulty: medium, source: prod_incident_4471}
  budget: {max_steps: 8, max_usd: 0.05}
```

规模节奏:**50 条能跑起来,150-300 条能做门禁,500+ 才谈得上切片统计**。

### 阶段 2:指标怎么定(与门禁挂钩)

- **主指标(做 gate)**:按任务类型的成功率,由结果级验证器判定,报 **pass^3 或 3 次平均 + 方差**——单次运行不算数。
- **副指标(看趋势不阻塞)**:每任务成本、步数 P95、端到端延迟 P95、分工具错误率。
- **安全指标(硬 gate,零容忍)**:禁用动作触发数、越权访问数、注入用例通过数——这类阈值是 0,不设波动带。
- **阈值怎么定**:先用当前版本跑 k 次建立基线,阈值 = 基线 − 由实测方差推出的波动带。**先测方差再定阈值**是最常被跳过的一步;拍脑袋定 90% 的团队,最后都会因为噪声报警而关掉门禁。

### 阶段 3:怎么接 CI(分层跑,别把 CI 跑成钱包)

| 层 | 触发时机 | 内容 | 预算目标 | 作用 |
|---|---|---|---|---|
| 快集 | 每个 PR | 20-50 条 + 纯程序化验证器 + 单次运行 | < 10 分钟 / < $1 | 抓"明显坏了" |
| 慢集 | 合并主干 / 每日 / 发版前 | 全量集 + k 次重复 + judge 维度 | 小时级 | 出报告,超阈值阻塞发版 |
| 冒烟集 | 部署后 | 5-10 条只读、无副作用的真实环境用例 | 分钟级 | 抓配置与环境问题 |

工程细节三条:①**eval 与提示词/工具 schema 同仓同 PR**——改提示词就必须跑 eval,这是把评估变成习惯的唯一机制;②结果写成结构化 artifact 入库,带 `commit_sha + model_id + dataset_version`;③CI 里的外部依赖一律 mock,任何真实外部调用都会把门禁变成 flaky 报警器。

### 阶段 4:怎么防回归(这一段最能拉开差距)

- **版本三元组归因**:每次结果记 (代码 sha, prompt/skill 版本, 模型 id + 参数)。**模型是一个会自己变化的依赖**——供应商更新或你换档位都会移动基线,所以要有"只换模型、不动代码"的定期基线重跑。
- **不可变锚定集**:一小组永不修改的 case 作为跨版本可比的标尺,其余数据集自由增长。**数据集变了,分数就不可比**——这条几乎所有团队都会踩一次。
- **回归即 case**:线上事故 → 复现 → 进回归集 → 永久保留。回归集只增不减。
- **judge 也要防回归**:judge 的提示词或模型变更等同于更换量具,必须在锚定集上重新标定并记录换算关系(见 Q2)。
- **线上兜底**:离线过了不等于线上好。小流量金丝雀对照(成功率、接管率、成本),配自动回滚阈值,才是最终验收。

**实例**(目录结构与 CI 骨架,展示"评估是代码"的落法):

```
evals/
  datasets/     anchor/          # 不可变锚定集,跨版本比较用
                regression/      # 事故回流,只增不减
                seed/            # 手写核心路径
                sampled/         # 生产 trace 标注回流
  verifiers/    refund.py  code_patch.py  judges/hallucination.md
  fixtures/     seeds/  mocks/
  runner.py     # 建环境 → 跑 agent → 收 trace → 判定 → 写 JSON 报告
  thresholds.yaml
```

```yaml
# .github/workflows/evals.yml(骨架)
on: [pull_request]
jobs:
  fast-evals:
    steps:
      - run: python -m evals.runner --suite seed+anchor --repeat 1 --programmatic-only
      - run: python -m evals.gate  --report out/report.json --thresholds evals/thresholds.yaml
      # gate 规则:安全指标必须为 0;成功率不得低于 baseline - 波动带;成本涨幅 > 30% 告警
```

**边界与误区**:
- 误区:一上来追求大而全的数据集。**50 条能跑通的闭环 > 500 条跑不起来的计划**;闭环先通,再扩样本。
- 误区:全用 LLM judge。贵、慢、不稳;能程序化的一律程序化,judge 只留给主观维度(见 Q2)。
- 误区:eval 只在发版前跑。反馈延迟越长,归因越难;PR 级快集才是主力,发版集只是兜底。
- 误区:把线上指标当 eval。线上指标滞后且混杂(流量、用户、季节都在变),做不了因果归因;离线 eval 与线上指标是互补的两条腿。
- 边界:低频高风险任务(月结、清算、事故处置)样本天然稀缺——靠合成 + **影子运行**(agent 跑但不生效,与人工结果对照)积累,别指望等到自然样本。
- 边界:评估本身也有成本上限。当 eval 账单接近生产账单时,该做的是分层采样与缓存,而不是砍掉重复运行——**砍掉重复运行等于砍掉方差信息,是最坏的省钱方式**。

**追问预判**:
- 追问:"只有两周,先做哪一件?" → 答:**成功定义 + 埋点**。没有可判定的成功定义,后面所有数字都无意义;没有 trace,数据集就永远缺真实分布来源。judge、大数据集、漂亮的仪表盘都可以后补,这两件不能。
- 追问:"怎么说服团队投入?" → 答:算两笔账。一是回归成本——一次线上事故的排查 + 回滚 + 信任损失,对比一条 eval case 的边际成本;二是迭代速度——**"你敢不敢随手改提示词"直接等价于"你有没有门禁"**,没有 gate 的团队最终会不敢动系统,而不敢动就等于停止改进。

**关联**:[kb/06-evals/agent-evals.md](../kb/06-evals/agent-evals.md) · [kb/06-evals/observability-tracing.md](../kb/06-evals/observability-tracing.md) · [kb/08-production/deployment-patterns.md](../kb/08-production/deployment-patterns.md) · [kb/08-production/reliability.md](../kb/08-production/reliability.md) · [interview/01-agent-fundamentals.md](01-agent-fundamentals.md)(Q5 上下文管理与本组回归设计同源)
