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

状态存在数据库里（我们用的是 `session.metadata`），每轮只给 LLM 当前步骤的指令。

```
方案 A（LLM 记）：
  prompt 里塞 10 个步骤的完整指令
  LLM 从对话历史猜 "我大概做到第 6 步了？"

方案 B（代码记）：
  session.metadata["playbook"] = {current_step: "run_image"}
  只注入 "当前步骤：运行图片模块，调用 run_workflow"
```

方案 B 的状态不会丢、不会猜错、不占多余 token。这就是状态机的价值。

## 怎么做

我们的实现借鉴了 [Burr](https://github.com/DAGWorks-Inc/burr)（Apache 孵化的 Python FSM 框架）的核心设计，但做了简化，嵌入到已有的 Agent 架构里。

### 数据模型：4 个 dataclass

```python
from dataclasses import dataclass, field
from typing import Any

@dataclass(frozen=True)
class Transition:
    """一条边"""
    target: str
    condition: dict[str, Any] | None = None  # None = 无条件

@dataclass(frozen=True)
class Step:
    """一个步骤"""
    id: str
    name: str
    instruction: str                         # 注入给 Agent 的指令
    transitions: tuple[Transition, ...] = ()
    advance_mode: str = "manual"             # "auto" | "manual"
    auto_advance_on: tuple[str, ...] = ()    # auto 模式触发的 tool 名

@dataclass(frozen=True)
class Definition:
    """一套完整的 Playbook 定义"""
    id: str
    name: str
    description: str
    steps: tuple[Step, ...]
    initial_step: str

@dataclass
class State:
    """运行时状态，存 session.metadata["playbook"]"""
    playbook_id: str
    current_step: str
    data: dict[str, Any] = field(default_factory=dict)
    step_history: list[str] = field(default_factory=list)
    completed: bool = False
```

`Transition` 的 `condition` 就是 Burr 里 `when()` 的简化版——`{"approved": True}` 表示 `state.data["approved"] == True` 时走这条边。`None` 表示无条件跳转。

### 状态转移：遍历 transitions，匹配第一条

```python
def _advance(self, defn, state):
    step = self._get_step(defn, state.current_step)
    for t in step.transitions:
        if self._check_condition(t.condition, state.data):
            state.step_history.append(state.current_step)
            state.current_step = t.target
            return True
    return False

def _check_condition(self, condition, data):
    if condition is None:
        return True
    return all(data.get(k) == v for k, v in condition.items())
```

就这么简单。回退也是这个机制——`condition={"approved": False}` 匹配时 target 指向前面的步骤，对 engine 来说"回退"和"前进"没有区别，都是状态转移。

### 上下文注入：每轮只给当前步骤

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

这段文字通过 `build_messages()` 注入到对话历史前面，和 memory、context_summary 一样的机制。Agent 不需要从历史推断进度——进度条明确写在那里。

### 一个 tool 搞定所有操作

Agent 通过一个 `playbook` tool 管理整个生命周期：

```python
# 启动
playbook(action="start", playbook_id="ugc_video")

# 推进（带数据）
playbook(action="advance", step_data={"approved": true})

# 回退（也是推进，只是条件匹配到回退的边）
playbook(action="advance", step_data={"approved": false})

# 取消
playbook(action="cancel")
```

tool 的 `description` 和 `parameters.enum` 从 Engine 动态生成——加新 Playbook 不需要改 tool 代码。

### 两种推进模式

**MANUAL**：Agent 调 `playbook(action="advance")` 推进。用于需要判断的步骤（收集需求、审核结果）。

**AUTO**：Agent 调了特定 tool 后自动推进。比如 `run_script` 步骤的 `auto_advance_on=("run_workflow",)`，Agent 调完 `run_workflow` 后 engine 自动推进到 `review_script`，不需要 Agent 额外调 `advance`。

```python
# loop.py 里，Agent 执行完后：
if self._playbook_engine.try_auto_advance(state, tools_used):
    session.metadata["playbook"] = state.to_dict()
```

## 条件分支和回退

transitions 是有序列表，第一条匹配的执行。这意味着你可以声明任意复杂的路由：

```python
# 三种走向
Step(
    id="review_script",
    transitions=(
        Transition(target="build_image",   condition={"decision": "approve"}),
        Transition(target="revise_script", condition={"decision": "revise"}),
        Transition(target="build_script",  condition={"decision": "redo"}),
    ),
)

# 多条件 AND
Transition(target="premium", condition={"style": "vlog", "quality": "high"})

# 兜底 default
Transition(target="ask_again")  # 没有 condition = 无条件
```

Agent 调 `playbook(action="advance", step_data={"decision": "revise"})` 时，engine 遍历 transitions，匹配到第二条，跳到 `revise_script`。全部声明式，不需要写 if-else。

## 和 Agent 原有能力的关系

状态机是**叠加**在 Agent 上的，不是替换：

```
┌─────────────────────────────────────┐
│  Playbook 层（状态机）               │
│  - 每轮注入当前步骤指令              │
│  - 检查自动推进                      │
│  - 不限制 Agent 的任何能力           │
├─────────────────────────────────────┤
│  Agent 能力层                       │
│  - 所有原有 tools 可用               │
│  - 所有 skills 可用                  │
│  - LLM 推理 + tool calling          │
├─────────────────────────────────────┤
│  基础设施层                         │
│  - AgentLoop + AgentExecutor        │
│  - Session + PromptBuilder          │
└─────────────────────────────────────┘
```

Playbook 激活后，Agent 的 tool 集合是原有的全部 tools + 一个 `playbook` tool。Agent 在 Playbook 模式下依然可以搜索、读文件、加载 skill。用户问了个无关问题，Agent 回答完继续 Playbook——因为状态在 `session.metadata` 里，不会因为聊了别的就丢失。

## 加新 Playbook 的成本

写一个 Python 文件，定义 Definition + Steps，加到注册表里：

```python
# creato/core/playbook/definitions/ugc_video.py
UGC_VIDEO = Definition(
    id="ugc_video",
    name="UGC 视频制作",
    description="脚本→生图→生视频的三模块流水线",
    initial_step="collect_requirements",
    steps=(
        Step(id="collect_requirements", name="收集需求", ...),
        Step(id="build_script", name="构建脚本模块", ...),
        # ...
    ),
)

# definitions/__init__.py
ALL_DEFINITIONS = [UGC_VIDEO]
```

不需要改 engine、不需要改 loop、不需要改 tool。`playbook` tool 的 description 自动更新，Agent 自动能看到新选项。

## 这套方案的来源

调研了 30+ 个开源项目后提炼出来的。几个关键参考：

- **Burr**（~2k stars）：纯 FSM 框架，`Action + State + when() + halt_before` 的设计直接启发了我们的 `Step + Transition + condition`
- **claude-workflow**：YAML 状态机驱动 Claude Code，验证了"每步返回 prompt 指导 Agent"的模式可行
- **Lang*（28.7k stars）：`interrupt() + Command(resume=)` 是最成熟的暂停/恢复方案，但对我们来说太重了
- **PocketFlow**（10.4k stars）：100 行核心代码的极简 FSM，`post()` 返回字符串决定路由的设计很优雅

最终选了 Burr 的模式做简化——因为它最干净，声明式 transitions + 条件路由 + halt 暂停，和我们的 chatflow 架构（每轮用户消息触发一次处理）天然匹配。

整个实现大概 300 行 Python，改了 2 个现有文件（loop.py 加了 ~25 行，builder.py 加了 3 行）。状态机不是什么高深的东西，但用对了地方，能让 Agent 的多步骤流程从"靠 LLM 记忆"变成"靠代码保证"。
