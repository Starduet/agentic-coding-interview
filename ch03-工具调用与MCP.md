# ch03 · 工具调用与 MCP

> 覆盖:function calling 演进与机制、工具 schema 设计方法论(Anthropic《Writing tools for agents》)、工具过载与 tool_search、MCP 协议全解(机制/传输/原语/治理/安全)、CLI vs MCP vs Code Execution、结构化输出、采样参数。

---

## 3.1 八股题(基础概念)

**Q3.1【八股·L1·平台】OpenAI function calling 的关键时间线和机制?**
答:
- 2023.06:发布 function calling(模型输出结构化 JSON 参数);2023.11:parallel function calls 默认开启 + JSON mode;2024.08:Structured Outputs(constrained decoding,strict 模式 100% 符合 schema)。
- 现行 API 五步循环:带 tools 定义请求→模型返回 tool calls→执行→按 call_id 回传结果→模型继续或结束;`tool_choice`:auto / required / 强制指定 / none。
- 计费要点:**工具定义会被注入请求并按输入 token 计费**——工具是上下文的一部分。

**Q3.2【八股·L1·平台】Anthropic 的 tool use 机制与 OpenAI 有何不同?**
答:
- 请求带 tools(name/description/input_schema 为 JSON Schema);模型返回 `tool_use` content block(工具名+input+id);客户端执行后回传 `tool_result` block(按 tool_use_id 对应,可标 is_error)。
- tool_choice:auto/any/tool/none,可 disable_parallel_tool_use;`strict: true` 强制 schema 一致。
- 差异细节:stop_reason=tool_use 驱动循环;每个工具定义有固定 token 开销;缺参行为因模型而异(Opus 倾向追问,Sonnet 可能自行脑补)。

**Q3.3【八股·L2·平台】MCP 是什么?解决什么问题?**
答:
- Model Context Protocol,Anthropic 2024.11.25 开源,定位"连接 AI 系统与数据源的通用开放标准",被称"AI 的 USB-C"。
- 解决 **N×M 集成问题**:M 个应用 × N 个数据源原本要写 M×N 个连接器,标准化后变成 M+N。
- 基于 JSON-RPC 2.0;三概念:host(如 IDE)创建并管理多个 client,每个 client 与一个 server 1:1 连接;设计原则:**server 不能读整个对话、也看不见其他 server**(安全边界)。

**Q3.4【八股·L2·平台】MCP 的原语(primitives)有哪些?各是谁提供给谁的?**
答:
- Server 侧提供:**tools**(可执行动作)、**resources**(可读数据)、**prompts**(模板)。
- Client/host 侧提供:**sampling**(server 反向请求 host 做模型调用)、**roots**(暴露文件系统根)、**elicitation**(server 向用户追问信息)。
- 2025-06-18 规范新增:结构化工具输出(outputSchema)、elicitation、resource links;(2026-07 修订版转向无状态、sampling/roots 弃用——答出"协议仍在快速演进"即可加分)。

**Q3.5【八股·L2·平台】MCP 传输层和授权的演进?**
答:
- 初版:stdio + HTTP+SSE;2025-03-26:Streamable HTTP 取代 SSE、新增 OAuth 2.1 授权框架、工具注解(read-only/destructive);2025-06-18:server 归类为 OAuth Resource Server(protected resource metadata + RFC 8707,防恶意 server 骗 token)。
- 治理:2025.12 Anthropic 把 MCP 捐给 Linux Foundation 旗下 Agentic AI Foundation;官方 Registry 2025.09 上线(GitHub OAuth/OIDC 验证、命名空间管理)。
- 采纳时间线:OpenAI 2025.03 → Google 2025.04 → 微软 2025.05。

**Q3.6【八股·L1·用户】"code execution as action"是什么主张?为什么说 bash/python 是万能工具?**
答:
- Anthropic《Code execution with MCP》(2025):与其把上百个工具定义塞进上下文、让中间结果反复流经模型,不如**把 MCP server 当代码 API,让 agent 写程序调用它们**,数据与控制流留在执行环境里。
- 依据:LLM 写代码可以组合出任意工具行为;官方数字:工具定义从 15 万 token 降到 2 千(省 98.7%);1 万行表格在代码里过滤后模型只见 5 行;循环/条件/错误处理在代码里跑,不再是一轮轮模型往返。
- 代价:需要带资源限制与监控的安全沙箱。

