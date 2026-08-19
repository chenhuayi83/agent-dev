# 信源清单

> A 层每日必扫;B 层每日轮换 2-3 个(按 run_count 对 B 层条数取模,从游标处顺序取);C 层每周(run_count % 7 == 0)扫一次。
> 评分:产出有效条目 +1(封顶 5);连续 3 次空转 → 降层;连续 3 次打不开 → 移入候删区(P5 处理)。
> 方式:优先结构化接口(JSON API / atom feed / raw 文件),其次 WebFetch 页面,WebSearch 兜底。

## A 层(每日必扫)

| 名称 | 方法 | 质量分 | 最近有效 | 备注 |
|---|---|---|---|---|
| Anthropic News | WebFetch https://www.anthropic.com/news | 3 | 2026-08-19(窗口空) | 官方发布 |
| Claude Code CHANGELOG | WebFetch https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md | 4 | 2026-08-19 | raw 文件,取窗口内版本 |
| OpenAI News | WebFetch https://openai.com/news/ | 3 | — | 2026-08-19 HTTP 403(失败 1/3);WebSearch 兜底可用 |
| HN Algolia(agent) | WebFetch https://hn.algolia.com/api/v1/search_by_date?query=agent&tags=story&numericFilters=points%3E20 | 4 | 2026-08-19 | 结构化 JSON 自带时间戳,最省预算,今日产出最高 |
| LangChain Blog | WebFetch https://www.langchain.com/blog/ | 4 | 2026-08-19 | 旧域名 blog.langchain.com 已 301,勿再用 |

## B 层(轮换,每日 2-3 个)

| 名称 | 方法 | 质量分 | 最近有效 | 备注 |
|---|---|---|---|---|
| Anthropic Engineering | WebFetch https://www.anthropic.com/engineering | 3 | — | agent 工程实践深文 |
| modelcontextprotocol.io | WebFetch https://modelcontextprotocol.io/ | 3 | — | MCP 官方,关注 spec changelog |
| Simon Willison | WebFetch https://simonwillison.net/ | 3 | — | 高信噪比独立观察 |
| arXiv(LLM agent) | WebFetch http://export.arxiv.org/api/query?search_query=all:%22LLM+agent%22&sortBy=submittedDate&sortOrder=descending&max_results=10 | 2 | — | atom 格式论文列表 |
| Google Developers Blog | WebFetch https://developers.googleblog.com/ | 2 | — | Gemini / ADK 动态 |
| Latent Space | WebFetch https://www.latent.space/archive | 2 | — | 行业访谈/周报 |
| GitHub Trending | WebFetch https://github.com/trending?since=daily | 2 | — | 新兴 agent 仓库发现 |

## C 层(每周)

| 名称 | 方法 | 备注 |
|---|---|---|
| openai/openai-agents-python releases | WebFetch https://github.com/openai/openai-agents-python/releases | |
| langchain-ai/langgraph releases | WebFetch https://github.com/langchain-ai/langgraph/releases | |
| google/adk-python releases | WebFetch https://github.com/google/adk-python/releases | 文档站已迁 adk.dev(旧 google.github.io/adk-docs 301,2026-08-19 核实) |
| anthropics/claude-agent-sdk-python releases | WebFetch https://github.com/anthropics/claude-agent-sdk-python/releases | |
| SWE-bench 榜单 | WebFetch https://www.swebench.com/ | |

## 兜底 WebSearch 模板

- `AI agent framework release <当月英文>`
- `"context engineering" <当月英文>`
- `MCP "Model Context Protocol" update`
- `multi-agent orchestration <当月英文>`

## 候删区

(暂无)
