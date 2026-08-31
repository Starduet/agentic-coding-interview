# ch04 · Agent 架构与循环

> 覆盖:agent loop 的工程实现、停止条件与死循环防护、权限系统、hooks 机制、Harness 四层模型与四问(书)、plan mode 与 Explore-Plan-Code-Commit、system prompt 的结构、headless/自动化模式。

---

## 4.1 八股题(基础概念)

**Q4.1【八股·L2·平台】写出 Claude Code 风格 agent 主循环的伪代码,标出所有终止条件。**
答:
```
while true:
    response = API(system_prompt, tools, history)
    if response.stop_reason == "end_turn":      # 终止1:模型不再调用工具
        return final_text
    for tool_call in response.tool_use:          # stop_reason == "tool_use"
        (可选)权限检查: permission_mode / canUseTool / PreToolUse hook
        result = execute(tool_call)              # 终止2:max_turns 只计工具轮
        history.append(tool_result)              # 终止3:max_budget_usd 预算熔断
    if 上下文接近上限:
        auto_compact(history)                    # 清旧工具输出→摘要
```
- 官方三阶段描述:gather context → take action → verify results,可串联数十个动作并中途纠偏。

**Q4.2【八股·L2·平台】Claude Code 的权限模式有哪几种?规则评估顺序?**
答:
- 模式:default(逐次批准)、acceptEdits、plan(只读)、auto(独立分类器模型审查,2026 起默认,官方称拦截 89% 人类漏判的危险操作)、dontAsk(未 allow 即拒)、bypassPermissions(仅隔离环境)。
- 顺序:**deny → ask → allow,fail-closed**(未匹配即问人/拒绝);任何层的 deny 无法被其他层 allow 覆盖(managed > 项目 > 用户)。
- 关键细节:**PreToolUse hook 在权限提示之前运行,退出码 2 可阻断且 allow 规则绕不过**——典型用法是 allow 全部 Bash 再用 hook 拦 rm -rf、git push --force。

**Q4.3【八股·L2·平台】hooks 的核心事件有哪些?退出码语义?**
答:
- 事件:PreToolUse / PostToolUse / UserPromptSubmit / Notification / **Stop** / SubagentStop / PreCompact / SessionStart(End);(2026 版新增 ToolSelect、PermissionRequest 等)。
- matcher 支持精确名或正则(如 `Bash(git push:*)`、`Edit|Write`);工具输入以 JSON 经 stdin 传入。
- 退出码:0=成功(stdout 仅进 transcript);**2=阻塞**(stderr 回喂给 Claude);其他非零=非阻塞告警。高级 JSON 输出可返回 permissionDecision: allow/deny/ask、改写 updatedInput、注入 additionalContext。
- 典型集成:PostToolUse(Edit|Write)→自动 format/lint;PreToolUse 拦危险命令;Stop→全量测试通过才许结束(连续拦截 8 次后强制放行,防死锁)。
- 安全注意:hooks 是 shell 执行=提示注入新攻击面,只信任入库的 settings。

**Q4.4【八股·L1·用户】书的 Harness 四问是什么?**
答:①agent 在哪里跑(执行环境:worktree/容器/沙箱/权限);②agent 能不能看懂"正在发生什么"(运行时可见性:日志/指标/DOM/截图/trace);③agent 怎么证明自己做对了(机械验证:tests/lint/typecheck/契约/截图对比);④失败之后系统会不会越跑越聪明(收敛与抗熵:review loop/清理任务/经验升级为 rule)。
对应四层能力:**执行环境层 / 运行时可见性层 / 机械验证层 / 收敛与抗熵层**。

**Q4.5【八股·L1·用户】什么是 plan mode?它解决什么问题?**
答:
- 只读规划模式:agent 先探索代码、产出实施计划,人批准后再进入执行(书:plan 模式本质是轻量 Spec 入口)。
- 解决:①探索产生的大量检索噪声不污染执行上下文(计划批准后从头开始执行);②人对方向提前纠偏,避免"写完才发现方向错"的返工;③Anthropic 工作流 Explore→Plan→Code→Commit 的中间态。
- 判据(best practices):"能用一句话描述 diff 的改动就跳过计划"。

