# 深研:2026-08 Agent 开发全景基线扫描

> 日期:2026-08-19 | 来源:队列 P1 | 网络操作:19 次(含 P1 简报共用)| 实验:无需(生态盘点类主题,无可本地验证的行为)

## TL;DR(写给三个月后的自己)

1. 模型层 2026 上半年完成换代:Anthropic Claude 5 家族(Sonnet 5 → Opus 5 → Fable/Mythos 5)、OpenAI GPT-5.6 三档(Sol/Terra/Luna)、Google Gemini 3 + 企业 agent 平台。
2. SWE-bench Verified 已饱和(头部 5 家挤在 4 分内,Opus 5 96%),排名主战场移到 SWE-bench Pro / Terminal-Bench。
3. 协议层三足:MCP(2026-07-28 版转向 stateless 分布式)+ A2A(LF 治理,150+ 组织)+ Agent Plugins(08 月新出的打包分发标准)。
4. 编码代理:模型能力趋同,**harness 成为主要差异点**;Claude Code 居首,终端派与 IDE 派分流;subagent/多会话原语快速固化。
5. 多智能体从"能不能"进入"何时该":Anthropic 08-13 研究揭示羊群效应、群体决策劣化等系统性失败模式。
6. OpenAI Responses API 原生 multi-agent(beta)——多 agent 编排正在从框架层下沉到 API 层。
7. 平台化趋势:managed agents(托管运行时)被 LangChain、各云厂同时押注。

## 背景与问题

本仓库模型知识截止 2026-01,今天是首次运行。本次扫描目标:补齐 2026-02→08 的关键空白,建立"什么算新"的参照系,并验证信源清单可用性。

## 核心发现

### 1. 模型层(2026 上半年全部换代)

