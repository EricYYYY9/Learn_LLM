# Tools vs MCP：拆解到底层的区别

> **一句话总结**：
> **Tool 是"一个能力的定义"，MCP 是"把一堆能力交付给 LLM 的标准化协议"。**
> 一个回答**做什么**，一个回答**怎么接进来**。

> **术语提醒（口语化用法）**：
> 日常交流中，大家经常会直接说"**查询天气 MCP**" / "**GitHub MCP**" / "**接一个 XX MCP**"——这其实是**口语化的简写**，准确含义是 **"通过 MCP 协议暴露 XX 能力的那个 MCP Server"**。
>
> - 严格来说：**MCP 是协议本身**（一套规范），不是某个具体工具
> - 口语里："**XX MCP**" ≈ "**XX MCP Server**" ≈ "**通过 MCP 协议提供 XX 能力的工具集**"
>
> 阅读本文时遇到"某某 MCP"的说法，**默认指代的是"提供该能力的 MCP Server"**，不是协议本身。

---

## 一、本质定义

- **Tool**：一个**给 LLM 暴露的可调用函数**——由 `name` + `description` + `input_schema`（JSON Schema）三件套定义，是 **LLM API 协议层**的概念。
- **MCP**：一套**让 LLM 应用能动态发现并远程调用一组工具的标准化通信协议**（基于 JSON-RPC 2.0），是**应用与外部工具集合之间的传输 / 集成层规范**。

二者**不在同一抽象层**：

| 概念 | 抽象层 | 回答的问题 |
|---|---|---|
| **Tool** | 协议字段 / 数据结构 | "**LLM 能调用什么函数？**" |
| **MCP** | 通信协议 / 集成规范 | "**这些函数怎么被发现、怎么被远程调用？**" |

→ MCP 不是一种新工具，**MCP 是把工具批量、标准化暴露给 LLM 的通道**。最终交到 LLM 面前的，仍然是 Tool。

---

## 二、最关键的认知：MCP 最终也是给 LLM 暴露 Tool

```
                                          ┌─────────────────┐
                                          │  Claude API     │
                                          │  tools=[...]    │ ← LLM 只认 Tool
                                          └────────▲────────┘
                                                   │
            ┌────────────────────┬─────────────────┼──────────────────┬──────────────────┐
            │                    │                 │                  │                  │
    ┌───────┴───────┐   ┌────────┴────────┐  ┌─────┴──────┐   ┌───────┴────────┐  ┌──────┴──────┐
    │ 直接定义 Tool   │   │ 模型平台内置     │   │ Agent 产品 │   │  MCP Server    │  │  其他 MCP    │
    │ （应用代码写）   │   │ Tool（厂商提供） │   │ 内置 Tool  │   │  filesystem    │  │  Server     │
    │                │   │                │  │            │   │                │  │  github     │
    │ get_weather()  │   │ web_search     │  │ Read       │   │ read_file      │  │ create_issue│
    │ send_email()   │   │ code_execution │  │ Edit       │   │ write_file     │  │ list_pr     │
    │                │   │ computer_use   │  │ Shell      │   │ list_dir       │  │ get_pr_diff │
    └────────────────┘   └────────────────┘  └────────────┘   └────────────────┘  └─────────────┘
          ↑                      ↑                  ↑                  ↑                  ↑
          │                      │                  │                  │                  │
    手写 + 框架/手动挂   API 声明即可，由         产品（Cursor /    启动 MCP Server     启动另一个
    完全自定义           Anthropic / OpenAI       Claude Code 等）  进程，由 MCP        MCP Server
                         在服务端自动执行         自带，不用配置     Client 自动发现
```

**核心结论**：**对 LLM 来说，最终看到的都是 Tool，但 Tool 的来源有多种**。
MCP 只是其中一种"**让 Tool 标准化、可跨应用复用**"的分发机制。

---

### 2.1 Tool 的三种来源（重要）