**Q4.6【八股·L2·用户】Claude Code 的 headless/自动化模式怎么用?典型场景?**
答:
- `claude -p "..." --output-format stream-json --verbose` 逐行 JSON 输出,进 CI/脚本;`--permission-mode auto -p` 全自动(如"修掉所有 lint 错误");`--allowedTools "Edit,Bash(git commit *)"` 白名单放行;`--max-turns` 限轮次。
- 场景:for 循环扇出批量迁移、GitHub Actions 里 @claude 触发 PR 处理、/batch 用 5–30 个子代理并行开 PR。

**Q4.7【八股·L2·平台】Claude Code 的 system prompt 是"一条 prompt"吗?泄露分析揭示了什么结构?**
答:
- 不是:数百条字符串(515+)按环境/配置动态拼装;含 27 个内置工具描述、subagent prompt(Explore 862 tokens、Plan 1066 tokens)、各种工具流程 prompt;片段从 12 tokens 到 12,515 tokens 的安全监视器。
- 内容类型:tone 指令(简洁、禁 emoji、代码引用 file:line 格式)、任务管理(强制 TodoWrite 跟踪)、安全指令(对外部内容设不可信信任边界)、git 安全(worktree/共享 stash 警告)。
- 启示:生产 system prompt 是**模块化拼装系统**,不是作文——这也是缓存设计(稳定拼接)与配置管理(按环境开关)的需要。

---

## 4.2 原理深挖

**Q4.8【原理·L3·平台】设计 agent 的停止条件体系:四层闸门分别是什么?各自防什么?**
答:
1. **自然停止**:模型不再产出 tool_use(end_turn)——任务完成的主路径;
2. **硬上限**:max_tokens(每次调用必填)、max_turns(只计工具轮)、预算熔断——防失控烧钱;
3. **确定性闸门**:Stop hook / 独立 evaluator(如 /goal 每轮复查)——防"模型自认为完成"(自我验证不可靠的实证见 ch07);
4. **退化检测**:重复动作(同工具同参反复调用)、上下文 thrashing(超大输出反复触发 compact,几次后放弃报错)、成本增速异常。
- 经验法则:同一问题连续 2 次修正失败就该 /clear 换思路,而不是在同一上下文里继续堆错误。

**Q4.9【原理·L3·架构】为什么说"guardrails 不是 Harness 本体"?(书 Part 3.4)**
答:
- guardrails(护栏:权限、审批、deny 规则)回答"不能做什么",是防御性的;
- Harness 本体是**让 agent 真能稳定推进**的整条 how loop:执行现场(隔离环境)、最小 verifier、运行时观测、checkpoint、rollback、review loop 一起进入主干开发;
- 只有护栏没有验证链的 agent 是"安全地不干活";只有验证没有护栏是"高效地闯祸"。四层能力缺一不可。

**Q4.10【原理·L4·平台】Anthropic《Effective harnesses for long-running agents》(2025.11)的"轮班工程师"比喻对应的工程方案是什么?**
答:
- 问题:长任务跨多个会话/上下文窗口,每个新会话像没有上一班记忆的接班工程师。
- 方案:双 agent 架构——initializer 建环境(init.sh、claude-progress.txt、JSON 功能清单),coding agent 每会话只做一个 feature;
- **git commit 即 checkpoint**;每会话固定"开机序列":读 git log→读进度文件→选最高优先级未完成项→跑端到端测试;
- 细节智慧:功能清单选 **JSON 而非 Markdown**(模型更不易乱改);用 Puppeteer 截图"像人类用户一样测试"。
- 本质:把"记忆"从上下文迁移到 repo——与 ch06 的 system of record 一脉相承。

**Q4.11【原理·L3·架构】worktree 隔离解决什么问题?Claude Code 的实现要点?**
答:
- 问题:多个 agent/任务共用一个工作目录会互相污染(半成品文件、dev server 状态、git 冲突)。
- 实现:`claude --worktree/-w <name>` 在 .claude/worktrees/ 下建工作树;`--worktree "#1234"` 直接检入 PR;`.worktreeinclude` 复制 .env 等未跟踪文件;运行期持有 git worktree lock;隔离会话有 4 项检查阻止 agent 改主检出目录(git -C、GIT_DIR 等逃逸都被拦);subagent 可声明 `isolation: worktree`。
- 书的呼应:OpenAI Harness 文章的"per git worktree 启动应用"——一任务一 worktree、一 app 实例、一套日志,是稳定性的前提,不是花活。

