# OpenClaw 模型调用设计笔记：缓存标记、工程设计、与 Claude Code 对比

> 源码版本：openclaw-2026.7.1；对比对象：本地 `src 2`（从代码特征判断是 Claude Code CLI 源码）
> 涉及函数：`createOpenRouterSystemCacheWrapper`（`src/llm/providers/stream-wrappers/proxy.ts:134`）、`packages/ai` openai-completions 传输层、`applyAnthropicPayloadPolicyToParams`（`src/agents/anthropic-payload-policy.ts:256`）
>
> 目录：
> - 第一至七节：OpenRouter 和直连 Anthropic 的缓存标记
> - 第八节：OpenClaw 模型调用的其他工程设计
> - 第九节：常见问题（包装链、cache-ttl 裁剪、冷却探测）
> - 第十节：与 Claude Code（src 2）的对比
> - 第十一节：缓存共享与计费（DeepSeek vs Claude、多 pod）
> - 第十二节：`cache_edits` 删工具结果的具体案例

## TL;DR

- Anthropic 的 prompt cache 必须在请求里显式带 `cache_control` 断点才会生效。OpenClaw 有**两条路径**会打这个标记：
  - **直连 Anthropic**：Anthropic Messages 原生格式，传输层组装请求体时一次打好，最多用 4 个断点。
  - **OpenRouter**：OpenAI chat 格式，OpenRouter 会把 content parts 上的 `cache_control` 透传给 Anthropic。这条路径由**两层**配合完成：
    1. `packages/ai` 的 openai-completions 传输层打一套完整标记：system 稳定前缀、最后一个 tool、最后一条稳定的对话消息；
    2. `createOpenRouterSystemCacheWrapper` 在请求发出前再补一遍：给 system 最后一块打标记，并删掉 thinking 块上的标记。
- OpenRouter 默认走哪个传输层，结果差别很大：
  - 默认走 `packages/ai`：两层都生效，结果和直连差不多。
  - 配了 `request.proxy`、`request.tls` 或 `localService`：会换成 OpenClaw 自己的传输层，**只剩 wrapper 在 system 上打的一个断点**，缓存分界也会被删掉。
- TTL：默认 5 分钟。显式配置 `cacheRetention: "long"` 时是 1h，`"none"` 时不打标记。

---

## 一、OpenRouter 路径

### 1.1 第一层：`packages/ai` 传输层（默认）

判断条件（`packages/ai/src/providers/openai-completions.ts:1373`）：

```ts
const cacheControlFormat =
  provider === "openrouter" && model.id.startsWith("anthropic/") ? "anthropic" : undefined;
```

model 配置里也可以用 `compat.cacheControlFormat` 显式覆盖（`:1439`）。

命中后，在组装请求体时打标记（`openai-completions.ts:756`、`:867`）：

```ts
function applyAnthropicCacheControl(messages, tools, cacheControl, cacheOptOutIndexes) {
  addCacheControlToSystemPrompt(messages, cacheControl);   // 第一条 system/developer
  addCacheControlToLastTool(tools, cacheControl);          // 最后一个 tool 定义
  addCacheControlToLastConversationMessage(messages, cacheControl, cacheOptOutIndexes);
}
```

- **system**：`buildCacheControlledTextParts`（`:966`）按缓存分界 `<!-- OPENCLAW_CACHE_BOUNDARY -->` 切开，**只给稳定前缀**打标记，动态后缀不打。分界在 `src/agents/system-prompt.ts:1291` 插入，前面是每轮不变的内容，后面是动态项目上下文。
- **对话**：从后往前找最后一条 user 或 assistant 消息，跳过 `cacheOptOutIndexes` 里的运行时上下文消息（`runtimeContextCarrier`，也就是每轮都变的元数据）。
- **TTL**（`getCompatCacheControl`，`:856`）：`"none"` 时不打标记；`"long"` 且 `compat.supportsLongCacheRetention` 为真时给 1h（这一项只有 Together、Cloudflare 为 false）；其他情况给 5 分钟。
- 测试：`openai-completions.test.ts:789`（按分界切开）、`:831`（跳过运行时上下文消息）。

### 1.2 第二层：`createOpenRouterSystemCacheWrapper`