| 来源 | 谁提供 | 谁执行 | 典型例子 | 用法 |
|---|---|---|---|---|
| **① 应用自定义 Tool** | 你的应用代码 | 你的应用进程 | `get_weather` / `send_email` 等业务函数 | 手写函数 + JSON Schema，手动或框架自动挂载 |
| **② 模型平台内置 Tool** | LLM 厂商（Anthropic / OpenAI 等） | **厂商服务端**（或客户端 SDK） | Claude 的 `web_search` / `code_execution` / `computer_use` / `text_editor` / `bash`；OpenAI 的 `code_interpreter` / `file_search` / `image_generation` | API 里**只需声明类型**，无需实现：`{"type": "web_search_20250305", "name": "web_search"}` |
| **③ Agent 产品内置 Tool** | Agent 产品（Cursor / Claude Code / ChatGPT 等） | 产品自身 | Cursor 的 `Read` / `Edit` / `Grep` / `Shell` / `Task`；Claude Code 的 `Read` / `Write` / `Bash` / `WebFetch` | 用户**开箱即用**，不用配置任何东西 |
| **④ MCP Server 暴露的 Tool** | MCP Server 进程 | MCP Server 进程 | `mcp-server-filesystem`、`mcp-server-github`、`mcp-server-postgres` | 配置 `mcpServers`，自动通过协议发现 |

#### 举个最直观的例子：Claude 的内置 Tool

调用 Claude API 时，**不用自己写 `web_search` 实现**，只需在 `tools` 里声明：

```python
client.messages.create(
    model="claude-sonnet-4-5",
    tools=[
        # ① 平台内置：仅需声明类型，Anthropic 服务端会自动执行搜索
        {"type": "web_search_20250305", "name": "web_search"},
        # ② 平台内置：服务端代为执行 Python 代码
        {"type": "code_execution_20250522", "name": "code_execution"},
        # ③ 应用自定义：必须自己写实现
        {
            "name": "get_weather",
            "description": "查城市天气",
            "input_schema": {...}
        }
    ],
    messages=[...]
)
```

**关键差别**：
- **内置 Tool（②③）**：你只是"启用"它，**实现完全不在你这边**——厂商或产品已经帮你做好了
- **自定义 Tool（①）**：你必须**自己写函数实现** + Schema
- **MCP Tool（④）**：你既不用写实现，也不用写 Schema，**由 MCP Server 进程提供**

#### 内置 Tool 和 MCP 有什么区别？

| 维度 | 模型/产品内置 Tool | MCP Server 提供的 Tool |
|---|---|---|
| **来源** | LLM 厂商或 Agent 产品**官方预置** | 第三方/自己写的独立进程 |
| **是否需要配置** | ❌ 默认可用（部分需 API 启用） | ✅ 需要在配置里声明 `mcpServers` |
| **是否可扩展** | ❌ 用户无法新增（只能用厂商提供的那几个） | ✅ 任何人都可以写 MCP Server 扩展能力 |
| **执行位置** | 厂商服务端 / 产品本地 | 独立 MCP Server 进程 |
| **典型场景** | 通用刚需（搜索、代码执行、文件读写） | 业务专用 / 第三方系统集成 |

→ **内置 Tool 是"官方套餐"，MCP 是"自助加菜"**。两者互补，不冲突。

---

## 三、对比维度细看

| 维度 | Tool | MCP |
|---|---|---|
| **本质** | 一个函数 + JSON Schema 定义 | 一套 RPC 协议（基于 JSON-RPC 2.0）+ Server 规范 |
| **解决的问题** | "LLM 怎么知道我有什么函数可以调？" | "我怎么把一堆现成的工具集合标准化地接入 LLM？" |
| **需要写代码吗** | ✅ 必须自己写函数 + 注册到 `tools` 参数 | ❌ 启动现成 MCP Server 即可（如 `npx @modelcontextprotocol/server-filesystem`），完全无需写代码 |
| **粒度** | **单个函数**（一个 Tool = 一个能力） | **一组工具的集合**（一个 Server 通常导出几十个 Tool） |
| **复用性** | 只在你这个应用里能用 | 一份 Server 实现可被 Claude Desktop / Cursor / 自研 LLM 应用共用 |
| **谁在管理** | 应用代码 | 独立进程的 MCP Server |
| **运行位置** | 你的应用进程内 | 独立进程（stdio 子进程 或 HTTP Server） |
| **生态** | 各厂商 API 格式不一样（OpenAI / Anthropic / Gemini 各有差异） | 开放统一标准（Anthropic 主导，所有客户端通用） |
| **动态性** | 通常启动时静态注册 | Server 可动态增删工具（`tools/list_changed` 通知） |
| **隔离性** | 与应用共享进程 / 内存 | 独立进程，可单独管理权限、审计、崩溃隔离 |

---

## 四、用代码看区别（最直观）

### 写法 1：直接定义 Tool（不用 MCP）

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4",
    max_tokens=1024,
    tools=[
        {
            "name": "get_weather",
            "description": "查城市天气",
            "input_schema": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"]
            }
        }
    ],
    messages=[{"role": "user", "content": "北京天气怎么样？"}]
)

