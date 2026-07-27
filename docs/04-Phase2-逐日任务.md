# Phase 2:精读 pi 源码 —— 用生产级实现校准判断力

> 周期:W5-8(28 天,Day 29-56)。语言:读 TS(只读不写)。教材:`pi/packages/`(本仓库)。对照项目:`/home/xinghe/cursor-project/pico-agent/`(你 Phase 1 写的 Python agent)。
> 本文档在 `pi/docs/`。本阶段 AI 使用规则:**源码自己读、笔记自己写,AI 只能解释概念/答报错,结论自己下**(见 `01-README.md` 第 13 行)。

## 为什么这一阶段存在

Phase 1 让你建立了 agent loop 的心智模型——但那是"一个人的最小实现"。Phase 2 的目的是:**把你的模型和生产级实现对撞,发现你没想到的设计选择,校准你的判断力**。

不是"学 pi 怎么写代码",是"学 pi 为什么这么设计,我当初为什么没那么设计,谁对"。

读完不要求你能复刻 pi(那是 TS,你不写),要求你**面对任何 agent 代码能快速定位:它的 loop 在哪、工具协议长啥样、上下文怎么管、哪里会埋雷**。

## 两个包的定位(先建立地图)

| 包 | npm 名 | 角色 | 本阶段定位 |
|---|---|---|---|
| `packages/agent` | `@earendil-works/pi-agent-core` | **内核**:loop / 类型 / harness / 流式抽象 | **精读起点**,W5 主战场 |
| `packages/coding-agent` | `@earendil-works/pi-coding-agent` | **应用层**:具体工具 / TUI / CLI / session / compaction | W6-W8,依赖 agent 内核 |

依赖方向:`coding-agent` → `agent` → `ai`(LLM 抽象)。loop 只有一份,在 `packages/agent/src/agent-loop.ts`。**先把内核吃透,应用层才读得懂**。

## 目标与毕业标准

**认知目标**:建立"读任意 agent 代码"的能力。过完这关,看任何 agent 框架(LangGraph / AutoGen / 你的副业竞品)都能 30 分钟内说清它的 loop、工具协议、上下文策略。

**毕业测试(必须全过才进 Phase 3)**:
1. **对照报告**:针对 agent loop、工具协议、流式、上下文管理四个主题,各写一段"pi 怎么做 vs pico-agent 怎么做 vs 谁更好为什么"。不是抄代码,是判断。
2. **白板讲解**:不看源码,画出 pi 的 `runLoop` 双层循环状态机(外层 follow-up / 内层 tool-call + steering),说出 3 个 pi 处理了而你没处理的边界。
3. **坑位清单**:列出 pi 源码里 5 个"生产级才会遇到的坑"(提示:并发工具执行、流式分片、压缩触发点、权限分层、热更新 system prompt),说清 pi 怎么解的。
4. **迁移判断**:针对你副业可能做的 RAG agent,判断 pi 的哪些设计该抄、哪些该丢、哪些要改。

## 每周结构

- **W5(Day 29-35)**:内核三件套——类型协议 / agent loop / Agent 状态机。
- **W6(Day 36-42)**:工具体系与权限——AgentTool 协议 / 7 个内置工具 / 三层权限。
- **W7(Day 43-49)**:流式与上下文——stream 抽象 / SSE proxy / compaction / system prompt。
- **W8(Day 50-56)**:应用层编排与毕业——AgentSession 编排 / TUI 事件衔接 / 对照报告。

---

## W5 · 内核三件套(理解 loop 的所有边界)

**本周目标**:把 `packages/agent` 的内核读穿。读完你能默写 pi 的 loop 结构(不是抄,是理解后复述)。

**阅读顺序固定,不要跳**:`types.ts`(协议)→ `agent-loop.ts`(无状态 loop)→ `agent.ts`(有状态 Agent)。先看协议,否则 loop 里每个字段都要回头查。

