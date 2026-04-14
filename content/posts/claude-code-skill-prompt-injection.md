---
title: "翻了 Claude Code 源码，搞明白了 Skill Prompt 该往哪塞"
date: 2026-04-14
tags: ["AI Agent", "Claude Code", "Prompt Engineering", "LLM", "Tool Use"]
---

做 Agent 做到需要动态加载 prompt 的时候，第一直觉是做成一个 tool——模型识别到场景后调这个 tool，把对应的 prompt 作为 tool result 返回。

直觉很自然，但效果一般。模型对 tool result 里的内容遵从度不高，时不时就自己发挥，跳过关键步骤。

带着这个问题去翻了 Claude Code 的源码。Claude Code 有一套 Skill 系统，`/commit`、`/review-pr` 这些斜杠命令背后都是一段完整的 prompt，场景完全一样。它是怎么处理的？

## 系统提示词是动态拼装的

先看系统提示词的架构。Claude Code 的 system prompt 不是一整块文本，而是在 `constants/prompts.ts` 的 `getSystemPrompt()` 里运行时拼装的：

```typescript
return [
  // --- 静态部分（跨用户可缓存）---
  getSimpleIntroSection(),        // 身份
  getSimpleSystemSection(),       // 系统规则
  getSimpleDoingTasksSection(),   // 任务规则
  getActionsSection(),            // 操作安全
  getUsingYourToolsSection(),     // 工具使用
  getSimpleToneAndStyleSection(), // 语气
  getOutputEfficiencySection(),   // 输出效率

  // === 缓存分界线 ===
  SYSTEM_PROMPT_DYNAMIC_BOUNDARY,

  // --- 动态部分（会话相关）---
  sessionGuidance,     // 会话级指引
  memory,              // 记忆
  envInfo,             // 环境信息
  language,            // 语言偏好
  mcpInstructions,     // MCP 服务器指令
  ...
].filter(s => s !== null)
```

中间那个 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 是缓存分界线。之前的内容对所有用户一样，命中 prompt cache；之后的内容每个会话可能不同。

关键点：**skill 的完整 prompt 不在 system prompt 里。** 里面只放了 skill 的名称和触发条件，相当于一个路由索引：

```
- commit: 提交代码时使用
- review-pr: 审查 PR 时使用
- claude-api: 涉及 Claude API 开发时使用
  TRIGGER when: code imports `anthropic`...
  DO NOT TRIGGER when: file imports `openai`...
```

几十个 skill 的索引加起来也没多少 token，而且不会变，稳稳命中缓存。

## Skill Prompt 注入到了 user message

这是最核心的发现。

看 `SkillTool.ts` 的 `call()` 方法，skill 执行后返回的不只是 tool result，还有一个 `newMessages` 字段：

```typescript
// SkillTool.ts:767
return {
  data: {
    success: true,
    commandName,
  },
  newMessages,  // ← skill prompt 在这里
}
```

`newMessages` 是通过 `createUserMessage` 创建的：

```typescript
// SkillTool.ts:1101-1107（remote skill 的路径）
return {
  data: { success: true, commandName, status: 'inline' },
  newMessages: tagMessagesWithToolUseID(
    [createUserMessage({ content: finalContent, isMeta: true })],
    toolUseID,
  ),
}
```

而 tool result 本身只返回一句简短确认：

```typescript
// SkillTool.ts:857-861
mapToolResultToToolResultBlockParam(result, toolUseID) {
  return {
    type: 'tool_result',
    tool_use_id: toolUseID,
    content: `Launching skill: ${result.commandName}`,
  }
}
```

所以发给 Claude API 的消息序列实际上是这样的：

```
user:      "/commit"
assistant: tool_use(Skill, { skill: "commit" })
user:      tool_result("Launching skill: commit")     ← 简短确认
user:      "[完整的 commit skill prompt 内容...]"       ← 真正的指令
assistant: [按 skill prompt 开始执行]
```

**Skill prompt 不在 tool_result 的 content 里，而是作为一条独立的 `role: "user"` 消息追加到对话历史中。**

`isMeta: true` 是 Claude Code 内部的元数据标记，控制 UI 显示和消息合并逻辑。看 `createUserMessage` 的实现（`messages.ts:502`），发给 API 的时候它就是一条普通的 user message：

```typescript
const m: UserMessage = {
  type: 'user',
  message: {
    role: 'user',        // ← API 层面就是 role: "user"
    content: content,
  },
  isMeta,                // ← 内部标记，不发给 API
}
```

## 为什么不塞 tool result

这涉及到模型对不同消息类型的"信任层级"。

模型看到 tool_result 里的内容，会当作"我查到的数据"。它会参考，但不会像执行指令一样严格遵从。

模型看到 `role: "user"` 的内容，会当作"用户/系统给我的指令"。遵从度高很多。

大致的优先级：

```
system prompt > user message > tool result
```

Claude Code 选了中间档。不塞 system prompt（会破坏缓存），不塞 tool result（遵从度低），而是作为 user message 注入。

## 为什么不全塞 system prompt

如果 prompt 数量少、总量不大，全塞 system prompt 没问题，遵从度最高。

但 Claude Code 有几十个 skill，每个 prompt 可能几千 token。全塞进去有两个问题：

1. 每轮对话都带几万 token 的 system prompt，实际只用到其中一个，纯浪费
2. 每次触发不同 skill 就要改 system prompt 内容，prompt cache 直接废掉

所以分层：system prompt 放路由规则（固定不变，命中缓存），skill prompt 按需加载到 user message（只在触发时占用 token）。

## 还有一个容易忽略的细节

把 prompt 注入到 user message 后，模型可能会把它当成"用户说的话"去回应，而不是当指令去执行。

Claude Code 在 system prompt 里提前埋了一条解码规则：

```
If you see a <command-name> tag in the current conversation turn,
the skill has ALREADY been loaded —
follow the instructions directly instead of calling this tool again
```

注入的时候用 XML 标签包裹，模型结合 system prompt 里的规则就知道这不是用户在说话，而是要执行的指令。

两步配合：system prompt 里告诉模型"看到这个标签就执行"，注入时用标签明确标记。缺了任何一步，模型都可能困惑。

## 跨模型兼容

这个方案在 Claude 和 OpenAI 的 API 上实现方式不太一样。

Claude API 的 tool result 是 user message 里的 content block，可以在同一条消息里放 tool_result block 和 text block：

```json
{
  "role": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "xxx", "content": "Loaded." },
    { "type": "text", "text": "<instructions>完整 prompt</instructions>" }
  ]
}
```

OpenAI API 的 tool result 是独立的 `role: "tool"` 消息，不支持混合 content block。替代方案是在 tool message 后面追加一条 `developer` message：

```json
[
  { "role": "tool", "tool_call_id": "xxx", "content": "Loaded." },
  { "role": "developer", "content": "完整 prompt..." }
]
```

OpenAI 的 `developer` message 遵从度最高，GPT-4o 以上都支持。

兼容写法：

```python
def inject_prompt(messages, tool_id, prompt, provider):
    if provider == "anthropic":
        messages.append({
            "role": "user",
            "content": [
                {"type": "tool_result", "tool_use_id": tool_id, "content": "ok"},
                {"type": "text", "text": prompt}
            ]
        })
    elif provider == "openai":
        messages.append({"role": "tool", "tool_call_id": tool_id, "content": "ok"})
        messages.append({"role": "developer", "content": prompt})
```

核心思路一样——把 prompt 从 tool result 的 content 里拿出来，提升到指令层级。只是两家 API 的字段不同。
