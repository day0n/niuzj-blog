---
title: "给 AI Agent 加状态机，踩了个 instruction 注入的坑"
date: 2026-04-10
tags: ["AI Agent", "状态机", "Prompt Engineering", "Prompt Caching", "LLM"]
---

做 Agent 做到一定复杂度，一定会遇到这个问题：多步骤流程怎么编排。

我们的场景是 UGC 视频制作——用户说"帮我做一条唇釉的带货视频"，Agent 需要走完意图分析、搜索对标、制定策略、仿写脚本、达人选角、生成素材这一整套流程。每一步都有不同的指令，需要调不同的工具，用户还可能在任何一步说"不满意，重来"。

靠 system prompt 里写一大段流程说明让 LLM 自己记住做到哪了？试过，对话一长就乱。跳步、重复、忘记之前收集的信息，各种问题。

所以我们给 Agent 装了一个状态机。

## 为什么是状态机

状态机的核心价值是把"Agent 应该做什么"从 LLM 的记忆里抽出来，变成一个确定性的外部数据结构。

不用状态机的时候，Agent 的行为完全依赖 LLM 对上下文的理解。LLM 需要从几十轮对话历史里推断出"我现在在第几步、下一步该做什么、之前收集了哪些信息"。这在 3-5 轮对话里还行，到了 10+ 轮就不可靠了。

用了状态机之后，每一轮对话开始时，系统从外部状态里读出"当前在哪一步"，把这一步的 instruction 注入到 LLM 的上下文里。LLM 不需要记忆，它只需要执行当前步骤的指令。

这个思路不是我们发明的。研究了 30 多个开源 Agent 框架后发现，所有做多步骤编排的框架本质上都是这个模式：

```
                    ┌─────────────────────┐
                    │    状态机 / 工作流    │
                    │  （确定性，外部维护）  │
                    │                     │
                    │  当前步骤 → instruction
                    │  转移条件 → 下一步    │
                    │  共享数据 → state.data │
                    └──────────┬──────────┘
                               │ 注入 instruction
                               ▼
                    ┌─────────────────────┐
                    │     LLM Agent       │
                    │  （概率性，执行层）   │
                    │                     │
                    │  看到 instruction    │
                    │  → 调用工具          │
                    │  → 回复用户          │
                    │  → 推进状态机        │
                    └─────────────────────┘
```

确定性的编排层告诉 Agent "该做什么"，概率性的 Agent 层决定"怎么做"。两层各管各的。

## 我们的状态机怎么设计的

借鉴了 Burr 框架的 FSM 模式。核心就三个概念：

**Step**：一个步骤，带有 instruction（注入给 LLM 的指令）、transitions（出边）、advance_mode（推进方式）。

**Transition**：一条有向边，带有 condition。condition 用 Burr 的 `when()` 语义——`condition={"approved": True}` 表示当 `state.data["approved"] == True` 时走这条边。`condition=None` 是无条件默认边。

**State**：运行时状态，记录当前步骤、已完成步骤、跨步骤共享数据。存在 session 里，每轮对话自动持久化。

UGC 视频制作的状态图大概长这样：

```
意图分析 ──→ 搜索对标 ──→ 选择对标 ──→ 制定策略 ──→ 审核策略
                ↑          │                          │
                └── 换方向 ─┘              不满意 ←────┘
                                            │
                                                      搭建脚本工作流 ──→ 审核脚本 ──→ 达人选角 ──→ 选择达人
                    ↑              │                        │
                    └── 不满意 ────┘              不满意 ←──┘
                                                   │
                                    ┌── 盲盒 ──────┤ 满意
                                    ↓              ↓
                              生成视频素材    生成达人定型图 ──→ 审核定型图
                                    ↑                            │
                                    └────────────────────────────┘
                                    ↓
                               审片 ──→ 完成
                                    ↑    │
                                    └────┘ 不满意
```

