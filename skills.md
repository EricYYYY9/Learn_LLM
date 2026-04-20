# Skills 全解：本质、设计、与 Prompt/MCP/Function Call 的协作

> 面试常问：
> - **「Skills 和 System Prompt 有什么区别？你会怎么设计一个 Skill？」**
> - **「Function Call、MCP、Skills 都是让 Agent 能干更多事，能用一个统一的比喻讲清楚吗？」**
>
> 本文先校对原文，再讲清 Skills 的本质，最后用一个完整链路把三者协作打通。

---

## 目录

- [一、原文校对总览](#一原文校对总览)
- [二、Skills 是什么？解决什么问题？](#二skills-是什么解决什么问题)
- [三、Skills vs System Prompt（修正版对比表）](#三skills-vs-system-prompt修正版对比表)
- [四、Skill 文件示例（带分项解读）](#四skill-文件示例带分项解读)
- [五、Skills 的激活机制](#五skills-的激活机制)
- [六、Skills vs Few-shot vs RAG —— 三个相邻概念的边界](#六skills-vs-few-shot-vs-rag-三个相邻概念的边界)
- [七、Function Call / MCP / Skills 三维对比（修正版）](#七function-call--mcp--skills-三维对比修正版)
- [八、三者协作的完整流程（修正版）](#八三者协作的完整流程修正版)
- [九、面试一句话总结](#九面试一句话总结)

---

## 一、原文校对总览

原文整体方向**正确**，技术比喻很到位。下面几处建议增强或修正：

| 原文 | 状态 | 需要修正的地方 |
|---|---|---|
| Skills 解决"领域专家经验编码" | ✅ 准确 | 补一句"Skills 不是行业标准术语"——主要来自 Anthropic Claude Skills（2024 推出）等少数框架，LangChain / LangGraph 里没有同名概念，是用 Tools + Prompts + RAG 组合实现的 |
| "Skills 教方法论，Few-shot 教格式" | ✅ 比喻好 | 补充：Skills 本质上**仍是 prompt 注入**，区别在"模块化 + 按需激活 + 结构化领域知识" |
| Skills vs System Prompt 表 | ✅ 准确 | 加一行"是否模块化" |
| Skill 文件 YAML + Markdown 格式 | ✅ 准确 | 这是 Claude Skills 的实际格式，别的框架（CrewAI Agent / LangChain Prompt Hub）格式不同 |
| 激活机制 | ⚠️ 略简 | 实际有 4 种：**关键词匹配 / 向量检索 / 显式调用 / LLM 自选**——关键词只是其中一种 |
| 三维对比表"运行位置 - 你的应用程序" | ⚠️ 表述模糊 | 改为 "**LLM API 调用过程中**"更准确 |
| "Agent 规划（受 Skill 指导）" | ⚠️ 不严谨 | **是 LLM 在规划**，Skill 只是 prompt 内容，不会自己规划 |
| "Skills → MCP → Function Call" 顺序 | ⚠️ 容易误读 | 这是**抽象层次**顺序，不是**运行时执行**顺序（详见 [§八](#八三者协作的完整流程修正版)） |

---
 
## 二、Skills 是什么？解决什么问题？

### 出发点：光有工具（MCP）还不够

Agent 通过 MCP 拿到了一堆工具，但仍然不知道：

- **代码审查**该用什么标准（先看安全？先看性能？）
- **写 SQL** 时 DBA 的最佳实践（必须参数化、禁止 `SELECT *`）
- **回复客户**时品牌的语气要求（克制？亲切？正式？）

这些**领域专家经验**就是 Skills 要编码的东西。

### 一句话定义

> **Skill = 一段结构化的领域知识 prompt，可按需激活，模块化维护。**

> ⚠️ **澄清术语来源**：
> "Skills" 不是一个全行业标准术语。它主要来自：
> - **Anthropic Claude Skills**（2024 推出，YAML front matter + Markdown 格式）
> - **CrewAI** 的 Agent Role 定义
> - 其它框架可能叫 "Persona"、"Behavior Pack"、"Knowledge Pack"
>
> 在 LangChain / LangGraph 里没有显式叫 Skills 的对象，但用 **Tools + Custom Prompts + RAG** 的组合可以实现完全等价的能力。

### Skills 解决的核心痛点

| 痛点 | Skills 怎么解 |
|---|---|
| 领域知识散落各处难管理 | 一个 `.md` 文件 = 一份完整领域知识 |
| System prompt 越写越长 | 按需激活，不用每次都加载所有规则 |
| 多个领域规则互相冲突 | 模块化隔离，各自独立 |
| 团队协作改 prompt 易出错 | 文件粒度评审，可 git diff，可回归测试 |

---

## 三、Skills vs System Prompt（修正版对比表）

| 维度 | System Prompt | Skills |
|---|---|---|
| **作用范围** | 全局，每次都生效 | 按需激活，场景触发 |
| **内容** | 通用行为规范 | 特定领域专业指导 |
| **加载方式** | 每次请求都注入 | 匹配场景才注入 |
| **模块化** | ❌ 一坨字符串，难拆分 | ✅ 一个领域一个文件 |
| **可维护性** | 随功能增多越来越乱 | 模块化，各自独立可测试 |
| **本质** | Prompt 字符串 | 仍是 Prompt，但有元数据 + 触发条件 |
| **典型内容** | "你是一个助手，要礼貌、专业……" | 完整的代码审查 SOP / SQL DBA 守则 / 客服话术库 |

> 💡 **关键认知**：Skills 在底层**仍然是 prompt**——区别只在"何时注入"和"怎么组织"。Skills 不是新协议、新 API，它是一套**对 prompt 工程的约定 + 工程化管理方式**。

---

## 四、Skill 文件示例（带分项解读）

下面是一个完整的 Code Review Skill，结构 = **YAML 元数据头 + Markdown 内容体**：

```markdown
---
name: Senior_Code_Reviewer
description: 当用户要求进行代码审查时激活此技能
triggers:
  - "帮我 review 代码"
  - "code review"
  - "检查这段代码"
allowed-tools:
  - read_file
  - search_codebase
  - run_linter
---

# 角色定位
你是有 10 年经验的资深后端架构师，对代码质量有极高要求。

# 审查维度（必须全部覆盖）

## 1. 安全性（最高优先级）
- SQL 注入风险
- 越权访问漏洞
- 敏感信息硬编码（密码 / 密钥）
- 反序列化安全

## 2. 性能
- N+1 查询问题
- 未释放的资源
- 不必要的重复计算
- 缓存策略是否合理

## 3. 代码质量
- 单一职责原则
- 方法长度不超过 50 行
- 命名是否准确表意
- 注释是否必要且准确

# 输出格式
必须输出 Markdown 报告，包含：
1. 总体评分（1-10 分）
2. 严重问题（必须修复）
3. 建议优化（非强制）
4. 每个问题必须附代码示例（原始 vs 修复）

# 语气要求
专业、直接，不废话。发现问题就直说，别用"可能""也许"这类模糊表达。
```

### 五大组成部分

| 块 | 作用 |
|---|---|
| **YAML `name`** | Skill 唯一标识 |
| **YAML `description`** | 让 LLM 决定"该不该激活我" |
| **YAML `triggers`** | 关键词匹配（粗筛） |
| **YAML `allowed-tools`** | 权限白名单——这个 Skill 激活时只能用这几个工具 |
| **Markdown body** | 真正的领域知识：角色 / 工作流程 / 注意事项 / 输出规范 |

> 💡 **设计哲学**：身份定位 + 工作流程 + 注意事项 + 输出规范——比 Few-shot 给几个示例强得多。**Few-shot 教的是格式，Skills 教的是方法论。**

---

## 五、Skills 的激活机制

> 原文只提了"关键词匹配 + 语义相似度"，实际**至少有 4 种**激活方式，生产中通常是组合使用：

| 激活方式 | 怎么触发 | 优点 | 缺点 |
|---|---|---|---|
| **关键词匹配** | `triggers` 列表里的词命中 | 快、可解释 | 死板，需要人工维护词表 |
| **向量检索** | 用户输入向量化后，与 Skill `description` 向量算 cosine 相似度 | 灵活，能处理同义改写 | 有阈值要调，可能误激活 |
| **显式调用** | 用户写 `/code-review` 或 Skill 在工具菜单里被点选 | 100% 准确 | 用户得知道有这个 Skill |
| **LLM 自选** | 把所有 Skill 的 `description` 列表给 LLM，让它自己决定激活哪个 | 最智能，能组合多个 Skill | 多一次 LLM 调用，有成本 |

### 激活流程示意

```
用户输入
   │
   ▼
扫描所有可用 Skills (取它们的 description + triggers)
   │
   ├─► 关键词匹配命中？  ──是──► 激活
   │
   ├─► 向量相似度 > 阈值？──是──► 激活
   │
   ├─► 用户显式 /skill 调用？──是──► 激活
   │
   └─► LLM 决策选中？      ──是──► 激活

一个或多个 Skill 被激活
   │
   ▼
把它们的 Markdown body 注入到 LLM 的 system prompt 里
   │
   ▼
LLM 按 Skill 指导执行任务
```

> 多个 Skill 可同时激活（Code Review Skill + Python Style Skill 一起注入）。

---

## 六、Skills vs Few-shot vs RAG —— 三个相邻概念的边界

这三个都"往 LLM 里塞额外信息"，面试容易问"你怎么选"，要会区分。下面分别看长什么样、怎么对比、怎么选。

### 6.1 Few-shot 长什么样子

**核心机制**：把 N 个 input/output 示例对**写进 prompt 模板**，LLM 通过"模仿"完成同类任务。

完整示例（情感分类）：

```
你是情感分类器。请按下面例子的格式输出。

【示例 1】
输入：这家餐厅菜品超棒！
输出：positive

【示例 2】
输入：服务态度太差了，再也不来。
输出：negative

【示例 3】
输入：环境一般，价格偏高。
输出：neutral

【现在】
输入：{{user_input}}
输出：
```

要点：
- **每次请求都带着这一坨示例**——即使用户问的是无关问题
- 示例是**人工写死**的，不会动态变
- **本质是 In-context Learning**（上下文学习），不修改模型权重
- **典型用途**：分类 / 抽取 / 翻译 / 改写——这类**有明确格式约束**的任务

### 6.2 RAG 长什么样子

**核心机制**：query 来了 → 从知识库**动态检索**相关内容 → 拼进 prompt → LLM 基于检索结果回答。

完整链路（伪代码）：

```python
# === 离线：建索引 ===
docs = split("公司内部文档.md")
vector_db.upsert([(d, embed(d)) for d in docs])


# === 在线：用户查询时 ===
def answer(query):
    q_vec = embed(query)
    chunks = vector_db.search(q_vec, top_k=3)

    prompt = f"""请基于下面的文档回答问题，不要编造。

【文档片段】
{chunks}

【问题】
{query}
"""
    return llm(prompt)
```

要点：
- 注入的内容**每次都不一样**（取决于检索到什么）
- **本质是用外部知识库扩展 LLM 的"工作记忆"**
- 解决 LLM 两大短板：① 训练数据有截止时间，不知道最新信息 ② 私有数据没在训练里
- **典型用途**：知识问答 / 客户资料查询 / 实时数据查询

#### RAG 和 MCP 是不是很像？

> 表面看都是"从外部拿东西塞给 LLM"，但**目的、协议、触发时机、副作用**完全不同。一句话总结：**RAG 是"找资料"，MCP 是"派活/查询"**。

| 维度 | RAG | MCP |
|---|---|---|
| **核心目的** | 检索**知识 / 事实片段** | 调用**工具 / 服务** |
| **能做什么** | 只读：查知识、查文档 | 读 + 写：查数据、发邮件、改数据库、调 API… |
| **返回内容** | 非结构化文本片段（一段段文字） | 结构化数据 / 操作结果（JSON） |
| **协议** | 向量检索（embedding + cosine similarity） | JSON-RPC 2.0（标准化 RPC 协议） |
| **触发时机** | **LLM 推理之前**：应用层先检索，把结果拼进 prompt | **LLM 推理中**：LLM 输出 Function Call → Host 路由到 Server |
| **谁决定调** | 应用代码（"用户问问题就先去检索一下"） | LLM 自己（"我需要哪个工具，自己输出 tool_call"） |
| **副作用** | 无（只读） | 可能有（取决于工具：发邮件、转账等） |
| **典型场景** | 知识问答、客服文档检索、产品手册查询 | 改 Jira、查数据库、发 Slack、读文件 |
| **替代方案** | 也可以包装成 MCP Server 提供 `search_knowledge_base` 工具 | 也可以用直接 SDK / HTTP 调用代替 |

**关键区别用一句话**：

- **RAG** 是**「先检索后推理」**——应用层先去知识库捞内容塞 prompt，LLM 看着这些片段写答案
- **MCP** 是**「推理中调用」**——LLM 边想边决定调哪个工具，工具执行后结果再喂回 LLM 继续推理

**两者会重合的情况**：

> 当你**把 RAG 包装成一个 MCP Server**（提供 `search_knowledge_base(query)` 工具）时，它确实"披着 MCP 外衣"。但这只是**接入方式**变了——RAG 的本质（检索知识）没变。
>
> 区别在于：
> - **传统 RAG**：每次用户提问，应用层**强制**先检索再调 LLM
> - **RAG-as-MCP-Tool**：LLM**自己判断**是否需要查知识库，需要才调

**协作而非替代**：

实际项目里，RAG 和 MCP 经常一起用——比如客服 Agent：
- **RAG**：每次用户提问先检索 FAQ 库，把相关 FAQ 片段塞 prompt
- **MCP**：LLM 决定要查订单状态时，通过 `order_query` MCP 工具实时查
- **Skills**：客服话术 SOP 告诉 LLM "先查 FAQ，FAQ 没答案再查订单系统"

### 6.3 Skills 长什么样子（回顾）

详见 [§四 Skill 文件示例](#四skill-文件示例带分项解读)。核心特征：

- **结构化的方法论文档**（YAML 元数据 + Markdown 正文）
- **匹配场景才注入**（不是每次都加载）
- **教 LLM 怎么思考、怎么决策、按什么标准输出**

### 6.4 三者深入对比

| 维度 | Few-shot | Skills | RAG |
|---|---|---|---|
| **prompt 里塞什么** | 几个 input/output 示例对 | 一份完整方法论 .md 文档 | 实时检索到的事实片段 |
| **注入时机** | **每次请求都注入**（写死在模板里） | **匹配场景才注入**（trigger / 向量 / LLM 自选） | **每次查询都动态检索后注入** |
| **教什么** | **格式**（按这样输出） | **方法**（按这流程思考） | **事实**（这是当前真实数据） |
| **内容来源** | 静态人工写的几个例子 | 静态人工写的领域 SOP | 动态从向量库检索 |
| **会不会过时** | ❌ 不会（教模式） | ❌ 不会（教方法） | ✅ 会（事实在变，必须及时同步知识库） |
| **Token 占用** | 小（几个示例） | 中（一份完整文档） | 中-大（top-k 文档片段） |
| **依赖外部系统** | 无 | 无（本地 .md 文件） | 需要向量库 + embedding 模型 |
| **怎么改** | 改示例（重写 prompt 模板） | 改 .md 文件 | 改知识库内容（不动 prompt） |
| **典型应用** | 分类 / 抽取 / 翻译这类有明确格式的任务 | 代码审查 / 客服话术 / 财务审查这类需要专业判断的任务 | 知识问答 / 客户资料查询 / 实时数据查询 |

### 6.5 决策树：什么时候用谁？

```
你要解决的问题是？
│
├─► 任务简单，主要是"按特定格式输出"
│       → 用 Few-shot（最轻量，加几个例子就行）
│
├─► 任务需要"领域专家的方法论 / 工作流程"
│       → 用 Skills（把 SOP 写成 .md 文件，按需激活）
│
├─► 任务需要"实时的、会变的事实数据"
│       → 用 RAG（向量库 + 检索）
│
└─► 三者结合（生产中最常见）
        → Skills 教方法 + Few-shot 给输出范例 + RAG 拉实时知识
        例：合同审查 Agent
        - Skill 告诉 LLM "怎么审一份合同"（流程 / 维度 / 优先级）
        - Few-shot 告诉 LLM "审查报告写成啥样"（输出格式范例）
        - RAG 拉来 "公司最新法务条款 + 历史相似合同"（事实依据）
```

### 6.6 一句话精髓

> **Few-shot 是「示范」，Skills 是「教程」，RAG 是「资料库」**——示范让 LLM 模仿，教程让 LLM 思考，资料库让 LLM 不胡编。三者互补不互斥，生产中常组合使用。

---

## 七、Function Call / MCP / Skills 三维对比（修正版）

| 维度 | Function Call | MCP | Skills |
|---|---|---|---|
| **解决问题** | LLM 如何把"想调函数"的意图说给应用 | 应用如何标准化接入一堆工具 | 把领域专家经验编码进 prompt |
| **本质** | LLM API 输出格式协议（JSON Schema） | 应用 ↔ 工具服务的 RPC 协议（JSON-RPC 2.0） | 模块化的 Prompt 工程约定 |
| **运行位置** | LLM API 调用过程中 | 独立的 MCP Server 进程 | LLM 上下文窗口（system prompt 区） |
| **是否调外部** | ❌ 自己不调，只输出意图 | ✅ 调外部 Server | ❌ 不调任何东西 |
| **标准化程度** | 各厂商格式不统一（OpenAI / Claude / Gemini 各有差异） | 开放统一标准（Anthropic 主导） | 无业界统一标准（各框架自定义） |
| **何时生效** | LLM 每次调用 | 工具被调用时 | 被激活注入到 prompt 时 |
| **谁去管理** | LLM 厂商 + 应用代码 | 工具厂商（提供 Server） | 业务团队（写 .md） |
| **生活类比** | 打电话的基础能力 | 公司通讯录 + 电话系统 | 岗位培训手册 |

> ⚠️ **修正原文表述**：
> - 原表 "Function Call 运行位置 = 你的应用程序" 其实模糊——更准确是 "LLM API 调用过程中"
> - 原表 "Skills 解决领域知识编码" 准确，但要补一句"Skills 不是新技术，是 prompt 工程的工程化包装"

---

## 八、三者协作的完整流程（修正版）

> ⚠️ **关键澄清**：很多文章给的总结 "**Skills → MCP → Function Call**" 其实是**抽象层次**顺序，**不是运行时执行顺序**。下面分两个视角讲清。

### 视角 1：抽象层次（从高层认知到底层协议）

```
Skills（怎么想） ─► MCP（用什么） ─► Function Call（怎么调）
   高层认知层       中层工具市场       底层调用协议
```

### 视角 2：运行时执行顺序（一次实际请求的时间线）

```
Skills（思考激活） ─► Function Call（LLM 输出意图） ─► MCP（路由到 Server 执行）
```

> 注意 **Function Call 在 MCP 之前**——LLM 永远先用 Function Call 表达"我要调什么"，Host 才能根据这个名字去 MCP Server 列表里找谁来执行。

### 完整链路（用户："帮我审查 agent.py 这个文件"）

| 步骤 | 涉及层 | 谁参与 | 发生了什么 |
|---|---|---|---|
| ① | **Skills 激活** | Host | 检测到"审查"关键词 + 向量检索匹配 → 激活 `Senior_Code_Reviewer` Skill → 把它的 Markdown body 注入 system prompt |
| ② | **构造 Prompt** | Host | system prompt = 通用 prompt + Skill body；user prompt = "帮我审查 agent.py"；tools = 从所有 MCP Server 拉到的工具列表（含 `read_file`、`run_linter` 等） |
| ③ | **LLM 决策第 1 步** | Host ↔ LLM | LLM 看了 Skill 的"先看安全 / 性能 / 质量"指引 → 决定先读文件 → 用 **Function Call** 输出 `tool_calls: [{name: "read_file", args: {path: "agent.py"}}]` |
| ④ | **MCP 执行** | Host ↔ filesystem MCP Server | Host 通过 MCP 协议调 filesystem Server → 拿到文件内容 → 通过 Function Call 把内容用 `role: "tool"` 回传 LLM |
| ⑤ | **LLM 决策第 2 步** | Host ↔ LLM | LLM 接着用 **Function Call** 输出 `tool_calls: [{name: "run_linter", args: {...}}]` |
| ⑥ | **MCP 执行** | Host ↔ linter MCP Server | 类似 ④，调 linter Server → 回传 lint 结果 |
| ⑦ | **LLM 综合输出** | Host ↔ LLM | LLM 按 Skill 指定的 "Markdown 报告 + 安全 → 性能 → 质量 顺序 + 不用'可能/也许'" 生成最终评审报告 |

### 三层各自的角色

| 层 | 在这个流程中的角色 | 没有它会怎样 |
|---|---|---|
| **Skills** | 在 ① ⑦ 决定"怎么想 / 怎么写报告" | LLM 没了方法论，只能按通用知识泛泛点评 |
| **Function Call** | 在 ③ ⑤ ⑦ 让 LLM 跟应用沟通调用意图 | LLM 没法表达"我要调工具" |
| **MCP** | 在 ④ ⑥ 让应用调到 filesystem / linter | 应用得自己手写 filesystem / linter 集成代码 |

---

## 九、面试一句话总结

### Q1：Skills 和 System Prompt 有什么区别？

> "Skills 本质上仍然是 prompt——区别在于**它是模块化、可按需激活、有元数据约束（trigger / allowed-tools）的领域知识包**。System Prompt 是全局每次都加载的'通用守则'，越写越长越乱；Skills 是'按需激活的专业守则'，每个领域一个 .md 文件，可独立评审、回归测试、版本管理。本质区别不是技术层面（都是 prompt 注入），而是**工程化管理方式**——Skills 把 prompt 工程从'一坨字符串'升级成'可工程化维护的模块系统'。"

### Q2：怎么用一个比喻讲清 Function Call / MCP / Skills？

> "用'新员工入职'类比：
> - **Function Call** 是'打电话的基础能力'——员工得会按电话键
> - **MCP** 是'公司统一的通讯录 + 电话系统'——员工不用挨个去问每个部门的电话，查通讯录就行
> - **Skills** 是'岗位培训手册'——员工知道遇到不同问题该怎么想、找谁、用什么标准
>
> 三者**抽象层次**是 Skills（怎么想）→ MCP（用什么）→ Function Call（怎么调），从高到低；
> 但**运行时**顺序是 Skills（思路）→ Function Call（输出调用意图）→ MCP（路由执行），因为 LLM 永远先通过 Function Call 表达意图，Host 才知道要去 MCP Server 列表里找谁。"

### Q3：怎么设计一个 Skill？

> "一个完整 Skill 由两部分组成：
> 1. **YAML 元数据头**：name（标识）、description（让 LLM 决策是否激活）、triggers（关键词粗筛）、allowed-tools（权限白名单）
> 2. **Markdown 内容体**：角色定位 + 工作流程（多维度方法论）+ 注意事项 + 输出规范
>
> 关键设计原则：**教方法论而非格式**（这是 Skills 与 Few-shot 的本质区别）；**单一职责**（一个 Skill 只覆盖一个领域）；**写明输出格式**（让 LLM 输出可解析）；**配 allowed-tools**（防止激活时滥用工具）。
>
> Skill 文件应该当作代码管理：Git 版本控制、PR 评审、A/B 测试、效果监控。"