挂载位置在 `src/agents/embedded-agent-runner/extra-params.ts:834`（`applyPostPluginStreamWrappers`），每次请求都会经过它。

```ts
// src/llm/providers/stream-wrappers/proxy.ts
if (
  !modelId ||
  !isAnthropicModelRef(modelId) ||                               // A：id 以 anthropic/ 开头
  !(
    endpointClass === "openrouter" ||                            // B1：host 命中 openrouter.ai
    (endpointClass === "default" && provider === "openrouter")   // B2：没配 baseUrl 且 provider=openrouter
  )
) {
  return underlying(model, context, options);                    // 原样透传
}
return streamWithPayloadPatch(underlying, model, context,
  stripCacheRetentionOption(options),
  (payloadObj) => applyAnthropicEphemeralCacheControlMarkers(
    payloadObj,
    resolveAnthropicEphemeralCacheControl(model.baseUrl, cacheRetention) ?? null,
  ));
```

- **条件 A**：`isAnthropicModelRef`（`anthropic-family-cache-semantics.ts:12`）只检查 `startsWith("anthropic/")`。
- **条件 B**：`endpointClass` 由 `resolveProviderEndpoint(baseUrl)` 算出（`src/agents/provider-attribution.ts:432`）：

  | baseUrl | endpointClass |
  |---|---|
  | 未配置 | `default` |
  | host 命中插件 manifest 的 `hostSuffixes: ["openrouter.ai"]`（`extensions/openrouter/openclaw.plugin.json:29`） | `openrouter` |
  | localhost / `*.local` / `*.internal` | `local` |
  | 其他 host（自建网关、任意 OpenAI 兼容代理） | `custom` |

- **注入方式**：`streamWithPayloadPatch`（`stream-payload-utils.ts:5`）挂一个 `onPayload` 钩子。下层 `buildParams` 组装完请求体后会调用它，钩子直接修改请求体对象，修改完才发出去（`openai-completions.ts:173-174`）。
- **`stripCacheRetentionOption`**：把 `cacheRetention` 从 options 里删掉。但更内层的 extraParams wrapper（`extra-params.ts:552-568`）会根据配置重新写回，所以下层还是能读到配置的值。
- **打标记**（`applyAnthropicEphemeralCacheControlMarkers`，`anthropic-payload-policy.ts:291`）：
  - system/developer 的 content 是字符串时，改写成 `[{ type: "text", text, cache_control }]`；
  - 是数组时，**无条件**给最后一个块（thinking 除外）打标记；
  - assistant 消息里 thinking/redacted_thinking 块上的 `cache_control` **一律删除**。OpenRouter 会拒绝挂在 reasoning 内容上的这个标记，所以 `"none"` 时也照样删（见测试 `extra-params.openrouter-cache-control.test.ts:96`）。

### 1.3 两种底层传输的实际效果

| 底层传输层 | 什么时候走 | 效果 |
|---|---|---|
| `packages/ai` openai-completions（`streamSimple`） | 默认 | 第一层打完整标记，第二层再叠加 |
| OpenClaw 传输层 `src/agents/openai-transport-stream.ts:4287` | 配了 `request.proxy`、`request.tls` 或 `localService`（`provider-transport-stream.ts:103`、`:121`） | 会先 `stripSystemPromptCacheBoundary` 删掉分界，自己不打标记，**只剩第二层在 system 上的一个断点**，对话历史不走缓存 |

### 1.4 两层的判断条件不一致

| 场景 | 第一层（看 provider 名） | 第二层（看 baseUrl host） |
|---|---|---|
| provider=`openrouter`，baseUrl=openrouter.ai 或未配置 | ✅ | ✅ |
| provider=`openrouter`，baseUrl 是自建网关 | ✅ | ❌（`custom`） |
| provider 是别的名字，baseUrl=openrouter.ai | ❌ | ✅ |
| 模型 id 没有 `anthropic/` 前缀 | ❌ | ❌ |

---

## 二、直连 Anthropic 路径

调用链：`src/agents/anthropic-transport-stream.ts:1012` 调 `resolveAnthropicPayloadPolicy({ enableCacheControl: true, ... })`，然后在 `:1124` 调 `applyAnthropicPayloadPolicyToParams`（`src/agents/anthropic-payload-policy.ts:256`）。

