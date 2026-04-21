# Claude 的四种产品形态：从 API 到 Agent

> Anthropic 把 Claude 能力暴露成了四种不同抽象层级的"产品形态"：
> **Claude.ai / Claude Desktop**、**The Claude API**、**Claude Code**、**Agent built with Claude Agent SDK**。
>
> 它们目标用户、抽象层级、定制粒度完全不同，但都能加载同一份 **Skill**——这是 Claude 生态的核心设计。

---

## 一、整体地图

### 1.1 抽象层级图（从底到上）

```
┌────────────────────────────────────────────────────────────────────┐
│  Claude.ai / Claude Desktop      ← 终端用户产品（聊天界面）         │
├────────────────────────────────────────────────────────────────────┤
│  Claude Code                     ← 终端用户产品（CLI / IDE 编程）   │
├────────────────────────────────────────────────────────────────────┤
│  Agent built with Claude Agent SDK  ← 你用 SDK 自己造的 Agent 应用  │
├────────────────────────────────────────────────────────────────────┤
│  Claude Agent SDK                ← 框架（封装 Loop / Tool / Skills）│
├────────────────────────────────────────────────────────────────────┤
│  The Claude API                  ← 最底层：模型调用接口             │
├────────────────────────────────────────────────────────────────────┤
│  Claude 模型本体（Sonnet / Opus / Haiku）                          │
└────────────────────────────────────────────────────────────────────┘
```

→ **越往下越底层、越通用**，越往上越场景化、越开箱即用。
→ 上面的产品都是**基于下面的接口搭起来的**。

### 1.2 一句话区分

| 产品 | 一句话定义 | 面向谁 |
|---|---|---|
| **Claude.ai / Claude Desktop** | 官方做好的**对话产品**（网页/桌面 App），开箱即用 | **普通用户**（不写代码） |
| **The Claude API** | 给开发者调用的**原始 LLM 接口** | **后端开发者**（自己造产品） |
| **Claude Code** | Anthropic 官方做的**命令行编程 Agent**（IDE 集成） | **程序员**（用它干活） |
| **Agent + Claude Agent SDK** | 用官方 SDK **自己搭的 Agent 应用** | **Agent 应用开发者** |

---

## 二、Claude.ai / Claude Desktop（终端用户产品）

### 2.1 定位

Anthropic 官方做的**通用对话产品**，对应 ChatGPT 的网页 + 桌面客户端形态。

- **Claude.ai**：网页版，浏览器打开 https://claude.ai 就能用
- **Claude Desktop**：原生桌面 App（Mac / Windows），与网页同源，但能调用本地 MCP

### 2.2 核心特性

| 特性 | 说明 |
|---|---|
| **零门槛** | 注册账号即可使用，**完全不用写代码** |
| **内置工具** | Web Search、Artifacts（生成文档/代码/网页预览）、文件上传、图像理解 |
| **Projects** | 把上下文/参考资料/Skills 组织成"项目"，每次对话沿用 |
| **Skills 支持** | ✅ 在项目设置里上传 Skill，自动激活 |
| **MCP 支持** | ✅ Claude Desktop 支持本地 MCP Server（编辑配置文件 `claude_desktop_config.json`）|
| **多设备同步** | 网页 + 桌面 + 移动端账号同步 |

### 2.3 适用场景

| 场景 | 适合度 |
|---|---|
| 写邮件、润色文档、翻译 | ✅✅✅ |
| 写代码（一次性问题）| ✅✅ |
| 整理资料、做 PPT 大纲 | ✅✅✅ |
| 集成到自己的产品 | ❌（用 API） |
| 自动化批处理 | ❌（用 API） |

### 2.4 收费

| 套餐 | 价格 | 主要差异 |
|---|---|---|
| Free | 免费 | 限速度 + 限模型 |
| Pro | $20/月 | 高频使用、更新模型、Projects |
| Max | $100/月 | 大文件 + 高优先级 |
| Team | $25/人/月 | 团队协作 |

### 2.5 典型用户

- 学生、文员、产品经理、运营、设计师
- "**我只是想用一下 AI**"，不想写一行代码

---

## 三、The Claude API（最底层接口）

