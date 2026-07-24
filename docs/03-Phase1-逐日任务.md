# Phase 1:撕开黑箱 —— 从零写最小 agent

> 周期:W1-4(28 天)。语言:Python。目录:`/home/xinghe/cursor-project/pico-agent/`(独立 git repo)。
> 本文档在 `pi/docs/`。本阶段 AI 使用规则:**核心循环闭卷手写**。Claude Code 只能问概念/解释报错,不能生成 agent 代码。

## 目标与毕业标准

**认知目标**:建立完整的 agent 运行心智模型,过完这关,任何 agent 代码对你不再是黑箱。

**毕业测试(必须全过才进 Phase 2)**:
1. **闭卷默写**:从空白文件开始,写出 tool-calling 循环骨架(调模型 → 解析 tool_call → 执行 → 喂回 → 判停)。
2. **口述 3 个坑**:说出 agent loop 最容易出的 3 种 bug(提示:死循环、工具异常、状态不一致)。
3. **能解释**:token / context window / temperature 各自怎么影响你的 agent。

## 每周结构

- W1:单轮对话(打地基 + 知识补课)
- W2:**手写核心循环(最关键的一周)**
- W3:流式 + 错误处理(体会状态不一致)
- W4:加真实工具,agent 第一次干活

---

## W1 · 单轮对话与 API 基础

**本周目标**:能用裸 HTTP 调通 LLM API,搞懂 token 计费。

### Day 1 · 环境 + 第一次调用
- [ ] 在 `pico-agent/` 建 Python 项目(`uv` 或 venv),装 `httpx`。
- [ ] 选一个 provider,读它的 **API 原始文档**(不读 SDK 文档)。建议 OpenAI 兼容接口。
- [ ] **闭卷**:用 `httpx.post` 发一个最简请求,拿回一段文字。
- [ ] **读**:provider 文档里关于 `messages` / `role` / `max_tokens` 的章节。
- **今日产出**:`pico-agent/day1_hello.py`,一个能跑的"调用 LLM"脚本。

### Day 2 · 搞懂 token
- [ ] 找一个在线 tokenizer,把你的 prompt 和回复贴进去看 token 数。
- [ ] 在代码里打印出这次调用花了多少 token、多少钱。
- [ ] **知识检查点**:见 `02-知识检查点.md` 的 Token 部分。
- **今日产出**:代码里加 token/成本统计函数。

### Day 3 · system prompt 与 role
- [ ] 实现 `system` + `user` 多 message 的拼装。
- [ ] 实验:同一个问题,加/不加 system prompt,回答差异。
- [ ] **写笔记**:system prompt 到底在控制什么?(答:角色/边界/输出格式)
- **今日产出**:支持多 message 的调用封装。

### Day 4 · 单轮但带"假工具"
- [ ] **不接真 API**,在 prompt 里描述一个工具,让模型用文本"假装"调用(如输出 `CALL: read_file path=...`)。
- [ ] 自己解析这段文本,模拟执行,把结果拼回 prompt。
- [ ] **思考**:这就是没有 function calling 标准时,早期 agent 怎么做的。为什么 OpenAI 要发明 function calling?
- **今日产出**:一个纯文本约定的"伪 tool calling"demo。

### Day 5 · W1 复盘 + 知识检查
- [ ] 过一遍 `02-知识检查点.md` 的 Token / Sampling 部分。
- [ ] 写 W1 笔记:我学到的 3 件事 + 1 个还没搞懂的。
- [ ] **休息或补前面的卡点**。

---

## W2 · 手写核心循环(最关键的一周)

**本周原则:核心循环闭卷。卡住了翻 Day 6 的设计草图,不许让 AI 写。**

### Day 6 · 先画图,再写代码
- [ ] **不写代码**。在纸上或 `docs/` 画 agent loop 的状态机:哪些状态?哪些转移?终止条件是什么?
- [ ] 写伪代码(中文/英文都行),说清楚每一步。
- [ ] **设计草图参考**(卡住时才看):
  ```
  messages = [system, user]
  loop:
      response = call_llm(messages)
      if response 没有 tool_calls:
          print(response.text); break
      for tool_call in response.tool_calls:
          result = 执行(tool_call)
          messages.append(tool_call)        # 模型的调用记录
          messages.append(tool 角色的结果)  # 执行结果
      # 继续 loop,让模型看到结果后决定下一步
  ```
- **今日产出**:`docs/notebook/` 里的状态机图 + 伪代码。

### Day 7 · 写出循环骨架
- [ ] 基于 Day 6 的设计,**闭卷**写出循环。允许查 API 文档,不允许 AI 生成代码。
- [ ] 只实现"单工具"场景(如只有一个 `echo` 工具)。
- [ ] 跑通一个完整循环:user → 模型决定调 echo → 执行 → 喂回 → 模型给出最终回答。
- **今日产出**:`pico-agent/agent_loop.py`,能跑通单工具循环。

### Day 8 · 多工具 + tool 注册
- [ ] 把工具做成"注册表":`tools = {"echo": echo_fn, "add": add_fn}`。
- [ ] 把工具的 JSON schema 也注册进去,生成给模型的 `tools` 参数。
- [ ] 实验:让模型在 `echo` / `add` 之间选择。
- **今日产出**:可插拔的工具注册机制。

