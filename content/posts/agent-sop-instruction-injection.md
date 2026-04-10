---
title: "Agent 状态机的指令注入时机：研究了 5 个框架后的最优解"
date: 2026-04-10
tags: ["AI Agent", "状态机", "Prompt Engineering", "Prompt Caching", "LLM"]
---

给 Agent 装了状态机之后，马上碰到一个新问题：**状态切换了，但 Agent 不知道新步骤该干嘛。**

上一篇讲了怎么用 Burr 风格的 FSM 给 Agent 编排多步骤工作流。状态机本身跑得很顺，但实际跑起来发现一个很蛋疼的 timing 问题——Agent 在对话中途推进了状态机，新步骤的指令（instruction）却要等下一轮用户消息才能注入到 LLM 上下文里。

中间这一轮，Agent 就在瞎猜。

## 问题出在哪

先看我们的 Agent Loop 结构：

```
用户发消息
    ↓
读 SOP 当前步骤 → 注入 instruction 到 messages
    ↓
构建 messages → 调 LLM
    ↓
LLM 调 tool → 执行 → 结果追加到 messages → 再调 LLM → ...
    ↓
LLM 输出文字回复 → 返回给用户
```

问题在于：instruction 只在每轮开头注入一次。如果 Agent 在 executor 循环里调了 `sop(action="advance")`，状态机从 A 推进到了 B，但 messages 里注入的还是 A 的 instruction。LLM 在同一轮里看不到 B 的指引。

举个具体例子：

```
轮 1: 用户说"策略定了，开始写脚本"
  → 注入 review_strategy 的 instruction
  → Agent 调 sop(advance) → 状态推进到 build_script
  → Agent 回复"好的，开始搭建工作流"
  → 但它根本没看到 build_script 的 instruction
  → 它不知道应该调 subagent 去搭建工作流

轮 2: 用户随便说点什么
  → 这时才注入 build_script 的 instruction
  → Agent 才知道该调 subagent
```

中间白白浪费了一轮。

## 别人怎么做的

研究了 5 个主流框架，发现大家处理"状态转换 + 指令注入"的方式分成三个流派。

### 流派一：每步独立 LLM 会话

LangGraph 和 CrewAI 走的是这条路。

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Step A  │────→│  Step B  │────→│  Step C  │
│          │     │          │     │          │
│ 独立 LLM │     │ 独立 LLM │     │ 独立 LLM │
│ 独立 prompt│    │ 独立 prompt│    │ 独立 prompt│
│ 独立 tools │    │ 独立 tools │    │ 独立 tools │
└──────────┘     └──────────┘     └──────────┘
      │                │                │
      └────── state 通过 shared dict 传递 ──┘
```

每个步骤就是一个独立的函数，自己构建 LLM 请求，自己决定用什么 system prompt。框架不管 prompt 怎么写，它只管状态转换和数据传递。

LangGraph 里长这样——每个 node 是个普通函数：

```python
def researcher_node(state):
    messages = [SystemMessage("你是研究员")] + state["messages"]
    return {"messages": [llm.invoke(messages)]}

def editor_node(state):
    messages = [SystemMessage("你是编辑")] + state["messages"]
    return {"messages": [llm.invoke(messages)]}
```

CrewAI 更极端——每个 task 执行前直接 `messages.clear()`，完全重置对话历史。前一个 task 的输出作为字符串拼接到下一个 task 的 prompt 里。

这种方式的好处是干净，每步都是全新的 LLM 调用，不存在"旧 instruction 残留"的问题。坏处是丢失了对话上下文的连续性。

### 流派二：单循环 swap 指针

OpenAI Agents SDK 走的是这条路，也是跟我们架构最接近的。

```
while True:
    ┌─────────────────────────────────┐
    │  system_prompt = current_agent  │ ← 每轮重新获取
    │  .get_system_prompt()           │
    │                                 │
    │  response = call_llm(           │
    │    system_prompt,               │
    │    input_items                  │
    │  )                              │
    │                                 │
    │  if handoff:                    │
    │    current_agent = new_agent    │ ← swap 指针
    │    continue ──────────────────────→ 下一轮用新 instruction
    │                                 │
    │  if final_output:               │
    │    break                        │
    └─────────────────────────────────┘