### Day 29 · 建立地图:README + types 入口
- [ ] 读 `packages/agent/README.md` 全文(508 行)。重点:Core Concepts、Message Flow 图、Event Flow 时序图。
- [ ] 读 `packages/agent/src/index.ts`(49 行),看清这个包对外暴露什么、不暴露什么。
- [ ] 读 `packages/agent/src/types.ts` 的 `AgentState`(L327-352)和 `AgentContext`(L406-413),对照 `01-README.md` 里的状态机草图。
- **判断题**:pi 的 `AgentState` 比 pico-agent 的状态多了哪些字段?每个多出来的字段解决了什么问题?(提示:`isStreaming` / `streamingMessage` / `pendingToolCalls` / `errorMessage`)
- **今日产出**:`docs/notebook/` 里一张"pi 内核模块依赖图"(手画或文字)。

### Day 30 · 类型协议:消息与事件
- [ ] 读 `types.ts` 的消息类型(`AgentMessage` 及相关,L53 附近的 `AgentToolCall`)和事件类型(`AgentEvent`)。
- [ ] 读 `harness/messages.ts` 的 `convertToLlm`(L120-164),理解"内部消息 vs LLM 消息"为什么要分两层。
- **对照**:pico-agent 里消息和 LLM 消息是同一个结构吗?pi 为什么要拆?
- **判断题**:pi 把工具调用的"模型记录"和"执行结果"分别用什么 role 存?和 OpenAI 标准一致吗?
- **今日产出**:笔记"pi 的消息分层 vs 我的实现"。

### Day 31 · 核心循环:runLoop 双层结构(最难的一天)
- [ ] 读 `packages/agent/src/agent-loop.ts` 的 `runLoop`(L155-275)。**分段读**:
  - L170-205:内层 while(处理 tool calls + steering)。
  - L193:流式拉取 assistant 响应。
  - L203-222:处理 toolCalls。
  - L232-257:`prepareNextTurn` / `shouldStopAfterTurn` 钩子。
  - L170-272:外层 while(处理 follow-up 队列)。
- [ ] 对照 Day 6(Phase 1)你画的状态机:pi 多了哪一层?为什么?
- **判断题**:pi 为什么需要双层循环?pico-agent 的单层循环在什么场景下会出问题?(提示:用户在 agent 执行中途插话 = steering)
- **今日产出**:更新状态机图,标出 pi 的内外两层循环和终止条件。

### Day 32 · 工具调度:顺序 vs 并行
- [ ] 读 `agent-loop.ts` 的 `executeToolCalls`(L411-426)→ 分发到 `executeToolCallsSequential`(L433-487)和 `executeToolCallsParallel`(L489-554)。
- [ ] 读 `prepareToolCall`(L600-664)、`executePreparedToolCall`(L666-707)、`finalizeExecutedToolCall`(L709-754)。
- **判断题**:pi 怎么决定一个工具走顺序还是并行?看哪个字段?(提示:`AgentTool.executionMode`,types.ts L380-403)
- **对照**:pico-agent 的工具是并行还是串行?pi 的并行执行对 agent 行为有什么影响?
- **今日产出**:笔记"工具调度的三种模式(顺序/并行/流式 partial)"。

### Day 33 · Agent 状态机:有状态包装
- [ ] 读 `packages/agent/src/agent.ts` 的 `Agent` 类(L166 起)、`createMutableAgentState`(L67-94)、`processEvents`(L529-576)。
- [ ] 读消息队列:`PendingMessageQueue`(L123-157)、steering(L276-278)/ followUp(L281-283)、`QueueMode`("all"|"one-at-a-time")。
- **判断题**:`agentLoop`(无状态函数)和 `Agent`(有状态类)为什么要分开?pico-agent 是怎么混在一起的?
- **今日产出**:笔记"无状态 loop + 有状态 Agent 的分层动机"。

