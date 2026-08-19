---
last_updated: 2026-08-19
maturity: growing
deep_dives: []
---

# 记忆架构

短期(会话内)/长期(跨会话)记忆分层;文件式、向量式、图式(时序知识图谱)记忆的机制与取舍;写入策略(何时记、记什么)。

## 原理(稳定)

(待充实)

## 现状(as-of 2026-08-19,核实于面试题 05 组撰写,详见 [interview/05-memory.md](../../interview/05-memory.md))

- **文件式记忆的工业实现(Claude Code)**:`CLAUDE.md` 四级作用域 + **auto memory**(自动跨会话积累);auto memory 存于 `~/.claude/projects/<project>/memory/`,索引文件 `MEMORY.md` 仅加载前 200 行或 25KB,超限返回错误要求重写索引;条目带 `modified` ISO 8601 时间戳。**关键语义:memory 是 context(供模型参考),不是 enforced configuration(强制配置)**——想要强制必须用 hooks
- **Anthropic memory 工具**为 client-side 文件操作形态(2025-09-29 context management 公告),厂商自报评测口径的收益数字(如 token 削减比例)**不可跨系统横比**
- **图式记忆**代表实现 Graphiti:bi-temporal 时间追踪、temporal edge invalidation(事实失效而非删除)、vector + BM25 + graph traversal 三路混合检索——解决"过期事实"这一记忆系统的核心失败模式
- 记忆产品(Mem0/Zep/Letta/LangMem)的具体能力对比仍待专题深研(队列 P12)

## 时间线

## 开放问题

- 文件式记忆(memory tool 路线)vs 向量记忆的实证对比?
- 记忆写入的质量控制(防垃圾累积)?
