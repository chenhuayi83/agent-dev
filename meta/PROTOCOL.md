# 每日运行协议(PROTOCOL)

> 版本:v1.0(2026-08-19)。全部变更史见 `meta/EVOLUTION.md`。
> 本文件是每日自动运行的执行手册,按 P0→P6 顺序执行。宪法(`meta/CONSTITUTION.md`)优先级高于本文件。

## 总原则

- **checkpoint 纪律**:每个阶段结束即 `git add -A && git commit` + 更新 `meta/state.json` 的 `pending` 字段。任何时刻会话被杀,下次运行都能从 pending 续跑。
- **预算**:P1 ≤12 次网络操作,P2 ≤12 次,全局软预算 ≤30 次(宪法硬上限 40)。上下文使用感知超过约 70% 时,跳过未开始的可选阶段,直奔 P6。
- **变体**:
  - 用户点名主题 → P2 深研该主题(跳过队列取题);
  - `/deep-dive <题>` → P0 → P2(P1 跳过)→ P3(条件触发)→ P4 → P6;
  - `/status` → 只读汇报,不修改任何文件、不联网;
  - 同日第二次运行(state.last_run == 今天)→ 轻量模式:只完成 pending 和简报增量,不重复深研、不重复消耗预算。

## P0 启动自检(零联网,≤2 分钟)

1. 定位环境(不硬编码路径):
   ```bash
   COMMON=$(git rev-parse --path-format=absolute --git-common-dir)
   MAIN=$(dirname "$COMMON")          # 主仓库路径
   BR=$(git branch --show-current)    # 当前分支;等于 main 则 P6 跳过合并
   ```
2. 读 `meta/state.json`:
   - `pending` 非空 → 先完成遗留阶段,再继续当天流程;
   - `last_run == 今天` → 轻量模式(见变体);
   - 扫描窗口 = `last_run` 至今,上限 7 天(隔多天跑不逐日补齐,一次覆盖)。
3. **前日自愈**:`git -C "$MAIN" branch --no-merged main --list 'claude/*'` 列出未合并分支(排除当前分支)。对每个:尝试 `git -C "$MAIN" merge --ff-only <分支>`;不能 ff 的不强行处理,记入收尾汇报。处理完后本分支执行 `git merge main --no-edit` 把成果带进来。
4. 维护(best-effort,失败忽略):`git -C "$MAIN" worktree prune`;`git -C "$MAIN" branch -d` 删除已合并的旧 `claude/*` 分支(排除当前分支)。

## P1 情报收集 → 每日简报(≤12 次网络操作)

1. 读 `meta/sources.md`:A 层全扫 + B 层按 `run_count` 取模轮换 2-3 个;`run_count % 7 == 0` 时加扫 C 层。
2. 只关注扫描窗口内的新内容。优先结构化接口(HN Algolia API、arXiv API、GitHub releases),其次页面抓取。
3. **去重**:每条候选先用 URL(去协议、去查询参数)`grep -F` 查 `meta/seen.jsonl`,再用标题 2-3 个关键词模糊 grep;命中即弃。
4. 值得展开的最多 4 条,WebFetch 原文定向提炼(带 prompt 参数,不整页搬运)。
5. 按 `meta/templates/briefing.md` 产出 `briefings/YYYY/YYYY-MM-DD.md`,≤120 行。硬规则:
   - 要闻 ≤5 条,每条必须有:一句话事实 + 为什么重要 + 来源链接(发布日期)+ 处置标记;
   - 值得一瞥 ≤8 条,每条一行;
   - 平静日直接写"平静日",保持短(宪法 C7)。
6. 全部新条目追加 `meta/seen.jsonl`:`{"d":"日期","u":"url","t":"标题","k":"news|paper|release|post"}`(按年轮转:seen-2026.jsonl 之类,当前统一写 seen.jsonl)。
7. checkpoint 提交:`brief: YYYY-MM-DD`

## P2 专题深研(≤12 次网络操作)

1. 取题:`state.current_deep_dive` 非空(跨日大主题)→ 续做;否则取 `meta/queue.md` 待研区最高优先级;用户点名则覆盖。
2. 打法:1-2 次 WebSearch 定位一手资料 → 6-8 次 WebFetch 精读(官方文档/spec/源码/作者博文 优先于二手转述)→ 关键断言尽量两处独立来源交叉验证。
3. 按 `meta/templates/deep-dive.md` 产出 `deep-dives/YYYY/YYYY-MM-DD-<slug>.md`,≤400 行:TL;DR ≤10 行;核心发现每个关键断言带来源+日期;实践启示;知识库回写清单;开放问题。
4. 判断是否配实验(P3 三条件);不满足则在报告中明确写"无需实验,理由:…"。
5. 队列回写:该题移入"已完成"区并链接产出;报告开放问题中最有价值的 1-2 个入队。
6. checkpoint 提交:`deep-dive: <slug>`

## P3 代码实验(条件触发,不是每天)

触发三条件(**全满足才做**):① 主题行为可本地验证(API 行为/协议消息/框架对比/性能);② 预估 30 分钟内跑通最小版本;③ 结果能实质改变结论置信度。纯生态盘点/新闻类主题不做实验。