**Q3.7【八股·L2·平台】JSON mode 和 Structured Outputs 的区别?为什么 agent 必须要后者级别的可靠性?**
答:
- JSON mode 只约束"输出合法 JSON",不保证符合 schema:复杂 schema 下 gpt-4-0613 常低于 40%,gpt-4o 靠提示工程也只有约 93%。
- Structured Outputs(OpenAI 2024.08):给 JSON Schema + strict:true,通过**约束解码**在采样阶段动态限制 token,100% 符合 schema;代价是 schema 限制(additionalProperties:false、全字段 required)、不能与并行函数调用同用、首个请求要编译 schema。
- agent 必须要:工具参数、路由分类、状态转移、评测解析全依赖机器可读输出;1% 的坏 JSON 在 agent 循环里被放大成整条任务失败。

**Q3.8【八股·L2·模型】temperature=0 就完全确定了吗?采样参数在 agent 里的正确用法?**
答:
- 不:批处理内核、浮点、路由、基础设施都会引入波动;OpenAI 的 seed 也只是 best-effort(配合 system_fingerprint 观测)。
- 用法:工具选择/抽取/编辑等"要对"的任务低温度;多候选生成高温。趋势:Anthropic 已在新模型上**弃用 temperature/top_p/top_k**(只接受 1.0)——可控性从采样参数转移到约束解码+工具设计+评测。

---

## 3.2 原理深挖

**Q3.9【原理·L3·平台】Anthropic《Writing tools for agents》(2025.09)的核心工具设计原则有哪些?**
答:
1. **宁少勿滥、按工作流合并**:一个 `schedule_event`(查空闲+创建)好过 list_users/list_events/create_event 三件套;`search_contacts` 优于 `list_contacts`(别让 agent 暴力遍历)。
2. description 像给初级工程师写 docstring;按 agent 的"可供性"设计,别直接包 API 端点。
3. **返回高信号数据**:把 UUID 解析成语义名显著降低幻觉;提供 response_format: detailed|concise(省约 2/3 token);错误信息带"正确示例"而不是裸 traceback;分页/过滤给合理默认值。
4. 命名空间化前缀(asana_projects_search);参数名无歧义(user_id 而非 user)。
5. 实证:仅改进工具描述就让 Sonnet 3.5 拿到当时 SWE-bench Verified SOTA;"我们优化工具花的时间比优化 prompt 还多"。

**Q3.10【原理·L3·平台】工具过载的量化证据和缓解手段?**
答:
- 证据:多 server 场景(GitHub+Slack+Sentry+Grafana+Splunk)工具定义未干活先吃约 55K token;**工具选择准确率在超过 30–50 个工具后退化**。
- 缓解:①tool_search/延迟加载(只放名称,命中后展开,可省 85%+ 定义 token,单次最多 1 万个延迟工具,常留 3–5 个核心工具不延迟);②核心工具保持少量(经验 ≤20)+ **subagent 分组**(独立上下文+工具白名单);③合并语义重叠工具。
- 与 ch02 联动:工具定义抖动是缓存最大杀手之一——工具集稳定既是质量问题也是成本问题。

**Q3.11【原理·L4·架构】MCP 的三大工程坑(n+1 鉴权 / 工具发现与上下文爆炸 / 安全)分别怎么缓解?**
答:
- **n+1 鉴权**:每个 server 一套 OAuth 流程,用户要对 n 个 server 分别授权。缓解:规范层的 resource server 化;产品层的 connectors/集中身份代理;企业侧统一网关+凭证托管。
- **工具发现与上下文爆炸**:发现靠 registry/目录,但定义全量进上下文。缓解:tool_search 延迟加载;按需附挂 server(会话中动态 attach);code execution 模式(定义变文件树,按需读)。
- **安全**:Invariant Labs 对约 3000 个 server 审计出 426 个警告——tool poisoning(描述里藏指令)、rug pull(上线后改描述)、cross-server shadowing、敏感信息泄漏、明文 API key。缓解:审计入库的 server 配置、工具描述 diff 监控、最小权限、外部内容当不可信输入。