### Day 34-35 · W5 复盘 + 内核默写
- [ ] Day 34:闭卷默写 pi 的 `runLoop` 结构(双层 while + 钩子点),对照源码找差距。**允许默写不完整,但要标出"我漏了哪个钩子"**。
- [ ] Day 35:对着 `02-知识检查点.md` 的 Function Calling / Streaming 部分,用 pi 的实现回答每个概念题。
- **本周产出**:`docs/notebook/` 的 W5 笔记,含状态机图、内核默写、对照 pico-agent 的差距清单。

---

## W6 · 工具体系与权限模型

**本周目标**:理解"工具如何从定义变成 agent 能力",以及"生产级 agent 怎么管权限"。读完你能判断任何 coding agent 的工具/权限设计好坏。

### Day 36 · AgentTool 协议(回到内核)
- [ ] 重读 `packages/agent/src/types.ts` 的 `AgentTool`(L380-403)、`AgentToolResult`(L355-369)。
- [ ] 读 `harness/agent-harness.ts` 的工具管理(L185-215,`Map<string, TTool>` + `activeToolNames` 子集 L211-214)。
- **判断题**:pi 的工具是"注册中心模式"还是"数据数组模式"?(答:数据,放 `context.tools` 数组,按 name 查找)。为什么 pi 不用注册中心?
- **对照**:pico-agent Day 8 的 `tools = {"echo": echo_fn}` 注册表,和 pi 哪个更灵活?
- **今日产出**:笔记"工具作为数据 vs 工具作为注册项"。

### Day 37 · schema 与参数校验
- [ ] 读 `agent-loop.ts` 的 `prepareToolCallArguments`(L586-598)→ 调用 `validateToolArguments`(来自 `pi-ai`,`packages/ai/src/utils/validation.ts:278`)。
- [ ] 读任一内置工具的 schema 定义,如 `packages/coding-agent/src/core/tools/read.ts` L20 的 `readSchema`(typebox `Type.Object`)。
- **判断题**:pi 用 typebox 而不是 JSON Schema 原生字符串,图什么?(提示:类型推导 `Static<typeof schema>` + 跨 provider 转换)
- **今日产出**:把 pico-agent 的工具 schema 和 pi 的对照,写差异清单。

### Day 38 · 7 个内置工具巡礼
- [ ] `packages/coding-agent/src/core/tools/`:每个工具一个文件(`read`/`bash`/`edit`/`write`/`grep`/`find`/`ls`)。每个工具读 schema + execute 主体,不抠细节。
- [ ] 读 `tools/index.ts`(L83 `ToolName` 联合类型,L96-186 工厂函数 `createAllTools`/`createCodingTools`)。
- **判断题**:为什么 `read`/`grep`/`find`/`ls` 是只读工具,`bash`/`edit`/`write` 是 mutating?这个区分在权限层有用吗?(提示:W6 后半段)
- **今日产出**:表格"7 个工具的 schema 关键字段 + 执行副作用"。

### Day 39 · 权限第一层:项目信任(Trust)
- [ ] 读 `packages/coding-agent/src/core/trust-manager.ts`(L22 `trusted`、L179 注释)、`core/project-trust.ts`、`cli/project-trust.ts`。
- [ ] 读 CLI 参数:`cli/args.ts` L180/L274 的 `--approve/-a`、`--no-approve`。
- **判断题**:Trust 是"加载期准入"还是"执行期审批"?未信任目录会发生什么?(提示:不加载 `.pi` 下的扩展/技能)
- **对照**:pico-agent 有任何权限层吗?pi 的 Trust 解决了你 Phase 1 没考虑的什么风险?
- **今日产出**:笔记"权限三层模型之 Trust"。

### Day 40 · 权限第二、三层:工具白名单 + Extension hooks
- [ ] 读 `agent-session.ts` 的 `allowedToolNames`/`excludedToolNames`(L209-211、L348/L384、L2459-2530 过滤)。
- [ ] 读 extension 拦截:`agent-session.ts` L467 `beforeToolCall`、L488 `afterToolCall` → `extensions/runner.ts` L915 `emitToolCall()` / L866 `emitToolResult()`。
- [ ] 读 `extensions/types.ts` L133 的 `ui.confirm()`(扩展可调用的确认对话框)。
- **判断题**:pi 没有传统 sandbox/逐操作审批模块(grep 全包 `permission` 仅命中 MIT 文本),它靠什么做到安全?(答:Trust + 白名单 + Extension hooks 三层叠加)。这种设计的取舍是什么?
- **今日产出**:笔记"权限三层模型完整版"+ 一段判断"为什么 pi 不做 sandbox"。