请求体是 Anthropic Messages 原生格式：顶层 `system` 块数组 + `tools` + `messages`。

1. **system**（`applyAnthropicCacheControlToSystem`）：
   - 有缓存分界：切成稳定前缀和动态后缀，**只给前缀打标记**；
   - 没有分界：给每个还**没有** `cache_control` 的 text 块打标记，已有的不覆盖；
   - 用 OAuth 登录（Claude 订阅）时，system 前面还有两个固定块（计费块、"You are Claude Code…"，`anthropic-transport-stream.ts:1037`），它们也各占一个断点。
2. **messages**（`applyAnthropicCacheControlToMessages`）：
   - 预算是 `ANTHROPIC_CACHE_CONTROL_LIMIT (4)` 减去 system 和 tools 已经用掉的断点；
   - 从后往前找**最后一条 user 消息**，跳过 `cacheBreakpointOptOutMessageIndexes` 里的运行时上下文消息（在 `convertAnthropicMessages` 转换时记录，`:422`、`:463`）；
   - 给最后一个 text 或 image 块打标记；有 `tool_result` 且还有剩余断点时，把 `tool_result` 也标上；只剩 1 个断点时优先标 `tool_result`。
3. **tools**：不主动打标记，只计入断点数量。
4. **`"none"`**：不打任何标记，只把 system 里的分界标记文本删掉（`stripAnthropicSystemPromptBoundary`）。

## 三、TTL 规则

wrapper 和直连路径用的是同一个 `resolveAnthropicEphemeralCacheControl`（`anthropic-payload-policy.ts:63`）：

| cacheRetention | 结果 |
|---|---|
| `"none"` | 不打标记 |
| 没配，也没有 `OPENCLAW_CACHE_RETENTION=long` | `{ type: "ephemeral" }`，5 分钟 |
| 显式配置 `"long"` | `{ type: "ephemeral", ttl: "1h" }` |
| 只通过环境变量得到 long | 只有 host 是 `api.anthropic.com` 或 Vertex 时才给 1h |

`packages/ai` 传输层走自己的 `getCompatCacheControl`，规则见 1.1。

---

## 四、对比

| 维度 | 直连 Anthropic | OpenRouter（默认传输层 + wrapper） |
|---|---|---|
| 请求格式 | Anthropic Messages：顶层 `system` 数组，内容块 | OpenAI chat：system 是一条消息，content 是 parts 数组 |
| 在哪打 | 组装请求体时打，由传输层负责 | 传输层组装时打一遍，wrapper 在 `onPayload` 里再补一遍 |
| system | 按分界切，只标稳定前缀；已有标记不覆盖 | 传输层标稳定前缀，wrapper **无条件**再标最后一块 |
| tools | 不打，只计数 | 标最后一个 tool |
| 对话历史 | 最后一条稳定的 user 消息，可带 tool_result | 最后一条稳定的 user 或 assistant 消息 |
| 断点上限 | 显式按 4 个预算分配 | 没有显式计数 |
| thinking 块 | 不需要特殊处理 | wrapper 删掉上面的 `cache_control` |
| 路由校验 | 不需要 | 只在 OpenRouter + `anthropic/*` 上打 |

## 五、为什么不一样

1. **请求格式不同。** Anthropic 原生格式的 system 在顶层，是块数组。OpenAI 格式的 system 是一条消息。OpenRouter 是把 content parts 上的 `cache_control` 原样透传，所以必须在 OpenAI 格式里的对应位置放标记，没法复用同一个函数。
2. **谁掌握上下文，谁才能标对话历史。** 直连路径和 `packages/ai` 都是在把内部消息转成请求体的**过程中**打标记，知道哪几条是运行时上下文消息，所以能把断点放在最后一条稳定消息上。wrapper 只看得到拼好的最终 JSON，分辨不出这些消息，所以只敢标位置固定的 system。它的定位是 OpenRouter 专用补丁，对话历史的缓存本来就应该由传输层负责。
3. **任意 OpenAI 兼容代理不一定接受 `cache_control`。** 发给不认识这个字段的后端可能直接报错，所以两层都只在确认是 OpenRouter + `anthropic/*` 时才打。Anthropic 官方端点天然支持，不需要这层判断。
4. **OpenRouter 会拒绝挂在 reasoning 内容上的 `cache_control`**，所以 wrapper 必须删掉 thinking 块上的标记。直连路径没有这个限制。