13 个步骤，6 个用户确认门控点。每个门控点都有"满意往前走"和"不满意回退重做"两条路。

推进方式分两种：

**MANUAL**：Agent 主动调 `sop(action="advance", step_data={"approved": true})` 推进。用在需要等用户决策的步骤。

**AUTO**：Agent 调了指定的 tool 后，外层 loop 自动检测并推进。比如 `cast_avatar` 步骤标记了 `auto_advance_on=("match_avatar",)`，Agent 调完 `match_avatar` 后 loop 自动推进到 `select_avatar`。用在 Agent 调完 tool 就该进下一步的场景。

这套东西跑起来之后，状态转换、条件路由、回退都没问题。但很快碰到了一个更深层的问题。

## 踩坑：instruction 注入和状态推进的时序脱节

状态机的 instruction 是在每轮 `_process_message` 开头注入的：

```
_process_message 开始
    │
    ├── 读 session.metadata["sop"] → 当前步骤 A
    ├── build_context(A) → 生成 A 的 instruction
    ├── 注入到 messages
    │
    ├── 启动 executor 循环（LLM ↔ tool 迭代）
    │       │
    │       ├── LLM 调 sop(advance) → 状态 A → B
    │       ├── LLM 继续回复（但 messages 里还是 A 的 instruction）
    │       └── executor 结束
    │
    ├── auto-advance 检查
    └── 返回回复给用户
```

问题很明显：**instruction 注入发生在 executor 循环之前，状态推进发生在 executor 循环之中。** 推进之后，LLM 在同一轮里看不到新步骤的 instruction。

这不是一个边缘 case，而是每次状态推进都会遇到的问题。Agent 推进到新步骤后，要么瞎猜下一步该做什么，要么说一句"好的，下一步我来处理"然后等用户再发一条消息——白白浪费一轮交互。

## 研究了 5 个框架的做法

带着这个问题去看了 LangGraph、CrewAI、OpenAI Agents SDK、Burr、Google ADK 的源码。

发现一个关键共识：**所有框架都是"状态转换 = 新一轮 LLM 调用"。没有任何框架把新 instruction 塞到 tool result 里。**

但实现方式分成三个流派：

### 流派一：Break & Rebuild

LangGraph 和 CrewAI 的做法。每个步骤是一个独立的函数，自己构建 LLM 请求。框架只管状态转换和数据传递，不管 prompt 怎么写。

```
┌─────────┐  state  ┌─────────┐  state  ┌─────────┐
│ Node A  │ ──────→ │ Node B  │ ──────→ │ Node C  │
│         │         │         │         │         │
│ 自己的   │         │ 自己的   │         │ 自己的   │
│ prompt  │         │ prompt  │         │ pr│
│ 自己的   │         │ 自己的   │         │ 自己的   │
│ LLM call│         │ LLM call│         │ LLM call│
└─────────┘         └─────────┘         └─────────┘
```

CrewAI 更激进——每个 task 执行前 `messages.clear()`，完全重置对话历史。前一个 task 的输出作为字符串拼到下一个 task 的 prompt 里。干净，但丢失了对话的连续性。

这种方式不适合我们。我们的 Agent 是一个持续对话的助手，用户在整个 SOP 过程中跟同一个 Agent 聊天，对话上下文不能断。

### 流派二：Swap & Continue

OpenAI Agents SDK 的做法，跟我们的架构最接近。

```
while True:
    system_prompt = current_agent.get_system_prompt()
    response = call_llm(system_prompt, items)

    if handoff:
        current_agent = new_agent    ← swap
        continue                     ← 下一轮用新 prompt
    if final:
        break
```

一个 `while True` 循环。每轮开头从 `current_agent` 获取 system prompt。发生 handoff 时 swap 指针，`continue` 回到循环顶部，下一轮自然用新 agent 的 instruction。对话历史通过 `input_filter` 或 `nest_handoff_history` 控制携带多少。