**Q4.12【原理·L3·平台】解释 checkpoint/rewind 机制及其与 git 的关系。**
答:
- Claude Code(2025.09):每次编辑前自动快照;双击 Esc 或 /rewind 恢复代码/对话/环境三态;**仅覆盖 Claude 的编辑类工具,Bash 造成的外部改动不在内**;文件历史本地存 30 天;官方明确"不是 git 替代"。
- 设计含义:回滚粒度是"agent 的编辑动作",不是"仓库状态"——bash 的副作用(安装、迁移、外部系统调用)需要额外的快照/设计才能回滚,这是 agent 回滚与传统事务回滚的本质差异。

---

## 4.3 品味与判断题

**Q4.13【品味·L3·平台】"必须发生的事交给 hooks,不许发生的事交给权限,最好发生的事交给 prompt"——请 critique 这个分工。**
参考思路:
- 合理内核:确定性递增(hook 是代码、权限是策略、prompt 是软引导);书的原则"硬约束交给 hooks 而非 CLAUDE.md"同源。
- 补充与例外:hooks 本身是攻击面(不可信仓库的 settings 不能执行);权限系统管工具层管不了语义层(allow Bash 拦不住恶意脚本内容);prompt 层的责任被低估——"信任边界"提示(system prompt 里的不可信数据声明)是注入防御的第一道。
- 优秀回答会指出第四格:验证(Stop hook/verifier)是"必须不发生假完成"的确定性,常被漏掉。

**Q4.14【品味·L4·架构】一个 agent 系统应该"胖 loop 瘦工具"还是"瘦 loop 胖工具"?**
参考思路:
- 胖 loop(编排逻辑在 harness:复杂状态机、重试、路由)→ 可控可审计,但僵硬、把智能从模型挪回代码;
- 瘦 loop(while + 少量正交工具,智能在模型)→ 灵活、跟模型能力一起涨,但可预测性差;
- 2024–2026 的行业收敛:瘦 loop + 好工具 + 确定性闸门在关键节点(权限/验证/停止);"把控制流写死"的框架(LangChain 式链)在 coding 场景败给了 while 循环。
- 判断函数:错误成本越高,越多逻辑下沉到确定性代码;探索性越强,越多留给模型。

**Q4.15【品味·L3·用户】你如何决定什么时候打断正在运行的 agent?给出你的"干预阈值"。**
参考思路:
- 方向性错误(读了 3 个文件就在错模块动手)→立即打断,方向错误的每一步都是负资产;
- 小错误在验证环里自愈(测试红了)→不拦,让它跑闭环;
- 连续 2 次修同一问题失败→打断并 /clear 或降级任务;
- 危险动作(权限弹窗)→永远人工;
- 书的呼应:course-correct 早期,纠偏成本随任务推进超线性增长。

**Q4.16【品味·L4·管理】auto 权限模式(分类器模型代替人审批)你敢在生产开吗?给决策依据。**
参考思路:
- 支持面:官方评测拦截 89% 人类漏判;人对高频审批会疲劳性全点允许,人工兜底常常是幻觉;
- 反对面:分类器误放的尾部风险不可审计;敏感环境(生产凭证、付费操作、不可逆动作)应有不可自动化的硬闸;
- 决策:按动作分级——读操作自动、写工作区自动+沙箱、外发/付费/生产变更保留人工;分级清单本身要 version control。

---

## 4.4 开放性问题

**Q4.17【开放·L5·架构】agent loop 会演化成什么?"loop 消失"(模型原生循环/连续执行流)意味着什么?**
讨论框架:interleaved thinking 已经把"思考"嵌入循环;训练侧 agentic RL 让模型学会多轮策略(见 ch09);loop 的工程职责(权限/验证/恢复)会下沉为平台原语还是保留在 harness;可观测性如何跟上一个"不轮询"的执行流。