### 3.1 定位

Anthropic 提供的**原始 HTTP 接口**——直接调模型，没有任何"产品化"封装。

```
POST https://api.anthropic.com/v1/messages
Content-Type: application/json
x-api-key: sk-ant-xxx
anthropic-version: 2023-06-01

{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "messages": [{"role": "user", "content": "Hello"}],
  "tools": [...]
}
```

### 3.2 核心特性

| 特性 | 说明 |
|---|---|
| **最大灵活性** | 你完全控制 prompt、tools、上下文、流式输出 |
| **无 Agent Loop** | ❌ API 是**单次请求-响应**，工具调用循环要你自己写 |
| **无内置工具实现** | tools 字段只是**声明**，实现要你自己写 |
| **平台内置 Tool** | ✅ 部分官方 Tool（`web_search` / `code_execution` / `computer_use` 等）由 Anthropic 服务端执行 |
| **Skills 支持** | ✅ 通过 `skills` 参数加载 |
| **MCP 支持** | ❌ API 本身不直接支持 MCP；要在你应用层接 MCP Client |
| **Prompt Caching** | ✅ 重复内容（系统提示、文档）可缓存，输入价格降到 1/10 |
| **Batch API** | ✅ 异步批量请求享 50% 折扣 |

### 3.3 适用场景

| 场景 | 适合度 |
|---|---|
| 给已有 SaaS 集成 AI 文本能力 | ✅✅✅ |
| 自动化批处理（总结大量文档）| ✅✅✅ |
| 完全自定义的对话产品 | ✅✅ |
| 复杂 Agent 应用 | ⚠️（自己写 Loop 累，建议用 Agent SDK） |
| 个人偶尔聊天 | ❌（用 Claude.ai） |

### 3.4 收费（按 Token 计费，2026 年初）

| 模型 | 输入 / 1M tokens | 输出 / 1M tokens |
|---|---|---|
| Claude Opus 4 | $15 | $75 |
| **Claude Sonnet 4.5** | **$3** | **$15** |
| Claude Haiku 4 | $1 | $5 |

> Prompt Caching 命中时输入降至约 $0.30/M（10% 折扣）；Batch API 整体打 5 折。

### 3.5 典型用户

- 后端工程师
- 产品集成工程师
- "**我要把 Claude 嵌到我自己的产品里**"

---

## 四、Claude Code（编程专用 Agent）

### 4.1 定位

Anthropic 官方**用 Claude Agent SDK 造的"编程 Agent"成品**——CLI + IDE 插件形态，专门优化代码工程任务。

```bash
npm install -g @anthropic-ai/claude-code
claude   # 进入对话式 CLI
```

### 4.2 核心特性

| 特性 | 说明 |
|---|---|
| **专为代码场景优化** | 内置 system prompt 针对工程任务调优 |
| **内置工具集** | `Read` / `Write` / `Edit` / `Grep` / `Glob` / `Shell` / `Task` / `WebFetch` / `WebSearch` |
| **Subagent** | 可派生子 Agent 处理复杂子任务（探索代码库、并行测试） |
| **Plan Mode** | 大型任务先规划再实施 |
| **Skills 支持** | ✅ 在 `~/.claude/skills/` 或项目 `.claude/skills/` 下放 Skill 即可加载 |
| **MCP 支持** | ✅ 通过 `~/.claude/mcp.json` 配置任意 MCP Server |
| **IDE 集成** | Cursor / VSCode / JetBrains 插件 |
| **Hooks** | 提交前钩子、命令拦截、安全策略 |
| **GitHub 集成** | `gh` CLI + 内置 PR 评审工作流 |

### 4.3 它和"用 SDK 自己造的 Agent"是什么关系？

**Claude Code 自己就是用 Claude Agent SDK 造的**——
它是 Anthropic 给你做的一个"**编程场景的样板 Agent**"。

```
Claude Agent SDK  ─── 用来造 ───▶ Claude Code
                  ─── 用来造 ───▶ 你公司的客服 Agent
                  ─── 用来造 ───▶ 你公司的运维 Agent
```

### 4.4 适用场景

