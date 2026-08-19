---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# 基准测试

SWE-bench(Verified/Pro)、GAIA、tau-bench、terminal-bench、OSWorld 等 agent 基准的构成、饱和度与污染问题。

## 原理(稳定)

基准的生命周期几乎必然走完三步:**有区分度 → 被优化 → 失效**。失效有两条独立路径,判断一个基准还能不能用要分别检查:

1. **饱和(saturation)**:头部模型分数挤在几分之内,差异落进噪声区,排序不再有意义;
2. **污染(contamination)**:基准的题目与解答进入了训练数据,分数测量的是"见过多少"而非"能力多强";
3. 此外还有第三条常被忽略的:**测试用例本身有缺陷**(过窄——强制某种实现细节;过宽——测了题面没要求的功能),这会让"失败"不代表模型不行。

工程含义:**任何用基准分数做的决策(选型、验收、对外宣称),都必须同时说明基准版本、测量时间与污染状态**。分数是有保质期的事实。

## 现状(as-of 2026-08-19)

### ⚠️ SWE-bench Verified 已被官方判定失效,不应再用于前沿模型比较

**OpenAI 公开宣布不再报告 SWE-bench Verified 成绩**,并推荐行业改报 **SWE-bench Pro**。理由有三层:

- **测试缺陷**:审计 o3 持续失败的 138 个问题,**59.4% 含有缺陷测试用例**(过窄或过宽)——即近六成"失败"不是模型的问题;
- **训练污染**:2026-02-23 的分析发现主要前沿模型(含 GPT-5.2、Claude Opus 4.5、Gemini 3 Flash)均有在基准解答上训练过的证据;
- **官方结论**(引述要点):Verified 上的进步已不再反映真实软件开发能力的提升,而越来越多地反映模型在训练时接触该基准的程度。
- **落差数据**:Verified 上约 80% 的模型,在抗污染设计的 **SWE-bench Pro** 上掉到约 **23%**。

> 来源与置信度:官方页面 [openai.com/index/why-we-no-longer-evaluate-swe-bench-verified](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)(2026-08-19 尝试抓取返回 **HTTP 403**,内容依据为搜索引擎返回的该官方页面摘要 + [OpenAI Devs 官方 X 帖](https://x.com/OpenAIDevs/status/2026002219909427270) + 多家独立转载互相印证)。**置信度:高**(官方立场明确),但**原文未直接读取**,引用具体数字前建议再核。另有二手报道称该推荐一度被回撤,未获证实。

### 本仓库同日早些时候的判断需修正

今日全景基线深研与 kb/10-landscape 曾据第三方榜单记录"Verified 头部拥挤、接近饱和"。**方向正确但归因不足**:Verified 的问题不止是饱和,更是**污染 + 测试缺陷导致的测量失效**;因此那批 96%/95% 级别的分数**不应作为能力排序依据**,包括本仓库自己记录的那份表(已在 [model-capabilities.md](../10-landscape/model-capabilities.md) 加警示)。

### 该看什么(as-of 2026-08-19)

| 基准 | 定位 | 状态 |
|---|---|---|
| **SWE-bench Pro** | 抗污染设计的接任者,含公开/私有分割 | OpenAI 推荐改报;分数区间远低于 Verified(~23% vs ~80%) |
| **Terminal-Bench** | 终端环境端到端任务 | 2.0 版本约 16 类 89 题(二手来源,待核) |
| **HAL**(Holistic Agent Leaderboard) | accuracy–cost 帕累托前沿,不只看分数 | 关注成本维度的评测方向 |
| SWE-bench Verified | 500 个人工验证的 GitHub issue | ⚠️ 已失效,仅作历史参照 |
| GAIA / tau-bench / OSWorld | 通用助手 / 工具对话 / 桌面操作 | 待专题核实 |

## 时间线

- 2026-08-19 本仓库核实:OpenAI 官方宣布停报 SWE-bench Verified,推荐 SWE-bench Pro
- 2026-02-23 OpenAI 分析发现主要前沿模型存在基准解答训练污染证据(据官方页面摘要)

## 开放问题

- 拿到 OpenAI 官方原文全文(当前 403),核实 59.4%、23% 等具体数字与"回撤"传闻 → **已入队列**
- SWE-bench Pro 的构成、私有分割机制与当前榜单排序?
- 抗污染基准的通用设计手法(私有集、动态生成、时间切分)有哪些成熟范式?