- 结构:`experiments/YYYY-MM-DD-<slug>/{README.md, src/, FINDINGS.md}`(模板见 `meta/templates/experiment.md`)。
- 自包含:Python 用 `uv run`(PEP 723 内联依赖);Node 带 lockfile;一条命令可复跑。
- 默认不调付费 API;必须调时 ≤10 次且用最小可用模型(宪法 C5);无 key 环境须支持 `--dry-run` 降级。
- 装不上依赖/跑不通:降级为纸面分析,README 标 `status: blocked`,不死磕。
- checkpoint 提交:`exp: <slug>`

## P4 知识沉淀回写

1. **毕业标准**(满足其一才进 kb):三个月后仍有用;属于某主题的机制/原理/权衡;纠正 kb 中已过时的断言。
2. 事件类信息(发版/发布/融资)只在对应 kb 文件"时间线"小节加一行,不展开。
3. kb 文件规范:front-matter(`last_updated` / `maturity: stub|growing|mature` / `deep_dives` 关联);"原理"(稳定)与"现状"(易变,断言带 as-of 日期)分节;单文件 ≤500 行,超限拆分或精炼(精炼=压缩表达,不减信息量)。
4. 更新 `kb/_index.md` 对应行(简介+成熟度)。
5. checkpoint 提交:`kb: <涉及文件>`

## P5 自我进化复盘

1. **重读 `meta/CONSTITUTION.md` 全文**(强制,防漂移锚点)。
2. 复盘三问:
   - 今天哪个环节浪费了预算或产出了水货?
   - 哪个信源连续空转?(连续 3 次无产出 → 降层或移入候删区)
   - 协议哪一条实际没被遵守?(说明它不合理或表述差,是改进候选)
3. 允许改(宪法边界内):`meta/sources.md`(增删/评分/换扫描方式)、`meta/queue.md` 排序、`meta/templates/*`、`meta/PROTOCOL.md` 的流程细节与预算参数(硬上限内)。
4. PROTOCOL.md 改动纪律(宪法 C4):每次 ≤1 处实质改动;版本 +0.1;**独立 commit**(`meta: protocol v1.x <原因>`,便于单独 revert);`meta/EVOLUTION.md` 追加记录(改动/动机/预期检验/回滚方式/幅度:小=参数措辞、中=步骤增删;大改走 PROPOSALS)。**没有值得改的就不改——为改而改也是漂移。**
5. 周元复盘(`run_count % 7 == 0`):抽查近 7 天简报是否同质化;kb 园艺(合并重复、修剪过长时间线);队列健康(<5 题从 kb"开放问题"补队);验收近一周协议改动是否兑现"预期检验",未兑现 → `git revert` 回滚该改动。
6. checkpoint 提交:`meta: 复盘 YYYY-MM-DD`(若有协议改动,协议改动单独成 commit)

## P6 收尾(优先级最高,宪法 C10)

1. 更新 `meta/state.json`:`last_run`、`run_count`+1、`pending` 清空、`counters`、`recent_new_items`(最近 3 天简报新条目数,质量金丝雀)、`current_deep_dive`、`source_rotation_cursor`。
2. 更新 `README.md`"最近产出"表(保留最近 7 天,旧行删除不算删历史——历史在 briefings/ 里)。
3. 最后提交:`daily: YYYY-MM-DD 收尾`。
4. 合并回 main(`BR == main` 则整段跳过):
   ```bash
   # ① worktree 内先拉平 main(main 未动时为 no-op;冲突在这里解:
   #    state.json 以本次运行为准重写;queue/seen/sources 取并集;briefings/deep-dives/kb 两边内容都保留)
   git merge main --no-edit
   # ② 主仓库已跟踪文件有脏改动 → 封存为提交(绝不 add -A,防吸入未跟踪杂物;不用 stash)
   if [ -n "$(git -C "$MAIN" status --porcelain --untracked-files=no)" ]; then
     git -C "$MAIN" add -u && git -C "$MAIN" commit -m "chore: 封存 main 手工改动(daily merge 前)"
     git merge main --no-edit   # 回到①再拉平,至多循环 2 次
   fi
   # ③ 主仓库中与本分支新增文件同路径的未跟踪文件 → 改名 <name>.local-backup-YYYYMMDD,并在汇报中说明
   # ④ 快进合并(经①②后应必然可 ff)
   git -C "$MAIN" merge --ff-only "$BR"
   #    仍失败 → 不做任何破坏性操作;保留分支;汇报手工命令;下次运行 P0 自愈(宪法 C3)
   # ⑤ best-effort push(失败/超时只记录,绝不阻塞收尾)
   GIT_SSH_COMMAND="ssh -o BatchMode=yes -o ConnectTimeout=5" git -C "$MAIN" push origin main || true
   ```
5. 汇报(≤30 行,宪法 C9):今日要闻 ≤3 条 / 深研结论一段 / 协议改动(或"无")/ 异常与待批提案 / 明日预告(队列下一题)。

## P7 附录:bootstrap(已于 2026-08-19 完成,防御性保留)

仅当 `meta/PROTOCOL.md` 不存在时触发(正常情况下永不再触发):建 `.gitignore`(必须第一个提交)→ 建全部骨架(CLAUDE.md / meta/* / kb stubs / templates / commands / README)→ 照常跑 P1–P6,深研题固定为"全景基线扫描",当日预算可上浮至宪法硬上限。