### Day 41 · bash 工具深读:为什么没有沙箱也能用
- [ ] 读 `core/bash-executor.ts`(直接 spawn)+ `tools/bash.ts` 的 `BashOperations` 接口(可替换后端,如 SSH)。
- [ ] 读 `bun/restore-sandbox-env.ts`(仅环境变量还原,非权限校验)。
- **判断题**:pi 把 bash 执行做成可替换的 `BashOperations` 接口,图什么?(提示:扩展可换成 SSH/容器后端)
- **今日产出**:笔记"可替换执行后端 = 权限的另一种解法"。

### Day 42 · W6 复盘
- [ ] 写笔记:pi 的工具协议比 pico-agent 多了哪三个关键设计?(候选:executionMode / prepareArguments / 流式 partial result)。
- [ ] 写笔记:权限三层模型各自防什么?哪一层最重要?
- **本周产出**:工具协议对照表 + 权限三层模型笔记。

---

## W7 · 流式与上下文管理(agent 的两个硬骨头)

**本周目标**:理解"状态一致性"和"上下文爆炸"这两个生产级 agent 的头号难题,pi 怎么解的。这是 Phase 1 W3 的深化。

### Day 43 · 流式抽象:StreamFn
- [ ] 读 `packages/agent/src/types.ts` L28-32 的 `StreamFn` 类型(返回 `AssistantMessageEventStream`)。
- [ ] 读 `stream-fn.ts` 全文(20 行):`setDefaultStreamFn`/`getDefaultStreamFn`(全局默认,宿主注入)。
- **判断题**:pi 把 streamFn 做成"宿主注入的全局默认",而不是写死在 loop 里,图什么?(提示:不同运行环境——CLI/RPC/proxy——用不同实现)
- **对照**:pico-agent 的流式是写死的还是可注入的?
- **今日产出**:笔记"流式作为可注入策略"。

### Day 44 · 流式消费:事件攒齐
- [ ] 读 `agent-loop.ts` 的 `streamAssistantResponse`(L281-372,`for await` 消费 `start`/`text_delta`/`toolcall_delta`/`done`)。
- **判断题**:流式时 tool_call 是分片到达的,pi 在哪里攒齐?(提示:L317-361 的事件循环)。攒齐前如果网络断了,状态会怎样?
- **对照 Phase 1 Day 14**:你当时怎么攒分片的?pi 的做法比你高明在哪?
- **今日产出**:笔记"流式 tool_call 攒齐的两种实现对照"。

### Day 45 · SSE Proxy:省带宽的设计
- [ ] 读 `packages/agent/src/proxy.ts`:
  - L36-57:`ProxyAssistantMessageEvent`(`text_delta`/`thinking_delta`/`toolcall_delta`,服务器剥离 `partial` 字段)。
  - L191-213:SSE 解析(`fetch` + `buffer.split("\n")` + `line.startsWith("data: ")`)。
  - L250-333:客户端重建 partial 消息。
- **判断题**:为什么服务器要剥离 `partial` 字段?客户端为什么要重建?(提示:带宽 vs 计算的取舍)
- **今日产出**:笔记"proxy 模式:服务器精简 + 客户端重建"。

### Day 46 · 上下文压缩:触发与算法
- [ ] 读 `packages/coding-agent/src/core/compaction/compaction.ts`(29KB):
  - `shouldCompact()`(L235,按 `contextTokens/contextWindow` 阈值)。
  - `estimateTokens()`(L266,逐消息估 token)。
  - `compact()` 主流程、`calculateContextTokens()`、`findCutPoint()`。