def get_weather(city):
    return {"temp": 25, "weather": "晴"}
```

**特点**：每个工具的**函数实现 + JSON Schema 描述都得自己写**；至于"如何挂到 LLM 调用上"，可以**手动 append 到 `tools=[...]`**，也可以**借助框架的自动注册机制**（详见下文 §4.3）。

---

### 写法 2：用 MCP 接入一堆 Tool（不写代码）

**第一步**：在 Claude Desktop 配置文件里加：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/eric/docs"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "ghp_xxx"}
    }
  }
}
```

**第二步**：重启 Claude Desktop。

**结果**：Claude 自动获得 `read_file` / `write_file` / `list_dir` / `create_issue` / `get_pr_diff` 等**几十个 Tool**——你**一行代码都没写**。

**这背后发生了什么**：

1. Claude Desktop 启动那两个 MCP Server 进程
2. 通过 MCP 协议问它们："你有什么工具可以提供？"（`tools/list` 请求）
3. Server 返回工具列表（每个工具就是一个标准 Tool 定义）
4. Claude Desktop 把它们**翻译成 Claude API 的 `tools` 参数**
5. 当 Claude 决定调用某个工具，Desktop 通过 MCP 协议（`tools/call`）转发给对应 Server，再把结果返回 Claude

→ **你看，最后还是 Tool**，只不过 MCP 帮你把"定义 + 实现 + 转发"的活全做了。

---

### 4.3 澄清："自动挂载" ≠ MCP

很多 Agent 框架（LangChain、Claude Agent SDK、Cursor、自研 Agent 框架等）也允许通过**装饰器、文件扫描、配置文件**等方式"自动发现"本地手写的 Tool，无需每次手动 append 到 `tools=[...]`。例如：

```python
@agent.tool   # 装饰器自动注册
def get_weather(city: str) -> dict:
    """查城市天气"""
    return {"temp": 25, "weather": "晴"}

# 框架启动时扫描所有 @agent.tool，自动收集进 tools 列表
# 调用 LLM 时框架自动把它们塞进 API 请求
```

**这看起来和 MCP 很像，但本质完全不同**：

| 维度 | 框架自动挂载（手写 Tool） | MCP |
|---|---|---|
| **生效范围** | 仅限**当前应用进程内** | **跨进程、跨应用、跨语言、跨组织** |
| **协议** | 框架私有约定（每个框架一套，互不通用） | 公开标准（JSON-RPC 2.0 + MCP 规范） |
| **能否被别的客户端复用** | ❌ 换个框架就得重写 | ✅ Claude Desktop / Cursor / 任意 MCP Client 启动即用 |
| **工具实现位置** | 必须在你应用代码里（语言绑死） | 独立进程，可以是别人写的、其他语言写的 |
| **解决的问题** | "**省掉手动 append 的样板代码**"，提升单应用开发体验 | "**让工具能被任意 LLM 应用标准化地发现和调用**"，解决集成问题 |
| **底层机制** | 框架在调 LLM 前自动收集本地 Tool 描述塞进 `tools=[...]` | MCP Client 通过 RPC 远程问 MCP Server "你有什么工具" |

**结论**：

- 框架的"自动挂载"解决的是**单一应用内的开发体验**——少写两行注册代码而已
- MCP 解决的是**跨应用、跨进程、跨组织的工具集成标准**——是一种**生态层面的协议**

→ 即便框架帮你"自动挂载"了 Tool，那个 Tool 的**实现仍然绑死在你的应用进程里**；别人想复用就得拿到你的源码、用同样的框架、同样的语言去集成。
而 MCP Server **以独立进程对外暴露**，任何支持 MCP 的客户端启动它即可使用——这才是 MCP 的真正价值。

> **一句话**：**"自动挂载"是开发便利性，MCP 是跨边界的工具集成协议。两者不在一个维度上。**

---

## 五、运行时数据流对比

### 直接 Tool 调用流程

```
User → 应用 → Claude API (带 tools 列表)
                  ↓
              Claude 返回 tool_use 指令
                  ↓
            应用代码：自己 dispatch 到本地函数
                  ↓
              本地函数执行 → 结果
                  ↓
              再把结果发回 Claude
```

### MCP Tool 调用流程

