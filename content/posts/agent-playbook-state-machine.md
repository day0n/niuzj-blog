---
title: "给 AI Agent 装一个状态机：用 Playbook 模式编排多步骤工作流"
date: 2026-04-09
tags: ["AI Agent", "状态机", "Playbook", "工作流", "Python"]
---

前段时间在做 AI Agent 的时候遇到一个问题：用户说"帮我做个 UGC 视频"，Agent 需要分三步走——先生成脚本，再生图，最后生视频。每一步都要构建工作流模块、运行、展示结果给用户确认，不满意还得回退重做。

一开始想的是把整个流程写进 system prompt 里，让 LLM 自己记住做到哪了。试了一下发现不靠谱——对话一长，LLM 就忘了自己在第几步，有时候跳步，有时候重复。

后来研究了一圈主流框架（LangGraph、Burr、CrewAI、PocketFlow），发现大家解决这个问题的思路都一样：**给 Agent 装一个状态机**。

## 什么是状态机

状态机就是一张图：节点是"状态"，边是"转移条件"。系统在任意时刻只能处于一个状态，满足某个条件后跳到下一个状态。

用 UGC 视频制作举例：

```
collect_requirements → build_script → run_script → review_script
                          ↑                              │
                          └──── approved=false ───────────┘
                                                          │ approved=true
                                                          ↓
                        build_image → run_image → review_image
                          ↑                              │
                          └──── approved=false ───────────┘
                                                          │ approved=true
                                                          ↓
                        build_video → run_video → review_video → done
                          ↑                              │
                          └──── approved=false ───────────┘
```

每个 review 节点有两条出边：`approved=true` 走下一个模块，`approved=false` 回退到 build 重做。这就是状态机的全部——状态 + 转移条件。

## 为什么 Agent 需要状态机

核心问题：**谁来记住做到哪了？**

### 方案 A：让 LLM 自己记

把整个流程塞进 prompt，LLM 从对话历史推断当前进度。

问题：
- 对话历史一长就被压缩截断，进度信息丢失
- LLM 可能跳步、重复、忘记回退
- 10 个步骤的完整指令塞进 prompt 占 2000+ token，每轮都浪费

### 方案 B：代码来记

状态存在数据库里，每轮只给 LLM 当前步骤的指令。

```
方案 A（LLM 记）：
  prompt 里塞 10 个步骤的完整指令
  LLM 从对话历史猜 "我大概做到第 6 步了？"

方案 B（代码记）：
  数据库里存 {current_step: "run_image"}
  只注入 "当前步骤：运行图片模块"
```

方案 B 的状态不会丢、不会猜错、不占多余 token。这就是状态机的价值。

## 整体架构

状态机不替换 Agent 的任何能力，它是**叠加**上去的一层：

```
┌─────────────────────────────────────────────────────┐
│  Playbook 层（状态机）                                │
│  - 每轮注入当前步骤指令到 Agent 上下文                 │
│  - Agent 执行完后检查是否自动推进                      │
│  - 不限制 Agent 的任何原有能力                        │
├─────────────────────────────────────────────────────┤
│  Agent 能力层                                        │
│  - 所有原有 tools 可用（搜索、读文件、调子 Agent 等）   │
│  - 所有 skills 可用                                   │
│  - LLM 推理 + tool calling                           │
├─────────────────────────────────────────────────────┤
│  基础设施层                                          │
│  - Agent Loop（每轮处理循环）                         │
│  - Session（对话历史 + metadata 持久化）               │
│  - Prompt Builder（消息组装）                         │
└─────────────────────────────────────────────────────┘
```

Playbook 激活后，Agent 的 tool 集合 = 原有全部 tools + 一个 `playbook` tool。Agent 在 Playbook 模式下依然可以搜索、读文件、加载 skill。用户问了个无关问题，Agent 回答完继续 Playbook——因为状态在数据库里，不会因为聊了别的就丢失。

## 每轮的处理流程

```
用户发消息
  │
  ├─ ① 加载 Session → 读取 metadata 里的 Playbook 状态
  │     → {current_step: "review_script", data: {requirements: "..."}}
  │
  ├─ ② Engine 生成当前步骤的上下文文本
  │     → 进度条 + 当前步骤指令 + 已收集数据摘要
  │
  ├─ ③ 上下文文本注入到 Agent 的消息列表里
  │     （和长期记忆、历史摘要一样的注入机制）
  │
  ├─ ④ Agent 正常执行：LLM 推理 + tool calling
  │     → 看到步骤指令 + 用户消息 → 决定怎么做
  │
  ├─ ⑤ 执行完后检查推进：
  │     ├─ AUTO 步骤：检测 tools_used → 自动推进
  │     └─ MANUAL 步骤：Agent 调了 playbook tool → 已推进
  │
  └─ ⑥ Session 保存 → 状态持久化
```

关键点：**步骤 ④ 里 Agent 的执行方式完全没变**。它还是正常调 LLM、正常调 tool。状态机只是在输入端加了指令，在输出端检查了推进条件。

## 核心设计

### 四个概念

整个系统只有四个概念：

