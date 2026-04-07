---
title: "深入理解 Python asyncio：从事件循环到 AI Agent 并发调用"
date: 2025-10-15
tags: ["Python", "asyncio", "AI Agent", "并发"]
---

写 AI Agent 的时候，你一定遇到过这个场景：LLM 返回了两个 tool call，比如同时查天气和查日历。它们之间没有依赖关系，完全可以并发执行。但如果你用 `await` 一个一个等，白白浪费了时间。

这篇文章从事件循环讲起，搞清楚 `await` 和 `asyncio.create_task` 的本质区别，最后落到 Agent 开发中的实际用法。

## 事件循环：asyncio 的心脏

事件循环（Event Loop）是 asyncio 的核心调度器。你可以把它想象成一个单线程的任务调度中心——它维护一个任务队列，不断地检查：哪个任务可以往前推进了？

```python
import asyncio

async def say(msg, delay):
    await asyncio.sleep(delay)
    print(msg)

async def main():
    await say("hello", 1)
    await say("world", 1)

asyncio.run(main())
# 总耗时 2 秒：hello(1s) -> world(1s)，串行执行
```

`asyncio.run(main())` 做了三件事：
1. 创建一个事件循环
2. 把 `main()` 作为入口协程扔进去
3. 驱动事件循环直到 `main()` 完成

关键点：**事件循环是单线程的**。它不是靠多线程实现并发，而是靠"在等待 IO 的时候切换到别的任务"来实现并发。当一个协程 `await asyncio.sleep(1)` 的时候，事件循环知道这个任务要等 1 秒，就去执行别的任务了。

## await：挂起当前协程，等结果回来

`await` 的语义很明确：**挂起当前协程，等待目标协程完成，拿到返回值后继续往下走。**

```python
async def fetch_weather(city: str) -> dict:
    await asyncio.sleep(1)  # 模拟 API 调用
    return {"city": city, "temp": "22°C"}

async def main():
    # 串行：先查北京，等结果回来，再查上海
    beijing = await fetch_weather("北京")
    shanghai = await fetch_weather("上海")
    print(beijing, shanghai)
    # 总耗时 2 秒
```

这就像你在餐厅点菜，跟服务员说"先上第一道菜，等我吃完了再上第二道"。效率很低，但逻辑简单，适合有依赖关系的场景。

## create_task：扔进事件循环，不等它完成

`asyncio.create_task()` 的语义完全不同：**把协程包装成一个 Task 对象，立即注册到事件循环中开始调度，但不等它完成。**

```python
async def main():
    # 并发：两个任务同时开始
    task1 = asyncio.create_task(fetch_weather("北京"))
    task2 = asyncio.create_task(fetch_weather("上海"))

    # 两个任务已经在事件循环中跑了
    # 现在 await 拿结果
    beijing = await task1
    shanghai = await task2
    print(beijing, shanghai)
    # 总耗时 1 秒
```

这就像你同时跟两个服务员说"一个上北京烤鸭，一个上小笼包"，两道菜同时做，谁先好谁先上。

### 核心区别

| | `await coroutine()` | `asyncio.create_task(coroutine())` |
|---|---|---|
| 何时开始执行 | 立即执行，但阻塞当前协程 | 立即注册到事件循环，不阻塞 |
| 何时拿到结果 | await 返回时 | 后续 await task 时 |
| 并发能力 | 无，串行 | 有，多个 task 并发 |
| 适用场景 | 有依赖关系的调用 | 无依赖关系的调用 |

## 在 AI Agent 中的实际应用

现在把这些知识用到 AI Agent 开发中。一个典型的 Agent 循环长这样：

```
用户输入 → LLM 思考 → 返回 tool calls → 执行 tools → 结果喂回 LLM → ...
```

LLM 可能一次返回多个 tool call。比如用户问"北京和上海今天天气怎么样"，LLM 会同时吐出两个 `get_weather` 调用。这两个调用之间没有依赖，应该并发执行。

### 串行版本（慢）