关键细节：handoff 发生时，当前轮的 LLM 回复会被保存到 items 里，然后才 swap。所以新 agent 能看到上一个 agent 说了什么。

### 流派三：Halt & Return

Burr 的做法。框架在状态边界主动 yield 控制权给调用方。

```
action, result, state = app.run(halt_after=["step_a"])
# 调用方拿到结果，决定下一步
action, result, state = app.run(halt_after=["step_b"], inputs={...})
```

每个 action 是独立函数，自己构建 LLM 调用。框架不管 prompt，只管状态转换和 halt 控制。适合人工介入的场景，但需要外部调用方来驱动循环。

### 核心洞察

三个流派的实现差异很大，但底层逻辑完全一致：

**每次状态转换，都会产生一次新的 LLM 调用，新步骤的 instruction 在这次新调用的 prompt 里注入。**

变化的只是"谁来触发这次新调用"——是框架自动触发（LangGraph 的 node 调度），还是循环内 swap 后 continue（OpenAI SDK），还是外部调用方重新调用（Burr）。

## 最终方案：外层循环 + instruction 作为 user message

借鉴 OpenAI Agents SDK 的 swap + continue 模式，在 `_process_message` 里加了一个外层循环：

```
用户消息进来
    │
    ▼
┌─ 外层循环（SOP 步骤级）──────────────────────────┐
│                                                   │
│  读当前步骤 → 记录 step_before                     │
│  注入 instruction                                 │
│      │                                            │
│      ▼                                            │
│  ┌─ 内层循环（executor，tool 调用级）───────────┐  │
│  │                                              │  │
│  │  调 LLM → 有 tool_calls?                    │  │
│  │  ├── 是 → 执行 tool → 追加结果 → 再调 LLM   │  │
│  │  └── 否 → 输出文字 → break                   │  │
│  │                                              │  │
│  └──────────────────────────────────────────────┘  │
│      │                                            │
│      ▼                                            │
│  auto-advance 检查                                │
│  step_before vs step_after                        │
│  ├── 相同 → break，返回回复给用户                  │
│  └── 不同 → 状态推进了                             │
│       ├── 保存这轮对话到 session                   │
│       ├── 新步骤的 instruction → 下一轮 user msg   │
│       └── continue                                │
│                                                   │
└───────────────────────────────────────────────────┘
    │
    ▼
返回最后一轮回复
```

状态推进后，把新步骤的 `build_context(state)` 输出直接作为下一轮的 user message。这个输出包含进度条、当前步骤 instruction、已收集的数据。对 LLM 来说就像用户发了一条新消息。

走一遍实际流程感受一下：

```
外层循环 #1:
  当前步骤: review_strategy
  用户说: "策略定了"
  Agent 调 sop(advance, {approved: true}) → 推进到 build_script
  Agent 回复: "好的，策略确认了"
  step 变了 → 保存这轮 → continue

外层循环 #2:
  当前步骤: build_script
  user message = build_script 的 instruction
  Agent 看到: "调 subagent 搭建脚本仿写工作流"
  Agent 调 subagent → 搭建 + 运行工作流
  Agent 回复脚本结果
  auto-advance → review_script
  step 变了 → 保存 → continue

外层循环 #3:
  当前步骤: review_script
  Agent 回复: "脚本出来了，你看看，顺便上传产品图"
  step 没变 → break → 返回给用户
```

用户发了一条"策略定了"，Agent 自动走完了三个步骤，最终停在需要用户确认的地方。中间没有任何"瞎猜"的轮次。

## 为什么 instruction 要放在末尾 user message

外层循环解决了"什么时候注入"，但还有一个问题：instruction 放在 messages 数组的什么位置。

这个问题需要从两个维度分析。

### 维度一：Prompt Caching

OpenAI、Anthropic、Google 三家的 Prompt Caching 都基于**前缀匹配**。从 messages 数组第一个 token 开始连续匹配已缓存的序列，某个位置内容变了，后面全部失效。