**Q3.12【原理·L3·架构】CLI 工具 vs MCP server vs Code Execution,三者的选型框架?**
答(书 Part 1.1 的对比表 + Anthropic 观点):
- CLI:AI 擅长写命令、生态成熟、可 pipe 组合、错误处理简单——通用工具与自动化的默认起点(如 gh 优于 GitHub MCP 做仓库操作)。
- MCP:需要持久连接、结构化返回、权限管理(OAuth)、跨产品复用(同一 server 服务多个 host)时。
- Code Execution:多工具协调、大数据量中间结果、复杂控制流——一个执行环境的表达力≈无限个细粒度工具。
- 书的原则:**"一类能力,只认一个主入口"**;能力重叠时模型不是更聪明而是开始摇摆。

**Q3.13【原理·L3·模型】为什么"把 UUID 换成语义名能降低幻觉"?这背后是什么机制?**
答:
- 工具返回的密码学标识符(UUID、hash)对模型是"无语义噪声":模型要在上下文里维护 id→实体的映射,跨轮引用时容易错配或编造;
- 语义名(user_name、"Acme Corp invoice")把标识锚定在自然语言表征上,与模型训练分布一致;
- 这是"工具接口是给模型的用户界面"这一思想的典型案例——工具设计本质是 ACI(agent-computer interface)设计,与 HCI 同构。

---

## 3.3 品味与判断题

**Q3.14【品味·L3·平台】你的 agent 需要接入内部 30 个系统。有人提议"全部包成 MCP server 一次挂上",你怎么决策?**
参考思路:
- 反对全挂:工具过载(30–50 退化阈值)、定义 token 成本、缓存抖动、安全面扩大;
- 分层:高频核心(文件/命令/代码)用内置工具;按任务域拆 subagent,每个挂 3–10 个相关工具;低频长尾走 tool_search 延迟加载或 code execution(需要时现学 API);
- 先问:这个系统是"动作型"(MCP/CLI)还是"数据型"(检索/过滤→code execution 更优)?

**Q3.15【品味·L4·架构】"工具应该粗粒度(一个工具完成一个工作流)还是细粒度(正交原子操作)"——请给出你的判断函数。**
参考思路:
- 判断变量:被谁组合(模型组合→偏粗,代码组合→可细)、错误语义(中间态是否需要可见/回滚→需要则细)、上下文预算(每次调用的往返成本)、训练分布(模型对什么粒度的接口更熟)。
- 经验:对模型暴露粗粒度工作流工具,把原子操作留给代码执行去组合;但审计/合规要求可见中间态时,粗粒度是风险。
- 加分:引用 Anthropic 的合并案例与"response_format 枚举"作为粒度与信息量的正交解法。

**Q3.16【品味·L3·安全】MCP 生态里"第三方 server"的可信模型应该怎么建?**
参考思路:不可信(描述可投毒、可 rug pull)→凭证最小化(只读优先)、网络出口限制、工具描述变更 diff 告警、沙箱内执行、敏感 server 自建/审计;把"工具描述"当作可执行代码对待(它确实在指挥模型)。

**Q3.17【品味·L4·架构】如果明天 MCP 消失了,你账下哪些资产会归零,哪些会保留?这个答案如何影响你今天的架构?**
参考思路:归零:server 封装、协议适配层;保留:工具的语义设计(名称/描述/返回格式)、权限模型、审计日志、subagent 分组逻辑——它们是 ACI 资产,协议只是载体。启示:把聪明才智花在接口语义与治理上,而不是协议绑定上;为多协议(CLIs、OpenAPI、A2A)留一层适配。

---

## 3.4 开放性问题

**Q3.18【开放·L5·模型】"模型训练分布决定工具易用性"——如果让你为新一代模型重新发明工具调用协议,你会改什么?**
讨论框架:结构化输出的原生性(不经 JSON 文本层)、工具定义的渐进披露作为协议原语、错误语义的强类型、工具组合(call 融合/pipe)作为一等公民、权限声明内嵌 schema。

