---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# 模型能力快照

主流模型的 agent 相关能力对比(as-of 快照,过时即整段重写):工具调用质量、长上下文、思考模式、价格。

## 原理(稳定)

- 快照只记"agent 相关"维度:agentic 基准、工具调用、长任务持久性、价格;通用基准不在此维护。
- 不同评测口径数字会打架(官方 vs 第三方复测),记录时注明口径。

## 现状(as-of 2026-08-19,整段重写制)

**旗舰梯队(agentic 视角)**

| 家族 | 当前旗舰 | 发布 | agent 要点 |
|---|---|---|---|
| Anthropic Claude 5 | Opus 5;Fable/Mythos 5(同底,Mythos 面向获批组织);Sonnet 5 | 07-24 / 06-30 / 06-30 | Opus 5:$5/$25,OSWorld 2.0 超 Fable 5 且成本 1/3,长任务"验证-迭代到成功"叙事 |
| OpenAI GPT-5.6 | Sol(旗舰)/ Terra(均衡)/ Luna(廉价) | 07-09 | Responses API:multi-agent beta、programmatic tool calling、persisted reasoning、cache 控制 |
| Google Gemini 3 | Gemini 3 + Enterprise Agent Platform | 平台 04-22 | 企业 agent 平台绑定 Vertex;ADK 为官方框架 |
| 开源/其他 | Kimi K3、Grok 4.5 | 2026 | SWE-bench Verified 分别 93.4% / 86.6%,紧咬第一梯队 |

**SWE-bench Verified(as-of 2026-08-17,饱和中)**:Opus 5 96.0%(vals.ai 复测 97.0%)、GPT-5.6 Sol 96.2%、Mythos 5 95.5%、Fable 5 95.0%、Kimi K3 93.4%、GPT-5.6 Luna 93.0%、Opus 4.8 88.6%、Grok 4.5 86.6%。头部 5 家挤在 ~4 分内 → **区分度已失,看 SWE-bench Pro / Terminal-Bench**([BenchLM](https://benchlm.ai/benchmarks/swe-bench-verified))。

**判断**:同代旗舰在 agentic 能力上趋同;选型变量转向价格档位(GPT-5.6 三档 vs Claude 按型号)、生态绑定(工具/平台)、harness 适配。

## 时间线

(模型发布事件统一记在 [timeline.md](timeline.md),此处不重复)

## 开放问题

- SWE-bench Pro / Terminal-Bench 上的真实排序与差距?(队列 P7 评估工程覆盖)
- 各家旗舰的工具调用错误率/并行调用上限等工程指标缺公开数据