三种放法的缓存表现：

```
方案 A: instruction 放 system prompt 末尾
  [system(静态 + instruction)] → [history] → [user]
  system 每轮变 → 全部缓存失效

方案 B: instruction 放 history 前的独立 user message
  [system(静态)] → [user_dynamic(instruction)] → [history] → [user]
  system 不变 ✓，但 dynamic msg 变 → history 全部失效

方案 C: instruction 放最后一条 user message
  [system(静态)] → [history] → [user(instruction + 输入)]
  system 不变 ✓，history 不变 ✓，只有末尾一条是新的 ✓
```

Anthr有级联效应——system prompt 变了，system cache 和 messages cache 都 miss。缓存命中成本是基础输入的 10%，失效代价很高。

方案 C 的缓存失效范围最小，只有最后一条消息是增量计算。

### 维度二：LLM 注意力分布

斯坦福的 "Lost in the Middle"（Liu et al., 2023）发现了 U 型注意力曲线：LLM 对上下文开头和末尾的信息注意力最强，中间最弱。中间位置性能下降超过 20%，在某些设置下甚至低于不提供任何文档的基线。

Anthropic 官方给了量化数据：将查询放在末尾可以提升最多 30% 的响应质量。

方案 B 的 dynamic message 放在 history 前面，随着对话增长会逐渐滑入 messages 数组的中间位置，注意力衰减。方案 C 永远在末尾，不受对话长度影响。

### 综合对比

| 维度 | A. System 末尾 | B. History 前 | C. 末尾 User Message |
|------|--------------|-------------|-------------------|
| System cache | ✗  | ✓ hit |
| History cache | ✗ 全部 miss | ✗ 全部 miss | ✓ 全部 hit |
| 注意力强度 | 最强 | 中等，随对话衰减 | 很强，稳定 |
| 成本 | 最高 | 高 | 最低 |

方案 A 注意力最强但缓存全废，方案 B 两头不讨好，方案 C 是唯一一个在两个维度都不妥协的。

实际注入时用 XML 标签包裹，跟用户输入明确分隔：

```xml
<sop_instruction>
# 当前 SOP: UGC 视频制作

## 进度
  [x] 意图分析
  [x] 搜索对标
  [>] 制定创意策略  ← 当前

## 当前步骤: 制定创意策略
先调用 media_understanding 分析对标视频...

## 已收集数据
  - requirements: 唇釉UGC带货视频，投TikTok美国市场
  - reference_video: NYX Lip IV...
</sop_instruction>

---

策略定了，开始写脚本
```

LLM 看到这条消息就知道当前在哪一步、该做什么、之前收集了什么、用户说了什么。

## 几个工程细节

**无限循环防护**：`_MAX_SOP_LOOPS = 5`。正常流程不，但 SOP 定义如果有 bug（比如两个步骤互相跳转），需要兜底。

**Memory 只算一次**：长期记忆检索（`_retrieve_memory`）基于用户原始消息，外层循环的后续轮不需要重复检索。Context summary 同理。

**Session 持久化**：每轮循环结束都保存到 session。下一轮 `session.get_history()` 能拿到上一轮的完整对话，LLM 能看到自己之前说了什么、调了什么 tool。

**SSE 事件流**：外层循环的多轮 executor 都在同一个 SSE 连接里。前端看到的是一个连续的事件流，包含 `sop.state`（当前状态快照）和 `sop.transition`（步骤切换通知）事件，可以实时更新进度 UI。

这套方案跑了一段时间，状态推进后 Agent 能立即看到新 instruction 并执行，不再有"瞎猜"的问题。本质上就是 OpenAI Agents SDK 的 handoff 模式——检测到状态变化 → 保存当前轮 → 用新 instruction 重新跑一轮。他们是 swap agent 指针，我们是 swap SOP 步骤的 instructio
