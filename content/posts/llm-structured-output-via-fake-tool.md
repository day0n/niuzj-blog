---
title: "让 LLM 同时返回文字和结构化数据：用假 Tool Call 做 Side Channel"
date: 2026-04-08
tags: ["LLM", "Agent", "Tool Use", "Structured Output", "OpenAI"]
---

最近在做 AI Agent 的时候碰到一个很实际的问题：LLM 回复用户的时候，除了正常的文字，还需要同时返回一份结构化的 JSON 给前端渲染 UI 组件。

比如用户问"帮我分析一下这几个模板，推荐最适合的"，Agent 需要：

1. 用文字解释推荐理由（给用户看）
2. 同时返回一个结构化的推荐列表（给前端渲染可点选的卡片）

这两个东西必须在同一轮对话里出来。

## 直觉方案：让 LLM 在文本里嵌 JSON

最容易想到的办法是在 system prompt 里约束 LLM，让它在回复末尾加一段 JSON：

```
你的回复格式：
先用自然语言回答用户，然后在末尾用 ```json 代码块输出推荐数据...
```

这个方案能跑，但很脆弱：

- 你得写正则或者找分隔符从文本里提取 JSON
- LLM 有时候会把 JSON 格式搞乱，多个反引号、漏个逗号
- 流式输出的时候更麻烦，你不知道 JSON 什么时候开始什么时候结束
- prompt 越复杂，LLM 越容易忘记遵守格式

## 正经方案：三家 API 的 Structured Output

先看看三大 LLM 厂商怎么解决"让模型输出结构化数据"这个问题。

**OpenAI** 有原生的 `response_format`：

```bash
curl https://api.openai.com/v1/chat/completions \
  -d '{
    "model": "gpt-4o",
    "response_format": {
      "type": "json_schema",
      "json_schema": {
        "name": "recommendations",
        "strict": true,
        "schema": {
          "type": "object",
          "properties": {
            "items": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "title": {"type": "string"},
                  "reason": {"type": "string"}
                },
                "required": ["title", "reason"]
              }
            }
          },
          "required": ["items"]
        }
      }
    },
    "messages": [{"role": "user", "content": "推荐3个适合电商的工作流模板"}]
  }'
```

`strict: true` 保证输出 100% 符合 schema。

**Gemini** 也有类似的：

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=YOUR_KEY" \
  -d '{
    "contents": [{"role": "user", "parts": [{"text": "推荐3个模板"}]}],
    "generationConfig": {
      "response_mime_type": "application/json",
      "response_schema": {
        "type": "object",
        "properties": {
          "items": {"type": "array", "items": {...}}
        }
      }
    }
  }'
```

**Anthropic (Claude)** 没有原生 structured output，官方推荐用 `tool_choice` 强制调用一个假 tool 来实现。

但这三种方案都有同一个问题：**输出要么是纯文字，要么是纯 JSON，不能同时有。**

用了 `response_format` 之后，模型的 `content` 就是 JSON，没有自然语言了。你没法让模型既说人话又吐 JSON。

## 我的场景：文字 + 结构化数据必须同时返回

回到我的实际需求。我们的 Agent 架构大概是这样：

```
用户消息 → Agent Loop → LLM → 文字回复（SSE 流式推送）
                          ↓
                      Tool Call → 执行 → 结果喂回 LLM
                          ↓
                      Tool 事件 → SSE → 前端渲染 UI 组件
```

Agent 的 executor 有一个 `ToolResult` 类型，tool 执行完可以返回两样东西：

```python
class ToolResult(BaseModel):
    content: str                          # 喂回给 LLM 的文本
    events: list[ToolEventPayload] = []   # 通过 SSE 发给前端的事件
```

`content` 回到 LLM 继续对话，`events` 走 SSE 到前端。两条路天然分离。

我们已经有一个 `search_templates` tool 用了这个机制 — 它查 MongoDB 向量搜索，把搜索结果通过 `events` 发给前端渲染模板卡片，同时把结果文本喂回 LLM 让它生成推荐语。

但那个场景里，结构化数据来自**数据库查询结果**，不是 LLM 生成的。

新需求是：LLM 自己分析完内容后，**自己决定**推荐哪 3 个，这个推荐结果需要以结构化 JSON 的形式给前端。

## 解法：定义一个假 Tool

思路很简单 — 定义一个 tool，它不做任何实际操作，只是把 LLM 传过来的参数（结构化数据）通过事件管道转发给前端：

```python
class ShowRecommendationCards(Tool):
    name = "show_recommendation_cards"
    description = (
        "当你需要向用户推荐内容时，调用此工具展示可点选的推荐卡片。"
        "你必须同时在对话中用文字解释你的推荐理由。"
    )
    parameters = {
        "type": "object",
        "properties": {
            "cards": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "id": {"type": "string"},
                        "title": {"type": "string"},
                        "description": {"type": "string"},
                        "cover_url": {"type": "string"},
                        "score": {"type": "number"},
                    },
                    "required": ["id", "title", "description"],
                },
                "minItems": 1,
                "maxItems": 5,
            }
        },
        "required": ["cards"],
    }

    async def execute(self, **kwargs) -> ToolResult:
        cards = kwargs["cards"]
        return ToolResult(
            content=f"已向用户展示 {len(cards)} 张推荐卡片。",
            events=[ToolEventPayload(
                name="recommendation_cards",
                data={"cards": cards},
            )],
        )
```

