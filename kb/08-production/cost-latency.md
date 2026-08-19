---
last_updated: 2026-08-19
maturity: growing
deep_dives: []
---

# 成本与延迟优化

prompt caching、模型分级(路由)、batch API、并行化、token 预算护栏;成本可观测。

## 原理(稳定)

(待充实)

## 原理(稳定)

**Prompt caching 的回本模型**(这是成本优化里最常被误用的一项):缓存写入比普通输入**更贵**、读取则大幅便宜,因此存在一个"回本次数"——命中次数不够时,开缓存反而亏。计算方式:设写倍率 w、读倍率 r(相对普通输入价),则 n 次复用的相对成本为 `w + (n-1)r` vs 不缓存的 `n`,令其相等即得盈亏平衡点。**推论:一次性任务不要开缓存;高频复用同一前缀(系统提示+工具定义)才是缓存的主场。**

关键工程含义:**任何改动前缀的发布都会击穿缓存**(改系统提示、加工具、换模型),所以发布节奏与缓存收益直接耦合——这也是灰度发布要考虑成本尖峰的原因。

## 现状(as-of 2026-08-19,核实于面试题 08 组撰写,详见 [interview/08-production.md](../../interview/08-production.md) 事实基线小节)

- Anthropic 侧缓存:读 0.1×、写 1.25×(5 分钟档)/ 2×(1 小时档),最小可缓存前缀约 1024 token,缓存断点上限 4 个 → 5 分钟档约 2 次复用回本,1 小时档约 3 次
- OpenAI GPT-5.6(2026-07-09 GA)侧:显式缓存断点、最短 30 分钟缓存寿命、写 1.25×、读约 9 折优惠;另有 persisted reasoning 与 programmatic tool calling
- 两家写入倍率均为 1.25×,可互为独立佐证(而非单一来源)
- 模型定价参照:Claude Opus 5 $5/百万输入、$25/百万输出(2026-07-24 发布价)

## 底层支撑(推理引擎层)

本文讲的是**应用层**成本优化(缓存、路由、并行、预算护栏)。其下一层——KV Cache 显存与 MQA/GQA/MLA、Flash Attention、prefill/decode 两阶段与 TTFT/TPOT、continuous batching 与 PagedAttention、量化——决定了应用层能拿到的成本与延迟下界,系统整理在 [interview/12-llm-engineering.md](../../interview/12-llm-engineering.md)。

两条最该记住的联系:
- **prompt caching 的服务端真身就是 prefix KV 复用**——这解释了为什么前缀必须逐字节不变、为什么有最小长度门槛;
- **并发 agent 的瓶颈通常是 KV 显存而非算力**——长上下文 agent 尤其如此,加 batch 救不回吐字速度。

## 时间线

## 开放问题

- 模型路由(大小模型分工)的实证节省比例?
- caching 命中率的工程手段?
