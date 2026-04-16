---
title: "搞懂 LLM 缓存机制：一个改动让 API 调用省 80% Token"
date: 2026-04-11
tags: ["LLM", "Cache", "Claude", "OpenAI", "Token优化", "AI工作流"]
---

做 AI 工作流平台的时候，token 费用是绕不开的问题。我们的 Consumer 服务每天要调几千次模型 API，一个工作流跑下来可能串联 5-6 个节点，每个节点都带着一大坨 system prompt。算下来，光是重复发送 system prompt 的 token 就占了总消耗的一大半。

直到我搞明白了 KV Cache 这个东西，才发现原来 API 厂商已经帮你做好了缓存——你只需要知道怎么触发它。

## KV Cache 是什么

Transformer 的注意力公式大家都见过：

```
Attention(Q, K, V) = softmax(Q·K^T / √d) · V
```

关键点在于：Decoder-only 架构（GPT、Claude、Gemma 这些）用的是 causal masking，每个 token 只看前面的 token。这意味着历史 token 的 K 和 V 算完之后就不会变了。

所以模型推理的时候会把历史 token 的 Key 和 Value 矩阵缓存起来，下次只需要算新 token 的 Q，然后跟缓存的 K、V 做注意力计算就行。这就是 KV Cache。

对于 API 用户来说，这个机制被包装成了 **Prompt Caching**（前缀缓存）：如果你连续多次请求的 prompt 前缀相同，服务端会复用之前算好的 KV Cache，跳过重复计算。

## 用 curl 实测：缓存命中 vs 未命中

说再多不如跑一下。用 Anthropic 的 API 直接测，先发一个带 `cache_control` 的请求：

```bash
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: prompt-caching-2024-07-31" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 256,
    "system": [
      {
        "type": "text",
        "text": "你是一个专业的视频脚本创作助手。你需要根据用户的需求，生成包含分镜描述、旁白文案、画面建议的完整视频脚本。每个分镜需要包含：场景描述、镜头运动、时长建议、旁白内容。输出格式为 JSON...(此处省略，实际是一段 2000+ token 的详细 system prompt)",
        "cache_control": {"type": "ephemeral"}
      }
    ],
    "messages": [
      {"role": "user", "content": "帮我写一个30秒的产品宣传视频脚本，产品是一款智能手表"}
    ]
  }'
```

看返回的 `usage` 字段：

```json
{
  "usage": {
    "input_tokens": 2150,
    "output_tokens": 180,
    "cache_creation_input_tokens": 2048,
    "cache_read_input_tokens": 0
  }
}
```

第一次请求，`cache_creation_input_tokens: 2048`，说明缓存刚建立。紧接着换个用户问题再发一次：

```bash
# 同样的 system prompt，只改 messages
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: prompt-caching-2024-07-31" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 256,
    "system": [
      {
        "type": "text",
        "text": "你是一个专业的视频脚本创作助手...(同上，完全一样的 system prompt)",
        "cache_control": {"type": "ephemeral"}
      }
    ],
    "messages": [
      {"role": "user", "content": "帮我写一个15秒的美食探店短视频脚本"}
    ]
  }'
```

返回：

```json
{
  "usage": {
    "input_tokens": 120,
    "output_tokens": 195,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 2048
  }
}
```

`cache_read_input_tokens: 2048`——2048 个 token 全部命中缓存。Anthropic 对缓存读取的计费是正常价格的 **1/10**，相当于这 2048 个 token 只花了原来十分之一的钱。

## 这对工作流引擎意味着什么

我们的架构是这样的：用户在画布上拖节点建工作流，点运行后 Consumer 按 DAG 拓扑顺序执行每个节点。每个节点调一次模型 API。

一个典型的视频制作工作流可能长这样：

```
[创意构思] → [脚本生成] → [分镜描述] → [配音文案] → [画面提示词]
```

5 个节点，每个都带一段 system prompt 描述它的角色和输出格式。如果这些 system prompt 有公共前缀（比如都以项目背景、品牌调性开头），那从第二个节点开始，公共前缀部分就能命中缓存。

算一笔账。假设每个节点的 system prompt 有 2000 token，其中 1500 token 是公共前缀：

| | 无缓存 | 有缓存 |
|---|---|---|
| 节点 1 | 2000 token（全价） | 2000 token（全价，建立缓存） |
| 节点 2-5 | 2000 × 4 = 8000 token（全价） | 500 × 4 = 2000 全价 + 1500 × 4 = 6000 缓存价（1/10） |
| **总计** | **10000 token** | **2000 + 2000 + 600 = 4600 等效 token** |

省了 54%。如果工作流节点更多、公共前缀更长，省得更多。

## OpenAI 也有类似机制

OpenAI 的 Prompt Caching 更简单粗暴——自动生效，不需要你手动标记 `cache_control`。只要请求的前缀匹配到 1024 token 以上，就会自动缓存：

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {
        "role": "system",
        "content": "你是一个专业的视频脚本创作助手...(同样的长 system prompt)"
      },
      {
        "role": "user",
        "content": "写一个30秒的产品宣传视频脚本"
      }
    ]
  }'