```
User → 应用（MCP Client） → Claude API (带 tools 列表，来自 MCP Server)
                  ↓
              Claude 返回 tool_use 指令
                  ↓
            MCP Client：识别这是 MCP Server 提供的工具
                  ↓
            通过 JSON-RPC（stdio / HTTP）调 MCP Server
                  ↓
              MCP Server 执行 → 结果
                  ↓
              MCP Client 收到结果，发回 Claude
```

**差异**：多了一层"**MCP Client ↔ MCP Server**"的标准化通信层，但对 Claude 完全透明。

---

## 六、什么时候用 Tool、什么时候用 MCP

| 场景 | 选什么 | 为什么 |
|---|---|---|
| 需要联网搜索、执行代码、操作浏览器/电脑等通用能力 | **平台内置 Tool**（如 Claude `web_search` / `code_execution` / `computer_use`） | 厂商官方实现，零开发成本，且执行环境隔离/合规由厂商负责 |
| 一两个简单函数，应用内独有 | **直接 Tool** | 写一下 schema 就行，没必要起 MCP Server |
| 接入 GitHub / Postgres / Slack / 文件系统等成熟外部系统 | **MCP** | 社区已有现成 Server，启动即用，不用重复造轮子 |
| 想让团队的 Cursor / Claude Code / 自研应用共用一套工具 | **MCP** | 一份 MCP Server 实现，所有客户端都能挂 |
| 工具暴露敏感操作，需要独立进程隔离 | **MCP** | MCP Server 是独立进程，可单独管权限/审计/熔断 |
| 工具实现就在你应用代码里，且不需要被别人复用 | **直接 Tool** | 起 MCP Server 是过度设计 |
| 跨语言场景（应用是 Python，工具用 Go 写） | **MCP** | MCP 协议跨语言，进程隔离天然支持异构 |
| 工具列表需要运行时动态变化 | **MCP** | 支持 `tools/list_changed` 通知 |

---

## 七、常见误解澄清

### 误解 1："MCP 是用来替代 Tool 的"
❌ 错。MCP **不是替代** Tool，而是**为 Tool 提供一种标准化分发方式**。
对 LLM 而言，MCP Server 暴露的最终还是 Tool。

### 误解 2："用了 MCP 就不用写 Function Call 了"
❌ 错。MCP **底层依然是 Function Call**——LLM 还是输出 `tool_use` 指令，只不过这个指令的执行被路由到 MCP Server 而已。

### 误解 3："MCP 自带 LLM 调用能力"
❌ 错。MCP Server 本身**不调 LLM**，它只是个工具提供方。是 Host 端（Claude Desktop / Cursor 等）调 LLM，再通过 MCP Client 把工具转发给 MCP Server 执行。

### 误解 4："Tool 和 MCP 是互斥的"
❌ 错。一个应用里**完全可以同时用**：
- 自己写几个业务专用的 Tool
- 同时挂载 MCP Server 提供通用能力
两者最终在 `tools=[...]` 参数里**合并成一个列表**给 LLM。

### 误解 5："框架能自动加载手写 Tool，那它和 MCP 就是一回事了"
❌ 错。框架的"自动加载"只在**自己的进程、自己的框架、自己的语言**内有效，是**私有便利机制**；MCP 是**公开标准**，让工具能跨应用、跨进程、跨语言被任意 LLM 客户端发现和调用。详见 §4.3。

---

## 八、终极一句话

> **Tool 是「LLM 能调用的一个函数」，MCP 是「让一堆工具能被 LLM 自动发现和接入的标准协议」。**
>
> **Tool 有四种来源**：① 应用自己写的、② 模型平台内置（Claude `web_search` 等）、③ Agent 产品内置（Cursor 的 `Read/Edit` 等）、④ MCP Server 暴露的。
>
> - **没有 MCP，你也能用 Tool**——可以手写、可以用平台内置、也可以用产品自带
> - **没有 Tool，MCP 就没有意义**——因为 MCP Server 暴露给 LLM 的最终就是一堆 Tool
>
> **关系**：MCP 是 Tool 的"**批发渠道 + 标准化接口**"，Tool 才是"**商品本身**"。

---

## 九、扩展阅读

- [`mcp.md`](./mcp.md)：MCP 协议的详细解析
- [`function_call.md`](./function_call.md)：Function Call 与 LLM 工具调用的底层原理
- [`skills.md`](./skills.md)：Skills 与 Tool / MCP 的对比
- [`a2a.md`](./a2a.md)：A2A 协议（Agent ↔ Agent）与 MCP（Agent ↔ Tool）的区别
