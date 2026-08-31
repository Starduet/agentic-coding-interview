# Agentic Coding 面试题库

> 一套面向「AI 原生工程 / Agentic Coding」领域的综合面试题库。
> 内容来源:① 工作区开源书《AI 原生工程》(ai-native-engineering,五大部分:破局 / 定盘 / 长主干 / 接世界 / 自运行 + MultiAgent 长笔记);② 对 2024–2026 年官方工程博客、论文、基准榜单、产品文档的大规模 web 调研(Anthropic / OpenAI / Google / Scale / METR / arXiv 等 300+ 来源交叉核验)。
> 事实基线:2026-08。本领域迭代极快,涉及价格、模型版本、产品功能的数字请在引用前复核官方页面。

---

## 一、题库设计体系

### 1. 题型分类

| 题型 | 缩写 | 考什么 | 答题形态 |
| --- | --- | --- | --- |
| 八股题 | 【八股】 | 概念、机制、数字等可核验知识 | 有参考答案要点 |
| 原理深挖 | 【原理】 | 机制背后的 why、量化 trade-off | 有参考答案要点 |
| 品味题 | 【品味】 | 判断力、审美、取舍(无标准答案) | 给考察点 + 参考思路 |
| 开放题 | 【开放】 | 系统性思考、前瞻判断 | 给考察点 + 讨论框架 |
| 工程实战 | 【实战】 | 真实工程问题的解决方案设计 | 给期望回答框架 |
| 场景排错 | 【排错】 | 给定故障现象,定位根因 | 给排查链路与判据 |

### 2. 难度分级

| 级别 | 定位 | 对应候选人 |
| --- | --- | --- |
| L1 | 初级:知道概念、用过工具 | 校招、实习生、刚迁移到 AI 工作流的工程师 |
| L2 | 中级:理解机制、能落地配置 | 1–3 年经验、日常重度使用 coding agent |
| L3 | 高级:懂原理、能设计系统、有量化意识 | 平台/infra 工程师、tech lead |
| L4 | 专家:能权衡架构、有失败经验、能治理 | 资深架构师、agent 平台负责人 |
| L5 | 前沿:研究级判断、能与最新论文/实践对话 | 研究员、前沿探索者 |

### 3. 面试官视角(每题标注)

| 视角 | 标签 | 典型出题关注 |
| --- | --- | --- |
| 一线重度用户 | 【用户】 | 实操习惯、效率、翻车经验、工具选择 |
| Agent 平台工程师 | 【平台】 | loop 实现、上下文管理、成本、可观测性 |
| 模型/算法研究员 | 【模型】 | 训练范式、推理机制、评测方法学 |
| 架构师/技术负责人 | 【架构】 | 系统分层、边界设计、长期演进、组织落地 |
| 安全工程师 | 【安全】 | 攻击面、沙箱、权限、供应链、审计 |
| 产品/工程管理 | 【管理】 | 生态位判断、商业模式、组织效能度量 |

### 4. 与书的映射

| 题库章节 | 主要对应书的部分 | 调研补充 |
| --- | --- | --- |
| ch01 认知与范式 | Part 1 破局(范式转移/工具版图/控制面) | Agent 定义之争、Anthropic 六模式 |
| ch02 上下文工程与缓存 | Part 1 控制面 + Part 3 长任务 | 缓存/压缩/context rot 全部硬数字 |
| ch03 工具调用与 MCP | Part 1 CLI vs MCP + Part 4 接世界 | function calling/MCP 协议细节 |
| ch04 Agent 架构与循环 | Part 3 长主干(Harness) | agent loop/停止条件/hooks/worktree |
| ch05 多智能体系统 | MultiAgent 长笔记 + Part 5 Agent Team | Anthropic 研究系统/C 编译器/MAST |
| ch06 记忆与状态 | Part 5 system of record + Part 3 不断线 | CLAUDE.md 体系/memory tool/checkpoint |
| ch07 验证评估与基准 | Part 4 交付闭环 + Part 5 反脆弱 | SWE-bench 家族/METR/LLM-as-judge |
| ch08 安全与沙箱 | Part 5 治理 | prompt injection/供应链/沙箱谱系 |
| ch09 模型与训练 | Part 1 模型演进 | RLVR/test-time compute/模型横评 |
| ch10 工程落地与规模化 | Part 2 定盘 + Part 3–5 全部工程主线 | Spec Kit/best practices/CI 集成/成本 |
| ch11 产品生态与品味 | Part 1 工具版图 + Part 5 human-on-the-loop | 市场数据/生态位/商业模式 |
| ch12 综合实战与场景 | 全书主线项目 GraphSpec 思路 | 大型 system design + 排错题集 |

