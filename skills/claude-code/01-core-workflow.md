---
last_updated: 2026-08-19
type: skill
tool: claude-code
level: 入门→进阶
sources_asof: 2026-08-19
---

# 技能:Claude Code 核心心智模型与高效工作流

**你将获得**:
1. 一个能推导出所有官方最佳实践的心智模型(上下文第一性原理 + 验证闭环)
2. 四步黄金工作流的具体操作(含快捷键与何时跳过)
3. 识别并避开官方总结的五大失败模式

## 本质(是什么,为什么存在)

Claude Code 不是"会写代码的聊天机器人",是一个 **agentic coding 环境**:它读你的代码库、编辑文件、运行命令、根据结果自主迭代,你负责给方向和验收。工作方式从"你写代码、AI 审"倒转为"你描述意图、AI 探索-规划-实现,你审"。

这个倒转成立的前提是两件事被管好:
- **上下文**(它"知道"什么)——管不好,它会忘记指令、犯低级错误;
- **验证**(它怎么知道"做对了")——管不好,它在"看起来完成"时就停,你被迫当人肉验证环。

本篇所有技巧都是这两件事的推论。

## 机制(第一性原理)

### 原理一:上下文窗口是最稀缺资源

会话里的一切——你的每条消息、Claude 读过的每个文件、每条命令输出——都进入同一个上下文窗口。**窗口越满,性能越差**(遗忘早期指令、错误率上升)。一次调试或代码库探索就能消耗数万 token。

由此推导出的操作律:
- 无关任务之间 `/clear`(上下文清零成本远低于污染成本)
- 大量读文件的调查工作丢给 **subagent**(它在独立上下文里读,只把结论带回来)
- 长会话触发 auto-compact 前,主动 `/compact <指令>` 控制保留什么
- CLAUDE.md 每会话必载,所以必须极简——每一行都在挤占任务上下文

### 原理二:没有检查,就没有自主

Claude 在"工作看起来完成"时停止。**给它一个能自己运行的检查**(测试、构建退出码、linter、对比脚本、浏览器截图),循环就闭合了:干活 → 跑检查 → 读结果 → 迭代到通过。这是"你盯着的会话"与"你可以走开的会话"的本质区别。

官方提供了四档"检查强度",按无人值守程度递进(as-of 2026-08):
1. **提示词内嵌**:同一条消息里要求"实现后跑测试并修复失败"——任何任务当场可用
2. **`/goal` 条件**:独立评估器每轮后复查目标,未达成就继续
3. **Stop hook**:脚本作为确定性闸门,不通过就阻止回合结束(连续 8 次阻止后强制放行)
4. **对抗评审**:fresh context 的 subagent 只看 diff 和标准来挑错——干活的不给自己打分

### 原理三:具体度换纠错次数

Claude 能推断意图,但不能读心。提示词的具体度直接决定返工率。官方 before/after 模式:

| 模式 | 差 | 好 |
|---|---|---|
| 圈定范围 | "给 foo.py 加测试" | "为 foo.py 写测试,覆盖用户已登出的边界情况,不要用 mock" |
| 指路来源 | "为什么这个 API 这么怪" | "翻 ExecutionFactory 的 git history,总结这个 API 是怎么演化来的" |
| 参照既有模式 | "加个日历组件" | "先看首页现有 widget 的实现模式(HotDogWidget.php 是好例子),照该模式实现日历组件,不引新库" |
| 描述症状 | "修登录 bug" | "用户报告 session 超时后登录失败。查 src/auth/ 尤其 token refresh。先写一个能复现的失败测试,再修" |

例外:探索期故意用模糊提示("这个文件你会改进什么?")来发现你没想到的问题——模糊是工具,不是习惯。

## 实操(四步黄金工作流)

适用:方案不确定 / 涉及多文件 / 不熟悉的代码。**如果 diff 一句话能说清,直接让它干,别走流程**(计划模式有开销)。

1. **Explore(只读探索)**:`Shift+Tab` 切到 plan mode(状态栏 `⏸ plan mode on`),或 `claude --permission-mode plan` 启动。让它读代码回答问题,不改任何东西:
   > read /src/auth and understand how we handle sessions and login
