# MCP 全解：协议本质、架构

> 面试常问：
> 1. **「MCP 和 Function Call 有什么本质区别？」**
> 2. **「如果你要给团队接入 10 个外部工具，你会用 MCP 还是直接写 Function Call？为什么？」**
>
> 本文先把 MCP 讲透，然后回答两个深入问题：
> - **Q2**：为什么用了 MCP，集成成本能从 N × M 变成 N + M？

---

## 目录

- [一、MCP 解决的根本问题：N × M 爆炸](#一mcp-解决的根本问题nm-爆炸)
- [二、MCP 架构：Host / Client / Server 三角色](#二mcp-架构host--client--server-三角色)
- [三、MCP 提供的三类资源](#三mcp-提供的三类资源)
- [四、动态工具发现机制](#四动态工具发现机制)
- [五、安全性三层设计](#五安全性三层设计)
- [六、底层协议（面试加分）](#六底层协议面试加分)
- [七、Q1：Function Call vs MCP 本质区别](#七q1function-call-vs-mcp-本质区别)
- [八、Q2：为什么 N × M 能变成 N + M？](#八q2为什么-nm-能变成-nm)
- [九、面试题：10 个外部工具用 MCP 还是 Function Call？](#九面试题10-个外部工具用-mcp-还是-function-call)
- [十、Function Call / MCP / Skills 三层关系全景](#十function-call--mcp--skills-三层关系全景)

---

## 一、MCP 解决的根本问题：N × M 爆炸

> **MCP（Model Context Protocol）** 是 Anthropic 在 2024 年 11 月提出的开放协议，目标是给 LLM 应用接入外部工具/数据/资源时，提供一套**标准化的 RPC 协议**，避免每对接一个工具都要写一遍胶水代码。

### 没有 MCP 的世界：N × M 集成

```
应用 A ──┬─► GitHub 私有集成
         ├─► Slack  私有集成
         └─► DB     私有集成

应用 B ──┬─► GitHub 私有集成   （和 A 重写一遍）
         ├─► Slack  私有集成
         └─► DB     私有集成

应用 C ──┬─► GitHub 私有集成
         ├─► Slack  私有集成
         └─► DB     私有集成
```

**N=3 个应用 × M=3 个工具 = 9 套集成代码**。十个应用接二十个工具就是 200 套。

### 有了 MCP 的世界：N + M 集成

```
应用 A ──┐
应用 B ──┼─► MCP 协议 ─┬─► GitHub MCP Server
应用 C ──┘             ├─► Slack  MCP Server
                       └─► DB     MCP Server
```

**N=3 个应用 + M=3 个工具 = 6 套代码**。详见 [§八](#八q2为什么-nm-能变成-nm)。

---

## 二、MCP 架构：Host / Client / Server 三角色

| 角色 | 谁来扮演 | 职责 |
|---|---|---|
| **MCP Host** | 你用的 AI 应用（Claude Desktop / Cursor / 你的 Agent） | UI 入口、管理多个 Server 配置、把工具列表喂给 LLM |
| **MCP Client** | 住在 Host 进程里的"翻译官" | 负责按 MCP 协议跟 Server 通信（一对一长连接） |
| **MCP Server** | 第三方独立进程（如 GitHub 官方 MCP Server） | 暴露真正的工具/资源能力，对外讲 MCP 协议 |

```
┌──────────────────── Host (Cursor) ────────────────────┐
│                                                       │
│  ┌─── LLM ────┐                                       │
│  │ tool_calls │                                       │
│  └──────┬─────┘                                       │
│         │                                             │
│  ┌──────▼──────────────────────────────────────────┐  │
│  │   Client A          Client B          Client C  │  │
│  └──────┬──────────────────┬──────────────────┬────┘  │
└─────────┼──────────────────┼──────────────────┼───────┘
          │ JSON-RPC         │ JSON-RPC         │
          ▼                  ▼                  ▼
   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
   │ GitHub      │   │ Slack       │   │ Database    │
   │ MCP Server  │   │ MCP Server  │   │ MCP Server  │
   └─────────────┘   └─────────────┘   └─────────────┘
```

> 📌 **关键认知**：MCP Server 内部 **不调用 LLM**——它就是一个普通的服务程序，把外部 API 包装成 MCP 协议而已。详见 [§七](#七q1function-call-vs-mcp-本质区别)。

---

## 三、MCP 提供的三类资源

每个 MCP Server 可以对外暴露三种东西：

| 类型 | 含义 | 例子 |
|---|---|---|
| **Tools** | 可执行的操作 | `github.create_issue`、`slack.send_message`、`db.query` |
| **Resources** | 可读的数据源 | 文件内容、Wiki 页面、数据库表结构 |
| **Prompts** | 预设提示词模板 | "代码审查标准模板"、"PR 描述模板" |

> Tools 是最常用的；Resources 让 LLM 能"读"，Prompts 让 Server 能"教"应用怎么写 prompt。

---

## 四、动态工具发现机制

这是 MCP 最强大的特性之一：

```
启动阶段：
  1. Host 读配置文件，知道有 [GitHub Server, Slack Server, DB Server]
  2. Host 通过 Client 给每个 Server 发 tools/list 请求
  3. Server 返回自己提供的工具清单（含 JSON Schema）
  4. Host 把所有工具的 schema 拼起来，注入到 LLM 的 tools= 参数

运行阶段：
  1. 用户："帮我在 GitHub 创建一个 Issue"
  2. LLM 通过 Function Call 输出 tool_calls: [{name: "github_create_issue", ...}]
  3. Host 路由到 GitHub Client → 通过 MCP 协议调 GitHub Server
  4. GitHub Server 调真实 GitHub API → 返回结果
  5. Host 把结果回传给 LLM，LLM 生成最终回答
```

**革命性的地方**：今天给 Cursor 配一个 Slack MCP Server，**Cursor 不用改一行代码**，明天就能用 Slack 能力。

---

## 五、安全性三层设计

| 层 | 机制 |
|---|---|
| **能力声明** | Server 在 `tools/list` 里**白名单**式声明能干什么；Agent 不能调清单外的工具 |
| **授权控制** | Host 维护"哪个 Server 能用哪个工具"；敏感操作可强制人工 confirm |
| **审计追踪** | 所有 MCP RPC 调用都打日志，可追溯每一次 Agent 行为 |

> 工程实践：生产环境的 Host 通常会再加**速率限制 + 参数校验**这一层，防止 LLM 输出恶意参数。

---

## 六、底层协议（面试加分）

| 维度 | 选型 |
|---|---|
| **消息格式** | **JSON-RPC 2.0**（标准 RPC 规范，每条消息含 `id`、`method`、`params`） |
| **本地通信** | **stdio**（子进程标准输入输出，最简单，零依赖） |
| **远程通信** | **HTTP + SSE**（Server-Sent Events，支持流式返回） |

> 选 JSON-RPC 2.0 是因为它是个**早就成熟、跨语言、规范极简**的 RPC 协议——Anthropic 不想发明新轮子。

---

## 八、Q2：为什么 N × M 能变成 N + M？

> **你的疑问**：为什么用了 MCP，集成成本能从 N × M 变成 N + M？
> 这个降维很反直觉，下面用具体例子拆解。

### 1. 不用 MCP：每个应用要为每个工具单独写胶水

假设有：

- **N = 3 个应用**：Cursor、Claude Desktop、自己的 Agent
- **M = 3 个工具**：GitHub、Slack、Postgres

每个**应用**都需要：
- 知道 GitHub API 怎么调（认证、分页、错误码）
- 知道 Slack API 怎么调
- 知道 Postgres 怎么连
- 把这些工具的 schema 翻译成 LLM 能用的 tool 定义

每个**工具**也都被重复对接：
- GitHub 团队不维护任何东西，但每个应用各自写一份"GitHub 集成层"
- 一旦 GitHub 改 API，**3 个应用都要改**

**总代码量**：

```
                GitHub  Slack  Postgres
Cursor          [代码]  [代码]  [代码]      ← 3 份
Claude Desktop  [代码]  [代码]  [代码]      ← 3 份
自己的 Agent    [代码]  [代码]  [代码]      ← 3 份
                                           ─────
                                           N × M = 9 份
```

通用公式：**N 个应用 × M 个工具 = N × M 份集成代码**。
10 个应用 × 20 个工具 = **200 份代码**。

### 2. 用 MCP：每个应用和每个工具各自只实现一次

**关键转换**：MCP 在中间立了一个**标准协议**。

- 每个**应用**只需实现一次 **MCP Client**（讲 MCP 协议就行，不需要懂 GitHub/Slack/PG）
- 每个**工具**只需实现一次 **MCP Server**（GitHub 团队官方维护一个 GitHub MCP Server，Slack 团队维护一个，等等）
- 应用和工具之间通过 MCP 协议自动握手

```
                MCP Client          MCP Server
Cursor          [实现 1 次]    ◄─►  [GitHub Server,    1 次]
Claude Desktop  [实现 1 次]    ◄─►  [Slack  Server,    1 次]
自己的 Agent    [实现 1 次]    ◄─►  [Postgres Server, 1 次]
                ─────────              ─────────
                N = 3 份               M = 3 份
                              合计 N + M = 6 份
```

通用公式：**N 个应用 + M 个工具 = N + M 份代码**。
10 个应用 + 20 个工具 = **30 份代码**（相比 200 份省了 85%）。

### 3. 为什么"乘"会变成"加"？—— 一个生活类比

| 没有标准 | 有了标准 |
|---|---|
| **每个国家用自己的电源插头** —— 你出国一次，要为每个目的地买一个转换器（N 国 × M 设备 = 一堆转换器） | **统一 USB-C 标准** —— 你只需要 1 根线，所有支持 USB-C 的设备都能充（N 个设备 + 1 根线，几乎零边际成本） |
| **每对应用-工具都要协商私有协议** | **大家都讲 MCP** |

**乘法**来自"任意两两组合都要单独协商"；
**加法**来自"每方只需对接同一个标准协议"。

### 4. 数学表达

| 模式 | 代码总量 | 增长方式 |
|---|---|---|
| **私有集成（点对点）** | `N × M` | 任何一边新增，总量乘性增长 |
| **MCP（标准协议）** | `N + M` | 任何一边新增，总量线性增长 |

### 5. MCP 还顺带解决了什么？

| 收益 | 说明 |
|---|---|
| **维护责任下沉** | GitHub API 变了，**只有 GitHub MCP Server 要改一次**，所有应用零感知 |
| **生态共享** | 别人写的 MCP Server 你直接配上就能用（Anthropic 已有 200+ 官方/社区 Server） |
| **动态发现** | 加新 Server 不用改 Host 代码，热加载 |
| **跨语言互通** | 应用是 Python，Server 是 Node.js，靠 JSON-RPC 解耦 |

### 一句话答

> "N × M 来自'每对 (应用, 工具) 都要写一份私有集成'；MCP 在中间立了一个**标准 RPC 协议**——应用只需实现一次 MCP Client（不用懂任何具体工具），工具厂商只需实现一次 MCP Server（不用关心谁来调），两边自动握手。**任意一边新增只增加 1 份代码而非 M 份**，所以总量从乘性 N × M 退化成加性 N + M。"

---

## 九、面试题：10 个外部工具用 MCP 还是直接对接？

### 先澄清面试题里的不严谨

面试官原话「用 MCP 还是直接写 Function Call」其实是**口语简称**，严格说这不是一个公平的二选一——因为：

| 关键事实 | 说明 |
|---|---|
| **Function Call 是必备的** | 无论用不用 MCP，**LLM 跟应用之间永远靠 Function Call 沟通**。MCP 替代不了它 |
| **真正访问工具的不是 Function Call** | Function Call 只是 LLM 输出"我想调什么"的格式；**真正的调用执行由应用层做**——要么直连 SDK，要么走 MCP |
| **真正的二选一是"工具集成层"** | 工具的 schema 和执行是**应用自己写胶水代码**，还是**通过 MCP 标准协议接入 Server** |

### 真正的两条路径

| 路径 | 工具集成方式 | 完整通信链路 |
|---|---|---|
| **A. 直连** | 应用层自己 `import sdk` / 写 HTTP 调用 | LLM ─FC─► Host ─SDK/HTTP─► 工具 |
| **B. MCP** | 通过 MCP 协议调标准化 Server | LLM ─FC─► Host ─MCP─► Server ─SDK/HTTP─► 工具 |

> **两条路径都用 Function Call**——区别只在工具集成层走哪条路。

### 决策矩阵（面试可以画）

| 场景 | 推荐路径 | 理由 |
|---|---|---|
| **工具少（≤5）+ 自己内部工具** | **A 直连** | 自己写几行 SDK 调用最直接，MCP 进程通信是过度工程 |
| **工具多（10+）+ 涉及第三方** | **B MCP** | N + M 收益开始显现；可复用社区 Server |
| **需要在多个应用间共享工具** | **B MCP** | 同一份 Server 可被 Cursor / Claude Desktop / 自研 Agent 同时连接 |
| **第三方工具频繁变化** | **B MCP** | 维护责任下沉到工具厂商，应用零感知 |
| **对延迟极敏感** | **A 直连** | 省掉 MCP 的进程间通信开销 |
| **超严格安全合规** | **A 直连** | 不引入外部 Server 进程，攻击面更小，工具代码全在自己进程内 |

### 一句话答

> "首先澄清，'用 MCP 还是用 Function Call' 是个不严谨的口语提法——**Function Call 是必备的**（LLM 跟应用之间永远要它），真正的二选一是工具集成层走 **直连 SDK** 还是 **MCP 协议**。10 个外部工具我会选 **MCP 路径**：N + M 的集成成本优势开始显现，可以直接用 GitHub / Slack 等官方 MCP Server，不用自己维护胶水代码。如果工具少且都是公司内部 RPC，我会选 **直连路径**，避免 MCP 进程间通信带来的过度工程。"

---

## 十、Function Call / MCP / Skills 三层关系全景

下面这张图用「新员工入职」类比把三层关系讲透：


| 层 | 类比 | 职责 |
|---|---|---|
| **Skills 层** | 岗位培训手册 | 告诉 Agent**"遇到这类问题该怎么想"**——领域知识编码、按需激活 |
| **MCP 层** | 公司通讯录 + 电话系统 | 告诉 Agent**"有哪些工具可以调用"**——工具集成标准化、动态发现 |
| **Function Call 层** | 打电话的基础能力 | 告诉 Agent**"怎么调用一个函数"**——LLM 输出指令、应用执行回传 |

### 两种维度看顺序——别混淆

#### 维度 1：抽象层次（图片画的视角，从高层认知到底层协议）

```
Skills（怎么想） ─► MCP（用什么） ─► Function Call（怎么调）
   高层认知层       中层工具市场       底层调用协议
```

> 这是**资源/抽象层级**的描述，**不是执行顺序**。

#### 维度 2：运行时执行顺序（一次实际调用的时间线）

```
Skills（思考激活） ─► Function Call（LLM 输出意图） ─► MCP（路由到 Server 执行）
```

> 注意 **Function Call 在 MCP 之前**——LLM 永远先用 Function Call 表达"我要调什么"，Host 才能根据这个名字去 MCP Server 列表里找谁来执行。

### 具体一次执行（按运行时顺序）

| 步骤 | 涉及层 | 发生了什么 |
|---|---|---|
| ① | **Skills** | Agent 接到"代码审查"任务，激活"代码审查 Skill"——指导思路 "按 安全性 > 性能 > 质量 排序" |
| ② | **Skills + Function Call** | 这条 Skill 把"调用 `get_pr_diff` 拿 PR 内容"作为第一步告诉 LLM；LLM 用 **Function Call** 协议输出 `tool_calls: [{name: "get_pr_diff", args: {...}}]` |
| ③ | **MCP** | Host 拿到 tool_call 名字 → 查路由表 → 通过 **MCP 协议**把请求发给 GitHub Server → Server 调真实 GitHub API → 结果回传 |
| ④ | **Function Call** | Host 用 `role: "tool"` 把结果通过 **Function Call** 回传给 LLM |
| ⑤ | **Skills** | LLM 按 Skill 指导的"安全性 > 性能 > 质量"顺序生成 review 评论 |

### 三者关系一句话

> **Function Call 是基础能力（怎么调）→ MCP 是工具市场（用什么）→ Skills 是行业知识（怎么想）**。这是**抽象层次**的描述。
>
> 但**运行时**它们交织协作：Skills 在每个决策点提供"思路"，每次 LLM 输出动作都用 **Function Call**，每次工具执行都通过 **MCP**——三者互相独立、按层叠加，没有 Function Call 就没法跟 LLM 沟通，没有 MCP 就没法规模化接入工具，没有 Skills 就没法把工具用得专业。