```

返回的 `usage` 里会有 `prompt_tokens_details.cached_tokens`：

```json
{
  "usage": {
    "prompt_tokens": 2150,
    "completion_tokens": 180,
    "prompt_tokens_details": {
      "cached_tokens": 2048
    }
  }
}
```

OpenAI 缓存命中的价格是正常的 **1/2**（Anthropic 是 1/10，这点 Anthropic 更划算）。缓存 TTL 大概 5-10 分钟，流量大的时候可能更长。

## 缓存不是一整块，是三段独立前缀

一开始我以为 Anthropic 的缓存是"前缀匹配，要么全中要么全 miss"。实际上不是。

Anthropic 把一次 API 请求的内容分成了三段缓存，按顺序排列：

```
tools（工具定义） → system（系统提示） → messages（对话消息）
```

每段有自己独立的缓存状态。改了其中一段，不一定所有缓存都废掉。但因为是前缀匹配，前面的段失效了，后面的段一定跟着失效。

[Anthropic 官方文档](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#caching-strategies-and-considerations)给了一张失效表，直接看这个最清楚：

| 哪些变化 | 工具缓存 | 系统缓存 | 消息缓存 | 影响 |
|---|---|---|---|---|
| 工具定义 | ✗ | ✗ | ✗ | 改了工具的名称、描述、参数，整个缓存全部失效 |
| 网页搜索/引用切换 | ✓ | ✗ | ✗ | 启用/禁用会修改系统提示，tools 缓存不受影响 |
| 速度设置 | ✓ | ✗ | ✗ | `speed: "fast"` 切换会使系统和消息缓存失效 |
| tool_choice | ✓ | ✓ | ✓ | 只影响消息块的元数据，三段缓存都不失效 |
| 图片 | ✓ | ✓ | ✗ | 在提示符中添加/删除图像会影响消息块 |
| 思考参数 | ✓ | ✓ | ✗ | 启用/禁用/改预算只影响消息缓存 |

规律很明显：**失效从前往后传播**。tools 变了，后面的 system 和 messages 缓存全废。但 system 变了，前面的 tools 缓存还在。messages 层面的变化（图片、thinking）不影响 tools 和 system。

这对打断点的策略有直接影响。你可以在 tools 末尾和 system 末尾各打一个 `cache_control`，这样即使 system prompt 因为功能开关变了，tools 那段缓存还能继续命中。不是"全有或全无"。

几个硬限制：

- 最多 **4 个**显式断点（automatic caching 也占一个槽位）
- 最小可缓存长度：Sonnet 1,024 tokens，Opus 和 Haiku 4,096 tokens
- 有个 **20-block lookback window**——多轮对话中，如果新消息把断点推到距离上次缓存写入超过 20 个 block 的位置，之前的缓存条目就在回溯窗口之外了，等于白缓存
- TTL 默认 5 分钟，每次命中刷新；1 小时 TTL 可选，但缓存写入价格翻倍

## 实际落地：怎么在工作流引擎里用好缓存

搞清楚原理之后，优化方向就很明确了：

**1. system prompt 设计：公共部分放前面**

把所有节点共享的上下文（项目背景、品牌调性、输出语言偏好）放在 system prompt 的最前面，节点特有的指令放后面。这样公共前缀越长，缓存命中率越高。

```python
# 不好的写法：每个节点的 system prompt 完全独立
system_prompt = f"你是一个{node_type}专家。{node_specific_instructions}"

# 好的写法：公共前缀 + 节点特有指令
system_prompt = f"""{project_context}
{brand_guidelines}
{output_language_preference}

---
当前任务：你是一个{node_type}专家。
{node_specific_instructions}"""
```

**2. 同模型的节点尽量连续执行**

KV Cache 是跟模型绑定的。如果工作流里混用了 Claude 和 GPT，它们的缓存是完全独立的。DAG 调度的时候，同模型的节点尽量排在一起执行，能最大化缓存命中。

**3. 对 Anthropic API 显式标记 cache_control**

OpenAI 自动缓存不用管，但 Anthropic 需要你在 system prompt 上加 `cache_control: {"type": "ephemeral"}`。漏了这个标记，缓存就不会生效。

**4. 注意缓存 TTL**

Anthropic 的缓存 TTL 是 5 分钟（Pro 用户 1 小时），OpenAI 大概 5-10 分钟。如果两次请求间隔超过 TTL，缓存就失效了。对于工作流引擎来说，节点之间的执行间隔通常在秒级，所以基本不用担心。但如果是用户手动触发的"继续运行"，间隔可能比较长，这时候缓存大概率已经过期了。

## Claude Code 的缓存策略也值得参考

顺便提一下，Claude Code 的 prompt 结构设计得很精巧，分了好几层缓存：

- 第一层：全局 system instructions（所有用户共享，全球级缓存）
- 第二层：用户级内容，比如 CLAUDE.md 文件（组织级缓存）
- 第三层：对话历史（会话级缓存，在关键断点设置 `cache_control`）

这个分层思路对工作流引擎也有启发：全局配置 > 项目配置 > 节点配置，越稳定的内容放越前面，缓存命中率就越高。

还有个细节：Claude Code 会监控 `cache_read_input_tokens`，如果这个值突然下降超过 5% 且超过 2000 token，就说明缓存被打破了。我们也可以在 Consumer 里加类似的监控，追踪每次 API 调用的缓存命中率，发现异常及时排查。

## 一句话总结

KV Cache 是模型推理层面的优化，API 厂商把它包装成了 Prompt Caching 给开发者用。对于我们这种多节点工作流场景，只要把 system prompt 的公共部分提到前面、给 Anthropic 的请求加上 `cache_control`，就能省掉大量重复 token 的费用。改动不大，效果立竿见影。
