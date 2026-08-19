---
last_updated: 2026-08-19
maturity: growing
deep_dives: [2026-08-19-landscape-baseline]
---

# Google ADK

Google Agent Development Kit:多 agent 层级组合、工具生态、与 Gemini/Vertex 绑定程度、A2A 原生支持。

## 原理(稳定)

(待充实)

## 现状(易变,断言须带 as-of 日期)

as-of 2026-08-19(核实自 [adk.dev](https://adk.dev/)):
- **ADK 2.0 已 GA**,主打特性为 **graph workflows + collaborative agents**——把确定性代码与自适应 AI 推理组合进结构化图架构(与 LangGraph 的图模型正面竞争)
- agent 形态:简单 LLM agent、managed agents、多 agent 协作、模板化 workflow(sequential/loop/parallel/custom)
- **五语言 SDK**:Python、TypeScript/JavaScript、Go、Java、Kotlin(语言覆盖面是其相对优势)
- **model-agnostic**:适配 Gemini、Gemma、Anthropic Claude、OpenAI、Ollama、vLLM、LiteLLM 等,含本地模型;Gemini 为主推
- 部署:Agent Runtime / Cloud Run / GKE 托管,声称无需改代码
- **文档域名已迁移**:`google.github.io/adk-docs` → **`adk.dev`**(301,信源清单需同步)

## 时间线

- 2026-08-19 核实 ADK 2.0 GA(graph workflows);文档迁至 adk.dev
- 2026-04-22 Google Next '26 发布 Gemini Enterprise Agent Platform(ADK 的企业运行时)

## 开放问题

- ADK 在非 Gemini 模型上的可用性?
- ADK 的 evaluation 内置能力现状?