## 六、可能的问题（只读了代码，没跑验证）

走默认传输层时两层会叠加：
- 传输层把 system 切成 `[稳定前缀(cc), 动态后缀]`；
- wrapper **无条件**给最后一块打标记，最后一块正是**动态后缀**。

直连路径的 `applyAnthropicCacheControlToSystem` 会先检查块上有没有 `cache_control`，所以不会这样。

结果是 system 上多出一个断点，每轮动态内容一变就要写一次缓存，多一份缓存写入费用。数量上 system 2 个 + tool 1 个 + 对话 1 个，正好 4 个，没超过上限，不会报错，只是断点放得不太对。

验证方法：写一个测试，用默认 openai-completions 传输层跑 OpenRouter + `anthropic/*`，system 里带缓存分界，经过 `createOpenRouterSystemCacheWrapper` 之后，断言动态后缀上有没有 `cache_control`。

## 七、排查清单：为什么没命中缓存？

| 现象 | 原因 |
|---|---|
| provider 不叫 `openrouter`，baseUrl 也不是 openrouter.ai（比如自建网关） | 两层都不打标记 |
| 模型 id 写成 `claude-xxx`，没有 `anthropic/` 前缀 | 两层都不打标记 |
| 配了 `request.proxy`、`request.tls` 或 `localService` | 换成 OpenClaw 传输层，只剩 system 一个断点，对话历史不缓存 |
| 配置了 `cacheRetention: "none"` | 不打新标记 |
| 期望 1h，实际是 5 分钟 | 只设了环境变量，没有显式配置 `cacheRetention: "long"` |
| system 能命中，对话历史不能 | 看看是不是走了 OpenClaw 传输层，或者最后一条消息被判定成了运行时上下文 |

确认方法：检查实际的 provider 名、`baseUrl`、模型 id 写法、有没有 proxy/tls 配置，再抓一次请求体，看 `messages`、`tools` 里的 `cache_control` 都在哪。

---

## 八、OpenClaw 模型调用的其他工程设计

| # | 设计 | 位置 | 要点 |
|---|---|---|---|
| 1 | 包装链 + 请求体修改 | `extra-params.ts:828`、`stream-payload-utils.ts:5` | 统一的请求函数签名，各厂商的差异各写成一层包装；用 `onPayload` 在发送前修改请求体，不需要 fork SDK |
| 2 | 端点分类由插件声明 | `provider-attribution.ts:432`、插件 manifest 的 `hostSuffixes` | 核心代码不写死厂商域名，下游策略都基于分类结果 |
| 3 | 缓存分界 + 跳过运行时上下文 | `system-prompt.ts:1291`、`anthropic-transport-stream.ts:422` | 稳定前缀和动态内容分开；断点落在最后一条稳定消息上 |
| 4 | cache-ttl 裁剪 | `agent-hooks/context-pruning/extension.ts:16` | 缓存还没过期就不裁剪，过期后再清理旧的工具结果，保住前缀缓存 |
| 5 | 多账号轮换 + 分级冷却 | `auth-profiles/usage.ts:427` | 冷却 30 秒 → 1 分钟 → 最长 5 分钟；计费、认证、额度问题分别处理；状态存 SQLite |
| 6 | 备选链 + 冷却期间探测 | `model-fallback.ts:1194` | 每 30 秒最多探测一次，冷却到期前 2 分钟可以提前探测；没有配置备选模型时也会定期探测 |
| 7 | 首包超时 + 数据块间空闲超时 | `packages/ai/src/utils/stream-first-event-timeout.ts`、`run/llm-idle-timeout.ts:386` | 默认都是 120 秒（本地模型的首包超时是 300 秒）；连续 5 次空闲超时就熔断（`run/idle-timeout-breaker.ts`） |
| 8 | 压缩的一套防护 | `run.ts:1610`、`run/preemptive-compaction.ts`、`post-compaction-loop-guard.ts` | 发请求前预判是否压缩（预留至少 20k token）；溢出后最多压缩 3 次；压缩超时 180 秒；检测压缩后的死循环；结束状态统一归一 |