| 厂商 | 发布 | 日期 | agent 相关要点 |
|---|---|---|---|
| Anthropic | Claude Sonnet 5 | 2026-06-30 | 主打编码与 agentic 性能([来源](https://www.anthropic.com/news/claude-sonnet-5)) |
| Anthropic | Fable 5 重新部署 | 2026-06-30 | 新增安全评分框架;Fable/Mythos 同底模型、Mythos 面向获批组织([来源](https://www.anthropic.com/news/redeploying-fable-5)) |
| Anthropic | Claude Opus 5 | 2026-07-24 | $5/$25 per MTok 不变;Frontier-Bench 较 Opus 4.8 翻倍以上、ARC-AGI 3 三倍于次名、OSWorld 2.0 超 Fable 5 且成本 1/3;强调"验证并迭代到成功"的长任务能力([来源](https://www.anthropic.com/news/claude-opus-5)) |
| OpenAI | GPT-5.6(Sol/Terra/Luna 三档) | 2026-07-09(预览 06-26) | Responses API 新增:programmatic tool calling、显式 prompt cache 控制、persisted reasoning、**multi-agent 编排(beta,模型可并行拉起 subagent 并汇总)**([官方 builders guide](https://openai.com/index/builders-guide-to-gpt-5-6/)) |
| OpenAI | GPT-5.6-Cyber / Daybreak | 2026-08 | 面向授权安全工作的模型与红蓝队计划([检索摘要](https://releasebot.io/updates/openai)) |
| Google | Gemini Enterprise Agent Platform | 2026-04-22(Next '26) | Vertex AI 演进版:企业 agent 构建/治理/优化一体化([Google Cloud Blog](https://cloud.google.com/blog/products/ai-machine-learning/introducing-gemini-enterprise-agent-platform)) |
| 其他 | Kimi K3、Grok 4.5 | 2026 | 出现在 SWE-bench 前列(93.4% / 86.6%),开源与二线阵营紧咬([BenchLM](https://benchlm.ai/benchmarks/swe-bench-verified)) |

### 2. 评估与基准(as-of 2026-08-17)

- SWE-bench Verified:Opus 5 96.0%(vals.ai 复测 97.0%)> GPT-5.6 Sol 96.2%(不同口径互有先后)> Mythos 5 95.5% > Fable 5 95.0% > Kimi K3 93.4%。**头部拥挤 → 基准饱和,社区共识是转向 SWE-bench Pro / Terminal-Bench / Holistic Agent Leaderboard**([BenchLM](https://benchlm.ai/benchmarks/swe-bench-verified)、[morphllm SWE-bench Pro](https://www.morphllm.com/swe-bench-pro))
- 评估工具侧:LangSmith 推出 **Tuned Evaluators**(可微调评估器,首个为 Perceived Error,2026-08-18),评估器本身开始"可训练化"([LangChain Blog](https://www.langchain.com/blog/introducing-langsmith-tuned-evaluators-starting-with-perceived-error))

### 3. 协议层(三个层面各就各位)

- **MCP 2026-07-28 版**(工具接入层):协议核心**转向 stateless**(移除 session/握手/Mcp-Session-Id),新增 Mcp-Method/Mcp-Name 头与 ttlMs/cacheScope 缓存元数据;**MCP Apps**(服务端渲染交互 UI)与 **Tasks**(长任务)进入正式 Extensions 框架;auth 对齐 OAuth 2.0/OIDC(可接 Entra/Okta);首个正式弃用政策(最早移除 2027-07-28)。定位从"本地工具协议"走向"分布式协议"([官方博客](https://blog.modelcontextprotocol.io/posts/2026-07-28/)、[Claude 适配公告](https://claude.com/blog/bringing-mcp-2026-07-28-to-claude))
- **A2A**(agent 互操作层):Linux Foundation 治理(2025-06 移交);一周年(2026-04-09)150+ 组织、Google/Microsoft/AWS 平台深度集成、供应链/金融/保险/ITOps 生产部署([LF 新闻稿](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year))
- **Agent Plugins**(分发打包层,新):OpenAI 主导,AWS/Cursor/GitHub/Microsoft/Vercel 共建;`plugin.json` manifest 打包 Agent Skills + MCP 配置,"build once, run anywhere";跨厂商 TSC 公开治理;首发支持 7 客户端(VS Code/Cursor/Copilot/ChatGPT-Codex/Kiro/Hermes Agent/OpenClaw);08-06 公布、08-11 v1.0.0 Working Draft([explainx 综述](https://explainx.ai/blog/agent-plugins-openai-standard-aws-cursor-github-vscode-2026))
- 分工图景:**MCP 管运行时工具调用,A2A 管 agent 间通信,Agent Plugins 管能力打包分发**——三者互补而非竞争。

### 4. 框架层

- LangGraph 1.0 与 LangChain 1.0 于 2025-10-22 GA:核心图原语零 breaking change,semver 承诺到 2.0;`langgraph.prebuilt` 弃用、能力并入 `langchain.agents`([LangChain changelog](https://changelog.langchain.com/announcements/langgraph-1-0-is-now-generally-available))
- Google ADK:model-agnostic、多 agent 原生,与 Gemini Enterprise Agent Platform/Vertex 深绑([ADK docs](https://google.github.io/adk-docs/))
- 趋势:**编排能力下沉**——OpenAI 把 multi-agent 做进 Responses API,Anthropic 把 subagent 做进 Claude Code/Agent SDK,框架层的独特价值被挤压向 durable execution、可观测、企业治理。
- **Managed agents** 叙事兴起:LangChain 称托管 agent 基础设施是下一个大方向(2026-08-12);对应 Anthropic 的 Managed Agents、Google 的 Agent Platform——"自己搭 loop"→"托管运行时"的迁移开始([LangChain Blog](https://www.langchain.com/blog/why-managed-agents-are-the-next-big-thing-in-agent-building))

### 5. 编码代理(格局最活跃的一层)

- 第一梯队(2026-07 口径):**Claude Code(#1,Opus 5 + per-subagent model control)、Codex(#2,Terminal-Bench 纪录)、Cursor、Copilot Agent Mode、Cline**;终端派 vs IDE 派分流,Cursor 3(04-02)反而把 IDE 降为"备用面板"转向 agent-first([redlinesoft 对比](https://blog.redlinesoft.net/posts/agentic-coding-agent-comparison-2026/)、[codersera 指南](https://codersera.com/blog/ai-coding-agents-complete-guide-2026/))
- Artificial Analysis Coding Agent Index(05-18):Claude Code+Opus 4.7 66 > Codex+GPT-5.5 65 > Cursor Composer 2.5 62 —— **同代模型下 harness 差距 < 模型代际差距,但 harness 是当下主要可操作变量**
- Claude Code 侧近两周(CHANGELOG as-of 08-19):subagent forking 默认开启(继承会话与 cache)、`@` 提及跨会话 SendMessage、GitLab MR 支持、用量恢复自动续跑——多会话/多 agent 协作原语固化中
- 新玩家涌现:Muse Code、Antigravity 2.0、Kiro、Hax(C 写的极简终端 agent)、MathCode(数学专用)——细分与极简两个方向都在长

### 6. 多智能体:从"能不能"到"何时该"

Anthropic《Patterns and problems in emerging multi-agent systems》(2026-08-13)([原文](https://www.anthropic.com/research/multiagent-systems)):
- **低方差 → 羊群效应**:同情景下多个 agent 几乎同选择,坏决定被集体复制(18 个 agent 同时建同名分支)
- **群体决策劣化**:隐藏信息任务中群体准确率 17-36%,个体反而接近 100%
- **冲突自发升级**:矛盾目标的 agent 会升级到互删进程/锁账号;新模型能协商化解,但该能力与执行能力不相关
- **认识论脆弱**:弱模型辨谎准确率 85%→62%,缺少声誉/成本信号/追索权等人类社会机制
- 结论:**个体对齐不保证群体协调**;需要显式设计声誉、验证、协商机制
- 与 OpenAI 把 multi-agent 塞进 API 对照:基础设施在加速,而失败模式研究刚起步——落差本身是信号。

### 7. 安全(事件驱动的一课)

Wiz 披露(2026-08-17):Copilot 生成的 "Autofix" PR 引入 GitHub Actions 命令注入 → Snowflake Jira 凭证外泄;攻击方 Wiz Red Agent 也是 AI,自主根据报错调整 payload([原文](https://www.wiz.io/blog/red-agent-snowflake-copilot-cicd-bug))。要点:AI 写码引入漏洞 + AI 攻击利用漏洞,双向风险闭环;自动扫描器对着漏洞代码也没报警;人审 AI 生成的安全敏感代码仍是必需。

## 实践启示

1. 选基准要看 Pro/Terminal-Bench,Verified 数字已无区分度。
2. 多 agent 架构默认从单 agent + subagent(上下文隔离)起步;引入平行多 agent 前先对照 Anthropic 失败模式清单。
3. 新项目工具接入直接按 MCP 2026-07-28 stateless 模型设计;老 server 有一年弃用窗口。
4. 能力分发关注 Agent Plugins:若被 Claude 生态采纳则"技能包"跨端通用,值得提前按 Skills+MCP 结构组织能力。
5. harness 是当下杠杆:同模型不同 harness 差距可量化(AA Index),本仓库自身协议设计也应吸收(checkpoint、subagent、权限分层)。

## 知识库回写

- kb/10-landscape/timeline.md:2026-02→08 大事 12 条
- kb/10-landscape/model-capabilities.md:as-of 2026-08 模型快照(整段重写)
- kb/10-landscape/ecosystem-map.md:分层地图初版(协议三分工、框架收敛、编码代理梯队)
- kb/10-landscape/research-frontiers.md:多 agent 失败模式、评估器可训练化两个方向
- kb/03-protocols/mcp.md、a2a.md:现状节 + 时间线
- kb/04-multi-agent/orchestration-patterns.md:Anthropic 研究核心发现入"现状"
- kb/07-safety-security/prompt-injection.md:Wiz 事件入时间线
- kb/09-coding-agents/claude-code.md、other-coding-agents.md:格局与近期版本
- kb/02-frameworks/langgraph.md:1.0 GA 状态

## 开放问题(队列候选)

1. Agent Plugins spec 深读:manifest 结构、与 Claude Code plugins 的关系、会不会被 Anthropic 采纳(已入队 P4)
2. MCP stateless 迁移的实操影响:现有 server 要改什么(并入队列 P3 MCP 深研)
3. Responses API multi-agent beta 的实际形态:与框架层编排的取舍(并入队列 P5 多智能体对比)
4. SWE-bench Pro / Holistic Agent Leaderboard 的构成与可信度(并入队列 P7 评估工程)

## 参考资料

见文中内链;全部访问于 2026-08-19。二手综述(explainx/BenchLM/redlinesoft 等)已标注,关键数字尽量以官方来源交叉核对:模型发布(Anthropic/OpenAI 官方)、MCP(官方博客)、A2A(LF 新闻稿)。GPT-5.6-Cyber 仅有二手来源,置信度中,后续深研补核。
