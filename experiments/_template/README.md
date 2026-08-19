# 实验目录说明(_template)

新实验复制本目录为 `experiments/YYYY-MM-DD-<slug>/`,按 `meta/templates/experiment.md` 填写 README。

结构:
- `README.md` — 目的/假设/一条命令运行方式/status(done|blocked|dry-run)
- `src/` — 自包含代码(Python 用 `uv run` PEP 723 内联依赖;Node 带 lockfile)
- `FINDINGS.md` — 实验跑完必写:实际输出、结论、意外发现

约束(宪法 C5):默认不调付费 API;必须调时 ≤10 次且用最小可用模型;无 key 须支持 dry-run。