- [ ] 读触发点:`agent-session.ts` L643(在 `agent_end` 检查自动压缩)。
- **判断题**:pi 用"阈值触发"还是"每次压缩"?阈值大概多少?为什么不是 100%?(提示:留余量给回复)
- **对照**:pico-agent 有任何压缩吗?pi 的 `findCutPoint` 比朴素截断高明在哪?
- **今日产出**:笔记"compaction 触发条件 + cut point 选择策略"。

### Day 47 · 分支摘要与压缩代价
- [ ] 读 `compaction/branch-summarization.ts`(12KB)。
- [ ] 读 `packages/agent/src/harness/compaction/`(branch-summarization / compaction / utils)对照看内核层和应用层的分工。
- **判断题**:压缩的代价是什么?pi 怎么减少信息损失?(提示:分支摘要保留关键决策点)
- **今日产出**:笔记"压缩的信息损失与补救"。

### Day 48 · system prompt 拼装与热更新
- [ ] 读 `packages/coding-agent/src/core/system-prompt.ts` 的 `buildSystemPrompt()`(L40):组装工具列表 / guidelines / `<project_context>` / skills / cwd。
- [ ] 读资源发现:`core/resource-loader.ts` L282 `getSystemPrompt()`、L475 `discoverSystemPromptFile()`(读 `.pi` 下的 AGENTS.md 等)。
- [ ] 读热更新机制:`agent-session.ts` L524 `_installAgentNextTurnRefresh()`(每轮刷新 systemPrompt/tools/model)。
- **判断题**:为什么 system prompt 要每轮热更新,而不是启动时定死?(提示:工具列表/skills 可能动态变)
- **对照**:pico-agent 的 system prompt 是定死的吗?热更新解决了什么?
- **今日产出**:笔记"system prompt 的动态拼装"。

### Day 49 · W7 复盘
- [ ] 写笔记:流式 + 压缩这两个难题,pi 的解法各打几分(1-5),为什么。
- [ ] 写笔记:如果让 pico-agent 加上 compaction,你会怎么设计(借鉴 pi 的哪部分,丢哪部分)。
- **本周产出**:流式 + 上下文管理的完整对照笔记。

---

## W8 · 应用层编排与毕业(把前 7 天串起来)

**本周目标**:看懂 pi 怎么把内核 + 工具 + 流式 + 上下文 + TUI 编排成一个完整产品。读完你能讲清"一个 prompt 从用户输入到屏幕渲染,在 pi 内部走了哪些层"。

### Day 50 · AgentSession:编排核心
- [ ] 读 `packages/coding-agent/src/core/agent-session.ts` 的骨架(顶部注释 L1-13 + 类结构)。**不要逐行读 110KB,抓主线**:
  - L18-26:从 `pi-agent-core` 导入 `Agent`/`AgentEvent`/`AgentState`。
  - L391:`this.agent.subscribe(this._handleAgentEvent)`(订阅内核事件)。
  - L593 `_handleAgentEvent` → L620 `this._emit(...)`(转成 `AgentSessionEvent` 给上层)。
  - L466 `_installAgentToolHooks()` / L518 `_installAgentNextTurnRefresh()`。
  - L1062 `await this.agent.prompt(messages)`(触发 loop)。
- **判断题**:AgentSession 在 Agent(内核)和 TUI 之间扮演什么角色?为什么需要这一层?(提示:内核不懂 session/compaction/TUI,需要一个适配层)
- **今日产出**:笔记"AgentSession = 内核适配层"。

### Day 51 · session 持久化
- [ ] 读 `core/session-manager.ts`(L639 `appendMessage` 落盘 JSONL、L231 v1→v2 / L260 v2→v3 迁移)。
- **判断题**:为什么 session 用 JSONL 而不是 JSON/SQLite?(提示:增量写 / 人类可读 / 迁移简单)
- **今日产出**:笔记"session 格式选型的取舍"。