## 九、常见问题

### 9.1 包装是不是"每个都执行，符合条件的才处理"？
- 是的，但**不是互斥的**，同一个请求可以同时命中多层包装，各改各的；
- **越早套上的包装越靠里，它对请求体的修改越先执行**，因为 `streamWithPayloadPatch` 是先执行自己的修改、再调用外层的钩子；
- 判断时机有两种：大部分在每次请求时判断，也有一部分在套上时就决定了，不满足条件的根本不会套上（比如 `createStreamFnWithExtraParams` 返回 `undefined`，SiliconFlow、DeepSeek 相关的条件包装）。

### 9.2 会话一直在用的话，cache-ttl 裁剪是不是永远不触发？
- **对。** 每次正常结束的请求都会追加一条 `openclaw.cache-ttl` 时间戳（`run/attempt.thread-helpers.ts:82`），只要两轮之间不超过 5 分钟就不裁剪；
- 这种会话会一直涨到触发**预压缩**（估算值超过"上下文窗口减去预留量"），还不够就在溢出报错后压缩；
- 这样设计的原因：会话活跃时，读缓存大约是 0.1 倍价格，长历史其实很便宜；而裁剪会让整个缓存按约 1.25 倍重写，反而更贵；
- 这个模式需要显式配置 `contextPruning.mode: "cache-ttl"`。缓存过期后还要看工具结果在上下文里的占比：超过 30% 截成只保留首尾，超过 50% 直接替换成占位文本。

### 9.3 冷却期间的探测靠线程池吗？存在哪？
- **没有线程池，也没有定时任务**，是让下一个真实请求去试主模型（`resolveCooldownDecision`，`model-fallback.ts:1279`）；
- 失败时直接算出到期时间戳 `cooldownUntil`/`blockedUntil`：普通失败按退避时长，额度用尽时用厂商给出的 `reset_at`。之后判断 `now >= 最早到期时间 - 2 分钟` 就可以探测；
- 存储分两层：
  - 冷却状态存在每个 agent 的 SQLite `auth_profile_state` 表里，重启后保留；
  - 探测节流 `lastProbeAttempt` 存在进程内存的 `Map` 里，最多 256 个 key，24 小时过期；
- 过期的冷却记录**在读取时才处理**：解析 profile 顺序时调用 `clearExpiredCooldowns` 清掉（`auth-profiles/order.ts:258`），和 Redis 的惰性过期一样。

---

## 十、与 Claude Code（src 2）的对比

### 10.1 总体差异

| | Claude Code（src 2） | OpenClaw |
|---|---|---|
| 接哪些模型 | 只接 Anthropic 协议，部署渠道有官方 API、Bedrock、Vertex、Foundry 四种（`utils/model/providers.ts:6`） | 通过插件接入多家厂商、多种协议 |
| 请求怎么组装 | 用各渠道官方 SDK 的变体，在 `services/api/claude.ts`（3400 行）里一次性组装原生请求 | 包装链 + `onPayload` 修改请求体 |
| 取舍 | 只对接一家，缓存细节可以做到极致 | 要兼容多家，只能做通用抽象 |

### 10.2 缓存断点

| | Claude Code | OpenClaw |
|---|---|---|
| 对话历史上的断点 | **只打一个**，放在最后一条消息上；一次性的分支请求放到倒数第二条（`claude.ts:3078`） | 最多 4 个：system、最后一个 tool、最后一条稳定的消息 |
| system 的缓存范围 | 分三档：静态部分 `global`（所有用户共享）、CLI 前缀和 MCP 相关部分 `org`、动态部分不缓存（`utils/api.ts:321`） | 只分缓存和不缓存两档（`OPENCLAW_CACHE_BOUNDARY`） |
| 1h TTL | 资格和白名单**在会话内锁定**，防止 TTL 翻转导致缓存失效（`claude.ts:393`） | 直接取配置 |