```

核心思路：一个 `while True` 循环，每轮开头从 `current_agent` 获取 system prompt。当发生 handoff（类似我们的状态推进）时，swap `current_agent` 指针，`continue` 回到循环顶部。下一轮自然就用新 agent 的 instruction 了。

对话历史默认全量携带，也可以通过 `input_filter` 过滤或 `nest_handoff_history` 压缩。

### 流派三：Halt + 外部控制

Burr 走的是这条路。

```
caller:
    action, result, state = app.run(
        halt_after=["ai_response"]
    )
    # 拿到结果，决定下一步
    action, result, state = app.run(
        halt_after=["ai_response"],
        inputs={"prompt": new_input}
    )
```

框架在状态边界主动 yield 控制权给调用方。调用方决定什么时候重新进入、传什么输入。每个 action 是独立函数，自己构建 LLM 调用。

### 共同点

研究完这 5 个框架，发现一个关键共识：

**没有任何框架把新步骤的 instruction 塞到 tool result 里。**

所有框架都是同一个模式：状态转换 → 新一轮 LLM 调用 → 新 instruction 在新调用的 prompt 里注入。区别只在于循环怎么组织（break/restart vs swap/continue vs yield/resume）。

## 我们的方案：外层循环 + 模拟 user mes借鉴 OpenAI Agents SDK 的 swap + continue 模式，在 `_process_message` 里加了一个外层循环：

```
用户发消息
    ↓
┌─ while True: ──────────────────────────────────┐
│                                                 │
│  读 SOP 当前步骤 → 记录 step_before             │
│  注入 instruction 到 messages                   │
│  构建 messages → 跑 executor 循环               │
│  检查 auto-advance                              │
│                                                 │
│  step_before == step_after?                     │
│  ├── 是 → break，返回回复给用户                  │
│  └── 否 → 状态推进了！                           │
│       保存这轮对话到 session                     │
│       用新步骤的 instruction 作为下一轮的         │
│       user message → continue                   │
│                                                 │
└─────────────────────────────────────────────────┘
    ↓
返回最后一轮的回复给用户
```

关键设计：状态推进后，把新步骤的 `build_context(state)` 输出（包含进度条 + 当前步骤 instruction + 已收集数据）直接作为下一轮的 user message。对 LLM 来说，就像用户发了一条新消息一样。

这样每次状态转换后，LLM 都能在新一轮里看到完整的新 instruction，不会瞎猜。

走一遍实际流程：

```
轮 1（外层循环 #1）:
  step_before = review_strategy
  注入 review_strategy instruction
  Agent 调 sop(advance, {approved: true})
  → 状态推进到 build_script
  Agent 回复"好的，策略确认了"
  step_before ≠ step_after → 保存这轮 → continue

轮 2（外层循环 #2）:
  step_before = build_script
  注入 build_script instruction
  user message = build_script 的完整 instruction
  Agent 看到"调 subagent 搭建脚本仿写工作流"
  Agent 调 subagent → 搭建工作流 → 运行
  Agent 回复结果
  auto-advance → review_script
  step_before ≠ step_after → 保存 → continue

轮 3（外层循环 #3）:
  step_before = review_script
  注入 review_script instruction
  Agent 回复"脚本出来了，你看看，顺便上传产品图"
  step_before == step_after → break → 返回给用户