---

## 二、章节目录(全库共 305 题)

- [ch01 认知与范式:从补全到 Agent](./ch01-认知与范式.md)(29 题)— workflow vs agent、三层控制面、开发模式演进
- [ch02 上下文工程与缓存(重点章)](./ch02-上下文工程与缓存.md)(37 题)— 稳定前缀、prompt caching、compaction、工具结果压缩、context rot、经济账
- [ch03 工具调用与 MCP](./ch03-工具调用与MCP.md)(26 题)— function calling、工具设计、MCP 协议、工具过载、code execution
- [ch04 Agent 架构与循环](./ch04-Agent架构与循环.md)(24 题)— agent loop、停止条件、权限、hooks、harness 分层、沙箱化执行
- [ch05 多智能体系统](./ch05-多智能体系统.md)(27 题)— 三问、编排形态、上下文隔离、worktree、失败分类、A2A
- [ch06 记忆与状态管理](./ch06-记忆与状态.md)(23 题)— CLAUDE.md/AGENTS.md、memory/state/artifact、长任务断点、semantic drift
- [ch07 验证、评估与基准](./ch07-验证评估与基准.md)(25 题)— SWE-bench 家族、pass@k、verifier 设计、LLM-as-judge、evals as CI
- [ch08 安全、沙箱与风险](./ch08-安全与沙箱.md)(22 题)— 间接注入、Slalom、权限系统、供应链、终端转义
- [ch09 模型与训练前沿](./ch09-模型与训练.md)(23 题)— RLVR、agentic RL 环境、test-time compute、模型选型横评
- [ch10 工程落地与规模化](./ch10-工程落地与规模化.md)(28 题)— 需求收敛、spec-driven、execution graph、成本治理、反脆弱、CI/CD
- [ch11 产品生态与品味](./ch11-产品生态与品味.md)(23 题)— Claude Code vs Codex、Cursor/Google/开源/国产、生态位与商业模式
- [ch12 综合实战与场景题](./ch12-综合实战与场景题.md)(18 题)— system design 大题、故障排错集、take-home 实操

---

## 三、按岗位的选题路径建议

| 面试场景 | 建议重点章节 | 建议题型 |
| --- | --- | --- |
| 校招/实习生筛选 | ch01 + ch02 八股(L1) + ch11 基础 | 八股为主,1–2 道开放观察潜力 |
| 应用工程师(重度 agent 使用者) | ch02 + ch04 + ch06 + ch10 | 八股 L2 + 实战 + 排错 |
| Agent 平台/infra 工程师 | ch02 + ch03 + ch04 + ch07 | 原理 L3–L4 + 实战设计 |
| 多智能体/编排方向 | ch05 + ch02 + ch06 | 原理 + 品味 + 实战 |
| 安全方向 | ch08 + ch03(MCP 安全) + ch04(权限) | 原理 + 排错 + 攻防场景 |
| 技术负责人/架构师 | ch10 + ch11 + ch12 | 品味 + 开放 + system design |
| 模型研究员 | ch09 + ch07 + ch02(注意力/缓存机制) | 原理 L4–L5 + 开放 |

## 四、出题原则

1. **能核验的给答案,不能核验的给判据。** 八股/原理题附参考答案要点与关键数字;品味/开放题只给考察点和优秀回答的判据,不给"标准答案"。
2. **数字尽量给来源语境。** 如"Anthropic 缓存读 0.1×""multi-agent token 消耗约 15×(Anthropic 研究系统)"——数字会过时,但推理链不会。
3. **每章自带难度与视角分布**,便于按面组装卷子。
4. **反面观点也入题。** 如"Multi-Agent 不值钱论(Cognition)""METR RCT:AI 让资深开发者慢 19%"——好的候选人应该能站在正反两面论证。
