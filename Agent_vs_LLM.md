# LLM 与 Agent 架构问答整理

> 本文整理了关于 Agent 架构、LLM、规划/记忆/工具模块的一系列问答，适合面试准备与架构理解。

---

## 目录

- [一、LLM vs Agent：基础区别](#一llm-vs-agent基础区别)
- [二、Agent 完整请求处理流程](#二agent-完整请求处理流程)
- [三、Agent 四模块描述是否正确？](#三agent-四模块描述是否正确)
- [四、LLM 和规划模块的本质区别](#四llm-和规划模块的本质区别)
- [五、规划模块和 LLM 的关系：包含还是调用？](#五规划模块和-llm-的关系包含还是调用)
- [六、规划模块是大模型吗？LLM 等于大模型吗？](#六规划模块是大模型吗llm-等于大模型吗)
- [七、普通代码会不会"死板"？能否针对不同任务做不同规划？](#七普通代码会不会死板能否针对不同任务做不同规划)

---

## 一、LLM vs Agent：基础区别

### 面试场景

面试官常用问法：
- "你们项目里用的是 LLM 直接调用还是 Agent？为什么这么选？"
- "Agent 比 LLM 多了什么？"
- "如果让你从零设计一个 Agent，你会怎么做？"

### 1. LLM 是什么？

LLM（大语言模型）本质上就是一个**条件概率模型**，给它一段输入 token，它预测下一个 token 最可能是什么：

$$P(\text{token}_n \mid \text{token}_1, \text{token}_2, \ldots, \text{token}_{n-1})$$

你可以把它当成一个**无状态的函数**：

> 输入 Prompt → 输出文本

每次调用都是独立的，**没有记忆，没有状态，对外部世界一无所知**。

### 2. LLM 的四大天花板

面试时一定要能展开讲，不能只背关键词：

| # | 天花板 | 表现 |
|---|---|---|
| ① | **只会说不会做** | 它能告诉你"你可以去天气 App 查一下"，但它自己不会去查 |
| ② | **没有记忆** | 上下文窗口一满就"失忆"，跨会话什么都没留下 |
| ③ | **知识截止** | 训练数据有截止日期，昨天发生的事它不知道 |
| ④ | **不会规划** | 你让它"做一份竞品分析"，它只会线性回答，不会自己拆解成"先搜集资料、再逐个分析、再对比价格"这样的步骤 |

### 3. Agent 是什么？

**一句话**：

> Agent = LLM × (规划 + 记忆 + 工具)，在循环中自主完成目标。

> ⚠️ **修正**：这里用 `×` 而不是 `+` 是有意的——LLM 是底层能力，规划/记忆/工具是应用层模块，三者**乘上** LLM 才能形成 Agent。
>
> 流传更广的"Agent = LLM + 工具 + 记忆 + 规划"加法表达**便于理解**但**抽象层次混淆**：LLM 不是与其他三个模块并列的"第四个模块"，而是其他三个模块共同**调用**的推理底座。详见[第三章](#三agent-四模块描述是否正确)与[第五章](#五规划模块和-llm-的关系包含还是调用)的展开分析。

### 4. 一个例子说清楚本质区别

**任务**：帮我查一下明天北京的天气，如果下雨就取消我日历里的跑步计划。

| 角色 | 实际行为 |
|---|---|
| **LLM** | "您可以打开天气 App 查询北京明天天气，如果降雨概率超过 60% 建议取消户外运动，可以在日历 App 中删除该日程……" |
| **Agent** | 1. 调用天气 API → 明天北京中雨<br>2. 调用日历 API → 找到明天 7:00 跑步计划<br>3. 调用日历 API → 删除该计划<br>4. 回复："明天北京有雨，已为您取消跑步计划。" |

> **核心区别**：**LLM 告诉你怎么做，Agent 直接帮你做完**。

### 5. LLM vs Agent 对比图

```
┌──────────────────────────────────────────────────────────┐
│                          LLM                              │
│                                                          │
│        Prompt ───►  ┌─────────────┐  ───► Text           │
│                     │  无状态函数  │                      │
│                     └─────────────┘                      │
│                                                          │
│  特性：无记忆 / 无外部访问 / 无规划 / 单次响应           │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                         Agent                             │
│                                                          │
│   User Goal                                              │
│      │                                                   │
│      ▼                                                   │
│   ┌──────────────────────────────────────────────┐       │
│   │           Agent Loop（控制流）               │       │
│   │                                              │       │
│   │   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐│       │
│   │   │ 规划 │──►│  调  │──►│ 工具 │──►│ 观察 ││       │
│   │   │ 决策 │   │ LLM  │   │ 执行 │   │ 结果 ││       │
│   │   └──┬───┘   └──────┘   └──────┘   └──┬───┘│       │
│   │      │                                  │    │       │
│   │      └──────────────────────────────────┘    │       │
│   │              （循环直到目标完成）             │       │
│   └──────────────────────────────────────────────┘       │
│                          │                               │
│              ┌───────────┴───────────┐                   │
│              ▼                       ▼                   │
│         ┌─────────┐             ┌─────────┐              │
│         │  记忆   │             │  工具   │              │
│         │（持久） │             │（外部） │              │
│         └─────────┘             └─────────┘              │
│                                                          │
│  特性：有状态 / 可调外部 / 可规划 / 多步循环             │
└──────────────────────────────────────────────────────────┘
```

### 6. 面试加分点：四模块（修正版）

> 原话："Agent 由四个模块组合而成：LLM（大脑）负责理解意图、推理判断；规划模块负责任务拆解、步骤排序；记忆模块负责短期上下文与长期知识存储；工具模块负责调用外部 API、数据库、代码执行器等，是 Agent 的'手和脚'。"

**这段话基本正确，但有两处需要修正/澄清**，否则会被有经验的面试官追问：

| 原表述 | 问题 | 修正后表述 |
|---|---|---|
| "LLM 和规划/记忆/工具是**四个并列模块**" | LLM 不在同一抽象层——它是底层能力，其他三者是应用层 | "Agent 围绕 LLM 这个**推理内核**，组合**三大应用模块**——规划（控制流）、记忆（状态）、工具（执行）" |
| "规划模块负责任务拆解、步骤排序" | 拆解和排序的**实际推理**仍由 LLM 完成；规划模块是控制流代码 | "规划模块**驱动 LLM 进行任务拆解**，并把 LLM 的输出落地为可执行的步骤序列" |

**面试时建议这么讲**：

> "Agent 由四个**职责层**组成（注意是'职责'不是'独立的代码模块'）：
>
> 1. **LLM 内核**：作为 reasoning engine 提供理解、推理、生成能力，是无状态的；
> 2. **规划层**：基于 LLM 实现 ReAct / Plan-and-Execute 等控制循环，做任务拆解、步骤排序、错误恢复；
> 3. **记忆层**：分 working / episodic / semantic / procedural 四类，分别由 context window、conversation store、vector RAG、reflection log 实现；
> 4. **工具层**：对应 function calling / tool use，让 Agent 能调用 API、数据库、代码沙箱等'手脚'。
>
> 它们在 **Agent Loop（Observe → Think → Plan → Act → Observe）** 中协作：每一轮规划层调 LLM 决定动作 → 经工具层执行 → 把 observation 写入记忆 → 进入下一轮，直到规划层判定目标达成或失败。"

详细论证见后续章节。

---

## 二、Agent 完整请求处理流程

> **场景**：用户提出一个请求，从输入到响应整个链路 Agent 是如何调度 LLM、工具、规划、记忆模块的？
>
> 沿用上文例子：「**帮我查一下明天北京的天气，如果下雨就取消我日历里的跑步计划**」

### 1. 高层时序图

```
User           Agent           Memory          Planner          LLM           Tools
 │               │                │               │               │              │
 │── 请求 ─────►│                │               │               │              │
 │               │                │               │               │              │
 │               │── 加载历史 ──►│               │               │              │
 │               │◄── 上下文 ────│               │               │              │
 │               │                │               │               │              │
 │               │── 启动循环 ──────────────────►│               │              │
 │               │                │               │               │              │
 │               │   ╔═══════════ Agent Loop（循环 N 次）═══════════════╗       │
 │               │   ║            │               │               │       ║      │
 │               │   ║            │               │── prompt ────►│       ║      │
 │               │   ║            │               │◄── action ────│       ║      │
 │               │   ║            │               │               │       ║      │
 │               │   ║            │               │── 调用 ────────────────►│   │
 │               │   ║            │               │◄── observation ────────│   │
 │               │   ║            │               │               │       ║      │
 │               │   ║            │◄── 写状态 ───│               │       ║      │
 │               │   ║            │               │               │       ║      │
 │               │   ║（终止判断：FINISH or 继续）│               │       ║      │
 │               │   ╚════════════════════════════════════════════════════╝      │
 │               │                │               │               │              │
 │               │◄── 最终回答 ──────────────────│               │              │
 │               │                │               │               │              │
 │               │── 持久化 ────►│               │               │              │
 │               │                │               │               │              │
 │◄── 响应 ──────│                │               │               │              │
```

### 2. 分步详解（10 个阶段）

#### 阶段 0：用户请求进入

```
User: "帮我查明天北京的天气，如果下雨就取消我日历里的跑步计划"
```

Agent 服务的入口（HTTP / WebSocket / IM 接口）接收到这条消息。

#### 阶段 1：会话初始化

Agent 框架（如 LangGraph）创建一个 **Run** 对象，分配 trace_id，准备一个空的状态机：

```python
state = {
    "user_input": "帮我查明天北京的天气...",
    "user_id": "u_12345",
    "history": [],          # 当前 Run 内的步骤历史
    "scratchpad": [],       # 临时草稿
    "step": 0,
    "max_steps": 10,
    "done": False,
}
```

#### 阶段 2：记忆模块加载相关上下文

```
Memory.load(user_id="u_12345", query="跑步、日历偏好")
```

记忆模块做两件事：

- **Episodic**：从向量库检索历史对话片段（"上次用户提到他喜欢晨跑"）
- **Semantic**：从用户画像库取偏好（"用户日历 App 是 Google Calendar"）

返回的上下文被注入到 `state["memory_context"]`。

#### 阶段 3：规划模块构造首轮 Prompt

规划模块（控制流代码）拼装 Prompt：

```
[System]
你是一个能调用工具的 Agent。可用工具：
- get_weather(city, date) → 返回天气
- list_calendar_events(date) → 返回日历事件
- delete_calendar_event(event_id) → 删除事件

请用以下格式输出：
Thought: <你的分析>
Action: <工具调用 JSON> 或 FINISH
Final: <仅在 FINISH 时给出最终回复>

[User Memory]
- 用户偏好早晨跑步
- 日历是 Google Calendar

[User]
帮我查明天北京的天气，如果下雨就取消我日历里的跑步计划

[Scratchpad]
（空，第一轮）
```

#### 阶段 4：第 1 次调 LLM —— 决策"先做什么"

```
Planner ──► LLM.call(prompt)
            ◄── 返回：
            Thought: 用户要先确认天气，再根据结果决定。第一步查北京明天天气。
            Action: {"name": "get_weather", "args": {"city": "北京", "date": "2026-02-06"}}
```

#### 阶段 5：工具模块执行调用

工具模块解析 LLM 输出的 JSON，按 schema 校验后调真实 API：

```
Tools.execute("get_weather", {"city": "北京", "date": "2026-02-06"})
  ↓
HTTP GET https://api.weather.com/...
  ↓
Observation: {"condition": "rain", "precip_prob": 85, "temp": "5℃"}
```

观察结果写回 state：

```python
state["history"].append({
    "step": 1,
    "thought": "用户要先确认天气...",
    "action": {"name": "get_weather", "args": {...}},
    "observation": {"condition": "rain", ...}
})
```

#### 阶段 6：第 2 次调 LLM —— "拿到天气后下一步做什么"

规划模块**重新构造 Prompt**（把上一轮的 thought + action + observation 拼进 scratchpad），再次调 LLM：

```
[Scratchpad]
Step 1:
  Thought: 用户要先确认天气...
  Action: get_weather(...)
  Observation: {"condition": "rain", "precip_prob": 85}

请输出下一步：
```

LLM 返回：

```
Thought: 降雨概率 85%，确认要下雨。需要查明天的跑步日程。
Action: {"name": "list_calendar_events", "args": {"date": "2026-02-06"}}
```

#### 阶段 7：工具模块再次执行

```
Tools.execute("list_calendar_events", {"date": "2026-02-06"})
  ↓
Observation: [
  {"id": "evt_001", "title": "跑步", "time": "07:00"},
  {"id": "evt_002", "title": "晨会", "time": "10:00"}
]
```

#### 阶段 8：第 3 次调 LLM —— "找到跑步事件了，下一步呢"

```
Thought: 找到了 evt_001 是跑步事件，删除它。
Action: {"name": "delete_calendar_event", "args": {"event_id": "evt_001"}}
```

工具执行：

```
Observation: {"success": true}
```

#### 阶段 9：第 4 次调 LLM —— 终止判断

```
Thought: 跑步已取消，任务完成。
Action: FINISH
Final: 明天北京有雨（降雨概率 85%），已为您取消明早 7:00 的跑步计划。
```

规划模块解析到 `FINISH`，跳出循环。

#### 阶段 10：记忆持久化 + 返回响应

记忆模块写入两类记录：

```python
Memory.save_episodic(
    user_id="u_12345",
    summary="帮用户根据天气取消了跑步日程",
    raw_history=state["history"]
)

Memory.update_procedural(
    pattern="天气-日程联动决策",
    success=True
)
```

最终回复返回给用户。

### 3. 完整流程的 Python 伪代码

```python
def handle_request(user_input, user_id):
    # 阶段 1: 初始化
    state = init_state(user_input, user_id)

    # 阶段 2: 加载记忆
    state["memory_context"] = memory.load(user_id, query=user_input)

    # 阶段 3-9: Agent Loop
    while state["step"] < state["max_steps"] and not state["done"]:
        # 构造 Prompt
        prompt = build_react_prompt(state)

        # 调 LLM
        response = llm.call(prompt)
        thought, action = parse(response)

        # 终止？
        if action.type == "FINISH":
            state["final_answer"] = action.final
            state["done"] = True
            break

        # 调工具
        observation = tools.execute(action.name, action.args)

        # 更新状态
        state["history"].append({
            "step": state["step"],
            "thought": thought,
            "action": action,
            "observation": observation,
        })
        state["step"] += 1

    # 阶段 10: 持久化
    memory.save(user_id, state)

    # 返回
    return state["final_answer"]
```

### 4. 关键观察点

| 观察 | 说明 |
|---|---|
| LLM 被调用 **N 次**（N = 步骤数 + 1） | 每一步都需要 LLM 决策"下一步做什么"，最后再调一次输出 FINISH |
| LLM **始终无状态** | 每次调用都是独立的 prompt → response，状态完全由规划模块在外层维护 |
| 工具 **同步执行** | 一个工具不返回，下一步无法规划（也支持并行执行，需要规划模块支持 DAG） |
| 记忆 **首尾各一次** + **每步追加** | 加载（开始）→ 追加 history（每步）→ 持久化（结束） |
| 规划模块 **不做创造性决策** | 它只做"构造 prompt → 调 LLM → 解析输出 → 调工具 → 终止判断"五件机械事 |
| **失败处理**未画出 | 实际生产中每个阶段都要包 try-except：LLM 输出格式错误重试、工具调用失败 replan、超 max_steps 兜底回复 |

### 5. 不同 Agent 框架的"Loop"差异

不同框架的 loop 结构略有不同，但核心都是上述模式：

| 框架 | Loop 风格 | 特点 |
|---|---|---|
| **ReAct** | Thought → Action → Observation → Thought ... | 最经典，每步重决策 |
| **Plan-and-Execute** | Plan → Execute → Plan(if needed) → ... | 先一次性出全计划，再执行 |
| **LangGraph** | 状态机节点 → 条件边 → 节点 ... | 可视化的 DAG |
| **OpenAI Assistants API** | run.create → poll → tool_outputs → poll | 把循环托管给 OpenAI 服务端 |
| **CrewAI** | Agent 之间消息传递 | 多 Agent 协作 |

无论哪种，**"规划模块发起 LLM 调用 → LLM 返回决策 → 工具执行 → 观察喂回"**这个核心模式不变。

### 6. 面试一句话总结

> "Agent 处理一个请求的本质是：**规划模块作为带状态的控制器，反复调用无状态的 LLM 做决策，每次决策的产物（Action）由工具模块执行，执行结果（Observation）写回状态作为下次 LLM 决策的输入；记忆模块在循环开始时注入历史上下文、循环结束时持久化新的经验；整个循环直到 LLM 主动输出 FINISH 或达到 max_steps 兜底。**"

---

## 三、Agent 四模块描述是否正确？

### 题目

> Agent 由四个模块组合而成：LLM（大脑）负责理解意图、推理判断；规划模块负责任务拆解、步骤排序；记忆模块负责短期上下文与长期知识存储；工具模块负责调用外部 API、数据库、代码执行器等，是 Agent 的"手和脚"。面试时别只说"Agent 就是 LLM 加工具"，要展开讲这四个模块各自的作用，以及它们怎么在循环中协作。

### 整体结论

这段话**基本正确，符合主流 Agent 架构教科书的描述**（最经典版本来自 Lilian Weng 2023 年那篇 *LLM Powered Autonomous Agents* 博客，OpenAI、Anthropic、LangChain 文档基本都是这套框架）。但有一个**容易引起面试官追问的概念混淆点**——LLM 和"规划/记忆/工具"不在同一抽象层。

### 1. 正确的部分

| 模块 | 描述 | 评价 |
|---|---|---|
| LLM（大脑） | 理解意图、推理判断 | ✅ 准确，是主流共识 |
| 规划模块 | 任务拆解、步骤排序 | ✅ 准确，但实现方式需要澄清 |
| 记忆模块 | 短期上下文 + 长期知识 | ✅ 准确。短期=对话上下文/scratchpad；长期=向量数据库/RAG |
| 工具模块 | 调外部 API、数据库、代码执行器 | ✅ 准确，对应 function calling / tool use |
| 四模块在**循环**中协作 | ✅ 准确。这就是经典的 **Agent Loop**（Observe → Think → Plan → Act → Observe …），ReAct 框架的核心 |

### 2. 需要警惕的两个不准确之处

#### (1) "规划模块" 是个容易误导的实现描述

主流 Agent **没有一个独立的"规划模块"在 LLM 之外**。规划本质上是 **LLM 做出来的能力**，只是套了不同的 prompt 模板/框架。所以严谨表达应该是：

> "规划是 LLM 的一种**应用模式**，由特定的 prompt 框架（ReAct / Plan-and-Execute / Tree of Thoughts / Reflection）驱动 LLM 输出结构化的步骤"

少数高级架构里规划**才**是独立模块（比如 multi-agent 系统中专门有个 Planner Agent，可能用更便宜/更快的模型；或 LLM Compiler 这类 paper 里的静态规划器），但这是例外不是常态。

#### (2) "短期上下文与长期知识" 的描述偏笼统

更精确的拆法：

| 记忆类型 | 典型实现 |
|---|---|
| **Working Memory（工作记忆）** | LLM 当前 context window 里的 token，包括对话历史、scratchpad |
| **Episodic Memory（情景记忆）** | 跨会话的对话历史，用 vector store + similarity search 检索 |
| **Semantic Memory（语义记忆）** | RAG 检索的事实知识、外部文档库 |
| **Procedural Memory（程序性记忆）** | 学到的工具用法、行为模式（比如 reflection 机制写下的 "上次这么做失败了，下次别这样"）|

面试时如果想加分，把这 4 类讲出来。

### 3. 面试加分的"修订版"表述

> Agent 的标准架构由四个**职责层**组成（注意是"职责"不是"独立的代码模块"）：
>
> 1. **LLM 内核**：作为 reasoning engine 提供理解、推理、生成能力，是无状态的；
> 2. **规划层**：基于 LLM 实现 ReAct / Plan-and-Execute 等控制循环，做任务拆解、步骤排序、错误恢复，是**有状态的目标驱动控制器**；
> 3. **记忆层**：分 working / episodic / semantic / procedural 四类，分别由 context window、conversation store、vector RAG、reflection log 实现；
> 4. **工具层**：对应 function calling / tool use，让 Agent 能调用 API、数据库、代码沙箱等"手脚"。
>
> 它们在 **Agent Loop（Observe → Think → Plan → Act → Observe）** 中协作：每一轮规划层调 LLM 决定动作 → 经工具层执行 → 把 observation 写入记忆 → 进入下一轮，直到规划层判定目标达成或失败。

### 4. 可能被追问的延伸问题（提前准备）

| 追问 | 准备方向 |
|---|---|
| ReAct 和 Plan-and-Execute 的区别？ | ReAct 是边想边做（每步重新规划）；P&E 是先规划完整路径再执行 |
| 规划模块用的还是同一个 LLM 吗？ | 通常是。但成本敏感场景会用 cheaper model（如 Haiku/4o-mini）做规划，main task 用 strong model |
| 长期记忆怎么避免相关性退化？ | Hierarchical memory、memory compression、recency vs. importance 加权 |
| 工具模块怎么解决幻觉调用？ | JSON schema 强约束 + structured output + 工具 description 清晰 + few-shot examples |
| 多轮失败如何避免无限循环？ | max iterations + reflection（自我批评） + human-in-the-loop |

---

## 四、LLM 和规划模块的本质区别

### 区别一：抽象层次不同

| 维度 | LLM | 规划模块 |
|---|---|---|
| 角色 | **能力提供者**（capability provider） | **能力应用器**（capability orchestrator） |
| 类比 | CPU | 操作系统的进程调度器 |
| 输入 | 任意文本 | "用户的最终目标 + 当前状态" |
| 输出 | next-token 概率分布 | "下一步该做什么"的结构化描述（JSON / DSL / 工具调用） |
| 是否可替换 | 可以换不同模型（GPT-4 → Claude → Llama） | 可以换不同算法（ReAct → Plan-and-Execute → Reflexion） |

### 区别二：职责边界不同

- **LLM** 负责回答 *"这段话什么意思"* / *"这个步骤逻辑对吗"* / *"下一句应该说什么"* 这类**短程、单步**问题
- **规划模块**负责回答 *"为了达成 X，需要先做 A、B、C，遇到 D 该回退"* 这类**长程、多步、有状态**问题

举个具体例子，用户说 "帮我分析公司 Q3 营收然后写邮件给老板"：

| 步骤 | LLM 单独能做？ | 需要规划模块？ |
|---|---|---|
| 理解 "Q3" 是哪个季度 | ✅ | — |
| 拆成 "查数据 → 分析 → 写邮件 → 发送" 四步 | ⚠️ 能拆，但不会执行 | ✅ 由规划模块组织成可执行序列 |
| 决定用哪个工具查数据（SQL / API） | ⚠️ 一次性输出，不维护状态 | ✅ 维护 "已尝试 / 失败 / 待重试" 的状态机 |
| 中途数据库连接失败要 replan | ❌ LLM 不知道要重新规划 | ✅ 规划模块捕获 observation 后重新调 LLM 出新计划 |
| 生成邮件正文 | ✅ | — |

### 区别三：实现技术栈不同

- **LLM** 实现：Transformer 推理引擎（vLLM / TGI / TensorRT-LLM），关心的是吞吐、延迟、显存
- **规划模块**实现：通常是 Python/TypeScript 的 orchestration 框架（LangGraph / LlamaIndex / CrewAI / AutoGen），关心的是状态机、错误恢复、并发执行、观察 → 决策循环

### 区别四：面试时的"杀手锏"句

> "**LLM 是无状态的概率分布生成器，规划模块是有状态的目标驱动控制器**。LLM 可以告诉你'下一句应该说什么'，但只有规划模块能告诉你'为了达成最终目标，下一步该往哪走、走错了怎么回退'。规划模块在每一步都会**调用 LLM** 来做决策，但它额外承担了**记账（state tracking）、重试（retry/reflection）、终止判断（termination check）**的职责。"

---

## 五、规划模块和 LLM 的关系：包含还是调用？

### 题目

> 所以说规划模块和 LLM 并不是并立的关系，而是规划模块中包含了 LLM？

### 结论：是"调用"关系，不是"包含"关系

```
       ┌────────────────────────────────┐
       │       规划模块（Planner）       │
       │   ──────────────────────────   │
       │   - 维护目标 / 状态 / 历史     │
       │   - 决定何时调 LLM、调几次     │
       │   - 解析 LLM 输出、判断终止    │
       │                                │
       │      ┌──────────────────┐      │
       │      │  调用 → LLM API  │ ←─── 不是"内嵌"，是远程/本地调用
       │      └──────────────────┘      │
       └────────────────────────────────┘
                       │
                       ↓ 调用
              ┌────────────────┐
              │      LLM       │   ← 独立的服务/进程/模型权重
              │ （无状态推理） │
              └────────────────┘
```

| 表述 | 准确性 | 解释 |
|---|---|---|
| ❌ "规划模块和 LLM 并立" | 不准确 | 二者不在同一抽象层；面试时会被追问"那它俩怎么交互的？" |
| ⚠️ "规划模块包含 LLM" | **半对** | 在概念架构图上可以这么画，但容易误解为"LLM 是规划模块的子组件" |
| ✅ "规划模块**调用** LLM" | 最准确 | LLM 是被调用的服务，规划模块是调用方/编排方 |
| ✅ "规划模块**以 LLM 为推理引擎**" | 最准确 | 强调 LLM 提供能力、规划模块负责使用能力 |

### 为什么"包含"不够准确

#### 1. 物理层面：LLM 通常是独立部署的

- LLM 可能在 GPU 集群上、可能是 OpenAI 的 HTTPS 端点、可能是本地 vLLM 进程
- 规划模块通常是 Python/TS 应用代码，跑在 CPU 上
- 二者通过 **HTTP / gRPC / 进程间调用** 通信，**不是同一个二进制里的两个类**

如果说"规划模块包含 LLM"，会让人误以为它们是同一个模块的两个内部组件，那就抹掉了"LLM 可被替换"这个核心架构特性（同一个规划模块今天接 GPT-4，明天换成 Claude，是常见操作）。

#### 2. 职责层面：LLM 不是规划模块独占的

LLM 不仅被规划模块调用，**也被其他模块调用**：

```
                 ┌──────────────────────────┐
                 │     LLM (推理引擎)       │
                 └──────────────────────────┘
                    ↑      ↑      ↑      ↑
                    │      │      │      │
       ┌────────────┘      │      │      └──────────────┐
       │                   │      │                     │
  规划模块            记忆模块   工具模块           直接对话
（决定下一步）   （写入摘要/   （决定调哪    （Agent Loop 之外
                  长期记忆）    个工具）       的纯 chat）
```

- **记忆模块**也调 LLM：比如长对话需要做 **summarization compression**，要 LLM 把 100 轮对话压成一段摘要
- **工具模块**也调 LLM：比如根据自然语言 query 决定调 `search_db` 还是 `call_api`，参数怎么填
- **反思（Reflection）**也调 LLM：让 LLM 评判 "上一步做得怎么样"

LLM 是**横跨四个模块的共享基础设施**，不是规划模块的下属。

#### 3. 架构层面：正确的层次图

```
┌─────────────────────────────────────────────────┐
│  应用层（Application Layer）                     │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │ 规划   │  │ 记忆   │  │ 工具   │  │ 安全   │ │
│  │ 模块   │  │ 模块   │  │ 模块   │  │ 护栏   │ │
│  └────┬───┘  └────┬───┘  └────┬───┘  └────┬───┘ │
└───────┼───────────┼───────────┼───────────┼─────┘
        │           │           │           │
        └───────────┴─────┬─────┴───────────┘
                          ↓
        ┌───────────────────────────────────┐
        │   能力层（Capability Layer）       │
        │   ┌─────────┐  ┌──────────────┐    │
        │   │   LLM   │  │ Embedding    │    │
        │   │ (推理)  │  │ (检索)       │    │
        │   └─────────┘  └──────────────┘    │
        └───────────────────────────────────┘
```

- **LLM 在能力层**（基础设施）
- **规划/记忆/工具在应用层**（业务逻辑）
- 应用层各模块**并立**，但都依赖 LLM 这个底座

### 一句话总结

> 规划模块不"包含" LLM，而是"以 LLM 为推理后端"。规划模块是带状态的调度器，每次需要做决策时就 prompt LLM 拿到下一步动作；LLM 本身是无状态的纯推理服务，不知道自己是被 Agent 调还是被聊天框调。这个"调用关系"是 Agent 架构的核心，也是为什么换底层模型（GPT → Claude）不需要重写规划逻辑的原因。

---

## 六、规划模块是大模型吗？LLM 等于大模型吗？

### 1. "规划模块也是一种大模型吗？" → ❌ **不是**

**规划模块本质是一段普通的应用程序代码**（通常是 Python 或 TypeScript），它本身**没有任何模型权重、不做推理**。它的作用是：

- 维护任务状态（当前在第几步、做完了什么、还差什么）
- 决定**何时**调 LLM、**给 LLM 什么 prompt**
- 解析 LLM 返回的文本/JSON
- 路由到工具调用、错误重试、终止判断

举个具体例子，LangGraph 里的一个最小规划循环可能是这样：

```python
def planner_loop(goal):
    state = {"goal": goal, "history": [], "done": False}
    for step in range(MAX_STEPS):
        prompt = build_prompt(state)
        response = llm.call(prompt)        # ← 唯一调 LLM 的地方
        action = parse_action(response)
        if action.type == "FINISH":
            state["done"] = True
            break
        result = execute_tool(action)
        state["history"].append((action, result))
    return state
```

整段代码里**没有任何模型权重**。所谓"规划"就是这个 `for` 循环 + `build_prompt` + `parse_action` 的逻辑。**它是软件工程层面的编排，不是 AI 模型**。

> 类比：操作系统的进程调度器不是 CPU，但它指挥 CPU 怎么跑。

#### 例外情况（少数）

在一些研究性架构里，确实存在**专门训练用来做规划的模型**：

| 例子 | 是不是大模型？ |
|---|---|
| **LLM Compiler**（Berkeley 2024） | 是 LLM，但被 fine-tune 成专门输出 DAG 计划 |
| **Plansformer**（IBM） | 是个小型 transformer，专门做 PDDL 规划 |
| **AlphaGo 的 MCTS** | 不是 LLM，是搜索算法 + 神经网络估值 |
| **多 Agent 架构里的 Planner Agent** | 是 LLM（通常用便宜的小模型如 Haiku/4o-mini） |

但这些都是**特例**。**主流 Agent 架构里的"规划模块"是 prompt 工程 + 控制流代码，不是模型**。

### 2. "该模块底层会调用 LLM 吗？" → ✅ **会，这是它的本质工作**

规划模块的**每一步决策**都靠调 LLM 实现。典型的几种调用模式：

| 框架 | 规划模块每轮做什么 | 调 LLM 几次 |
|---|---|---|
| **ReAct** | 让 LLM 输出 `Thought + Action`，执行后把 `Observation` 喂回 | 每步 1 次 |
| **Plan-and-Execute** | 第 1 次让 LLM 输出完整 plan，之后每步执行时再调 LLM 决定细节 | 1 + N 次 |
| **Reflexion** | 执行完后再调一次 LLM 自我批评 | 每步 2 次（决策 + 反思） |
| **Tree of Thoughts** | 每个节点让 LLM 出 K 个候选，再让 LLM 评估哪个最好 | 每节点 K+1 次 |

所以"规划模块"≈ "**一段会反复调 LLM 的控制循环代码**"。它的"智能"完全来自 LLM，自身只贡献"什么时候调、调几次、怎么处理结果"的工程逻辑。

### 3. "LLM 等价于大模型吗？" → ⚠️ **几乎等价，但严格说不等价**

中文语境下经常混用，但严谨技术语境里有区别：

#### LLM = Large Language Model（大语言模型）

- 范围**只**包含**处理文本**的大模型
- 例：GPT-4、Claude、Llama、Qwen、DeepSeek、ChatGLM

#### 大模型 = Large Model / Foundation Model（大模型 / 基础模型）

- 范围**更广**，包括所有"大规模预训练 + 通用能力"的模型
- 不只是文本

#### 二者关系图

```
                  ┌─────────────────────────┐
                  │     大模型 / 基础模型    │
                  │   (Foundation Model)    │
                  └────────────┬────────────┘
                               │
        ┌──────────────────────┼─────────────────────────┐
        │                      │                         │
    ┌───▼───┐            ┌─────▼─────┐            ┌──────▼──────┐
    │  LLM  │            │   多模态  │            │   领域大模型 │
    │ (文本) │            │   (VLM)  │            │  (生物/代码)│
    └───────┘            └───────────┘            └──────────────┘
       │                       │                         │
   GPT-4                  GPT-4V              AlphaFold
   Claude                 Gemini Vision       BioGPT
   Llama                  Sora (视频)         CodeLlama
   Qwen                   Whisper (语音)
                          DALL-E (图像生成)
```

#### 一张精确对照表

| 模型 | 是 LLM 吗？ | 是大模型吗？ |
|---|---|---|
| GPT-4 (纯文本) | ✅ 是 | ✅ 是 |
| Claude 3.5 Sonnet | ✅ 是 | ✅ 是 |
| GPT-4V (带视觉) | ⚠️ 通常算 LLM 的扩展，严谨说叫 **VLM/MLLM** | ✅ 是 |
| Gemini 1.5 (原生多模态) | ❌ 严格说不算"L"LM，是 **多模态大模型（LMM）** | ✅ 是 |
| Whisper (语音转文字) | ❌ 不是 | ✅ 是（语音大模型） |
| DALL-E 3 (文生图) | ❌ 不是 | ✅ 是（图像生成大模型） |
| Stable Diffusion | ❌ 不是 | ✅ 是（扩散模型） |
| AlphaFold (蛋白质结构) | ❌ 不是 | ✅ 是（领域大模型） |
| BERT (110M 参数) | ⚠️ 边缘案例，曾经叫 LM，现在大模型时代不算"大" | ❌ 不算"大" |

#### 在中文实际使用中

口语和大众媒体里，**"大模型"和 "LLM" 经常混用**，因为目前最火的大模型几乎都是 LLM 或 LLM 的多模态扩展。所以日常对话里你说"大模型"基本等于说 LLM，没问题。

但如果**面试或写论文**，要注意区分：
- 谈 ChatGPT、Claude、Llama → 用 **LLM**
- 谈 Sora、DALL-E、Whisper、AlphaFold → 用 **大模型**（不要说 LLM）
- 谈 GPT-4V、Gemini、Claude 3.5 with vision → 用 **多模态大模型 / MLLM / LMM / VLM**

### 4. 一句话总结三个问题

| 问题 | 答案 |
|---|---|
| 规划模块是大模型吗？ | **不是**。它是一段调度 LLM 的应用代码，本身没有模型权重 |
| 规划模块底层调用 LLM 吗？ | **是**。每一步决策都靠调 LLM 完成，规划模块只是"何时调、怎么处理结果"的控制层 |
| LLM 等价于大模型吗？ | **几乎等价但不严格等价**。LLM 只是大模型家族中"处理文本"那一支；多模态、视觉、语音、生物等大模型都不算 LLM |

---

## 七、普通代码会不会"死板"？能否针对不同任务做不同规划？

### 1. "它们都是普通应用程序代码？" → ✅ **是的**

| 模块 | 实质 | 典型实现 |
|---|---|---|
| 规划模块 | for 循环 + 状态机 + prompt 模板 | LangGraph、CrewAI 的 Python 代码 |
| 记忆模块 | 数据库 CRUD + 向量检索 + 摘要触发 | Redis / Postgres / Pinecone / Chroma 的 Python 包装 |
| 工具模块 | function 注册表 + JSON Schema 校验 + 调用分发 | LangChain Tools / OpenAI function calling 的客户端代码 |

**没有任何一个模块自身有模型权重**。它们都是"调用 LLM、调用数据库、调用 API"的胶水代码。

但请注意：**"普通应用程序代码"≠"死板"**。

### 2. "它们会不会太死板？" → ❌ **不会，因为"灵性"是 LLM 注入的**

要解答这点，必须先理清一个**关键架构原则**：

> **规划模块自己不"思考"，它把所有"思考"任务全部外包给 LLM。规划模块只负责"把 LLM 的输出落地为可执行动作"。**

#### 死板代码 + 灵性 LLM = 灵性 Agent

打个比方：

```
死板代码 = 一个非常听话的秘书
灵性 LLM = 一个非常聪明的老板
```

秘书自己不做创造性决策，但每件事都问老板"下一步怎么办"，然后老板说什么她就办什么。整个团队的"灵性"完全来自老板，秘书只贡献执行力。

#### 看一段真实的"看似死板、实则灵性"的代码

```python
def react_loop(goal):
    history = []
    for _ in range(MAX_STEPS):
        prompt = f"""
        目标：{goal}
        历史：{history}
        可用工具：{TOOL_DESCRIPTIONS}
        请输出：
        Thought: <你的思考>
        Action: <要调的工具名 + 参数 JSON>
        如果完成则输出 Action: FINISH
        """
        response = llm.call(prompt)        # ← 灵性的源头
        thought, action = parse(response)
        if action == "FINISH":
            return synthesize(history)
        observation = execute_tool(action)
        history.append((thought, action, observation))
```

这段代码本身**完全死板**：就是个 for 循环 + 字符串拼接 + 函数调用。但：

- 每次 `llm.call(prompt)` 都可能输出**完全不同的 Thought / Action**
- 同一个用户问 "帮我查天气" 和 "帮我订机票"，规划模块**代码一字不改**，但 LLM 输出的 Action 序列截然不同
- 任务复杂度自动适应：简单任务 LLM 可能 1 步就 FINISH，复杂任务 LLM 自动多步推理

**死板的是控制流的"骨架"，灵性的是 LLM 填进骨架里的"血肉"**。

#### 一个具体的"灵性"案例

| 用户输入 | LLM 输出的 Action 序列（规划模块代码不变） |
|---|---|
| "今天北京天气" | `[get_weather("北京")]` → FINISH，1 步 |
| "我想去三亚旅游，帮我看看适不适合" | `[get_weather("三亚")] → [search_flights("三亚")] → [get_hotel_prices("三亚")] → [check_holiday_calendar()]` → FINISH，4 步 |
| "我刚从你那儿订的机票出问题了" | `[search_user_orders()] → [get_order_detail(id=123)] → [check_refund_policy()] → [draft_complaint_email()]` → 等用户确认 → `[send_email()]` → FINISH |

规划模块就是**那个 `for` 循环**，但因为 LLM 在每一轮都重新决策，**外部表现就是高度灵性的多步决策**。

### 3. "针对不同任务能做不同规划吗？" → ✅ **完全可以**

#### 维度 A：同一框架，不同 prompt → 自动适配任务（最常见）

规划模块代码不变，只换 prompt 里的 `tools` 描述和 `goal`，LLM 自然会输出适配该任务的步骤序列。

| 任务类型 | LLM 自动产出的规划风格 |
|---|---|
| 信息检索类 | 单步直接调 search 工具 |
| 数据分析类 | "查询 → 清洗 → 聚合 → 可视化" 多步 |
| 创作类 | "提纲 → 分段 → 润色 → 校对" 流水线 |
| 故障排查类 | "症状 → 假设 → 验证 → 修复 → 回归" 树形探索 |
| 编程类 | "理解需求 → 写代码 → 跑测试 → 修 bug" 循环 |

这种"任务适配"**100% 由 LLM 完成**，规划模块代码完全通用。这就是为什么 ChatGPT、Claude 的 Agent 能力对所有任务都有用，不需要为每类任务写专门的规划器。

#### 维度 B：不同框架处理不同任务复杂度（架构选型）

| 框架 | 适合的任务 | 规划风格 |
|---|---|---|
| **直接调用**（单步） | 简单 QA、翻译 | 不规划，直接答 |
| **ReAct** | 中等复杂度，路径不确定 | 边想边做，每步重新规划 |
| **Plan-and-Execute** | 路径相对清晰的复合任务 | 先制定完整计划，再执行 |
| **Tree of Thoughts** | 需要探索多种方案的难题 | 树形展开 + 评估剪枝 |
| **Reflexion** | 容易失败需要反思的任务 | 失败后自我批评再重试 |
| **Multi-Agent** | 需要多专家协作的任务 | 多个 LLM 实例分工协作 |
| **Hierarchical Planning** | 超长任务（如自动写一本书） | 高层 Planner + 低层 Executor 两级 |

工程上**可以根据任务类型预先选用不同框架**，甚至**让一个 LLM 自己判断该用哪种规划框架**（meta-planning）。

#### 维度 C：LLM 在运行时动态调整规划策略（高级能力）

成熟 Agent 框架支持 LLM 在执行中**动态修改自己的计划**：

- **Replan**：执行到一半发现假设错了，重新规划剩余步骤
- **Branch & Backtrack**：尝试方案 A 失败后回退到分支点试方案 B
- **Subgoal Decomposition**：发现一个步骤太大，临时拆成子目标
- **Tool Discovery**：发现现有工具不够用，请求人类提供新工具

这些"动态调整"的决策**还是 LLM 做的**，规划模块代码只是把 LLM 的"我想换个方案"这条输出**翻译成**修改内部状态机的具体动作。

### 4. 所谓"灵性"的真正分布

```
┌──────────────────────────────────────────────────────────┐
│                    Agent 的"灵性"分布                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   95%+ 来自 LLM       →  理解、推理、决策、创作、反思    │
│                                                          │
│   ~5%  来自规划模块   →  框架选型（ReAct? P&E?）         │
│                          状态保留（多轮记得上下文）       │
│                          错误恢复（重试/降级策略）        │
│                                                          │
│   ~0%  来自记忆/工具  →  纯执行（存/取/调）              │
│         模块                                             │
└──────────────────────────────────────────────────────────┘
```

**"灵性"和"代码死板"不冲突**，它们工作在不同抽象层：
- 代码层：死板、确定性、可测试、可维护
- 模型层：灵性、概率性、创造性、适应性

这种"**死板的骨架 + 灵性的内核**"恰恰是 Agent 架构最聪明的设计——把不确定性集中在 LLM 这一处，外围全用确定性代码包裹，既保证了能力上限（取决于 LLM），又保证了工程可靠性（取决于代码）。

### 5. 什么情况下规划模块**确实**会显得死板？

| 现象 | 真实原因 | 谁的锅 |
|---|---|---|
| 同样的问题反复给同样的错误答案 | 规划模块**没有反思机制**（没用 Reflexion） | 规划模块设计不足 |
| 复杂任务 Agent 摆烂、提前 FINISH | LLM 不够强，对长任务 follow instruction 能力差 | LLM 锅 |
| 工具用错（明明有 search 却没用） | 工具描述写得太烂，LLM 看不懂何时该用 | 工具模块设计不足 |
| 第二次问同一问题不记得上次 | 记忆模块没接入 / 没检索到相关历史 | 记忆模块设计不足 |
| 一遇到 API 报错就死循环 | 规划模块没设 max retries 或 backoff | 规划模块设计不足 |

**所以"灵性不足"通常不是 LLM 不灵，而是规划/记忆/工具模块的"骨架"没搭好**。这也是为什么这些"普通代码"的设计仍然是 Agent 工程的核心难点——骨架决定了 LLM 这个"灵性内核"的能力能发挥多少。

### 6. 一句话总结

> **规划/记忆/工具模块都是普通代码，但"灵性"被注入在 LLM 那一层。规划模块本身死板，但每一步都问 LLM "怎么办"，所以整体表现得灵性。同一份规划代码可以适配千差万别的任务，因为 LLM 会根据任务自动产出不同的步骤序列——"灵性"是 LLM 的，"骨架"是代码的，二者各司其职。**

---

## 附录：核心概念速查

### Agent Loop

> Observe（观察当前状态/工具返回） → Think（LLM 推理） → Plan（决定下一步） → Act（调用工具） → Observe（拿到 observation）→ ...

直到规划模块判定目标达成或达到 max iterations 终止。

### 经典规划框架对照

| 框架 | 论文/出处 | 一句话 |
|---|---|---|
| ReAct | Yao et al. 2022 | Thought → Action → Observation 循环 |
| Plan-and-Execute | BabyAGI / AutoGPT | 先全局规划，再逐步执行 |
| Tree of Thoughts | Yao et al. 2023 | 树形展开多个思路 + 评估剪枝 |
| Reflexion | Shinn et al. 2023 | 失败后让 LLM 自我反思再重试 |
| LLM Compiler | Kim et al. 2024 | 把任务编译成可并行执行的 DAG |

### 记忆四象限

| 类型 | 时间跨度 | 实现 |
|---|---|---|
| Working | 当前对话 | LLM context window |
| Episodic | 历史会话 | Vector store + similarity search |
| Semantic | 通用知识 | RAG / 知识图谱 |
| Procedural | 行为习惯 | Reflection log / fine-tuning |

### 关键术语英文对照

| 中文 | 英文 | 缩写 |
|---|---|---|
| 大语言模型 | Large Language Model | LLM |
| 大模型 / 基础模型 | Foundation Model | FM |
| 多模态大模型 | Large Multimodal Model | LMM / MLLM |
| 视觉语言模型 | Vision-Language Model | VLM |
| 工具调用 | Function Calling / Tool Use | — |
| 检索增强生成 | Retrieval-Augmented Generation | RAG |
| 智能体 | Agent | — |
| 规划 | Planning | — |
| 反思 | Reflection / Reflexion | — |
| 思维链 | Chain of Thought | CoT |
| 思维树 | Tree of Thoughts | ToT |