这个 tool 的 `execute` 方法几乎什么都没干 — 它只是把 LLM 传过来的 `cards` 参数原样包进 `ToolEventPayload`，通过 SSE 发给前端。

## 实际运行流程

```
用户: "帮我分析一下这几个模板，推荐最适合电商场景的"

LLM 返回:
  content: "根据你的需求，我推荐以下三个模板：\n\n1. **电商主图生成**..."
  tool_calls: [
    {
      name: "show_recommendation_cards",
      arguments: {
        "cards": [
          {"id": "tpl_001", "title": "电商主图生成", "description": "...", "score": 0.95},
          {"id": "tpl_002", "title": "模特试穿", "description": "...", "score": 0.88},
          {"id": "tpl_003", "title": "产品视频", "description": "...", "score": 0.82}
        ]
      }
    }
  ]

Executor 处理:
  1. content → 流式推给前端 → 用户看到文字推荐
  2. tool_call → execute → ToolResult.events → SSE → 前端渲染 3 张可点选卡片
  3. ToolResult.content("已展示3张卡片") → 喂回 LLM → LLM 知道卡片已展示
```

文字和结构化数据从 LLM 出来的那一刻就是分开的，不需要任何提取或解析。

## 为什么 LLM 能同时返回文字和 Tool Call？

这个我实测验证过。用 Gemini 3.1 Flash 测的：

**无依赖的多个 tool call，模型一次全吐出来：**

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=YOUR_KEY" \
  -d '{
    "contents": [{"role": "user", "parts": [{"text": "查北京和东京的天气"}]}],
    "tools": [{"function_declarations": [
      {"name": "get_weather", "parameters": {"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"]}}
    ]}]
  }'
```

返回 `parts` 里有 2 个 `functionCall`，一次性并行吐出。

**有依赖的 tool call，模型只吐第一步：**

```bash
# "查股价，然后算能买多少股"
# 模型只返回 get_stock_price，不会同时调 calculate_shares
# 因为 calculate_shares 的 price_per_share 参数依赖 get_stock_price 的结果
```

模型自己判断依赖关系。Gemini 官方文档说得很明确：

> "Parallel function calling lets you execute multiple functions at once and is used when the functions are **not dependent on each other**."

> "Compositional or sequential function calling allows Gemini to **chain multiple function calls together** to fulfill a complex request."

回到我们的场景，`show_recommendation_cards` 和文字回复之间没有依赖 — 模型可以同时输出 content（文字）和 tool_call（结构化数据）。

## 这个模式的本质

把 tool call 当成一个**类型安全的 side channel**。

正常的 tool 是：LLM 请求执行一个动作 → 你的代码执行 → 结果喂回 LLM。

假 tool 是：LLM 把结构化数据塞进 tool call 的参数 → 你的代码拿到结构化数据转发给前端 → 给 LLM 一个确认消息。

tool 的 `parameters` JSON Schema 就是你的结构化数据契约，LLM 必须按这个格式填参数，不符合 schema 的会被 SDK 拒绝。比从自由文本里提取 JSON 靠谱得多。

Anthropic 官方推荐的 structured output 方案就是这个思路 — 定义一个假 tool，用 `tool_choice` 强制调用，从 `input` 里取结构化数据。我们只是更进一步：不强制调用，让 LLM 自己决定什么时候需要展示卡片，同时保留自然语言回复能力。

## 三家 API 对比

顺便整理一下三家在 structured output 和 function calling 上的差异，做 Agent 的时候经常要查：

| | OpenAI | Gemini | Anthropic |
|---|---|---|---|
| 原生 Structured Output | `response_format.json_schema` | `response_mime_type` + `response_schema` | 没有，用假 tool 实现 |
| Tool Call 有唯一 ID | 有 `call_xxx` | 没有 | 有 `toolu_xxx` |
| arguments 类型 | JSON 字符串 | JSON 对象 | JSON 对象（叫 `input`） |
| finish_reason（有 tool call 时） | `"tool_calls"` | `"STOP"` | `"tool_use"` |
| 可以关闭并行 tool call | `parallel_tool_calls` 参数 | 不支持 | `disable_parallel_tool_use` |
| content + tool_calls 同时返回 | 支持但 content 经常为 null | 支持，都在 parts 里 | 支持，content 数组里混合 text 和 tool_use |

如果你的 Agent 需要同时支持多个 provider，tool call 的格式差异是最头疼的。我们的做法是在 provider 层做归一化，统一转成 OpenAI 格式的 `ToolCallRequest`，executor 只处理一种格式。

## 什么时候用这个模式

适合的场景：

- LLM 回复需要同时包含自然语言和结构化数据
- 前端需要根据结构化数据渲染特定 UI 组件（卡片、表格、图表）
- 你已经有 tool call 的处理链路，不想再搞一套 structured output 的解析逻辑

不适合的场景：

- 每次回复都必须带结构化数据（这种直接用 `response_format` 更干净）
- 结构化数据不是 LLM 生成的，而是来自外部查询（那就用正常的 tool）
- 你的 LLM provider 不支持 tool call（那只能回到正则提取的老路）