### Day 9 · 判停逻辑与边界
- [ ] 处理:模型不返回 tool_call 也不返回 text(空响应)。
- [ ] 处理:模型返回多个 tool_call。
- [ ] 加最大循环次数(如 10 次),防止死循环。
- [ ] **思考**:为什么需要最大循环次数?什么情况下模型会无限调工具?
- **今日产出**:健壮的判停 + 安全阀。

### Day 10 · 触发第一个真 bug
- [ ] 故意写一个会抛异常的工具,观察 agent 怎么挂掉。
- [ ] 故意让模型陷入循环(如工具永远返回"请再试一次"),观察安全阀是否生效。
- [ ] **写笔记**:今天我制造了哪两个 bug,它们的根因是什么?
- **今日产出**:`docs/notebook/` 新增"我踩过的坑"章节。

### Day 11-12 · W2 复盘 + 毕业测试预演
- [ ] Day 11:闭卷默写一遍 agent loop(不看任何资料)。对照 Day 7,找差距。
- [ ] Day 12:对着空气/录音,讲一遍 agent loop,并说出 3 个坑。录下来,自己听一遍。
- **本周产出**:你能不看代码讲清楚 agent loop。这是 Phase 1 的核心交付。

---

## W3 · 流式输出与错误处理

**本周目标**:理解"状态一致性"为什么是 agent 的头号难题。

### Day 13 · 流式输出
- [ ] 把 LLM 调用改成流式(SSE)。
- [ ] 边收边打印 token,体会"增量"。
- [ ] **知识检查点**:见 Streaming 部分。
- **今日产出**:流式版的单轮对话。

### Day 14 · 流式 + 工具调用(难)
- [ ] 流式时,tool_call 是分片到达的,需要攒齐再执行。
- [ ] 这是 Phase 1 最难的一天。卡住可以问 AI 解释原理,但代码自己写。
- **今日产出**:流式版 agent loop。

### Day 15 · 错误重试与超时
- [ ] 加请求超时、网络错误重试(带退避)。
- [ ] 思考:重试时要保证幂等吗?哪些操作能重试,哪些不能?
- **今日产出**:带重试的调用层。

### Day 16 · 工具执行异常处理
- [ ] 工具抛异常时,不要崩 agent,把错误信息喂回模型,让它自己决定怎么办。
- [ ] 实验:模型看到"工具报错"后,会调整策略吗?
- **今日产出**:容错的工具执行层。

### Day 17 · W3 复盘
- [ ] 写笔记:流式为什么会让状态一致性变难?
- [ ] 过 `02-知识检查点.md` 的 Streaming 部分。

---

## W4 · 真实工具 + 整合

**本周目标**:agent 第一次真能干活。

### Day 18 · 工具:读文件
- [ ] 实现 `read_file(path)` 工具,带路径校验(只允许读 `pico-agent/` 下的文件,防越权)。
- [ ] 让 agent 回答"这个文件里写了什么"。
- **今日产出**:文件读取工具。

### Day 19 · 工具:运行命令(沙箱)
- [ ] 实现 `run_command(cmd)` 工具。**强烈建议**只用白名单命令(如 `ls`, `cat`, `wc`),或限制目录。
- [ ] 思考:为什么 coding agent 的权限模型是核心安全问题?
- **今日产出**:受限的命令执行工具。

### Day 20 · 工具:grep / 搜索
- [ ] 实现 `search(pattern, path)` 工具。
- [ ] 给 agent 一个任务:"在 pico-agent/ 里找出所有 def 定义"。
- **今日产出**:搜索工具。

### Day 21 · 整合:让 agent 完成一个真实任务
- [ ] 综合三个工具,让 agent 完成一个多步任务,例如:"读 pyproject.toml,统计代码行数,告诉我项目结构"。
- [ ] 观察 agent 的规划行为:它会一步步来,还是一次调多个工具?
- **今日产出**:一个能完成小任务的完整 agent。

### Day 22-23 · 正式毕业测试
- [ ] **闭卷默写**(限时 1 小时):从空白文件写出 tool-calling 循环。
- [ ] **口述录频**:讲清 agent loop + 3 个坑 + token/temperature/context。
- [ ] 把成果整理到 `pico-agent/README.md` 和 `docs/notebook/`(Phase 1 笔记)。

### Day 24-28 · 缓冲 / 补卡 / 深化
- 实际学习会有卡顿,留 5 天缓冲。
- 用不完的时间:读 ReAct 论文,对照你的实现看它怎么描述这个循环。

---

## Phase 1 产出清单

- [ ] `pico-agent/`:~400 行的 agent,支持多工具、流式、错误处理。
- [ ] `docs/notebook/`:状态机图、踩坑记录、知识笔记(按编号命名)。
- [ ] 闭卷默写通过 + 口述录频。
- [ ] `pico-agent/README.md`:用你自己的话说清这个 agent 怎么工作。

## 进入 Phase 2 的检查点

- [ ] 闭卷默写 tool-calling 循环(通过)。
- [ ] 能口述 3 个 agent loop 的坑。
- [ ] 能解释 token / context window / temperature / function calling 的作用。
- [ ] `pico-agent/` 跑得起来,能完成一个多步任务。

全绿 = 进 Phase 2。
