# A2A 协议全解：Agent ↔ Agent 通信、与 MCP 的协作

> 面试常问：
> - **「MCP 和 A2A 都是协议，它们有什么区别？」**
> - **「为什么 MCP 解决不了 A2A 要解决的问题？」**
>
> 本文先校对原文，再讲清 A2A 的核心概念、与 MCP 的关系，最后用一个完整链路把"工具调用 + Agent 协作"串起来。

---

## 目录

- [一、原文校对总览](#一原文校对总览)
- [二、为什么需要 A2A？](#二为什么需要-a2a)
- [三、A2A 核心概念（修正版）](#三a2a-核心概念修正版)
- [四、A2A 通信流程](#四a2a-通信流程)
- [五、MCP vs A2A：纵向 vs 横向](#五mcp-vs-a2a纵向-vs-横向)
- [六、完整链路示例：竞品分析报告 Agent](#六完整链路示例竞品分析报告-agent)
- [七、A2A 技术基础与生态现状](#七a2a-技术基础与生态现状)
- [八、面试一句话总结](#八面试一句话总结)

---

## 一、原文校对总览

原文整体方向**正确**——A2A 确实是为解决 Agent ↔ Agent 通信、与 MCP 互补、由 Google 主推。下面是需要修正或增强的细节：

| 原文表述 | 状态 | 需要修正的地方 |
|---|---|---|
| "MCP 解决 Agent ↔ 工具，A2A 解决 Agent ↔ Agent" | ✅ 准确 | 这是核心定位，对的 |
| "Agent Card 是 JSON 描述文件" | ✅ 准确 | 补充：通常发布在 **`/.well-known/agent.json`** 这个标准位置（类似 robots.txt 的约定） |
| "Task 生命周期：CREATED → PROCESSING → COMPLETED / FAILED" | ⚠️ 不准确 | A2A 实际定义了 **8 种状态**：`submitted` / `working` / `input-required` / `completed` / `canceled` / `failed` / `rejected` / `auth-required`——比简单四态丰富得多，特别 `input-required` 是 A2A 的关键能力（任务执行中可向用户/上游 Agent 反问） |
| "Message 沟通 / Artifact 是成果" | ✅ 准确 | 补充：Message 包含 `parts`（文本/文件/结构化数据 多模态），Artifact 也是 `parts` 组合 |
| "技术基础：HTTP + JSON + SSE" | ⚠️ 略简 | 准确说是 **HTTP + JSON-RPC 2.0**（同步请求）**+ SSE**（流式输出）**+ Webhook/Push Notification**（长任务异步通知） |
| "MCP 推动者 Anthropic" | ✅ 但原文未提 | A2A 推动者：**Google 主导发起（2024.4），2025 年捐给 Linux Foundation**，目前是开放治理模式 |
| "2026 年 A2A 早期阶段" | ⚠️ 时间需更新 | 2026 年 A2A 已经在 Linux Foundation 治理下，规范进入 1.x 稳定版，主流 Agent 框架（LangGraph / CrewAI / AutoGen 等）都有适配 |
| "Agent A 不知道 Agent B 是 LangGraph 做的" | ✅ 比喻好 | 这就是 A2A 解决的"框架异构"问题——A2A 是协议层，与 Agent 内部实现解耦 |

---

## 二、为什么需要 A2A？

### MCP 解决的是 Agent ↔ 工具，没有解决 Agent ↔ Agent

回顾 MCP 的设计目标（详见 [mcp.md](./mcp.md)）：

> MCP = 让 Agent 标准化地接入**外部工具/资源**（数据库、文件系统、API 等）。

但当一个 Agent 想**让另一个 Agent 帮忙做子任务**时，MCP 完全没设计解决：

| 痛点 | MCP 能解吗？ | 为什么不行 |
|---|---|---|
| 不知道 Agent B 有什么能力 | ❌ | MCP Server 描述的是"工具列表"，不是"Agent 能力 + 角色" |
| 不知道怎么给 Agent B 发任务 | ❌ | MCP 是 stateless 的工具调用（一来一回），不支持**长任务/分阶段产出** |
| 不知道 Agent B 完成没有 | ❌ | MCP 没有"任务生命周期"概念 |
| 任务执行中 Agent B 想反问 | ❌ | MCP 不支持双向交互式协商 |
| Agent A 是 CrewAI / Agent B 是 LangGraph | ❌ | 框架内部协议各搞各的，无统一对接标准 |
| Agent B 处于另一个组织/公司 | ❌ | MCP 没设计身份认证 + 跨组织授权 |

### A2A 设计目标

> **A2A = 让任意两个 Agent（可能跨框架、跨组织）能像两个微服务一样标准化协作的协议。**

类比：
- **MCP** 像 USB-C——你的电脑（Agent）连各种外设（工具）的统一接口
- **A2A** 像 HTTP——任意两台机器（Agent）之间通信的统一协议

两者**完全互补**，不冲突。

---

## 三、A2A 核心概念（修正版）

### 3.1 Agent Card —— 智能体名片

每个 A2A 兼容的 Agent 都要发布一份 JSON 描述文件，**通常放在 `https://<agent-host>/.well-known/agent.json`**（这是 A2A 规范约定的标准位置）。

完整示例：

```json
{
  "name": "Research Agent",
  "description": "擅长技术调研、收集主流框架信息、产出结构化调研报告",
  "url": "https://research-agent.example.com/a2a",
  "version": "1.2.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "stateTransitionHistory": true
  },
  "authentication": {
    "schemes": ["bearer"]
  },
  "skills": [
    {
      "id": "tech-research",
      "name": "技术框架调研",
      "description": "给定主题，搜集主流方案、对比、生成 Markdown 报告",
      "inputModes": ["text"],
      "outputModes": ["text", "file"]
    }
  ]
}
```

要点：
- **`/.well-known/agent.json`**：标准发现路径，类似 `robots.txt` 的约定
- **`capabilities`**：声明这个 Agent 支不支持流式输出、Push 通知、状态历史
- **`authentication`**：声明认证方式（bearer / OAuth2 / API key 等）
- **`skills`**：列出 Agent 能做什么（注意：这里的 skills 是 A2A 协议定义的能力声明，**与 [skills.md](./skills.md) 里的 Claude Skills 不是同一概念**）

### 3.2 Task —— 标准化任务对象（修正生命周期）

原文给的 "CREATED → PROCESSING → COMPLETED / FAILED" **不准确**。A2A 实际定义了 **8 种状态**：

```
                 ┌──────────────► canceled
                 │
                 ├──────────────► failed
                 │
                 ├──────────────► rejected
                 │
   submitted ────► working ──────► completed
       │             ▲│
       │             │▼
       │         input-required ─► (用户/上游 Agent 输入)
       │
       └─────────► auth-required ─► (补充认证后回到 working)
```

| 状态 | 含义 |
|---|---|
| **`submitted`** | 任务已提交，待开始 |
| **`working`** | 处理中 |
| **`input-required`** | ⭐ **执行中需要补充输入**（可向调用方反问）—— 这是 A2A 区别于普通 RPC 的核心能力 |
| **`auth-required`** | 需要认证/授权 |
| **`completed`** | 成功完成 |
| **`canceled`** | 被取消 |
| **`failed`** | 执行失败 |
| **`rejected`** | 被 Agent 拒绝（能力不匹配 / 权限不足） |

> ⭐ **`input-required` 是关键**：A2A 任务可以是**长任务 + 双向交互**的，比如 Research Agent 调研中发现"这个主题太宽泛，请缩窄范围"，可以转成 `input-required` 反问编排 Agent，而不是直接 fail。

### 3.3 Message vs Artifact

| | Message | Artifact |
|---|---|---|
| **作用** | 任务过程中的沟通（思考、反问、进度） | 任务最终的产出物 |
| **结构** | 由多个 `parts` 组成 | 由多个 `parts` 组成 |
| **`parts` 类型** | TextPart / FilePart / DataPart（结构化数据） | 同上 |
| **典型场景** | "我开始调研了"、"需要再确认一下范围" | 最终的调研报告 .md / 数据表 .csv |

> Message 和 Artifact 都用统一的 `parts` 结构 → 天然支持**多模态**：文字 + 文件 + 结构化数据混合。

---

## 四、A2A 通信流程

### 标准 7 步

```
① 编排 Agent 接到用户需求
       │
       ▼
② 通过 Agent Registry / DNS 找到候选 Agent
       │
       ▼
③ 拉取候选 Agent 的 /.well-known/agent.json
       │
       ▼
④ 根据 skills + description 选定 Agent B
       │
       ▼
⑤ 通过 A2A 协议（HTTP + JSON-RPC）创建 Task
       │  POST https://agent-b/a2a
       │  { "method": "tasks/send", "params": { ... } }
       ▼
⑥ Agent B 处理任务（状态：working）
       ├─► 中间通过 SSE 流式回传 Message（进度/思考）
       ├─► 必要时转 input-required 反问
       │
       ▼
⑦ 完成后返回 Artifact，状态变 completed
```

### 关键交互模式

A2A 支持**3 种交互模式**：

| 模式 | 用什么 | 适用场景 |
|---|---|---|
| **同步请求-响应** | HTTP + JSON-RPC `tasks/send` | 短任务，几秒内完成 |
| **流式（SSE）** | `tasks/sendSubscribe` | 中等任务，需要看进度/中间产出 |
| **异步推送** | Webhook + Push Notification | 长任务（几分钟到几小时），断开连接后通过回调通知 |

---

## 五、MCP vs A2A：纵向 vs 横向

| 维度 | MCP | A2A |
|---|---|---|
| **解决问题** | Agent ↔ **工具** 的标准化接入 | Agent ↔ **Agent** 的标准化协作 |
| **方向比喻** | **纵向**（一个 Agent 向下连多个工具） | **横向**（多个 Agent 之间互相协作） |
| **被调方是什么** | 工具/资源（数据库、API、文件系统…） | 另一个 Agent（有自主推理能力） |
| **是否带状态** | 通常无状态（一来一回） | **有完整 Task 生命周期**（8 状态） |
| **是否支持反问** | ❌ 不支持 | ✅ 支持 `input-required` |
| **是否支持长任务** | 不擅长 | ✅ 通过 SSE / Push Notification 原生支持 |
| **是否需要身份/授权** | 通常本地/可信环境 | ✅ 支持跨组织、bearer/OAuth |
| **能力发现** | MCP Server 列出 Tools | Agent Card 列出 Skills + capabilities |
| **协议** | JSON-RPC 2.0（stdio / HTTP） | HTTP + JSON-RPC 2.0 + SSE + Webhook |
| **推动者** | Anthropic（2024.11） | Google 发起（2024.4）→ Linux Foundation（2025） |
| **生态成熟度（2026）** | 事实标准，工具丰富 | 1.x 稳定版，主流框架适配中 |

### 核心区别一句话

> **MCP 是「Agent 如何使唤工具」，A2A 是「Agent 如何使唤别的 Agent」。** 工具是哑的（执行就完了），Agent 是活的（可反问、可拒绝、可长跑、可跨组织）—— 所以 A2A 比 MCP 复杂得多。

---

## 六、完整链路示例：竞品分析报告 Agent

用户请求："写一份关于 AI Agent 技术的竞品分析报告"

```
┌─────────────────────────────────────────────────────────────────┐
│ 编排 Agent（CrewAI 实现）                                        │
└──────────┬──────────────────────────────────────────────────────┘
           │ ① 接到用户需求，规划：调研 → 写作
           │
           │ ② 通过 Agent Registry 找到：
           │    - Research Agent（LangGraph 实现，独立部署）
           │    - Writer Agent（AutoGen 实现，独立部署）
           │
           │ ③ 拉取它们的 /.well-known/agent.json
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│ A2A: tasks/sendSubscribe                                         │
│   POST https://research-agent/a2a                               │
│   { task: "搜集主流 Agent 框架信息" }                              │
└──────────┬──────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│ Research Agent（LangGraph 实现，内部用 MCP 调工具）              │
│                                                                  │
│   状态：submitted → working                                       │
│   ├─► MCP 调 web_search Server：搜索 LangChain / CrewAI / ...   │
│   ├─► MCP 调 github Server：拉 star 数 / commit 频率            │
│   ├─► MCP 调 arxiv Server：拉相关论文                            │
│   │                                                              │
│   ├─► 中途 SSE 推送 Message：「正在调研 5 个框架，已完成 3 个」  │
│   │                                                              │
│   └─► 状态：working → completed                                  │
│       Artifact：调研报告.md                                       │
└──────────┬──────────────────────────────────────────────────────┘
           │ 编排 Agent 收到 Artifact
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│ A2A: tasks/sendSubscribe                                         │
│   POST https://writer-agent/a2a                                 │
│   { task: "基于这份调研报告写竞品分析", input: <Artifact> }      │
└──────────┬──────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│ Writer Agent（AutoGen 实现，内部 Skills 指导写作风格）           │
│                                                                  │
│   状态：submitted → working                                       │
│   ├─► 加载 Skill：「分析报告写作 SOP」                           │
│   ├─► 中间 input-required：「读者定位是技术决策者还是工程师？」 │
│   │   ⬇ 编排 Agent 转给用户回答 → 重新进入 working               │
│   │                                                              │
│   └─► 状态：completed                                             │
│       Artifact：竞品分析报告.md                                   │
└──────────┬──────────────────────────────────────────────────────┘
           │
           ▼
   返回给用户
```

### 这个链路里 4 个协议各管什么

| 协议 | 在哪用 | 干什么 |
|---|---|---|
| **A2A** | 编排 Agent ↔ Research Agent / Writer Agent | Agent 之间的任务委托 + 状态跟踪 |
| **MCP** | Research Agent ↔ web_search / github / arxiv Server | Agent 内部调外部工具 |
| **Function Call** | 各 Agent 内部 ↔ LLM API | LLM 输出"我要调什么工具"的意图 |
| **Skills** | Writer Agent 内部 system prompt | 指导 LLM 怎么写分析报告 |

> **完整生态**：A2A（横向 Agent 协作）+ MCP（纵向工具接入）+ Function Call（LLM 调用协议）+ Skills（领域知识）—— 四者各司其职，构成 Agent 系统的完整协议栈。

---

## 七、A2A 技术基础与生态现状

### 技术栈

| 层 | 用什么 |
|---|---|
| **传输层** | HTTP / HTTPS |
| **请求格式** | JSON-RPC 2.0 |
| **流式输出** | Server-Sent Events (SSE) |
| **异步通知** | Webhook / Push Notification |
| **认证** | Bearer Token / OAuth 2.0 / API Key |
| **能力发现** | `/.well-known/agent.json`（HTTP GET） |

设计哲学：**完全基于现有 Web 标准**，零新基础设施——任何 HTTP 服务器都能实现 A2A。

### 治理与生态（2026 年）

| 项 | 说明 |
|---|---|
| **发起方** | Google（2024.4 在 Google Cloud Next 发布） |
| **当前治理** | **Linux Foundation**（2025 年捐赠，开放治理） |
| **协议版本** | 1.x 稳定版，规范见 [a2a.dev](https://a2a-protocol.org/)（社区维护） |
| **主要伙伴** | Salesforce / SAP / Atlassian / Cohere / MongoDB / Box / Workday 等 50+ |
| **框架适配** | LangGraph / CrewAI / AutoGen / Semantic Kernel 等主流框架都有官方/社区适配 |

### MCP vs A2A 时间线对比

| 时间 | 事件 |
|---|---|
| 2024.4 | Google 发布 A2A 协议 |
| 2024.11 | Anthropic 发布 MCP 协议 |
| 2025.中 | A2A 捐赠给 Linux Foundation |
| 2025.下 | MCP 成为事实标准，主流 LLM 厂商都支持 |
| 2026 | A2A 进入 1.x 稳定版，与 MCP 共同构成 Agent 协议栈双标准 |

---

## 八、面试一句话总结

### Q1：MCP 和 A2A 有什么区别？

> "**MCP 解决纵向问题：Agent ↔ 工具**——一个 Agent 怎么标准化地连数据库、API、文件系统。**A2A 解决横向问题：Agent ↔ Agent**——多个 Agent（可能跨框架、跨组织）怎么像微服务一样互相委托任务。比喻一下：MCP 像 USB-C（连外设），A2A 像 HTTP（机器间通信）。两者完全互补——一个 Agent 内部用 MCP 调工具，多个 Agent 之间用 A2A 协作。"

### Q2：为什么 MCP 解决不了 A2A 要解决的问题？

> "因为**工具是哑的，Agent 是活的**。MCP 设计的是一个简单的 stateless 工具调用：'调 read_file，返回内容'，一来一回就结束。而 Agent 协作有 4 个 MCP 完全没设计的场景：
> 1. **长任务**：调研报告可能要跑半小时，需要 SSE / Push Notification——MCP 无支持
> 2. **任务生命周期**：A2A 有 8 种状态（submitted/working/input-required/completed/...），MCP 只有'调用成功/失败'
> 3. **双向反问**：Agent B 执行中可能要追问'你这个范围太宽，能缩小吗？'——MCP 不支持
> 4. **跨组织身份**：Agent A 调用别公司的 Agent B，需要 OAuth 等完整认证——MCP 是本地工具场景
>
> 所以需要专门为 Agent 协作设计的 A2A，而不是把 MCP 强行扩展。"

### Q3：实际项目里 A2A、MCP、Function Call、Skills 怎么协作？

> "用一个'写竞品分析报告'的多 Agent 系统说：
> - **A2A** 让编排 Agent 把'调研'委托给 Research Agent、把'写作'委托给 Writer Agent（横向协作）
> - **MCP** 让 Research Agent 内部能调 web_search、github、arxiv 这些工具（纵向接入）
> - **Function Call** 是每个 Agent 内部 LLM 表达'我要调哪个工具'的协议（LLM 通信）
> - **Skills** 是 Writer Agent 的 system prompt 里那份'分析报告写作 SOP'（方法论）
>
> 四个协议各管一层，缺一不可：A2A 解决跨 Agent，MCP 解决跨工具，Function Call 解决跨 LLM API，Skills 解决跨领域知识。"