**Transition（转移）**：一条边，从当前步骤到目标步骤，可以带条件。`condition={"approved": true}` 表示满足时走这条边，`condition=None` 表示无条件。

**Step（步骤）**：一个节点，包含注入给 Agent 的指令文本、出边列表、推进模式（auto/manual）。

**Definition（定义）**：一套完整的 Playbook，包含所有步骤和入口步骤 ID。不可变。

**State（状态）**：运行时数据，记录当前在哪一步、已完成哪些步骤、跨步骤共享的数据。可变，存在 Session 里。

### 状态转移机制

Engine 的核心逻辑就一个函数——遍历当前步骤的 transitions，匹配第一条满足条件的：

```
_advance(state):
  当前步骤 = 找到 state.current_step 对应的 Step
  遍历 step.transitions:
    如果 condition 匹配 state.data → 执行转移，返回 true
  没有匹配的 → 返回 false
```

条件匹配的语义借鉴了 Burr 的 `when()`：`condition={"approved": true}` 就是检查 `state.data["approved"] == true`。多个 key 是 AND 关系。`None` 是无条件。

回退也是这个机制——`condition={"approved": false}` 匹配时 target 指向前面的步骤。对 engine 来说"回退"和"前进"没有区别，都是状态转移。

### 上下文注入

Agent 每轮收到的动态上下文长这样：

```markdown
# 当前 Playbook: UGC 视频制作

## 进度
  [x] 收集需求
  [x] 构建脚本模块
  [>] 运行脚本模块  ← 当前
  [ ] 审核脚本
  [ ] 构建图片模块
  ...

## 当前步骤: 运行脚本模块

调用 run_workflow 执行脚本模块。

已收集信息:
  - requirements: 30秒旅行vlog，轻松风格
```

这段文字注入到对话历史前面。Agent 不需要从历史推断进度——进度条明确写在那里。

### 一个 tool 管理全部

Agent 通过一个 `playbook` tool 管理整个生命周期，用 `action` 参数区分操作：

```
playbook(action="start", playbook_id="ugc_video")     → 启动
playbook(action="advance", step_data={"approved": true})  → 推进
playbook(action="advance", step_data={"approved": false}) → 回退
playbook(action="cancel")                                  → 取消
```

tool 的可用 Playbook 列表从 Engine 动态生成——加新 Playbook 不需要改 tool 代码。

### 两种推进模式

**MANUAL**：Agent 调 `playbook(action="advance")` 推进。用于需要判断的步骤，比如收集需求、审核结果。

**AUTO**：Agent 调了特定 tool 后自动推进，不需要额外操作。比如"运行脚本"步骤配置了 `auto_advance_on=("run_workflow",)`，Agent 调完 `run_workflow` 后 engine 自动推进到"审核脚本"。

## 条件分支和回退

transitions 是有序列表，第一条匹配的执行。可以声明任意复杂的路由：

**多分支**：审核后三种走向——满意、微调、重做：

```
review_script 的 transitions:
  → build_image    当 decision="approve"
  → revise_script  当 decision="revise"
  → build_script   当 decision="redo"
```

Agent 调 `playbook(action="advance", step_data={"decision": "revise"})` 时，engine 匹配到第二条，跳到微调步骤。

**多条件 AND**：`condition={"style": "vlog", "quality": "high"}` 表示两个条件同时满足才走。

**兜底 default**：最后一条不带 condition 的 transition 作为兜底，前面都不匹配时走这条。

**分支汇合**：多条分支的最后一步都指向同一个目标步骤，自然汇合。

全部声明式，不需要写 if-else。

## 加新 Playbook 的成本

写一个定义文件，声明步骤和转移条件，注册到列表里。不需要改 engine、不需要改 Agent Loop、不需要改 tool。`playbook` tool 的可用列表自动更新，Agent 自动能看到新选项。

## 这套方案的来源

调研了 30+ 个开源项目后提炼出来的。几个关键参考：

- **[Burr](https://github.com/DAGWorks-Inc/burr)**（Apache 孵化，~2k stars）：纯 FSM 框架，`Action + State + when() + halt_before` 的设计直接启发了我们的核心模型
- **[claude-workflow](https://github.com/AxGord/claude-workflow)**：YAML 状态机驱动 Claude Code，验证了"每步返回 prompt 指导 Agent"的模式可行
- **[LangGraph](https://github.com/langchain-ai/langgraph)**（28.7k stars）：`interrupt() + Command(resume=)` 是最成熟的暂停/恢复方案，但引入整个 LangGraph 生态太重了
- **[PocketFlow](https://github.com/The-Pocket/PocketFlow)**（10.4k stars）：100 行核心代码的极简 FSM，`post()` 返回字符串决定路由的设计很优雅

最终选了 Burr 的模式做简化——声明式 transitions + 条件路由 + halt 暂停，和 chatflow 架构（每轮用户消息触发一次处理）天然匹配。

整个实现大概 300 行代码。状态机不是什么高深的东西，但用对了地方，能让 Agent 的多步骤流程从"靠 LLM 记忆"变成"靠代码保证"。