```python
async def run_agent(user_input: str):
    messages = [{"role": "user", "content": user_input}]

    while True:
        response = await call_llm(messages)

        if not response.tool_calls:
            print(response.content)
            return

        # 串行执行每个 tool call —— 慢！
        for tool_call in response.tool_calls:
            result = await execute_tool(tool_call)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result
            })

        # 把结果喂回 LLM 继续
```

如果 LLM 返回了 3 个 tool call，每个耗时 1 秒，总共要等 3 秒。

### 并发版本（快）

```python
async def run_agent(user_input: str):
    messages = [{"role": "user", "content": user_input}]

    while True:
        response = await call_llm(messages)

        if not response.tool_calls:
            print(response.content)
            return

        # 并发执行所有 tool call
        tasks = [
            asyncio.create_task(execute_tool(tc))
            for tc in response.tool_calls
        ]
        results = await asyncio.gather(*tasks)

        for tool_call, result in zip(response.tool_calls, results):
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result
            })
```

同样 3 个 tool call，并发执行只需要 1 秒（取决于最慢的那个）。

### asyncio.gather vs create_task

上面用了 `asyncio.gather`，它本质上就是帮你做了 `create_task` + `await` 的组合：

```python
# 这两种写法等价

# 写法 1：手动 create_task
task1 = asyncio.create_task(execute_tool(tc1))
task2 = asyncio.create_task(execute_tool(tc2))
result1 = await task1
result2 = await task2

# 写法 2：gather 一步到位
result1, result2 = await asyncio.gather(
    execute_tool(tc1),
    execute_tool(tc2)
)
```

`gather` 更简洁，适合"一批任务全部完成后再继续"的场景。手动 `create_task` 更灵活，适合需要在中途检查某个任务状态的场景。

## 一个更完整的 Agent 示例

```python
import asyncio
import json

# 定义 tools
async def get_weather(city: str) -> str:
    """模拟天气 API 调用"""
    await asyncio.sleep(1)
    data = {"北京": "晴 22°C", "上海": "多云 25°C", "深圳": "雨 28°C"}
    return json.dumps({"city": city, "weather": data.get(city, "未知")})

async def get_calendar(date: str) -> str:
    """模拟日历 API 调用"""
    await asyncio.sleep(0.8)
    return json.dumps({"date": date, "events": ["团队周会 10:00", "代码评审 14:00"]})

TOOL_MAP = {
    "get_weather": get_weather,
    "get_calendar": get_calendar,
}

async def execute_tool(tool_call) -> str:
    """执行单个 tool call"""
    func = TOOL_MAP[tool_call.function.name]
    args = json.loads(tool_call.function.arguments)
    return await func(**args)

async def run_agent(user_input: str):
    messages = [{"role": "user", "content": user_input}]

    while True:
        response = await call_llm(messages)

        if not response.tool_calls:
            return response.content

        # 关键：并发执行所有 tool calls
        results = await asyncio.gather(
            *[execute_tool(tc) for tc in response.tool_calls]
        )

        # 组装结果
        messages.append(response.message)
        for tool_call, result in zip(response.tool_calls, results):
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result,
            })

# 用户问 "北京天气怎么样，顺便看看我今天有什么会"
# LLM 返回两个 tool call: get_weather("北京") + get_calendar("2025-10-15")
# create_task 让它们并发执行，1 秒搞定，而不是 1.8 秒
```

## 什么时候不该并发

不是所有 tool call 都能并发。如果 tool call 之间有依赖关系，必须串行：

```python
# 场景：先搜索文件，再读取搜索到的文件
# 这两步有依赖，必须串行
search_result = await execute_tool(search_call)  # 先搜索
read_result = await execute_tool(read_call)       # 再读取
```

在 Agent 框架中，通常的策略是：**LLM 单次返回的多个 tool call 并发执行，不同轮次的 tool call 串行执行。** 因为 LLM 在同一轮返回多个 tool call，说明它认为这些调用之间没有依赖。

## 总结

- **事件循环**是 asyncio 的调度中心，单线程通过切换任务实现并发
- **`await`** 串行等待，适合有依赖的场景
- **`create_task`** 把任务扔进事件循环并发跑，适合无依赖的场景
- **在 Agent 开发中**，LLM 同一轮返回的多个 tool call 应该用 `create_task` / `gather` 并发执行，显著降低延迟