**为什么只打一个断点：**
- 服务端缓存的是每个 token 的中间计算结果（KV cache）。断点的意思是"以后可能有请求要从这里接着算，请保留"；
- 命中缓存只需要最后一个断点：官方文档写明，服务端会从断点**往回最多查 20 个块**，所以能找到上一轮存下的位置；
- 如果再在倒数第二条消息上打一个断点，服务端会把那个位置的数据多留一轮，可下一轮是从最后一条接着算的，没有请求会再用到它，等于白占显存。src 2 的注释说，浪费的主要是局部注意力层那部分数据。这是 Anthropic 推理服务内部的优化，对调用方的命中效果影响不大。

**一次性分支请求（`skipCacheWrite`）：**
- 例子是 `utils/sideQuestion.ts:95` 和 `services/awaySummary.ts:56`：请求是"主对话 M1…M10 + 一条摘要指令 S"；
- 断点放在 S 上：要为 S 这部分付缓存写入的溢价，还占内存，可这个请求以后不会再发；
- 断点放在 M10 上：直接命中主对话已有的缓存，不写新内容。
- 原则：**断点只打在以后真的会有人接着用的位置。**

### 10.3 保持缓存稳定

| | Claude Code | OpenClaw |
|---|---|---|
| beta header | 一旦发过就一直带着，直到执行 `/clear` 或 `/compact`，避免中途开关导致 5~7 万 token 的缓存失效（`claude.ts:1405`） | 没有这种锁定 |
| 缓存失效检测 | `promptCacheBreakDetection.ts`：分别对 system、**每个工具**、beta、effort、extra body 等算哈希，缓存读取量下降时指出是哪一项变了，还会导出 diff | `prompt-cache-observability.ts`：跟踪 6 项，工具只算一个整体哈希，粒度较粗 |

### 10.4 上下文管理

| | Claude Code | OpenClaw |
|---|---|---|
| 空闲后裁剪 | 距上一条回复超过 60 分钟（此时 1h TTL 一定已过期），发请求前清掉旧的工具结果，保留最近 5 个（`timeBasedMCConfig.ts`） | cache-ttl 裁剪，默认 5 分钟 |
| 会话一直在用时 | **`cache_edits` 删工具结果，缓存不失效**（只在官方 API 的主线程上启用，见第十二节） | 只能等预压缩或溢出后压缩 |
| 自动压缩阈值 | 有效上下文窗口减 13k；连续失败 3 次就停止（`autoCompact.ts:62`） | 预留至少 20k；有四种处理方式；溢出后最多压缩 3 次；检测压缩后的死循环 |

### 10.5 失败重试与切换

| | Claude Code | OpenClaw |
|---|---|---|
| 退避 | 500ms × 2ⁿ，上限 32 秒，加 25% 随机抖动，优先用 `retry-after`，最多 10 次（`withRetry.ts:530`） | 按 profile 冷却，失败原因分类处理 |
| 切换模型 | 529 连续 3 次就改用唯一的 `fallbackModel`（`withRetry.ts:335`） | 跨厂商的备选链 |
| 多账号 | 没有 | 多 profile 轮换，状态存 SQLite |
| 过载保护 | **后台请求遇到 529 不重试**，避免放大（`withRetry.ts:58`、`:318`） | 空闲超时熔断，防止费用失控 |
| 连接问题 | 401 或 403（token 被吊销）时刷新 token；`ECONNRESET`/`EPIPE` 时关掉连接复用再重连 | OAuth 刷新有文件锁和刷新队列 |

### 10.6 流式输出卡住

```
发请求 ──► 收到响应头 ──► 第1个事件 ──► … ──► 第k个 ──► (卡住，不再有数据) ──► …
         └ SDK 的请求超时只管到这里
                        └ 首包超时管这一段
                                                   └ 数据块间的空闲超时管这种情况
```

- 两边**都有**数据块间的空闲检测：
  - Claude Code：90 秒没有新数据就中止，45 秒时先告警，中止后改用非流式请求重发（`claude.ts:1868`、`:818`）。这个功能需要通过环境变量开启；
  - OpenClaw：每读到一块数据就重新计时，默认 120 秒，之后走失败切换，连续 5 次就熔断。