### Day 52 · TUI 事件衔接
- [ ] 读 `src/modes/interactive/interactive-mode.ts` 的**事件订阅部分**(L376 `unsubscribe = session.subscribe(handler)`,不读全部 205KB):
  - L2896 `message_start`:新建 `AssistantMessageComponent`。
  - L2927 `message_update`:`streamingComponent.updateContent()` + toolCall 增量建 `ToolExecutionComponent`。
  - L2955 `message_end`:定稿 / abort / error。
- [ ] 读渲染后端:`packages/tui/src/tui.ts` L2 注释("差分重绘")+ `this.ui.requestRender()`。
- **判断题**:pi 自研 TUI(不用 ink/React),取舍是什么?(提示:依赖体积 / 二进制打包 / 控制力)
- **今日产出**:笔记"事件流:内核 → AgentSession → TUI 组件"。

### Day 53 · 全链路走查
- [ ] 选一个具体场景(如"用户输入 prompt → 模型调 read 工具 → 返回结果 → 模型最终回答"),从 TUI 输入框开始,逆推整条链路经过的每一层文件和函数。写在笔记里。
- **今日产出**:一张完整的"pi 全链路调用图"(文字版即可)。

### Day 54-55 · 对照报告起草
- [ ] Day 54:起草四个主题的对照(agent loop / 工具协议 / 流式 / 上下文管理),每个主题"pi 怎么做 vs pico-agent 怎么做 vs 谁更好为什么"。
- [ ] Day 55:起草"5 个生产级坑位"清单 + "副业 RAG agent 的迁移判断"。
- **今日产出**:毕业报告草稿(`docs/notebook/` 里)。

### Day 56 · 毕业测试
- [ ] **白板讲解**(录频):不看源码,画 pi 的 `runLoop` 双层循环,讲清 agent loop / 工具协议 / 流式 / 上下文四个主题,说出 3 个 pi 处理了而 pico-agent 没处理的边界 + 5 个生产级坑位。
- [ ] 把对照报告整理到 `docs/notebook/` 和 `pico-agent/README.md`(用你自己的话,不是抄源码)。
- **本周产出**:毕业报告 + 录频。

---

## Phase 2 产出清单

- [ ] `docs/notebook/`:W5-W8 笔记,含状态机图、对照表、坑位清单、全链路调用图。
- [ ] 对照报告:四个主题(pi vs pico-agent vs 判断)。
- [ ] 白板讲解录频(runLoop 双层循环 + 四主题 + 3 边界 + 5 坑)。
- [ ] `pico-agent/README.md` 更新:用校准后的判断重述你的 agent 设计。

## 进入 Phase 3 的检查点

- [ ] 能默写 pi 的 `runLoop` 双层循环结构(通过)。
- [ ] 能口述 pi 工具协议的 3 个关键设计(executionMode / prepareArguments / 流式 partial)。
- [ ] 能解释 pi 的权限三层模型各自防什么。
- [ ] 能讲清 compaction 的触发条件 + cut point 策略。
- [ ] 对照报告完成,且每个主题都有明确判断(不是"都挺好")。
- [ ] 能针对副业 RAG agent 场景,说出 pi 的哪些设计该抄 / 该丢 / 该改。

全绿 = 进 Phase 3(RAG 与检索工程,回到 Python)。

---

## 精读纪律提醒

1. **只读不写 TS**(技术栈约定,`01-README.md` 第 44 行)。判断力靠读 + 对照,不靠复刻。
2. **每天先读 pico-agent 对应部分,再读 pi**。带着自己的实现去对撞,否则会被 pi 带着走,失去判断。
3. **每个"判断题"必须写答案**,哪怕答案是"我还没想清楚"。没想清楚的就是盲区,记下来。
4. **不跳阶段**。Phase 1 毕业测试不过不要进 Phase 2(本蓝图先写着,执行等 Phase 1 全绿)。
5. **源码行号会随版本变**。本文档行号基于 v0.81.1(commit `20be4b18`),升级后以函数名为准重新定位。