**Q3.19【开放·L4·架构】Code Execution 会不会最终吞掉 MCP?什么会幸存?**
讨论框架:Cloudflare "Code Mode" 与 Anthropic 结论趋同是证据;但 MCP 的价值在**发现、鉴权、跨 host 复用**——这些不是代码能替代的;可能的终局:MCP 做目录与授权层,执行层让代码完成。追问:沙箱供给成本(数十万并发沙箱)谁来解决。

**Q3.20【开放·L4·管理】工具/接口的投资在组织里该怎么归属——平台团队、业务团队还是模型厂商生态?**
讨论框架:内聚于领域知识(业务)vs 横向复用(平台);自建 vs 社区 MCP 的 TCO;"接口即资产"的组织含义(review、版本管理、弃用流程)。

---

## 3.5 工程实战题

**Q3.21【实战·L3·平台】为一个内部"发布系统"设计 agent 工具集。现状:REST API 40 个端点。给出你的工具清单(≤10 个)、每个工具的返回设计、以及不暴露的端点和原因。**
期望框架:
- 按工作流合并:list_releases(status filter)、get_release_health(id→语义化摘要而非原始 JSON)、create_release(dry_run 参数)、rollback_release(需 approval)、search_incidents 等;
- 返回设计:语义名替代内部 id;concise/detailed 枚举;错误带修复建议;分页默认值;
- 不暴露:原始 CRUD(模型误操作面)、级联删除类、凭证管理类——走人工/其他通道;
- 附:工具描述示例文案,体现"docstring 给初级工程师"的风格。

**Q3.22【实战·L3·平台】实现 tool_search 式延迟加载的最小可行版本(不依赖厂商特性)。**
期望框架:注册表只含 name+一句话摘要(常驻);提供一个 search_tools(query) 元工具返回候选并动态注入完整定义(注意:动态注入会破坏缓存——应注入在尾部消息或子代理上下文);命中后缓存定义供本会话复用;记录命中率/选择准确率评估效果。

**Q3.23【实战·L4·平台】设计 agent 调用不可靠外部 API 的工具封装:API 经常超时、返回半截 JSON、限流。**
期望框架:工具侧做重试/退避与超时预算,但**返回给模型的是稳定的语义错误**("服务暂不可用,已重试 2 次;可等待 60s 后重试或改用 X");半截数据要么截断标注要么不出;限流时返回建议的替代路径;绝不让模型看到原始 traceback(它修不了网络,只会瞎试);附带监控与熔断(连续失败时工具层面直接拒绝,提示 agent 改道)。

---

## 3.6 场景排错题

**Q3.24【排错·L2·用户】agent 总是"忘记"用你配置的某个工具,转而用 bash 硬凑。排查顺序?**
排查链路:工具描述是否清晰(名字像什么、何时该用);是否与其他工具语义重叠(模型摇摆);工具数量是否过载;工具是否真的加载(权限/白名单);返回示例是否让模型上一次用过吃亏(错误信息不可读);最后才是 prompt 里显式点名引导。

**Q3.25【排错·L3·平台】某 MCP server 更新后,agent 开始偶发把用户数据发到外部端点。应急处置与根因?**
排查链路:应急——立即摘除该 server、吊销凭证、审计泄漏范围;根因排查——对比更新前后的工具描述与实现(rug pull/tool poisoning)、检查 server 是否有 sampling 能力被滥用、出口网络是否未限制;长期——server 配置版本化入库、描述 diff 审计、沙箱网络白名单、敏感数据分级(见 ch08)。

**Q3.26【排错·L3·平台】agent 调用工具时参数频繁填错(枚举值拼错、id 传成 name)。是模型问题还是接口问题?**
排查链路:先看接口——枚举值是否语义化(内部码 vs 自然语言);参数名是否有歧义;是否该用 strict schema 约束解码;返回值里是否从未给出正确的 id 形态(模型只能瞎猜);id→语义名映射是否在返回中提供。多数"模型笨"最终都落在接口没按 ACI 原则设计。