```

一条用户消息触发了 3 轮 executor 循环，Agent 从"确认策略"一路执行到"展示脚本等用户确认"，中间没有任何瞎猜。

## 指令放在哪：末尾 user message 是最优解

外层循环解决了"什么时候注入"的问题，但还有一个问题：**instruction 放在 messages 数组的什么位置？**

三个选项：

```
方案 A: [system(静态 + instruction)] → [history] → [user_new]
方案 B: [system(静态)] → [user_dynamic(instruction)] → [history] → [user_new]
方案 C: [system(静态)] → [history] → [user_new(instruction + 用户输入)]
```

从两个维度分析：Prompt Caching 命中率和 LLM 注意力分布。

### Prompt Caching：前缀匹配决定一切

OpenAI、Anthropic、Google 三家的 Prompt Caching 都基于前缀匹配——从 messages 数组第一个 token 开始连续匹配。一旦某个位置内容变了，后面全部缓存失效。

OpenAI 官方说得很直白：

> "Cache hits are only possible for exact prefix matches within a prompt. Structure prompts with **static content first, variable content last**."

Anthropic 的缓存还有级联效应——system prompt 变了，不光 system cache miss，连 messages cache 也跟着 miss。缓存命中的成本是基础输入的 10%，失效的代价很高。

三个方案的缓存表现：

```
方案 A: system prompt 每轮变 → 全部缓存失效 ✗
方案 B: system 不变 ✓，但 dynamic message 变 → history 全部失效 ✗
方案 C: system 不变 ✓，history 不变 ✓，只有末尾一条是新的 ✓
```

方案 C 的缓存失效范围最小。

### LLM 注意力：U 型曲线

斯坦福的 "Lost in the Middle" 论文（Liu et al., 2023）发现了一个 U 型注意力曲线：LLM 对上下文开头和末尾的信息注意力最强，中间最弱。

具体数据：GPT-3.5-Turbo 在 20 文档设置下，信息在开头时准确率约 75%，中间降到 55%，末尾回升到 72%。中间位置性能下降超过 20%。

Anthropic 官方也给了量化数据：

> "Queries at the end can improve response quality by **up to 30%** in tests, especially with complex, multi-document inputs."

方案 B 的 dynamic message 放在 history 前面，随着对话增长会逐渐滑入中间位置，注意力衰减。方案 C 永远在末尾，不受对话长度影响。

### 对比

| 维度 | A. System 末尾 | B. History 前 | C. 末尾 User Message |
|------|--------------|-------------|-------------------|
| System cache | ✗ miss | ✓ hit | ✓ hit |
| History cache | ✗ 全部 miss | ✗ 全部 miss | ✓ 全部 hit |
| 注意力强度 | 最强 | 中等，会衰减 | 很强，稳定 |
| 随对话增长 | 不变 | 下降 | 不变 |

方案 C 是唯一一个在缓存和注意力两个维度都不妥协的方案。

### 实际注入方式

把 SOP instruction 拼在用户输入前面，作为最后一条 user message 的一部分：

```
<sop_instruction>
# 当前 SOP: UGC 视频制作

## 进度
  [x] 意图分析
  [x] 搜索对标
  [>] 制定创意策略  ← 当前
  [ ] 脚本仿写
  ...

## 当前步骤: 制定创意策略
先调用 media_understanding 分析对标视频...

## 已收集数据
  - requirements: 唇釉UGC带货视频，投TikTok美国市场
  - reference_video: NYX Lip IV...
</sop_instruction>

---

策略定了，开始写脚本
```

用 XML 标签包裹，跟用户输入明确分隔。LLM 看到这条消息就知道：当前在哪一步、该做什么、之前收集了什么数据、用户说了什么。

## 防护机制

外层循环有几个边界情况要处理：

**无限循环防护**：`_MAX_SOP_LOOPS = 5`，防止 SOP 定义 bug 导致死循环。正常流程不会超过 2-3 轮。

**SOP 完成**：`completed=True` 时直接 break，不再循环。

**Memory 和 Context Summary**：只在第一轮算，后续轮复用。`_retrieve_memory` 有 API 调用开销，用户消息没变不需要重复检索。

**Media**：用户上传的图片/视频只在第一轮传，后续轮 `_sop_current_media = None`。

**Session 持久化**：每轮循环结束都 `_save_turn` + `sessions.save`，中间轮的对话也会存到 session history。下一轮 `session.get_history()` 能拿到完整的历史。

## 整体架构

把外层循环和 executor 内层循环放在一起看：

```
用户消息进来
    │
    ▼
┌─ 外层循环（SOP 步骤级）─────────────────────────┐
│                                                  │
│  注入当前步骤 instruction                         │
│      │                                           │
│      ▼                                           │
│  ┌─ 内层循环（executor，tool 调用级）──────────┐  │
│  │                                             │  │
│  │  调 LLM → 有 tool_calls?                   │  │
│  │  ├── 是 → 执行 tool → 结果追加 messages     │  │
│  │  │        → 再调 LLM → ...                  │  │
│  │  └── 否 → 输出文字 → break                  │  │
│  │                                             │  │
│  └─────────────────────────────────────────────┘  │
│      │                                           │
│      ▼                                           │
│  检查状态机是否推进了                              │
│  ├── 没推进 → break，返回回复给用户               │
│  └── 推进了 → 保存这轮 → 用新 instruction         │
│               作为 user message → continue        │
│                                                  │
└──────────────────────────────────────────────────┘
    │
    ▼
返回最后一轮回复
```

内层循环负责单步骤内的 tool 调用迭代，外层循环负责步骤间的 instruction 切换。两层各管各的，职责清晰。

这个模式跟 OpenAI Agents SDK 的 handoff 机制本质上是一样的——检测到状态变化 → 保存当前轮 → 用新 instruction 重新跑一轮。只不过他们是 swap agent 指针，我们是 swap SOP 步骤的 instruction。