| 场景 | 适合度 |
|---|---|
| 改 Bug、加功能、写测试 | ✅✅✅ |
| 重构代码 | ✅✅✅ |
| 跑测试、提 PR | ✅✅✅ |
| 探索陌生代码库 | ✅✅✅ |
| 写非编程文档 | ⚠️（能写但不是它最强领域） |
| 通用聊天 | ❌（用 Claude.ai） |

### 4.5 收费

- **订阅制**：$20/月 起（Pro 套餐共享 Claude.ai 额度）
- 也可走 API key 按 token 计费

### 4.6 典型用户

- **程序员**
- "**我要 AI 替我干编程的活**"

---

## 五、Agent built with Claude Agent SDK（自建 Agent）

### 5.1 定位

**你用 Anthropic 提供的 Claude Agent SDK，自己造一个 Agent 应用**——可以是任意业务场景。

> 如果说 Claude Code 是 Anthropic 用 SDK **替你造好的样板房**，
> 那"Agent + SDK"就是你拿同样的工具包**给自己造定制楼盘**。

### 5.2 SDK 提供了什么（节省你大量工作）

| 功能 | 自己用 API 实现 | 用 Agent SDK |
|---|---|---|
| **Agent Loop**（模型 → 工具调用 → 结果回喂 → 再决定）| 几十行循环 + 状态机 | ✅ 一行调用 |
| **Tool 注册和调度** | 手写 dispatch | ✅ 装饰器 / 配置 |
| **MCP 客户端** | 自己实现 JSON-RPC | ✅ 内置 |
| **Skills 加载** | 自己拼 prompt | ✅ 自动 |
| **上下文压缩 / 长对话内存** | 自己写策略 | ✅ 内置 |
| **流式输出** | 自己 parse SSE | ✅ |
| **Subagent 派生** | 自己实现 | ✅ |
| **权限控制 / Hooks** | 自己写 | ✅ |
| **错误重试 / 限流** | 自己写 | ✅ |

→ **核心价值**：让你**专注业务**（system prompt + 工具实现 + 数据集成），而不是反复造 Agent 框架的轮子。

### 5.3 典型代码结构

```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

options = ClaudeAgentOptions(
    system_prompt="""
你是某公司的客服 Agent，遵守以下规则：
1. 只处理订单、物流、退款相关问题
2. 涉及金额超过 1000 元的退款必须转人工
""",
    allowed_tools=["query_order", "query_logistics", "create_refund", "transfer_to_human"],
    skills=["./skills/refund_policy", "./skills/product_faq"],
    mcp_servers={
        "internal_crm": {"command": "node", "args": ["mcp-crm-server.js"]}
    }
)

async with ClaudeSDKClient(options) as client:
    await client.query("用户问：我昨天的订单 #12345 还没收到")
    async for message in client.receive_response():
        print(message)
```

### 5.4 适用场景

| 场景 | 例子 |
|---|---|
| 业务专属 Agent | 客服 Agent / 运维 Agent / 法律咨询 Agent / 数据分析 Agent |
| 内部生产力工具 | 公司专用代码评审 Agent / 自动写周报 Agent |
| To B 产品 | "AI + 你的行业"，做成 SaaS 卖给客户 |
| Claude Code 的竞品 | 用 SDK + 自己定制 system prompt + 工具集 |

### 5.5 部署

- **任意你能跑代码的地方**：服务器、Lambda、Docker、客户端 App、CI/CD pipeline
- 通常会暴露成自己的 API / 网页 / Slack 机器人 / 钉钉机器人

### 5.6 收费

- SDK 本身免费
- 实际调用 Claude 的成本走 **Claude API token 计费**

### 5.7 典型用户

- **Agent 应用开发团队**
- 想做 AI 产品的创业公司
- "**我要造一个我自己的 Agent，处理我的业务场景**"

---

## 六、四者综合对比

### 6.1 核心维度对比表