- Claude Code 注释里提到的坑：如果在"下一个数据块到来时"才计算间隔，那么下一个块永远不来时就永远不会触发。所以必须用计时器主动检测。

### 10.7 结论

- **Claude Code 的优势**：只对接一家，缓存做到极致（断点数量、全局缓存范围、header 锁定、`cache_edits`），缓存失效可以观测到具体原因；
- **OpenClaw 的优势**：多厂商抽象、多账号和跨厂商的高可用、压缩和运行循环的异常防护更完整；
- **OpenClaw 可以借鉴的**：缓存失效检测按单个工具算哈希；对影响缓存 key 的参数做会话内锁定；后台请求在服务过载时不重试；
- **借鉴不了的**：`cache_edits` 和 `global` 范围只有 Anthropic 官方 API 支持，公开文档里也没有。

---

## 十一、缓存共享与计费：DeepSeek vs Claude，多个 pod

### 11.1 两家的缓存机制

| | Anthropic | DeepSeek |
|---|---|---|
| 怎么开启 | 用 `cache_control` 显式打断点，也支持在请求顶层写一个 `cache_control` 自动打 | 默认自动，不需要任何标记 |
| 写缓存要不要钱 | 要，5 分钟 TTL 是 1.25 倍价格，1h TTL 是 2 倍 | 不额外收费，只有命中时打折 |
| 保留多久 | 5 分钟或 1 小时，每次命中都会续期 | 不再使用后自动清理，"通常几小时到几天" |
| 匹配规则 | 前缀逐字节完全一致；顺序固定是 `tools → system → messages`；从断点往回最多查 20 个块 | 必须完整匹配一个缓存单元；单元在用户输入结束、模型输出结束、长内容按固定间隔处切分，间隔多少没写 |
| 隔离范围 | 按组织隔离，官方 API 上还按 workspace 隔离；Bedrock、Vertex 只按组织隔离 | **文档没写** |

**为什么 Anthropic 要手动打标记：** 因为写缓存要多付钱，所以让调用方自己决定哪里值得缓存。这和能不能共享无关。

### 11.2 同一个账号、不同的人、加载的 skill 不同

OpenClaw 稳定前缀的顺序（`system-prompt.ts:1060-1291`）：

```
[tools 参数] → ①身份 → ②## Tooling（工具列表）→ ③固定说明 → ④## Skills → ⑤## Workspace
→ ⑥# Project Context（AGENTS.md 等）→ ─ BOUNDARY ─ → ⑦动态上下文 / Runtime
```

- skill 不同，意味着**从 ④ 开始全部对不上**，即使 ⑥ 的 AGENTS.md 每个人都一样也命中不了；工具策略不同的话，② 就已经分叉了；
- ①~③ 这一段大家都一样，但在 DeepSeek 上能不能跨人命中，要看两件事：这一段是否包含一个完整的缓存单元，以及缓存按什么范围隔离，两者文档都没写；
- 对不上的部分**不是被多收一次钱**，而是按正常输入价格计费，不能享受命中价；
- 同一个人自己的会话从第二轮起照样命中率很高。

### 11.3 公共前缀共享、每个人的对话各自一份

```
            [tools + system 稳定前缀]    ← 共享一份（字节完全一致 + 在同一个隔离范围内）
             /         |          \
       [张三的对话] [李四的对话] [王五的对话] ← 每人各自一份
```

- 这**不是 Claude 独有的**。DeepSeek 文档里也有例子：同一个 system 加同一份财报、问不同的问题，后面的请求能命中公共部分；
- Claude 的优势在于能**确定地**把公共前缀变成一个缓存条目：OpenClaw 已经在 system 稳定前缀的末尾打了断点。DeepSeek 要等服务端自己发现这一段被反复用到，才会切出来；
- 跨组织共享（`scope: 'global'`）只有 Anthropic 自家的 Claude Code 在用，公开 API 没有。

### 11.4 Eva 每个人一个 pod，用 Claude 能不能共用公共前缀

- **pod 和 workspace 不是一回事。** Anthropic 的 workspace 是控制台里的概念，API key 属于某个 workspace。缓存隔离看的是 key 的归属，**和请求从哪个 pod 发出无关**；