**Q4.18【开放·L4·架构】"把 agent 当操作系统进程设计"(信号/cgroup/命名空间/审计)这个类比哪里成立哪里失效?**
讨论框架:成立——隔离、资源限制、退出码、日志都同构;失效——agent 的"执行"是概率性的,重放不可精确、fork 语义(上下文复制)昂贵、副作用跨越数字边界(外部 API);推论:进程模型可以借鉴,验证模型必须重构。

---

## 4.5 工程实战题

**Q4.19【实战·L3·平台】从零实现一个最小 coding agent(Typescript,~300 行),列出你的组件清单与关键决策。**
期望框架:
- LLM client(messages API + tools 定义 + usage 统计);工具集:Read/Write/Edit/Glob/Grep/Bash(各带输出截断);权限层(canRun 钩子 + 默认 ask);循环(伪代码同 Q4.1,含 max_turns/budget);compaction 触发器(token 阈值,先清工具输出);transcript 落盘(JSONL,可重放);
- 关键决策:工具输出上限、稳定前缀(tools 冻结)、错误回喂格式(语义化)、终止条件分层。
- 加分:预留 hook 点、subagent 抽象、stream-json 输出。

**Q4.20【实战·L3·平台】为你们 agent 设计 Stop-gate:任务"必须通过全部测试+lint+无未说明的文件变更"才算完成。**
期望框架:Stop hook 拉起验证脚本;失败时 exit 2 并把失败摘要喂回;8 次拦截上限的自定义策略(3 次后建议 escalate 而不是硬拦);验证产物(测试报告)作为 additionalContext 注入;防止 agent 为了过 gate 篡改测试(测试只读、gate 脚本不在 agent 可写路径)。

**Q4.21【实战·L4·架构】设计一个支持 20 个并行 agent 任务的执行环境层(不依赖任何商业产品)。**
期望框架:每任务一个 worktree/容器;统一任务队列+锁文件(认领制);共享只读缓存(依赖、构建产物);网络策略(默认禁网+白名单);日志/trace 按任务 ID 隔离收集;资源限额(CPU/时长/token);回收策略(完成即清理,失败现场保留 N 天);人工介入通道(暂停/审批/接管)。参照 Anthropic C 编译器实验与 Codex worktrees。

---

## 4.6 场景排错题

**Q4.22【排错·L3·平台】现象:agent 在同一问题上循环 40 分钟,反复"读文件→改→跑测试→改回"。你的 loop 级检测与干预?**
排查链路:
- 检测:重复签名(工具+参数 hash 的滑窗重复率)、同一测试文件在 N 轮内被改多次、错误信息相似度;
- 干预分级:注入 system-reminder("你已尝试 X 次相同方案,请换思路");强制 compact(可能早期约束已丢);Stop-gate 拦截并要求输出 blocker 报告;升级人工;
- 根治:该问题通常是任务定义缺验收标准或工具反馈误导(测试本身 flaky/错误信息不可读),修 loop 不如修验证器。

**Q4.23【排错·L2·用户】agent 每次都在权限弹窗上等你,一天 200 次点击,但你又不敢全放行。怎么配置?**
排查链路:按风险分层放行——读类工具(Read/Grep/Glob)全 allow;Edit/Write 限工作区路径(gitignore 语法);Bash 用 allowlist 前缀(Bash(git *)、Bash(pnpm test*));高危(git push、rm、curl 外网)交给 PreToolUse hook 精确拦截;把"必须发生的"(format)交给 PostToolUse 自动化——目标:弹窗只剩真正需要判断的动作。

**Q4.24【排错·L4·平台】headless 模式在 CI 里随机挂:一半是 max_turns 超限,一半是等审批超时。诊断与修复?**
排查链路:CI 环境≠交互环境——审批必须有确定答案(acceptEdits/auto+白名单,dontAsk 拒绝也算确定);max_turns 超限的 case 分析:任务粒度太大(拆小)or 工具反馈差导致绕圈(修工具);为 CI 单独产出结构化结果(exit code 语义、JSON 报告);token/时间预算熔断并把部分完成的 diff 保留供人接手。