| 维度 | Claude.ai / Desktop | Claude API | Claude Code | Agent + SDK |
|---|---|---|---|---|
| **形态** | 网页 + 桌面 App | HTTP REST 接口 | CLI / IDE 插件 | 你自己写的程序 |
| **使用方式** | 打开网页就用 | 写代码调 `messages.create()` | 在终端里聊天/敲命令 | 写代码 + 运行你的应用 |
| **是否要写代码** | ❌ 不用 | ✅ 必须 | ❌ 不用 | ✅ 必须 |
| **目标场景** | 通用对话 | 任意自定义集成 | 写代码、改代码 | 业务专用 Agent |
| **是否有 Agent Loop** | ✅（产品内置）| ❌ 自己写 | ✅（产品内置）| ✅（SDK 提供） |
| **是否自带工具** | ✅ web_search/Artifacts | ❌ 仅声明，要自己实现 | ✅ Read/Write/Edit/Bash 等 | 部分内置，其余自加 |
| **是否支持 MCP** | ✅ Desktop 支持 | ❌ 自己接 | ✅ | ✅ |
| **是否支持 Skills** | ✅ | ✅ | ✅ | ✅ |
| **典型用户** | 学生/文员/PM | 后端工程师 | 程序员 | Agent 开发团队 |
| **付费方式** | 订阅 ($20-100/月) | 按 token 计费 | 订阅 / API 计费 | 按 token 计费 |
| **谁造的** | Anthropic | Anthropic 提供接口 | Anthropic 用 SDK 造的 | **你**用 SDK 造的 |

### 6.2 抽象层级和定制度对照

```
开箱即用 ◀────────────────────────────────────────▶ 完全自定义
   │                                                       │
Claude.ai      Claude Code      Agent + SDK         Claude API
(零代码)      (零代码)          (写代码)            (写代码)
                                                          
  通用聊天      编程场景          任意业务            最大灵活
                                                          
   ▲              ▲                  ▲                   ▲
   └──────────────┴──────────────────┴───────────────────┘
            底层都是 Claude API + Claude 模型
```

---

## 七、API、SDK、Agent SDK 三个名词的层级关系

很多人把 **API** 和 **SDK** 搞混，再加上 **Claude Agent SDK**，三个名词容易乱。

```
┌─────────────────────────────────────────────────────┐
│  你的应用代码（你写）                                │
└───────────────┬─────────────────────────────────────┘
                │ 调用 Agent SDK 提供的高级方法
                ↓
┌─────────────────────────────────────────────────────┐
│  Claude Agent SDK    ← Agent 专用高级封装            │
│  - Agent Loop                                        │
│  - Tool 注册 / 调度                                  │
│  - Skills 加载                                       │
│  - MCP 客户端                                        │
│  - 上下文管理 / Subagent / Hooks                     │
└───────────────┬─────────────────────────────────────┘
                │ 内部调用 Anthropic SDK
                ↓
┌─────────────────────────────────────────────────────┐
│  Anthropic SDK (anthropic-py / anthropic-sdk-ts)    │
│  - 把 HTTP API 封装成各语言 binding                  │
│  - 类型校验 / 重试 / 流式 SSE                        │
└───────────────┬─────────────────────────────────────┘
                │ 内部发 HTTP 请求
                ↓
┌─────────────────────────────────────────────────────┐
│  Claude API          ← 最底层 HTTP 接口              │
│  POST /v1/messages                                   │
└─────────────────────────────────────────────────────┘
```

### 三层 SDK/API 对比

| 层 | 封装级别 | 自动 Agent Loop | 自动 Tool 调度 | 典型代码量 |
|---|---|---|---|---|
| **裸 API** | 0（HTTP） | ❌ | ❌ | 几百行 |
| **Anthropic SDK** | 中（语言 binding） | ❌ | ❌ | 几十行 |
| **Claude Agent SDK** | 高（Agent 框架） | ✅ | ✅ | 几行 |

### 用代码看三层差异

**裸 API**：

```python
import requests
r = requests.post(
    "https://api.anthropic.com/v1/messages",
    headers={"x-api-key": "...", "anthropic-version": "2023-06-01"},
    json={"model": "claude-sonnet-4-5", "max_tokens": 1024,
          "messages": [{"role": "user", "content": "Hello"}]}
)
print(r.json()["content"][0]["text"])
```

**Anthropic SDK**：

```python
from anthropic import Anthropic
msg = Anthropic().messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)
print(msg.content[0].text)
```

