---
title: "用 PostHog 监控你的 AI Agent，比 Sentry 好用太多"
date: 2025-10-20
tags: ["AI Agent", "PostHog", "可观测性", "LLM"]
---

最近在公司做一个 Agent 项目，跑通了基本流程之后，遇到一个很现实的问题：线上用户反馈"回答不对"或者"响应太慢"，我根本没法排查。日志里只有零散的 print，完全看不出一次对话里到底调了几次模型、每次花了多久、tool call 有没有报错。

我需要一个能看清 Agent 每一步在干什么的监控工具。

## 先试了 Sentry，体验很差

第一反应是 Sentry，毕竟之前后端项目一直在用。Sentry 最近也加了 AI 相关的 tracing 功能，看文档说支持 OpenAI SDK 的自动 instrument。

实际接进去之后发现几个问题：

- Sentry 的 AI tracing 本质上还是把 LLM 调用当成普通的 HTTP span 来处理，看到的信息很有限
- 没有专门针对 token 用量、prompt/completion 的分析面板
- trace 里看不到 tool call 的输入输出细节
- 整体感觉就是在一个传统 APM 工具上硬加了个 AI 标签，不是为这个场景设计的

用了两天就放弃了。

## 发现 PostHog 的 LLM Analytics

后来偶然看到 PostHog 出了 LLM observability 的功能，抱着试试的心态接了一下，发现这东西是真的好用。

PostHog 把 AI Agent 的监控拆成了三层：

- **Trace** — 一次完整的用户对话
- **Generation** — 每一次 LLM API 调用
- **Span** — 每一次 tool 执行

这个层级结构跟 Agent 的实际运行方式完全对应。一个用户提问进来，Agent 可能调 3 次模型、执行 5 个 tool，这些全部串在一个 trace 下面，一目了然。

## 接入过程

接入非常简单，核心就是在几个关键节点上报事件。

初始化一个 client：

```python
from posthog import Posthog

client = Posthog(
    api_key="phc_xxx",
    host="https://us.i.posthog.com"
)
```

每次 LLM 调用完成后，上报一个 `$ai_generation` 事件：

```python
client.capture(
    distinct_id=user_id,
    event="$ai_generation",
    properties={
        "$ai_model": "gpt-4o",
        "$ai_provider": "openai",
        "$ai_input": messages,
        "$ai_output_choices": response.choices,
        "$ai_input_tokens": usage.prompt_tokens,
        "$ai_output_tokens": usage.completion_tokens,
        "$ai_latency": 1.23,
        "$ai_trace_id": trace_id,
        "$ai_tools": tools,
    }
)
```

tool 执行完上报一个 `$ai_span`：

```python
client.capture(
    distinct_id=user_id,
    event="$ai_span",
    properties={
        "$ai_span_name": "get_weather",
        "$ai_input": tool_args,
        "$ai_output": tool_result,
        "$ai_latency": 0.45,
        "$ai_trace_id": trace_id,
    }
)
```

整个对话结束后上报一个 `$ai_trace`，把整条链路收口。

我在项目里的做法是把这些上报逻辑封装在 LLM provider 的基类里，所有模型调用自动上报，不需要业务代码里到处埋点。tool 执行的上报放在 agent loop 的回调里，也是自动的。

## 实际效果

接进去之后，打开 PostHog 的 LLM Analytics 面板，能看到所有 trace 的列表：

![trace 列表](/images/posthog/trace-list.png)

每一行就是一次用户对话，能看到总耗时、token 用量、是否报错。

点进去某一条 trace，能看到完整的调用链：

![trace 详情](/images/posthog/trace-detail.png)

这个视图太有用了。之前用户说"回答慢"，我完全不知道慢在哪。现在一看 trace，发现是某个 tool call 卡了 8 秒，一下就定位到问题了。

每次 LLM 调用的 input/output 都能展开看，包括传了什么 system prompt、模型返回了什么 tool call、token 用了多少。排查"回答不对"的问题，直接看 prompt 和 response 就行，不用再去翻日志了。

## 还有个 AI 助手能直接分析数据

PostHog 还有一个 AI 功能，可以直接用自然语言问它问题，它会根据你记录的数据给出分析：

![AI 分析](/images/posthog/ai-chat.png)

比如问"过去一周哪个 tool 的错误率最高"、"平均每次对话消耗多少 token"，它直接给你答案。省得自己写查询了。

## 几个实践建议

用了一段时间，总结几点：

**1. 用 contextvars 传递 trace_id**

Agent 的调用链是异步的，trace_id 需要在整个调用链里传递。用 Python 的 `contextvars` 最干净：

```python
from contextvars import ContextVar

_posthog_ctx: ContextVar[PostHogContext] = ContextVar("posthog_ctx")

# 请求入口设置
ctx = PostHogContext(trace_id=uuid4().hex, distinct_id=user_id)
token = _posthog_ctx.set(ctx)

# 任何地方都能拿到
ctx = _posthog_ctx.get()
```

**2. 大内容要截断**

LLM 的 input/output 可能很大，特别是带图片或长文档的场景。上报前做截断，不然 PostHog 的事件会很臃肿：

```python
# 单条消息限制 8KB，tool call 参数限制 4KB
content = content[:8192] if len(content) > 8192 else content
```

**3. 加个隐私开关**

如果你的 Agent 处理敏感数据，加一个 privacy mode，开启后只上报 token 数量和耗时，不上报具体的 prompt 和 response 内容。

**4. 上报逻辑不要阻塞主流程**

所有 PostHog 的 capture 调用都应该是 fire-and-forget 的。PostHog 的 Python SDK 本身就是异步批量发送的，但你自己的封装逻辑也要注意别 await 不必要的东西。

## 和 Sentry 的对比

| | PostHog | Sentry |
|---|---|---|
| LLM 调用追踪 | 专门设计，字段完整 | 当 HTTP span 处理 |
| Token 用量分析 | 内置面板 需要自己加 tag |
| Tool call 追踪 | 原生支持 span | 需要手动 instrument |
| Trace 可视化 | 按 Agent 逻辑展示 | 按传统 APM 展示 |
| AI 数据分析 | 内置 AI 助手 | 无 |
| 定价 | 免费额度够用 | 免费额度够用 |

不是说 Sentry 不好，它在传统后端监控上依然很强。但在 AI Agent 这个场景下，PostHog 明显更懂你需要什么。

如果你也在做 Agent 开发，推荐试试 PostHog 的 LLM Analytics，接入成本很低，但能省掉大量排查问题的时间。
