# Agent 四大工作模式总结

> 面试常问：「你了解 ReAct 吗？除了 ReAct 还有哪些 Agent 工作模式？各自的优缺点？」「Agent 陷入死循环怎么办？」
> 本文先汇总四种主流模式，再回答两个本质问题：
> - **Q1**：Plan-and-Execute 中 execute 的每个 step 是 LLM 还是工具调用？
> - **Q2**：Multi-Agent 为什么要拆成多个 Agent？同一个 Agent 用不同模式不行吗？

---

## 目录

- [一、四大工作模式速览](#一四大工作模式速览)
- [二、模式一：ReAct（推理 + 行动）](#二模式一react推理--行动)
- [三、模式二：Plan-and-Execute（先规划再执行）](#三模式二plan-and-execute先规划再执行)
- [四、模式三：Reflection（自我反思）](#四模式三reflection自我反思)
- [五、模式四：Multi-Agent（多智能体协作）](#五模式四multi-agent多智能体协作)
- [六、防止 ReAct 死循环（面试高频）](#六防止-react-死循环面试高频)
- [七、Q1：Plan-and-Execute 中 execute 的每个 step 到底是什么？](#七q1plan-and-execute-中-execute-的每个-step-到底是什么)
- [八、Q2：Multi-Agent 为什么要拆成多个 Agent？](#八q2multi-agent-为什么要拆成多个-agent)

---

## 一、四大工作模式速览

| 模式 | 核心思路 | 一句话 | 典型场景 |
|---|---|---|---|
| **ReAct** | Thought → Action → Observation 循环 | 边想边做，每步重新判断 | 通用问答、客服、查询类 |
| **Plan-and-Execute** | 先一次性规划，再分步执行 | 先列大纲，再填内容 | 调研、报告、流程明确的多步任务 |
| **Reflection** | 生成 → 审查 → 修改 循环 | 写完自己改，或让另一个 Agent 改 | 代码生成、法律文书、创意写作 |
| **Multi-Agent** | 多个专业 Agent 协作 | 团队作战，分工合作 | 复杂项目、需要并行/专业分工 |

> 这四种模式**不是互斥的**——一个真实系统通常会组合使用，比如 Multi-Agent 里的每个子 Agent 可能用 ReAct，外层用 Plan-and-Execute 编排。

---

## 二、模式一：ReAct（推理 + 行动）

最经典的 Agent 工作模式，几乎所有主流框架（LangChain、LangGraph）的默认选择。

### 核心循环

```
┌──► Thought（思考）
│        │
│        ▼
│    Action（调工具）
│        │
│        ▼
└── Observation（观察结果）
```

直到 LLM 主动输出 `FINISH`。

### 实际跑起来长这样

```
Thought: 用户想查北京明天天气，我需要调用天气工具
Action: get_weather(city="北京", date="明天")
Observation: {"city":"北京","weather":"中雨","temp":"14-20°C"}

Thought: 天气查到了，是中雨。用户没说要不要取消日程，应该直接告诉他
Action: FINISH
Final Answer: 明天北京有中雨，气温 14-20°C，建议带伞。
```

### 优缺点

| 优点 | 缺点 |
|---|---|
| ✅ 透明可审计（每步思考可见） | ❌ Token 消耗大（每步都要完整推理） |
| ✅ 灵活适应（看到结果可调整） | ❌ 可能死循环（工具反复失败时） |
| ✅ 通用性强 | ❌ 延迟高（每步都要等 LLM 响应） |

---

## 三、模式二：Plan-and-Execute（先规划再执行）

### 出发点

ReAct 每一步都要重新思考全局，Token 消耗太大。Plan-and-Execute 的思路是：**先把计划想清楚，再按计划逐步执行**，省去每步的重复全局推理。

### 两阶段架构

```
┌──────────── 第一阶段：Plan（规划）────────────┐
│  Planner LLM 一次性生成完整步骤列表           │
│  ["搜集竞品列表",                             │
│   "逐个分析功能",                             │
│   "对比价格策略",                             │
│   "分析用户评价",                             │
│   "生成对比报告"]                             │
└──────────────┬─────────────────────────────────┘
               │
               ▼
┌──────────── 第二阶段：Execute（执行）─────────┐
│  Executor 按顺序处理每一步                    │
│  step 1 → step 2 → step 3 → ... → step N      │
└────────────────────────────────────────────────┘
```

### Token 消耗对比

- **ReAct**：每步都思考全局，消耗 100%
- **Plan-and-Execute**：规划一次性完成，执行省力，约 20~40%

### 致命问题：计划赶不上变化

执行中发现计划不合理怎么办？比如计划查 5 个竞品，结果搜出来 50 个。

**解决方案**：加入 **Replanning Checkpoint**（重新规划检查点）

```
Plan → Execute(step1) → Check ──不达标──► Replan
                          │
                       达标
                          ▼
                    Execute(step2) → Check → ...
```

LangGraph 官方的 plan-execute 模板就是这样实现的。

> 📌 详见 [§七 Q1](#七q1plan-and-execute-中-execute-的每个-step-到底是什么) 关于 step 本质的深入讨论。

---

## 四、模式三：Reflection（自我反思）

### 核心思路

让一个 Agent 生成，另一个 Agent 审查，循环迭代直到质量达标。

```
Writer Agent ──► 草稿 ──► Reviewer Agent ──► 反馈
     ▲                                          │
     │                                          ▼
     └────────── 修改 ◄──────────── 不通过 ─────┘
                                        │
                                       通过
                                        ▼
                                    最终输出
```

### 用代码 Review 类比最容易懂

1. Writer Agent 写代码
2. Reviewer Agent 发现问题（安全漏洞、性能问题、命名不规范）
3. Writer Agent 根据反馈修改
4. Reviewer Agent 复审 → 通过 → 输出

### 适用场景

对**输出质量要求高**、**值得多花 Token 反复打磨**的任务：

- 代码生成
- 法律文书
- 学术论文
- 创意写作（小说、文案）

### 面试加分：Reflection 还能用来防幻觉

当 Agent 发现自己给出的事实存疑时，可以触发"验证反思"——调用搜索工具去核实，而不是盲目输出。

```
Writer: "Linux 内核是 1995 年发布的"
Reflector: "这个年份我不太确定，需要核实"
→ 调用 search("Linux kernel first release")
→ 结果："1991 年"
→ 修正输出
```

---

## 五、模式四：Multi-Agent（多智能体协作）

### 架构

```
              ┌──────────────────┐
              │   Orchestrator   │  ← 理解需求、分配任务、汇总结果
              │ (协调 Agent)     │
              └────────┬─────────┘
            ┌─────────┼──────────┬──────────┐
            ▼         ▼          ▼          ▼
       ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────┐
       │Research│ │ Coder  │ │Reviewer│ │   ...   │
       │ Agent  │ │ Agent  │ │ Agent  │ │         │
       └────────┘ └────────┘ └────────┘ └─────────┘
```

### 主流框架对比

| 框架 | 特点 |
|---|---|
| **LangGraph** | 图结构编排，状态机模型，精细控制 |
| **CrewAI** | 角色化 Agent，任务分工，上手简单 |
| **OpenAI Agents SDK** | 官方推出，handoff 机制，原生工具调用 |
| **AutoGen** | 微软出品，对话式多 Agent，研究型友好 |

### Anthropic 的提醒（必背）

> **不要过早引入 Multi-Agent。一个强大的单 Agent 往往比多个简单 Agent 协作更稳定、更省钱。**

只有任务明确需要**并行处理**或**专业分工**时才引入多 Agent。盲目上多 Agent，调试地狱等着你。

> 📌 详见 [§八 Q2](#八q2multi-agent-为什么要拆成多个-agent) 关于"为什么要拆 Agent"的深入讨论。

---

## 六、防止 ReAct 死循环（面试高频）

三个方法要能脱口而出：

| 机制 | 做法 |
|---|---|
| **最大步数限制** | 通常 `max_steps = 15`，超过强制终止 |
| **重复动作检测** | 连续 N（典型 3）次调用同一工具+同一参数，直接退出 |
| **超时控制** | 整个任务设置 wall clock 上限（如 60s） |

### 参考实现

```python
class ReActAgent:
    def run(self, task, max_steps=15):
        steps = 0
        seen_actions = []
        history = []

        while steps < max_steps:
            thought, action = self.llm_think(task, history)

            if action.type == "FINISH":
                return action.final_answer

            if seen_actions[-3:].count(action) >= 3:
                return self.llm_summarize(
                    "工具持续失败，基于已有信息给出答案", history
                )

            seen_actions.append(action)
            observation = self.execute(action)
            history.append({"thought": thought, "action": action, "obs": observation})
            steps += 1

        return "达到最大步数限制，任务终止"
```

> 生产中还会加：成本上限（累计 token 超过阈值终止）、人工 break-glass 接管按钮。

---

## 七、Q1：Plan-and-Execute 中 execute 的每个 step 到底是什么？

### 简短回答

> **每个 step 通常是「Executor LLM + 工具调用」的组合**，而不是纯人工预先写死的 API 调用。

也就是说，**LLM 仍然在 execute 阶段被反复调用**，只是它不再做"全局规划"，只做"按指令执行"。

### 三种实现方式（从轻到重）

| 实现方式 | 每个 step 是什么 | 何时使用 |
|---|---|---|
| ① **纯工具调用** | 只调一个固定 API | step 描述明确到工具+参数（罕见，且 Planner 必须知道工具签名） |
| ② **小型 ReAct**（最常见） | Executor LLM 看 step 描述，决定调哪个工具，执行后返回结果 | LangChain / LangGraph 默认实现 |
| ③ **子 Agent** | 每个 step 启一个完整子 Agent（带自己的 ReAct 循环） | 复杂任务，如「搜集 5 家竞品的财报」需要循环搜索 |

### 为什么 step 通常需要 LLM？

**反例分析**：如果 step 是纯人工 API，会出什么问题？

```python
# 假设 Planner 输出的 step 是纯函数调用
plan = [
    "search_web('iPhone 15 价格')",
    "search_web('Galaxy S24 价格')",
    ...
]
```

致命问题：
1. **Planner 必须知道所有工具的精确签名**——但 Planner 是 LLM，它写出来的字符串经常拼错参数名
2. **失去灵活性**——搜出来 50 条结果，谁来挑哪条最相关？必须有 LLM 做语义判断
3. **错误处理困难**——API 报错时谁来 retry / 换参数？必须有 LLM 决策
4. **结果加工**——搜索返回的是 HTML，谁来抽取价格？还是要 LLM

所以 **step 描述用自然语言**（例："搜集 iPhone 15 在京东、淘宝的价格"），由 **Executor LLM 在运行时翻译成工具调用**，才合理。

### LangGraph 官方 Plan-and-Execute 模板的实际架构

```
┌───────────────────────────┐
│  Planner LLM (GPT-4)      │  ← 大模型，做 一次 全局规划
│  output: List[str]        │
└──────────┬────────────────┘
           │
           ▼
   ┌──────────────┐
   │  for step in │
   │     plan:    │
   │              │
   │   ┌──────────┴──────────┐
   │   │ Executor (Sub-Agent)│  ← 小模型 + ReAct 循环
   │   │  - llm_decide_tool  │     可能调多次 LLM 完成一个 step
   │   │  - call_tool        │
   │   │  - observe          │
   │   └──────────┬──────────┘
   │              │
   │  collect step_result    │
   └────────┬────────────────┘
            │
            ▼
   ┌────────────────────┐
   │ Replanner LLM      │  ← 检查执行结果，必要时改计划
   │ (用于 checkpoint)  │
   └────────────────────┘
```

LLM 在每个阶段都在被调用：
- 规划用 1 次大模型
- 每个 step 内部可能调 1~N 次小模型
- 每次 checkpoint 调 1 次重新规划模型

### 一句话总结

> "Plan-and-Execute 省的不是 LLM 调用次数，而是**省掉 ReAct 中每步都要做的全局推理**——Executor 只在小范围内决策，所以可以用更小、更便宜的 LLM。如果 step 真的能写成纯 API，那就退化成 Workflow 了。"

---

## 八、Q2：Multi-Agent 为什么要拆成多个 Agent？

### 你的疑问

> "明明同一个 Agent 用不同的工作模式（如 Reflection 模式做代码 Review），就能实现不同效果，为什么要拆成多个 Agent？"

**这个疑问非常对！这正是 Anthropic 反复强调"不要过早引入 Multi-Agent"的原因**。
单 Agent + 多模式确实能解决很多问题，**只在以下 6 个场景**才真正需要拆成多 Agent。

### 场景 1：上下文窗口隔离（最重要！）

单个 Agent 处理复杂任务时，所有 thought / observation / 中间结果都堆在**同一个 context window** 里：

```
单 Agent context:
[系统 prompt]
[搜资料的 50 个结果……]      ← 占 30K token
[写代码的中间草稿……]        ← 占 20K token
[review 反馈……]             ← 占 10K token
[再次写代码……]              ← 上下文已经爆炸
```

**结果**：context 爆掉、成本飙升、模型注意力分散、回答质量下降。

**Multi-Agent 解法**：

```
Research Agent: 自己的 context（只有搜索相关）
Coder Agent:    自己的 context（只接收 Research 的结论摘要）
Reviewer Agent: 自己的 context（只看代码 + review 标准）
```

每个 Agent 的 context 只装它需要的东西。Anthropic 在 *How we built our Claude Research feature* 里明确把"context isolation"列为引入 Multi-Agent 的**首要理由**。

### 场景 2：专业化 prompt 提升质量

LLM 对**窄而深**的 prompt 表现更好：

| 单 Agent prompt | Multi-Agent 各自 prompt |
|---|---|
| "你是一个全能助手，会搜索、写代码、review 代码、写邮件……" | Researcher："你是一名调研专员，目标是找全、找准事实，不要做评价"<br>Coder："你是一名 Python 工程师，写出能通过 CI 的代码"<br>Reviewer："你是 OWASP 安全专家，专门挑安全漏洞" |
| 所有规则在一个 prompt 里互相干扰 | 每个 prompt 高度聚焦 |

**收益**：每个 Agent 在自己的领域更专业；prompt 更短，更不易"忘记"。

### 场景 3：差异化模型选择（省钱）

不同子任务对模型能力要求不同：

```
Multi-Agent 配置：
  Planner    → Claude Opus    （需要强推理）
  Researcher → GPT-4o-mini    （够用就行，调用次数最多，必须便宜）
  Coder      → Claude Sonnet  （代码强）
  Reviewer   → GPT-4o         （安全场景需要稳）
```

单 Agent 只能用一个模型，要么贵要么不够强。

### 场景 4：差异化工具/权限边界

不同 Agent 可以有完全不同的工具集和权限：

| Agent | 能用的工具 | 不能用的工具 |
|---|---|---|
| Reviewer | 读代码、扫漏洞 | ❌ 不能写文件 |
| Coder | 写文件、跑测试 | ❌ 不能调用生产数据库 |
| DBA | 调数据库 | ❌ 不能改代码 |

这是**最小权限原则**在 Agent 系统中的落地——出问题时影响面可控。

单 Agent 即使切换"模式"，工具仍是同一套，权限边界做不出来。

### 场景 5：真正的并行执行

Multi-Agent 可以同时跑：

```
            ┌─► Researcher A: 搜竞品 1
Orchestrator┼─► Researcher B: 搜竞品 2     ← 三个并行，节省 2/3 时间
            └─► Researcher C: 搜竞品 3
                       │
                       ▼
                  汇总 → 报告
```

单 Agent 切换模式只是改 prompt，**仍然是串行**——不能并行调 LLM。

### 场景 6：失败隔离与可观测性

- 一个 Agent 挂掉，其他继续
- 每个 Agent 独立日志、独立 trace、独立指标
- 单 Agent 的一次"决策错误"会污染整个 history

### 那什么时候应该用单 Agent + 多模式？

**绝大多数场景！** 包括你说的代码 Review：

| 单 Agent + Reflection 模式 | 多 Agent (Writer + Reviewer) |
|---|---|
| 同一个 Agent 先写后改 | Writer 写、Reviewer 审 |
| Context 完整保留（"我之前为什么这么写"） | Reviewer 拿不到 Writer 的思考过程，可能误判 |
| 简单、便宜、易调试 | 多一倍 LLM 调用、多一份 prompt 维护成本 |

**对于个人开发者写小工具的代码 Review 场景，单 Agent + Reflection 完全够用，不需要拆 Multi-Agent。**

只有当出现以下情况之一时，再考虑拆 Agent：

| 触发条件 | 例子 |
|---|---|
| 上下文已经爆炸 | 任务涉及检索几十个文档 |
| 需要权限隔离 | Reviewer 不应该能改代码 |
| 子任务可以并行 | 同时分析 10 家竞品 |
| 需要不同模型 | 调研用便宜模型、决策用强模型 |
| Prompt 已经超过 1000 行 | 一个 prompt 写不下所有规则 |

### 经验法则（面试可以背）

> 1. **能用单 Agent 就别用 Multi-Agent**
> 2. **能用 Workflow 就别用 Agent**
> 3. **能不用 LLM 就别用 LLM**
>
> 这条"奥卡姆剃刀三连"中，**前两条是 Anthropic 在 *Building effective agents*（2024.12）中的明确建议**——原文是 "find the simplest solution possible, and only increase complexity when needed. This might mean not building agentic systems at all"。**第三条是工程界对这一思想的自然延伸**，不是 Anthropic 的原话，但与其精神一致。TODO：将能不用LLM就不用LLM相关的话语删除，说的不对

#### 为什么"能不用 LLM 就别用 LLM"？

LLM 看上去万能，但每次调用都带着一身"额外成本"：

| 维度 | LLM | 普通代码（规则 / 正则 / 字典 / SQL） |
|---|---|---|
| **金钱成本** | 每次调用按 token 收费，规模化后是大头 | 几乎为零 |
| **延迟** | 一次调用 100ms~10s，串成 Agent 后秒级起步 | 微秒~毫秒级 |
| **确定性** | 同输入可能输出不同结果（temperature>0 时） | 完全确定，可单元测试 |
| **幻觉风险** | 会编造看似合理但错误的内容 | 不会编造 |
| **可调试性** | 黑箱，错了只能改 prompt 试 | 单步调试、可断言、可回归 |
| **可监控** | 需要专门的 LLM-Ops 平台监控漂移、滥用 | 普通 APM 即可 |
| **数据合规** | 输入会进 LLM 厂商日志/训练，敏感数据要脱敏 | 完全在自己进程内 |
| **冷启动** | 依赖外部 API，挂了你也挂 | 无外部依赖 |

**判断准则**：

- 能用 **正则 / 字典查询 / SQL** 解决的——别上 LLM（如手机号校验、城市名归一化）
- 能用 **小型分类模型（fastText、BERT-tiny）** 解决的——别上 LLM（如意图分类、敏感词识别）
- 能用 **预定义模板填空** 解决的——别上 LLM（如发账单邮件、节假日问候）
- 只在以下情况上 LLM：
  1. 输入是**开放自然语言**且没有规则可循
  2. 输出需要**创造性 / 推理**
  3. 任务变化太快，写规则跟不上

**反例（典型踩坑）**：

```python
# ❌ 大材小用：用 LLM 判断用户输入是不是手机号
is_phone = llm.ask(f"'{text}' 是手机号吗？回答 yes/no")
# 成本：每次几分钱 + 几百毫秒延迟 + 偶尔答错

# ✅ 一行正则搞定
is_phone = bool(re.fullmatch(r"1[3-9]\d{9}", text))
# 成本：几微秒 + 100% 准确
```

### 一句话总结

> "拆 Agent 的核心动机不是'功能不同'（功能不同用模式切换就行），而是 **context、权限、并行、模型、可观测性**这五个维度的隔离需求。'Reflection 模式做代码 Review' 在大多数场景下就足够了；只有当上面这五条至少踩中一条时，才升级到 Multi-Agent。"
