# Function Call 全解：原理、实现、与 ReAct 的关系

> 面试常问：
> 1. **「Function Call 和普通的 Prompt + 正则解析有什么区别？」**
> 2. **「LLM 自己执行 Function Call 吗？」**
> 3. **「如果同时触发多个 Function Call 怎么处理？」**
>
> 本文先把原文校对一遍并补全细节，再回答上面三道面试题，最后专门解答一个高频疑惑：
> **「Function Call 指的是 Agent 中的 ReAct 模式吗？感觉很像啊」**

---

## 目录

- [一、原文校对总览](#一原文校对总览)
- [二、Function Call 是什么？](#二function-call-是什么)
- [三、四步流程详解](#三四步流程详解)
- [四、Parallel Function Call](#四parallel-function-call)
- [五、各厂商格式对比](#五各厂商格式对比)
- [六、面试 Q1：Function Call vs Prompt + 正则解析](#六面试-q1function-call-vs-prompt--正则解析)
- [七、面试 Q2：LLM 自己执行 Function Call 吗？](#七面试-q2llm-自己执行-function-call-吗)
- [八、面试 Q3：多个 Function Call 同时触发怎么处理？](#八面试-q3多个-function-call-同时触发怎么处理)
- [九、深度问答：Function Call 是 ReAct 吗？](#九深度问答function-call-是-react-吗)

---

## 一、原文校对总览

原文整体**技术准确**，没有事实性错误。下面几处建议增强（已在正文中体现）：

| 原文 | 状态 | 增强建议 |
|---|---|---|
| "LLM 自己并不执行函数" | ✅ 准确 | 是核心金句，要加粗 |
| 工具定义 JSON Schema | ✅ 准确 | 补充：Schema 越详细，LLM 调用准确率越高 |
| `tool_calls` 输出格式 | ✅ 准确（OpenAI 格式） | 补充：`arguments` 是 **JSON 字符串**而非对象，需要 `json.loads` |
| 串/并行延迟公式 | ✅ 准确 | 补充并行执行需要应用层用 `asyncio` / `Promise.all` |
| 厂商对比表 | ✅ 准确 | 补充：Gemini / Qwen / DeepSeek 也都已支持 |
| 安全性论述 | ✅ 准确 | 补充：但**参数注入**仍需校验，LLM 可能传出恶意参数 |

---

## 二、Function Call 是什么？

> **一句话定义**：Function Call 是让 LLM **输出结构化的工具调用指令**（一段 JSON），而不是普通文本，再由**应用程序**实际执行该函数。

### 关键认知（必背）

> **🔑 LLM 自己并不执行函数。它只告诉你"我想调用什么函数、传什么参数"，真正执行的是你的代码。**

LLM 在物理上就是一个文本生成模型——它**不能联网、不能访问文件、不能执行 Python**。Function Call 只是一种"约定的输出格式"，让 LLM 用 JSON 而不是自然语言来表达"我想调用 xxx 函数"。

---

## 三、四步流程详解

![Function Call 四步流程](assets/function_call_flow.png)

### Step 1：定义工具（你 → LLM）

把可用工具的 schema 告诉 LLM：

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "获取指定城市的实时天气信息",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string",
              "description": "城市名称，如：北京、上海"
            },
            "date": {
              "type": "string",
              "description": "日期，格式 YYYY-MM-DD，不填则为今天"
            }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

> 💡 **经验**：`description` 写得越具体（包括边界条件、示例），LLM 调用准确率越高。这是工程上最值得花时间的地方。

### Step 2：LLM 判断并生成调用指令（LLM → 你）

LLM 看到用户问"明天北京天气怎么样"，**不会去执行函数**，而是输出：

```json
{
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "get_weather",
        "arguments": "{\"city\": \"北京\", \"date\": \"2026-02-06\"}"
      }
    }
  ]
}
```

> ⚠️ **细节坑**：`arguments` 是**字符串形式的 JSON**（字符串套 JSON），不是直接的 JSON 对象，必须 `json.loads(call.function.arguments)`。

### Step 3：你的代码解析并执行

```python
def handle_tool_calls(tool_calls):
    results = []
    for call in tool_calls:
        func_name = call.function.name
        args = json.loads(call.function.arguments)

        if func_name == "get_weather":
            result = weather_api.get(args["city"], args.get("date"))
        elif func_name == "search_calendar":
            result = calendar_api.search(args["query"])
        else:
            result = {"error": f"unknown tool: {func_name}"}

        results.append({
            "tool_call_id": call.id,
            "role": "tool",
            "content": json.dumps(result, ensure_ascii=False)
        })
    return results
```

> 💡 **生产建议**：用 `pydantic` / `jsonschema` 校验 `args` 后再调真实 API——LLM 可能传错类型甚至恶意参数。

### Step 4：把结果传回 LLM，生成最终回答

```python
messages.append({"role": "assistant", "tool_calls": tool_calls})
messages.extend(tool_results)

final_response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages
)
```

LLM 拿到 `{"weather":"中雨","temp":"14-20°C"}`，再生成自然语言回复："明天北京有中雨，建议带伞。"

---

## 四、Parallel Function Call

GPT-4o / Claude 3.5+ / Gemini 1.5+ 都支持一次返回多个工具调用：

```json
{
  "tool_calls": [
    {"id": "call_1", "function": {"name": "get_weather",       "arguments": "{\"city\":\"北京\"}"}},
    {"id": "call_2", "function": {"name": "search_calendar",   "arguments": "{\"date\":\"明天\"}"}},
    {"id": "call_3", "function": {"name": "get_traffic",       "arguments": "{\"route\":\"上班路线\"}"}}
  ]
}
```

| 执行方式 | 总耗时 |
|---|---|
| 串行 | T = T₁ + T₂ + T₃ |
| 并行 | T = max(T₁, T₂, T₃) |

### 应用层并行的代码骨架（Python）

```python
async def execute_parallel(tool_calls):
    coros = [run_tool(call) for call in tool_calls]
    return await asyncio.gather(*coros, return_exceptions=True)
```

> ⚠️ **注意**：LLM 只是**返回多个 call**，**实际是否并行取决于你的代码**。同步 for 循环写下来仍然是串行。

---

## 五、各厂商格式对比

| | OpenAI / Azure / 兼容 API | Anthropic Claude | Google Gemini |
|---|---|---|---|
| **工具定义** | `tools` + `function` | `tools` + `input_schema` | `tools` + `function_declarations` |
| **调用输出** | `tool_calls` 数组 | `tool_use` content block | `function_call` part |
| **结果传回** | `role: "tool"` + `tool_call_id` | `role: "user"` + `tool_result` content block | `function_response` part |
| **并行支持** | ✅ | ✅（3.5+） | ✅（1.5+） |
| **国内对齐** | DeepSeek / Qwen / 智谱 / Kimi 都兼容 OpenAI 格式 | — | — |

> 实际工程中通常用 **LangChain / LiteLLM** 之类的统一封装，避免对接每家不同格式。

---

## 六、面试 Q1：Function Call vs Prompt + 正则解析

> **问**："Function Call 和普通的 Prompt + 正则解析有什么区别？"

### 0. 先回答一个最常见的初学者疑问

> **「Function Call 是不是就是一套 LLM 输出的 JSON 格式？是不是就是告诉 LLM '应该以何种格式向我返回'？」**

**答：是的，但远不止"用 prompt 让它按格式返回"。**

| 你以为是 | 实际是 |
|---|---|
| 你在 prompt 里写"请返回 JSON：{name, args}" | 你通过 API 的 **`tools` 参数**传一份 JSON Schema |
| 模型靠"指令跟随能力"理解格式，可能违反 | 模型在**训练阶段（SFT + RLHF）**就被专门微调，"看到 tools 字段就要输出对应 JSON"，**这是它的肌肉记忆** |
| 输出可能是文本里夹一段 JSON、可能是 markdown 代码块、可能漏字段 | 模型直接通过**专用输出通道** `tool_calls` 返回纯 JSON，不经过自然语言文本通道 |
| 解析靠正则，脆弱 | 厂商保证 `arguments` 字段是合法 JSON 字符串，`json.loads` 一下就能用 |

**类比**：
- 普通 prompt 让 LLM 按格式返回 = **口头跟员工说"请用 Excel 给我交报表"**——他可能用 Word、可能格式乱、可能写错列
- Function Call = **公司给员工发了一个 Excel 模板文件**，他打开模板，每个单元格都有数据校验——根本填不错

**所以 Function Call 的本质**：
1. **接口层面**：是一套约定的 JSON Schema 协议（你传 `tools`，它返 `tool_calls`）
2. **模型层面**：模型在训练时已经被对齐过这套协议，输出稳定性远高于 prompt 提示
3. **运行层面**：JSON 走的是 API 的**专用结构化输出字段**，不是塞在自然语言回答里——所以你的代码不需要"从一段话里把 JSON 抠出来"

> 一句话：**Function Call = 「JSON Schema 约定」 + 「模型训练对齐」 + 「API 专用输出通道」三位一体**。光是一套 JSON 格式，没有后两条加持，就退化成下文要批评的"老办法"。

---

### 老办法（Function Call 出来之前）

```
[Prompt]
你是一个工具助手。如果用户想查天气，请输出：
CALL: get_weather(city=XXX, date=YYY)

[User] 明天北京天气
[LLM]  CALL: get_weather(city=北京, date=明天)

[你的代码用正则解析这一行]
re.match(r"CALL: (\w+)\((.+)\)", llm_output)
```

### 端到端代码对照：同一个场景两种写法

> **场景**：用户问"明天北京天气"，调 `get_weather(city, date)`。
> 两份代码只关注**真正不同的三步**：① 怎么"告诉" LLM 工具 → ② 怎么解析输出 → ③ 怎么回传结果。

#### A. Prompt + 正则解析（老办法）

```python
SYSTEM = "查天气时只回复：CALL: get_weather(city=XXX, date=YYYY-MM-DD)"

resp = openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "system", "content": SYSTEM},
              {"role": "user",   "content": user_msg}])
text = resp.choices[0].message.content       # ← 拿到的是普通字符串

m = re.search(r"CALL:\s*(\w+)\((.+)\)", text)
name, args_str = m.group(1), m.group(2)
args = dict(kv.split("=") for kv in args_str.split(","))   # 手写解析器

result = get_weather(args["city"], args.get("date"))

openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user",      "content": user_msg},
              {"role": "assistant", "content": f"工具结果：{result}"}])
```

**痛点（直接对应代码 3 步）**：
- ① text 格式可能漂移（多空格 / 套 markdown / 漏括号），模型经常"违反约定"
- ② 正则脆弱：参数值含 `,` 或 `=` 就崩；类型全丢成字符串
- ③ 没有"工具结果"角色，只能假装 `assistant` 塞回，模型可能误以为是自己说的

---

#### B. Function Call（现代标准）

```python
TOOLS = [{"type": "function", "function": {
    "name": "get_weather",
    "parameters": {"type": "object",
        "properties": {"city": {"type": "string"},
                       "date": {"type": "string"}},
        "required": ["city"]}}}]

resp = openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": user_msg}],
    tools=TOOLS)                              # ← 不在 prompt 里
msg = resp.choices[0].message

call = msg.tool_calls[0]
args = json.loads(call.function.arguments)   # 标准 JSON 解析，不需要正则
result = get_weather(args["city"], args.get("date"))

openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": user_msg},
              msg,                                                        # 含 tool_calls
              {"role": "tool", "tool_call_id": call.id,
               "content": json.dumps(result)}])
```

**优势（同样对应 3 步）**：
- ① `tools=` 是 API 专属字段，模型在训练时被对齐过，输出**不会漂移**
- ② `tool_calls` 是结构化字段，`json.loads` 一行解析；类型由 schema 保证
- ③ `role: "tool"` + `tool_call_id` 是正式角色，多工具/多轮严格配对

---

#### 三个关键差异，一眼对比

| 步骤 | Prompt + 正则 | Function Call |
|---|---|---|
| **① 告诉 LLM 工具** | 写在 system prompt 文本里 | API 的 `tools=` 参数（强类型 schema） |
| **② 解析 LLM 输出** | `content` 是文本 → 正则 + 手写解析器 | `tool_calls` 是结构化字段 → `json.loads` |
| **③ 回传工具结果** | 假装 `role: "assistant"` 塞回 | `role: "tool"` + `tool_call_id` 配对 |

### 五大本质区别

| 维度 | Prompt + 正则 | Function Call |
|---|---|---|
| **输出可靠性** | LLM 经常忘记格式，多/少空格、漏括号 | 模型经过专门微调输出严格 JSON |
| **解析鲁棒性** | 正则脆弱，参数嵌套就崩 | JSON 结构化，标准 `json.loads` 即可 |
| **类型安全** | 全是字符串，自己 cast 易错 | Schema 驱动，可对接 pydantic 校验 |
| **多工具/多调用** | 难处理"同时调多个"的歧义输出 | 原生 `tool_calls` 数组，并行天然支持 |
| **训练对齐** | 模型没在该格式上训练，准确率低 | OpenAI/Anthropic 在 RLHF 阶段专门训过，准确率高 |

### 一句话答

> "Function Call 把'让 LLM 调用工具'从一个**靠 prompt 提示 + 正则解析**的脆弱工程做法，变成了一个**模型层原生支持 + JSON Schema 标准化**的可靠协议。前者像在自然语言里夹塞约定，模型经常违反；后者像给 LLM 装了'函数调用键盘'，输出格式由模型在训练时就被对齐过。"

---

## 七、面试 Q2：LLM 自己执行 Function Call 吗？

> **答：不！LLM 永远不会执行任何函数，它只是输出"我想调用什么"。**

### 为什么这个设计？

| 设计动机 | 说明 |
|---|---|
| **能力边界** | LLM 是 transformer 模型，物理上就只能预测下一个 token，没有进程/网络/文件能力 |
| **安全性** | 如果 LLM 能直接执行函数，提示词注入就能让它格式化你的硬盘；现在再恶意的 prompt 最多让它**输出**一个调用指令，**真正调不调由你的代码决定** |
| **可控性** | 应用层可以校验参数、限流、审计、灰度，相当于在 LLM 和真实世界之间放了一道**人类可控的关卡** |
| **可移植性** | 同一段 LLM 输出，本地环境可调本地 API，生产环境可调生产 API，模型本身不需要知道差别 |

### 安全性的边界

> ⚠️ "LLM 不能直接执行" **不等于** "Function Call 100% 安全"。

仍需在**应用层**做两件事：
1. **参数校验**：LLM 可能输出 `{"city": "'; DROP TABLE users--"}`，必须 schema + 黑名单过滤
2. **权限校验**：用户 A 的 LLM 不能调出查询用户 B 数据的工具，要在工具实现里加身份判断

### 一句话答

> "LLM **绝不执行**函数。它只是把'我想调 get_weather(city=北京)'这件事用 JSON 写出来，**真正调用的是应用层代码**。这个'输出意图 + 应用执行'的解耦设计，是 Function Call 的安全模型基石——但参数和权限校验仍是应用层的责任。"

---

## 八、面试 Q3：多个 Function Call 同时触发怎么处理？

### 场景

LLM 一次返回 3 个 call：

```json
"tool_calls": [
  {"id": "1", "name": "get_weather"},
  {"id": "2", "name": "search_calendar"},
  {"id": "3", "name": "get_traffic"}
]
```

### 处理要点

| 维度 | 处理方式 |
|---|---|
| **并发模型** | `asyncio.gather` / `Promise.all` / Java `CompletableFuture` 并行调；总耗时 = max |
| **错误隔离** | 用 `return_exceptions=True`，一个挂了其他继续；失败的 tool 返回 `{"error": "..."}` 而不是抛异常 |
| **顺序依赖** | 如果 `tool 2` 依赖 `tool 1` 的结果，**LLM 通常不会一次返回**——它会先返回 1，等结果再返回 2；如果你发现明显有依赖却被并行返回了，需要在 prompt 里强调"按依赖顺序逐步调用" |
| **结果回传顺序** | 必须用 `tool_call_id` 把每个 result 对回原 call，不能搞混 |
| **成本控制** | 单次 N 个 call 都失败的话，要避免无限重试触发雪崩；加重试上限 + 退避 |
| **可观测性** | 每个并行 call 都打 trace，排查时知道是哪个慢 / 错 |

### 代码骨架

```python
async def execute_parallel(tool_calls):
    async def run_one(call):
        try:
            args = json.loads(call.function.arguments)
            result = await TOOL_REGISTRY[call.function.name](**args)
            return {
                "tool_call_id": call.id,
                "role": "tool",
                "content": json.dumps(result),
            }
        except Exception as e:
            return {
                "tool_call_id": call.id,
                "role": "tool",
                "content": json.dumps({"error": str(e)}),
            }

    return await asyncio.gather(*(run_one(c) for c in tool_calls))
```

### 一句话答

> "多个 Function Call 用 `asyncio.gather` 并行执行，**总耗时降到最长那一个**；用 `tool_call_id` 把结果对回去；失败要错误隔离不能让一个挂了拖死整批；有依赖关系的 LLM 通常不会一次返回，靠多轮 round trip 自然解决。"

---

## 九、深度问答：Function Call 是 ReAct 吗？

> **「Function Call 指的是 Agent 中的 ReAct 模式吗？感觉很像啊」**

**简短答**：**不是。它们是不同抽象层的概念，但在现代 Agent 中经常组合使用。**

### 1. 抽象层完全不同

| 概念 | 抽象层 | 本质 |
|---|---|---|
| **Function Call** | **协议层 / API 接口层** | LLM 输出"我想调用 X"的**数据格式标准**（JSON schema） |
| **ReAct** | **Agent 架构层 / 控制流模式** | "Thought → Action → Observation"反复循环的**应用层程序结构** |

### 2. 类比理解

| | 协议 / 数据格式 | 使用模式 / 程序结构 |
|---|---|---|
| 网络 | HTTP | REST |
| 数据库 | SQL | CRUD 模式 / 仓储模式 |
| **LLM 调工具** | **Function Call** | **ReAct / Plan-and-Execute / Reflection** |

> REST 通常基于 HTTP，但 HTTP 也能做 GraphQL、SOAP；
> ReAct 通常基于 Function Call，但 Function Call 也能用在非 ReAct 的场景里。

### 3. 它们的关系：「ReAct 用 Function Call 实现一步」

ReAct 循环的每一轮长这样：

```
┌─────────────── ReAct 第 N 轮 ───────────────┐
│                                              │
│   规划模块组装 prompt (含历史)               │
│         │                                    │
│         ▼                                    │
│   调 LLM                                     │
│         │                                    │
│         ▼                                    │
│   ★★ LLM 返回 tool_calls (← Function Call) │
│         │                                    │
│         ▼                                    │
│   应用代码执行工具                           │
│         │                                    │
│         ▼                                    │
│   把 observation 写入历史                    │
│                                              │
└──────────────────────────────────────────────┘
```

**Function Call 只是 ReAct 一轮中"LLM 返回动作"那一步的具体实现协议**。
ReAct 是"反复循环 N 次"这件事——它是一个外层的 `while` 循环。

### 4. 四种组合都存在

| | 用 Function Call | 不用 Function Call |
|---|---|---|
| **用 ReAct** | **现代主流** —— LangGraph/CrewAI 默认 | 早期 ReAct（2022 论文）：Prompt 里要求 LLM 输出 `Action: xxx`，正则解析 |
| **不用 ReAct** | 单轮 Function Call —— 用户问"明天北京天气"，LLM 一次返回 `get_weather`，执行后回答即可，**没有循环** | 普通 LLM 对话，根本不调工具 |

### 5. 关键区别一图说清

```
┌───────────────────────────────────────────────────────────┐
│                        Agent 系统                          │
│                                                           │
│   while not done:           ←─── ReAct 循环（外层）      │
│       prompt = build(...)                                 │
│       resp = llm.call(prompt, tools=[...])                │
│       │                                                   │
│       ├─► resp.tool_calls   ←─── Function Call（内层协议）│
│       │       │                                           │
│       │       ▼                                           │
│       │   for call in tool_calls:                         │
│       │       result = TOOLS[call.name](**call.args)      │
│       │                                                   │
│       └─► append(result) → next loop                      │
└───────────────────────────────────────────────────────────┘
```

- **Function Call** 解决"LLM 怎么告诉我它想干什么"
- **ReAct** 解决"我什么时候停止再问 LLM"

### 6. 你为什么会觉得"很像"？

因为现代 ReAct **几乎都用 Function Call** 实现，所以日常看到的代码长得像同一个东西。但区别在于：

| 你看到的 | 是哪个？ |
|---|---|
| `tools=[{...}]`、`tool_calls`、`json.loads(arguments)` | **Function Call**（协议） |
| `while`、`max_steps`、`history.append`、"思考-执行-观察" | **ReAct**（模式） |

### 7. 一句话答

> "Function Call 是 LLM API 协议层的**数据格式标准**——告诉模型'用这个 JSON 格式输出工具调用意图'；ReAct 是 Agent 应用层的**控制循环模式**——'反复让 LLM 决策直到任务完成'。**ReAct 通常以 Function Call 作为每一轮的 LLM 调用协议**，但两者完全是不同抽象层的概念：Function Call 是协议（像 HTTP），ReAct 是模式（像 REST），后者通常基于前者，但不等于前者。"