2. **Plan(出计划)**:让它产出实现计划;`Ctrl+G` 在编辑器里直接改计划文本再继续。
3. **Implement(实现+自验)**:批准计划退出 plan mode,让它按计划实现,并**在同一指令里绑定验证**:
   > implement the OAuth flow from your plan. write tests for the callback handler, run the test suite and fix any failures
4. **Commit(收尾)**:让它用描述性 message 提交、开 PR。

配套操作(会话管理):
- **及时纠偏**:`Esc` 打断(上下文保留);`Esc Esc` 或 `/rewind` 回滚对话/代码到任意 checkpoint(每条 prompt 自动存档;注意只覆盖 Claude 编辑工具的改动,Bash 改的文件不在内——不能替代 git)
- **两次纠错定律**:同一问题纠正超过两次,说明上下文已被失败尝试污染——`/clear`,把学到的约束写进新的开场 prompt,几乎总是更快
- **喂料**:`@` 引用文件、直接贴截图、给 URL(`/permissions` 加白常用域)、`cat error.log | claude` 管道灌入
- **大功能先访谈**:"I want to build X. Interview me in detail using the AskUserQuestion tool... then write a complete spec to SPEC.md" → 新会话拿着 SPEC 干净执行

## 反模式与边界(官方五大失败模式)

| 失败模式 | 症状 | 修法 |
|---|---|---|
| 大杂烩会话 | 一个会话里穿插无关任务 | 无关任务间 `/clear` |
| 反复纠错 | 同一问题改了又改 | 两次后 `/clear` + 更好的开场 prompt |
| CLAUDE.md 过度膨胀 | 规则多到被忽略 | 逐行问"删了会出错吗?",不会就删 |
| 信任-验证缺口 | 看似合理的实现漏边界 | 没有可运行的验证就不算完成 |
| 无边界探索 | "调查一下X"读了几百个文件 | 圈定范围,或丢给 subagent |

边界:这套流程假设你在乎正确性与可维护性。纯一次性脚本/原型,直接说需求让它冲即可,流程是税。

## 版本敏感项(as-of 2026-08-19)

- 表面形态:Terminal CLI / VS Code / JetBrains / Desktop / Web(claude.ai/code)/ 移动端,同一引擎共享 CLAUDE.md、设置与 MCP
- Pro/Max/Team 交互会话默认 **auto mode**(分类器模型代你审查动作,只拦截险动作);手动档配 `/permissions` 白名单 + `/sandbox` OS 级隔离
- **auto memory**:Claude 自动跨会话积累构建命令、调试心得(CLAUDE.md 之外的自动层)
- `/goal`、Stop hooks(8 次阻止上限)、agent teams、background agents、routines(云端定时)、`--teleport`、`/desktop` 均为当前功能
- 来源:[官方 best practices](https://code.claude.com/docs/en/best-practices)、[overview](https://code.claude.com/docs/en/overview)(2026-08-19 抓取)

## 练习(自测)

1. 用四步工作流完成一个真实小需求,Plan 阶段用 `Ctrl+G` 手改一次计划
2. 造一个"信任-验证缺口":先不带验证要求提需求,再带上"跑测试并修复"重做同一需求,对比产出
3. 故意在一个会话里混两个无关任务,观察质量;然后 `/clear` 分开做,体感差异
4. 把一次 >50 行输出的调查改用 subagent 做("use subagents to investigate..."),对比主会话上下文占用
5. 面试自测:不看本文,向别人讲清"为什么 /clear 能提升质量"(考察是否真懂原理一)

## 关联

- 面试题:[01 Agent 基础](../../interview/01-agent-fundamentals.md)(Q5 上下文管理与本篇原理一同源)
- kb:[claude-code](../../kb/09-coding-agents/claude-code.md) · [harness-design](../../kb/09-coding-agents/harness-design.md) · [上下文工程总览](../../kb/01-context-engineering/overview.md)
- 后续技能(队列中):CLAUDE.md 工程学 / 上下文管理实战 / 验证闭环设计 / subagents 与并行