| 情况 | 能否跨 pod 共享 |
|---|---|
| 所有 pod 用同一个 key，或同一个 workspace 下的多个 key | 可以，前提是前缀字节完全一致 |
| 不同 pod 用不同 workspace 的 key | 不可以 |
| 通过网关或代理调用 | 看网关调用 Anthropic 时用的是哪个 key |
| 走 Bedrock 或 Vertex | 同一个组织内就可以 |

- 真正的障碍通常是**前缀不一致**：每个人的 skill、工具策略、工作目录、上下文文件不同；
- 优化方向：把所有实例完全相同的内容放在前面，并保证逐字节一致，实例自己的内容往后放；
- 验证方法：从两个 pod 各导出一次实际发出的请求体，对比前缀从哪个字节开始不同。

---

## 十二、`cache_edits` 删工具结果的具体案例

> 依据：核心实现文件 `cachedMicrocompact.js` 不在 src 2 里，公开文档也没有这个 API。下面的流程是从 `claude.ts:3060-3210`、`microCompact.ts:295-380` 的调用代码推出来的，`cache_edits` 里各字段的具体写法是示意，不是准确格式。

**场景：** 先读了一个大日志 t1（5 万 token），又 grep 得到 t2（3 万 token），然后开始改代码。

**第 5 轮：正常请求**

```
M1  user:      "帮我查一下为什么报错"
M2  assistant: tool_use  id=t1  Read(app.log)
M3  user:      tool_result t1   ← 5 万 token
M4  assistant: tool_use  id=t2  Grep(...)
M5  user:      tool_result t2   ← 3 万 token
M6  assistant: "找到了，是 X 导致的"
M7  user:      "那改一下吧"      ← cache_control（唯一的断点）
```

服务端缓存下"到 M7 为止"这一整段，大约 9 万 token。

**第 6 轮：决定删掉 t1、t2**

- **普通做法**：把 M3、M5 替换成 `[Old tool result content cleared]`。M3 的字节变了，从 M3 往后全部对不上，要按 1.25 倍价格全部重新写缓存；
- **src 2 的做法**：

```
M1 … M2
M3  user: tool_result t1 + cache_reference:"t1"   ← 原文照发，只加了个 ID（每轮都会加）
M4
M5  user: tool_result t2 + cache_reference:"t2"
M6 … M7                                           ← M1~M7 字节和上一轮一致 → 命中
M8  assistant: tool_use id=t3 Edit(...)
M9  user: tool_result t3
          + { type: "cache_edits", edits: [删除 "t1", 删除 "t2"] }   ← 删除指令追加在末尾
          + cache_control
```

- 服务端先按缓存恢复 M1~M7，再执行删除指令，把 t1、t2 从模型能看到的上下文里去掉；
- 响应的用量里返回 `cache_deleted_input_tokens`（约 8 万）。客户端用这个实际值统计压缩效果，不在本地估算。

**第 7 轮：删除指令保留在原位置**

```
M1 … M9（M9 里仍然带着同一条 cache_edits）   ← 和第 6 轮一致 → 命中
M10 …
M11 user … + cache_control
```

`pinCacheEdits` 会记住删除块插在哪条消息里，之后每轮都在原位置原样重发，让它成为稳定前缀的一部分。还有 `seenDeleteRefs` 做去重，同一个 ID 不会被删两次。

**一句话概括：** 不改历史，而是在历史末尾追加一条"删除 t1、t2"的指令，前缀字节一个都不动，缓存就不会失效。代价是请求里仍然带着原文，但模型不再处理它们。这部分 token 怎么计费，从客户端代码看不出来。

---

## 测试参考

- `packages/ai/src/providers/openai-completions.test.ts:789`、`:831`：OpenRouter 传输层的分界切分和运行时上下文跳过
- `src/agents/embedded-agent-runner/extra-params.openrouter-cache-control.test.ts`：wrapper 的命中和不命中用例
- `src/llm/providers/stream-wrappers/proxy.test.ts`
- `src/agents/embedded-agent-runner-extraparams-openrouter.test.ts`

## 参考资料

- [Anthropic Prompt caching 文档](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [DeepSeek Context Caching 文档](https://api-docs.deepseek.com/guides/kv_cache)