**Claude Agent SDK**：

```python
from claude_agent_sdk import query

async for msg in query(
    prompt="帮我把当前目录所有 .py 文件统计一下行数",
    options={"allowed_tools": ["Read", "Bash", "Glob"]}
):
    print(msg)
```

→ 越往上层封装越多、代码越短，但**牺牲一些灵活度**。

---

## 八、决策树：你该用哪个？

```
┌─ 你不写代码？ ──────────────────────► Claude.ai / Claude Desktop
│
├─ 你是程序员，要 AI 帮你写代码？ ─────► Claude Code
│
├─ 你要把 AI 集成到现有 SaaS / 后端？
│   └─ 只是简单总结/生成等单步操作 ───► Claude API（裸调）
│   └─ 要做对话/工具循环/Agent 行为 ──► Claude Agent SDK
│
└─ 你要做一个垂直领域的 Agent 产品？ ──► Claude Agent SDK
       (客服/运维/数据分析/法律/教育...)
```

### 真实场景对照

| 你的需求 | 选哪个 |
|---|---|
| "帮我写一份产品需求文档" | Claude.ai |
| "我要让 AI 帮我修这个 React bug" | Claude Code |
| "给我的 SaaS 加一个 AI 总结邮件功能" | Claude API |
| "公司要做一个'自动处理告警'的运维 Agent" | Claude Agent SDK |
| "我要做一个'AI 法律助手'卖给客户" | Claude Agent SDK |
| "我要每天自动批量总结 1000 篇新闻" | Claude API + Batch API |

---

## 九、Skills 在四种形态中的统一作用

> 这是 Anthropic 想表达的设计哲学：**写一份 Skill，四个产品都能用**。

```
                  你写一个 Skill（YAML metadata + Markdown 内容）
                              │
              ┌───────────────┼───────────────┬────────────────┐
              ↓               ↓               ↓                ↓
       Claude.ai       Claude API       Claude Code      你的 Agent (SDK)
       (项目里上传)    (skills 参数)    (~/.claude/skills/)  (options.skills)
              │               │               │                │
              └───────────────┴───────┬───────┴────────────────┘
                                      │
                              全部能识别同一份 Skill
```

→ Skill 不是绑定在某个产品上的，而是**跨产品的能力资产**。
这意味着：你为公司客服场景写的 Skill，**Claude.ai 能用、Claude Code 能用、你自己用 SDK 造的客服 Agent 也能用**——一次写、到处复用。

---

## 十、一句话总结

| 名称 | 一句话本质 |
|---|---|
| **Claude.ai / Desktop** | **给人用的成品对话产品** |
| **Claude API** | **给程序用的最底层 LLM 接口** |
| **Claude Code** | **Anthropic 用 SDK 造好的"编程 Agent 成品"** |
| **Agent + Claude Agent SDK** | **你用同款 SDK 造的"业务专属 Agent"** |
| **Anthropic SDK** | **API 的语言 binding**（薄封装） |
| **Claude Agent SDK** | **专为 Agent 应用准备的高级框架**（Loop/Tool/Skill/MCP 全包） |

> 类比建房子：
> - **API** ≈ 钢筋水泥（原材料）
> - **Anthropic SDK** ≈ 预制建材（梁柱、墙板）
> - **Claude Agent SDK** ≈ 模块化"装配式建筑"工具包
> - **Claude Code / Claude.ai** ≈ Anthropic 已经盖好的精装样板房（拎包入住）
> - **你用 SDK 自建的 Agent** ≈ 用 Anthropic 给的工具包**给自己造一栋专属办公楼**

---

## 十一、关联文档

- [`skills.md`](./skills.md)：Skills 的详细设计与跨产品复用
- [`mcp.md`](./mcp.md)：MCP 协议——四个产品都用它接外部工具
- [`tools_vs_mcp.md`](./tools_vs_mcp.md)：Tool 与 MCP 的本质区别
- [`function_call.md`](./function_call.md)：Function Call —— Tool 调用的底层协议
- [`a2a.md`](./a2a.md)：A2A 协议——多 Agent 之间的协作（不在本文范围内）
